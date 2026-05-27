---
title: Nginx Proxy Manager + Fail2Ban avec support IPv4 & IPv6 (Docker)
description: Guide technique pour configurer Nginx Proxy Manager et Fail2Ban sous Docker. Gérez les vraies IP (IPv4/IPv6), le géo-blocage et le bannissement incrémental. À adapter à votre propre configuration.
published: true
date: 2026-05-27T17:46:15.704Z
tags: docker, proxmox, nginx, fail2ban, ipv6, securite, geoip2
editor: markdown
dateCreated: 2026-05-27T17:15:10.772Z
---

> **Auteur** : ce guide est proposé par **Amaury aka BlablaLinux**. Retrouvez l'ensemble de mes services sur [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).
> **Environnement :** Proxmox VE — Debian LXC — Docker
> **Dernière mise à jour :** 27 mai 2026

⚠️ **Note importante :** Ce guide est assez technique et demande d'avoir déjà quelques bases avec Docker et la gestion réseau. Les exemples fournis (chemins de dossiers, fichiers de conf, blocs d'IP) proviennent directement de ma propre installation. Pensez donc à bien adapter ces éléments pour que tout fonctionne correctement chez vous !

---

## Aperçu

Ce guide explique comment configurer **Nginx Proxy Manager (NPM)** et **Fail2Ban** dans des conteneurs Docker, avec une prise en charge complète des **vraies IP des clients** (IPv4 et IPv6), du géo-blocage via GeoIP2 et du bannissement incrémental.

Les choix techniques importants sont :

* NPM tourne avec `network_mode: host` pour récupérer directement les vraies IP des clients (sans masquage par le bridge Docker).
* Fail2Ban utilise une action personnalisée qui cible la chaîne `INPUT` (au lieu de `DOCKER-USER`) pour fonctionner correctement avec le réseau de l'hôte.
* Un seul fichier d'action gère de manière transparente l'IPv4 (`iptables`) et l'IPv6 (`ip6tables`).

---

## Prérequis

* Docker avec l'IPv6 activé sur l'hôte.
* Le fichier `/etc/docker/daemon.json` configuré comme ceci :

```json
{
  "log-driver": "journald",
  "ipv6": true,
  "fixed-cidr-v6": "fd00:xxxx:x::/64",
  "ip6tables": true,
  "experimental": true
}

```

> Remplacez `fd00:xxxx:x::/64` par votre propre préfixe IPv6 privé.

* Bases de données GeoLite2 disponibles (via `maxmindinc/geoipupdate` ou manuellement).

---

## Structure des dossiers

```
~/
├── npm/
│   ├── docker-compose.yml
│   ├── data/
│   │   ├── logs_geo/
│   │   │   └── global_access_geo.log     ← Log partagé (NPM → Fail2Ban)
│   │   ├── geoip2/
│   │   │   └── GeoLite2-City.mmdb
│   │   └── nginx/
│   │       └── custom/
│   │           ├── http_top.conf
│   │           ├── server_proxy.conf
│   │           ├── ai-blocklist.conf
│   │           └── searxng-custom-block.conf
│   └── letsencrypt/
└── f2b/
    ├── docker-compose.yml
    └── data/
        ├── action.d/
        │   └── docker-action.conf
        ├── filter.d/
        │   ├── npm.conf
        │   ├── npm-403-abuse.conf
        │   └── npm-geo-abuse.conf
        └── jail.d/
            └── npm.local

```

---

## Partie 1 — Nginx Proxy Manager

### `npm/docker-compose.yml`

```yaml
services:
  npm:
    image: jc21/nginx-proxy-manager:latest
    container_name: npm
    restart: always

    # network_mode: host donne à NPM un accès direct à la pile réseau de l'hôte.
    # Cela permet de loguer directement les vraies IP des clients (IPv4 et IPv6),
    # sans qu'elles soient masquées par un bridge Docker (172.x.x.x).
    # Les ports suivants sont automatiquement ouverts sur l'hôte :
    # - 80  (HTTP)
    # - 443 (HTTPS)
    # - 81  (Interface admin NPM)
    network_mode: host

    environment:
      TZ: Europe/Brussels
      DISABLE_IPV6: 'false'
      IP_RANGES_FETCH_ENABLED: 'false'
      SKIP_CERTBOT_OWNERSHIP: 'true'
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
      - ./logrotate.custom:/etc/logrotate.d/nginx-proxy-manager
      - ./data/geoip2:/data/geoip2:ro
    healthcheck:
      test:
        - CMD
        - /usr/bin/check-health
      interval: 10s
      timeout: 3s

```

---

### `npm/data/nginx/custom/http_top.conf`

Ce fichier configure les recherches GeoIP2, définit les IP et pays autorisés, et configure le format de log partagé utilisé par Fail2Ban.

> Avec `network_mode: host`, `set_real_ip_from` n'est plus nécessaire — les IP des clients arrivent directement.

```nginx
charset utf-8;

geoip2 /data/geoip2/GeoLite2-City.mmdb {
    auto_reload 3h;
    $geoip2_data_country_code default=XX source=$remote_addr country iso_code;
}

geo $allowed_ip {
    default no;
    127.0.0.1/32 yes;
    ::1/128 yes;
    192.168.2.0/24 yes;
    fd00:d0ca:1::/64 yes;   # À adapter avec votre préfixe IPv6 Docker
}

map $geoip2_data_country_code $allowed_country {
    default yes;
    RU no;
    CN no;
    VN no;
    IN no;
    ID no;
    BR no;
    IR no;
    KP no;
    RO no;
    UA no;
    BY no;
    PL no;
    HK no;
}

log_format proxy_geo '[$time_local] [Client $remote_addr] [$allowed_country $geoip2_data_country_code] [$host] "$request" $status';

access_log /data/logs_geo/global_access_geo.log proxy_geo;

```

---

### `npm/data/nginx/custom/server_proxy.conf`

Appliqué à chaque hôte proxy. Gère le géo-blocage, le blocage des bots et l'application du code 403.

```nginx
# 1. Activer les logs pour Fail2Ban
access_log /data/logs_geo/global_access_geo.log proxy_geo;

# 2. Initialiser le flag de blocage
set $block 0;

# 3. Bloquer par pays (défini dans http_top.conf)
if ($allowed_country = no) {
    set $block 1;
}

# 4. Bloquer les bots IA connus (géré par un script externe)
include /data/nginx/custom/ai-blocklist.conf;

# 5. Règles personnalisées (scrapers, crawlers LinkedIn, etc.)
include /data/nginx/custom/searxng-custom-block.conf;

# 6. Toujours autoriser le robots.txt
if ($request_uri = "/robots.txt") {
    set $block 0;
}

# 7. Appliquer le verdict
if ($block) {
    return 403;
}

```

---

## Partie 2 — Fail2Ban

### `f2b/docker-compose.yml`

```yaml
services:
  fail2ban:
    image: crazymax/fail2ban:latest
    container_name: fail2ban
    network_mode: host
    cap_add:
      - NET_ADMIN
      - NET_RAW
    volumes:
      - ./data/filter.d:/etc/fail2ban/filter.d:ro
      - ./data/jail.d:/etc/fail2ban/jail.d:ro
      - ./data/action.d:/etc/fail2ban/action.d:ro
      - ./data/db:/data
      - /root/npm/data/logs_geo:/var/log:ro   # Volume du log partagé
    environment:
      - TZ=Europe/Brussels
      - F2B_LOG_TARGET=STDOUT
      - F2B_LOG_LEVEL=INFO
      - F2B_DB_PURGE_AGE=28d
    restart: always

```

---

### `f2b/data/action.d/docker-action.conf`

C'est le cœur du support double-pile (dual-stack) IPv4/IPv6.

Comme NPM utilise `network_mode: host`, les bannissements doivent cibler la chaîne `INPUT` (et non `DOCKER-USER`).
L'astuce `if echo "<ip>" | grep -q ':'` permet de sélectionner `ip6tables` pour les adresses IPv6 et `iptables` pour les IPv4.

```ini
[Definition]

actionstart = iptables -N f2b-npm 2>/dev/null || true
              iptables -F f2b-npm
              iptables -C INPUT -j f2b-npm 2>/dev/null || iptables -I INPUT -j f2b-npm
              iptables -A f2b-npm -j RETURN
              ip6tables -N f2b-npm 2>/dev/null || true
              ip6tables -F f2b-npm
              ip6tables -C INPUT -j f2b-npm 2>/dev/null || ip6tables -I INPUT -j f2b-npm
              ip6tables -A f2b-npm -j RETURN

actionstop = iptables -D INPUT -j f2b-npm 2>/dev/null || true
             iptables -F f2b-npm
             iptables -X f2b-npm
             ip6tables -D INPUT -j f2b-npm 2>/dev/null || true
             ip6tables -F f2b-npm
             ip6tables -X f2b-npm

actioncheck = iptables -n -L INPUT | grep -q 'f2b-npm'

actionban = if echo "<ip>" | grep -q ':'; then
              ip6tables -I f2b-npm -s <ip> -j DROP
            else
              iptables -I f2b-npm -s <ip> -j DROP
            fi

actionunban = if echo "<ip>" | grep -q ':'; then
                ip6tables -D f2b-npm -s <ip> -j DROP
              else
                iptables -D f2b-npm -s <ip> -j DROP
              fi

```

---

### `f2b/data/filter.d/npm.conf`

Attrape les erreurs HTTP des clients légitimes (400, 401, 404, 444, 5xx).

```ini
[Definition]

failregex = ^\[.*\] \[Client <ADDR>\] \[yes \w+\] \[.*\] ".*" (400|401|404|444|5\d\d)$

ignoreregex = ^.*minio\.example\.com.*$

```

---

### `f2b/data/filter.d/npm-403-abuse.conf`

Attrape les scanners agressifs qui touchent les pays autorisés avec des réponses 403.

```ini
[Definition]

failregex = ^\[.*\] \[Client <ADDR>\] \[yes \w+\] \[.*\] ".*" 403$

ignoreregex = ^.*minio\.example\.com.*$

```

---

### `f2b/data/filter.d/npm-geo-abuse.conf`

Attrape toutes les requêtes provenant d'un pays géo-bloqué (marqué `no` dans le log).

```ini
[Definition]

failregex = ^\[.*\] \[Client <ADDR>\] \[no \w+\] \[.*\] ".*" 403$

ignoreregex =

```

---

### `f2b/data/jail.d/npm.local`

```ini
[DEFAULT]

# =========================================================
# Paramètres généraux
# =========================================================

bantime  = 3600
findtime = 600
maxretry = 6

# =========================================================
# Bannissement incrémental
# =========================================================

bantime.increment = true
bantime.factor    = 2.0
bantime.maxtime   = 2592000
bantime.rndtime   = 600

# =========================================================
# IP ignorées (à adapter à votre réseau)
# =========================================================

ignoreip = 127.0.0.1/8 ::1 192.168.2.0/24 172.16.0.0/12 VOTRE.IP.PUBLIQUE/32 fe80::/10 fd00:d0ca:1::/64


# =========================================================
# GEO ABUSE — Géo-blocage massif (pays bloqués)
# =========================================================

[npm-geo-abuse]

enabled = true

logpath = /var/log/global_access_geo.log
filter  = npm-geo-abuse

maxretry = 1
findtime = 60

bantime = 604800
bantime.increment = false

action = docker-action


# =========================================================
# HTTP ABUSE — Bots, scanners, léger bruteforce
# =========================================================

[npm]

enabled = true

logpath = /var/log/global_access_geo.log
filter  = npm

maxretry = 20
findtime = 60

bantime = 3600
bantime.increment = true

action = docker-action


# =========================================================
# 403 ABUSE — Analyse agressive / accès interdit
# =========================================================

[npm-403-abuse]

enabled = true

logpath = /var/log/global_access_geo.log
filter  = npm-403-abuse

maxretry = 6
findtime = 600

bantime = 7200
bantime.increment = true

action = docker-action

```

---

## Partie 3 — Déploiement

### À appliquer dans le bon ordre

```bash
# 1. Démarrer ou redémarrer Fail2Ban
cd ~/f2b && docker compose down && docker compose up -d

# 2. Vérifier que les deux chaînes iptables sont bien créées
docker exec fail2ban iptables  -L INPUT | grep f2b-npm
docker exec fail2ban ip6tables -L INPUT | grep f2b-npm

# 3. Démarrer ou redémarrer NPM
cd ~/npm && docker compose down && docker compose up -d

# 4. Confirmer que NPM utilise bien le réseau host
docker inspect npm | grep NetworkMode

# 5. Suivre le log en direct
tail -f ~/npm/data/logs_geo/global_access_geo.log

```

---

## Partie 4 — Vérification

### Format de log attendu

```
[27/May/2026:18:38:24 +0200] [Client 2001:ee0:8104:c291:9d63:c5fb:1102:9237] [no VN] [searxng.example.be] "GET /search?..." 403
[27/May/2026:18:38:22 +0200] [Client 136.27.13.21] [yes US] [mastodon.example.be] "GET /users/..." 200

```

* Vraies adresses clients IPv4 et IPv6 — pas d'adresses de bridge `172.x.x.x`.
* `[no VN]` → pays bloqué → déclenche la prison `npm-geo-abuse`.
* `[yes US]` → pays autorisé → surveillé par `npm` et `npm-403-abuse`.

### Vérifier les bannissements actifs

```bash
docker exec fail2ban fail2ban-client status npm
docker exec fail2ban fail2ban-client status npm-403-abuse
docker exec fail2ban fail2ban-client status npm-geo-abuse

```

### Compter les IP bannies dans chaque chaîne

```bash
docker exec fail2ban iptables  -L f2b-npm | wc -l
docker exec fail2ban ip6tables -L f2b-npm | wc -l

```

---

## Notes

* Le `network_mode: host` sur NPM **supprime le besoin** de `set_real_ip_from` dans la config Nginx.
* La chaîne `DOCKER-USER` **n'est plus utilisée** — les bannissements vont directement dans `INPUT`.
* La sélection IPv4/IPv6 dans `actionban` utilise un simple test shell : si l'IP contient `:`, c'est de l'IPv6.
* Le bannissement incrémental de Fail2Ban (`bantime.increment`) signifie que les récidivistes ont des bannissements de plus en plus longs, jusqu'à 30 jours (`bantime.maxtime = 2592000`).
* La prison `npm-geo-abuse` n'utilise **pas** le bannissement incrémental — une seule requête d'un pays bloqué entraîne un bannissement immédiat de 7 jours.

---

## Bonus — Script pour IP publique dynamique

Si votre serveur a une IP publique dynamique, ce script met à jour automatiquement `ignoreip` dans `npm.local` et recharge Fail2Ban :

```bash
#!/bin/bash
set -euo pipefail

FAIL2BAN_CONFIG="/root/f2b/data/jail.d/npm.local"
FAIL2BAN_CONTAINER="fail2ban"
IP_FILE="/var/run/current_public_ip.txt"
LOG_FILE="/var/log/fail2ban_ip_update.log"

OLD_IP_CIDR=""
if [ -f "$IP_FILE" ]; then
    OLD_IP_CIDR=$(head -n 1 "$IP_FILE" 2>/dev/null || true)
fi

NEW_IP=$(curl -s --max-time 5 https://api.ipify.org || true)
if [ -z "$NEW_IP" ]; then
    echo "$(date) - ERREUR : impossible de récupérer l'IP publique" >> "$LOG_FILE"
    exit 1
fi

NEW_IP_CIDR="${NEW_IP}/32"

if [ "$OLD_IP_CIDR" = "$NEW_IP_CIDR" ]; then
    echo "$(date) - IP inchangée ($NEW_IP_CIDR)" >> "$LOG_FILE"
    exit 0
fi

echo "$(date) - IP modifiée : ${OLD_IP_CIDR:-} -> $NEW_IP_CIDR" >> "$LOG_FILE"

TMP_FILE=$(mktemp)
awk -v old="$OLD_IP_CIDR" -v new="$NEW_IP_CIDR" '
BEGIN { in_default=0; done=0 }
/^\[DEFAULT\]/ { in_default=1; print; next }
/^\[/ && !/^\[DEFAULT\]/ { in_default=0 }
{
    if (in_default && !done && /^[[:space:]]*ignoreip[[:space:]]*=/) {
        sub(/^[[:space:]]*ignoreip[[:space:]]*=[[:space:]]*/, "")
        n = split($0, arr, /[[:space:]]+/)
        count = 0
        delete ordered
        delete seen
        for (i = 1; i <= n; i++) {
            ip = arr[i]
            if (ip == "") continue
            if (old != "" && (ip == old)) continue
            if (ip in seen) continue
            seen[ip] = 1
            ordered[++count] = ip
        }
        if (!(new in seen)) { ordered[++count] = new }
        line = "ignoreip ="
        for (i = 1; i <= count; i++) { line = line " " ordered[i] }
        print line
        done=1
        next
    }
    print
}
' "$FAIL2BAN_CONFIG" > "$TMP_FILE"

if [ ! -s "$TMP_FILE" ]; then
    echo "$(date) - ERREUR : fichier temporaire vide, annulation" >> "$LOG_FILE"
    rm -f "$TMP_FILE"
    exit 1
fi

mv "$TMP_FILE" "$FAIL2BAN_CONFIG"
echo "$NEW_IP_CIDR" > "$IP_FILE"

if docker exec "$FAIL2BAN_CONTAINER" fail2ban-client reload >> "$LOG_FILE" 2>&1; then
    echo "$(date) - Rechargement Fail2Ban OK" >> "$LOG_FILE"
else
    echo "$(date) - Échec du rechargement, redémarrage du conteneur" >> "$LOG_FILE"
    docker restart "$FAIL2BAN_CONTAINER" >> "$LOG_FILE" 2>&1
fi

exit 0

```

Planifiez-le avec cron (toutes les 5 minutes) :

```bash
*/5 * * * * /root/scripts/update_fail2ban_ip.sh

```

---

*Cette configuration a été testée et validée sur Proxmox VE avec des conteneurs Debian LXC, Docker 27+, NPM 2.x, et Fail2Ban 1.1.0.*

---

## Ressources associées

### Nginx Proxy Manager
- [Utiliser Nginx Proxy Manager](https://wiki.blablalinux.be/fr/utiliser-nginx-proxy-manager)
- [Docker Compose Nginx Proxy Manager](https://wiki.blablalinux.be/fr/docker-compose-nginx-proxy-manager)
- [Meilleurs headers NPM / Nginx (sécurité + gzip)](https://wiki.blablalinux.be/fr/meilleurs-headers-npm-nginx-securite-gzip)
- [Injection d’un logo flottant dans Nginx / NPM](https://wiki.blablalinux.be/fr/injection-logo-flottant-nginx-npm)
- [Gestion simple du robots.txt (Wiki.js)](https://wiki.blablalinux.be/fr/gestion-robots-txt-simple-npm-wikijs)
- [Gestion centralisée du robots.txt dans NPM](https://wiki.blablalinux.be/fr/gestion-centralisee-robots-txt-nginx-proxy-manager)
- [WordPress : récupérer l’IP réelle derrière NPM](https://wiki.blablalinux.be/fr/wordpress-ip-reelle-nginx-proxy-manager)

### Fail2Ban
- [Docker Compose Fail2Ban](https://wiki.blablalinux.be/fr/docker-compose-fail2ban)
- [Sécurisation NPM + Fail2Ban + GeoIP2](https://wiki.blablalinux.be/fr/securisation-npm-fail2ban-geoip2)
- [Optimisation des logs Fail2Ban + NPM](https://wiki.blablalinux.be/fr/optimisation-logs-fail2ban-nginx-proxy-manager)
- [IP publique dynamique + Fail2Ban](https://wiki.blablalinux.be/fr/ip-dynamique-fail2ban)

### IPv6 / Réseau
- [Déploiement IPv6 OVH + Proxmox + Docker + NPM](https://wiki.blablalinux.be/fr/deploiement-ipv6-ovh-npm-proxmox-docker)
- [Installation alias GeoIP multisources](https://wiki.blablalinux.be/fr/installation-alias-geoip-multisources)

### Logs / Analyse
- [Docker Compose GoAccess](https://wiki.blablalinux.be/fr/docker-compose-goaccess)
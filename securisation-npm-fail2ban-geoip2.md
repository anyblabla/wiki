---
title: Sécurisation totale de Nginx Proxy Manager, Fail2Ban et GeoIP2
description: Guide complet pour protéger Nginx Proxy Manager avec Fail2Ban et GeoIP2 sous Docker. Bloquez les scans et les pays à risque grâce au bannissement automatique par iptables sur l'hôte.
published: true
date: 2026-05-10T18:22:24.127Z
tags: docker, nginx, fail2ban, sécurité, linux, sysadmin, nginxproxymanager, geoip, autohébergement, maxmind, protection
editor: markdown
dateCreated: 2026-05-07T00:10:11.725Z
---

Ce guide fournit une configuration « clés en main » pour protéger une infrastructure Docker avec une détection géographique et une riposte automatique via Fail2Ban.

## 📺 Vidéo de présentation

Retrouvez la démonstration complète et les explications détaillées dans cette vidéo :
👉 [https://peertube.blablalinux.be/w/3GvZh3PoGqYeRcDHq58ENz](https://peertube.blablalinux.be/w/3GvZh3PoGqYeRcDHq58ENz)

---

## 📁 1. Architecture des fichiers (arborescence)

Pour que tout fonctionne avec le fichier `docker-compose.yml` ci-dessous, vos fichiers doivent être organisés ainsi :

```text
/home/votre-user/npm/
├── docker-compose.yml
├── data/                      # Dossier de données de NPM
│   ├── geoip2/                # Contient les bases .mmdb
│   ├── logs_geo/              # Dossier de logs partagé
│   ├── nginx/
│   │   └── custom/
│   │       ├── root_top.conf
│   │       ├── http_top.conf
│   │       └── server_proxy.conf
└── fail2ban/                  # Dossier de données de Fail2Ban
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

## 🏗️ 2. Infrastructure Docker (`docker-compose.yml`)

**Note sur les volumes :** si vous regroupez ces services, veillez à ce que les chemins vers `logs_geo` et `geoip2` soient identiques pour les services qui les partagent.

**Astuce :** Une fois vos bases opérationnelles, vous pouvez les interroger directement en ligne de commande en suivant ce guide : [L'alias GeoIP ultime (MaxMind + ip66.dev)](/fr/installation-alias-geoip-multisources).

```yaml
services:
  # --- Nginx Proxy Manager ---
  npm:
    image: jc21/nginx-proxy-manager:latest
    container_name: npm
    restart: always
    ports:
      - 80:80
      - 443:443
      - 81:81
    environment:
      TZ: Europe/Brussels # À personnaliser
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
      - ./data/geoip2:/data/geoip2:ro

  # --- Mise à jour automatique des bases GeoIP ---
  # Un compte doit être créé sur https://www.maxmind.com pour obtenir
  # un ID utilisateur et une clé de licence (License Key).
  geoip-upd:
    image: maxmindinc/geoipupdate:latest
    container_name: geoip-upd
    restart: always
    environment:
      - GEOIPUPDATE_ACCOUNT_ID=VOTRE_ID_MAXMIND
      - GEOIPUPDATE_LICENSE_KEY=VOTRE_CLE_LICENCE
      - GEOIPUPDATE_EDITION_IDS=GeoLite2-ASN GeoLite2-City GeoLite2-Country
      - GEOIPUPDATE_FREQUENCY=12
    volumes:
      - ./data/geoip2:/usr/share/GeoIP

  # --- Fail2Ban (protection iptables) ---
  fail2ban:
    image: crazymax/fail2ban:latest
    container_name: fail2ban
    network_mode: host
    cap_add:
      - NET_ADMIN
      - NET_RAW
    volumes:
      - ./fail2ban/data/filter.d:/etc/fail2ban/filter.d:ro
      - ./fail2ban/data/jail.d:/etc/fail2ban/jail.d:ro
      - ./fail2ban/data/action.d:/etc/fail2ban/action.d:ro
      - ./data/logs_geo:/var/log:ro # Accès aux logs de NPM
    environment:
      - TZ: Europe/Brussels # À personnaliser
      - F2B_LOG_TARGET: STDOUT
      - F2B_LOG_LEVEL: INFO
    restart: always
```

---

## 🌐 3. Configuration de Nginx (côté NPM)

### Fichier `root_top.conf`

*(Chargement des modules système)*

```nginx
load_module /usr/lib/nginx/modules/ngx_http_geoip2_module.so;
load_module /usr/lib/nginx/modules/ngx_stream_geoip2_module.so;
```

### Fichier `http_top.conf`

*(Logique GeoIP globale et format de log)*

```nginx
charset utf-8;

geoip2 /data/geoip2/GeoLite2-City.mmdb {
    auto_reload 3h;
    $geoip2_data_country_code default=XX source=$remote_addr country iso_code;
}

# IPs de confiance (ne jamais bloquer)
geo $allowed_ip {
    default no;
    127.0.0.1/32 yes;
    192.168.x.x/24 yes; # Votre réseau local
}

# Liste noire des pays
map $geoip2_data_country_code $allowed_country {
    default yes;
    RU no; CN no; VN no; IN no; ID no; BR no; IR no; KP no;
}

# Format de log compatible avec les filtres Fail2Ban
log_format proxy_geo '[$time_local] [Client $remote_addr] [$allowed_country $geoip2_data_country_code] [$host] "$request" $status';

access_log /data/logs_geo/global_access_geo.log proxy_geo;
```

### Fichier `server_proxy.conf`

*(Application du blocage par proxy host)*

```nginx
# Activation du log pour Fail2Ban (indispensable !)
access_log /data/logs_geo/global_access_geo.log proxy_geo;

# Initialisation du blocage
set $block 0;

# Blocage pays (RU, CN, etc. — défini dans http_top.conf)
if ($allowed_country = no) { set $block 1; }

# Inclusion liste de bots/IA
# Pour plus d'explications sur ce blocage :
# 👉 https://wiki.blablalinux.be/fr/blocage-robots-ia-nginx-proxy-manager
include /data/nginx/custom/ai-blocklist.conf;

# Exception pour le fichier robots.txt
if ($request_uri = "/robots.txt") { set $block 0; }

# Verdict final (renvoi du 403 pour le filtre Fail2Ban)
if ($block) { return 403; }
```

---

## 🛡️ 4. Configuration de Fail2Ban (côté sécurité)

### Fichier `action.d/docker-action.conf`

```ini
[Definition]
actionstart = iptables -N f2b-npm-docker 2>/dev/null || true
              iptables -C DOCKER-USER -j f2b-npm-docker 2>/dev/null || iptables -I DOCKER-USER -j f2b-npm-docker
              iptables -F f2b-npm-docker
              iptables -A f2b-npm-docker -j RETURN

actionstop = iptables -D DOCKER-USER -j f2b-npm-docker 2>/dev/null || true
             iptables -F f2b-npm-docker
             iptables -X f2b-npm-docker

actioncheck = iptables -n -L DOCKER-USER | grep -q 'f2b-npm-docker'
actionban = iptables -I f2b-npm-docker -s <ip> -j DROP
actionunban = iptables -D f2b-npm-docker -s <ip> -j DROP
```

### Dossier `filter.d/` (les 3 fichiers)

**Fichier `npm.conf` :**

```ini
[Definition]

failregex = ^\[.*\] \[Client <HOST>\] \[yes \w+\] \[.*\] ".*" (400|401|404|444|5\d\d)$

ignoreregex = ^.*votre-sous-domaine\.votre-domaine\.tld.*$
```

**Fichier `npm-403-abuse.conf` :**

```ini
[Definition]

failregex = ^\[.*\] \[Client <HOST>\] \[yes \w+\] \[.*\] ".*" 403$

ignoreregex = ^.*votre-sous-domaine\.votre-domaine\.tld.*$
```

**Fichier `npm-geo-abuse.conf` :**

```ini
[Definition]

failregex = ^\[.*\] \[Client <HOST>\] \[no \w+\] \[.*\] ".*" (400|401|403|444|5\d\d)$

ignoreregex =
```

> **Note :** Si vous avez plusieurs sous-domaines à exclure, ajoutez une ligne par domaine en respectant l'indentation :
> ```ini
> ignoreregex = ^.*premier-sous-domaine\.votre-domaine\.tld.*$
>               ^.*second-sous-domaine\.votre-domaine\.tld.*$
> ```
> Le filtre `npm-geo-abuse.conf` ne comporte volontairement aucune exclusion — les pays géobloqués ne bénéficient d'aucune exception.

### Fichier `jail.d/npm.local`

*(La prison avec bannissement progressif)*

```ini
[DEFAULT]
bantime.increment = true
bantime.rndtime = 600
bantime.factor = 2
bantime.formula = ban.Time * (ban.Count if ban.Count<15 else 15) * banFactor
bantime.maxtime = 2592000
ignoreip = 127.0.0.1/8 ::1 192.168.x.x/24 votre_ip_publique/32

[npm-geo-abuse]
enabled = true
logpath = /var/log/global_access_geo.log
filter = npm-geo-abuse
maxretry = 1
findtime = 60
bantime = 604800
bantime.increment = false
action = docker-action

[npm]
enabled = true
logpath = /var/log/global_access_geo.log
filter = npm
maxretry = 100
findtime = 60
bantime = 3600
action = docker-action

[npm-403-abuse]
enabled = true
logpath = /var/log/global_access_geo.log
filter = npm-403-abuse
maxretry = 12
findtime = 600
bantime = 7200
action = docker-action
```

> **Note :** L'`ignoreip` est centralisée dans `[DEFAULT]` et s'applique automatiquement aux trois jails. Le script `update_fail2ban_ip.sh` maintient votre IP publique à jour automatiquement toutes les 5 minutes en cas de changement.

---

## 🔍 5. Comprendre les filtres (Explication des Regex)

Chaque filtre analyse les lignes du fichier de log global `global_access_geo.log`, généré par le format `proxy_geo` défini dans `http_top.conf`. Voici le format d'une ligne de log :

```
[10/May/2026:12:40:05 +0200] [Client 1.2.3.4] [yes FR] [monservice.mondomaine.be] "GET /page HTTP/2.0" 200
```

### A. La structure de base

Toutes les regex commencent par isoler l'IP du visiteur :

`^\[.*\] \[Client <HOST>\] ...`

* `^` : Début de la ligne.
* `\[.*\]` : Ignore le timestamp (date/heure).
* `\[Client <HOST>\]` : Capture l'adresse IP via l'étiquette Fail2Ban `<HOST>`.

> **Important :** Les crochets `[` et `]` doivent être échappés (`\[` et `\]`) dans les regex Fail2Ban car ce sont des caractères spéciaux.

### B. Filtre `npm.conf` (Comportements anormaux)

`failregex = ... \[yes \w+\] \[.*\] ".*" (400|401|404|444|5\d\d)$`

* `\[yes \w+\]` : Détecte un pays **autorisé** par GeoIP (ex: `[yes FR]`, `[yes BE]`).
* `(400|401|404|444|5\d\d)` : Cible les codes HTTP d'erreurs souvent liés à des scans (requêtes malformées, pages inexistantes, erreurs serveur répétées).
* **But** : Évincer les bots qui scannent vos services autorisés à la recherche de vulnérabilités.

### C. Filtre `npm-403-abuse.conf` (Forçage d'accès)

`failregex = ... \[yes \w+\] \[.*\] ".*" 403$`

* `403` : Code HTTP "Forbidden" (accès refusé).
* **But** : Si une IP insiste sur une ressource interdite (ex : interface d'administration, fichiers sensibles), elle est bannie. C'est une barrière contre le brute-force ou l'exploration forcée.

### D. Filtre `npm-geo-abuse.conf` (Répression géographique)

`failregex = ... \[no \w+\] \[.*\] ".*" (400|401|403|444|5\d\d)$`

* `\[no \w+\]` : **L'élément crucial.** Détecte uniquement les IP marquées comme **non autorisées** par la géolocalisation (ex: `[no CN]`, `[no RU]`).
* **But** : Tolérance zéro. Dès qu'une IP d'un pays banni génère une erreur, Fail2Ban déclenche le bannissement immédiat au niveau du pare-feu.

### E. Les exclusions (`ignoreregex`)

```ini
ignoreregex = ^.*votre-sous-domaine\.votre-domaine\.tld.*$
```

* **But** : Éviter les faux positifs sur certains sous-domaines qui génèrent naturellement des codes d'erreur légitimes. Par exemple, un stockage objet (S3/MinIO) derrière NPM retourne des `403` pour les accès directs non signés — sans exclusion, vos visiteurs PeerTube seraient bannis à tort.

> **Note :** L'exclusion sur les extensions de fichiers statiques (`.jpg`, `.css`, etc.) n'est pas nécessaire dans ce setup car ces assets ne génèrent jamais de codes d'erreur dans une configuration NPM standard.

---

## ✅ 6. Vérification et rechargement

Après toute modification des filtres ou du jail, validez avant de recharger :

```bash
# Tester les filtres contre les logs réels
docker exec fail2ban fail2ban-regex \
  /var/log/global_access_geo.log \
  /etc/fail2ban/filter.d/npm.conf

docker exec fail2ban fail2ban-regex \
  /var/log/global_access_geo.log \
  /etc/fail2ban/filter.d/npm-403-abuse.conf

docker exec fail2ban fail2ban-regex \
  /var/log/global_access_geo.log \
  /etc/fail2ban/filter.d/npm-geo-abuse.conf

# Tester la syntaxe du jail
docker exec fail2ban fail2ban-client --test

# Recharger si tout est OK
docker exec fail2ban fail2ban-client reload

# Vérifier le statut des jails
docker exec fail2ban fail2ban-client status
docker exec fail2ban fail2ban-client status npm
docker exec fail2ban fail2ban-client status npm-403-abuse
docker exec fail2ban fail2ban-client status npm-geo-abuse
```

---

> **Auteur** : ce guide est proposé par **Amaury aka BlablaLinux**. Retrouvez l'ensemble de mes services sur [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).

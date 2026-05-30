---
title: Nginx Proxy Manager — Sécurisation avancée avec Fail2Ban et blocage des chemins sensibles
description: Mettre en place une stack de sécurité complète sous Docker avec Nginx Proxy Manager, Fail2Ban et GeoIP2 pour bloquer bots, crawlers et chemins sensibles.
published: true
date: 2026-05-30T14:57:50.031Z
tags: docker, nginx, fail2ban, sécurité, geoip2, nginx-proxy-manager, self-hosting
editor: markdown
dateCreated: 2026-05-30T14:55:02.347Z
---

> **Environnement :** Proxmox VE — Debian LXC — Docker

⚠️ **Note importante :** Ce guide est assez technique et demande d'avoir déjà quelques bases avec Docker et la gestion réseau. Les exemples fournis (chemins de dossiers, fichiers de conf, blocs d'IP) sont des exemples génériques. Pensez à bien adapter ces éléments pour que tout fonctionne correctement chez vous !

> 💡 Ce guide s'appuie sur la configuration de base décrite dans [Sécurisation NPM + Fail2Ban + GeoIP2](https://wiki.blablalinux.be/fr/securisation-npm-fail2ban-geoip2). Il est recommandé de l'avoir mis en place avant de continuer.

---

## Aperçu

Cette page documente la mise en place d'une stack de sécurité complète pour **Nginx Proxy Manager (NPM)** associé à **Fail2Ban**, permettant de :

- Bloquer les bots IA et crawlers indésirables
- Bloquer par géolocalisation (GeoIP2)
- Bloquer les chemins sensibles universels (`.env`, `.git`, `/wp-admin`, etc.)
- Bannir automatiquement les IPs abusives via Fail2Ban

Les choix techniques importants sont :

- NPM tourne avec `network_mode: host` pour récupérer directement les vraies IPs des clients (sans masquage par le bridge Docker).
- Fail2Ban utilise une action personnalisée qui cible la chaîne `INPUT` (et non `DOCKER-USER`) pour fonctionner correctement avec le réseau de l'hôte.
- Un seul fichier d'action gère de manière transparente l'IPv4 (`iptables`) et l'IPv6 (`ip6tables`).

---

## Prérequis

- Docker et Docker Compose installés, avec l'IPv6 activé sur l'hôte
- Nginx Proxy Manager déployé (image `jc21/nginx-proxy-manager`) — voir [Docker Compose Nginx Proxy Manager](https://wiki.blablalinux.be/fr/docker-compose-nginx-proxy-manager)
- Fail2Ban déployé (image `crazymax/fail2ban`) — voir [Docker Compose Fail2Ban](https://wiki.blablalinux.be/fr/docker-compose-fail2ban)
- GeoIP2 configuré (base de données `GeoLite2-City.mmdb` disponible) — voir [Installation alias GeoIP multisources](https://wiki.blablalinux.be/fr/installation-alias-geoip-multisources)
- NPM configuré en `network_mode: host` pour récupérer les vraies IPs clients

> **Pourquoi `network_mode: host` ?**
> Sans cette option, NPM voit les IPs du bridge Docker interne (`172.x.x.x`) au lieu des vraies IPs clients. Cela rend le blocage par IP et le ban Fail2Ban inopérants. Voir aussi [Déploiement IPv6 OVH + Proxmox + Docker + NPM](https://wiki.blablalinux.be/fr/deploiement-ipv6-ovh-npm-proxmox-docker).

---

## Architecture des fichiers

```
/chemin/vers/npm/data/nginx/custom/
├── http_top.conf              ← GeoIP2, variables globales, format de log
├── server_proxy.conf          ← Chef d'orchestre : applique tous les blocs
├── ai-blocklist.conf          ← Blocage des bots IA et crawlers connus
├── custom-block.conf          ← Blocage par URI et User-Agent suspects
└── sensitive-paths-block.conf ← Blocage des chemins sensibles

/chemin/vers/fail2ban/data/
├── filter.d/
│   ├── npm.conf               ← Filtre erreurs HTTP classiques
│   ├── npm-403-abuse.conf     ← Filtre abus 403
│   ├── npm-geo-abuse.conf     ← Filtre abus géographiques
│   └── npm-wp-login.conf      ← Filtre brute-force WordPress
├── jail.d/
│   └── npm.local              ← Configuration des jails
└── action.d/
    └── docker-action.conf     ← Action iptables pour conteneurs Docker
```

> **Note :** Si tu as suivi le guide de base, tu as peut-être un fichier nommé `searxng-custom-block.conf`. Ce nom ne reflète plus son rôle réel — il protège l'ensemble des services, pas uniquement SearXNG. Il est recommandé de le renommer en `custom-block.conf` et de mettre à jour l'include dans `server_proxy.conf` en conséquence.

---

## Étape 1 — Format de log GeoIP2 (`http_top.conf`)

Ce fichier configure les recherches GeoIP2, définit les IPs et pays autorisés, et configure le format de log partagé utilisé par Fail2Ban.

> Avec `network_mode: host`, `set_real_ip_from` n'est plus nécessaire — les IPs des clients arrivent directement. Voir [Optimisation des logs Fail2Ban + NPM](https://wiki.blablalinux.be/fr/optimisation-logs-fail2ban-nginx-proxy-manager).

**Chemin :** `<répertoire NPM>/data/nginx/custom/http_top.conf`

```nginx
charset utf-8;

geoip2 /data/geoip2/GeoLite2-City.mmdb {
    auto_reload 3h;
    $geoip2_data_country_code default=XX source=$remote_addr country iso_code;
}

# IPs locales toujours autorisées
geo $allowed_ip {
    default no;
    127.0.0.1/32 yes;
    ::1/128 yes;
    192.168.0.0/16 yes;         # Adapter à ton réseau LAN
    fd00:d0ca:1::/64 yes;       # Adapter avec ton préfixe IPv6 Docker
}

# Blocage par pays (adapter selon tes besoins)
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
    # Ajouter d'autres codes pays ISO 3166-1 alpha-2 si nécessaire
}

# Format de log enrichi — utilisé par Fail2Ban
log_format proxy_geo '[$time_local] [Client $remote_addr] [$allowed_country $geoip2_data_country_code] [$host] "$request" $status';

access_log /data/logs_geo/global_access_geo.log proxy_geo;
```

> **Note :** Le fichier de log `/data/logs_geo/global_access_geo.log` doit être monté en lecture dans le conteneur Fail2Ban. Adapter les chemins selon ta configuration Docker.

---

## Étape 2 — Chef d'orchestre (`server_proxy.conf`)

Ce fichier est inclus dans chaque vhost NPM via le mécanisme de custom configs (`server_proxy[.]conf`). Il orchestre tous les blocs de sécurité dans le bon ordre.

**Chemin :** `<répertoire NPM>/data/nginx/custom/server_proxy.conf`

```nginx
# 1. Activation du log pour Fail2Ban (indispensable !)
access_log /data/logs_geo/global_access_geo.log proxy_geo;

# 2. Initialisation du blocage
set $block 0;

# 3. Blocage pays (défini dans http_top.conf)
if ($allowed_country = no) {
    set $block 1;
}

# 4. Inclusion de la liste des bots IA (gérée séparément)
include /data/nginx/custom/ai-blocklist.conf;

# 5. Inclusion des règles personnalisées (URI/User-Agent suspects)
include /data/nginx/custom/custom-block.conf;

# 6. Blocage des chemins sensibles
include /data/nginx/custom/sensitive-paths-block.conf;

# 7. Exception pour le robots.txt
if ($request_uri = "/robots.txt") {
    set $block 0;
}

# 8. Application du verdict (403 → détecté par Fail2Ban)
if ($block) {
    return 403;
}
```

> **Pourquoi le 403 ?** C'est le code HTTP utilisé par les filtres Fail2Ban. Un `444` (drop silencieux Nginx) ne serait pas loggé et donc invisible pour Fail2Ban. Un `404` induirait en erreur certains crawlers qui ignorent ce code.

---

## Étape 3 — Blocage des bots IA (`ai-blocklist.conf`)

Ce fichier contient la liste des User-Agents des bots IA et crawlers connus. Il est maintenu séparément pour faciliter les mises à jour.

**Chemin :** `<répertoire NPM>/data/nginx/custom/ai-blocklist.conf`

```nginx
if ($http_user_agent ~* "(Amazonbot|anthropic\-ai|Bytespider|CCBot|ChatGPT\-User|ClaudeBot|Diffbot|GPTBot|meta\-externalagent|PerplexityBot|SemrushBot|YandexBot)") {
    set $block 1;
}
```

> Cette liste est donnée à titre d'exemple. Il est recommandé de la maintenir à jour en consultant des sources comme [dark.visitors](https://dark.visitors.app) ou [ai.robots.txt](https://github.com/ai-robots-txt/ai.robots.txt). Le fichier peut être régénéré automatiquement par un script cron. Voir aussi [Gestion centralisée du robots.txt dans NPM](https://wiki.blablalinux.be/fr/gestion-centralisee-robots-txt-nginx-proxy-manager).

---

## Étape 4 — Règles personnalisées (`custom-block.conf`)

Ce fichier bloque les requêtes suspectes par URI et User-Agent, avec une exception pour le réseau local.

**Chemin :** `<répertoire NPM>/data/nginx/custom/custom-block.conf`

```nginx
# --- Blocage par mots-clés dans l'URI (spam, scraping, requêtes malveillantes) ---
if ($request_uri ~* "(\.php\?=|union.*select|eval\(|base64_decode|\/etc\/passwd)") {
    set $block 1;
}

# --- Blocage par User-Agent (clients HTTP scripts/outils de scraping) ---
# Exception pour les services légitimes qui utilisent ces clients (adapter selon tes services)
if ($http_user_agent ~* "^(?!.*(ton-service-legitime)).*(python-requests|go-http-client|axios|curl|wget|scrapy|headless|llamaindex|langchain)") {
    set $block 1;
}

# --- Exception réseau local (ne jamais bloquer le LAN) ---
if ($allowed_ip = "yes") {
    set $block 0;
}
```

> **Important :** L'exception réseau local (`$allowed_ip = "yes"`) remet `$block` à 0 pour les IPs locales, annulant les blocages précédents. Elle n'affecte **pas** le fichier `sensitive-paths-block.conf` qui utilise des `return 403` directs via des blocs `location`.

---

## Étape 5 — Blocage des chemins sensibles (`sensitive-paths-block.conf`)

C'est le fichier central de cette page. Il utilise des blocs `location ~*` plutôt que des `if` + `$block`, ce qui lui donne une **priorité supérieure** sur le `location /` des applications (WordPress, etc.) et garantit que le blocage s'applique avant que l'application backend ne réponde.

> C'est un point crucial : un simple `if ($request_uri ~* "/.env") { set $block 1; }` placé dans `server_proxy.conf` ne suffit pas. WordPress (ou tout autre backend) intercepte la requête via son propre `location /` avant que le verdict `$block` ne soit appliqué. L'utilisation de blocs `location ~*` dédiés contourne ce problème.

**Chemin :** `<répertoire NPM>/data/nginx/custom/sensitive-paths-block.conf`

```nginx
# =============================================================================
# SENSITIVE PATHS BLOCK
# Bloque les chemins sensibles universels via location blocks
# Priorité supérieure à location / — le backend ne voit jamais ces requêtes
# =============================================================================

# --- WordPress ---
location ~* "^/(wp-admin|wp-login\.php|xmlrpc\.php)(/|$)" {
    return 403;
}
location ~* "^/wp-json/wp/v2/users" {
    return 403;
}

# --- Fichiers et dossiers sensibles ---
location ~* "(\.env|\.git|\.htaccess|\.htpasswd|/backup|/dump)" {
    return 403;
}

# --- Shells et exploits ---
location ~* "^/(shell|cmd|exec|cgi-bin)(/|$)" {
    return 403;
}

# --- Path traversal ---
location ~* "\.\.\/" {
    return 403;
}

# --- Outils d'administration exposés ---
location ~* "^/(phpmyadmin|adminer|pma|server-status|nginx_status|actuator|metrics)(/|$)" {
    return 403;
}

# --- Exception : Let's Encrypt (ACME challenge) ---
# Ne jamais bloquer ce chemin, sous peine de rendre le renouvellement SSL impossible
location ~* "^/\.well-known/acme-challenge/" {
    allow all;
}
```

### Chemins à adapter selon tes services

Certains services self-hostés utilisent des chemins qui pourraient entrer en conflit avec des règles trop agressives. Voici les exclusions à prévoir si tu héberges ces services :

| Service | Chemins à ne pas bloquer |
|---|---|
| Matrix / Synapse | `/_matrix/`, `/_synapse/` |
| Element (client Matrix) | `/.well-known/matrix/` |
| Nextcloud | `/remote.php`, `/ocs/`, `/status.php` |
| Mastodon | `/api/`, `/.well-known/` |
| Vaultwarden / Bitwarden | `/api/`, `/identity/`, `/notifications/` |
| Gitea / Forgejo | `/api/`, `/user/oauth2/` |
| Joplin Server | `/api/` |

> En pratique avec NPM, si chaque service a son propre proxy host (son propre sous-domaine), les chemins spécifiques à ces services sont dans un vhost séparé et ne sont pas affectés par les règles WordPress ou phpmyadmin. Les conflits ne surviennent que si plusieurs services partagent le même domaine racine.

---

## Étape 6 — Configuration de Fail2Ban

### Action Docker (`action.d/docker-action.conf`)

Cette action utilise `iptables` directement sur l'hôte depuis le conteneur Fail2Ban (qui tourne en `network_mode: host` avec les capabilities `NET_ADMIN` et `NET_RAW`).

Comme NPM utilise `network_mode: host`, les bannissements doivent cibler la chaîne `INPUT` (et non `DOCKER-USER`). L'astuce `if echo "<ip>" | grep -q ':'` permet de sélectionner `ip6tables` pour les adresses IPv6 et `iptables` pour les IPv4.

**Chemin :** `<répertoire Fail2Ban>/data/action.d/docker-action.conf`

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

### Filtres (`filter.d/`)

**`npm.conf`** — Erreurs HTTP classiques (400, 401, 404, 5xx) :

```ini
[Definition]
failregex = ^\[.*\] \[Client <HOST>\] \[yes \w+\] \[.*\] ".*" (400|401|404|444|5\d\d)$
ignoreregex = ^.*ton-service-a-ignorer\.example\.com.*$
```

**`npm-403-abuse.conf`** — Scans agressifs, chemins interdits :

```ini
[Definition]
failregex = ^\[.*\] \[Client <HOST>\] \[yes \w+\] \[.*\] ".*" 403$
ignoreregex = ^.*ton-service-a-ignorer\.example\.com.*$
```

**`npm-geo-abuse.conf`** — IPs de pays bloqués :

```ini
[Definition]
failregex = ^\[.*\] \[Client <HOST>\] \[no \w+\] \[.*\] ".*" 403$
ignoreregex =
```

**`npm-wp-login.conf`** — Brute-force WordPress (voir [WordPress : récupérer l'IP réelle derrière NPM](https://wiki.blablalinux.be/fr/wordpress-ip-reelle-nginx-proxy-manager)) :

```ini
[Definition]
failregex = ^\[.*\] \[Client <HOST>\] \[yes \w+\] \[ton-domaine\.example\.com\] "POST /wp-login\.php HTTP/.*" 401$
ignoreregex =
```

> Remplacer `ton-domaine\.example\.com` par le domaine hébergeant WordPress. Échapper les points avec `\.`.

### Jails (`jail.d/npm.local`)

```ini
[DEFAULT]

# =========================================================
# Paramètres généraux
# =========================================================

bantime  = 3600
findtime = 600
maxretry = 6

# =========================================================
# Bannissement incrémental (doublement à chaque récidive, max 30 jours)
# =========================================================

bantime.increment = true
bantime.factor    = 2.0
bantime.maxtime   = 2592000
bantime.rndtime   = 600

# =========================================================
# IPs jamais bannies (adapter à ton réseau)
# =========================================================

ignoreip = 127.0.0.1/8 ::1 192.168.0.0/16 172.16.0.0/12 fe80::/10
# Ajouter ton IP fixe publique si tu en as une : 1.2.3.4/32
# Si ton IP est dynamique, voir le script en bas de page


# =========================================================
# Abus GeoIP — ban massif immédiat
# =========================================================

[npm-geo-abuse]
enabled  = true
logpath  = /var/log/global_access_geo.log  # Adapter au montage dans le conteneur F2B
filter   = npm-geo-abuse
maxretry = 1
findtime = 60
bantime  = 604800          # 7 jours
bantime.increment = false
action   = docker-action


# =========================================================
# Abus HTTP classique — bots, scans, bruteforce léger
# =========================================================

[npm]
enabled  = true
logpath  = /var/log/global_access_geo.log
filter   = npm
maxretry = 20
findtime = 60
bantime  = 3600
bantime.increment = true
action   = docker-action


# =========================================================
# Abus 403 — scans agressifs, chemins sensibles
# =========================================================

[npm-403-abuse]
enabled  = true
logpath  = /var/log/global_access_geo.log
filter   = npm-403-abuse
maxretry = 6
findtime = 600
bantime  = 7200
bantime.increment = true
action   = docker-action


# =========================================================
# Abus WordPress — brute-force login
# =========================================================

[npm-wp-login]
enabled  = true
logpath  = /var/log/global_access_geo.log
filter   = npm-wp-login
maxretry = 3
findtime = 60
bantime  = 86400           # 24 heures
bantime.increment = true
action   = docker-action
```

> **Bannissement incrémental :** la prison `npm-geo-abuse` n'utilise **pas** le bannissement incrémental — une seule requête d'un pays bloqué entraîne un bannissement immédiat de 7 jours. Pour toutes les autres jails, chaque récidive double la durée du ban jusqu'à 30 jours maximum.

---

## Étape 7 — Vérification et rechargement

### Tester la configuration Nginx

```bash
docker exec npm nginx -t
```

Résultat attendu :
```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Les `[warn] duplicate MIME type "text/html"` sont des avertissements bénins générés par NPM lui-même, non liés à cette configuration.

### Recharger Nginx

```bash
docker exec npm nginx -s reload
```

> NPM ne doit **pas** être redémarré : `nginx -s reload` suffit pour appliquer les changements sans interruption de service.

### Vérifier Fail2Ban

```bash
docker exec fail2ban fail2ban-client status
docker exec fail2ban fail2ban-client status npm-403-abuse
docker exec fail2ban fail2ban-client status npm-geo-abuse
```

### Compter les IPs bannies dans chaque chaîne

```bash
docker exec fail2ban iptables  -L f2b-npm | wc -l
docker exec fail2ban ip6tables -L f2b-npm | wc -l
```

---

## Étape 8 — Tests fonctionnels

Tester le blocage des chemins sensibles depuis l'extérieur :

```bash
curl -sk -o /dev/null -w "/.env        → %{http_code}\n" https://ton-domaine.example.com/.env
curl -sk -o /dev/null -w "/wp-admin/   → %{http_code}\n" https://ton-domaine.example.com/wp-admin/
curl -sk -o /dev/null -w "/.git/config → %{http_code}\n" https://ton-domaine.example.com/.git/config
curl -sk -o /dev/null -w "/phpmyadmin  → %{http_code}\n" https://ton-domaine.example.com/phpmyadmin
curl -sk -o /dev/null -w "/xmlrpc.php  → %{http_code}\n" https://ton-domaine.example.com/xmlrpc.php
```

Tous doivent retourner `403`.

Tester que Let's Encrypt n'est pas bloqué :

```bash
curl -sk -o /dev/null -w "/.well-known/acme-challenge/ → %{http_code}\n" \
  https://ton-domaine.example.com/.well-known/acme-challenge/test
```

Doit retourner `404` (chemin inexistant mais non bloqué) et non `403`.

### Format de log attendu

```
[30/May/2026:16:24:16 +0200] [Client 18.97.9.102] [yes US] [exemple.com] "GET /article/" 403
[30/May/2026:16:25:01 +0200] [Client 2001:ee0:8104:c291::1] [no VN] [exemple.com] "GET /search" 403
[30/May/2026:16:25:44 +0200] [Client 192.168.1.10] [yes BE] [exemple.com] "GET /page/" 200
```

- Vraies adresses clients IPv4 et IPv6 — pas d'adresses de bridge `172.x.x.x`
- `[no VN]` → pays bloqué → déclenche la jail `npm-geo-abuse`
- `[yes US]` avec `403` → pays autorisé mais accès interdit → déclenche `npm-403-abuse`

---

## Flux de décision complet

```
Requête entrante
       │
       ▼
  $block = 0
       │
       ├─ Pays bloqué (GeoIP) ?      → $block = 1
       ├─ Bot IA connu ?              → $block = 1
       ├─ URI/UA suspect ?            → $block = 1
       └─ IP locale ?                 → $block = 0  (reset)
       │
       ▼
  location ~* chemin sensible ?
       ├─ Oui → return 403 DIRECT (bypass $block, Fail2Ban le voit)
       └─ Non → continue
       │
       ▼
  $request_uri = /robots.txt ?
       └─ Oui → $block = 0
       │
       ▼
  $block = 1 ? → return 403 → Fail2Ban → ban iptables/ip6tables
  $block = 0 ? → proxy_pass vers le backend
```

---

## Points importants à retenir

**Pas de conflit entre les fichiers** : `ai-blocklist.conf`, `custom-block.conf` et `sensitive-paths-block.conf` agissent sur des mécanismes différents et complémentaires. Les deux premiers utilisent `$block` (variable), le troisième utilise `return 403` direct via `location`.

**L'exception LAN dans `custom-block.conf`** remet `$block` à 0 pour les IPs locales, mais n'affecte pas `sensitive-paths-block.conf`. Un client LAN qui accède à `/.env` reçoit quand même un `403` — c'est le comportement recommandé (défense en profondeur).

**Fail2Ban voit tous les `403`** : qu'ils viennent du blocage par pays, par User-Agent, par URI ou par chemin sensible, tous génèrent un `403` loggé dans `global_access_geo.log` et donc traités par les jails.

**La chaîne `DOCKER-USER` n'est pas utilisée** — les bannissements vont directement dans `INPUT`, ce qui est indispensable avec `network_mode: host`.

**La sélection IPv4/IPv6** dans `actionban` utilise un simple test shell : si l'IP contient `:`, c'est de l'IPv6 → `ip6tables`, sinon → `iptables`.

---

## Bonus — Script pour IP publique dynamique

Si ton serveur a une IP publique dynamique, ce script met à jour automatiquement `ignoreip` dans `npm.local` et recharge Fail2Ban. Voir aussi [IP publique dynamique + Fail2Ban](https://wiki.blablalinux.be/fr/ip-dynamique-fail2ban).

```bash
#!/bin/bash
set -euo pipefail

FAIL2BAN_CONFIG="/chemin/vers/f2b/data/jail.d/npm.local"
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

mv "$TMP_FILE" "$FAIL2BAN_CONFIG"
echo "$NEW_IP_CIDR" > "$IP_FILE"

if docker exec "$FAIL2BAN_CONTAINER" fail2ban-client reload >> "$LOG_FILE" 2>&1; then
    echo "$(date) - Rechargement Fail2Ban OK" >> "$LOG_FILE"
else
    docker restart "$FAIL2BAN_CONTAINER" >> "$LOG_FILE" 2>&1
fi
```

Planifier avec cron (toutes les 5 minutes) :

```bash
*/5 * * * * /chemin/vers/scripts/update_fail2ban_ip.sh
```

---

## Ressources associées

### Nginx Proxy Manager
- [Utiliser Nginx Proxy Manager](https://wiki.blablalinux.be/fr/utiliser-nginx-proxy-manager)
- [Docker Compose Nginx Proxy Manager](https://wiki.blablalinux.be/fr/docker-compose-nginx-proxy-manager)
- [Meilleurs headers NPM / Nginx (sécurité + gzip)](https://wiki.blablalinux.be/fr/meilleurs-headers-npm-nginx-securite-gzip)
- [Injection d'un logo flottant dans Nginx / NPM](https://wiki.blablalinux.be/fr/injection-logo-flottant-nginx-npm)
- [Gestion simple du robots.txt (Wiki.js)](https://wiki.blablalinux.be/fr/gestion-robots-txt-simple-npm-wikijs)
- [Gestion centralisée du robots.txt dans NPM](https://wiki.blablalinux.be/fr/gestion-centralisee-robots-txt-nginx-proxy-manager)
- [WordPress : récupérer l'IP réelle derrière NPM](https://wiki.blablalinux.be/fr/wordpress-ip-reelle-nginx-proxy-manager)

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

---

*Cette configuration a été testée et validée sur Proxmox VE avec des conteneurs Debian LXC, Docker 27+, NPM 2.x et Fail2Ban 1.1.0.*

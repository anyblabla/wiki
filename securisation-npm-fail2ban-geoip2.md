---
title: Sécurisation totale de Nginx Proxy Manager, Fail2Ban et GeoIP2
description: Guide complet pour protéger Nginx Proxy Manager avec Fail2Ban et GeoIP2 sous Docker. Bloquez les scans et les pays à risque grâce au bannissement automatique par iptables sur l'hôte.
published: true
date: 2026-05-07T00:39:56.241Z
tags: docker, nginx, fail2ban, sécurité, linux, sysadmin, nginxproxymanager, geoip, autohébergement, maxmind, protection
editor: markdown
dateCreated: 2026-05-07T00:10:11.725Z
---

Ce guide fournit une configuration « clés en main » pour protéger une infrastructure Docker avec une détection géographique et une riposte automatique via Fail2Ban.

## 📺 Vidéo de présentation
Retrouvez la démonstration complète et les explications détaillées dans cette vidéo :
👉 [https://video.blablalinux.be/w/56vT186n8Yp5VzS8S7WzPz](https://video.blablalinux.be/w/56vT186n8Yp5VzS8S7WzPz)

---

## 📁 1. Architecture des fichiers (arborescence)

Pour que tout fonctionne, vos fichiers doivent être organisés ainsi sur votre hôte :

```text
/home/votre-user/npm/
├── docker-compose.yml
├── data/
│   ├── geoip2/                # Contient les bases .mmdb
│   ├── logs_geo/              # Dossier de logs partagé
│   ├── nginx/
│   │   └── custom/
│   │       ├── root_top.conf
│   │       ├── http_top.conf
│   │       └── server_proxy.conf
├── fail2ban/
│   ├── data/
│   │   ├── action.d/docker-action.conf
│   │   ├── filter.d/ (npm.conf, npm-403-abuse.conf, npm-geo-abuse.conf)
│   │   └── jail.d/npm.local
```

---

## 🏗️ 2. Infrastructure Docker (`docker-compose.yml`)

**Note sur les volumes :** si vous regroupez ces services, veillez à ce que les chemins vers `logs_geo` et `geoip2` soient identiques pour les services qui les partagent.

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
      TZ: Europe/Brussels
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
      - ./data/geoip2:/data/geoip2:ro

  # --- Mise à jour automatique des bases GeoIP ---
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
      - TZ: Europe/Brussels
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
*(Application du blocage par proxy host - à mettre dans l'onglet "Advanced" ou en config globale)*
```nginx
access_log /data/logs_geo/global_access_geo.log proxy_geo;

set $block 0;

# Blocage pays (RU, CN, etc.)
if ($allowed_country = no) { set $block 1; }

# Inclusion liste de bots/IA (optionnel)
include /data/nginx/custom/ai-blocklist.conf;

# Exception pour le fichier robots.txt
if ($request_uri = "/robots.txt") { set $block 0; }

# Verdict final
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
`failregex = ^\[.*\] \[Client <HOST>\] \[yes \w+\] \[.*\] ".*" (400|401|404|444|5\d\d)$`
`ignoreregex = ^.*votre-domaine\.com.*$`

**Fichier `npm-403-abuse.conf` :**
`failregex = ^\[.*\] \[Client <HOST>\] \[yes \w+\] \[.*\] ".*" 403$`
`ignoreregex = ^.*votre-domaine\.com.*$`

**Fichier `npm-geo-abuse.conf` :**
`failregex = ^\[.*\] \[Client <HOST>\] \[no \w+\] \[.*\] ".*" (400|401|403|444|5\d\d)$`

### Fichier `jail.d/npm.local`
*(La prison complète avec bannissement progressif)*
```ini
[DEFAULT]
bantime.increment = true
bantime.rndtime = 600
bantime.factor = 2
bantime.formula = ban.Time * (ban.Count if ban.Count<15 else 15) * banFactor
bantime.maxtime = 2592000

[npm-geo-abuse]
enabled = true
logpath = /var/log/global_access_geo.log
filter = npm-geo-abuse
ignoreip = 127.0.0.1/8 ::1 192.168.x.x votre_ip_publique
maxretry = 1
findtime = 60
bantime = 86400
action = docker-action

[npm]
enabled = true
logpath = /var/log/global_access_geo.log
filter = npm
ignoreip = 127.0.0.1/8 ::1 192.168.x.x votre_ip_publique
maxretry = 8
findtime = 300
bantime = 3600
action = docker-action

[npm-403-abuse]
enabled = true
logpath = /var/log/global_access_geo.log
filter = npm-403-abuse
ignoreip = 127.0.0.1/8 ::1 192.168.x.x votre_ip_publique
maxretry = 12
findtime = 120
bantime = 7200
action = docker-action
```

---

> **Auteur** : ce guide est proposé par **Amaury aka BlablaLinux**. Retrouvez l'ensemble de mes services sur [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).
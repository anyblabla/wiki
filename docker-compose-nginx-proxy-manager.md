---
title: Docker Compose Nginx Proxy Manager
description: Nginx Proxy Manager (NPM) est un proxy inverse open source utilisé pour rediriger le trafic d'un site Web vers l'endroit approprié.
published: true
date: 2025-07-16T23:43:08.813Z
tags: nginx, proxy
editor: markdown
dateCreated: 2024-06-13T20:53:40.020Z
---

Pratique pour déployer rapidement [Nginx Proxy Manager](https://nginxproxymanager.com) (NPM) dans [Portainer](https://www.portainer.io/) en créant une pile ([stack](https://docs.portainer.io/user/docker/stacks)) à partir d'un fichier [compose YAML](https://docs.docker.com/compose/compose-application-model/).

Je pars du principe que vous maîtrisez un minimum Docker avec Portainer 😉

# Nginx Proxy Manager, c'est quoi ?

*Nginx Proxy Manager (NPM) est un proxy inverse open source utilisé pour rediriger le trafic d'un site Web vers l'endroit approprié. L'utilisation de Nginx Proxy Manager vous permet d'utiliser une seule adresse IP publique pour héberger de nombreux services Web différents.*

## Liens utiles

-   [NPM Site officiel](https://nginxproxymanager.com)
-   [NPM sur GitHub](https://github.com/NginxProxyManager/nginx-proxy-manager)
-   [NPM sur Docker Hub](https://hub.docker.com/r/jc21/nginx-proxy-manager)

# Docker Compose

-   Nous allons utiliser un docker compose avec base de données [MariaDB](https://mariadb.org)…

```plaintext
version: '3.8'
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: always
    ports:
      # These ports are in format <host-port>:<container-port>
      - '80:80' # Public HTTP Port
      - '443:443' # Public HTTPS Port
      - '81:81' # Admin Web Port
      # Add any other Stream port you want to expose
      # - '21:21' # FTP
    environment:
      # Mysql/Maria connection parameters:
      DB_MYSQL_HOST: "db"
      DB_MYSQL_PORT: 3306
      DB_MYSQL_USER: "npm"
      DB_MYSQL_PASSWORD: "npm"
      DB_MYSQL_NAME: "npm"
      # Uncomment this if IPv6 is not enabled on your host
      # DISABLE_IPV6: 'true'
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
    depends_on:
      - db

  db:
    image: 'jc21/mariadb-aria:latest'
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: 'npm'
      MYSQL_DATABASE: 'npm'
      MYSQL_USER: 'npm'
      MYSQL_PASSWORD: 'npm'
    volumes:
      - ./mysql:/var/lib/mysql
```

## Variables à personnaliser

-   Vous pouvez personnaliser ces variables :

`DB_MYSQL_USER: "npm"`

`DB_MYSQL_PASSWORD: "npm"`

`MYSQL_USER: 'npm'`

`MYSQL_PASSWORD: 'npm'`

**Attention de faire correspondre les variables ensemble !**

## Exécution initiale​

-   Après la première exécution de l'application, les événements suivants se produisent :

1.  Les clés JWT (JSON Web Token) seront générées et enregistrées dans le dossier de données.
2.  La base de données s'initialisera.
3.  Un utilisateur administrateur par défaut sera créé.

Ce processus peut prendre quelques minutes selon votre machine.

## Utilisateur administrateur par défaut​

-   Email : admin@example.com
-   Password : changeme

## Captures

![](/docker-compose-nginx-proxy-manager/npm-login.png)

Nginx Proxy Manager - Login

![](/docker-compose-nginx-proxy-manager/npm-dashboard.png)

Nginx Proxy Manager - Dashboard

![](/docker-compose-nginx-proxy-manager/npm-proxy-hosts.png)

Nginx Proxy Manager - Proxy Hosts

## Bonus

-   Le Dockerfile qui construit ce projet n'inclut pas de HEALTHCHECK (vérification de l'état de santé du conteneur) mais vous pouvez opter pour cette fonctionnalité en ajoutant ce qui suit au service dans votre fichier.yml docker-compose :

```plaintext
healthcheck:
  test: ["CMD", "/usr/bin/check-health"]
  interval: 10s
  timeout: 3s
```

-   Vous pouvez configurer la valeur de l'en-tête X-FRAME-OPTIONS en la spécifiant comme variable d'environnement Docker. La valeur par défaut, si elle n'est pas spécifiée, est DENY (refuser) :

```plaintext
  ...
  environment:
    X_FRAME_OPTIONS: "sameorigin"
  ...
```

-   Par défaut, NPM effectue une rotation hebdomadaire des journaux d'accès et d'erreurs, et conserve respectivement 4 et 10 fichiers journaux. Selon l'utilisation, cela peut conduire à des fichiers journaux volumineux, en particulier des journaux d'accès. Vous pouvez personnaliser la configuration de logrotate via un montage d'un volume. Si votre configuration personnalisée est logrotate.custom, voici ce que ça donne :

```plaintext
  volumes:
    ...
    - ./logrotate.custom:/etc/logrotate.d/nginx-proxy-manager
```

**Pour référence, la configuration par défaut est :**

```plaintext
/data/logs/*_access.log /data/logs/*/access.log {
    su npm npm
    create 0644
    weekly
    rotate 4
    missingok
    notifempty
    compress
    sharedscripts
    postrotate
    kill -USR1 `cat /run/nginx/nginx.pid 2>/dev/null` 2>/dev/null || true
    endscript
}

/data/logs/*_error.log /data/logs/*/error.log {
    su npm npm
    create 0644
    weekly
    rotate 10
    missingok
    notifempty
    compress
    sharedscripts
    postrotate
    kill -USR1 `cat /run/nginx/nginx.pid 2>/dev/null` 2>/dev/null || true
    endscript
}
```

**Cette configuration est disponible** [**ICI**](https://github.com/NginxProxyManager/nginx-proxy-manager/blob/develop/docker/rootfs/etc/logrotate.d/nginx-proxy-manager)**.**

-   Pour activer le module [geoip2](https://docs.nginx.com/nginx/admin-guide/dynamic-modules/geoip2/) (géolocalisation basée sur l'adresse IP), vous pouvez créer (dans le conteneur) le fichier de configuration personnalisé /data/nginx/custom/root\_top.conf et inclure l'extrait suivant :

```plaintext
load_module /usr/lib/nginx/modules/ngx_http_geoip2_module.so;
load_module /usr/lib/nginx/modules/ngx_stream_geoip2_module.so;
```

## Mon fichier.yml docker-compose personnel

-   Compose valable jusqu'à la version 1.12.5, avec peut-être un problème de lenteur au démarrage de l'application avec la version 1.12.3 et 1.12.4 ! Voici à quoi ressemble mon fichier.yml docker-compose en date du 06-02-2025 sur ma nouvelle installation :

```plaintext
services:
  npm:
    image: 'jc21/nginx-proxy-manager:latest'
    container_name: npm
    restart: always
    ports:
      # These ports are in format <host-port>:<container-port>
      - '80:80' # Public HTTP Port
      - '443:443' # Public HTTPS Port
      - '81:81' # Admin Web Port
      # Add any other Stream port you want to expose
      # - '21:21' # FTP

    environment:
      # Uncomment this if you want to change the location of
      # the SQLite DB file within the container
      # DB_SQLITE_FILE: "/data/database.sqlite"

      # Uncomment this if IPv6 is not enabled on your host
      # DISABLE_IPV6: 'true'

      # X-FRAME-OPTIONS Header
      X_FRAME_OPTIONS: "sameorigin"

    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
      - ./logrotate.custom:/etc/logrotate.d/nginx-proxy-manager

    healthcheck:
      test: ["CMD", "/usr/bin/check-health"]
      interval: 10s
      timeout: 3s
```
-   Compose à utiliser à partir de la version 1.12.5 ; Ce dernier propose deux nouvelles variables qui vont corriger le problème de lenteur au démarrage de l'application ;
```plaintext
services:
  npm:
    image: jc21/nginx-proxy-manager:2.12.6
    container_name: npm
    restart: always
    ports:
      - 80:80
      - 443:443
      - 81:81
    environment:
      IP_RANGES_FETCH_ENABLED: false
      SKIP_CERTBOT_OWNERSHIP: true
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
      - ./logrotate.custom:/etc/logrotate.d/nginx-proxy-manager
    healthcheck:
      test:
        - CMD
        - /usr/bin/check-health
      interval: 10s
      timeout: 3s
```
Fichier compose également disponible sur [ByteStash Blabla Linux](https://bytestash.blablalinux.be/public/snippets).
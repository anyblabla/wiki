---
title: Docker Compose GoAccess pour NPM
description: GoAccess est une application d'analyse Web open source pour les systèmes d'exploitation de type Unix.
published: true
date: 2025-07-17T00:49:47.408Z
tags: nginx, monitoring, analytic, real-time, npm
editor: markdown
dateCreated: 2025-02-06T12:30:45.317Z
---

Pratique pour déployer rapidement [GoAcces](https://goaccess.io) [pour Nginx Proxy Manager](https://github.com/xavier-hernandez/goaccess-for-nginxproxymanager) (NPM) dans [Portainer](https://www.portainer.io/) en créant une pile ([stack](https://docs.portainer.io/user/docker/stacks)) à partir d'un fichier [compose YAML](https://docs.docker.com/compose/compose-application-model/). Ou avec Docker seul, sans Portainer 😊

Je pars du principe que vous maîtrisez un minimum Docker avec Portainer 😉

## GoAcces, qu'est-ce que c'est ?

GoAcces va vous permettre de visualiser en temps réel, sous forme de tableaux, l'ensemble des logs (fichiers journaux) des hôtes proxy de votre proxy inverse Nginx Proxy Manager.

<p style="text-align: center"><img src="/docker-compose-goaccess/goaccess-bright.png"></p>


## Liens utiles

-   [Site officiel](https://goaccess.io)
-   [Live Demo](http://rt.goaccess.io/?20250113204951)
-   [GitHub](https://github.com/allinurl/goaccess)
-   [Pour Nginx Proxy Manager](https://github.com/xavier-hernandez/goaccess-for-nginxproxymanager)

## fichier.yml docker-compose

-   Partons du principe que l'on déploie avec Docker, nous créons un répertoire “goaccess” :

```plaintext
mkdir goaccess
```

-   On se place dedans :

```plaintext
cd goaccess
```

-   On crée notre fichier docker-compose.yml :

```plaintext
nano docker-compose.yml 
```

-   On colle ce qui suit (mon fichier .yml docker-compose personnel - fichier compose également disponible sur [ByteStash Blabla Linux](https://bytestash.blablalinux.be/public/snippets)) :

```plaintext
version: '3.8'
services:
  goaccess:
    image: 'xavierh/goaccess-for-nginxproxymanager:latest'
    container_name: goaccess
    restart: always
    ports:
      - '7880:7880'
    environment:
      - TZ=Europe/Brussels #optional
      - LANG=fr_FR.UTF-8 #optional
      - LANGUAGE=fr_FR.UTF-8 #optional
      - SKIP_ARCHIVED_LOGS=False #optional
      - DEBUG=False #optional
      - BASIC_AUTH=False #optional
      - BASIC_AUTH_USERNAME=user #optional
      - BASIC_AUTH_PASSWORD=pass #optional   
      - EXCLUDE_IPS=127.0.0.1 #optional - comma delimited 
      - LOG_TYPE=NPM+R #optional
      - ENABLE_BROWSERS_LIST=True #optional
      - CUSTOM_BROWSERS=Kuma:Uptime,TestBrowser:Crawler #optional - comma delimited
      - HTML_REFRESH=5 #optional - Refresh the HTML report every X seconds. https://goaccess.io/man
      - KEEP_LAST=30 #optional - Keep the last specified number of days in storage. https://goaccess.io/man
      - PROCESSING_THREADS=1 #optional - This parameter sets the number of concurrent processing threads in the program's execution, affecting log data analysis, typically adjusted based on CPU cores. Default is 1. https://goaccess.io/man
    volumes:
      - /data/compose/1/data/logs:/opt/log
      #- /path/to/host/custom:/opt/custom #optional, required if using log_type = CUSTOM
```

-   Lignes à modifier :

`- TZ=Europe/Brussels #optional`

`- /data/compose/1/data/logs:/opt/log`

**Cette dernière est la plus importante ! C'est l'emplacement où se situent les fichiers journaux de Nginx Proxy Manager.**

Les plus habitués d'entre vous auront remarqué ici l'utilisation de Portainer pour déployer GoAcces ! D'où le début de la ligne…

`/data/compose/1/`

Si vous ajoutez GoAcces au compose de Nginx Proxy Manager, et que vous n'avez pas modifié les volumes de ce dernier, alors la ligne sera…

`- ./data/logs:/opt/log`

## Autres lignes

-   On peut voir que j'ai passé mon instance GoAccess en français :

`- LANG=fr_FR.UTF-8 #optional`

`- LANGUAGE=fr_FR.UTF-8 #optional`

-   On peut voir que je demande à GoAccess de ne pas prendre en compte les journaux archives (.gz) :

`- SKIP_ARCHIVED_LOGS=False #optional`

-   On peut voir que mon instance ne demande pas d'authentification :

`- BASIC_AUTH=False #optional`

`- BASIC_AUTH_USERNAME=user #optional`

`- BASIC_AUTH_PASSWORD=pass #optional`

-   Si vous désirez devoir vous authentifier :

`- BASIC_AUTH=True #optional`

`- BASIC_AUTH_USERNAME=anyblabla #optional`

`- BASIC_AUTH_PASSWORD=blablalinux #optional`

-   On peut voir que la machine hôte est exclue :

`- EXCLUDE_IPS=127.0.0.1 #optional - comma delimited`

Vous pouvez ajouter plusieurs adresses IP séparées par une virgule.

-   Cette ligne est importante. Par défaut, la configuration sera définie pour lire les journaux NPM. Les options possibles sont : CUSTOM, NPM, NPM+R, NPM+ALL, TRAEFIK, NCSA\_COMBINED, CADDY\_V1.

`- LOG_TYPE=NPM+R #optional`

⇒ NPM, le ou les fichiers suivants sont lus et analysés :

⇢ proxy-host-\*\_access.log.gz (si vous avez demandé la prise en compte des archives !)

⇢ proxy-host-\*\_access.log

⇢ proxy\*host-\*.log

⇒ NPM+R, une deuxième instance de GoAccess est créée ; ajouter « /redirection » à l'URL pour accéder à l'instance, par exemple _http://localhost:7880/redirection/ ;_  le ou les fichiers suivants sont lus et analysés :

⇢ redirection\*host-\*.log\*.gz (si vous avez demandé la prise en compte des archives !)

⇢ redirection\*host-\*.log

⇢ fallback\_access.log\*.gz (si vous avez demandé la prise en compte des archives !)

⇢ fallback\_access.log

⇢ dead-host\*.log\*.gz (si vous avez demandé la prise en compte des archives !)

⇢ dead-host\*.log

⇒ NPM+ALL, une deuxième et une troisième instance de GoAccess sont créées ; ajouter « /redirection » à l'URL pour accéder à l'instance, par exemple http://localhost:7880/redirection/ ; le ou les fichiers suivants sont lus et analysés :

⇢ redirection\*host-*.log*.gz (si vous avez demandé la prise en compte des archives !)

⇢ redirection\*host-\*.log

**OU**

Ajouter « /error » à l'URL pour accéder à l'instance, par exemple http://localhost:7880/error/ ; le ou les fichiers suivants sont lus et analysés :

⇢ \*\_error.log\*.gz (si vous avez demandé la prise en compte des archives !)

⇢ \*\_error.log

-   On peut voir que la page HTML sera rafraîchie toutes les cinq secondes :

`- HTML_REFRESH=5 #optional`

-   On peut voir que les statistiques sur trente jours seront prises en compte :

`- KEEP_LAST=30 #optional`

-   **Une fois que tout est réglé, on peut lancer notre conteneur :**

```plaintext
docker compose up -d
```

## Remarque

Je n'ai pas tout détaillé ! Je vous conseille de vous rendre sur [la page de projet GitHub](https://github.com/xavier-hernandez/goaccess-for-nginxproxymanager), ou encore mieux, [sur la page “man” du site officiel](https://goaccess.io/man).
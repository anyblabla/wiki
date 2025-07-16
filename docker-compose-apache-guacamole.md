---
title: Docker Compose Apache Guacamole
description: Apache Guacamole est une passerelle de bureau à distance sans client. Il prend en charge les protocoles standards tels que VNC, RDP et SSH.
published: true
date: 2025-07-16T22:15:24.626Z
tags: vnc, apache, remote, guacamole, ssh, rdp, kubernetes
editor: markdown
dateCreated: 2024-07-10T12:42:55.354Z
---

Pratique pour déployer rapidement [Apache Guacamole](https://guacamole.apache.org) dans [Portainer](https://www.portainer.io/) en créant une pile ([stack](https://docs.portainer.io/user/docker/stacks)) à partir d'un fichier [compose YAML](https://docs.docker.com/compose/compose-application-model/).

Je pars du principe que vous maîtrisez un minimum Docker avec Portainer 😉

# Apache Guacamole, c'est quoi ?

*Apache Guacamole est une passerelle de* [bureau à distance](https://w.wiki/Acop) *sans client.*

*Il prend en charge les protocoles standards tels que* [VNC](https://w.wiki/Acot)*,* [RDP](https://w.wiki/Acou) *et* [SSH](https://w.wiki/Acov)*.*

*Nous l'appelons sans client, car aucun* [plugin](https://w.wiki/Acox) *ou logiciel client n'est requis.*

*Grâce au* [HTML5](https://w.wiki/9mA4)*, une fois Guacamole installé sur un serveur, tout ce dont vous avez besoin pour accéder à vos bureaux est un navigateur Web.*

# Liens utiles

-   [Site officiel](https://guacamole.apache.org)
-   [Documentation](https://guacamole.apache.org/doc/gug/)
-   [Code source](https://github.com/search?utf8=%E2%9C%93&q=repo%3Aapache%2Fguacamole-client+repo%3Aapache%2Fguacamole-server+repo%3Aapache%2Fguacamole-manual+repo%3Aapache%2Fguacamole-website&type=repositories&ref=searchresults)

# Installation

Manuelle, ou via Portainer.

“[sudo](https://fr.wikipedia.org/wiki/Sudo)” OU PAS “sudo” ? À vous de savoir. Personnellement, je suis sur un [LXC](https://fr.wikipedia.org/wiki/LXC) [Debian](https://fr.wikipedia.org/wiki/Debian) [Proxmox](https://fr.wikipedia.org/wiki/Proxmox_VE), je travaille donc en “[root](https://fr.wikipedia.org/wiki/Root)”, pas besoin de “sudo” !

## Manuelle

-   Créer un dossier qui va contenir les différents fichiers et dossier pour notre environnement Guacamole…

```plaintext
mkdir guacamole
```

-   Dans le dossier “guacamole”, créer le fichier “docker-compose.yml”…

```plaintext
touch docker-compose-yml
```

-   Ouvrez maintenant “docker-compose.yml” pour l'éditer…

```plaintext
nano docker-compose.yml
```

-   Voici le contenu du fichier “docker-compose.yml”, adaptez-le à votre environnement…

```plaintext
version: '3.8'

services:
    guacamole_db:
        container_name: guacamole_db
        hostname: guacamole_db
        image: mariadb:10.11
        restart: always
        volumes:
            - ./guacamole_db:/var/lib/mysql
        environment:
            - MYSQL_ROOT_PASSWORD=blablalinux
            - MYSQL_DATABASE=guacamole_db
            - MYSQL_USER=anyblabla
            - MYSQL_PASSWORD=blabla
        expose:
            - 3306
    
    guacd:
        container_name: guacd
        hostname: guacd
        image: guacamole/guacd:latest
        restart: always
        volumes:
            - ./guacd_drive:/drive:rw 
            - ./guacd_record:/record:rw 
        expose:
            - 4822

    guacamole:
        container_name: guacamole
        hostname: guacamole
        restart: always
        image: guacamole/guacamole:latest
        depends_on:
            - guacamole_db
            - guacd
        ports:
            - 8080:8080
        links:
            - guacd
        environment:
            - GUACD_HOSTNAME=guacd
            - MYSQL_HOSTNAME=guacamole_db
            - MYSQL_DATABASE=guacamole_db
            - MYSQL_USER=anyblabla
            - MYSQL_PASSWORD=blabla
            - REMOTE_IP_VALVE_ENABLED=true
```

Fichier compose également disponible sur [ByteStash Blabla Linux](https://bytestash.blablalinux.be/public/snippets).

### Compose personnalisation

Dans “environment” du service “guacamole\_db”…

-   N'oubliez pas de personnaliser le mot de passe “root” pour MySQL…

`- MYSQL_ROOT_PASSWORD=blablalinux`

-   N'oubliez pas de personnaliser le nom utilisateur pour MySQL…

`- MYSQL_USER=anyblabla`

-   N'oubliez pas de personnaliser le mot de passe pour MySQL…

`- MYSQL_PASSWORD=blabla`

**Les informations “environement” du service “guacamole\_db” doivent être identiques que les informations “environment” du service “guacamole” !**

-   La variable “- REMOTE\_IP\_VALVE\_ENABLED=” est à activer si vous utilisez un [Proxy inverse](https://w.wiki/Acu3)…

`- REMOTE_IP_VALVE_ENABLED=true`

### Bonus Compose personnalisation

-   Pour activer la [double authentification](https://w.wiki/Acu6), il suffit d'ajouter cette variable en dessous de la variable “- REMOTE\_IP\_VALVE\_ENABLED=true”…

`- TOTP_ENABLED=true`

## Portainer

-   Il suffit de créer une pile stack avec le nom de votre choix, ici, “guacamole”, et de coller le contenu du fichier Compose ci-dessus…

![](/docker-compose-apache-guacamole/guacamole-stack-portainer.jpg)

---

Vous pouvez lancer le container !

-   Avec une installation manuelle, simplement être dans le répertoire “guacamole” et…

```plaintext
docker-compose up -d
```

-   Avec une pile stack Portainer, un clic sur “Deploy the stack”…

![](/docker-compose-apache-guacamole/deploy-the-stack.jpg)

L'identifiant et le mot de passe par défaut est : **guacadmin**

---

## Manuelle/Portainer - Instructions communes

Il faut maintenant initialiser la base de données MySQL.

-   Si vous n'êtes pas en “root”, passez-y…

```plaintext
sudo su
```

-   Récupérer le script d’initialisation de la base de données MySQL…

```plaintext
docker run --rm guacamole/guacamole /opt/guacamole/bin/initdb.sh --mysql > initdb.sql
```

-   Injecter le fichier de la base de données MySQL…

```plaintext
docker exec -i guacamole_db mysql --user anyblabla --password=blabla guacamole_db < initdb.sql
```

-   La commande doit être adaptée à votre environnement…

`--user anyblabla --password=blabla`

# Guacamole en fonctionnement

Je vous propose cette vidéo pour vous rendre compte du résultat…

-   [Facebook](https://www.facebook.com/blablalinux/videos/320245721056954/)
-   [X (Twitter)](https://x.com/BlablaLinux/status/1810306929278832882)
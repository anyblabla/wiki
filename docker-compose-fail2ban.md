---
title: Docker Compose Fail2ban pour NPM
description: Fail2ban est un framework de prévention contre les intrusions, écrit en Python. Il fonctionne sur les systèmes POSIX possédant une interface de contrôle des paquets (tel que TCP Wrapper) ou un pare-feu (tel que Netfilter).
published: true
date: 2025-04-19T22:48:58.646Z
tags: docker, nginx, npm, ban, fail, fail2ban
editor: markdown
dateCreated: 2025-02-06T21:58:30.959Z
---

Pratique pour déployer rapidement [Fail2ban](https://github.com/fail2ban/fail2ban) [pour Nginx Proxy Manager](https://github.com/xavier-hernandez/goaccess-for-nginxproxymanager) (NPM) dans [Portainer](https://www.portainer.io/) en créant une pile ([stack](https://docs.portainer.io/user/docker/stacks)) à partir d'un fichier [compose YAML](https://docs.docker.com/compose/compose-application-model/). Ou avec Docker seul, sans Portainer 😊

Je pars du principe que vous maîtrisez un minimum Docker avec Portainer 😉

## Fail2ban, qu'est-ce que c'est ?

Fail2ban est un service analysant en temps réel les journaux d'évènement de divers services (SSH, Apache, FTP, entre autres) à la recherche de comportements malveillants et permet d'exécuter une ou plusieurs actions lorsqu'un évènement malveillant est détecté.

La suite, c'est [ICI](https://fr.wikipedia.org/wiki/Fail2ban).

![](/docker-compose-fail2ban/fail2ban-02.png)

## Liens utiles

-   [Site officiel](https://github.com/fail2ban/fail2ban)
-   [Wikipédia](https://fr.wikipedia.org/wiki/Fail2ban)
-   [Docker Linuxserver.io sur DockerHub](https://hub.docker.com/r/linuxserver/fail2ban)
-   [Docker par CrazyMax sur GitHub](https://github.com/crazy-max/docker-fail2ban) (**image utilisée pour la suite**)

## fichier.yml docker-compose

-   Partons du principe que l'on déploie avec Docker, nous créons un répertoire “fail2ban” :

```plaintext
mkdir fail2ban
```

-   On se place dedans :

```plaintext
cd fail2ban
```

-   On crée notre fichier docker-compose.yml :

```plaintext
nano docker-compose.yml
```

-   On colle ce qui suit (mon fichier .yml docker-compose personnel) :

```plaintext
version: "3.7"
services:
  fail2ban:
    image: crazymax/fail2ban:latest
    container_name: fail2ban
    network_mode: host
    cap_add:
      - NET_ADMIN
      - NET_RAW
    volumes:
      - ./data:/data
      - /data/compose/1/data/logs:/var/log:ro
    environment:
      - TZ=Europe/Brussels
      - F2B_LOG_TARGET=STDOUT
      - F2B_LOG_LEVEL=INFO
      - F2B_DB_PURGE_AGE=28d
    restart: always
```

-   Lignes à modifier :

`- /data/compose/1/data/logs:/var/log:ro`

**Cette dernière est la plus importante ! C'est l'emplacement où se situent les fichiers journaux de Nginx Proxy Manager.**

Les plus habitués d'entre vous auront remarqué ici l'utilisation de Portainer pour déployer GoAcces ! D'où le début de la ligne…

`/data/compose/1/`

Si vous ajoutez Fail2ban au compose de Nginx Proxy Manager, et que vous n'avez pas modifié les volumes de ce dernier, alors la ligne sera…

`- ./data/logs:/opt/log`

## Autres lignes

-   On peut voir que j'utilise la Time Zone Europe/Brussels :

`- TZ=Europe/Brussels`

-   On peut voir que je demande tous les 28 jours, une purge de la base de données :

`- F2B_DB_PURGE_AGE=28d`

Par défaut, cette variable est à 1 !

-   J'attire votre attention sur cette ligne :

`- ./data:/data`

C'est à cet endroit que vous allez trouver les répertoires de travail “action.d”, “filter.d” et “jail.d”.

## “action.d”, “filter.d” et “jail.d”

-   Dans le répertoire “action.d”, vous allez créer un fichier “docker-action.conf”, et coller ce qui suit :

```plaintext
[Definition]

actionstart = iptables -N f2b-npm-docker
              iptables -A f2b-npm-docker -j RETURN
              iptables -I FORWARD -p tcp -m multiport --dports 0:65535 -j f2b-npm-docker

actionstop = iptables -D FORWARD -p tcp -m multiport --dports 0:65535 -j f2b-npm-docker
             iptables -F f2b-npm-docker
             iptables -X f2b-npm-docker

actioncheck = iptables -n -L FORWARD | grep -q 'f2b-npm-docker[ \t]'

actionban = iptables -I f2b-npm-docker -s <ip> -j DROP

actionunban = iptables -D f2b-npm-docker -s <ip> -j DROP
```

Une action est tout simplement la ligne de commande, outil ou script qui va être exécuté lorsque les événements le voudront. Typiquement, Fail2ban exécutera une action lorsqu'une ligne de log détectée par nos filtres apparaîtra un certain nombre de fois. Les actions peuvent alors être des événements de prévention ou de protections (bannir l'IP en question, redémarrer le service XY, vider le cache de telle application…) ou alors des événements d'alerte (on va envoyer un mail à l'administrateur système, on va générer une alerte dans le système de supervision…).

-   Dans le répertoire “filter.d”, vous allez créer un fichier “npm.conf”, et coller ce qui suit :

```plaintext
[INCLUDES]

[Definition]

#failregex = ^\s*(?:\[\] )?\S+ \S+ (?:[4-5]0\d) - \S+ \S+ \S+ "[^"]*" \[Client <ADDR>\] (?:\[\w+ [^\]]*\] )*\[Sent-to <F-CONTAINER>[^\]]*</F-CONTAINER>\] "<F-USERAGENT>[^"]*</F-USERAGENT>"

failregex = ^\s*(?:\[\] )?\S+ \S+ (?:(?P<nf>404)|[4-5]0(?!4)\d|504) - \S+ \S+ \S+ "(?(nf)/(?:[^/]*/)*[^\.]*(?!\.(?:png|txt|jpg|ico|js|css|ttf|woff|woff2)[^\."]*)|[^"]*)" \[Client <ADDR>\] (?:\[\w+ [^\]]*\] )*\[Sent-to <F-CONTAINER>[^\]]*</F-CONTAINER>\] "<F-USERAGENT>[^"]*</F-USERAGENT>"
```

Un filtre dans Fail2ban est une expression régulière qui va être utilisée pour scanner les fichiers de logs ciblés dans nos jails. On applique un filtre sur les nouvelles entrées dans un fichier de logs afin de voir si celles-ci contiennent des informations relevant d'une intrusion ou non. Un filtre n'est rien d'autre que le moyen de détection dont se sert Fail2ban.

Je dois dire que je suis particulièrement fier de ces deux regex, car ils ont été écrits spécialement pour notre configuration, par une personne du projet Fail2ban, rien que ça 🤙

Le premier regex (commenté), va s'occuper des erreurs type 40x et 50x (côté client).

Le premier regex (commenté), va s'occuper des erreurs type 40x et 50x (côté client), mais avec une exception de laissé passé concernant les erreurs type 404 et 504 (côté client) qui pointe sur du png, txt, jpg, ico, js, css, ttf, woff et woff2.

-   Dans le répertoire “jail.d”, vous allez créer un fichier “npm.local”, et coller ce qui suit :

```plaintext
[DEFAULT]

# "bantime.increment" allows to use database for searching of previously banned ip's to increase a
# default ban time using special formula, default it is banTime * 1, 2, 4, 8, 16, 32...
bantime.increment = true


# "bantime.rndtime" is the max number of seconds using for mixing with random time
# to prevent "clever" botnets calculate exact time IP can be unbanned again:
bantime.rndtime = 1200


# "bantime.maxtime" is the max number of seconds using the ban time can reach (don't grows further)
#bantime.maxtime =


# "bantime.factor" is a coefficient to calculate exponent growing of the formula or common multiplier,
# default value of factor is 1 and with default value of formula, the ban time
# grows by 1, 2, 4, 8, 16 ...
bantime.factor = 1


# "bantime.formula" used by default to calculate next value of ban time, default value bellow,
# the same ban time growing will be reached by multipliers 1, 2, 4, 8, 16, 32...
bantime.formula = ban.Time * (1<<(ban.Count if ban.Count<20 else 20)) * banFactor

# more aggressive example of formula has the same values only for factor "2.0 / 2.885385" :
#bantime.formula = ban.Time * math.exp(float(ban.Count+1)*banFactor)/math.exp(1*banFactor)


# "bantime.multipliers" used to calculate next value of ban time instead of formula, coresponding
# previously ban count and given "bantime.factor" (for multipliers default is 1);
# following example grows ban time by 1, 2, 4, 8, 16 ... and if last ban count greater as multipliers count,
# always used last multiplier (64 in example), for factor '1' and original ban time 600 - 10.6 hours
#bantime.multipliers = 1 2 4 8 16 32 64

# following example can be used for small initial ban time (bantime=60) - it grows more aggressive at begin,
# for bantime=60 the multipliers are minutes and equal: 1 min, 5 min, 30 min, 1 hour, 5 hour, 12 hour, 1 day, 2 day
#bantime.multipliers = 1 5 30 60 300 720 1440 2880


[npm]
enabled = true
ignoreself = true
ignoreip =  127.0.0.1/8 ::1 # A good idea is to mention your LAN IP address range (example: 192.168.2.0/24) as well as your WAN IP.
chain = DOCKER-USER
logpath = /var/log/proxy-host-*_access.log
maxretry = 5
findtime = 1h
bantime  = 10h
action = docker-action
```

Jail pour prison est notre fichier de configuration principal. La section \[DEFAULT\] va appliquer et/ou outrepasser la configuration par défaut de Fail2ban. 

La section \[npm\] ne s'applique qu'à NPM.

⇒ J'attire votre attention sur cette ligne :

⇢ `ignoreip =  127.0.0.1/8 ::1`

Vous pouvez ajouter votre plage d'adresses IP LAN (exemple : 192.168.2.0/24) ainsi que votre adresse IP WAN pour ne pas vous bannir vous-même 🤪

Les “attaquants” aurons droit à cinq tentatives sur une durée d'une heure, avant de se faire bannir pour une durée de dix heures ! Après dé-bannissement, s'il y a récidive, le prochain bannissement sera durci avec incrémentation !

⇒ D'où la ligne :

⇢ `bantime.increment = true`

Cette dernière va prendre en compte `bantime.factor = 1` et `bantime.formula = ban.Time * (1<<(ban.Count if ban.Count<20 else 20)) * banFactor`.

Ce qui veut dire que si une IP incriminée est trouvée pour la 4ème fois, Fail2Ban va augmenter le temps d'interdiction pour cette IP d'une heure à seize heures ! 

⇒ Ligne bonus :

⇢ `bantime.rndtime = 1200`

"bantime.rndtime" est le nombre maximal de secondes à utiliser pour le mélange avec un temps aléatoire afin d'empêcher les botnets "intelligents" de calculer le moment exact auquel l'IP peut être à nouveau débloquée 😉

-   Il est plus que temps de créer et de lancer notre conteneur :

```plaintext
docker compose up -d
```

## Commandes utiles

Je vais vous donner quelques commandes sympas à utiliser directement via le conteneur.

-   Status Jail :

```plaintext
docker exec -t fail2ban fail2ban-client status
```

-   Status Jail NPM :

```plaintext
docker exec -t fail2ban fail2ban-client status npm 
```

-   Effectuer une recherche dans les fichiers journaux Nginx Proxy Manager, sur les requêtes qu'a effectuées une adresse IP bannie :

`grep <IP-ADRESS> /EMPLACEMENT/RĖPERTOIRE/LOGS/proxy-host-*_access.log`

⇒ Ce qui donne chez moi avec un exemple :

```plaintext
grep 20.171.207.160 /data/compose/1/data/logs/proxy-host-*_access.log
```

-   Ban list :

```plaintext
docker exec -t fail2ban fail2ban-client banned
```

-   Ban IP :

```plaintext
docker exec -t fail2ban fail2ban-client set npm banip <IP>
```

-   UnBan IP :

```plaintext
docker exec -t fail2ban fail2ban-client set npm unbanip <IP>
```

-   Fail2Ban version :

```plaintext
docker exec -t fail2ban fail2ban-client version
```

-   Et encore plein d'autre en vous laissant découvrir l'aide :

```plaintext
docker exec -t fail2ban fail2ban-client --help
```

## Bonus jail.d / npm.com

Je vais vous fournir deux autres fichiers de configuration npm.conf pour le répertoire jail.d 👋

Ils sont similaires au premier ci-dessus. Ce sont différents réglages concernant l'incrémentation des bannissements. Des lignes dé commentées vont être commentées, et inversement.

### bantime.multipliers à la place de bantime.formula

```plaintext
[DEFAULT]

# "bantime.increment" allows to use database for searching of previously banned ip's to increase a
# default ban time using special formula, default it is banTime * 1, 2, 4, 8, 16, 32...
bantime.increment = true


# "bantime.rndtime" is the max number of seconds using for mixing with random time
# to prevent "clever" botnets calculate exact time IP can be unbanned again:
bantime.rndtime = 1200


# "bantime.maxtime" is the max number of seconds using the ban time can reach (don't grows further)
#bantime.maxtime =


# "bantime.factor" is a coefficient to calculate exponent growing of the formula or common multiplier,
# default value of factor is 1 and with default value of formula, the ban time
# grows by 1, 2, 4, 8, 16 ...
#bantime.factor = 1


# "bantime.formula" used by default to calculate next value of ban time, default value bellow,
# the same ban time growing will be reached by multipliers 1, 2, 4, 8, 16, 32...
#bantime.formula = ban.Time * (1<<(ban.Count if ban.Count<20 else 20)) * banFactor

# more aggressive example of formula has the same values only for factor "2.0 / 2.885385" :
#bantime.formula = ban.Time * math.exp(float(ban.Count+1)*banFactor)/math.exp(1*banFactor)


# "bantime.multipliers" used to calculate next value of ban time instead of formula, coresponding
# previously ban count and given "bantime.factor" (for multipliers default is 1);
# following example grows ban time by 1, 2, 4, 8, 16 ... and if last ban count greater as multipliers count,
# always used last multiplier (64 in example), for factor '1' and original ban time 600 - 10.6 hours
#bantime.multipliers = 1 2 4 8 16 32 64

# following example can be used for small initial ban time (bantime=60) - it grows more aggressive at begin,
# for bantime=60 the multipliers are minutes and equal: 1 min, 5 min, 30 min, 1 hour, 5 hour, 12 hour, 1 day, 2 day
bantime.multipliers = 1 5 30 60 300 720 1440 2880


[npm]
enabled = true
ignoreself = true
ignoreip =  127.0.0.1/8 ::1 # A good idea is to mention your LAN IP address range (example: 192.168.2.0/24) as well as your WAN IP.
chain = DOCKER-USER
logpath = /var/log/proxy-host-*_access.log
maxretry = 5
findtime = 1h
bantime  = 1h
action = docker-action
```

### bantime.multipliers (2) à la place de bantime.formula

```plaintext
[DEFAULT]

# "bantime.increment" allows to use database for searching of previously banned ip's to increase a
# default ban time using special formula, default it is banTime * 1, 2, 4, 8, 16, 32...
bantime.increment = true


# "bantime.rndtime" is the max number of seconds using for mixing with random time
# to prevent "clever" botnets calculate exact time IP can be unbanned again:
bantime.rndtime = 1200


# "bantime.maxtime" is the max number of seconds using the ban time can reach (don't grows further)
#bantime.maxtime =


# "bantime.factor" is a coefficient to calculate exponent growing of the formula or common multiplier,
# default value of factor is 1 and with default value of formula, the ban time
# grows by 1, 2, 4, 8, 16 ...
#bantime.factor = 1


# "bantime.formula" used by default to calculate next value of ban time, default value bellow,
# the same ban time growing will be reached by multipliers 1, 2, 4, 8, 16, 32...
#bantime.formula = ban.Time * (1<<(ban.Count if ban.Count<20 else 20)) * banFactor

# more aggressive example of formula has the same values only for factor "2.0 / 2.885385" :
#bantime.formula = ban.Time * math.exp(float(ban.Count+1)*banFactor)/math.exp(1*banFactor)


# "bantime.multipliers" used to calculate next value of ban time instead of formula, coresponding
# previously ban count and given "bantime.factor" (for multipliers default is 1);
# following example grows ban time by 1, 2, 4, 8, 16 ... and if last ban count greater as multipliers count,
# always used last multiplier (64 in example), for factor '1' and original ban time 600 - 10.6 hours
bantime.multipliers = 1 2 4 8 16 32 64

# following example can be used for small initial ban time (bantime=60) - it grows more aggressive at begin,
# for bantime=60 the multipliers are minutes and equal: 1 min, 5 min, 30 min, 1 hour, 5 hour, 12 hour, 1 day, 2 day
#bantime.multipliers = 1 5 30 60 300 720 1440 2880


[npm]
enabled = true
ignoreself = true
ignoreip =  127.0.0.1/8 ::1 # A good idea is to mention your LAN IP address range (example: 192.168.2.0/24) as well as your WAN IP.
chain = DOCKER-USER
logpath = /var/log/proxy-host-*_access.log
maxretry = 5
findtime = 1h
bantime  = 10h
action = docker-action
```

### Pas de bantime.increment

```plaintext
[DEFAULT]

# "bantime.increment" allows to use database for searching of previously banned ip's to increase a
# default ban time using special formula, default it is banTime * 1, 2, 4, 8, 16, 32...
#bantime.increment = true


# "bantime.rndtime" is the max number of seconds using for mixing with random time
# to prevent "clever" botnets calculate exact time IP can be unbanned again:
bantime.rndtime = 1200


# "bantime.maxtime" is the max number of seconds using the ban time can reach (don't grows further)
#bantime.maxtime =


# "bantime.factor" is a coefficient to calculate exponent growing of the formula or common multiplier,
# default value of factor is 1 and with default value of formula, the ban time
# grows by 1, 2, 4, 8, 16 ...
#bantime.factor = 1


# "bantime.formula" used by default to calculate next value of ban time, default value bellow,
# the same ban time growing will be reached by multipliers 1, 2, 4, 8, 16, 32...
#bantime.formula = ban.Time * (1<<(ban.Count if ban.Count<20 else 20)) * banFactor

# more aggressive example of formula has the same values only for factor "2.0 / 2.885385" :
#bantime.formula = ban.Time * math.exp(float(ban.Count+1)*banFactor)/math.exp(1*banFactor)


# "bantime.multipliers" used to calculate next value of ban time instead of formula, coresponding
# previously ban count and given "bantime.factor" (for multipliers default is 1);
# following example grows ban time by 1, 2, 4, 8, 16 ... and if last ban count greater as multipliers count,
# always used last multiplier (64 in example), for factor '1' and original ban time 600 - 10.6 hours
#bantime.multipliers = 1 2 4 8 16 32 64

# following example can be used for small initial ban time (bantime=60) - it grows more aggressive at begin,
# for bantime=60 the multipliers are minutes and equal: 1 min, 5 min, 30 min, 1 hour, 5 hour, 12 hour, 1 day, 2 day
#bantime.multipliers = 1 5 30 60 300 720 1440 2880


[npm]
enabled = true
ignoreself = true
ignoreip =  127.0.0.1/8 ::1 # A good idea is to mention your LAN IP address range (example: 192.168.2.0/24) as well as your WAN IP.
chain = DOCKER-USER
logpath = /var/log/proxy-host-*_access.log
maxretry = 5
findtime = 1h
bantime  = 10h
action = docker-action
```

Le fichier compose ainsi que l'ensemble des fichiers de configurations sont également disponibles sur [ByteStash Blabla Linux](https://bytestash.blablalinux.be/public/snippets).
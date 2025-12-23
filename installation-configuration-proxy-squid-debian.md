---
title: Déploiement d'un serveur proxy cache Squid sur Debian
description: Apprenez à déployer Squid sur Debian pour optimiser votre navigation. Ce guide couvre l'installation, la configuration du cache et l'utilisation d'alias GNOME pour un contrôle total de vos flux web.
published: true
date: 2025-12-23T03:23:08.594Z
tags: cache, proxy, debian, squid, administration système
editor: markdown
dateCreated: 2025-12-23T00:16:17.440Z
---

## Qu'est-ce qu'un serveur proxy ?

Un serveur proxy est un intermédiaire entre votre ordinateur (le client) et internet (le serveur distant). Au lieu de se connecter directement à un site web, votre navigateur envoie la requête au proxy, qui va chercher la ressource pour vous et vous la retransmet.

## Pourquoi le choix de Squid ?

Squid est la référence absolue dans le monde du logiciel libre pour les serveurs mandataires. Mes raisons pour ce choix sont simples :

* **Performances :** Une gestion du cache (RAM et disque) robuste.
* **Contrôle :** Une visibilité totale sur les flux sortants via les journaux d'accès.
* **Légèreté :** Consomme très peu de ressources (~50 Mo de RAM).
* **Confidentialité :** Anonymise le trafic local en masquant l'IP réelle des clients.

## Le rôle de Squid en 2025 (L'ère du tout HTTPS)

Pourquoi utiliser un proxy alors que 95% du web est chiffré ? Par sécurité, Squid ne peut pas "voir" ni mettre en cache le contenu des flux HTTPS (tunneling). Cependant, il reste indispensable pour :

1. **L'observabilité :** Squid logue les domaines consultés. C'est votre "boîte noire" réseau.
2. **L'anonymisation :** Il masque l'empreinte de vos navigateurs et l'IP de vos machines.
3. **Le cache résiduel :** Redoutable pour les dépôts `apt` (Debian/Ubuntu) qui utilisent encore le HTTP.

## Installation sur une base Debian

```bash
apt update && apt install squid -y

```

## Paramétrage du fichier de configuration

1. **Sauvegardez l'origine :** `cp /etc/squid/squid.conf /etc/squid/squid.conf.bak`
2. **Videz et éditez :** `true > /etc/squid/squid.conf && nano /etc/squid/squid.conf`

### Configuration optimisée

> **Note :** Pensez à remplacer `192.168.2.0/24` par votre propre plage réseau.
> **Conseil Proxmox :** Pour un container LXC de **8 Go** de disque, nous allons allouer **3 Go** au cache.

```text
# --- 1. ACL ET ACCÈS ---
acl SSL_ports port 443
acl Safe_ports port 80
acl Safe_ports port 443
acl CONNECT method CONNECT

# REMPLACER l'adresse ci-dessous par votre propre plage réseau
acl mon_reseau_local src 192.168.2.0/24

http_access deny !Safe_ports
http_access deny CONNECT !SSL_ports
http_access allow localhost manager
http_access deny manager
http_access allow mon_reseau_local
http_access allow localhost
http_access deny to_localhost
http_access deny all

# --- 2. RÉSEAU ET ANONYMISATION ---
http_port 3128
forwarded_for off
request_header_access Via deny all
request_header_access X-Forwarded-For deny all

# --- 3. GESTION DU CACHE OPTIMISÉE ---
cache_mem 100 MB
maximum_object_size_in_memory 512 KB
cache_dir ufs /var/spool/squid 3000 16 256
maximum_object_size 50 MB
minimum_object_size 0 KB

# --- 4. PERSISTENCE DU CACHE (BOOST) ---
refresh_pattern -i \.(gif|png|jpg|jpeg|ico|svg|webp)$ 10080 90% 43200 override-expire
refresh_pattern -i \.(js|css|woff|woff2|otf|ttf)$ 10080 90% 43200 override-expire

access_log daemon:/var/log/squid/access.log squid

```

## Focus sur l'optimisation du cache

* **`cache_mem 100 MB`** : Alloue 100 Mo de RAM pour les objets récents.
* **`cache_dir ufs ... 3000`** : 3 Go d'espace disque dédié au cache.
* **`maximum_object_size 50 MB`** : On privilégie les petits fichiers web et paquets `.deb` pour ne pas saturer le stockage.
* **`override-expire`** : Force la conservation des images/scripts pour booster la navigation.

## Initialisation et Gestion du service

```bash
systemctl stop squid
squid -z
systemctl enable squid
systemctl start squid

```

## Monitoring et Diagnostic

* **Voir tout le trafic :** `tail -f /var/log/squid/access.log`
* **Vérifier les succès du cache (HIT) :** `tail -f /var/log/squid/access.log | grep -E "TCP_HIT|TCP_MEM_HIT"`

## Activation sur les clients

### Via l'interface Gnome (Graphique)

1. Ouvrez les **Paramètres**.
2. Allez dans **Réseau**.
3. Cliquez sur **Serveur mandataire réseau**.
4. Sélectionnez le mode **Manuel** : IP du serveur, port **3128**.

### Via les alias Bash (Terminal)

Pour éviter de retourner dans les menus de Gnome à chaque fois, nous allons créer des "raccourcis" clavier appelés **Alias**.

#### 1. Éditer le fichier des alias

Sur votre machine Linux (client), ouvrez le fichier caché `.bash_aliases` situé dans votre dossier personnel :

```bash
nano ~/.bash_aliases

```

#### 2. Ajouter les commandes

Copiez et collez les lignes suivantes à la fin du fichier (n'oubliez pas d'adapter l'IP du serveur proxy) :

```bash
# --- SECTION PROXY SQUID ---

# Pour tout le système (Gnome) : gproxyon active le proxy, gproxyoff le désactive.
alias gproxyon='gsettings set org.gnome.system.proxy mode "manual"'
alias gproxyoff='gsettings set org.gnome.system.proxy mode "none"'

# Pour le terminal uniquement : proxyon active le passage par le proxy pour les commandes comme curl ou wget.
alias proxyon='export http_proxy="http://192.168.2.153:3128" https_proxy="http://192.168.2.153:3128"'
alias proxyoff='unset http_proxy https_proxy'

```

#### 3. Appliquer les changements

Pour que votre terminal prenne en compte ces nouveaux raccourcis immédiatement, tapez :

```bash
source ~/.bashrc

```

#### Explication des alias :

* **`gproxyon` / `gproxyoff**` : Agit directement sur les paramètres système. C'est comme si vous cliquiez manuellement dans les réglages de Gnome. Très pratique pour tout le surf web.
* **`proxyon` / `proxyoff**` : Indique à vos logiciels en ligne de commande (comme `apt` ou `curl`) d'utiliser le proxy. Cela ne dure que le temps où votre fenêtre de terminal est ouverte.
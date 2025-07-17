---
title: Utiliser Nginx Proxy Manager
description: Nous allons voir comment utiliser Nginx Proxy Manager (NPM) qui est est un proxy inverse open source utilisé pour rediriger le trafic du site Web vers l'endroit approprié.
published: true
date: 2025-07-17T01:01:56.438Z
tags: nginx, proxy, web
editor: markdown
dateCreated: 2024-07-12T11:46:50.527Z
---

# Nginx Proxy Manager, c'est quoi ?

*Nginx Proxy Manager est un proxy inverse* [open source](https://fr.wikipedia.org/wiki/Open_source) *utilisé pour rediriger le trafic d'un site* [Web](https://fr.wikipedia.org/wiki/World_Wide_Web) *vers l'endroit approprié. L'utilisation de Nginx Proxy Manager permet d'utiliser une seule adresse* [IP](https://fr.wikipedia.org/wiki/Adresse_IP) *publique pour héberger de nombreux services Web différents.*

# Liens utiles

-   [NginxProxyManager.com](https://nginxproxymanager.com/)
-   [NginxProxyManager / nginx-proxy-manager on Github](https://github.com/NginxProxyManager/nginx-proxy-manager)
-   [jc21/nginx-proxy-manager on Docker Hub](https://hub.docker.com/r/jc21/nginx-proxy-manager)

# Nginx Proxy Manager déjà sur ce Wiki

-   Installer Nginx Proxy Manager grâce à [Docker](https://www.docker.com) ([Portainer](https://www.portainer.io)) et un fichier [Compose](https://docs.docker.com/compose/) : [/docker-compose-nginx-proxy-manager](/docker-compose-nginx-proxy-manager)

# Utilisation

Le but de cette page de Wiki est d'expliquer comment utiliser Nginx Proxy Manager pour rediriger le trafic d'un site Web de manière sécurisée.

La première chose à faire est la redirection des ports sur le routeur.

## Redirection ports 80 et 443 sur le routeur

On redirige le port **80** et **443** vers l'adresse IP LAN de Nginx Proxy Manager.

Expliquer exactement la marche à suivre, je ne saurais pas. Elle est propre à chaque routeur.

Néanmoins, je peux fournir une capture des redirections sur un de mes routeurs, qui est le [TP-Link Archer C80](https://www.tp-link.com/fr-be/home-networking/wifi-router/archer-c80/).

![](/utiliser-nginx-proxy-manager/npm-redirections-ports-routeur.png)

TP-Link Archer C80 - Redirections de ports

## Interface Nginx Proxy Manager

On va maintenant se rendre sur l'interface (dashboard) de Nginx Proxy Manager.

Pour ma part, ce sera l'adresse IP 192.168.2.210 avec le port 81 qui va bien.

Ce qui nous intéresse ici, et qui est la partie la plus importante de ce que propose Nginx Proxy Manager, c'est la fonctionnalité “Proxy Hosts”.

![](/utiliser-nginx-proxy-manager/npm-dashboard-proxy-hosts.png)

Nginx Proxy Manager Dashboard - Proxy Hosts

L'ensemble des redirections chez moi…

![](/utiliser-nginx-proxy-manager/nginx-proxy-hosts.png)

Ensemble des Proxy Hosts Blabla Linux

## Ajouter un Proxy Host

On clique sur le bouton en haut à droite “Add Proxy Host”…

<p style="text-align: center"><img src="/utiliser-nginx-proxy-manager/nginx-ajout-proxy-host.png"></p>


Nginx Proxy Manager - Add Proxy Host

On arrive sur cette fenêtre…

![](/utiliser-nginx-proxy-manager/nginx-ajout-proxy-host.-02.png)

Nginx Proxy Manager - Add New Proxy Host

## Exemple concret

On va prendre comme exemple le Wiki Blabla Linux, onglet “Details”…

<p style="text-align: center"><img src="/utiliser-nginx-proxy-manager/nginx-proxy-host-wiki-blabla-linux.png"></p>

Nginx Proxy Manager - Edit Proxy Host

### Avant d'aller plus loin !

Avant d'aller plus loin, vous devez avoir un [nom de domaine](https://fr.wikipedia.org/wiki/Nom_de_domaine) !

Pour ma part, mon nom de domaine est “[blablalinux.be](https://yourls.blablalinux.be/blog)”.

Il renvoie vert le blog Blabla Linux.

À partir de ce dernier, j'ai créé plusieurs sous domaine :

-   [etherpad](https://yourls.blablalinux.be/etherpad)
-   [fichiers](https://yourls.blablalinux.be/fichiers)
-   [jitsimeet](https://yourls.blablalinux.be/jitsimeet)
-   [linkwarden](https://yourls.blablalinux.be/linkwarden)
-   [mattermost](https://yourls.blablalinux.be/mattermost)
-   [nextcloud](https://yourls.blablalinux.be/nextcloud)
-   [picsur](https://yourls.blablalinux.be/picsur)
-   [privatebin](https://yourls.blablalinux.be/privatebin)
-   [psitransfer](https://yourls.blablalinux.be/psi-transfer)
-   [stirlingpdf](https://yourls.blablalinux.be/stirlingpdf)
-   [wiki](https://yourls.blablalinux.be/wikijs)

… et encore d'autres.

Nginx Proxy Manager s'occupe des redirections vers les différentes machines qui hébergent les services.

#### Pour comprendre fonctionnement

-   Chez le fournisseur de domaine, le (sous) domaine pointe vers l'adresse publique IP WAN…

<p style="text-align: center"><img src="/utiliser-nginx-proxy-manager/redirection-ip-wan-wiki-blabla-linux.png"></p>

Redirection adresse publique IP WAN Wiki blabla Linux

-   Les requêtes internet non sécurisées (port 80 - non SSL) et sécurisées (port 443 - SSL) arrivent sur le routeur et sont réceptionnées par Nginx Proxy Manager…

![](/utiliser-nginx-proxy-manager/npm-redirections-ports-routeur.png)

Routeur - Redirection adresse IP LAN Nginx Proxy Manager

-   Nginx Proxy Manager, d'après le (sous) domaine de la requête, redirige vers la bonne adresse IP LAN, et sur le bon port…

<p style="text-align: center"><img src="/utiliser-nginx-proxy-manager/nginx-proxy-host-wiki-blabla-linux-02.png"></p>

Wiki Blabla Linux - Redirection IP LAN Nginx Proxy Manager - Onglet “Details”

-   Le service est affiché…

![](/utiliser-nginx-proxy-manager/wiki-blabla-linux.png)

Wiki.js Blabla Linux

## Exemple concret (la suite)

Nous allons voir ce qu'il faut remplir/cocher pour une bonne redirection.

### Dans l'onglet “Details"

**_Domain Names_**

Le (sous) domaine doit être identique à celui créé chez le [bureau d'enregistrement](https://fr.wikipedia.org/wiki/Registraire_de_nom_de_domaine) ([OVH](https://ovh.com) pour moi).

**_Scheme_**

Comment accède-t-on au service ? Via HTTP ou HTTPS ?

**_Forward Hostname / IP_**

Le nom d'hôte de la machine (wiki.local) ou l'adresse IP LAN de celle-ci (192.168.2.154).

**_Forward Port_**

Le port au travers duquel on accède au service sur l'adresse IP LAN du service.

**_Cache Assets_**

Sans rentrer dans trop de technique, “Cache Assets” est un système de mise en cache côté Nginx, côté serveur. À utiliser seulement pour des services qui sont fortement sollicités. Les navigateurs sont modernes. Ils disposent déjà d'un système de mise en cache. On parle de mise en cache côté client. Si on choisit malgré tout d'activer le cache Nginx, il est conseillé, au début, de surveiller comment se comporte le service au niveau des ressources utilisées.

**_Block Common Exploits_**

Le serveur proxy inverse peut filtrer le trafic malveillant. Nginx Proxy Manager inclut un mode “bloquer les exploits courants”. Il peut également filtrer les adresses IP. Par exemple, on souhaite peut-être autoriser uniquement l'accès à certaines adresses IP.

**_Websockets Support_**

Le protocole WebSocket permet d'ouvrir un canal de communication bidirectionnel (ou "full-duplex") sur un socket TCP entre le navigateur et le serveur web.

Plus spécifiquement, il permet :

-   La notification au client d'un changement d'état du serveur.
-   L'envoi de données en mode “pousser” (méthode “Push”) du serveur vers le client, sans que ce dernier ait à effectuer une requête.

Exemple, le service utilise de l'audio, de la vidéo. Le service envoi des notifications, comme Mattermost, on active cette case.

*Je passe l'onglet “Custom Location” et “Advanced”. Ils seront vus lors d'une mise à jour de cette page de Wiki. Le plus important est l'onglet “Details”, que nous venons de voir, et l'onglet “SSL” que nous allons voir.*

### Dans l'onglet “SSL"

<p style="text-align: center"><img src="/utiliser-nginx-proxy-manager/nginx-proxy-host-ssl.png"></p>

Nginx Proxy Manager - Onglet SSL

Partons du principe que l'on souhaite sécuriser la requête. On va donc sélectionner “Request a new SSL Certificate” dans “SSL Certificate”. 

**_Force SSL_**

Forcer SSL force le navigateur à rediriger HTTP vers HTTPS.

**_HTTP/2 Support_**

[HTTP/2](https://fr.wikipedia.org/wiki/Hypertext_Transfer_Protocol/2) est la version révisée du protocole HTTP original. C'est la nouvelle version. Voulez-vous qu'il soit supporté ou non ?

**_HSTS enabled_**

Il ne pourra être activé que si “Force SSL” est d'abord activé. HTTP Strict Transport Security (HSTS) est un mécanisme de politique de sécurité proposé pour HTTP.

Lorsque la politique HSTS est active pour un [site web](https://fr.wikipedia.org/wiki/Site_web), l'agent utilisateur compatible opère comme suit :

-   Il remplace automatiquement tous les liens non sécurisés par des liens sécurisés. Par exemple, http://www.exemple.com/une/page/ est automatiquement remplacé par https://www.exemple.com/une/page/ avant d'accéder au serveur.

La politique HSTS aide à protéger les utilisateurs de sites web contre quelques attaques réseau passives ([écoute clandestine](https://fr.wikipedia.org/wiki/%C3%89coute_clandestine)) et actives. Une attaque du type [man-in-the-middle](https://w.wiki/AeMb) ne peut pas intercepter des requêtes tant que le HSTS est actif pour le site.

Plus d'information sur [Wikipédia](https://fr.wikipedia.org/wiki/HTTP_Strict_Transport_Security).

**_HSTS Subdomains_**

Ne pourra être activé uniquement si “HSTS enabled” l'est auparavant. Il permet d'utiliser la politique HSTS également sur les sous domaine.

**_Use DNS Challenge_**

Cette option sert à prouver que le domaine vous appartient !

Pour utiliser cette option, il faut créer une entrée DNS type TXT chez le fournisseur de domaine.

*Plus haut, j'ai donné l'onglet “Details” du Wiki Blabla Linux. Je vais achever en donnant l'onglet SSL du Wiki Blabla Linux…*

<p style="text-align: center"><img src="/utiliser-nginx-proxy-manager/nginx-proxy-host-ssl-wiki-blabla-linux.png"></p>

Wiki Blabla Linux - Redirection IP LAN Nginx Proxy Manager - Onglet “SSL”

# Conclusion

On est loin d'avoir fait le tour de ce formidable utilitaire Nginx Proxy Manager. 

On a seulement vu deux onglets de “Proxy Hosts” !

On a vu les options principales à utiliser pour effectuer une redirection sécurisée avec Let's Encrypt grâce à Nginx Proxy Manager.

Il y aura donc, soit une mise à jour de cette page avec des explications supplémentaires, ou, la création de nouvelles pages (part. 2, part. 3, etc.) qui compléteront celle-ci.
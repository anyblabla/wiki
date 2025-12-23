---
title: Déploiement d'un serveur proxy cache Squid sur Debian
description: Apprenez à déployer Squid sur Debian pour optimiser votre navigation. Ce guide couvre l'installation, la configuration du cache et l'utilisation d'alias GNOME pour un contrôle total de vos flux web.
published: true
date: 2025-12-23T00:52:47.824Z
tags: cache, proxy, debian, squid, administration système
editor: markdown
dateCreated: 2025-12-23T00:16:17.440Z
---

## Qu'est-ce qu'un serveur proxy ?

Un serveur proxy est un intermédiaire entre votre ordinateur (le client) et internet (le serveur distant). Au lieu de se connecter directement à un site web, votre navigateur envoie la requête au proxy, qui va chercher la ressource pour vous et vous la retransmet.

## Pourquoi le choix de Squid ?

Squid est la référence absolue dans le monde du logiciel libre pour les serveurs mandataires. Mes raisons pour ce choix sont simples :

* **Performances :** Une gestion du cache (RAM et disque) extrêmement robuste pour économiser de la bande passante.
* **Contrôle :** Une visibilité totale sur les flux sortants via les journaux d'accès.
* **Légèreté :** Il consomme très peu de ressources, ce qui en fait un candidat idéal pour un container LXC sous Proxmox.
* **Confidentialité :** Il permet d'anonymiser le trafic local en masquant l'adresse IP réelle de la machine cliente.

## Installation sur une base Debian

Connectez-vous en root sur votre machine ou container Debian :

```bash
apt update && apt install squid -y

```

## Paramétrage complet du fichier de configuration

Le fichier par défaut étant saturé de commentaires, nous allons le vider pour repartir sur une base saine, lisible et sécurisée.

1. Sauvegardez le fichier d'origine :
`cp /etc/squid/squid.conf /etc/squid/squid.conf.bak`
2. Videz le contenu et ouvrez le fichier :
`true > /etc/squid/squid.conf && nano /etc/squid/squid.conf`

### Configuration optimisée

```text
# Définition du réseau local
acl mon_reseau_local src 192.168.2.0/24

# Règles d'accès (http_access)
# Interdire les ports dangereux
http_access deny !Safe_ports
http_access deny CONNECT !SSL_ports

# Autoriser le manager uniquement depuis localhost
http_access allow localhost manager
http_access deny manager

# Autorisation du réseau local et de la machine elle-même
http_access allow mon_reseau_local
http_access allow localhost

# Protection contre les accès aux services locaux du container
http_access deny to_localhost

# Interdire tout le reste
http_access deny all

# Paramètres réseau et anonymisation
http_port 3128

# Masquer l'adresse IP réelle des clients (Anonymisation)
forwarded_for off
request_header_access Via deny all
request_header_access X-Forwarded-For deny all

# Gestion du cache (Optionnel)
# Décommentez la ligne suivante pour activer le cache disque (100 Mo)
# cache_dir ufs /var/spool/squid 100 16 256

# Emplacement des fichiers de log
access_log daemon:/var/log/squid/access.log squid

```

## Gestion du service et démarrage automatique

Appliquez la configuration et assurez-vous que le service démarre avec le système :

```bash
# Activer le démarrage automatique
systemctl enable squid

# Redémarrer pour appliquer les changements
systemctl restart squid

# Vérifier le statut
systemctl status squid

```

## Vérification du bon fonctionnement

Une fois le service redémarré, il est indispensable de vérifier que Squid est bien opérationnel.

### 1. Vérifier l'écoute du port 3128

Vérifiez que Squid attend bien des connexions :

```bash
ss -tulpn | grep 3128

```

### 2. Tester le proxy localement

Simulez une requête web pour confirmer que le proxy répond :

```bash
curl -I -x http://localhost:3128 http://www.google.com

```

*Un code `HTTP/1.1 200 OK` indique un succès.*

### 3. Surveiller les journaux

Pour voir les accès en temps réel, utilisez la commande suivante. **Notez bien** que vous ne verrez défiler des lignes qu'une fois le proxy activé sur votre client (Gnome ou Terminal) et après avoir effectué vos premières requêtes (navigation web) :

```bash
tail -f /var/log/squid/access.log

```

## Activation via les paramètres de Gnome

Pour que l'ensemble de votre système utilise le proxy, suivez cette procédure :

1. Ouvrez les **Paramètres** de Gnome > **Réseau**.
2. Cliquez sur la roue crantée de **Serveur mandataire réseau**.
3. Sélectionnez le mode **Manuel**.
4. Remplissez **HTTP**, **HTTPS** et **FTP** avec l'adresse IP du serveur et le port **3128**.
5. **Important :** Laissez le champ **Hôte SOCKS** vide.
6. Dans la section **Ignorer pour**, saisissez : `localhost, 127.0.0.0/8, ::1, 192.168.2.0/24`.

## Automatisation via les alias Bash

L'utilisation de l'interface graphique est fastidieuse. Ces alias permettent d'activer ou désactiver le serveur mandataire instantanément sans ouvrir les paramètres de Gnome.

Pour éditer vos alias :

```bash
nano ~/.bash_aliases

```

### Raccourcis à ajouter

```bash
### --- SECTION PROXY SQUID ---

## Proxy pour le terminal (Session shell actuelle)
alias proxyon='export http_proxy="http://192.168.2.153:3128" https_proxy="http://192.168.2.153:3128" ftp_proxy="http://192.168.2.153:3128"'
alias proxyoff='unset http_proxy https_proxy ftp_proxy'
alias checkproxy='curl -I http://www.google.com | grep -i "Via\|Squid"'

## Proxy pour l'environnement Gnome (Système complet)
# gproxyon  : Active le mode proxy manuel (évite l'interface graphique)
# gproxyoff : Désactive le proxy système
alias gproxyon='gsettings set org.gnome.system.proxy mode "manual"'
alias gproxyoff='gsettings set org.gnome.system.proxy mode "none"'
alias gproxycheck='gsettings get org.gnome.system.proxy mode'

```

Rechargez avec : `source ~/.bash_aliases`. Désormais, un simple `gproxyon` au clavier remplacera avantageusement toute la navigation dans les menus de Gnome.

> Si vous souhaitez utiliser ce proxy à l'extérieur, ne redirigez jamais le port 3128 sur votre routeur. Connectez-vous via votre VPN (WireGuard par exemple) et utilisez l'adresse IP locale du serveur.
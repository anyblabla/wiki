---
title: Utilisation de Nginx Proxy Manager (NPM)
description: Nginx Proxy Manager est un reverse proxy open source qui permet de rediriger le trafic Web vers l'emplacement approprié. Il est essentiel pour utiliser une seule adresse IP publique afin d'héberger de nombreux services Web différents (auto-hébergement).
published: true
date: 2025-10-28T14:21:56.260Z
tags: nginx, proxy, web
editor: markdown
dateCreated: 2024-07-12T11:46:50.527Z
---

**Nginx Proxy Manager** est un *reverse proxy* **open source** qui permet de rediriger le trafic Web vers l'emplacement approprié. Il est essentiel pour utiliser une seule adresse **IP publique** afin d'héberger de nombreux services Web différents (auto-hébergement).

---

## 1. Liens et Installation

| Catégorie | Lien |
| :--- | :--- |
| **Site Officiel** | [NginxProxyManager.com](https://nginxproxymanager.com/) |
| **GitHub** | [NginxProxyManager / nginx-proxy-manager](https://github.com/NginxProxyManager/nginx-proxy-manager) |
| **Docker Hub** | [jc21/nginx-proxy-manager](https://hub.docker.com/r/jc21/nginx-proxy-manager) |
| **Installation** | Voir le guide : [/docker-compose-nginx-proxy-manager](/docker-compose-nginx-proxy-manager) (via Docker/Portainer) |

---

## 2. Prérequis : Redirection de Ports sur le Routeur

Avant de configurer les *Proxy Hosts* dans NPM, vous devez rediriger le trafic entrant de votre routeur vers l'adresse IP locale (LAN) de votre conteneur Nginx Proxy Manager.

* **Port 80 (HTTP) :** Essentiel pour la validation des certificats SSL (Let's Encrypt).
* **Port 443 (HTTPS) :** Essentiel pour le trafic sécurisé.

> La procédure est propre à chaque routeur. Vous devez créer deux règles de redirection ciblant l'**IP LAN de NPM**. 

---

## 3. Configuration des Proxy Hosts

Le cœur de NPM réside dans la fonctionnalité **"Proxy Hosts"** (accessible via le tableau de bord). Chaque *Proxy Host* définit une règle de redirection spécifique basée sur un nom de domaine.

![](/utiliser-nginx-proxy-manager/npm-dashboard-proxy-hosts.png)

![](/utiliser-nginx-proxy-manager/nginx-proxy-hosts.png)

<p style="text-align: center"><img src="/utiliser-nginx-proxy-manager/nginx-ajout-proxy-host.png"></p>

![](/utiliser-nginx-proxy-manager/nginx-ajout-proxy-host.-02.png)

### 3.1. Structure de Redirection (Exemple : `wiki.blablalinux.be`)

1.  **Enregistreur de Domaine (OVH, etc.) :** Votre sous-domaine (`wiki.blablalinux.be`) doit pointer vers votre adresse **IP publique (WAN)**. 
2.  **Routeur :** Les requêtes (ports 80/443) arrivent sur le routeur, qui les transfère à l'**IP LAN de NPM**. 
3.  **NPM :** NPM analyse le sous-domaine et redirige la requête vers la machine/le service interne approprié (hôte LAN et port). 

<p style="text-align: center"><img src="/utiliser-nginx-proxy-manager/redirection-ip-wan-wiki-blabla-linux.png"></p>

<p style="text-align: center"><img src="/utiliser-nginx-proxy-manager/npm-redirections-ports-routeur.png"></p>

<p style="text-align: center"><img src="/utiliser-nginx-proxy-manager/nginx-proxy-host-wiki-blabla-linux.png"></p>

### 3.2. Onglet "Details" (Redirection interne)

Cet onglet définit où le trafic doit être envoyé sur votre réseau local.

| Champ | Explication |
| :--- | :--- |
| **Domain Names** | Le nom de domaine complet (ex. : `wiki.blablalinux.be`). Doit correspondre à l'entrée DNS. |
| **Scheme** | Protocole d'accès au service cible sur votre réseau (`HTTP` ou `HTTPS`). |
| **Forward Hostname / IP** | Nom d'hôte ou **adresse IP LAN** du service cible (ex. : `192.168.2.154`). |
| **Forward Port** | Le port sur lequel le service cible est accessible sur son hôte LAN (ex. : `3000` pour Wiki.js). |
| **Block Common Exploits** | **Recommandé.** Active un filtrage de base contre le trafic malveillant. |
| **Websockets Support** | **À activer** si le service utilise l'audio, la vidéo, ou l'envoi de notifications en temps réel (ex. : Mattermost, Jitsi). |

<p style="text-align: center"><img src="/utiliser-nginx-proxy-manager/nginx-proxy-host-wiki-blabla-linux-02.png"></p>

### 3.3. Onglet "SSL" (Sécurisation HTTPS)

Cet onglet permet d'obtenir et de gérer les certificats Let's Encrypt pour sécuriser le domaine.

1.  **SSL Certificate :** Sélectionnez **"Request a new SSL Certificate"**.
2.  **Force SSL :** **Recommandé.** Force le navigateur à rediriger toutes les requêtes HTTP vers HTTPS.
3.  **HSTS enabled :** **Recommandé** (si Force SSL est actif). Ajoute l'en-tête *HTTP Strict Transport Security* pour forcer le navigateur à n'utiliser que le HTTPS pour ce domaine, protégeant contre certaines attaques *man-in-the-middle*.
4.  **HSTS Subdomains :** (Si HSTS est actif). Applique la politique HSTS aux sous-domaines du domaine actuel.

> **Use DNS Challenge :** Cette option est utilisée lorsque la validation par le port 80 est impossible (par exemple, si le port 80 est déjà utilisé ailleurs). Elle requiert la création d'une entrée DNS de type TXT chez votre fournisseur de domaine pour prouver la propriété.

<p style="text-align: center"><img src="/utiliser-nginx-proxy-manager/nginx-proxy-host-ssl.png"></p>

---

## Conclusion

Nous avons couvert les bases de la redirection et de la sécurisation d'un service avec Nginx Proxy Manager via les onglets "Details" et "SSL" des *Proxy Hosts*. D'autres fonctionnalités avancées (comme "Custom Location" et "Advanced") seront abordées dans une prochaine mise à jour ou page.
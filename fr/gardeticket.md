---
title: GardeTicket - Garde tes tickets et garanties sur ton téléphone
description: >-
  GardeTicket : application libre et privée pour garder tes tickets de caisse et
  suivre tes garanties. Photo ou PDF, alertes avant expiration, fonctionne hors
  ligne. Tout reste sur ton téléphone. Code sur GitHub et Gitea, déploiement
  Debian + nginx.
published: true
date: '2026-11-28T16:25:23.770Z'
tags:
  - gardeticket
  - pwa
  - garantie
  - tickets de caisse
  - vie privée
  - logiciel libre
  - auto hébergement
  - nginx
  - debian
  - debian 13
  - nginx proxy manager
editor: markdown
dateCreated: '2026-11-24T22:43:00.046Z'
---

**GardeTicket** est une application web (PWA) pour garder tes tickets de caisse et suivre tes garanties. Tout est stocké **uniquement dans le navigateur de ton téléphone** : aucune donnée n'est envoyée sur un serveur.

👉 **Utiliser l'application : [gardeticket.blablalinux.be](https://gardeticket.blablalinux.be)** (à ouvrir depuis un smartphone ou une tablette)

## Ce que fait GardeTicket

- 📷 Photographier un ticket (recadrage automatique, filtre « scanner » noir et blanc pour les tickets pâles) ou importer une facture PDF
- 📅 Calculer la fin de garantie (1, 2, 3, 5 ans ou à vie) et afficher des repères de couleur : vert, orange (moins de 3 mois), gris (expirée)
- 🔔 Afficher un bandeau d'alerte avant l'expiration et créer un rappel dans ton agenda (fichier `.ics`), 30 jours avant la fin
- 🔍 Retrouver un achat par objet, magasin ou numéro de série
- 📊 Consulter les statistiques de tes achats (nombre de tickets, montant total suivi, valeur encore sous garantie, répartition par année et par magasin)
- 📤 Partager le ticket ou la facture avec un service après-vente
- 💾 Sauvegarder tes tickets dans un fichier (`.json`), les envoyer vers ton cloud (`.txt`) et les restaurer
- 🎨 Choisir parmi 8 thèmes de couleurs, avec mode clair ou sombre automatique
- 📴 Fonctionner même sans connexion internet

## Installer l'application sur ton téléphone

GardeTicket s'ouvre dans le navigateur, mais tu peux l'installer comme une vraie application :

- **Android** (Chrome, Vivaldi, Brave…) : menu (les trois points) → *Ajouter à l'écran d'accueil* ou *Installer l'application*
- **iPhone** (Safari) : bouton *Partager* → *Sur l'écran d'accueil*

> ℹ️ L'application est conçue pour les smartphones et tablettes : l'accès depuis un ordinateur de bureau est volontairement bloqué.

## Vie privée et sauvegardes

Tes tickets restent dans le navigateur de ton téléphone. Si tu vides les données du navigateur, si tu le désinstalles ou si tu changes de téléphone, **ils seront effacés**.

Pense à utiliser régulièrement le bouton **💾 Sauvegarder** et garde le fichier obtenu dans un endroit sûr. Le bouton **📂 Restaurer** permet de tout récupérer (fichiers `.json` ou `.txt`), sans doublons.

## Héberger GardeTicket chez toi

L'application est un simple site statique (HTML, CSS, JavaScript), sans base de données ni dépendance. Tu peux l'héberger toi-même pour garder la maîtrise de ton instance.

**Ce qu'il te faut :**

- un serveur (une machine virtuelle, un conteneur LXC ou un Raspberry Pi suffisent) ;
- un **domaine ou sous-domaine dédié** : l'application doit être publiée à la racine (`/`) ;
- **HTTPS**, obligatoire pour installer l'application sur le téléphone, utiliser le mode hors ligne et accéder à l'appareil photo.

**Quelle option choisir ?**

| Option | Pour qui ? |
| --- | --- |
| **1. Docker Compose** | Recommandée si tu auto-héberges déjà tes services avec Docker |
| **2. Script derrière un reverse proxy** | Serveur Debian 13 avec Nginx Proxy Manager (ou autre) pour le HTTPS |
| **3. Script autonome** | Serveur Debian 13 qui gère lui-même le HTTPS avec Let's Encrypt |

### Option 1 : avec Docker Compose (recommandé)

L'image multi-architecture (`linux/amd64` et `linux/arm64`) est publiée automatiquement sur le GitHub Container Registry (`ghcr.io/anyblabla/gardeticket:latest`) et sur mon Gitea (`gitea.blablalinux.be/blablalinux/gardeticket:latest`).

```bash
mkdir -p gardeticket && cd gardeticket
cat << 'YAML' > docker-compose.yml
services:
  gardeticket:
    image: ghcr.io/anyblabla/gardeticket:latest
    container_name: gardeticket
    restart: unless-stopped
    ports:
      - "8080:80"
YAML
docker compose up -d
```

> ℹ️ Même hébergée via Docker, l'application reste **100 % côté client** : aucun volume n'est nécessaire (et en ajouter un bloquerait les mises à jour automatiques de l'image), les tickets restent stockés dans le navigateur de chaque téléphone.

<details>
<summary>Personnaliser les fichiers de l'application avec Docker (avancé)</summary>

Si tu souhaites modifier toi-même le code HTML/CSS/JS tout en utilisant Docker, extrais d'abord le dossier web du conteneur avant de le monter en volume (les mises à jour automatiques de l'interface ne s'appliqueront alors plus) :

```bash
docker cp gardeticket:/var/www/gardeticket ./www
```

Ajoute ensuite le volume dans `docker-compose.yml` et redémarre le conteneur :

```yaml
    volumes:
      - ./www:/var/www/gardeticket:ro
```

</details>

L'application répond alors en HTTP sur le port `8080` de ton serveur. Place le conteneur derrière un reverse proxy en **HTTPS** (voir [Réglages Nginx Proxy Manager](#réglages-nginx-proxy-manager)).

### Option 2 : avec le script d'installation, derrière un reverse proxy (Debian 13)

Le dépôt contient un script qui installe nginx et déploie les fichiers dans `/var/www/gardeticket` :

```bash
sudo apt-get install -y git
git clone [https://github.com/anyblabla/gardeticket.git](https://github.com/anyblabla/gardeticket.git)
cd gardeticket
sudo bash deploy/install.sh
```

Le serveur sert alors l'application en HTTP sur le port `80` avec un vhost minimal. Il ne reste qu'à configurer ton reverse proxy (voir [Réglages Nginx Proxy Manager](#réglages-nginx-proxy-manager)).

### Option 3 : avec le script d'installation, en serveur autonome (HTTPS Let's Encrypt)

```bash
sudo bash deploy/install.sh --standalone --domain tickets.exemple.be --email toi@exemple.be
```

Le vhost nginx porte alors lui-même les en-têtes de sécurité, la CSP, le blocage des ordinateurs de bureau et le certificat Let's Encrypt.

<details>
<summary>Les options du script <code>install.sh</code></summary>

| Option | Effet |
| --- | --- |
| `--domain NOM` | nom de domaine du vhost (défaut : site par défaut) |
| `--standalone` | vhost autonome (en-têtes, CSP, blocage des ordinateurs) |
| `--email ADRESSE` | HTTPS Let's Encrypt (implique `--standalone`, nécessite `--domain`) |
| `--web-root CHEMIN` | dossier de publication (défaut `/var/www/gardeticket`) |
| `--allow-desktop` | avec `--standalone` : n'interdit pas les ordinateurs |
| `--update` | redéploie seulement les fichiers, sans toucher à nginx |
| `--force-nginx` | régénère le vhost même s'il existe déjà |
| `--uninstall` | supprime le site (nginx reste installé) |

Le script est relançable : un vhost existant n'est pas écrasé (sauf avec `--force-nginx`) et l'ancien contenu du site est conservé dans `/var/backups/gardeticket/previous`.

</details>

### Réglages Nginx Proxy Manager

Pour les options 1 et 2, voici les réglages de l'hôte dans NPM :

- **Détails** : `http`, IP du serveur, port `8080` (Docker) ou `80` (script) ;
- **SSL** : certificat, *Forcer SSL*, *HTTP/2*, *HSTS* ;
- **Configuration Nginx personnalisée** : colle le contenu de [`deploy/npm-advanced-config.conf`](https://github.com/anyblabla/gardeticket/blob/main/deploy/npm-advanced-config.conf) (en-têtes de sécurité, CSP, blocage des ordinateurs de bureau) ;
- **Page 403 « mobile uniquement »** : copie [`deploy/403_gardeticket/index.html`](https://github.com/anyblabla/gardeticket/blob/main/deploy/403_gardeticket/index.html) dans le dossier `data/403_gardeticket/` de NPM (visible dans le conteneur sous `/data/403_gardeticket/`). C'est ce que verront les visiteurs sur ordinateur ;
- **Emplacements personnalisés** : ajoute `/sw.js` (vers `http://<IP_SERVEUR>:<PORT>`) avec `Cache-Control "no-cache, no-store, must-revalidate"` et `X-Content-Type-Options "nosniff"`, pour éviter qu'un vieux service worker reste en cache.

## Mettre à jour

**Avec Docker Compose :**

```bash
docker compose pull && docker compose up -d
```

Compatible aussi avec Watchtower pour des mises à jour automatiques.

**Avec le script d'installation :**

```bash
cd gardeticket
git pull
sudo bash deploy/install.sh --update
```

> ⚠️ Le service worker met les fichiers en cache hors ligne. Si tu modifies toi-même les fichiers de `app/`, incrémente `CACHE_NAME` dans `app/sw.js` (par exemple `gardeticket-v28`), sinon les téléphones garderont l'ancienne version.

## Points importants

- **HTTPS obligatoire** : sans lui, le service worker et l'installation en application ne fonctionnent pas.
- **Publier à la racine d'un domaine ou sous-domaine** : `manifest.json` et `sw.js` utilisent des chemins absolus (`/`, `/icon-192.png`…).
- **CSP stricte** (`script-src 'self'`) : pas de JavaScript inline dans les pages, d'où le fichier `theme-init.js`. Pas de `onclick="…"` non plus.
- **Mobile uniquement** : le blocage des ordinateurs repose sur le User-Agent, il est donc contournable. L'exception `Uptime-Kuma` laisse passer la supervision.

## Un souci ?

- **Le bouton « Installer l'application » n'apparaît pas, ou l'appareil photo ne fonctionne pas** : vérifie que le site est bien servi en **HTTPS**.
- **L'ancienne version s'affiche après une mise à jour** : vérifie l'emplacement personnalisé `/sw.js` (sans cache) et, si tu as modifié les fichiers, le numéro `CACHE_NAME` dans `app/sw.js`.
- **Une page « accès refusé » s'affiche sur ordinateur** : c'est le blocage volontaire des ordinateurs de bureau. En mode autonome, tu peux l'enlever avec `--allow-desktop`.
- **Mes tickets ont disparu** : ils sont stockés dans le navigateur, donc effacés si ses données ont été vidées. Utilise **📂 Restaurer** avec ta dernière sauvegarde.

## Code source

Le code est libre et disponible sur deux dépôts identiques :

- 🐙 **GitHub** : [github.com/anyblabla/gardeticket](https://github.com/anyblabla/gardeticket)
- 🍵 **Gitea** (mon instance) : [gitea.blablalinux.be/blablalinux/gardeticket](https://gitea.blablalinux.be/blablalinux/gardeticket)

Les suggestions et signalements de problèmes sont les bienvenus sur l'un ou l'autre.

## Licence

GardeTicket est un logiciel libre distribué sous licence **GNU AGPL v3**.

---

> **Author**: this guide is proposed by **Amaury aka BlablaLinux**. Find all my services on [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).

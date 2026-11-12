---
title: Déployer Remark42 avec Docker et l'intégrer dans Wiki.js 3
description: >-
  Déployez et configurez Remark42 avec Docker Compose et Nginx Proxy Manager
  pour intégrer un système de commentaires léger, respectueux de la vie privée
  et en français sur votre instance Wiki.js 3.
published: true
date: '2026-11-12T15:41:59.529Z'
tags:
  - remark42
  - wikijs
  - docker
  - nginx proxy manager
  - auto hébergement
  - commentaire
editor: markdown
dateCreated: '2026-11-12T14:46:41.886Z'
---

**Remark42** est un moteur de commentaires auto-hébergé, léger et respectueux de la vie privée. Il fonctionne sans base de données externe (il utilise BoltDB en local) et permet aux visiteurs de commenter vos pages via une connexion par e-mail ou via des fournisseurs tiers comme GitHub.

Ce guide présente l'installation complète de Remark42 avec Docker Compose, sa configuration derrière **Nginx Proxy Manager** et son intégration en français dans **Wiki.js 3**.

* **Documentation officielle :** [remark42.com/docs/getting-started/installation/](https://remark42.com/docs/getting-started/installation/)
* **Dépôt GitHub du projet :** [github.com/umputun/remark42](https://github.com/umputun/remark42)

---

## Prérequis

* Un serveur Linux (Debian ou Ubuntu) avec **Docker** et **Docker Compose** installés.
* Une instance **Wiki.js 3** fonctionnelle (par exemple sur `https://wiki.exemple.com`).
* Un sous-domaine dédié pour Remark42 pointant vers votre serveur (par exemple `https://remark.exemple.com`).
* **Nginx Proxy Manager** (NPM) pour gérer le reverse proxy et les certificats SSL/TLS.

---

## 1. Déployer Remark42 avec Docker Compose

Créez un dossier dédié pour votre conteneur sur le serveur hôte :

```bash
mkdir -p ~/remark42
cd ~/remark42
```

### Générer le secret JWT

Remark42 a besoin d'une clé secrète robuste pour signer les jetons d'authentification. Générez-la avec `openssl` :

```bash
openssl rand -hex 32
```

Conservez la chaîne générée pour la prochaine étape.

### Créer le fichier `.env`

Créez le fichier d'environnement `.env` afin d'isoler vos paramètres sensibles :

```bash
nano .env
```

Collez et adaptez la configuration suivante selon vos besoins :

```env
# ============================================================
# REMARK42 - CONFIGURATION GENERALE
# ============================================================

REMARK_URL=https://remark.exemple.com

# Identifiant du site utilisé dans Wiki.js
SITE=wiki

# Secret JWT généré avec openssl
SECRET=remplacer_par_votre_cle_secrete_generee_avec_openssl

# Fuseau horaire
TIME_ZONE=Europe/Brussels

# ============================================================
# ADMINISTRATION & MODERATION
# ============================================================

# Identifiant de l'administrateur (à récupérer après la première connexion)
ADMIN_SHARED_ID=
ADMIN_SHARED_EMAIL=contact@exemple.com

# ============================================================
# REVERSE PROXY, SECURITE & OPTIONS DE CONFORT
# ============================================================

# IP de Nginx Proxy Manager + réseau interne bridge Docker
TRUSTED_PROXIES=192.168.1.10,172.16.0.0/12

# Domaines autorisés à embarquer les commentaires
ALLOWED_HOSTS=wiki.exemple.com,remark.exemple.com

# Gestion des cookies dans l'iframe (évite les déconnexions sur Firefox/Brave)
AUTH_SAME_SITE=none

# Activer les émojis et définir le délai d'édition des commentaires
EMOJI=true
EDIT_TIME=15m

# ============================================================
# AUTHENTIFICATION GITHUB (OPTIONNEL)
# ============================================================

AUTH_GITHUB_CID=votre_client_id_github
AUTH_GITHUB_CSEC=votre_client_secret_github

# ============================================================
# AUTHENTIFICATION PAR EMAIL
# ============================================================

AUTH_EMAIL_ENABLE=true
AUTH_EMAIL_FROM="Remark42 - Mon Wiki <contact@exemple.com>"
AUTH_EMAIL_SUBJ=Confirmation de connexion Remark42

# ============================================================
# CONFIGURATION SMTP
# ============================================================

SMTP_HOST=smtp.exemple.com
SMTP_PORT=465
SMTP_TLS=true
SMTP_USERNAME=contact@exemple.com
SMTP_PASSWORD=votre_mot_de_passe_smtp

# ============================================================
# NOTIFICATIONS EMAIL
# ============================================================

NOTIFY_USERS=email
NOTIFY_ADMINS=email
NOTIFY_EMAIL_FROM="Remark42 - Mon Wiki <contact@exemple.com>"
```

Sécurisez les permissions du fichier `.env` :

```bash
chmod 600 .env
```

> **Note concernant GitHub OAuth :** Si vous activez la connexion via GitHub, créez une application OAuth directement depuis vos paramètres développeur GitHub sur [github.com/settings/applications/new](https://github.com/settings/applications/new). Renseignez `https://remark.exemple.com` dans **Homepage URL** et `https://remark.exemple.com/auth/github/callback` dans **Authorization callback URL**, puis récupérez votre **Client ID** et générez un **Client Secret**.

### Créer le fichier `docker-compose.yml`

Créez le fichier de composition Docker :

```bash
nano docker-compose.yml
```

Ajoutez le contenu suivant :

```yaml
services:
  remark42:
    image: ghcr.io/umputun/remark42:latest
    container_name: remark42
    hostname: remark42
    restart: always

    env_file:
      - .env

    ports:
      - "8080:8080"

    volumes:
      - ./var:/srv/var

    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "5"

    security_opt:
      - no-new-privileges:true
```

### Démarrer le conteneur

Lancez le service en arrière-plan :

```bash
docker compose up -d
```

Vérifiez que le conteneur tourne correctement :

```bash
docker compose ps
```

---

## 2. Configurer Nginx Proxy Manager

Remark42 s'affiche dans Wiki.js sous la forme d'une `iframe` hébergée sur un sous-domaine distinct (`remark.exemple.com` intégré dans `wiki.exemple.com`). Il faut donc configurer les deux hôtes proxy dans Nginx Proxy Manager pour autoriser cette communication.

### A. Proxy Host pour Remark42 (`remark.exemple.com`)

1. Dans **Hosts > Proxy Hosts**, cliquez sur **Add Proxy Host**.
2. **Onglet Détails :**
   * **Domain Names :** `remark.exemple.com`
   * **Scheme :** `http`
   * **Forward Hostname / IP :** l'adresse IP locale de votre serveur Docker (ex. `192.168.1.50`)
   * **Forward Port :** `8080`
   * **Cache Assets :** désactivé
   * **Block Common Exploits :** désactivé (recommandé sur un wiki technique afin d'éviter que les extraits de code dans les commentaires ne soient bloqués par le pare-feu applicatif)
   * **Websockets Support :** activé
3. **Onglet SSL :**
   * Sélectionnez votre certificat SSL (`*.exemple.com` ou Let's Encrypt), puis activez **Force SSL**, **HTTP/2 Support**, **HSTS Enabled** et **HSTS Subdomains**.
4. **Onglet Emplacement personnalisé (Custom Locations) :**
   * Ajoutez l'emplacement `/` pointant vers `http://192.168.1.50:8080`.
   * Cliquez sur la roue crantée de cet emplacement pour ouvrir sa configuration avancée et ajoutez les en-têtes suivants afin d'autoriser l'intégration en `iframe` uniquement depuis votre wiki :

```nginx
proxy_hide_header X-Powered-By;
proxy_hide_header X-Frame-Options;

add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header X-Xss-Protection "1; mode=block" always;
add_header X-Robots-Tag "noindex, noarchive, nofollow" always;
add_header Content-Security-Policy "frame-ancestors 'self' [https://wiki.exemple.com](https://wiki.exemple.com);" always;
```

### B. Adapter la CSP du Proxy Host de Wiki.js (`wiki.exemple.com`)

Si vous appliquez une politique de sécurité **Content Security Policy (CSP)** dans Nginx Proxy Manager pour votre wiki, vous devez autoriser le domaine `https://remark.exemple.com` dans les directives `script-src`, `frame-src`, `connect-src` et `style-src` :

```nginx
proxy_hide_header Content-Security-Policy;
add_header Content-Security-Policy "default-src 'self' 'unsafe-inline' 'unsafe-eval'; frame-src 'self' [https://remark.exemple.com](https://remark.exemple.com); connect-src 'self' [https://remark.exemple.com](https://remark.exemple.com); script-src 'self' 'unsafe-inline' 'unsafe-eval' [https://remark.exemple.com](https://remark.exemple.com); img-src * data: blob: [https://wiki.exemple.com](https://wiki.exemple.com); style-src 'self' 'unsafe-inline' [https://remark.exemple.com](https://remark.exemple.com);" always;
```

Enregistrez les modifications dans Nginx Proxy Manager.

---

## 3. Intégrer Remark42 dans Wiki.js 3

### Activer le module de commentaires natif

1. Connectez-vous à l'interface d'administration de Wiki.js (`https://wiki.exemple.com/_admin`).
2. Dans le menu de gauche, ouvrez **Commentaires**.
3. Sélectionnez **Remark42** dans la liste des fournisseurs et renseignez les champs :
   * **Server URL :** `https://remark.exemple.com`
   * **Site ID :** `wiki` (doit correspondre exactement à la variable `SITE` du fichier `.env`)
   * **Theme :** `Dark` ou `Light` selon l'apparence de votre wiki
   * **Comments Shown :** `15` (nombre de commentaires affichés avant le bouton de chargement)
4. Cliquez sur **Appliquer** en haut à droite.

### Passer l'interface Remark42 en français

Le module natif de Wiki.js 3 ne propose pas de sélecteur de langue et force la configuration par défaut en anglais. Pour afficher les boutons et libellés de Remark42 en français tout en conservant la barre d'outils de mise en forme complète, interceptez la variable globale `window.remark_config` dans l'en-tête du thème :

1. Dans l'administration de Wiki.js, allez dans **Thème**.
2. Repérez la section **Head HTML Injection** et collez le script suivant :

```html
<script>
  (function() {
    var _remarkConfig = {
      locale: 'fr'
    };
    Object.defineProperty(window, 'remark_config', {
      configurable: true,
      enumerable: true,
      get: function() {
        return _remarkConfig;
      },
      set: function(val) {
        _remarkConfig = Object.assign({}, val, {
          locale: 'fr'
        });
      }
    });
  })();
</script>
```

3. Cliquez sur **Appliquer**, puis rechargez une page publique de votre wiki avec `Ctrl + F5`. L'espace commentaires s'affiche désormais en français sous vos articles avec tous ses boutons de formatage Markdown.

---

## 4. Activer les droits de modération

Remark42 ne dispose pas d'un panneau d'administration séparé : les outils de modération apparaissent directement dans le bloc de commentaires sous vos articles une fois votre identifiant déclaré.

1. Rendez-vous sur une page de votre wiki et connectez-vous dans le module Remark42 avec votre compte administrateur.
2. Cliquez sur votre nom d'utilisateur en haut à gauche du bloc de commentaires pour ouvrir le panneau latéral d'informations.
3. Copiez votre identifiant complet affiché sous votre pseudo (par exemple `github_a1b2c3d4e5f6...` ou `email_a1b2c3d4e5f6...`).
4. Éditez le fichier `~/remark42/.env` sur votre serveur et renseignez la variable `ADMIN_SHARED_ID` :

```env
ADMIN_SHARED_ID=github_a1b2c3d4e5f6...
```

5. Recréez le conteneur pour appliquer la modification :

```bash
cd ~/remark42
docker compose up -d
```

Après rechargement de votre wiki, vous pouvez désormais épingler ou supprimer des commentaires, bloquer des utilisateurs indésirables et verrouiller la discussion sur une page spécifique.

---

## 5. Supervision avec Uptime Kuma

La racine `https://remark.exemple.com` renvoie par défaut un code HTTP `404`. Pour superviser la disponibilité de votre instance dans **Uptime Kuma** sans générer de fausse alerte, utilisez l'une des deux URL suivantes :

* **`https://remark.exemple.com/ping`** : point de terminaison officiel de santé (*healthcheck*) qui renvoie `pong` avec un code HTTP `200`.
* **`https://remark.exemple.com/web/`** : page de démonstration intégrée qui renvoie également un code HTTP `200` et offre un affichage propre si le lien est cliquable depuis une page de statut publique.

> **Auteur** : ce guide est proposé par **Amaury aka BlablaLinux**. Retrouvez tous mes services sur [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).

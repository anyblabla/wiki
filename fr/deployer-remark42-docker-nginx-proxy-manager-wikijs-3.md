---
title: Déployer Remark42 avec Docker et l'intégrer dans Wiki.js 3
description: >-
  Déployez et configurez Remark42 avec Docker Compose et Nginx Proxy Manager
  pour intégrer un système de commentaires auto-hébergé, filtré, sécurisé et
  harmonisé (formulaire, boutons et prévisualisation) avec le thème de votre
  instance Wiki.js 3.
published: true
date: '2026-11-13T01:07:01.811Z'
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

Ce guide présente l'installation complète de Remark42 avec Docker Compose, sa configuration sécurisée derrière **Nginx Proxy Manager**, le filtrage des contenus indésirables ainsi que son intégration en français et aux couleurs de **Wiki.js 3**.

* **Documentation d'installation :** [remark42.com/docs/getting-started/installation/](https://remark42.com/docs/getting-started/installation/)
* **Paramètres de configuration :** [remark42.com/docs/configuration/parameters/](https://remark42.com/docs/configuration/parameters/)
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

Créez le fichier d'environnement `.env` afin d'isoler vos paramètres sensibles et de centraliser toute la configuration du serveur :

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
# FILTRAGE DE CONTENU (MOTS INTERDITS)
# ============================================================

RESTRICTED_WORDS=connard,connasse,salope,enculé,encule,putain,merde,bordel,fdp,ntm,tg,ta gueule,ferme ta gueule,abruti,débile,imbécile,crétin,taré,trou du cul,fils de pute,va te faire,nique,bâtard,batard,bouffon,clochard,casino,paris sportifs,argent facile,gagner de l'argent,revenus passifs,investissement garanti,trading,forex,cryptomonnaie,bitcoin gratuit,prêt rapide,crédit rapide,rachat de crédit,marabout,voyance gratuite,retour affectif,viagra,cialis,porno,escort,plan cul,rencontre coquine,onlyfans

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

Sécurisez les permissions du fichier `.env` afin qu'il ne soit lisible que par son propriétaire :

```bash
chmod 600 .env
```

### Explications détaillées des paramètres clés

* **`TRUSTED_PROXIES`** : Par défaut, Remark42 ignore les en-têtes `X-Forwarded-For` et `X-Real-IP` tant que l'adresse du reverse proxy n'est pas déclarée. Sans cette variable, tous vos visiteurs partagent la même adresse IP (celle du proxy) et la protection anti-spam risque de bloquer les utilisateurs légitimes. Il est important d'indiquer **à la fois l'adresse IP de votre Nginx Proxy Manager** (ici `192.168.1.10`) **et la plage du réseau bridge Docker** (`172.16.0.0/12`), car le NAT interne de Docker réécrit souvent l'adresse source avant qu'elle n'atteigne le conteneur.
* **`ALLOWED_HOSTS`** : Limite strictement les noms de domaine autorisés à charger et interroger votre instance Remark42. Cela empêche un site tiers d'embarquer votre module de commentaires à votre insu.
* **`AUTH_SAME_SITE=none`** : Comme votre wiki (`wiki.exemple.com`) affiche Remark42 (`remark.exemple.com`) dans une `iframe` cross-domaine, les navigateurs stricts (Firefox, Brave, Safari) bloquent les cookies de session par défaut. Ce paramètre autorise le maintien de la session utilisateur dans l'iframe (nécessite obligatoirement le HTTPS).
* **`EMOJI=true`** : Désactivé par défaut, ce paramètre ajoute un sélecteur d'émojis directement dans la barre d'outils de l'éditeur de commentaires.
* **`EDIT_TIME=15m`** : Définit la durée pendant laquelle un visiteur peut modifier son commentaire après publication (par défaut `5m`). Passer à `15m` laisse le temps de corriger une faute ou une ligne de commande.
* **`RESTRICTED_WORDS`** : Permet de bloquer automatiquement la publication d'un commentaire contenant l'un des mots ou expressions listés (insensible à la casse, séparés par des virgules sans espace).

> **Attention avec `RESTRICTED_WORDS` sur un wiki technique :** Remark42 recherche une **correspondance de chaîne de caractères** et non un mot isolé. Évitez d'ajouter des mots très courts susceptibles de faire partie du vocabulaire informatique courant (par exemple, interdire `con` bloquerait aussi *configuration*, *conteneur* ou *connexion*). Privilégiez des mots entiers ou des expressions ciblées.

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

> **Sauvegardes automatiques :** Remark42 génère chaque jour une sauvegarde de sa base de données BoltDB dans `/srv/var/backup` (en conservant les 10 dernières par défaut). Grâce au montage `./var:/srv/var`, vos sauvegardes sont automatiquement conservées sur l'hôte.

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

Remark42 s'affiche dans Wiki.js sous la forme d'une `iframe` hébergée sur un sous-domaine distinct (`remark.exemple.com` intégré dans `wiki.exemple.com`). Comme les navigateurs bloquent le CSS d'un site parent à l'intérieur d'une `iframe` externe, nous utilisons également le module `sub_filter` de Nginx Proxy Manager pour harmoniser directement les couleurs du module (formulaire, boutons, liens et fenêtre de prévisualisation) avec la charte graphique du wiki.

### A. Proxy Host pour Remark42 (`remark.exemple.com`)

1. Dans **Hosts > Proxy Hosts**, cliquez sur **Add Proxy Host**.
2. **Onglet Détails :**
   * **Domain Names :** `remark.exemple.com`
   * **Scheme :** `http`
   * **Forward Hostname / IP :** l'adresse IP locale de votre serveur Docker (ex. `192.168.1.50`)
   * **Forward Port :** `8080`
   * **Cache Assets :** désactivé
   * **Block Common Exploits :** désactivé (recommandé sur un wiki technique afin d'éviter que les extraits de code ou commandes Linux dans les commentaires ne soient bloqués par le pare-feu applicatif)
   * **Websockets Support :** activé
3. **Onglet SSL :**
   * Sélectionnez votre certificat SSL (`*.exemple.com` ou Let's Encrypt), puis activez **Force SSL**, **HTTP/2 Support**, **HSTS Enabled** et **HSTS Subdomains**.
4. **Onglet Emplacement personnalisé (Custom Locations) :**
   * Ajoutez l'emplacement `/` pointant vers `http://192.168.1.50:8080`.
   * Cliquez sur la roue crantée de cet emplacement pour ouvrir sa configuration avancée et ajoutez les en-têtes de sécurité ainsi que l'injection CSS (`sub_filter`) :

```nginx
proxy_hide_header X-Powered-By;
proxy_hide_header X-Frame-Options;

add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header X-Xss-Protection "1; mode=block" always;
add_header X-Robots-Tag "noindex, noarchive, nofollow" always;
add_header Content-Security-Policy "frame-ancestors 'self' [https://wiki.exemple.com](https://wiki.exemple.com);" always;

# Injection CSS pour harmoniser les couleurs avec Wiki.js
gzip off;
proxy_set_header Accept-Encoding "";
sub_filter_types text/html;
sub_filter_once on;
sub_filter '</body>' '<style>html, body, [class*="root_theme_dark"] { background: transparent !important; } form { background-color: #1e293b !important; border: 1px solid #155e75 !important; border-radius: 8px !important; padding: 12px !important; } form > div { background-color: #1e293b !important; } textarea, form > div:not(:first-child):not(:has(button)):not(:has(textarea)), form > div:not(:first-child):not(:has(button)):not(:has(textarea)) > div { background-color: #0f172a !important; color: #f8fafc !important; border: 1px solid #155e75 !important; border-radius: 6px !important; } form > div:not(:first-child):not(:has(button)):not(:has(textarea)) { padding: 12px !important; margin-top: 12px !important; } form > div:not(:first-child):not(:has(button)):not(:has(textarea)) > div { border: none !important; padding: 0 !important; } form code, form pre { background-color: #1e293b !important; color: #38bdf8 !important; } form button[type="button"], form button[type="submit"], [class*="auth-panel"] button { background-color: #0e7490 !important; border-color: #0e7490 !important; color: #ffffff !important; } form button[type="button"]:hover, form button[type="submit"]:hover, [class*="auth-panel"] button:hover { background-color: #f97316 !important; border-color: #f97316 !important; } a { color: #38bdf8 !important; } a:hover { color: #f97316 !important; }</style></body>';
```

> **Comment fonctionne ce CSS injecté ?**
> * **Transparence du fond :** `background: transparent` supprime le fond gris par défaut de Remark42 afin de laisser apparaître le fond sombre de Wiki.js.
> * **Formulaire et zone de saisie :** `form` et `textarea` reprennent les teintes bleu nuit (`#1e293b` et `#0f172a`) avec une bordure uniforme de `1px` (`#155e75`) qui remplace la bordure épaisse asymétrique d'origine.
> * **Fenêtre de prévisualisation (`Prévisualiser`) :** En production, Remark42 génère des noms de classes aléatoires. Pour styliser le bloc d'aperçu avant publication sans dépendre d'une classe susceptible de changer, le sélecteur structurel `form > div:not(:first-child):not(:has(button)):not(:has(textarea))` cible directement le conteneur de prévisualisation et lui applique le même fond `#0f172a` et la même bordure `#155e75` que la zone de saisie (ainsi que la coloration des extraits de code `code` et `pre`).
> * **Boutons et liens :** Les boutons principaux adoptent la couleur primaire du thème (`#0e7490`) et basculent sur l'accent orange (`#f97316`) au survol. Vous pouvez bien entendu adapter ces codes hexadécimaux à votre propre palette.

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

### Passer l'interface Remark42 en français et rendre le cadre transparent

1. Dans l'administration de Wiki.js, allez dans **Thème**.
2. Repérez la section **Head HTML Injection** et collez le script suivant afin de forcer la langue française tout en conservant la barre d'outils complète de formatage Markdown :

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

> **Note :** Évitez d'ajouter `simple_view: true` dans ce script si vous souhaitez conserver les boutons de mise en forme (gras, italique, code, citations, listes et émojis) au-dessus de la zone de saisie.

3. Toujours dans **Thème**, repérez la section **CSS Override** et ajoutez ce bloc pour garantir que le conteneur de l'iframe reste bien transparent et laisse passer le fond de votre page :

```css
#remark42,
#remark42 iframe {
  background: transparent !important;
}
```

4. Cliquez sur **Appliquer**.
5. **Vider le cache de l'iframe :** Ouvrez `https://remark.exemple.com/web/iframe.html` dans un onglet et faites `Ctrl + F5`, puis rechargez une page publique de votre wiki avec `Ctrl + F5`. L'espace commentaires s'affiche désormais en français et aux couleurs exactes de votre thème.

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

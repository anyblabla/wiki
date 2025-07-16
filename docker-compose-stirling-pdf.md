---
title: Docker Compose Stirling PDF
description: Stirling PDF vous fournit des outils puissants pour gérer vos fichiers PDF. Fusionnez, divisez ou convertissez vos fichiers PDF en toute simplicité.
published: true
date: 2025-07-16T22:15:17.280Z
tags: pdf, stirling
editor: markdown
dateCreated: 2024-06-14T12:54:05.800Z
---

Pratique pour déployer rapidement [Stirling PDF](https://stirlingtools.com) dans [Portainer](https://www.portainer.io/) en créant une pile ([stack](https://docs.portainer.io/user/docker/stacks)) à partir d'un fichier [compose YAML](https://docs.docker.com/compose/compose-application-model/).

Je pars du principe que vous maîtrisez un minimum Docker (avec Portainer) 😉

# Stirling PDF, c'est quoi ?

Stirling PDF vous fournit des outils puissants pour gérer vos fichiers PDF. Fusionnez, divisez ou convertissez vos fichiers PDF en toute simplicité.

La sécurité de vos fichiers est la priorité. Stirling PDF ne conserve aucun fichier, suivi ou donné. Il fonctionne entièrement sur votre machine locale, garantissant la confidentialité et le contrôle de vos données.

Stirling-PDF est conçu dans un souci d'orientation utilisateur. L'interface, le nom de l'application et la description sont personnalisables, vous permettant d'ajuster les paramètres en fonction de vos préférences et de vos besoins.

-   [Site officiel](https://stirlingtools.com)
-   [GitHub](https://github.com/Stirling-Tools/Stirling-PDF)
-   [Docker Hub](https://hub.docker.com/r/frooodle/s-pdf)

# Docker Compose

```plaintext
version: '3.3'
services:
  stirling-pdf:
    image: frooodle/s-pdf:latest
    restart: always
    ports:
      - '8080:8080'
    volumes:
      - /usr/share/tessdata:/usr/share/tessdata #Required for extra OCR languages
      - /configs:/configs
#      - /location/of/customFiles:/customFiles/
#      - /location/of/logs:/logs/
    environment:
      - DOCKER_ENABLE_SECURITY=false
      - INSTALL_BOOK_AND_ADVANCED_HTML_OPS=false
      - LANGS=fr_FR
```

Fichier compose également disponible sur [ByteStash Blabla Linux](https://bytestash.blablalinux.be/public/snippets).

# Personnalisation / Configuration

On peut ajouter des variables d'environnement au fichier compose, ou alors, modifier le fichier “settings.yml” qui se trouve dans “/configs”. Les variables d'environnement prévalent toujours sur le fichier de configuration.

## Tags

Il existe plusieurs images. 

-   J'ai utilisé ici l'image avec la dernière version stable…

`frooodle/s-pdf:latest`

Cette image reprend quasi toutes les fonctionnalités. Son poids compressé est de +ou- 700 MB.

-   Si le stockage est un souci pour vous, vous pouvez utiliser une image plus légère, mais alors avec des fonctionnalités en moins. Son poids compressé est de +ou- 250 MB…

`frooodle/s-pdf:latest-ultra-lite`

À contrario, si l'espace de stockage n'est pas un problème, il existe une image lourde qui reprend la totalité des fonctionnalités. Son poids est de +ou- 1050 MB… 

`frooodle/s-pdf:latest-fat`

## Interface (UI)

-   Pour personnaliser un peu notre interface Stirling PDF, avec un nom et une description, on peut ajouter ces variables…

```plaintext
- UI_APP_NAME=Blabla Linux Stirling PDF
- UI_HOME_DESCRIPTION=Boîte à outils de gestion PDF libre et open source, auto hébergé par Blabla Linux grâce à Stirling..
- UI_APP_NAVBAR_NAME=Blabla Linux Stirling PDF
```

-   Ou alors, éditer le fichier “settings.yml” dans “/configs”…

```plaintext
nano /configs/settings
```

-   Voici la section qui correspond à nos trois nouvelles variables…

```plaintext
ui:
  appName: null # Application's visible name
  homeDescription: null # Short description or tagline shown on homepage.
  appNameNavbar: null # Name displayed on the navigation bar
```

## Fichier de configuration par défaut

-   Voici le fichier de configuration dans son ensemble après l'installation…

```plaintext
# Welcome to settings file
# Remove comment marker # if on start of line to enable the configuration
# If you want to override with environment parameter follow parameter naming SECURITY_INITIALLOGIN_USERNAME

security:
  enableLogin: false # set to 'true' to enable login
  csrfDisabled: true # Set to 'true' to disable CSRF protection (not recommended for production)
  loginAttemptCount: 5 # lock user account after 5 tries
  loginResetTimeMinutes : 120 # lock account for 2 hours after x attempts
#  initialLogin:
#    username: "admin" # Initial username for the first login
#    password: "stirling" # Initial password for the first login
#  oauth2:
#    enabled: false # set to 'true' to enable login (Note: enableLogin must also be 'true' for this to work)
#    issuer: "" # set to any provider that supports OpenID Connect Discovery (/.well-known/openid-configuration) end-point
#    clientId: "" # Client ID from your provider
#    clientSecret: "" # Client Secret from your provider
#    autoCreateUser: false # set to 'true' to allow auto-creation of non-existing users
#    useAsUsername: "email" # Default is 'email'; custom fields can be used as the username
#    scopes: "openid, profile, email" # Specify the scopes for which the application will request permissions
#    provider: "google" # Set this to your OAuth provider's name, e.g., 'google' or 'keycloak'

system:
  defaultLocale: 'en-US' # Set the default language (e.g. 'de-DE', 'fr-FR', etc)
  googlevisibility: false # 'true' to allow Google visibility (via robots.txt), 'false' to disallow
  enableAlphaFunctionality: false # Set to enable functionality which might need more testing before it fully goes live (This feature might make no changes)
  showUpdate: true # see when a new update is available
  showUpdateOnlyAdmin: false # Only admins can see when a new update is available, depending on showUpdate it must be set to 'true'
  customHTMLFiles: false # enable to have files placed in /customFiles/templates override the existing template html files

ui:
  appName: null # Application's visible name
  homeDescription: null # Short description or tagline shown on homepage.
  appNameNavbar: null # Name displayed on the navigation bar

endpoints:
  toRemove: [] # List endpoints to disable (e.g. ['img-to-pdf', 'remove-pages'])
  groupsToRemove: [] # List groups to disable (e.g. ['LibreOffice'])

metrics:
  enabled: true # 'true' to enable Info APIs (`/api/*`) endpoints, 'false' to disable

# Automatically Generated Settings (Do Not Edit Directly)
AutomaticallyGenerated:
  key: example
```

## Écran de connexion

Par défaut, Stirling PDF sera accessible sans identifiants. Si vous exposez Stirling PDF sur le web, toutes les personnes disposant de votre (sous)domaine ou de votre IP publique, pourra utiliser votre instance Stirling PDF. Comme je l'ai fait moi.

-   Pour activer la connexion à Stirling PDF, on commence par passer cette variable dans le compose de “false”…

`DOCKER_ENABLE_SECURITY=false`

-   …à “true”…

`DOCKER_ENABLE_SECURITY=true`

-   Ensuite dans le fichier de configuration, on passe cette ligne de “false”…

`enableLogin: false`

-   …à “true”…

`enableLogin: true`

-   Pour initialiser la première connexion, on décommande ces deux lignes…

`#    username: "admin" # Initial username for the first login`

`#    password: "stirling" # Initial password for the first login`

-   …pour obtenir ceci…

`username: "admin" # Initial username for the first login`

`password: "stirling" # Initial password for the first login`

-   On aurait pu aussi directement ajouter des variables au compose…

`- SECURITY_INITIALLOGIN_USERNAME=admin`

`- SECURITY_INITIALLOGIN_PASSWORD=stirling`

## Locales

-   Il faut maintenant passer les “locales” en français avec cette ligne en passant de “en-US”…

`defaultLocale: 'en-US' # Set the default language (e.g. 'de-DE', 'fr-FR', etc)`

-   …à “fr-FR”…

`defaultLocale: 'fr-FR' # Set the default language (e.g. 'de-DE', 'fr-FR', etc)`

## Référencement

-   Pour être référencé par Google, il faut passer cette ligne de “false”…

`googlevisibility: false # 'true' to allow Google visibility (via robots.txt), 'false' to disallow`

-   …à “true”…

`googlevisibility: true # 'true' to allow Google visibility (via robots.txt), 'false' to disallow`

## Reconnaissance OCR

Stirling est capable d'effectuer de la reconnaissance de caractères (OCR). Par défaut, il gère l'anglais, et uniquement l'anglais. Il va falloir lui donner d'autres fichiers, selon les langues que l'on souhaite faire reconnaître.

Il existe des fichiers de reconnaissance de langue légers, rapide à charger, mais dont la reconnaissance ne sera pas parfaite.

Ou il existe des fichiers de reconnaissance de langue plus lourds, donc plus lents à charger, mais la reconnaissance sera bien meilleure.

Je suis gentil, je vous fournis ces fichiers grâce au [cloud Blabla Linux](https://yourls.blablalinux.be/nextcloud).

-   [Fichiers légers](https://nextcloud.blablalinux.be/index.php/s/4ezDSHy3XoTZARb)
-   [Fichiers lourds](https://nextcloud.blablalinux.be/index.php/s/bPp4C7YXtTeKpXt)

Vous téléchargez le fichier, ou les fichiers, ou tous les fichiers dont vous avez besoin, vous les placerez ensuite dans le répertoire “tessdata” qui se trouve dans “/usr/share”.

![](/docker-compose-stirling-pdf/stirling-pdf-bbl-ocr.png)

-   D'où cette ligne dans le compose…

`- /usr/share/tessdata:/usr/share/tessdata #Required for extra OCR languages`

On ne supprime pas le fichier “**eng.traineddata**” ! Vous pouvez le remplacer par une autre version (légère/lourde), mais ce fichier doit rester en place. Stirling PDF en a besoin.

# Explications

Je n'ai abordé ici que la méthode installation via Docker et fichier compose. Simplement parce que c'est la méthode que j'ai utilisée. Je n'ai abordé que quelques points de paramétrages, pas tous.

Si vous préférez une autre méthode d'installation, et/ou aller plus loin dans le paramétrage, [la documentation](https://yourls.blablalinux.be/stirling-doc) est là ! Bonne lecture 😉

# Minimum requis

Il faut savoir que 1 GO de mémoire vive suffise à faire tourner Stirling PDF, au repos.

Une fois que vous allez utiliser certaines fonctionnalités comme l'extraction d'images sur plusieurs fichiers PDF, les besoins en mémoire vive peuvent s'envoler ! 

Si les ressources en mémoire vive sont insuffisantes lors de l'utilisation d'une fonctionnalité, vous aurez droit un beau message d'erreur. Mais, Stirling PDF ne freeze pas. Il suffit de rafraîchir la page et c'est OK.

Personnellement, mon instance tourne sur Docker (Portainer) dans un LXC Proxmox avec 2 cœurs de processeur, 8 GO de mémoire vice et un espace d'échange de 2 GO.

# Captures de Stirling PDF Blabla Linux

-   [https://yourls.blablalinux.be/stirlingpdf](https://yourls.blablalinux.be/stirlingpdf)

![](/docker-compose-stirling-pdf/stirling-pdf-bbl-outils.png)

Stirling PDF Blabla Linux - Outils
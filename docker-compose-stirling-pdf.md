---
title: Stirling PDF V1 (docker-compose)
description: Ce tutoriel explique comment déployer rapidement Stirling PDF V1 (outil puissant de gestion PDF) en utilisant une pile Docker (stack) dans Portainer à partir d'un fichier compose YAML.
published: true
date: 2025-11-26T14:44:21.685Z
tags: docker, pdf, stirling, compose, tools
editor: markdown
dateCreated: 2024-06-14T12:54:05.800Z
---

## 1\. Stirling PDF, qu'est-ce que c'est ?

Stirling PDF est une boîte à outils pour fichiers PDF permettant de fusionner, diviser, convertir et plus encore. L'application met l'accent sur la **sécurité et la confidentialité** : elle ne conserve aucun fichier, suivi ou donnée, et fonctionne entièrement sur votre machine locale. L'interface, le nom et la description sont personnalisables.

### Liens utiles

  * [Site officiel](https://stirlingtools.com)
  * [GitHub](https://github.com/Stirling-Tools/Stirling-PDF)
  * [Docker Hub](https://hub.docker.com/r/stirlingtools/stirling-pdf)

-----

## 2\. Fichier `docker-compose.yml` de base

Le fichier Compose de base pour le déploiement de Stirling PDF :

> ⚠️ **Note importante sur l'image Docker :**
> L'image recommandée a été mise à jour de `frooodle/s-pdf` à **`stirlingtools/stirling-pdf`**.
> La version **`1.6.0-fat`** est la toute dernière et la plus complète de la **V1** de Stirling PDF.

```yaml
version: '3.3'
services:
  stirling-pdf:
    image: stirlingtools/stirling-pdf:1.6.0-fat # ⚠️ Nouvelle image et tag V1
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

Fichier compose également disponible sur [ByteStash Blabla Linux](https://bytestash.blablalinux.be/s/f1254114dd45f18e1aba759566f4fc29).

### Explications des volumes

| Volume | Chemin conteneur | Description |
| :--- | :--- | :--- |
| **OCR** | `/usr/share/tessdata` | **Obligatoire** pour ajouter des langues supplémentaires pour la reconnaissance de caractères (OCR). |
| **Configs** | `/configs` | Contient le fichier de configuration principal **`settings.yml`**. |

-----

## 3\. Personnalisation et configuration

La configuration peut se faire soit via des **variables d'environnement** (qui priment toujours), soit en modifiant le fichier **`settings.yml`** situé dans le volume `/configs`.

### Choix de l'image Docker (tags)

Plusieurs tags sont disponibles en fonction de la taille et des fonctionnalités souhaitées. La version **1.6.0** est la dernière **V1**.

| Tag | Poids (compressé) | Fonctionnalités | Utilisation |
| :--- | :--- | :--- | :--- |
| **`1.6.0-fat`** | $\approx 1050$ MB | **Totalité** des fonctionnalités. | **Recommandé** (celui utilisé dans le Compose). |
| **`1.6.0-lite`** | $\approx 700$ MB | Quasi toutes les fonctionnalités. | Si l'espace est une légère contrainte. |
| **`1.6.0-ultra-lite`** | $\approx 250$ MB | Moins de fonctionnalités (plus léger). | Idéal si le stockage est une contrainte. |

### Configuration de l'interface utilisateur (UI)

Vous pouvez personnaliser le nom et la description de l'application en ajoutant des variables d'environnement au service `stirling-pdf` :

```plaintext
    environment:
      # ... autres variables ...
      - UI_APP_NAME=Blabla Linux Stirling PDF
      - UI_HOME_DESCRIPTION=Boîte à outils de gestion PDF libre et open source, auto hébergé par Blabla Linux grâce à Stirling..
      - UI_APP_NAVBAR_NAME=Blabla Linux Stirling PDF
```

*(Ces variables remplacent les valeurs `null` dans la section `ui` du fichier `settings.yml`.)*

### Activation de la sécurité et de la connexion

Par défaut, l'application est accessible sans identifiant. Pour activer un écran de connexion (essentiel si exposé sur le web) :

1.  **Variable Compose :** Passer la variable d'environnement de `false` à **`true`** :
    ```plaintext
    - DOCKER_ENABLE_SECURITY=true
    ```
2.  **Fichier `settings.yml` :** Modifier la ligne `security.enableLogin` de `false` à **`true`**.
    ```yaml
    security:
      enableLogin: true # set to 'true' to enable login
      # ...
    ```
3.  **Identifiants initiaux :** Dé-commenter et personnaliser les lignes suivantes dans `settings.yml` (ou utiliser des variables d'environnement) :
    ```yaml
    # settings.yml
    #  initialLogin:
      username: "admin" # Initial username for the first login
      password: "stirling" # Initial password for the first login
    ```
    *Alternative (variables d'environnement) :*
    ```plaintext
    - SECURITY_INITIALLOGIN_USERNAME=admin
    - SECURITY_INITIALLOGIN_PASSWORD=stirling
    ```

### Configuration de la langue (locale)

Pour passer l'interface en français, modifiez la ligne `defaultLocale` dans le fichier `settings.yml` :

```yaml
system:
  defaultLocale: 'fr-FR' # Set the default language (e.g. 'de-DE', 'fr-FR', etc)
```

### Ajout de langues pour la reconnaissance OCR

Par défaut, seul l'anglais est géré. Pour ajouter d'autres langues :

1.  **Téléchargez** les fichiers de reconnaissance de langue (`.traineddata`) nécessaires (fichiers légers ou lourds) depuis la documentation.

Je suis gentil, je vous fournis ces fichiers grâce au [cloud Blabla Linux](https://yourls.blablalinux.be/nextcloud).

  * [Fichiers légers](https://nextcloud.blablalinux.be/index.php/s/4ezDSHy3XoTZARb)
  * [Fichiers lourds](https://nextcloud.blablalinux.be/index.php/s/bPp4C7YXtTeKpXt)

<!-- end list -->

2.  **Placez** ces fichiers dans le répertoire de votre hôte que vous avez monté sur `/usr/share/tessdata` (ici : `/usr/share/tessdata`).
    > **Note :** Ne supprimez pas le fichier **`eng.traineddata`**, Stirling PDF en a besoin.
    
![stirling-pdf-bbl-ocr.png](/docker-compose-stirling-pdf/stirling-pdf-bbl-ocr.png)    

-----

## 4\. Lancement et minimum requis

Une fois configuré, lancez votre pile :

```bash
docker compose up -d
```

### Ressources

  * **Mémoire vive au repos :** Minimum **1 Go**.
  * **Mémoire vive en usage :** Certaines fonctionnalités gourmandes (extraction d'images, gros fichiers) peuvent faire s'envoler les besoins en RAM. Si les ressources sont insuffisantes, vous aurez un message d'erreur, mais l'application ne "freeze" pas ; il suffit de rafraîchir la page.

Vous accédez à l'interface à l'adresse **`http://<Votre_IP_Hôte>:8080`**.

![stirling-pdf-bbl-outils.png](/docker-compose-stirling-pdf/stirling-pdf-bbl-outils.png)

[Stirling PDF Blabla Linux - Outils](https://yourls.blablalinux.be/stirlingpdf)
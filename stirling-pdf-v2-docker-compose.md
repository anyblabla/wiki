---
title: Stirling PDF V2 (docker-compose)
description: Déploiement de Stirling PDF V2 en auto-hébergement avec Docker Compose. Ce guide utilise l'image latest-fat pour toutes les fonctionnalités, le healthcheck, et les options de sécurité avancées (OAuth2, API).
published: true
date: 2025-11-26T14:41:02.102Z
tags: docker, pdf, stirling, compose, v2
editor: markdown
dateCreated: 2025-11-26T14:41:02.102Z
---

## 1\. Stirling PDF, qu'est-ce que c'est ?

Stirling PDF est une boîte à outils pour fichiers PDF permettant de fusionner, diviser, convertir et plus encore. L'application met l'accent sur la **sécurité et la confidentialité** : elle ne conserve aucun fichier, suivi ou donnée, et fonctionne entièrement sur votre machine locale. L'interface, le nom et la description sont personnalisables.

> 💡 **Version V2 :** La version 2 apporte des améliorations majeures au niveau de l'architecture, une meilleure gestion des utilisateurs, l'intégration potentielle d'une base de données externe et des fonctionnalités avancées (OAuth2, Google Drive, etc.).

### Liens utiles

  * [Site officiel](https://stirlingtools.com)
  * [GitHub](https://github.com/Stirling-Tools/Stirling-PDF)
  * [Docker Hub](https://hub.docker.com/r/stirlingtools/stirling-pdf)

-----

## 2\. Fichier `docker-compose.yml` de base (V2)

Le fichier Compose de base pour le déploiement de Stirling PDF V2. Nous utilisons le tag `latest-fat` pour obtenir la totalité des fonctionnalités de la dernière version stable.

```yaml
services:
  stirling-pdf:
    image: stirlingtools/stirling-pdf:latest-fat
    container_name: stirling-pdf-generic
    restart: always
    healthcheck: # 🆕 Ajout du Healthcheck V2
      test:
        - CMD-SHELL
        - curl -f http://localhost:8080/api/v1/info/status | grep -q 'UP' &&
          curl -fL http://localhost:8080/ | grep -qv 'Please sign in'
      interval: 5s
      timeout: 10s
      retries: 16
    ports:
      - 8080:8080
    volumes:
      - ./tessdata:/usr/share/tessdata
      - ./configs:/configs
      - ./logs:/logs # 🆕 Volume pour les logs
      - ./pipeline:/pipeline # 🆕 Volume pour les tâches de pipeline
    # Si vous utilisez une base de données externe (décommenter la section `db` ci-dessous):
    #depends_on:
      #- db
    environment:
      - DISABLE_ADDITIONAL_FEATURES=false # Gère les fonctionnalités non essentielles
      - SECURITY_ENABLELOGIN=false
      # Informations de connexion initiales (décommenter si `SECURITY_ENABLELOGIN` est 'true' ou manuellement activé)
      #- SECURITY_INITIALLOGIN_USERNAME=your_username
      #- SECURITY_INITIALLOGIN_PASSWORD=your_secure_password
      - LANGS=fr_FR
      - SYSTEM_DEFAULTLOCALE=fr_FR
      - MODE=BOTH # Mode de fonctionnement : UI + API
      - SYSTEM_MAXFILESIZE=250
      - SPRING_SERVLET_MULTIPART_MAX_FILE_SIZE=250MB
      - SPRING_SERVLET_MULTIPART_MAX_REQUEST_SIZE=250MB
      - JAVA_TOOL_OPTIONS="-Xms512m -Xmx6g" # Ajustement des ressources Java
      # Clé API (décommenter pour l'activer)
      #- X-API-KEY=your_api_key_here
      
      # Configuration de la messagerie (décommenter pour l'activer)
      #- MAIL_ENABLED=true
      #- MAIL_ENABLEINVITES=true
      #- MAIL_SMTP_HOST=smtp.example.com
      #- MAIL_SMTP_PORT=587 
      #- MAIL_SMTP_USERNAME=your_email@example.com
      #- MAIL_SMTP_PASSWORD=your_smtp_password
      #- MAIL_SMTP_TLS_ENABLED=true
      
      # Paramètres OAuth2 (décommenter et remplir pour l'activer)
      #- SECURITY_ENABLELOGIN=true
      #- SECURITY_OAUTH2_ENABLED=true
      #- SECURITY_OAUTH2_CLIENTID=your_oauth2_client_id
      #- SECURITY_OAUTH2_CLIENTSECRET=your_oauth2_client_secret
      #- SECURITY_OAUTH2_ISSUER=your_oauth2_issuer_uri
      
      # Paramètres Google Drive (décommenter et remplir pour l'activer)
      #- PREMIUM_PRO_FEATURES_GOOGLE_DRIVE_ENABLED=true
      #- PREMIUM_PRO_FEATURES_GOOGLE_DRIVE_CLIENT_ID=your_google_client_id
      #- PREMIUM_PRO_FEATURES_GOOGLE_DRIVE_API_KEY=your_google_api_key
      #- PREMIUM_PRO_FEATURES_GOOGLE_DRIVE_APP_ID=your_google_app_id

      - UI_LOGOSTYLE=modern
      - SYSTEM_SHOWUPDATE=true
      - SYSTEM_SHOWUPDATEONLYADMIN=true
      - ALLOW_GOOGLE_VISIBILITY=true # Permettre l'accès à Google Visibility

# Base de données PostgreSQL (décommenter pour utiliser une base de données locale)
#db:
  #image: postgres:17.2-alpine
  #container_name: db
  #restart: always
  #ports:
    #- 5432:5432
  #environment:
    #- POSTGRES_DB=stirling_pdf
    #- POSTGRES_USER=admin
    #- POSTGRES_PASSWORD=stirling # À changer pour un mot de passe sécurisé en production
```

### Explications des volumes

| Volume | Chemin conteneur | Description |
| :--- | :--- | :--- |
| **OCR** | `/usr/share/tessdata` | **Obligatoire** pour ajouter des langues supplémentaires pour la reconnaissance de caractères (OCR). |
| **Configs** | `/configs` | Contient le fichier de configuration principal **`settings.yml`**. |
| **Logs** | `/logs` | **Nouveau \!** Contient les fichiers journaux (logs) de l'application. |
| **Pipeline** | `/pipeline` | **Nouveau \!** Utilisé pour le traitement des tâches asynchrones et l'automatisation. |

-----

## 3\. Personnalisation et configuration

La configuration peut se faire soit via des **variables d'environnement** (qui priment toujours), soit en modifiant le fichier **`settings.yml`** situé dans le volume `/configs`.

### Choix de l'image Docker (tags)

La version V2 utilise des tags simples pour les dernières versions :

| Tag | Fonctionnalités | Utilisation |
| :--- | :--- | :--- |
| **`latest-fat`** | **Totalité** des fonctionnalités, y compris OCR. | **Recommandé** (celui utilisé dans le Compose). |
| **`latest-lite`** | Moins de fonctionnalités (plus léger). | Si l'espace est une contrainte. |
| **`latest`** | Alias de `latest-lite` ou `latest-fat` selon les dernières conventions. | À utiliser avec prudence ; préférez `fat` ou `lite`. |

### 🛠️ Configuration des ressources et performance

Le nouveau Compose utilise la variable `JAVA_TOOL_OPTIONS` pour allouer des ressources à la machine virtuelle Java :

  * `-Xms512m` : Mémoire minimale de **512 Mo**.
  * `-Xmx6g` : Mémoire maximale de **6 Go**.

Vous pouvez ajuster ces valeurs en fonction de vos ressources hôtes pour améliorer les performances sur les tâches gourmandes (gros fichiers, OCR intensif).

### Activation de la sécurité et de la connexion

Pour activer la connexion (utilisateur/mot de passe ou OAuth2) :

1.  **Variable Compose :** Passer la variable d'environnement `SECURITY_ENABLELOGIN` de `false` à **`true`**.
    ```plaintext
    - SECURITY_ENABLELOGIN=true
    ```
2.  **Identifiants initiaux :** Dé-commenter et personnaliser les lignes suivantes dans `environment` :
    ```plaintext
    - SECURITY_INITIALLOGIN_USERNAME=your_username
    - SECURITY_INITIALLOGIN_PASSWORD=your_secure_password
    ```

> **Note :** La V2 gère également l'intégration **OAuth2** (Google, Azure, etc.) en décommentant et configurant les variables `SECURITY_OAUTH2_*`.

### Configuration de la langue (locale)

Le Compose V2 utilise la variable d'environnement pour définir la locale de l'interface :

```plaintext
- SYSTEM_DEFAULTLOCALE=fr_FR
```

### 🗄️ Gestion de la taille des fichiers

La taille maximale des fichiers est gérée par plusieurs variables dans la V2 (définies ici à **250 Mo**) :

```plaintext
- SYSTEM_MAXFILESIZE=250
- SPRING_SERVLET_MULTIPART_MAX_FILE_SIZE=250MB
- SPRING_SERVLET_MULTIPART_MAX_REQUEST_SIZE=250MB
```

### Ajout de langues pour la reconnaissance OCR

Par défaut, seul l'anglais est géré. Pour ajouter d'autres langues :

1.  **Téléchargez** les fichiers de reconnaissance de langue (`.traineddata`) nécessaires (fichiers légers ou lourds) depuis la documentation.

Je suis gentil, je vous fournis ces fichiers grâce au [cloud Blabla Linux](https://yourls.blablalinux.be/nextcloud).

  - [Fichiers légers](https://nextcloud.blablalinux.be/index.php/s/4ezDSHy3XoTZARb)
  - [Fichiers lourds](https://nextcloud.blablalinux.be/index.php/s/bPp4C7YXtTeKpXt)

<!-- end list -->

2.  **Placez** ces fichiers dans le répertoire de votre hôte que vous avez monté sur `/usr/share/tessdata` (ici : `./tessdata`).
    > **Note :** Ne supprimez pas le fichier **`eng.traineddata`**, Stirling PDF en a besoin.
    
![stirling-pdf-bbl-ocr.png](/docker-compose-stirling-pdf/stirling-pdf-bbl-ocr.png)    

-----

## 4\. Lancement et minimum requis

Une fois configuré, lancez votre pile :

```bash
docker compose up -d
```

### Ressources

  * **Mémoire vive au repos :** Minimum **1 Go**. L'utilisation de `-Xms512m` et des fonctionnalités V2 exige une bonne allocation.
  * **Mémoire vive en usage :** Les fonctionnalités gourmandes peuvent toujours demander des pics de RAM. Ajustez la variable `-Xmx` (mémoire max.) si vous rencontrez des erreurs de manque de mémoire (`OOM`).

Vous accédez à l'interface à l'adresse **`http://<Votre_IP_Hôte>:8080`**.

![stirling-pdf-v2-bbl-outils.png](/docker-compose-stirling-pdf-v2/stirling-pdf-v2-bbl-outils.png)

[Stirling PDF Blabla Linux - Outils](https://yourls.blablalinux.be/stirlingpdf)
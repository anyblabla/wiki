---
title: Installation de VirusTotal Uploader (GUI) sur Debian/Ubuntu/Mint
description: Ce guide explique comment installer l'outil graphique QT VirusTotal Uploader sur Linux Mint 21.3 (base Ubuntu 22.04) en compilant les sources.
published: true
date: 2025-10-28T14:29:48.474Z
tags: virus, virustotal, uploader
editor: markdown
dateCreated: 2024-06-12T18:14:23.441Z
---

**VirusTotal Uploader** est l'application de bureau officielle pour soumettre des fichiers, des domaines, des adresses IP et des URL suspects à la plateforme **VirusTotal** pour une analyse par de multiples moteurs antivirus.

![](/virus-total-uploader/virus-total-uploader-compte-amaury-libert.png)


## 1\. Liens Utiles

| Catégorie | Lien |
| :--- | :--- |
| **Site Officiel** | [VirusTotal (Outil en ligne)](https://www.virustotal.com/gui/home/upload) |
| **Dépôt GitHub** | [VirusTotal/qt-virustotal-uploader](https://github.com/VirusTotal/qt-virustotal-uploader) |
| **Wikipédia** | [VirusTotal](https://fr.wikipedia.org/wiki/VirusTotal) |

-----

## 2\. Installation des Dépendances

L'installation nécessite les outils de compilation et plusieurs bibliothèques, notamment QT5 et `libjansson-dev`/`libcurl`.

1.  **Mettre à jour les dépôts :**

    ```bash
    sudo apt update
    ```

2.  **Installer les dépendances de compilation (selon votre environnement) :**

      * **Linux Mint Cinnamon (ou Ubuntu/Debian standard) :**
        ```bash
        sudo apt-get install build-essential qtchooser qt5-default libjansson-dev libcurl4-openssl-dev git zlib1g-dev
        ```
      * **Linux Mint XFCE :**
        ```bash
        sudo apt install qtbase5-dev qtchooser qt5-qmake qtbase5-dev-tools build-essential libjansson-dev libcurl4-openssl-dev git zlib1g-dev
        ```

-----

## 3\. Compilation de la Bibliothèque Dépendante `c-vtapi`

L'application VirusTotal Uploader dépend de la bibliothèque C **`c-vtapi`**, qui doit être compilée et installée en premier.

1.  **Cloner le dépôt et y accéder :**

    ```bash
    git clone https://github.com/VirusTotal/c-vtapi.git
    cd c-vtapi
    ```

2.  **Installer les dépendances supplémentaires pour `c-vtapi` :**

    ```bash
    sudo apt-get install automake autoconf libtool libjansson-dev libcurl4-openssl-dev
    ```

3.  **Configurer et compiler la bibliothèque :**

    ```bash
    autoreconf -fi && ./configure && make
    ```

4.  **Installer la bibliothèque sur le système (dans `/usr/local/lib`) :**

    ```bash
    sudo make install
    ```

5.  **Mettre à jour l'éditeur de liens dynamique** pour que le système trouve la nouvelle bibliothèque :

    ```bash
    sudo sh -c 'echo "/usr/local/lib" > /etc/ld.so.conf.d/usr-local-lib.conf'
    sudo ldconfig
    ```

6.  **Revenir au répertoire de base :**

    ```bash
    cd ..
    ```

-----

## 4\. Compilation et Lancement de VirusTotal Uploader

Maintenant que la dépendance est installée, vous pouvez compiler le programme principal.

1.  **Cloner le dépôt de l'Uploader et y accéder :**

    ```bash
    git clone https://github.com/VirusTotal/qt-virustotal-uploader.git
    cd qt-virustotal-uploader
    ```

2.  **Préparer la compilation avec `qmake` (pour QT5) :**

    ```bash
    qtchooser -run-tool=qmake -qt=5
    ```

3.  **Compiler le programme :**

    ```bash
    make -j4
    ```

4.  **Lancer le logiciel (nécessite `sudo` pour l'exécution directe si vous n'avez pas installé dans un chemin utilisateur) :**

    ```bash
    sudo ./VirusTotalUploader
    ```

    *C'est à vous de créer ensuite un lanceur d'application dans votre menu Cinnamon/XFCE pour un accès facile.*

-----

## 5\. Configuration (Optionnelle mais Recommandée)

Pour une utilisation optimale et des analyses plus efficaces, vous devriez configurer votre **Clé API personnelle** VirusTotal.

1.  **Obtenir votre Clé API :**
      * Créez un compte ou connectez-vous sur [VirusTotal](https://www.virustotal.com/gui/home/upload).
      * Cliquez sur votre profil en haut à droite, puis choisissez **"API Key"** pour copier votre clé.
2.  **Ajouter la Clé dans l'Application :**
      * Dans le logiciel **VirusTotal Uploader**, allez dans la barre d'outils : **"Files"** =\> **"Preferences"**.
      * Collez votre Clé API personnelle dans le champ dédié.

Vous pouvez maintenant glisser et déposer des fichiers ou dossiers dans l'application pour les analyser.

![](/virus-total-uploader/virus-total-uploader-api-key.png)

![](/virus-total-uploader/virus-total-uploader-software-api-key.png)

![](/virus-total-uploader/virus-total-uploader-software-file-upload.gif)
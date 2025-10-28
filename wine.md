---
title: Installation de WineHQ sur Linux Mint / Debian (Stable, Dev, Staging)
description: Ce guide fournit la procédure pour installer la dernière version de Wine (Stable, Development ou Staging) directement depuis le dépôt officiel WineHQ sur les systèmes basés sur Debian/Ubuntu/Linux Mint (testé et fonctionnel sur Linux Mint 22.x).
published: true
date: 2025-10-28T14:46:19.895Z
tags: wine, émulateur, émulation, qemu
editor: markdown
dateCreated: 2024-05-05T20:15:53.231Z
---

## 1\. Préparation de l'Architecture

Wine a besoin du support 32 bits (`i386`) pour exécuter la plupart des programmes Windows.

1.  **Vérifiez l'architecture de base (doit répondre `amd64`) :**

    ```bash
    dpkg --print-architecture
    ```

2.  **Vérifiez l'architecture étrangère (doit répondre `i386`) :**

    ```bash
    dpkg --print-foreign-architectures
    ```

3.  **Ajouter le support 32 bits** si `i386` est absent :

    ```bash
    sudo dpkg --add-architecture i386
    ```

4.  **Vérifiez à nouveau** que les deux architectures (`amd64` et `i386`) sont prises en charge :

    ```bash
    dpkg --print-foreign-architectures
    ```

-----

## 2\. Ajout du Dépôt WineHQ

Nous allons ajouter la clé GPG et les sources officielles de WineHQ pour garantir une installation à jour et sécurisée.

1.  **Créer le répertoire des clés APT** avec les permissions appropriées :

    ```bash
    sudo mkdir -pm755 /etc/apt/keyrings
    ```

2.  **Télécharger et ajouter la clé GPG** du référentiel WineHQ :

    ```bash
    sudo wget -O /etc/apt/keyrings/winehq-archive.key https://dl.winehq.org/wine-builds/winehq.key
    ```

3.  **Télécharger le fichier des sources** (`winehq-jammy.sources`) :

    ```bash
    sudo wget -NP /etc/apt/sources.list.d/ https://dl.winehq.org/wine-builds/ubuntu/dists/jammy/winehq-jammy.sources
    ```

4.  **Mettre à jour la base de données des paquets :**

    ```bash
    sudo apt update
    ```

-----

## 3\. Installation de la Version Souhaitée

Choisissez et exécutez **une seule** des commandes d'installation ci-dessous :

| Version | Description | Commande d'Installation |
| :--- | :--- | :--- |
| **Stable** | Recommandée pour la plupart des utilisateurs. | `sudo apt install --install-recommends winehq-stable` |
| **Development** | Version de développement (plus récente, mais potentiellement moins stable). | `sudo apt install --install-recommends winehq-devel` |
| **Staging** | Version expérimentale (inclut des correctifs et fonctionnalités non encore dans *Development*). | `sudo apt install --install-recommends winehq-staging` |

-----

## 4\. Finalisation et Configuration

1.  **Vérifiez que l'installation a réussi** et obtenez le numéro de version :

    ```bash
    wine --version
    ```

2.  **Configurez Wine** (création du répertoire `~/.wine` et lancement de l'interface de configuration) :

    ```bash
    wine winecfg
    ```

3.  **Testez** l'exécution d'une application de base :

    ```bash
    wine clock
    ```

4.  **Installer le lanceur d'application (Mint-Menu)** :
    Pour créer un menu "Wine" dans le menu principal de Linux Mint, vous pouvez installer ce paquet :

    ```bash
    sudo apt install wine-installer
    ```

<p style="text-align: center"><img src="/wine/wine-mint-menu.png"></p>

-----

**Démonstration en vidéo :** [https://peertube.blablalinux.be/w/13JYaBK9TFASrHPwYfwBAT](https://peertube.blablalinux.be/w/13JYaBK9TFASrHPwYfwBAT)
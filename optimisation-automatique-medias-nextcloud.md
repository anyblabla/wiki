---
title: Optimisation automatique des médias sur Nextcloud
description: Apprenez à compresser automatiquement vos photos et vidéos sur Nextcloud. Un guide pas à pas pour optimiser l'espace disque de votre serveur reconditionné sans sacrifier sa fluidité habituelle.
published: true
date: 2026-03-02T11:49:21.584Z
tags: cron, crontab, script, bash, ffmpeg, auto-hébergement, optimisation, nextcloud, reconditionnement, imagemagick
editor: markdown
dateCreated: 2026-03-02T11:46:46.923Z
---

Ce guide vous permet de mettre en place un système de compression automatique pour vos photos et vidéos. L'objectif est de réduire l'espace disque consommé par vos sauvegardes mobiles tout en préservant la réactivité de votre serveur, même sur du matériel reconditionné.

## 📋 Fonctionnement du système

1. **Upload** : vous envoyez un média depuis votre smartphone.
2. **Déclencheur (workflow)** : Nextcloud détecte l'arrivée du fichier et appelle un script Bash.
3. **Traitement (script)** : le serveur compresse le média en arrière-plan avec une priorité CPU très basse.
4. **Mise à jour** : Nextcloud scanne le nouveau fichier pour actualiser sa taille dans l'interface.

---

## 🛠️ Étape 1 : installation des prérequis

### Côté serveur (SSH)

Connectez-vous en root et installez les outils de traitement :

```bash
apt update && apt install ffmpeg imagemagick -y

```

* **FFmpeg** : pour la compression des vidéos (H.264).
* **ImageMagick** : pour le redimensionnement et l'optimisation des photos.

### Côté Nextcloud (interface web)

Avant de continuer, vous devez installer l'application suivante depuis le magasin d'applications de votre instance :

* **Workflow external scripts** (Scripts externes de flux).

> ![optimisation-automatique-medias-nextcloud.png](/optimisation-automatique-medias-nextcloud/optimisation-automatique-medias-nextcloud.png)
> *Légende : Recherche et installation de l'application "Workflow external scripts" dans le magasin d'applications Nextcloud.*

---

## 📜 Étape 2 : création du script universel

Ce script s'adapte automatiquement à votre installation, qu'elle soit dans `/var/www` ou `/srv`.

Créez le fichier :

```bash
nano /usr/local/bin/nc-compress.sh

```

Copiez-y ce contenu :

```bash
#!/bin/bash
# Auteur : Amaury aka BlablaLinux (https://link.blablalinux.be)

# --- AUTO-DÉTECTION DES CHEMINS ---
if [ -d "/var/www/nextcloud" ]; then
    NC_ROOT="/var/www/nextcloud"
elif [ -d "/srv/nextcloud" ]; then
    NC_ROOT="/srv/nextcloud"
else
    NC_ROOT="/var/www/html"
fi

# Récupération automatique du dossier DATA depuis la config
DATA_DIR=$(grep "datadirectory" $NC_ROOT/config/config.php | cut -d"'" -f4)
OCC_BIN="$NC_ROOT/occ"

# --- TRAITEMENT DU FICHIER ---
RELATIVE_PATH="$1"
FILE_PATH="$DATA_DIR/$RELATIVE_PATH"

if [ ! -f "$FILE_PATH" ]; then exit 1; fi

EXTENSION=$(echo "${FILE_PATH##*.}" | tr '[:upper:]' '[:lower:]')

# Section vidéo (MP4, MOV, MKV, AVI)
if [[ "$EXTENSION" =~ ^(mp4|mkv|avi|mov)$ ]]; then
    TEMP_VIDEO="${FILE_PATH%.*}.tmp.mp4"
    /usr/bin/nice -n 19 /usr/bin/ionice -c 3 /usr/bin/ffmpeg -i "$FILE_PATH" \
    -c:v libx264 -b:v 6000k -preset ultrafast -threads 1 -b:a 128k "$TEMP_VIDEO" -y
    
    if [ $? -eq 0 ]; then
        mv "$TEMP_VIDEO" "${FILE_PATH%.*}.mp4"
        [ "$EXTENSION" != "mp4" ] && rm "$FILE_PATH"
        FINAL_FILE="${FILE_PATH%.*}.mp4"
    fi

# Section photo (JPG, JPEG)
elif [[ "$EXTENSION" =~ ^(jpg|jpeg)$ ]]; then
    /usr/bin/nice -n 19 /usr/bin/convert "$FILE_PATH" -resize "3468x3468>" -quality 80% "$FILE_PATH"
    FINAL_FILE="$FILE_PATH"
fi

# Scan final pour Nextcloud
if [ -f "$OCC_BIN" ] && [ -n "$FINAL_FILE" ]; then
    REL_PATH=$(echo "$FINAL_FILE" | sed "s|$DATA_DIR/||")
    /usr/bin/php "$OCC_BIN" files:scan --path="$REL_PATH"
fi

```

Rendez le script exécutable :

```bash
chmod +x /usr/local/bin/nc-compress.sh

```

---

## ⚙️ Étape 3 : configuration des flux (workflows)

Attention à bien utiliser les flux d'administration (colonne de gauche, section Administration) et non les flux personnels.

Rendez-vous dans : **Paramètres** > **Administration** > **Flux**.

> ![optimisation-automatique-medias-nextcloud-02.png](/optimisation-automatique-medias-nextcloud/optimisation-automatique-medias-nextcloud-02.png)
> *Légende : Distinction entre les paramètres personnels et la section Administration pour accéder aux flux.*

Cliquez sur **Ajouter un nouveau flux** (bouton en haut à droite) ou sélectionnez la tuile **Run script** et configurez les deux règles suivantes :

> ![optimisation-automatique-medias-nextcloud-03.png](/optimisation-automatique-medias-nextcloud/optimisation-automatique-medias-nextcloud-03.png)
> *Légende : Configuration détaillée des deux règles de flux (images et vidéos) avec l'appel du script Bash.*

1. **Flux images** :
* **Quand** : `Fichier créé`
* **et** : `Type MIME du fichier` / `est une image`
* **Action** : `Exécuter le script` -> `/bin/bash /usr/local/bin/nc-compress.sh %n`


2. **Flux vidéos** :
* **Quand** : `Fichier créé`
* **et** : `Type MIME du fichier` / `correspond à l'expression régulière` / `/^video\/.*/i`
* **Action** : `Exécuter le script` -> `/bin/bash /usr/local/bin/nc-compress.sh %n`



---

## 🕒 Étape 4 : automatisation (cron)

Pour que les changements de taille de fichiers soient rapidement visibles dans votre interface, vérifiez votre tâche Cron pour l'utilisateur `www-data` :

```bash
crontab -u www-data -e

```

Ligne à vérifier ou ajouter :

```text
*/5 * * * * /usr/bin/php -f /var/www/nextcloud/cron.php

```

---

## 💡 Conseil d'expert (Amaury aka BlablaLinux)

L'utilisation de `nice -n 19` et `-threads 1` dans le script est cruciale pour le matériel reconditionné. Cela garantit que la compression tourne en tâche de fond sans jamais ralentir votre navigation sur Nextcloud ou vos autres services auto-hébergés.
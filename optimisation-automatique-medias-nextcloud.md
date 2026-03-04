---
title: Optimisation automatique des médias sur Nextcloud
description: Apprenez à compresser automatiquement vos photos et vidéos sur Nextcloud. Un guide pas à pas pour optimiser l'espace disque de votre serveur reconditionné sans sacrifier sa fluidité habituelle.
published: true
date: 2026-03-04T11:06:12.030Z
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

## 🧐 Pourquoi ces réglages de compression ?

Les paramètres par défaut de ce script ont été minutieusement calibrés en fonction des capacités réelles de mon smartphone (capteur 4:3, 16:9 et 1:1) et de la puissance de mon serveur. L'objectif est d'offrir le meilleur compromis entre **gain d'espace** et **respect de la qualité visuelle**, tout en restant léger pour le processeur.

### Pour les photos (`-resize "3468x3468>" -quality 80%`)

Sur mon appareil, les photos natives montent jusqu'à **3072 x 4096** pixels.

* **Le choix du 3468px** : ce chiffre est une valeur charnière idéale. Il permet de réduire légèrement la hauteur des photos les plus grandes (4:3 et 16:9) pour gagner du poids, tout en laissant intactes les photos carrées (3072x3072) ou plus petites. On conserve ainsi une définition largement supérieure au standard 4K, sans gâchis de stockage.
* **La qualité** : à **80%**, le poids du fichier est souvent divisé par 2 ou 3 sans qu'aucune dégradation ne soit visible, même en zoomant sur un bel écran.

### Pour les vidéos (`-b:v 6000k -preset ultrafast`)

L'analyse des fichiers originaux montre des débits allant jusqu'à **25 Mb/s** pour du 1080p à 60 im/s, soit environ 200 Mo pour une seule minute de film.

* **Le débit (bitrate)** : en limitant à **6000k** (6 Mb/s), on divise le poids par 4. La fluidité du 60 im/s est préservée et l'image reste très propre pour une consultation familiale ou sur smartphone.
* **Le preset et les threads** : l'usage de `ultrafast` et d'un seul `thread` est ma "touche spéciale" pour le matériel reconditionné. Le serveur compresse tranquillement en tâche de fond. Cela prend un peu plus de temps, mais votre Nextcloud reste 100% réactif pour vos autres usages quotidiens.

---

## 🎨 Personnalisation des réglages

Si mes réglages ne vous conviennent pas (par exemple si vous avez un serveur très puissant ou si vous voulez une compression plus forte), vous pouvez modifier les lignes suivantes dans le script :

### Modifier la compression vidéo

Dans la section vidéo du script, cherchez la ligne commençant par `/usr/bin/nice ... ffmpeg`.

* **Qualité/Poids** : modifiez `-b:v 6000k` (le débit binaire). Plus le chiffre est bas, plus le fichier sera léger, mais moins la qualité sera bonne.
* **Vitesse** : le paramètre `-preset ultrafast` privilégie la vitesse sur la taille. Vous pouvez utiliser `medium` pour une meilleure compression si votre CPU le permet.
* **Puissance CPU** : si vous n'êtes pas sur un petit processeur, vous pouvez augmenter `-threads 1` à `2` ou `4` pour compresser plus vite.

### Modifier la compression photo

Dans la section photo, cherchez la ligne commençant par `/usr/bin/nice ... convert`.

* **Résolution** : changez `3468x3468>` pour augmenter ou réduire la taille maximale des images.
* **Qualité** : modifiez `-quality 80%`. Un réglage à `70%` gagnera beaucoup d'espace, tandis que `90%` sera presque invisible à l'œil nu mais le fichier sera plus lourd.

---

## 💡 Conseil d'expert (Amaury aka BlablaLinux)

L'utilisation de `nice -n 19` et `-threads 1` dans le script est cruciale pour le matériel reconditionné. Cela garantit que la compression tourne en tâche de fond sans jamais ralentir votre navigation sur Nextcloud ou vos autres services auto-hébergés.
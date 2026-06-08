---
title: Optimisation automatique des médias sur Nextcloud
description: Apprenez à compresser automatiquement vos photos et vidéos sur Nextcloud. Un guide pas à pas pour optimiser l'espace disque de votre serveur reconditionné sans sacrifier sa fluidité habituelle.
published: true
date: 2026-06-08T10:17:23.516Z
tags: cron, crontab, script, bash, ffmpeg, auto-hébergement, optimisation, nextcloud, reconditionnement, imagemagick
editor: markdown
dateCreated: 2026-03-02T11:46:46.923Z
---

Ce guide vous permet de mettre en place un système de compression automatique pour vos photos et vidéos. L'objectif est de réduire l'espace disque consommé par vos sauvegardes mobiles tout en préservant la réactivité de votre serveur, même sur du matériel reconditionné.

## 📋 Fonctionnement du système

1. **Upload** : vous envoyez un média depuis votre smartphone.
2. **Déclencheur (workflow)** : Nextcloud détecte l'arrivée du fichier et appelle un script Bash.
3. **Traitement (script)** : le serveur analyse le fichier (résolution, FPS, débit) et compresse intelligemment en arrière-plan avec une priorité CPU très basse.
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

## 📜 Étape 2 : création du script de compression intelligent

Ce script analyse automatiquement chaque vidéo (résolution, FPS, débit) avant de décider s'il doit compresser et vers quel débit cible. Il gère aussi bien les vidéos filmées en portrait qu'en paysage.

Créez le fichier :

```bash
nano /usr/local/bin/nc-compress.sh
```

Copiez-y ce contenu en **adaptant les deux premières variables** à votre installation :

```bash
#!/bin/bash
# Script : nc-compress.sh
# Auteur : Amaury aka BlablaLinux (https://link.blablalinux.be)
# Compression intelligente : résolution + FPS + débit

DATA_DIR="/srv/nextcloud/data"   # ← À adapter selon votre installation
OCC_BIN="/srv/nextcloud/occ"     # ← À adapter selon votre installation

RELATIVE_PATH="$1"
FILE_PATH="$DATA_DIR/$RELATIVE_PATH"
LOGFILE="$DATA_DIR/nc-compress.log"

log() { echo "$(date '+%Y-%m-%d %H:%M:%S') $1" >> "$LOGFILE"; }

if [ ! -f "$FILE_PATH" ]; then
    log "[SKIP] Fichier introuvable : $FILE_PATH"
    exit 1
fi

EXTENSION_LOWER=$(echo "${FILE_PATH##*.}" | tr '[:upper:]' '[:lower:]')

# --- SECTION VIDÉO ---
if [[ "$EXTENSION_LOWER" =~ ^(mp4|mkv|avi|mov)$ ]]; then

    # Récupération du débit global
    BITRATE=$(/usr/bin/ffprobe -v error -select_streams v:0 \
        -show_entries format=bit_rate \
        -of csv=p=0 "$FILE_PATH" 2>/dev/null | tr -d ',')

    # Récupération des FPS
    FPS=$(/usr/bin/ffprobe -v error -select_streams v:0 \
        -show_entries stream=r_frame_rate \
        -of csv=p=0 "$FILE_PATH" 2>/dev/null | tr -d ',' \
        | awk -F'/' '{if($2>0) printf "%.0f\n", $1/$2; else print $1}')
    [ -z "$FPS" ] && FPS=30

    # Récupération largeur et hauteur
    WIDTH=$(/usr/bin/ffprobe -v error -select_streams v:0 \
        -show_entries stream=width \
        -of csv=p=0 "$FILE_PATH" 2>/dev/null | tr -d ',')
    HEIGHT=$(/usr/bin/ffprobe -v error -select_streams v:0 \
        -show_entries stream=height \
        -of csv=p=0 "$FILE_PATH" 2>/dev/null | tr -d ',')
    [ -z "$WIDTH" ]  && WIDTH=1920
    [ -z "$HEIGHT" ] && HEIGHT=1080

    # Petite dimension = résolution réelle (gère portrait ET paysage)
    SHORT_SIDE=$(( WIDTH < HEIGHT ? WIDTH : HEIGHT ))

    # --- Choix du seuil et de la cible selon résolution + FPS ---
    if [ "$SHORT_SIDE" -ge 2160 ]; then
        # 4K
        if [ "$FPS" -gt 45 ]; then
            LIMIT_BITRATE=30000000; TARGET_BITRATE="20000k"
        else
            LIMIT_BITRATE=20000000; TARGET_BITRATE="15000k"
        fi
        LABEL="4K"
    elif [ "$SHORT_SIDE" -ge 1440 ]; then
        # 1440p
        if [ "$FPS" -gt 45 ]; then
            LIMIT_BITRATE=15000000; TARGET_BITRATE="10000k"
        else
            LIMIT_BITRATE=10000000; TARGET_BITRATE="7000k"
        fi
        LABEL="1440p"
    elif [ "$SHORT_SIDE" -ge 1080 ]; then
        # 1080p — cas principal (smartphone + captures PC)
        if [ "$FPS" -gt 45 ]; then
            LIMIT_BITRATE=12000000; TARGET_BITRATE="8000k"
        else
            LIMIT_BITRATE=8000000;  TARGET_BITRATE="6000k"
        fi
        LABEL="1080p"
    else
        # 720p et moins
        if [ "$FPS" -gt 45 ]; then
            LIMIT_BITRATE=5000000; TARGET_BITRATE="3000k"
        else
            LIMIT_BITRATE=3000000; TARGET_BITRATE="2000k"
        fi
        LABEL="720p-"
    fi

    BUFSIZE="$(( ${TARGET_BITRATE%k} * 2 ))k"

    if [ -n "$BITRATE" ] && [ "$BITRATE" -gt "$LIMIT_BITRATE" ]; then
        TEMP_VIDEO="${FILE_PATH%.*}.tmp.mp4"
        DEST_VIDEO="${FILE_PATH%.*}.mp4"
        log "[VIDEO] Compression : $FILE_PATH (${LABEL}, ${FPS}fps, ${BITRATE}bps → $TARGET_BITRATE)"

        /usr/bin/nice -n 19 /usr/bin/ionice -c 3 /usr/bin/ffmpeg -i "$FILE_PATH" \
            -c:v libx264 -b:v "$TARGET_BITRATE" -maxrate "$TARGET_BITRATE" \
            -bufsize "$BUFSIZE" -preset veryfast -threads 2 -b:a 256k \
            -map_metadata 0 -movflags use_metadata_tags \
            "$TEMP_VIDEO" -y 2>>"$LOGFILE"

        if [ $? -eq 0 ]; then
            mv "$TEMP_VIDEO" "$DEST_VIDEO"
            [ "$EXTENSION_LOWER" != "mp4" ] && rm "$FILE_PATH"
            FINAL_FILE="$DEST_VIDEO"
            log "[VIDEO] OK : $FINAL_FILE"
        else
            rm -f "$TEMP_VIDEO"
            log "[VIDEO] ERREUR ffmpeg sur : $FILE_PATH"
            exit 1
        fi
    else
        FINAL_FILE="$FILE_PATH"
        log "[VIDEO] Débit OK (${LABEL}, ${FPS}fps, ${BITRATE}bps ≤ seuil ${LIMIT_BITRATE}bps) : $FILE_PATH"
    fi

# --- SECTION PHOTO ---
elif [[ "$EXTENSION_LOWER" =~ ^(jpg|jpeg)$ ]]; then
    TEMP_IMG="${FILE_PATH%.*}.tmp.jpg"
    /usr/bin/nice -n 19 /usr/bin/convert "$FILE_PATH" \
        -resize "3468x3468>" -quality 80% "$TEMP_IMG"

    if [ $? -eq 0 ]; then
        mv "$TEMP_IMG" "$FILE_PATH"
        FINAL_FILE="$FILE_PATH"
        log "[PHOTO] OK : $FILE_PATH"
    else
        rm -f "$TEMP_IMG"
        log "[PHOTO] ERREUR convert sur : $FILE_PATH"
        exit 1
    fi
fi

# --- SCAN NEXTCLOUD ---
if [ -f "$OCC_BIN" ] && [ -n "$FINAL_FILE" ]; then
    REL_PATH=$(echo "$FINAL_FILE" | sed "s|$DATA_DIR/||")
    log "[SCAN] occ files:scan --path=$REL_PATH"
    /usr/bin/php8.3 "$OCC_BIN" files:scan --path="$REL_PATH" >> "$LOGFILE" 2>&1
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

Ligne à vérifier ou ajouter (adaptez le chemin selon votre installation) :

```text
*/5 * * * * /usr/bin/php -f /srv/nextcloud/cron.php
```

---

## 🧐 Pourquoi ces réglages de compression ?

### Logique intelligente : résolution + FPS + débit

Contrairement à une compression uniforme, ce script analyse chaque vidéo avant d'agir. Il pose une seule question : **le débit de cette vidéo est-il trop élevé pour sa résolution et ses FPS ?**

Si oui, il compresse vers un débit cible adapté. Si non, il ne touche à rien — pas de dégradation inutile.

| Résolution | FPS ≤ 45 | FPS > 45 | Cible ≤ 45 | Cible > 45 |
|---|---|---|---|---|
| 4K (2160p) | 20 Mbps | 30 Mbps | 15000k | 20000k |
| 1440p | 10 Mbps | 15 Mbps | 7000k | 10000k |
| 1080p | 8 Mbps | 12 Mbps | 6000k | 8000k |
| 720p et moins | 3 Mbps | 5 Mbps | 2000k | 3000k |

**Exemples concrets :**

* Une vidéo 1080p à 60fps filmée à 25 Mbps → seuil 12 Mbps dépassé → compression vers **8000k** → division du poids par 3, qualité très propre sur grand écran ✅
* Une capture d'écran PC 1080p à 30fps à 4 Mbps → seuil 8 Mbps non atteint → **aucune compression**, fichier laissé intact ✅

Le script gère également correctement les vidéos filmées en **portrait** (ex. 1080x1920 depuis un smartphone) : il détecte la plus petite dimension pour identifier la vraie résolution, évitant ainsi une mauvaise classification.

### Pour les photos (`-resize "3468x3468>" -quality 80%`)

Sur la plupart des smartphones récents, les photos natives montent jusqu'à **3072 x 4096** pixels.

* **Le choix du 3468px** : ce chiffre est une valeur charnière idéale. Il permet de réduire légèrement la hauteur des photos les plus grandes (4:3 et 16:9) pour gagner du poids, tout en laissant intactes les photos carrées (3072x3072) ou plus petites. On conserve ainsi une définition largement supérieure au standard 4K, sans gâchis de stockage.
* **La qualité** : à **80%**, le poids du fichier est souvent divisé par 2 ou 3 sans qu'aucune dégradation ne soit visible, même en zoomant sur un bel écran.

### Sécurité des fichiers

Le script utilise systématiquement un **fichier temporaire** pendant la compression. Le fichier original n'est remplacé qu'une fois la compression validée avec succès. En cas d'erreur, le fichier original est préservé et l'erreur est consignée dans le log.

---

## 🔍 Consulter les logs

Toutes les opérations sont enregistrées dans un fichier de log situé à la racine de votre dossier data :

```bash
tail -f /srv/nextcloud/data/nc-compress.log   # À adapter selon votre installation
```

Exemple de sortie :

```
2026-06-08 02:14:20 [VIDEO] Compression : .../VID20260608015329.mp4 (1080p, 60fps, 25904000bps → 8000k)
2026-06-08 02:14:34 [VIDEO] OK : .../VID20260608015329.mp4
2026-06-08 02:14:34 [SCAN] occ files:scan --path=Amaury/files/InstantUpload/Camera/VID20260608015329.mp4
```

---

## 🎨 Personnalisation des réglages

### Modifier les seuils et débits cibles

Les seuils de déclenchement et les débits cibles sont définis dans le bloc de conditions du script. Pour chaque résolution, vous pouvez ajuster `LIMIT_BITRATE` (seuil de déclenchement en bits/s) et `TARGET_BITRATE` (débit cible après compression).

**Exemple** : si vous souhaitez une compression plus agressive en 1080p à 60fps, passez la cible de `8000k` à `5000k`.

### Modifier la compression photo

Dans la section photo, cherchez la ligne commençant par `/usr/bin/nice ... convert`.

* **Résolution** : changez `3468x3468>` pour augmenter ou réduire la taille maximale des images.
* **Qualité** : modifiez `-quality 80%`. Un réglage à `70%` gagnera beaucoup d'espace, tandis que `90%` sera presque invisible à l'œil nu mais le fichier sera plus lourd.

### Modifier le preset d'encodage

Le paramètre `-preset veryfast` offre un bon compromis entre vitesse d'encodage et qualité à débit égal. Si votre serveur est puissant, vous pouvez passer à `faster` ou `medium` pour une meilleure efficacité de compression. Si au contraire vous êtes sur du matériel très limité, `superfast` ou `ultrafast` réduiront la charge CPU au détriment d'une légère perte de qualité.

---

## 💡 Conseil d'expert (Amaury aka BlablaLinux)

L'utilisation de `nice -n 19` et `ionice -c 3` dans le script est cruciale, surtout sur du matériel reconditionné. Cela garantit que la compression tourne en tâche de fond avec la priorité CPU et disque la plus basse possible, sans jamais ralentir votre navigation sur Nextcloud ou vos autres services auto-hébergés.
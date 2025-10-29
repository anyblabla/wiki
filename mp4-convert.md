---
title: Conversion Vidéo (MP4) avec FFMPEG
description: Cette page documente les alias Bash basés sur FFMPEG, conçus pour la conversion et l'optimisation des fichiers vidéo au format MP4.
published: false
date: 2025-10-29T23:46:41.944Z
tags: bash, convert, mp4, ffmpeg, alias
editor: markdown
dateCreated: 2025-10-29T23:46:41.944Z
---

## 🎯 Objectif

Ces alias permettent de convertir tous les fichiers `*.mp4` du répertoire courant en appliquant un **débit binaire vidéo** spécifique (`-b:v`) pour contrôler la taille et la qualité du fichier de sortie.

  * **Portabilité :** Les alias utilisent la variable `$HOME` pour garantir qu'ils fonctionnent quel que soit l'utilisateur.
  * **Répertoire de sortie :** Tous les fichiers convertis sont placés dans **`$HOME/Vidéos/MP4convert/`**.
  * **Sécurité :** Le fichier de sortie est préfixé par le débit binaire (ex: `3000k-`) pour **éviter d'écraser** l'original.

-----

## ⚙️ Alias et Débits Binaires

Chaque alias utilise FFMPEG avec un **codec vidéo H.264** (`-c:v libx264`) et un débit audio standard de **96 kbps** (`-b:a 96k`), sauf si indiqué.

| Alias | Débit Binaire Vidéo (`-b:v`) | Qualité Relative |
| :--- | :--- | :--- |
| `mp4convert200` | 200k | Très faible (Aperçu) |
| `mp4convert1000` | 1000k (1 Mbps) | Standard (Web, 720p) |
| **`mp4convert3000`** | **3000k (3 Mbps)** | **Haute (Standard HD)** |
| `mp4convert6000` | 6000k (6 Mbps) | Très Haute (Haute définition) |
| `mp4convertnextcloud` | 6000k | Spécifique (pour plateformes) |

-----

## 💾 Code à Insérer dans `.bash_aliases`

Ces alias doivent être placés dans votre fichier **`~/.bash_aliases`** :

```bash
## 🎥 Conversion Vidéo (FFMPEG)
# Utilise $HOME/Vidéos/MP4convert/ pour la portabilité inter-utilisateurs.
alias mp4convert200='for file in *.mp4; do ffmpeg -i "$file" -b:v 200k -b:a 96k -c:v libx264 "$HOME/Vidéos/MP4convert/200k-$file"; done'
alias mp4convert500='for file in *.mp4; do ffmpeg -i "$file" -b:v 500k -b:a 96k -c:v libx264 "$HOME/Vidéos/MP4convert/500k-$file"; done'
alias mp4convert1000='for file in *.mp4; do ffmpeg -i "$file" -b:v 1000k -b:a 96k -c:v libx264 "$HOME/Vidéos/MP4convert/1000k-$file"; done'
alias mp4convert1500='for file in *.mp4; do ffmpeg -i "$file" -b:v 1500k -b:a 96k -c:v libx264 "$HOME/Vidéos/MP4convert/1500k-$file"; done'
alias mp4convert2000='for file in *.mp4; do ffmpeg -i "$file" -b:v 2000k -b:a 96k -c:v libx264 "$HOME/Vidéos/MP4convert/2000k-$file"; done'
alias mp4convert2500='for file in *.mp4; do ffmpeg -i "$file" -b:v 2500k -b:a 96k -c:v libx264 "$HOME/Vidéos/MP4convert/2500k-$file"; done'
alias mp4convert3000='for file in *.mp4; do ffmpeg -i "$file" -b:v 3000k -b:a 96k -c:v libx264 "$HOME/Vidéos/MP4convert/3000k-$file"; done'
alias mp4convert3500='for file in *.mp4; do ffmpeg -i "$file" -b:v 3500k -b:a 96k -c:v libx264 "$HOME/Vidéos/MP4convert/3500k-$file"; done'
alias mp4convert4000='for file in *.mp4; do ffmpeg -i "$file" -b:v 4000k -b:a 96k -c:v libx264 "$HOME/Vidéos/MP4convert/4000k-$file"; done'
alias mp4convert4500='for file in *.mp4; do ffmpeg -i "$file" -b:v 4500k -b:a 96k -c:v libx264 "$HOME/Vidéos/MP4convert/4500k-$file"; done'
alias mp4convert5000='for file in *.mp4; do ffmpeg -i "$file" -b:v 5000k -b:a 96k -c:v libx264 "$HOME/Vidéos/MP4convert/5000k-$file"; done'
alias mp4convert5500='for file in *.mp4; do ffmpeg -i "$file" -b:v 5500k -b:a 96k -c:v libx264 "$HOME/Vidéos/MP4convert/5500k-$file"; done'
alias mp4convert6000='for file in *.mp4; do ffmpeg -i "$file" -b:v 6000k -b:a 96k -c:v libx264 "$HOME/Vidéos/MP4convert/6000k-$file"; done'
alias mp4convertnextcloud='for file in *.mp4; do ffmpeg -i "$file" -b:v 6000k -c:v libx264 "$HOME/Vidéos/MP4convert/nextcloud-$file"; done'
```

-----

## 🧪 Exemple complet d'Alias

Voici l'alias `mp4convert3000` décortiqué pour illustrer sa structure :

```bash
alias mp4convert3000='
    for file in *.mp4; # Pour chaque fichier .mp4 dans le dossier
    do 
        ffmpeg -i "$file" \
            -b:v 3000k \
            -b:a 96k \
            -c:v libx264 \
            "$HOME/Vidéos/MP4convert/3000k-$file"; # Chemin de sortie portable
    done
'
```
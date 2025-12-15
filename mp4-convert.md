---
title: Conversion vidéo (MP4) avec FFMPEG
description: Cette page documente les alias Bash basés sur FFMPEG, conçus pour la conversion et l'optimisation des fichiers vidéo au format MP4.
published: true
date: 2025-12-15T11:03:12.638Z
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

Chaque alias utilise FFMPEG avec un **codec vidéo H.264** (`-c:v libx264`). La majorité utilise un débit audio standard de **$96\text{ kbps}$** (`-b:a 96k`), sauf l'alias `mp4convertnextcloud`.

| Alias | Débit Binaire Vidéo (`-b:v`) | Qualité Relative |
| :--- | :--- | :--- |
| `mp4convert200` | 200k | Très faible (Aperçu) |
| `mp4convert1000` | 1000k (1 Mbps) | Standard (Web, 720p) |
| **`mp4convert3000`** | **3000k (3 Mbps)** | **Haute (Standard HD)** |
| `mp4convert6000` | 6000k (6 Mbps) | Très Haute (Compression Audio) |
| `mp4convertnextcloud` | 6000k | **Optimisation Spécifique (Audio Original)** |

-----

## 🧐 Focus Spécifique : Différence entre les Alias $6000\text{k}$

Les deux alias de $6000\text{k}$ diffèrent par la manière dont ils gèrent la piste audio, ce qui influence leur usage final :

| Alias | Débit Audio | Contexte d'Utilisation |
| :--- | :--- | :--- |
| **`mp4convert6000`** | **$96\text{k}$** (`-b:a 96k`) | Conversion de haute qualité où l'audio est compressé à un niveau standard (réduction de la taille globale du fichier). |
| **`mp4convertnextcloud`** | **Original** (non spécifié) | Normalisation de sources de très haute qualité (ex: $1080\text{p} \text{ à } 60\text{ fps}$ et $25000\text{k}$ source). Il réduit la vidéo à $6\text{ Mbps}$ tout en **conservant la qualité sonore native**. |

### Rôle de `mp4convertnextcloud`

Cet alias permet de réduire la taille du fichier d'environ $75\%$ (en passant de $25\text{ Mbps}$ à $6\text{ Mbps}$ pour la vidéo) tout en **préservant l'intégrité audio** pour l'archivage haute fidélité ou le partage sur des plateformes exigeantes en qualité sonore.

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
    for file in *.mp4; # 1. Initialise la boucle pour chaque fichier .mp4 dans le dossier actuel
    do 
        ffmpeg -i "$file" \
            -b:v 3000k \
            -b:a 96k \
            -c:v libx264 \
            "$HOME/Vidéos/MP4convert/3000k-$file"; # 2. Chemin de sortie portable et préfixé
    done
'
```
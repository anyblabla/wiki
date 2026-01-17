---
title: Conversion vidéo optimisée (FFMPEG)
description: Guide complet pour automatiser la conversion vidéo massive sous Linux via FFMPEG. Inclut la configuration des pilotes VA-API (Intel) et des alias pour l'encodage CPU et GPU.
published: true
date: 2026-01-17T23:26:26.985Z
tags: bash, convert, mp4, ffmpeg, alias
editor: markdown
dateCreated: 2025-10-29T23:46:41.944Z
---

## Objectif

Ces alias permettent de convertir tous les fichiers `*.mp4` du répertoire courant en appliquant un débit binaire vidéo spécifique (`-b:v`) pour contrôler la taille et la qualité du fichier de sortie.

* **Portabilité :** les alias utilisent la variable `$HOME` pour garantir qu'ils fonctionnent quel que soit l'utilisateur.
* **Répertoire de sortie :** tous les fichiers convertis sont placés dans **`$HOME/Vidéos/MP4convert/`**.
* **Sécurité :** le fichier de sortie est préfixé par le débit ou la méthode (ex : `3000k-` ou `gpu-`) pour **éviter d'écraser** l'original.

L'utilisation d'alias permet de basculer entre deux stratégies :

1. **Méthode CPU (logicielle) :** utilise `libx264`. Meilleure qualité d'image par bit.
2. **Méthode GPU (matérielle) :** utilise `VA-API`. Conversions ultra-rapides sans solliciter le processeur.

> **⚠️ Avertissement sur le matériel ancien (reconditionnement) :**
> Si vous utilisez un processeur Intel d'ancienne génération (ex : Sandy Bridge / Core i7-2xxx), l'accélération matérielle (GPU) peut échouer malgré une configuration correcte. Les noyaux Linux récents (6.x+) restreignent parfois l'accès à ces puces pour des raisons de sécurité. Dans ce cas, les alias **CPU** restent votre solution la plus fiable.

---

## Installation des pilotes VA-API (Intel)

Pour activer l'accélération matérielle (alias préfixés par `gpu-`), installez le pilote correspondant à votre processeur.

### 1. Installation des paquets de base

```bash
sudo apt update
sudo apt install -y vainfo ffmpeg

```

### 2. Choix du pilote selon la génération

* **Générations anciennes (Broadwell et antérieurs) :**
*Note : Pour les processeurs de 2ème à 4ème génération, installez la version `-shaders` pour débloquer l'encodage.*

```bash
sudo apt install -y i965-va-driver-shaders
echo "LIBVA_DRIVER_NAME=i965" | sudo tee -a /etc/environment

```

* **Générations récentes (Skylake et plus récent) :**

```bash
sudo apt install -y intel-media-va-driver-non-free
echo "LIBVA_DRIVER_NAME=iHD" | sudo tee -a /etc/environment

```

> **💡 Note sur les droits d'accès :**
> Pour que le GPU soit accessible, votre utilisateur doit impérativement appartenir aux groupes `video` et `render`.
> `sudo usermod -aG video,render $USER` (puis redémarrez votre session).

---

## Alias et débits binaires

Chaque alias utilise un codec H.264. La majorité utilise un débit audio standard de **96 kbps** (`-b:a 96k`), sauf l'alias spécifique à Nextcloud.

| Alias | Débit binaire vidéo (`-b:v`) | Qualité relative |
| --- | --- | --- |
| `mp4convert200` | 200k | Très faible (aperçu) |
| `mp4convert1000` | 1000k (1 Mbps) | Standard (web, 720p) |
| **`mp4convert3000`** | **3000k (3 Mbps)** | **Haute (standard HD)** |
| `mp4convert6000` | 6000k (6 Mbps) | Très haute |
| `mp4convertnextcloud` | 6000k | **Optimisation audio original** |

### Focus spécifique : différence entre les alias 6000k

Les deux alias de 6000k diffèrent par la gestion de la piste audio :

| Alias | Débit audio | Contexte d'utilisation |
| --- | --- | --- |
| **`mp4convert6000`** | **96k** | Conversion haute qualité avec compression audio standard (gain de place). |
| **`mp4convertnextcloud`** | **Original** | Réduction vidéo à 6 Mbps tout en **conservant la qualité sonore native**. |

---

## Code à insérer dans .bash_aliases

### 1. Méthode CPU (qualité maximale / libx264)

```bash
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

### 2. Méthode GPU (vitesse éclair - VA-API Intel)

```bash
alias gpu-mp4convert200='for file in *.mp4; do ffmpeg -hwaccel vaapi -vaapi_device /dev/dri/renderD128 -i "$file" -vf "format=nv12,hwupload" -c:v h264_vaapi -b:v 200k -b:a 96k "$HOME/Vidéos/MP4convert/gpu-200k-$file"; done'
alias gpu-mp4convert500='for file in *.mp4; do ffmpeg -hwaccel vaapi -vaapi_device /dev/dri/renderD128 -i "$file" -vf "format=nv12,hwupload" -c:v h264_vaapi -b:v 500k -b:a 96k "$HOME/Vidéos/MP4convert/gpu-500k-$file"; done'
alias gpu-mp4convert1000='for file in *.mp4; do ffmpeg -hwaccel vaapi -vaapi_device /dev/dri/renderD128 -i "$file" -vf "format=nv12,hwupload" -c:v h264_vaapi -b:v 1000k -b:a 96k "$HOME/Vidéos/MP4convert/gpu-1000k-$file"; done'
alias gpu-mp4convert1500='for file in *.mp4; do ffmpeg -hwaccel vaapi -vaapi_device /dev/dri/renderD128 -i "$file" -vf "format=nv12,hwupload" -c:v h264_vaapi -b:v 1500k -b:a 96k "$HOME/Vidéos/MP4convert/gpu-1500k-$file"; done'
alias gpu-mp4convert2000='for file in *.mp4; do ffmpeg -hwaccel vaapi -vaapi_device /dev/dri/renderD128 -i "$file" -vf "format=nv12,hwupload" -c:v h264_vaapi -b:v 2000k -b:a 96k "$HOME/Vidéos/MP4convert/gpu-2000k-$file"; done'
alias gpu-mp4convert2500='for file in *.mp4; do ffmpeg -hwaccel vaapi -vaapi_device /dev/dri/renderD128 -i "$file" -vf "format=nv12,hwupload" -c:v h264_vaapi -b:v 2500k -b:a 96k "$HOME/Vidéos/MP4convert/gpu-2500k-$file"; done'
alias gpu-mp4convert3000='for file in *.mp4; do ffmpeg -hwaccel vaapi -vaapi_device /dev/dri/renderD128 -i "$file" -vf "format=nv12,hwupload" -c:v h264_vaapi -b:v 3000k -b:a 96k "$HOME/Vidéos/MP4convert/gpu-3000k-$file"; done'
alias gpu-mp4convert3500='for file in *.mp4; do ffmpeg -hwaccel vaapi -vaapi_device /dev/dri/renderD128 -i "$file" -vf "format=nv12,hwupload" -c:v h264_vaapi -b:v 3500k -b:a 96k "$HOME/Vidéos/MP4convert/gpu-3500k-$file"; done'
alias gpu-mp4convert4000='for file in *.mp4; do ffmpeg -hwaccel vaapi -vaapi_device /dev/dri/renderD128 -i "$file" -vf "format=nv12,hwupload" -c:v h264_vaapi -b:v 4000k -b:a 96k "$HOME/Vidéos/MP4convert/gpu-4000k-$file"; done'
alias gpu-mp4convert4500='for file in *.mp4; do ffmpeg -hwaccel vaapi -vaapi_device /dev/dri/renderD128 -i "$file" -vf "format=nv12,hwupload" -c:v h264_vaapi -b:v 4500k -b:a 96k "$HOME/Vidéos/MP4convert/gpu-4500k-$file"; done'
alias gpu-mp4convert5000='for file in *.mp4; do ffmpeg -hwaccel vaapi -vaapi_device /dev/dri/renderD128 -i "$file" -vf "format=nv12,hwupload" -c:v h264_vaapi -b:v 5000k -b:a 96k "$HOME/Vidéos/MP4convert/gpu-5000k-$file"; done'
alias gpu-mp4convert5500='for file in *.mp4; do ffmpeg -hwaccel vaapi -vaapi_device /dev/dri/renderD128 -i "$file" -vf "format=nv12,hwupload" -c:v h264_vaapi -b:v 5500k -b:a 96k "$HOME/Vidéos/MP4convert/gpu-5500k-$file"; done'
alias gpu-mp4convert6000='for file in *.mp4; do ffmpeg -hwaccel vaapi -vaapi_device /dev/dri/renderD128 -i "$file" -vf "format=nv12,hwupload" -c:v h264_vaapi -b:v 6000k -b:a 96k "$HOME/Vidéos/MP4convert/gpu-6000k-$file"; done'
alias gpu-mp4convertnextcloud='for file in *.mp4; do ffmpeg -hwaccel vaapi -vaapi_device /dev/dri/renderD128 -i "$file" -vf "format=nv12,hwupload" -c:v h264_vaapi -b:v 6000k "$HOME/Vidéos/MP4convert/gpu-nextcloud-$file"; done'

```

---

## 🧪 Exemple complet d'alias décortiqué

Voici l'alias `gpu-mp4convert3000` illustré pour comprendre sa structure :

```bash
alias gpu-mp4convert3000='
    for file in *.mp4; # 1. boucle pour chaque fichier .mp4 du dossier
    do 
        ffmpeg -hwaccel vaapi -vaapi_device /dev/dri/renderD128 \ # 2. activation accélération matérielle
            -i "$file" \
            -vf "format=nv12,hwupload" \ # 3. préparation du flux pour le GPU
            -c:v h264_vaapi \ # 4. utilisation de l encodeur matériel
            -b:v 3000k -b:a 96k \ # 5. réglage des débits vidéo et audio
            "$HOME/Vidéos/MP4convert/gpu-3000k-$file"; # 6. sortie sécurisée et préfixée
    done
'

```

---

## Dépannage (Troubleshooting)

### Erreur : `get chip id failed: -1 [13]` ou `Permission denied`

Si vous obtenez cette erreur avec un alias `gpu-` alors que vous êtes bien dans le groupe `render`, cela confirme que votre processeur est trop ancien (ex : Sandy Bridge / Core i7-2xxx) pour les pilotes DRM actuels du noyau Linux 6.x.

**Solution :** Ne perdez pas de temps à essayer de forcer le GPU. Votre CPU possède suffisamment de threads pour gérer la conversion via les alias **CPU** classiques sans le préfixe `gpu-`.
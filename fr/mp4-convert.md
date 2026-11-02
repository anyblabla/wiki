---
title: Conversion vidéo optimisée (FFMPEG)
description: >-
  Guide complet pour automatiser la conversion vidéo massive sous Linux via
  FFMPEG. Inclut la configuration des pilotes VA-API (Intel) et des alias pour
  l'encodage CPU et GPU.
published: true
date: '2026-11-02T23:29:07.598Z'
tags:
  - bash
  - convert
  - mp4
  - ffmpeg
  - alias
editor: markdown
dateCreated: '2025-10-29T23:46:41.944Z'
---

a## Objectif

Ces alias permettent de convertir tous les fichiers vidéo du répertoire courant en appliquant un débit binaire vidéo spécifique (`-b:v`) pour contrôler la taille et la qualité du fichier de sortie.

* **Portabilité :** les alias utilisent la variable `$HOME` pour garantir qu'ils fonctionnent quel que soit l'utilisateur.
* **Répertoire de sortie :** tous les fichiers convertis sont placés dans **`$HOME/Vidéos/MP4convert/`**.
* **Sécurité :** le fichier de sortie est préfixé par le débit ou la méthode (ex : `3000k-` ou `gpu-`) pour **éviter d'écraser** l'original.
* **Universalité :** Les alias acceptent désormais les formats **MKV, AVI, MOV et MP4** et génèrent un `.mp4` propre (sans double extension).

L'utilisation d'alias permet de basculer entre deux stratégies :

1. **Méthode CPU (logicielle) :** utilise `libx264`. Meilleure qualité d'image par bit.
2. **Méthode GPU (matérielle) :** utilise `VA-API`. Conversions ultra-rapides sans solliciter le processeur.

> **⚠️ Avertissement sur le matériel ancien (reconditionnement) :**
> Si vous utilisez un processeur Intel d'ancienne génération (ex : Sandy Bridge / Core i7-2xxx), l'accélération matérielle (GPU) peut échouer malgré une configuration correcte. Les noyaux Linux récents (6.x+) restreignent parfois l'accès à ces puces pour des raisons de sécurité. Dans ce cas, les alias **CPU** restent votre solution la plus fiable.

---

## Préparation du système

### 1. Création du répertoire de sortie

FFmpeg ne créera pas le dossier automatiquement. Lancez cette commande avant d'utiliser les alias :

```bash
mkdir -p $HOME/Vidéos/MP4convert/

```

*Note : Si vous choisissez une autre destination, vous devrez modifier les alias en conséquence.*

### 2. Droits d'accès et groupes

Pour que le GPU soit accessible, votre utilisateur doit impérativement appartenir aux groupes `video` et `render`.

* **Vérifier vos groupes :** `groups`
* **S'ajouter aux groupes :**

```bash
sudo usermod -aG video,render $USER

```

*(Un redémarrage de session ou du système est nécessaire).*

---

## Installation des pilotes VA-API (Intel)

> **🚨 Important : Configuration des dépôts Debian (12 & 13)**
> Pour que les paquets ci-dessous soient trouvables par votre système, vous devez impérativement activer les sections `contrib`, `non-free` et `non-free-firmware`.
> 1. Modifiez votre fichier de sources : `sudo nano /etc/apt/sources.list`
> 2. Ajoutez les composants à la fin de vos lignes de dépôts pour qu'elles ressemblent à ceci :
> `deb http://deb.debian.org/debian/ trixie main contrib non-free non-free-firmware`
> 3. Mettez à jour : `sudo apt update`

### 1. Installation des paquets de base

```bash
sudo apt update
sudo apt install -y vainfo ffmpeg intel-gpu-tools nano

```

### 2. Choix du pilote selon la génération

* **Générations anciennes (Broadwell et antérieurs) :**

```bash
sudo apt install -y i965-va-driver-shaders
echo "LIBVA_DRIVER_NAME=i965" | sudo tee -a /etc/environment

```

* **Générations récentes (Skylake et plus récent) :**

```bash
sudo apt install -y intel-media-va-driver-non-free
echo "LIBVA_DRIVER_NAME=iHD" | sudo tee -a /etc/environment

```

> **💡 Note sur les systèmes hybrides (Double GPU) :**
> Sur certains PC portables (ex: HP Pavilion dv7), vous avez un **iGPU Intel** (intégré) et un **GPU discret AMD/NVIDIA**.
> Lancez `intel_gpu_top`. Si le GPU Intel apparaît sur `/dev/dri/card1`, vous devrez utiliser **`/dev/dri/renderD129`** dans vos alias. Si c'est sur `card0`, utilisez **`/dev/dri/renderD128`**.

---

## Installation des Alias avec Nano

1. Ouvrez le fichier de configuration : `nano ~/.bash_aliases`
2. Collez les blocs de code CPU et GPU ci-dessous.
3. Sauvegardez (`Ctrl+O` puis `Entrée`) et quittez (`Ctrl+X`).
4. Actualisez votre terminal : `source ~/.bashrc`

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

| Alias | Débit audio | Contexte d'utilisation |
| --- | --- | --- |
| **`mp4convert6000`** | **96k** | Conversion haute qualité avec compression audio standard (gain de place). |
| **`mp4convertnextcloud`** | **Original** | Réduction vidéo à 6 Mbps tout en **conservant la qualité sonore native**. |

---

## Code à insérer dans .bash_aliases

### 1. Méthode CPU (qualité maximale / libx264)

```bash
alias mp4convert200='for file in *.{mp4,mkv,avi,mov}; do [ -e "$file" ] || continue; ffmpeg -i "$file" -b:v 200k -b:a 96k -c:v libx264 "$HOME/Vidéos/MP4convert/200k-${file%.*}.mp4"; done'
alias mp4convert500='for file in *.{mp4,mkv,avi,mov}; do [ -e "$file" ] || continue; ffmpeg -i "$file" -b:v 500k -b:a 96k -c:v libx264 "$HOME/Vidéos/MP4convert/500k-${file%.*}.mp4"; done'
alias mp4convert1000='for file in *.{mp4,mkv,avi,mov}; do [ -e "$file" ] || continue; ffmpeg -i "$file" -b:v 1000k -b:a 96k -c:v libx264 "$HOME/Vidéos/MP4convert/1000k-${file%.*}.mp4"; done'
alias mp4convert1500='for file in *.{mp4,mkv,avi,mov}; do [ -e "$file" ] || continue; ffmpeg -i "$file" -b:v 1500k -b:a 96k -c:v libx264 "$HOME/Vidéos/MP4convert/1500k-${file%.*}.mp4"; done'
alias mp4convert2000='for file in *.{mp4,mkv,avi,mov}; do [ -e "$file" ] || continue; ffmpeg -i "$file" -b:v 2000k -b:a 96k -c:v libx264 "$HOME/Vidéos/MP4convert/2000k-${file%.*}.mp4"; done'
alias mp4convert2500='for file in *.{mp4,mkv,avi,mov}; do [ -e "$file" ] || continue; ffmpeg -i "$file" -b:v 2500k -b:a 96k -c:v libx264 "$HOME/Vidéos/MP4convert/2500k-${file%.*}.mp4"; done'
alias mp4convert3000='for file in *.{mp4,mkv,avi,mov}; do [ -e "$file" ] || continue; ffmpeg -i "$file" -b:v 3000k -b:a 96k -c:v libx264 "$HOME/Vidéos/MP4convert/3000k-${file%.*}.mp4"; done'
alias mp4convert3500='for file in *.{mp4,mkv,avi,mov}; do [ -e "$file" ] || continue; ffmpeg -i "$file" -b:v 3500k -b:a 96k -c:v libx264 "$HOME/Vidéos/MP4convert/3500k-${file%.*}.mp4"; done'
alias mp4convert4000='for file in *.{mp4,mkv,avi,mov}; do [ -e "$file" ] || continue; ffmpeg -i "$file" -b:v 4000k -b:a 96k -c:v libx264 "$HOME/Vidéos/MP4convert/4000k-${file%.*}.mp4"; done'
alias mp4convert4500='for file in *.{mp4,mkv,avi,mov}; do [ -e "$file" ] || continue; ffmpeg -i "$file" -b:v 4500k -b:a 96k -c:v libx264 "$HOME/Vidéos/MP4convert/4500k-${file%.*}.mp4"; done'
alias mp4convert5000='for file in *.{mp4,mkv,avi,mov}; do [ -e "$file" ] || continue; ffmpeg -i "$file" -b:v 5000k -b:a 96k -c:v libx264 "$HOME/Vidéos/MP4convert/5000k-${file%.*}.mp4"; done'
alias mp4convert5500='for file in *.{mp4,mkv,avi,mov}; do [ -e "$file" ] || continue; ffmpeg -i "$file" -b:v 5500k -b:a 96k -c:v libx264 "$HOME/Vidéos/MP4convert/5500k-${file%.*}.mp4"; done'
alias mp4convert6000='for file in *.{mp4,mkv,avi,mov}; do [ -e "$file" ] || continue; ffmpeg -i "$file" -b:v 6000k -b:a 96k -c:v libx264 "$HOME/Vidéos/MP4convert/6000k-${file%.*}.mp4"; done'
alias mp4convertnextcloud='for file in *.{mp4,mkv,avi,mov}; do [ -e "$file" ] || continue; ffmpeg -i "$file" -b:v 6000k -c:v libx264 "$HOME/Vidéos/MP4convert/nextcloud-${file%.*}.mp4"; done'

```

### 2. Méthode GPU (vitesse éclair - VA-API Intel)

*Note : Adapté pour renderD129 (systèmes hybrides). Changez en renderD128 si nécessaire.*

```bash
alias gpu-mp4convert200='for file in *.{mp4,mkv,avi,mov}; do [ -e "$file" ] || continue; ffmpeg -hwaccel vaapi -vaapi_device /dev/dri/renderD129 -i "$file" -vf "format=nv12,hwupload" -c:v h264_vaapi -b:v 200k -b:a 96k "$HOME/Vidéos/MP4convert/gpu-200k-${file%.*}.mp4"; done'
alias gpu-mp4convert500='for file in *.{mp4,mkv,avi,mov}; do [ -e "$file" ] || continue; ffmpeg -hwaccel vaapi -vaapi_device /dev/dri/renderD129 -i "$file" -vf "format=nv12,hwupload" -c:v h264_vaapi -b:v 500k -b:a 96k "$HOME/Vidéos/MP4convert/gpu-500k-${file%.*}.mp4"; done'
alias gpu-mp4convert1000='for file in *.{mp4,mkv,avi,mov}; do [ -e "$file" ] || continue; ffmpeg -hwaccel vaapi -vaapi_device /dev/dri/renderD129 -i "$file" -vf "format=nv12,hwupload" -c:v h264_vaapi -b:v 1000k -b:a 96k "$HOME/Vidéos/MP4convert/gpu-1000k-${file%.*}.mp4"; done'
alias gpu-mp4convert1500='for file in *.{mp4,mkv,avi,mov}; do [ -e "$file" ] || continue; ffmpeg -hwaccel vaapi -vaapi_device /dev/dri/renderD129 -i "$file" -vf "format=nv12,hwupload" -c:v h264_vaapi -b:v 1500k -b:a 96k "$HOME/Vidéos/MP4convert/gpu-1500k-${file%.*}.mp4"; done'
alias gpu-mp4convert2000='for file in *.{mp4,mkv,avi,mov}; do [ -e "$file" ] || continue; ffmpeg -hwaccel vaapi -vaapi_device /dev/dri/renderD129 -i "$file" -vf "format=nv12,hwupload" -c:v h264_vaapi -b:v 2000k -b:a 96k "$HOME/Vidéos/MP4convert/gpu-2000k-${file%.*}.mp4"; done'
alias gpu-mp4convert2500='for file in *.{mp4,mkv,avi,mov}; do [ -e "$file" ] || continue; ffmpeg -hwaccel vaapi -vaapi_device /dev/dri/renderD129 -i "$file" -vf "format=nv12,hwupload" -c:v h264_vaapi -b:v 2500k -b:a 96k "$HOME/Vidéos/MP4convert/gpu-2500k-${file%.*}.mp4"; done'
alias gpu-mp4convert3000='for file in *.{mp4,mkv,avi,mov}; do [ -e "$file" ] || continue; ffmpeg -hwaccel vaapi -vaapi_device /dev/dri/renderD129 -i "$file" -vf "format=nv12,hwupload" -c:v h264_vaapi -b:v 3000k -b:a 96k "$HOME/Vidéos/MP4convert/gpu-3000k-${file%.*}.mp4"; done'
alias gpu-mp4convert3500='for file in *.{mp4,mkv,avi,mov}; do [ -e "$file" ] || continue; ffmpeg -hwaccel vaapi -vaapi_device /dev/dri/renderD129 -i "$file" -vf "format=nv12,hwupload" -c:v h264_vaapi -b:v 3500k -b:a 96k "$HOME/Vidéos/MP4convert/gpu-3500k-${file%.*}.mp4"; done'
alias gpu-mp4convert4000='for file in *.{mp4,mkv,avi,mov}; do [ -e "$file" ] || continue; ffmpeg -hwaccel vaapi -vaapi_device /dev/dri/renderD129 -i "$file" -vf "format=nv12,hwupload" -c:v h264_vaapi -b:v 4000k -b:a 96k "$HOME/Vidéos/MP4convert/gpu-4000k-${file%.*}.mp4"; done'
alias gpu-mp4convert4500='for file in *.{mp4,mkv,avi,mov}; do [ -e "$file" ] || continue; ffmpeg -hwaccel vaapi -vaapi_device /dev/dri/renderD129 -i "$file" -vf "format=nv12,hwupload" -c:v h264_vaapi -b:v 4500k -b:a 96k "$HOME/Vidéos/MP4convert/gpu-4500k-${file%.*}.mp4"; done'
alias gpu-mp4convert5000='for file in *.{mp4,mkv,avi,mov}; do [ -e "$file" ] || continue; ffmpeg -hwaccel vaapi -vaapi_device /dev/dri/renderD129 -i "$file" -vf "format=nv12,hwupload" -c:v h264_vaapi -b:v 5000k -b:a 96k "$HOME/Vidéos/MP4convert/gpu-5000k-${file%.*}.mp4"; done'
alias gpu-mp4convert5500='for file in *.{mp4,mkv,avi,mov}; do [ -e "$file" ] || continue; ffmpeg -hwaccel vaapi -vaapi_device /dev/dri/renderD129 -i "$file" -vf "format=nv12,hwupload" -c:v h264_vaapi -b:v 5500k -b:a 96k "$HOME/Vidéos/MP4convert/gpu-5500k-${file%.*}.mp4"; done'
alias gpu-mp4convert6000='for file in *.{mp4,mkv,avi,mov}; do [ -e "$file" ] || continue; ffmpeg -hwaccel vaapi -vaapi_device /dev/dri/renderD129 -i "$file" -vf "format=nv12,hwupload" -c:v h264_vaapi -b:v 6000k -b:a 96k "$HOME/Vidéos/MP4convert/gpu-6000k-${file%.*}.mp4"; done'
alias gpu-mp4convertnextcloud='for file in *.{mp4,mkv,avi,mov}; do [ -e "$file" ] || continue; ffmpeg -hwaccel vaapi -vaapi_device /dev/dri/renderD129 -i "$file" -vf "format=nv12,hwupload" -c:v h264_vaapi -b:v 6000k "$HOME/Vidéos/MP4convert/gpu-nextcloud-${file%.*}.mp4"; done'

```

---

## 🧪 Exemple complet d'alias décortiqué

Voici l'alias `gpu-mp4convert3000` illustré pour comprendre sa structure :

```bash
alias gpu-mp4convert3000='
    for file in *.{mp4,mkv,avi,mov}; # 1. boucle pour chaque fichier vidéo du dossier actuel
    do 
        [ -e "$file" ] || continue; # 2. sécurité si aucun fichier ne correspond
        ffmpeg -hwaccel vaapi -vaapi_device /dev/dri/renderD129 \ # 3. Hardware Acceleration : cible l iGPU Intel spécifique
            -i "$file" \ # 4. Input : fichier source (mkv, avi, mp4, mov)
            -vf "format=nv12,hwupload" \ # 5. Video Filter : prépare le format pour le GPU
            -c:v h264_vaapi \ # 6. Codec Video : utilise l encodeur matériel
            -b:v 3000k -b:a 96k \ # 7. Bitrate : règle les débits vidéo et audio
            "$HOME/Vidéos/MP4convert/gpu-3000k-${file%.*}.mp4"; # 8. Output : nettoie l extension originale et force le .mp4
    done
'

```

---

## Dépannage (Troubleshooting)

### Erreur : `get chip id failed: -1 [13]` ou `Permission denied`

Si vous obtenez cette erreur avec un alias `gpu-` alors que vous êtes bien dans le groupe `render`, cela peut signifier deux choses :

1. **Conflit de périphérique :** Vous ciblez peut-être la mauvaise carte (D128 au lieu de D129).
2. **Obsolescence :** Votre processeur est trop ancien (ex : Sandy Bridge) pour les pilotes récents.

**Solution :** Ne perdez pas de temps à essayer de forcer le GPU. Votre CPU possède suffisamment de threads pour gérer la conversion via les alias **CPU** classiques sans le préfixe `gpu-`.

## Démonstration

Dans cette vidéo, je vous montre comment j'utilise mes alias personnalisés pour automatiser et accélérer le traitement vidéo sur mon matériel reconditionné.
<br>
<iframe 
  title="Boostez vos conversions vidéo sur Linux : CPU vs GPU (VA-API Intel)" 
  src="https://peertube.blablalinux.be/videos/embed/hKp8v7Zm4waNeoEE6Aj5pR" 
  allow="fullscreen" 
  loading="lazy" 
  sandbox="allow-same-origin allow-scripts allow-popups allow-forms" 
  style="border: 0; width: 100%; aspect-ratio: 16 / 9;" 
  allowfullscreen>
</iframe>
<p align="center">
  <a href="https://peertube.blablalinux.be/w/hKp8v7Zm4waNeoEE6Aj5pR" target="_blank">
    📺 <b>Voir la vidéo : Boostez vos conversions vidéo sur Linux : CPU vs GPU (VA-API Intel)</b>
  </a>
</p>

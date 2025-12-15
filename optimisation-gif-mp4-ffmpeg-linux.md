---
title: Conversion et optimisation de GIF vers MP4 avec FFmpeg
description: Convertissez des fichiers GIF animés en MP4 optimisés pour le web et les systèmes Linux légers grâce à la commande FFmpeg. Apprenez à utiliser les options clés comme le CRF pour contrôler la qualité et la taille du fichier.
published: true
date: 2025-12-15T22:33:18.848Z
tags: mp4, ffmpeg, video, gif, conversion, optimisation
editor: markdown
dateCreated: 2025-12-15T22:33:18.848Z
---

## Introduction
Cette page documente la méthode recommandée pour convertir un fichier GIF animé (`.gif`) en un fichier vidéo MP4 (`.mp4`) optimisé pour le web. Cette conversion permet de réduire considérablement la taille du fichier tout en conservant l'animation, ce qui améliore la performance des pages web et des systèmes à faibles ressources.

## Prérequis
Pour exécuter cette commande, le logiciel `ffmpeg` doit être installé sur votre système.

* **Vérification de l'installation :**
```bash
ffmpeg -version

```


* **Installation sur les systèmes basés sur Debian/Ubuntu (Debian, Ubuntu, Linux Mint, Emmabuntüs) :**
```bash
sudo apt update
sudo apt install ffmpeg

```



## La commande d'optimisation
Voici la commande complète, suivie d'une explication détaillée de chaque option :

```bash
ffmpeg -i animated.gif -movflags faststart -pix_fmt yuv420p -vf "scale=trunc(iw/2)*2:trunc(ih/2)*2" video.mp4

```

| Partie de la commande | Description |
| --- | --- |
| **`ffmpeg`** | L'outil de ligne de commande pour la conversion multimédia. |
| **`-i animated.gif`** | Définit le **fichier d'entrée** (`-i`). |
| **`-movflags faststart`** | **Optimisation web :** Déplace les métadonnées en début de fichier, permettant au lecteur de commencer la lecture avant le téléchargement complet. |
| **`-pix_fmt yuv420p`** | **Compatibilité :** Définit le format de pixel `yuv420p`, essentiel pour une **compatibilité maximale** avec les navigateurs. |
| **`-vf "scale=..."`** | **Filtre vidéo :** Applique un filtre de redimensionnement. |
| **`trunc(iw/2)*2:trunc(ih/2)*2`** | **Ajustement de la résolution :** S'assure que les dimensions sont des **multiples de 2**, requis par le format `yuv420p`. |
| **`video.mp4`** | Le **fichier de sortie** au format MP4. |

## Options avancées : contrôle de la qualité et de la compression (CRF)
Pour les vidéos encodées en H.264, vous pouvez contrôler la qualité de la compression à l'aide de l'option **Constant Rate Factor (`-crf`)**.

Plus la valeur est élevée (de **0 à 51**), plus la compression est forte (petite taille, qualité inférieure). Une valeur de **23** est la valeur par défaut recommandée.

### Exemple avec CRF (Taille réduite)
```bash
ffmpeg -i animated.gif -movflags faststart -pix_fmt yuv420p -crf 28 -vf "scale=trunc(iw/2)*2:trunc(ih/2)*2" video_taille_reduite.mp4

```

### Intégration HTML : Remplacer un GIF par un MP4
Pour que le fichier MP4 se comporte comme un GIF (lecture automatique et boucle), utilisez la balise `<video>` avec les attributs suivants (`autoplay`, `loop`, `muted`, `playsinline`).

| Attribut | Rôle |
| --- | --- |
| **`autoplay`** | Démarre la lecture de la vidéo. |
| **`loop`** | Répète la vidéo indéfiniment. |
| **`muted`** | **Coupe le son** (essentiel pour l'autostart dans les navigateurs). |
| **`playsinline`** | Assure la lecture dans l'espace de la page (mobile). |

#### 1. Code d'intégration standard
```html
<video autoplay loop muted playsinline width="100%" height="auto">
  <source src="video_opti.mp4" type="video/mp4">
  Votre navigateur ne supporte pas la balise vidéo.
</video>

```

#### 2. Code avec image de prévisualisation (Poster)
L'attribut `poster` permet d'afficher une image statique (`preview_image.jpg`) tant que la vidéo n'est pas chargée.

```html
<video autoplay loop muted playsinline poster="preview_image.jpg" width="100%" height="auto">
  <source src="video_opti.mp4" type="video/mp4">
  Votre navigateur ne supporte pas la balise vidéo.
</video>

```
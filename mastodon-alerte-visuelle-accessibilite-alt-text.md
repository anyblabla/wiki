---
title: Mastodon - Alerte visuelle pour l'accessibilité (alt-text)
description: Améliorez l'accessibilité de Mastodon avec ce script CSS. Il affiche une bordure hachurée personnalisée autour des images, vidéos et GIF sans texte alternatif pour ne plus oublier les descriptions.
published: true
date: 2025-12-24T17:18:39.644Z
tags: mastodon, css, design, tutoriel, accessibilité, fediverse
editor: markdown
dateCreated: 2025-12-24T01:36:11.083Z
---

## Présentation

Sur Mastodon et le Fediverse, l'accessibilité est une valeur fondamentale. Cependant, il arrive d'oublier d'ajouter une description textuelle (texte alternatif) sur un média. Ce tutoriel explique comment ajouter un style CSS personnalisé qui entourera automatiquement d'une bordure hachurée tout média publié sans texte alternatif.

> **Remerciement :** Un grand merci à **<a href="[https://mastodon.jfmblinux.fr/@jfmblinux](https://mastodon.jfmblinux.fr/@jfmblinux)" target="_blank" rel="noopener">@jfmblinux@mastodon.jfmblinux.fr</a>** qui m'a partagé le code initial.

## Option 1 : Code CSS Standard (Images et Audio)

Ce code cible les images classiques et les lecteurs audio. Il est simple et efficace pour la majorité des utilisateurs.

```css
/* --- ALERTE MÉDIAS SANS TEXTE ALTERNATIF --- */
.media-gallery__item-thumbnail img:not([alt]),
.audio-player__canvas:not([title]),
.media-gallery__item-thumbnail img[alt=""], 
.audio-player__canvas[title=""] {
    box-sizing: border-box !important;
    border: 5px solid !important;
    border-image-slice: 1 !important;
    border-image-source: repeating-linear-gradient(
        -55deg,
        #125856,        /* Vert Blabla Linux */
        #125856 15px,
        #F88013 15px,   /* Orange Blabla Linux */
        #F88013 30px
    ) !important;
}

```

## Option 2 : Code CSS Universel (Images, Vidéos et GIF)

**Version recommandée.** Elle inclut un correctif pour les vidéos et les GIF. Sur Mastodon, la description de ces médias est souvent stockée dans l'attribut `aria-label`. Ce code vérifie les deux attributs (`title` et `aria-label`) pour éviter que la bordure ne s'affiche si une description est bien présente.

```css
/* --- ALERTE UNIVERSELLE (Version sans faux positifs) --- */

/* 1. Images et Audio */
.media-gallery__item-thumbnail img:not([alt]),
.media-gallery__item-thumbnail img[alt=""],
.audio-player__canvas:not([title]),
.audio-player__canvas[title=""],

/* 2. Vidéos et GIF (Vérification double : title ET aria-label) */
.video-player video:not([title]):not([aria-label]),
.media-gallery__gifv video:not([title]):not([aria-label]),
.video-player video[title=""][aria-label=""],
.media-gallery__gifv video[title=""][aria-label=""] {
    box-sizing: border-box !important;
    border: 5px solid !important;
    border-image-slice: 1 !important;
    border-image-source: repeating-linear-gradient(
        -55deg,
        #125856,        /* Vert Blabla Linux */
        #125856 15px,
        #F88013 15px,   /* Orange Blabla Linux */
        #F88013 30px
    ) !important;
}

```

## Décorticage technique : comment ça marche ?

### 1. Les sélecteurs (la cible)

Le code repère les erreurs d'accessibilité de deux manières :

* **L'absence d'attribut** (`:not([alt])`) : l'attribut est totalement manquant dans le code HTML.
* **L'attribut vide** (`[alt=""]`) : l'attribut existe (on a cliqué sur "ajouter une description") mais rien n'a été écrit dedans.

### 2. L'aspect visuel (le ruban)

* **repeating-linear-gradient** : cette fonction crée l'effet de hachures.
* **L'angle (-55deg)** : incline les bandes pour donner cet aspect "ruban de chantier".
* **Les paliers (15px / 30px)** : ils définissent la largeur des bandes colorées. Plus les chiffres sont petits, plus les hachures sont serrées.

### 3. La mise en page sécurisée

L'utilisation de `box-sizing: border-box` est cruciale. Elle force la bordure à s'afficher **vers l'intérieur** de l'image. Sans cela, l'épaisseur de la bordure (5px) décalerait les éléments voisins, ce qui pourrait "casser" la mise en page de votre interface Mastodon.

## Personnalisation possible

Vous pouvez adapter ce code à vos propres goûts ou à votre identité visuelle :

| Propriété | Rôle | Suggestion de modification |
| --- | --- | --- |
| `border: 5px` | Épaisseur du cadre | Passez à `2px` ou `3px` pour plus de discrétion. |
| `15px / 30px` | Largeur des hachures | Augmentez à `30px / 60px` pour de larges bandes. |
| `#125856` | Couleur 1 (Vert) | Remplacez par `#000000` (Noir). |
| `#F88013` | Couleur 2 (Orange) | Remplacez par `#FFFF00` (Jaune) pour un look "Danger". |

## Procédure d'installation

### Pour un usage personnel (utilisateur)

1. **Installer Stylus :** <a href="[https://addons.mozilla.org/fr/firefox/addon/styl-us/](https://addons.mozilla.org/fr/firefox/addon/styl-us/)" target="_blank" rel="noopener">Version Firefox</a> | <a href="[https://chromewebstore.google.com/detail/stylus/clngdbkpkpeebahjckkjfobafhncgmne](https://chromewebstore.google.com/detail/stylus/clngdbkpkpeebahjckkjfobafhncgmne)" target="_blank" rel="noopener">Version Chrome / Vivaldi</a>
2. Cliquez sur l'icône de l'extension -> **"Gérer"** -> **"Créer un nouveau style"**.
3. Spécifiez le domaine de votre instance (ex: `mastodon.social`).
4. Collez le code CSS choisi et enregistrez.

### Pour une instance (administrateur)

1. Rendez-vous dans **Administration** > **Configuration de l'instance** > **Apparence**.
2. Collez le code dans le champ **CSS personnalisé** et enregistrez.

---

> **Note :** Ce code est un rappel visuel pour nous encourager à rendre le web plus accessible à tous.

![mastodon-alerte-visuelle-accessibilité-alt-text.jpg](/mastodon-alerte-visuelle-accessibilité-alt-text/mastodon-alerte-visuelle-accessibilité-alt-text.jpg)
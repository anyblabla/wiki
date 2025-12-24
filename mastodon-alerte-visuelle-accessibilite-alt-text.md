---
title: Mastodon - Alerte visuelle pour l'accessibilité (alt-text)
description: Améliorez l'accessibilité de Mastodon avec ce script CSS. Il affiche une bordure hachurée personnalisée autour des images, vidéos et GIF sans texte alternatif pour ne plus oublier les descriptions.
published: true
date: 2025-12-24T16:34:41.302Z
tags: mastodon, css, design, tutoriel, accessibilité, fediverse
editor: markdown
dateCreated: 2025-12-24T01:36:11.083Z
---

## Présentation

Sur Mastodon et le Fediverse, l'accessibilité est une valeur fondamentale. Cependant, il arrive d'oublier d'ajouter une description textuelle (texte alternatif) sur un média.

Ce tutoriel explique comment ajouter un style CSS personnalisé qui entourera automatiquement d'une bordure hachurée (style ruban de signalisation) tout média publié sans texte alternatif. C'est un excellent outil d'auto-discipline et de sensibilisation.

> **Remerciement :** Un grand merci à **[@jfmblinux@mastodon.jfmblinux.fr](https://www.google.com/search?q=https://mastodon.jfmblinux.fr/%40jfmblinux)** qui m'a partagé ce code précieux pour la communauté.

## Le code CSS (version Blabla Linux)

Ce code utilise les couleurs identitaires de **Blabla Linux** (vert et orange) pour signaler les médias non conformes.

```css
/* --- ALERTE MÉDIAS SANS TEXTE ALTERNATIF --- */
/* Cible les images, vidéos, GIF et lecteurs audio sans alt ou title */

.media-gallery__item-thumbnail img:not([alt]),
.audio-player__canvas:not([title]),
.video-player video:not([title]), 
.media-gallery__gifv video:not([title]),
.media-gallery__item-thumbnail img[alt=""], 
.audio-player__canvas[title=""], 
.video-player video[title=""], 
.media-gallery__gifv video[title=""] {
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

Le code repère deux types d'erreurs d'accessibilité dans le code HTML :

* **:not([alt])** ou **:not([title])** : l'attribut est totalement absent.
* **[alt=""]** ou **[title=""]** : l'attribut est présent mais vide (validation trop rapide).

### 2. L'aspect visuel (le dégradé)

La propriété `repeating-linear-gradient` crée l'effet visuel :

* **L'angle (-55deg)** : incline les bandes pour l'aspect "chantier".
* **Les paliers (15px / 30px)** : définissent la largeur des bandes colorées.

### 3. La mise en page (le cadre)

L'utilisation de `box-sizing: border-box` est cruciale. Elle force la bordure à s'afficher vers l'intérieur de l'image pour ne pas briser la mise en page de l'instance.

## Personnalisation possible

| Propriété | Effet | Suggestion |
| --- | --- | --- |
| `border: 5px` | Épaisseur de la ligne | Réduisez à `3px` pour plus de finesse. |
| `15px / 30px` | Largeur des hachures | Augmentez pour des bandes plus larges. |
| Couleurs | L'identité visuelle | `#000` et `#FFFF00` pour un look "danger" classique. |

## Procédure d'installation

### Pour un usage personnel (utilisateur)

Si vous souhaitez voir ces alertes uniquement sur votre navigateur (sur votre propre instance ou celles des autres), installez l'extension libre **Stylus** :

1. **Installer Stylus :** <a href="https://addons.mozilla.org/fr/firefox/addon/styl-us/" target="_blank" rel="noopener">Version Firefox</a> | <a href="https://chromewebstore.google.com/detail/stylus/clngdbkpkpeebahjckkjfobafhncgmne" target="_blank" rel="noopener">Version Chrome / Vivaldi</a>
2. Créez un nouveau style pour le domaine de votre instance (ex: `mastodon.social`).
3. Collez le code CSS et enregistrez.

### Pour une instance (administrateur)

Si vous gérez votre propre instance Mastodon :

1. Rendez-vous dans **Administration** > **Configuration de l'instance** > **Apparence**.
2. Collez le code dans le champ **CSS personnalisé**.
3. Enregistrez les modifications pour que l'alerte soit visible par tous vos membres.

---

> **Note sur l'accessibilité :** Ce code est un rappel visuel. Son but est de nous encourager à ne pas oublier les personnes utilisant des lecteurs d'écran pour naviguer.

![mastodon-alerte-visuelle-accessibilité-alt-text.jpg](/mastodon-alerte-visuelle-accessibilité-alt-text/mastodon-alerte-visuelle-accessibilité-alt-text.jpg)
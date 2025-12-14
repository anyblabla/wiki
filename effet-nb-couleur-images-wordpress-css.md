---
title: Effet noir et blanc au repos, couleur au survol (WordPress/Astra)
description: Appliquez un filtre noir et blanc par défaut sur les images d'articles WordPress (thème Astra) et activez la couleur au survol de la souris. Ce guide CSS étape par étape vous montre comment ajouter un style dynamique et professionnel à votre blog.
published: true
date: 2025-12-14T09:47:49.991Z
tags: css, wordpress, astra, thème, featured image, technique, webdesign, filtrer
editor: markdown
dateCreated: 2025-12-14T09:20:54.224Z
---

Ce guide explique comment ajouter un effet visuel dynamique sur les images de mise en avant (images vedettes) de la page d'accueil d'un blog WordPress utilisant le thème Astra, en utilisant uniquement du code CSS personnalisé.

## I. Objectif de l'implémentation
L'objectif est d'appliquer un filtre de couleur **noir et blanc** (*Grayscale*) par défaut sur toutes les images de la page d'accueil, et de retirer ce filtre (image en **couleur**) avec une transition douce lorsque l'utilisateur passe la souris sur l'image (effet `:hover`).

## II. Code CSS à injecter
Le code suivant est optimisé pour fonctionner avec le thème **Astra** et la classe standard des images de mise en avant WordPress (`.wp-post-image`). L'utilisation de `!important` garantit que le style personnalisé prime sur les styles par défaut du thème.

> **Note technique :** Nous ciblons la classe `.wp-post-image`, car c'est la balise `<img>` elle-même qui reçoit le filtre CSS.

```css
/* ========================================================== */
/* EFFET N&B -> COULEUR AU SURVOL (pour la page d'accueil .home) */
/* Thème : Astra | Cible : .wp-post-image */
/* ========================================================== */

.home .wp-post-image {
    /* Paramètres de base pour l'état N&B */
    filter: grayscale(100%) !important;
    -webkit-filter: grayscale(100%) !important; /* Compatibilité navigateurs */
    
    /* Transition douce : applique un changement sur 0.5s pour le filtre et 0.3s pour le zoom */
    transition: filter 0.5s ease-in-out, transform 0.3s ease-in-out;
    
    /* S'assure que l'image peut appliquer les transformations (scale) */
    display: block; 
}

.home .wp-post-image:hover {
    /* Au survol : Retour à la couleur (0%) */
    filter: grayscale(0%) !important;
    -webkit-filter: grayscale(0%) !important;
    
    /* Léger zoom visuel pour accentuer l'effet de survol */
    transform: scale(1.05);
}

```

## III. Étapes d'implémentation dans WordPress
1. **Accès au personnalisateur :**
* Dans le tableau de bord WordPress, naviguez vers **Apparence** > **Personnaliser**.


2. **Insertion du code :**
* Dans la barre latérale du personnalisateur, cliquez sur l'onglet **CSS additionnel** (ou **CSS personnalisé**).
* Collez le code CSS ci-dessus dans l'éditeur.


3. **Sauvegarde et publication :**
* Cliquez sur le bouton **Publier** en haut de la barre latérale pour enregistrer les modifications.



## IV. Dépannage et vérification
### 1. Le code ne s'applique pas
Si l'effet n'apparaît pas, assurez-vous de vider :

* Le cache de votre navigateur (Ctrl+Shift+R ou Cmd+Shift+R).
* Le cache de tout plugin d'optimisation WordPress (si vous en utilisez un).

### 2. Comprendre les propriétés utilisées
* `filter: grayscale(100%)` : La propriété CSS essentielle qui applique le filtre noir et blanc.
* `transition` : Assure une animation fluide (0.5 seconde) entre les états.
* `transform: scale(1.05)` : Ajoute un léger zoom au survol pour améliorer l'effet visuel.

## V. Résultat

![nb.gif](/effet-nb-couleur-images-wordpress-css/nb.gif)
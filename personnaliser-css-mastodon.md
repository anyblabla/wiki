---
title: Personnaliser l'apparence de Mastodon avec du CSS
description: Découvrez comment modifier l'identité visuelle de votre instance Mastodon. Guide basé sur la configuration réelle de Blabla Linux pour harmoniser votre serveur avec votre charte.
published: false
date: 2025-12-23T14:09:45.319Z
tags: mastodon, css, auto-hébergement, design, administration, tutoriel
editor: markdown
dateCreated: 2025-12-23T14:09:45.319Z
---

Pourquoi se contenter du bleu par défaut ? L'administration de Mastodon permet d'injecter du code CSS pour modifier l'interface utilisateur (version Web) sans toucher au code source de l'application. Cela permet d'harmoniser votre instance avec votre blog ou votre charte graphique personnelle.

## Pourquoi modifier le CSS ?

L'identité visuelle est essentielle pour créer un sentiment d'appartenance à une communauté. Sur **mastodon.blablalinux.be**, l'objectif était de retrouver les couleurs du blog et du wiki (le vert canard et l'orange) pour offrir une expérience cohérente sur l'ensemble de l'écosystème **Blabla Linux**.

Ce tutoriel s'appuie sur la configuration réelle que j'ai mise en place. Vous pouvez l'utiliser telle quelle ou l'adapter avec vos propres couleurs.

## Le code utilisé par Blabla Linux

Voici le code complet injecté sur mon instance. Il sépare la structure (en vert) des éléments d'interaction et de lecture (en orange).

```css
/* --- Surcharge CSS Blabla Linux --- */

:root {
  /* Vos variables de couleurs personnalisées */
  --blabla-green: #125856;
  --blabla-orange: #F88013;
}

/* 1. Structure et boutons principaux */
.button.button-primary,
.admin-wrapper .sidebar ul li a.selected,
.navigation-bar__profile-edit .button,
.column-header,
.tabs-bar__wrapper {
    background-color: var(--blabla-green) !important;
    border-color: var(--blabla-green) !important;
    color: #ffffff !important;
}

.button.button-primary:hover {
    background-color: #0d4442 !important;
}

/* 2. Liens et hashtags */
.status__content a,
.status__content a.mention,
.hashtag-button,
.column-link {
    color: var(--blabla-orange) !important;
}

/* 3. Accents et notifications */
.icon-button.active,
.notification__message .fa-retweet, 
.status__prepend .fa-retweet,
.pulldown-indicator,
.account__section-headline a.active::after,
.notification__filter-bar a.active::after,
.hero-widget__counter span,
.column-header__button.active {
    color: var(--blabla-orange) !important;
}

/* 4. Barre de défilement */
::-webkit-scrollbar-thumb {
    background: var(--blabla-green) !important;
    border-radius: 4px;
}

```

## Décortiquons le code pour l'adapter

Le code est divisé en plusieurs sections logiques pour vous permettre de l'ajuster selon vos besoins :

### Les variables de couleur

La section `:root` définit vos variables globales.

* **Modification :** Changez simplement les codes hexadécimaux (ex: `#125856`) par vos propres couleurs pour que le changement s'applique à tout le bloc.

### La structure et les boutons

* **.column-header** : Modifie le haut des colonnes (Accueil, Notifications).
* **.button-primary** : Change la couleur du bouton de publication et des boutons "Suivre".
* **!important** : Cette mention est capitale. Elle force Mastodon à ignorer ses couleurs par défaut pour utiliser les vôtres, quel que soit le thème (sombre ou clair) choisi par l'utilisateur.

### Les liens et les textes cliquables

Dans l'exemple de Blabla Linux, j'utilise l'orange pour que les liens soient bien visibles sur un fond sombre. Les **hashtags** et les **mentions** sont également ciblés ici.

### Les indicateurs d'état

Cette partie gère les "Boosts" (icône de partage), les étoiles de favoris et les points de notification. Utiliser une couleur contrastée permet à l'utilisateur de voir immédiatement ce qui est actif.

## Comment appliquer ce code ?

La procédure est simple et ne nécessite pas de redémarrer vos conteneurs Docker :

1. Connectez-vous à votre instance Mastodon avec un compte administrateur.
2. Allez dans les **Préférences**.
3. Dans le menu latéral, cliquez sur **Administration**, puis sur **Configuration du serveur**.
4. Cliquez sur l'onglet **Apparence**.
5. Repérez la zone de texte nommée **CSS personnalisé**.
6. Collez votre code et cliquez sur **Enregistrer les modifications**.
---
title: >-
  Changer la couleur principale (primaire) de votre Wiki.js 2.x (guide injection
  HEAD)
description: >-
  Cette documentation explique comment modifier les couleurs par défaut du thème
  principal de votre installation Wiki.js (tel que le thème par défaut ou tout
  autre thème basé sur le framework Vuetify).
published: true
date: '2026-11-02T09:39:38.896Z'
tags:
  - wikijs
  - head
  - html
  - color
  - injection
editor: markdown
dateCreated: '2025-10-30T15:54:26.600Z'
---

### 📝 Introduction

**L'objectif est de remplacer la couleur principale de mise en surbrillance** (boutons, titres, éléments actifs) et d'autres couleurs spécifiques en utilisant l'outil d'**Injection de code CSS** de Wiki.js.

Cette méthode est **non destructive**, car elle utilise un code CSS (`<style>`) injecté dans la section `<head>` du document pour surcharger les styles existants, garantissant que vos modifications persistent même après les mises à jour du thème.

-----

### 1\. ⚙️ Procédure d'injection

Suivez ces étapes pour placer le code de personnalisation au bon endroit :

1.  Accédez à la **zone d'administration** de votre Wiki.js.
2.  Naviguez vers la section **Thème**.
3.  Cliquez sur l'onglet **Injection de code**.
4.  Collez le code fourni à l'étape 2 dans le champ **Injection HTML dans le HEAD**.
5.  Cliquez sur **Appliquer** pour sauvegarder et tester les changements.

-----

### 2\. 🎨 Le code CSS d'override à injecter

Le code ci-dessous utilise des **variables CSS** pour définir vos couleurs et les applique en forçant la priorité sur les classes de couleur par défaut de Vuetify.

  * **Couleur principale (`--override`) :** `#125856` (vert sombre/canard)
  * **Couleur du texte (`--textcolor`) :** `#F88013` (orange)

#### Code à copier/coller

Collez le bloc complet ci-dessous dans le champ **Injection HTML dans le HEAD** :

```html
<style>
/* 1. Définition des variables : Vos couleurs personnalisées */
:root { 
    /* Couleur principale : Définit le vert sombre pour les éléments primaires */
    --override: #125856; 
    /* Couleur du texte : Définit l'orange pour certains textes */
    --textcolor: #F88013; 
}

/* 2. Styles d'override (Surcharge des classes Vuetify) */

/* Couleur de fond et de bordure pour les éléments 'primary' et 'deep-purple' */
.v-application .primary,
.v-application .deep-purple {
    background-color: var(--override) !important;
    border-color: var(--override) !important;
}

/* Couleur du titre H1 et de sa ligne décorative */
.v-main .contents h1 { 
    color: var(--override) !important; 
}
.v-main .contents h1:after { 
    background: linear-gradient(90deg, var(--override), rgba(255, 255, 255, 0)) !important; 
}

/* Couleur de fond de l'en-tête des commentaires */
.comments-header { 
    background-color: var(--override) !important; 
}

/* Style de la barre de défilement verticale */
.__bar-is-vertical { 
    background: rgba(255, 255, 255, 0.75) !important; 
}

/* Couleur du titre des éléments de liste (dans les menus, navigation latérale) */
.v-list-item__title { 
    color: var(--textcolor) !important; 
}

/* Surcharge des classes de couleur 'blue' de Vuetify */
.v-application .blue {
    background-color: rgba(0, 0, 0, 0.1) !important;
}
.v-application .blue.darken-2,
.v-application .blue.darken-3 {
    background-color: rgba(0, 0, 0, 0.2) !important;
}
</style>
```

-----

### 3\. ✏️ Comment modifier les couleurs

Pour adapter le code à votre propre charte graphique, modifiez les deux lignes suivantes au début du code en remplaçant les codes hexadécimaux :

```css
:root { 
    /* Remplacez #125856 par votre code couleur principal (ex: #007bff pour le bleu) */
    --override: #125856; 
    
    /* Remplacez #F88013 par votre code couleur pour certains textes (ex: #333333 pour le noir) */
    --textcolor: #F88013; 
}
```

-----

## 📱 Aller plus loin : optimisation du rendu mobile

Après avoir personnalisé les couleurs de votre thème, vous souhaiterez peut-être améliorer l'expérience utilisateur sur smartphone. Le rendu mobile par défaut de Wiki.js peut présenter des problèmes de défilement horizontal et d'affichage des boutons.

Nous avons créé un guide expliquant comment injecter du CSS additionnel pour rendre le thème plus **acceptable** sur les petits écrans pour vos visiteurs.

👉 **Consultez la page :** [**Amélioration du rendu mobile pour les visiteurs (Wiki.js)**](https://wiki.blablalinux.be/fr/wikijs-css)

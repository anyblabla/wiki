---
title: Effet de neige léger sur Wiki.js
description: Guide d'intégration d'un effet visuel de flocons de neige léger sur Wiki.js. Utilise l'injection de code CSS et JavaScript pour une performance optimale sans toucher aux fichiers sources.
published: true
date: 2025-12-13T17:09:01.658Z
tags: css, vuetify, javascript, personnalisation, snow, neige
editor: markdown
dateCreated: 2025-12-13T16:24:19.006Z
---

## 1. Introduction

Cette page documente la méthode pour ajouter un effet de flocons de neige discret et performant à l'ensemble du site **wiki.blablalinux.be**.

L'objectif est d'utiliser la fonctionnalité d'injection de code intégrée à l'administration de Wiki.js pour insérer du CSS et du JavaScript, garantissant que l'effet ne sera **jamais écrasé** lors des mises à jour du système.

## 2. Code CSS : le style et l'animation

Le CSS est responsable de la forme des flocons, de leur positionnement fixe, et surtout, de l'animation de chute via les *keyframes*.

### 2.1 Emplacement de l'injection

Le code CSS doit être placé dans la fenêtre dédiée au remplacement de la feuille de style.

> 📍 **Chemin précis :** Administration → Theme → Injection de code → Remplacement de CSS

### 2.2 Code à ajouter

Ce code doit être ajouté à la suite de votre code CSS existant :

```css
#snow-container {
    position: fixed; 
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none; /* Crucial : permet d'interagir avec le contenu */
    z-index: 9999; /* Assure la superposition au-dessus de l'interface Vuetify */
    overflow: hidden;
}

.flake {
    position: absolute;
    background: #fff; /* Couleur par défaut : blanc */
    border-radius: 50%;
    opacity: 0.8;
    animation-name: fall;
    animation-timing-function: linear;
    animation-iteration-count: infinite;
}

/* Définition de l'animation de chute */
@keyframes fall {
    0% {
        transform: translateY(-10vh); /* Départ au-dessus de l'écran */
    }
    100% {
        transform: translateY(100vh); /* Arrivée sous l'écran */
    }
}
```

## 3. Code JavaScript : la création et la logique

Le JavaScript crée dynamiquement les éléments HTML et leur attribue des propriétés aléatoires (taille, position, vitesse) pour un effet plus naturel.

### 3.1 Emplacement de l'injection

Le JavaScript doit être placé à la fin du corps de la page (`<body>`), ce qui est la meilleure pratique pour les performances.

> 📍 **Chemin précis :** Administration → Theme → Injection de code → Injection HTML dans le body

### 3.2 Code à ajouter

Collez ce bloc complet (incluant la balise `<script>`) dans cet emplacement :

```html
<script>
document.addEventListener('DOMContentLoaded', () => {
    // Création du conteneur
    const snowContainer = document.createElement('div');
    snowContainer.id = 'snow-container';
    document.body.appendChild(snowContainer); 

    const numberOfFlakes = 60; // DENSITÉ

    for (let i = 0; i < numberOfFlakes; i++) {
        const flake = document.createElement('div');
        flake.classList.add('flake');
        
        // Position horizontale aléatoire
        flake.style.left = `${Math.random() * 100}%`; 
        
        // Position verticale aléatoire de départ hors écran
        flake.style.top = `${-Math.random() * 100}vh`; 
        
        // Taille aléatoire
        const size = Math.random() * 3 + 3; 
        flake.style.width = `${size}px`;
        flake.style.height = `${size}px`;
        
        // Durée (vitesse) et délai (départ) aléatoires
        flake.style.animationDuration = `${Math.random() * 8 + 12}s`; 
        flake.style.animationDelay = `${Math.random() * 8}s`; 

        snowContainer.appendChild(flake);
    }
});
</script>
```

## 4. Personnalisation des paramètres

L'un des avantages de cette méthode est que vous pouvez facilement ajuster l'effet en modifiant quelques variables dans les codes injectés.

### 4.1 Densité et mouvement (JavaScript)

| Paramètre | Ligne de code | Description et ajustement |
| --- | --- | --- |
| **Densité** | `const numberOfFlakes = 60;` | Nombre total de flocons affichés à l'écran. Augmenter la valeur rend la neige plus dense. |
| **Taille** | `const size = Math.random() * 3 + 3;` | Plage de taille (3px à 6px). Pour des flocons plus petits : `Math.random() * 2 + 2;` (2px à 4px). |
| **Vitesse (lenteur)** | `flake.style.animationDuration = ...` | Temps de chute en secondes. Des valeurs plus grandes rendent la chute plus lente. |

### 4.2 Couleur et opacité (CSS)

Ces paramètres sont gérés par les règles CSS appliquées à la classe `.flake`.

| Paramètre | Bloc de code CSS | Description et ajustement |
| --- | --- | --- |
| **Couleur** | `.flake { background: #fff; }` | Modifiez la couleur (exemple : `#ADD8E6` pour bleu clair). |
| **Opacité** | `.flake { opacity: 0.8; }` | Diminuer la valeur (ex : `0.5`) rend les flocons plus transparents. |
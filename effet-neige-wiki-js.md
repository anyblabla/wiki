---
title: Effet de neige léger sur Wiki.js
description: Guide d'intégration d'un effet visuel de flocons de neige léger sur Wiki.js. Utilise l'injection de code CSS et JavaScript pour une performance optimale sans toucher aux fichiers sources.
published: false
date: 2025-12-13T16:24:19.006Z
tags: css, vuetify, javascript, personnalisation, snow, neige
editor: markdown
dateCreated: 2025-12-13T16:24:19.006Z
---

# ##1. IntroductionCette page documente la méthode pour ajouter un effet de flocons de neige discret et performant à l'ensemble du site **wiki.blablalinux.be**.

L'objectif est d'utiliser la fonctionnalité d'injection de code intégrée à l'administration de Wiki.js pour insérer du CSS et du JavaScript, garantissant que l'effet ne sera **jamais écrasé** lors des mises à jour du système.

---

##2. Code CSS : le style et l'animationLe CSS est responsable de la forme des flocons, de leur positionnement fixe, et surtout, de l'animation de chute via les *keyframes*.

###2.1. Emplacement de l'injectionLe code CSS doit être placé dans la fenêtre dédiée au remplacement de la feuille de style.

> 📍 **Chemin précis :** `Administration` -> `Theme` -> `Injection de code` -> **Remplacement de CSS**

###2.2. Code à ajouterCe code doit être ajouté **à la suite** de votre code CSS existant :

```css
/* --- Code des Flocons de Neige (À ajouter dans "Remplacement de CSS") --- */

#snow-container {
    position: fixed; 
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    pointer-events: none; 
    z-index: 9999; 
    overflow: hidden;
}

.flake {
    position: absolute;
    background: #fff; 
    border-radius: 50%;
    opacity: 0.8;
    animation-name: fall;
    animation-timing-function: linear;
    animation-iteration-count: infinite;
}

/* Définition de l'animation de chute */
@keyframes fall {
    0% {
        transform: translateY(-10vh);
    }
    100% {
        transform: translateY(100vh);
    }
}

```

---

##3. Code JavaScript : la création et la logiqueLe JavaScript crée dynamiquement les éléments HTML et leur attribue des propriétés aléatoires (taille, position, vitesse) pour un effet plus naturel.

###3.1. Emplacement de l'injectionLe JavaScript doit être placé à la fin du corps de la page (`<body>`), ce qui est la meilleure pratique pour les performances.

> 📍 **Chemin précis :** `Administration` -> `Theme` -> `Injection de code` -> **Injection HTML dans le body**

###3.2. Code à ajouterCollez ce bloc complet (incluant la balise `<script>`) dans cet emplacement :

```html
<script>
document.addEventListener('DOMContentLoaded', () => {
    // Création du conteneur qui couvre l'écran
    const snowContainer = document.createElement('div');
    snowContainer.id = 'snow-container';
    document.body.appendChild(snowContainer); 

    const numberOfFlakes = 60; // DENSITÉ: 60 flocons par défaut.

    for (let i = 0; i < numberOfFlakes; i++) {
        const flake = document.createElement('div');
        flake.classList.add('flake');
        
        // Position horizontale aléatoire
        flake.style.left = `${Math.random() * 100}%`; 
        
        // Taille aléatoire (entre 3px et 6px)
        const size = Math.random() * 3 + 3; 
        flake.style.width = `${size}px`;
        flake.style.height = `${size}px`;
        
        // Durée (Vitesse) et Délai (Départ) aléatoires
        flake.style.animationDuration = `${Math.random() * 8 + 12}s`; 
        flake.style.animationDelay = `${Math.random() * 8}s`; 

        snowContainer.appendChild(flake);
    }
});
</script>

```

---

##4. Personnalisation des paramètresL'un des avantages de cette méthode est que vous pouvez facilement ajuster l'effet en modifiant quelques variables dans les codes injectés.

###4.1. Densité et mouvement (JavaScript)Les variables JavaScript définissent la quantité et la vitesse de la neige.

| Paramètre | Ligne de code | Description et ajustement |
| --- | --- | --- |
| **Densité** | `const numberOfFlakes = 60;` | C'est le nombre total de flocons affichés à l'écran. **Augmenter** la valeur rend la neige plus dense. |
| **Taille** | `const size = Math.random() * 3 + 3;` | Modifie la plage de taille (ici, entre 3px et 6px). <br>

<br>– Pour des flocons plus petits : essayez `Math.random() * 2 + 2;` (entre 2px et 4px). |
| **Vitesse (Lenteur)** | `animationDuration = ${Math.random() * 8 + 12}s` | Définit le temps de chute (en secondes). **Augmenter** les nombres rend la chute **plus lente**. |

###4.2. Couleur et Opacité (CSS)Ces paramètres sont gérés par les règles CSS appliquées à la classe `.flake`.

| Paramètre | Bloc de code CSS | Description et ajustement |
| --- | --- | --- |
| **Couleur** | `.flake { background: #fff; }` | Modifiez le code hexadécimal. Exemple : `#ADD8E6` pour une teinte bleu clair. |
| **Opacité (Transparence)** | `.flake { opacity: 0.8; }` | **Diminuer** la valeur (ex: `0.5`) rend les flocons plus transparents. |
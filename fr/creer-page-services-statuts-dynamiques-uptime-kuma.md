---
title: Créer une page de services avec statuts dynamiques Uptime Kuma
description: >-
  Apprenez à intégrer des badges de statut dynamiques d'Uptime Kuma dans un
  tableau de bord. Un guide pas à pas pour afficher la disponibilité de vos
  services auto-hébergés en temps réel.
published: true
date: '2026-11-11T22:39:53.372Z'
tags:
  - wordpress
  - tutoriel
  - uptime kuma
  - dashboard
  - badges
  - auto hébergement
editor: markdown
dateCreated: '2026-02-14T12:20:52.350Z'
---

Ce guide explique comment intégrer des badges de statut dynamiques provenant d'une instance **Uptime Kuma** dans un tableau **WordPress** ou une page **HTML**, afin de créer un tableau de bord de services public.

## Prérequis

* Une instance **Uptime Kuma** fonctionnelle.
* Des moniteurs déjà configurés pour vos services.
* Un accès à l'éditeur de votre site (WordPress ou autre).

---

## Étape 1 : récupérer les badges dans Uptime Kuma

1. Connectez-vous à votre tableau de bord **Uptime Kuma**.
2. Repérez le service que vous souhaitez afficher.

> ![uk-badges.png](/creer-page-services-statuts-dynamiques-uptime-kuma/uk-badges.png)
> *(Cette capture montre la liste des sondes et l'icône de réglage à cliquer)*

3. Cliquez sur l'icône de réglage (la roue crantée) du moniteur concerné.
4. Dans la fenêtre qui s'ouvre, cliquez sur le bouton vert **« Ouvre le générateur de lien badge »**.

> ![uk-badges-02.png](/creer-page-services-statuts-dynamiques-uptime-kuma/uk-badges-02.png)
> *(Cette capture montre l'emplacement du bouton vert dans les réglages de la sonde)*

## Étape 2 : personnaliser le style du badge

Une fois le générateur ouvert, vous pouvez configurer l'apparence de votre badge :

1. **Type de badge** : choisissez "status".
2. **Style de badge** : sélectionnez `flat`, `flat-square` (recommandé pour les tableaux), `plastic`, etc.
3. **Couleurs** : vous pouvez personnaliser les codes hexadécimaux pour correspondre à votre charte graphique.
4. **URL du badge** : copiez l'URL générée en bas de la fenêtre.

> ![uk-badges-03.png](/creer-page-services-statuts-dynamiques-uptime-kuma/uk-badges-03.png)
> *(Cette capture illustre l'interface complète de personnalisation des couleurs et du style)*

## Étape 3 : insérer le badge dans votre tableau

Dans la colonne « État » de votre tableau (WordPress ou HTML), insérez l'image en utilisant l'URL récupérée.

* **En HTML pur :**

```html
<img src="https://votre-kuma.be/api/badge/ID/status?style=flat-square" alt="Statut">

```

* **Dans WordPress :** utilisez le mode « Modifier en HTML » sur la cellule du tableau pour coller la balise `<img>`.

## Étape 4 : optimiser pour le mobile (responsive)

Pour éviter que le tableau ne soit écrasé sur smartphone, ajoutez ce CSS dans les réglages de votre thème :

```css
.wp-block-table {
    overflow-x: auto !important;
    display: block;
    width: 100%;
}
.wp-block-table table {
    min-width: 650px; /* Force une largeur lisible */
}

```

### Exemple concret

Vous pouvez voir une mise en œuvre réelle de ce tutoriel sur ma page dédiée :
👉 **[Voir le dashboard dynamique de Blabla Linux](https://blablalinux.be/mes-services-publics/)**

### Sources et documentation officielle

* Pour plus d'options de personnalisation (uptime, ping, temps de réponse), consultez la **[documentation officielle d'Uptime Kuma sur les badges](https://github.com/louislam/uptime-kuma/wiki/Badge)**.

### Astuce de Blabla Linux 🐧

> Pensez à utiliser les paramètres `upLabel` et `downLabel` dans l'URL si vous souhaitez traduire les textes « Up » et « Down » en français !

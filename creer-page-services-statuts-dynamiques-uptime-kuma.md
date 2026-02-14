---
title: Créer une page de services avec statuts dynamiques Uptime Kuma
description: Apprenez à intégrer des badges de statut dynamiques d'Uptime Kuma dans un tableau de bord. Un guide pas à pas pour afficher la disponibilité de vos services auto-hébergés en temps réel.
published: false
date: 2026-02-14T12:20:52.350Z
tags: wordpress, auto-hébergement, tutoriel, uptime kuma, dashboard, badges
editor: markdown
dateCreated: 2026-02-14T12:20:52.350Z
---

Ce guide explique comment intégrer des badges de statut dynamiques provenant d'une instance **Uptime Kuma** dans un tableau **WordPress** ou une page **HTML**, afin de créer un tableau de bord de services public.

## Prérequis

* Une instance **Uptime Kuma** fonctionnelle (version 1.16.0 ou supérieure).
* Des moniteurs déjà configurés pour vos services.
* Un accès à l'éditeur de votre site (WordPress ou autre).

---

## Étape 1 : récupérer les badges dans Uptime Kuma

1. Connectez-vous à votre tableau de bord **Uptime Kuma**.
2. Accédez à l'un de vos services. Les badges sont générés localement pour tous les moniteurs publiés sur une page de statut.
3. Vous pouvez récupérer l'URL du badge via le générateur intégré :
* Allez dans l'édition d'une **Page de statut**.
* Cliquez sur l'icône de réglages, puis sur le bouton **« Open Badge Maker »**.
* Personnalisez votre badge (style, labels, couleurs) et copiez l'URL générée.



## Étape 2 : choisir le style du badge

Uptime Kuma propose plusieurs styles basés sur *shields.io* :

* `flat` (par défaut)
* `flat-square` (recommandé pour l'intégration en tableau)
* `plastic`, `for-the-badge` ou `social`

*L'URL type ressemble à ceci : `https://kuma.votre-domaine.be/api/badge/:monitorID/status?style=flat-square`.*

## Étape 3 : insérer le badge dans votre tableau

Dans la colonne « État » de votre tableau (WordPress ou HTML) :

* **En HTML pur :**
```html
<img src="https://votre-kuma.be/api/badge/ID/status?style=flat-square" alt="Statut">

```


* **Dans WordPress :** Utilisez le mode « Modifier en HTML » sur la cellule du tableau pour coller la balise `<img>`.

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

---

## Exemple concret

Vous pouvez voir un exemple de mise en œuvre réelle de ce tutoriel sur ma page dédiée :
👉 **[Voir le dashboard dynamique de Blabla Linux](https://blablalinux.be/mes-services-publics/)**

### Sources et documentation officielle

* Pour plus d'options de personnalisation (uptime, ping, temps de réponse), consultez la **[documentation officielle d'Uptime Kuma sur les badges](https://github.com/louislam/uptime-kuma/wiki/Badge)**.

### Astuce de Blabla Linux 🐧

> Pensez à utiliser les paramètres `upLabel` et `downLabel` dans l'URL si vous souhaitez traduire les textes « Up » et « Down » en français !
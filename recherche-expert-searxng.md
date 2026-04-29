---
title: Maîtriser la recherche sur le Web
description: Devenez un expert de la recherche avec mon guide SearXNG. Opérateurs universels, raccourcis "bangs" avec exemples, requêtes spéciales et astuces pour optimiser la vitesse de vos résultats.
published: true
date: 2026-04-29T17:30:56.805Z
tags: performance, logiciel libre, searxng, vie privé, astuces, guide
editor: markdown
dateCreated: 2026-04-29T17:26:59.453Z
---

Savoir bien chercher, c'est savoir parler au moteur de recherche. Ce guide vous explique comment utiliser des commandes précises pour trouver exactement ce que vous voulez, sans perdre de temps.

> **Note :** La plupart de ces astuces sont **universelles**. Elles fonctionnent sur presque tous les moteurs. Cependant, pour une recherche sans pistage et sans publicité, **je vous recommande d'utiliser mon instance auto-hébergée**.

## 🔍 Mon instance de recherche privée
Pour appliquer ce guide et découvrir ma philosophie sur le respect de la vie privée, rendez-vous sur :
👉 **[https://searxng.blablalinux.be](https://searxng.blablalinux.be)** | **[À propos de ce service](https://searxng.blablalinux.be/info/fr/about)**

![searxng.blablalinux.be.png](/recherche-expert-searxng/searxng.blablalinux.be.png)

---

## 1. Les Opérateurs de précision (Universels)
Ces commandes de base vous permettent de filtrer les résultats directement depuis la barre de recherche.

* **"Expression exacte"** : Pour chercher un groupe de mots dans cet ordre précis.
    * *Exemple :* `"installation debian 12 sans interface graphique"`
* **Exclusion (-)** : Pour supprimer les résultats qui contiennent un mot inutile.
    * *Exemple :* `recette dessert -sucre` (pour des recettes sans sucre).
* **Recherche sur un site spécifique (site:)** : Pour fouiller uniquement un domaine.
    * *Exemple :* `proxmox site:blablalinux.be` (cherche uniquement sur mon blog).
* **Type de fichier (filetype: ou ext:)** : Pour trouver des documents précis.
    * *Exemple :* `guide survie linux filetype:pdf`
* **Le joker (*)** : Pour remplacer un mot inconnu dans une phrase.
    * *Exemple :* `"la peur n'évite pas le *"`
* **Tranche de nombres (..)** : Très utile pour les prix ou les dates.
    * *Exemple :* `ordinateur reconditionné 200..400 eur`

---

## 2. Les "Bangs" : Les raccourcis de puissance
Sur mon instance, le symbole `!` (bang) vous permet de rediriger votre recherche vers un service précis (voir la **[syntaxe complète](https://searxng.blablalinux.be/info/fr/search-syntax)**).

* **!gh** : Chercher sur GitHub. -> *Exemple : `!gh docker-compose`*
* **!debian** : Chercher dans la documentation Debian. -> *Exemple : `!debian kernel`*
* **!wp** : Chercher directement sur Wikipédia. -> *Exemple : `!wp logiciel libre`*
* **!yt** : Chercher une vidéo sur YouTube. -> *Exemple : `!yt blablalinux`*
* **!osm** : Chercher un lieu sur OpenStreetMap. -> *Exemple : `!osm Ciney`*
* **!images** : Basculer sur la recherche d'images. -> *Exemple : `!images serveur rack`*

---

## 3. Personnalisation et Performance
Vous pouvez configurer votre expérience sur mesure dans les **[Paramètres](https://searxng.blablalinux.be/preferences)**.

* **Choix des moteurs :** Dans l'onglet **"Moteurs"**, vous choisissez vos sources (Google, Bing, Qwant, etc.). 
* **Vitesse de réponse :** Attention, plus vous activez de moteurs simultanément, plus le temps de réponse sera long. Je vous conseille de ne garder que l'essentiel.
* **Synchronisation :** Vos réglages sont enregistrés via un cookie technique anonyme. Pour retrouver vos préférences sur vos autres appareils (smartphone, tablette) :
    1. Réglez vos préférences sur votre PC.
    2. En bas de la page des préférences, copiez le **"Lien de configuration"**.
    3. Ouvrez ce lien sur vos autres appareils pour tout synchroniser.

---

## 4. Requêtes Spéciales (Réponses Immédiates)
Mon instance intègre des modules intelligents que vous pouvez activer ou désactiver dans l'onglet **"Requêtes Spéciales"** (et certains dans l'onglet **"Général"**). Ils affichent la réponse directement en haut de page.

* **Calculatrice :** Pas besoin de site tiers.
    * *Exemple :* `(150 * 1.21)`
* **Conversions d'unités :** Très complet pour les monnaies ou mesures.
    * *Exemple :* `100 usd to eur` ou `50 kg in lbs`
* **Météo :** Un bulletin immédiat sans pub.
    * *Exemple :* `weather ciney`
* **Outils techniques :** Pour les admins et curieux.
    * *Exemple :* `pwgen` (génère un mot de passe) ou `ip` (affiche votre IP publique).
* **Open Access (DOI) :** Pour les chercheurs, permet de trouver des versions ouvertes de publications scientifiques (via l'onglet **"Général"**).

---

### Pourquoi j'héberge cette solution ?
En utilisant **[searxng.blablalinux.be](https://searxng.blablalinux.be)**, vous profitez d'un outil puissant qui agrège les meilleurs résultats du web. Mais contrairement aux géants du secteur, **je ne collecte aucune de vos données.** Pas d'historique, pas de profil publicitaire, juste une recherche libre et efficace.

---

> **Auteur** : ce guide est proposé par **Amaury aka BlablaLinux**. Retrouvez l'ensemble de mes services sur [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).

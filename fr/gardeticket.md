---
title: GardeTicket - Garde tes tickets et garanties sur ton téléphone
description: >-
  GardeTicket : application libre et privée pour garder tes tickets de caisse et
  suivre tes garanties. Photo ou PDF, alertes avant expiration, fonctionne hors
  ligne. Tout reste sur ton téléphone. Code sur GitHub et Gitea, déploiement
  Debian + nginx.
published: true
date: '2026-11-24T22:43:00.046Z'
tags:
  - gardeticket
  - pwa
  - garantie
  - tickets de caisse
  - vie privée
  - logiciel libre
  - auto hébergement
  - nginx
  - debian
  - debian 13
  - nginx proxy manager
editor: markdown
dateCreated: '2026-11-24T22:43:00.046Z'
---

**GardeTicket** est une application web (PWA) pour garder tes tickets de caisse et suivre tes garanties. Tout est stocké **uniquement dans la mémoire de ton téléphone** : aucune donnée n'est envoyée sur un serveur.

👉 **Utiliser l'application : [gardeticket.blablalinux.be](https://gardeticket.blablalinux.be)** (à ouvrir depuis un smartphone ou une tablette)

## Ce que fait GardeTicket

- 📷 Photographier un ticket (recadrage automatique, filtre « scanner » noir et blanc pour les tickets pâles) ou importer une facture PDF
- 📅 Calculer la fin de garantie (1, 2, 3, 5 ans ou à vie) et afficher des repères de couleur : vert, orange (moins de 3 mois), gris (expirée)
- 🔔 Afficher un bandeau d'alerte avant l'expiration et créer un rappel dans ton agenda, 30 jours avant la fin
- 🔍 Retrouver un achat par objet, magasin ou numéro de série
- 📤 Partager le ticket ou la facture avec un service après-vente
- 💾 Sauvegarder et restaurer tes tickets dans un fichier
- 🎨 Choisir parmi 8 thèmes de couleurs, avec mode clair ou sombre automatique
- 📴 Fonctionner même sans connexion internet

## Installer l'application sur ton téléphone

GardeTicket s'ouvre dans le navigateur, mais tu peux l'installer comme une vraie application :

- **Android** (Chrome, Vivaldi, Brave…) : menu (les trois points) → *Ajouter à l'écran d'accueil* ou *Installer l'application*
- **iPhone** (Safari) : bouton *Partager* → *Sur l'écran d'accueil*

> ℹ️ L'application est conçue pour les smartphones et tablettes : l'accès depuis un ordinateur de bureau est volontairement bloqué.

## Vie privée et sauvegardes

Tes tickets restent dans le navigateur de ton téléphone. Si tu vides les données du navigateur, si tu le désinstalles ou si tu changes de téléphone, **ils seront effacés**. Pense à utiliser régulièrement le bouton **💾 Sauvegarder** et garde le fichier obtenu dans un endroit sûr. Le bouton **📂 Restaurer** permet de tout récupérer, sans doublons.

## Héberger GardeTicket chez toi

L'application est un simple site statique (HTML, CSS, JavaScript), sans base de données ni dépendance. Le dépôt contient un script qui installe nginx et déploie les fichiers sur Debian 13 :

```bash
git clone https://github.com/anyblabla/gardeticket.git
cd gardeticket
sudo bash deploy/install.sh
```

- Le script propose un mode « derrière un reverse proxy » (par défaut) et un mode autonome avec HTTPS Let's Encrypt (`--standalone --domain … --email …`).
- **HTTPS est obligatoire** pour installer l'application sur le téléphone et utiliser le mode hors ligne.
- Pour mettre à jour : `git pull && sudo bash deploy/install.sh --update`

Tous les détails sont dans le fichier `README.md` du dépôt.

## Code source

Le code est libre et disponible sur deux dépôts identiques :

- 🐙 **GitHub** : [github.com/anyblabla/gardeticket](https://github.com/anyblabla/gardeticket)
- 🍵 **Gitea** (mon instance) : [gitea.blablalinux.be/blablalinux/gardeticket](https://gitea.blablalinux.be/blablalinux/gardeticket)

Les suggestions et signalements de problèmes sont les bienvenus sur l'un ou l'autre.

## Licence

GardeTicket est un logiciel libre distribué sous licence **GNU AGPL v3**.

---

Auteur : ce guide est proposé par Amaury alias BlablaLinux. Retrouvez tous mes services sur [Mes services publics](https://blablalinux.be/mes-services-publics/).

---
title: Activer l'authentification GitHub sur Wiki.js v3
description: >-
  Guide complet étape par étape pour configurer la connexion OAuth via GitHub
  sur Wiki.js version 3 derrière un proxy inverse Nginx Proxy Manager (NPM),
  avec contrôle des inscriptions.
published: true
date: '2026-11-12T22:30:36.742Z'
tags:
  - wikijs
  - github
  - oauth
  - authentification
  - nginx proxy manager
  - auto hébergement
editor: markdown
dateCreated: '2026-11-12T22:30:36.742Z'
---

Ajouter la connexion via **GitHub** sur une instance **Wiki.js v3** permet aux visiteurs de s'identifier sans créer un nouveau mot de passe, tout en servant de filtre naturel contre les robots spammeurs. Il est également possible de préparer l'intégration tout en gardant les inscriptions fermées.

---

## Prérequis

* Une instance **Wiki.js version 3** fonctionnelle en HTTPS (par exemple derrière **Nginx Proxy Manager**).
* Un compte **GitHub** pour créer une application OAuth.
* Un accès administrateur à votre Wiki.

---

## Étape 1 : Vérifier les en-têtes du proxy inverse

Si votre Wiki est placé derrière un proxy inverse (Nginx Proxy Manager, Traefik, Caddy), Wiki.js doit faire confiance aux en-têtes transmis pour générer des URLs de rappel en `https://` et non en `http://`.

1. Connectez-vous à l'administration de Wiki.js (`/_admin`).
2. Dans le menu de gauche, allez dans **Système → Sécurité**.
3. Activez l'option **Trust X-Forwarded-* Proxy Headers**.
4. Cliquez sur **APPLIQUER** en haut à droite.
5. Redémarrez le conteneur ou le service Wiki.js pour appliquer ce paramètre.

> **Note pour Nginx Proxy Manager :** Assurez-vous que votre hôte proxy transmet bien les en-têtes `X-Forwarded-Proto`, `X-Forwarded-Host` et `X-Forwarded-For`, et que votre éventuelle directive `Content-Security-Policy` (CSP) autorise `https://github.com` et `https://api.github.com` dans `connect-src`.

---

## Étape 2 : Créer une OAuth App sur GitHub

Pour relier Wiki.js à GitHub, il faut générer une paire **Client ID** et **Client Secret**.

1. Ouvrez la page de création d'application OAuth sur GitHub : [https://github.com/settings/applications/new](https://github.com/settings/applications/new).
2. Remplissez le formulaire avec vos informations :
   * **Application name :** Le nom de votre Wiki (ex. `Mon Wiki`).
   * **Homepage URL :** L'adresse complète de votre Wiki (ex. `https://wiki.exemple.be`).
   * **Application description :** Une courte description (optionnel).
   * **Redirect URI :** Indiquez une adresse provisoire (ex. `https://wiki.exemple.be/callback`), le temps que Wiki.js génère l'URL exacte.
3. Laissez **Allow wildcard matching** et **Enable Device Flow** décochés. Vous pouvez décocher **Expire user access tokens** pour garder un comportement standard avec vos autres applications autohébergées.
4. Cliquez sur **Register application**.
5. Sur la page suivante, copiez le **Client ID** affiché en haut, puis cliquez sur **Generate a new client secret** et copiez immédiatement le **Client Secret** généré.

---

## Étape 3 : Configurer la stratégie dans Wiki.js

1. Retournez dans l'administration de votre Wiki : **Utilisateurs → Authentification**.
2. Cliquez sur **AJOUTER UNE STRATÉGIE** et sélectionnez **GitHub**.
3. Renseignez les champs suivants :
   * **Nom :** `GitHub` (nom affiché sur le bouton de connexion).
   * **Client ID :** Collez l'identifiant récupéré sur GitHub.
   * **Client Secret :** Collez la clé secrète générée sur GitHub.
   * Laissez **GitHub Enterprise Host** et **Restrict to Organization** vides (sauf usage spécifique en entreprise ou organisation).
4. Réglez les deux interrupteurs selon l'ouverture souhaitée :
   * **Activée :** Active la stratégie techniquement.
   * **Authentification :** **Désactivez (rouge)** si vous souhaitez bloquer la création automatique de nouveaux comptes, ou **activez (vert)** si vous ouvrez le Wiki aux contributions.
5. Cliquez sur le bouton vert **APPLIQUER** en haut à droite.
6. Faites défiler tout en bas de la page dans la section **Référence de configuration** et copiez l'URL générée dans le champ **Authorization Callback URL**. Elle doit impérativement commencer par `https://`.

---

## Étape 4 : Mettre à jour l'URL de rappel sur GitHub

1. Revenez sur la fiche de votre application dans vos paramètres développeur GitHub ([https://github.com/settings/developers](https://github.com/settings/developers)).
2. Remplacez l'URL provisoire dans le champ **Redirect URI** par l'**Authorization Callback URL** exacte copiée depuis Wiki.js.
3. Cliquez sur **Update application** pour enregistrer.

---

## Étape 5 : Afficher ou masquer le bouton sur la page de connexion

Dans Wiki.js v3, l'activation technique d'une stratégie est séparée de son affichage sur l'écran de connexion public.

1. Dans la colonne de gauche de l'administration, sous le nom de votre site, cliquez sur **Authentification** (icône clé).
2. Activez ou désactivez l'interrupteur **GitHub** selon que vous souhaitez rendre le bouton visible ou le garder en réserve sur la page `/login`.
3. Cliquez sur **APPLIQUER**.

---

## Vérification

1. Ouvrez une fenêtre de navigation privée et rendez-vous sur l'adresse de votre Wiki (`/login`).
2. Cliquez sur le bouton **GitHub** et validez l'autorisation sur GitHub :
   * Si l'auto-inscription est **activée** (ou si votre compte admin utilise déjà l'adresse e-mail principale de votre compte GitHub), vous êtes connecté automatiquement.
   * Si l'auto-inscription est **désactivée**, Wiki.js bloque la création de profil et affiche `ERR_REGISTRATION_DISABLED`, ce qui confirme que la liaison OAuth fonctionne tout en maintenant l'accès fermé.

---

> **Auteur** : ce guide est proposé par **Amaury aka BlablaLinux**. Retrouvez tous mes services sur [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).

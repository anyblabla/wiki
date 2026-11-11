---
title: Firefox Sync auto-hébergé
description: >-
  Synchronisez vos favoris, mots de passe, onglets et historique Firefox sur un
  serveur auto-hébergé libre plutôt que chez Mozilla. Explications, réglages
  pour Firefox existant ou neuf, avantages.
published: true
date: '2026-11-11T22:21:45.780Z'
tags:
  - firefox
  - sync
  - vie-privee
  - docker
  - postgresql
  - virustotal
  - auto-hebergement
  - logiciel libre
editor: markdown
dateCreated: '2026-11-08T20:07:04.032Z'
---

Firefox sait synchroniser vos favoris, vos mots de passe, vos onglets et votre historique entre tous vos appareils. Par défaut, ces données transitent par les serveurs de Mozilla. Ici, elles sont stockées sur **mon propre serveur**, hébergé et sauvegardé par mes soins.

> [!NOTE] Adresse du service
> `https://syncstorage-rs.blablalinux.be`

---

## C'est quoi, Firefox Sync ?

Firefox Sync est la fonction de Firefox qui garde vos appareils à jour entre eux. Vous ajoutez un favori sur votre ordinateur, vous le retrouvez sur votre téléphone. Vous enregistrez un mot de passe d'un côté, il est disponible de l'autre.

Elle synchronise :

- les favoris
- l'historique
- les onglets ouverts
- les mots de passe enregistrés
- les extensions et les préférences
- les adresses et moyens de paiement enregistrés dans Firefox

Pour fonctionner, Sync a besoin de deux choses :

1. **Un compte Mozilla**, qui identifie l'utilisateur (adresse e-mail et mot de passe).
2. **Un serveur de stockage**, qui conserve les données synchronisées.

Le point important : ces deux briques sont séparées, et on peut remplacer la seconde.

## À quoi sert ce service ?

Le serveur que je fournis remplace **le serveur de stockage** de Mozilla. Concrètement :

- **Votre compte Mozilla reste chez Mozilla.** Votre e-mail et votre mot de passe sont toujours vérifiés par leurs serveurs. Vous ne créez pas de nouveau compte chez moi.
- **Vos données de synchronisation vont chez moi.** Favoris, historique, onglets, mots de passe et préférences sont enregistrés dans une base de données sur mon serveur, et non plus chez Mozilla.

Ces données sont **chiffrées par Firefox avant l'envoi**, avec une clé dérivée de votre compte. Ni Mozilla ni moi ne voyons le contenu : le serveur ne manipule que des données chiffrées. L'intérêt n'est donc pas de me faire confiance, mais de décider **où** vos données sont stockées, qui les sauvegarde et qui gère l'infrastructure.

En résumé : au lieu de passer par le serveur de Mozilla, Firefox passe par le mien.

## Comment ça marche ?

Il suffit de dire à Firefox d'utiliser une autre adresse pour le stockage. Cela se fait avec un seul réglage dans `about:config` :

| Réglage | Valeur |
|---|---|
| `identity.sync.tokenserver.uri` | `https://syncstorage-rs.blablalinux.be/1.0/sync/1.5` |

Ensuite, vous vous connectez avec votre compte Mozilla habituel, comme d'habitude.

> [!WARNING] Un réglage unique par navigateur
> Il faut l'appliquer sur chaque appareil que vous souhaitez synchroniser, et **avant** de vous connecter à Sync.

---

## Sur une nouvelle installation de Firefox

C'est le cas le plus simple.

1. Installez Firefox, **sans vous connecter** à un compte.
2. Tapez `about:config` dans la barre d'adresse et validez. Acceptez l'avertissement.
3. Recherchez `identity.sync.tokenserver.uri`.
4. Si la préférence n'existe pas, créez-la : choisissez le type **Chaîne** (String), puis cliquez sur le bouton « + ».
5. Saisissez la valeur :

   ```
   https://syncstorage-rs.blablalinux.be/1.0/sync/1.5
   ```

6. Redémarrez Firefox.
7. Connectez-vous avec votre compte Mozilla et activez la synchronisation.
8. Vérifiez que tout fonctionne : ouvrez `about:sync-log` pour consulter les journaux de synchronisation.

## Sur un Firefox déjà synchronisé chez Mozilla

Un Firefox déjà relié à Sync ne bascule pas tout seul vers un autre serveur. Il faut le déconnecter, changer le réglage, puis le reconnecter.

> [!WARNING] Avant de commencer
> Exportez vos favoris (menu *Favoris* → *Gérer les favoris* → *Importer et sauvegarde*), et gardez un appareil complet sous la main pendant la transition.

1. Dans les paramètres de Firefox, section **Synchronisation**, déconnectez-vous de Sync.
2. Ouvrez `about:config` et réglez `identity.sync.tokenserver.uri` sur :

   ```
   https://syncstorage-rs.blablalinux.be/1.0/sync/1.5
   ```

3. Redémarrez Firefox.
4. Reconnectez-vous avec votre compte Mozilla et activez la synchronisation. Firefox devrait alors renvoyer vos données locales vers le nouveau serveur.
5. Contrôlez le résultat dans `about:sync-log`, puis vérifiez que vos favoris, mots de passe et onglets sont bien présents.
6. Répétez l'opération sur chaque appareil.

Ce changement n'efface pas les données déjà présentes sur les serveurs de Mozilla. Si vous voulez les supprimer, il faut passer par les paramètres de votre compte Mozilla.

## Sur Firefox pour Android

La configuration mobile est moins directe, car `about:config` n'est pas disponible de la même manière.

1. Ouvrez **Paramètres** → **À propos de Firefox**.
2. Appuyez plusieurs fois sur le logo Firefox jusqu'à l'activation du **menu de débogage**.
3. Revenez dans les paramètres : une option permettant de définir un **serveur de synchronisation personnalisé** apparaît. Saisissez l'adresse du service.
4. Redémarrez l'application, puis connectez-vous à votre compte.

> [!NOTE] iPhone et iPad
> Je n'ai pas détaillé la procédure pour iOS ici. Si vous êtes concerné, contactez-moi.

---

## Les avantages de cette solution auto-hébergée

- **Vos données restent sous contrôle.** Elles sont stockées sur mon infrastructure, en Belgique, et non dans un service tiers dont on ne maîtrise rien.
- **Logiciel libre de bout en bout.** Le serveur est le projet open source `syncstorage-rs`, développé par Mozilla, publié sur GitHub. Tout le code est lisible et auditable.
- **Un chiffrement qui ne dépend pas de moi.** Le chiffrement se fait dans votre navigateur : l'hébergeur ne peut pas lire le contenu de vos synchronisations.
- **Des sauvegardes régulières.** La base est sauvegardée de mon côté, ce qui n'est pas toujours le cas avec un compte gratuit chez un grand éditeur.
- **Indépendance.** Vous réduisez votre dépendance à une seule entreprise pour le stockage de vos données de navigation.
- **Un service à taille humaine.** Pas de publicité, pas d'exploitation commerciale de vos données, et quelqu'un à qui parler en cas de problème.
- **Une démarche cohérente avec l'esprit du libre.** Réutiliser, héberger soi-même et partager les connaissances, comme dans le reste de ma démarche autour du reconditionnement et du logiciel libre.

## Limites et points d'attention

Pour être honnête, cette solution a aussi ses contraintes :

- **Il faut toujours un compte Mozilla.** L'authentification passe par eux ; seul le stockage est déplacé.
- **Le service est hébergé par une seule personne.** Je fais de mon mieux pour qu'il soit disponible, mais il n'a pas la redondance d'un grand service commercial.
- **Les places sont limitées.** Le serveur n'est pas ouvert à tout le monde : le nombre d'utilisateurs est plafonné et les inscriptions peuvent être fermées une fois les places prises. Contactez-moi via les réseaux BlablaLinux pour demander l'accès.
- **La configuration demande une manipulation.** Le réglage dans `about:config` est simple, mais moins direct qu'un bouton dans l'interface, surtout sur mobile.

> [!TIP] Installer votre propre serveur
> Ce service vous intéresse et vous voulez l'héberger vous-même ? J'ai détaillé toute ma mise en place, de Docker Compose et PostgreSQL à la configuration de Nginx Proxy Manager, en passant par la gestion des inscriptions et les sauvegardes : [Installer un serveur Firefox Sync auto-hébergé](/fr/installer-firefox-sync-auto-heberge).

## Pour les curieux : sous le capot

Le service repose sur :

- **syncstorage-rs**, le serveur de synchronisation officiel de Mozilla, écrit en Rust ;
- **PostgreSQL**, pour stocker les données ;
- **Docker Compose**, dans un conteneur LXC Debian ;
- **Nginx Proxy Manager**, en reverse proxy avec un certificat HTTPS.

Documentation officielle : [Run your own Sync-1.5 server with Docker](https://mozilla-services.github.io/syncstorage-rs/how-to/how-to-run-with-docker.html)

Code source : [github.com/mozilla-services/syncstorage-rs](https://github.com/mozilla-services/syncstorage-rs)

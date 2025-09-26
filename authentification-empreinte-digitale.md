---
title: Authentification par Empreinte Digitale
description: Installez fprintd et libpam-fprintd. Enregistrez votre doigt (CLI ou GUI). Activez la fonction système avec pam-auth-update. Important : Utilisez le mot de passe pour la connexion initiale afin de déverrouiller le trousseau de clés.
published: true
date: 2025-09-26T18:28:24.997Z
tags: debian, authentification, empreinte, fingerprint
editor: markdown
dateCreated: 2025-09-26T18:28:24.997Z
---

# Gestion de la Sécurité : Authentification par Empreinte Digitale

L'authentification par empreinte digitale est disponible dans Debian grâce au projet [fprint](https://www.freedesktop.org/wiki/Software/fprint/).
*(Voir la [liste des périphériques supportés](https://fprint.freedesktop.org/)).*

## 1\. Installation

Installez les paquets `fprintd` (pour la gestion des empreintes) et `libpam-fprintd` (pour l'activation de la connexion par empreinte digitale) :

```bash
# apt install fprintd libpam-fprintd
```

## 2\. Configuration (Enrôlement)

Les empreintes peuvent être ajoutées via la ligne de commande (`fprintd-enroll`) ou par les interfaces graphiques de divers environnements de bureau, tels que l'interface des **Paramètres de GNOME** ou les **Paramètres Système de KDE** (sous *Système / Utilisateurs / Configurer l'authentification par empreinte digitale*).

### Enrôlement via la ligne de commande

Si `fprintd-enroll` est exécuté (par l'utilisateur souhaitant enregistrer son empreinte) sans aucun argument, il demandera le mot de passe de l'utilisateur actuel. Après une authentification réussie, il commencera l'enregistrement de l'**index droit**.

Lorsque vous voyez des lignes similaires aux suivantes dans la sortie du terminal :

```
Using device /net/reactivated/Fprint/Device/0
Enrolling right-index-finger finger.
```

Commencez à toucher (ou à glisser, selon le type de votre appareil) le capteur avec votre index droit. Pour chaque touche enregistrée correctement, le programme réagira avec une ligne comme la suivante :

```
Enroll result: enroll-stage-passed
```

Continuez à toucher le capteur, en plaçant votre doigt sous un angle légèrement différent à chaque fois, jusqu'à ce que le programme signale `Enroll result: enroll-completed` et quitte.

À ce stade, vous devriez pouvoir vous connecter à votre gestionnaire d'affichage (par exemple, GDM, LightDM ou SDDM) en utilisant votre index droit.

  * GDM, par exemple, affichera `(ou faites glisser votre doigt)` sous le champ du mot de passe lors de la demande du mot de passe utilisateur. SDDM affichera un message similaire.
  * La connexion normale par mot de passe restera toujours disponible.

### Commandes utiles

Vous pouvez également vérifier que votre empreinte a été enregistrée correctement en exécutant :

```bash
$ fprintd-verify
```

D'autres commandes utiles sont `fprintd-list` et `fprintd-delete`. Consultez `man fprintd.1` pour plus d'informations.

## 3\. Activation de l'Authentification

Pour activer l'authentification par empreinte digitale à l'**échelle du système**, exécutez :

```bash
# pam-auth-update
```

... et activez le profil **"Fingerprint authentication"** en cochant la case correspondante, puis en appuyant sur **"OK"**.

Ou exécutez simplement :

```bash
# pam-auth-update --enable fprintd
```

Ceci activera l'authentification par empreinte digitale dans diverses installations du système, telles que `sudo` et les environnements de bureau / gestionnaires d'affichage (par exemple, KDE / SDDM).

Généralement, l'authentification par mot de passe restera possible. Par exemple, `sudo` demandera initialement l'authentification par empreinte digitale. Si l'utilisateur ne s'authentifie pas avec une empreinte, `sudo` demandera l'authentification par mot de passe après un délai d'attente.

## 4\. Mises en Garde (Caveats)

Il y a quelques mises en garde ou particularités à garder à l'esprit lors de l'utilisation de l'authentification par empreinte digitale.

  * **Interférence du processus `fprintd` lors de l'enrôlement**
    Au moins à la date de cette modification, sous Bookworm, le processus `fprintd` en arrière-plan, qui appartient à `root`, interfère parfois avec la capacité d'un utilisateur non-root à enregistrer des empreintes. Si cela se produit, **tuez simplement ce processus** et, si vous utilisez l'interface de paramètres Gnome pour enregistrer des empreintes, **fermez et rouvrez simplement la fenêtre d'enrôlement** après que le processus `fprintd` ait été tué. Vous devriez alors être en mesure d'enregistrer correctement vos empreintes.

  * **Déverrouillage du trousseau de clés (Keyring/Wallet)**
    Si vous utilisez votre empreinte digitale pour votre connexion initiale, vous devrez toujours saisir votre mot de passe pour déverrouiller le trousseau de clés/portefeuille de votre environnement de bureau, à moins que vous n'ayez choisi de ne pas le chiffrer avec votre mot de passe lors de sa création.
    Tout comme un téléphone intelligent, ces services utilisent ce mot de passe pour chiffrer leur contenu localement sur votre disque.
    Il semble donc que la **meilleure pratique** pour utiliser l'authentification par empreinte digitale soit la suivante :

    1.  Vous connecter initialement avec votre **mot de passe réel** pour vous assurer que tous les services sont déverrouillés ou démarrés correctement.
    2.  Utiliser ensuite votre **empreinte digitale** pour déverrouiller l'écran, exécuter des commandes avec `sudo`, etc.

-----

### Source

  * **Titre :** SecurityManagement/fingerprint authentication
  * **Lien :** [https://wiki.debian.org/SecurityManagement/fingerprint%20authentication](https://wiki.debian.org/SecurityManagement/fingerprint%20authentication)
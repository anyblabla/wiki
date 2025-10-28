---
title: Authentification par Empreinte Digitale
description: Installez fprintd et libpam-fprintd. Enregistrez votre doigt (CLI ou GUI). Activez la fonction système avec pam-auth-update. Important : Utilisez le mot de passe pour la connexion initiale afin de déverrouiller le trousseau de clés.
published: true
date: 2025-10-28T11:54:03.359Z
tags: debian, authentification, empreinte, fingerprint, fprintd
editor: markdown
dateCreated: 2025-09-26T18:28:24.997Z
---

L'authentification biométrique par empreinte digitale est prise en charge par Debian grâce au projet **`fprint`**. Ce projet fournit les outils nécessaires à la gestion des capteurs et à l'intégration au système de connexion (PAM).

*(Veuillez consulter la [liste des périphériques supportés](https://fprint.freedesktop.org/)).*

-----

## 1\. Installation des paquets

Installez les deux paquets nécessaires :

  * **`fprintd`** : Le service de démon (daemon) gérant le capteur et les empreintes enregistrées.
  * **`libpam-fprintd`** : La librairie PAM (Pluggable Authentication Modules) qui active l'authentification par empreinte digitale pour les sessions système.

<!-- end list -->

```bash
# apt install fprintd libpam-fprintd
```

-----

## 2\. Configuration et enrôlement

L'enregistrement des empreintes (ou "enrôlement") peut se faire via l'interface graphique de votre environnement de bureau (ex. : **Paramètres de GNOME** ou **Paramètres Système de KDE** sous *Système / Utilisateurs / Configurer l'authentification par empreinte digitale*), ou directement par la ligne de commande.

### Enrôlement via la ligne de commande

Cette méthode est essentielle si vous n'utilisez pas d'environnement de bureau. Exécutez la commande suivante en tant que l'utilisateur souhaitant enregistrer son empreinte :

```bash
$ fprintd-enroll
```

1.  La commande demandera d'abord le mot de passe de l'utilisateur pour l'authentification initiale.

2.  Après validation, elle commence l'enregistrement de l'**index droit** par défaut :

    ```
    Using device /net/reactivated/Fprint/Device/0
    Enrolling right-index-finger finger.
    ```

3.  **Action :** Touchez (ou glissez) le capteur d'empreinte digitale. Pour chaque contact correct, le programme répond :

    ```
    Enroll result: enroll-stage-passed
    ```

4.  **Finalisation :** Continuez à toucher le capteur sous des angles légèrement différents jusqu'à ce que le programme signale : `Enroll result: enroll-completed` et se termine.

À ce stade, l'authentification par empreinte est active pour votre gestionnaire d'affichage (GDM, LightDM, SDDM, etc.). Votre gestionnaire de connexion affichera un message similaire à **`(ou faites glisser votre doigt)`** près du champ du mot de passe.

> **Note :** La connexion normale par mot de passe reste toujours disponible.

### Commandes utiles

Pour vérifier que votre empreinte a été enregistrée correctement, exécutez :

```bash
$ fprintd-verify
```

D'autres commandes pour la gestion sont :

  * **`fprintd-list`** : Liste les empreintes enregistrées.
  * **`fprintd-delete`** : Supprime une empreinte spécifique.

Consultez `man fprintd.1` pour plus d'informations.

-----

## 3\. Activation de l'authentification à l'échelle du système (PAM)

Pour activer l'authentification par empreinte digitale pour toutes les tâches du système nécessitant une identification (comme `sudo`, `su`, ou l'écran de verrouillage), vous devez mettre à jour la configuration PAM.

Exécutez la commande suivante en tant que `root` :

```bash
# pam-auth-update
```

Dans l'interface qui apparaît, activez le profil **"Fingerprint authentication"** en cochant la case correspondante, puis en appuyant sur **"OK"**.

Ou, utilisez la version directe de la commande :

```bash
# pam-auth-update --enable fprintd
```

Ceci intègre l'authentification par empreinte digitale dans la séquence PAM. En pratique :

  * `sudo` demandera initialement l'authentification par empreinte digitale.
  * Si l'utilisateur échoue l'authentification biométrique ou la laisse expirer, `sudo` reviendra automatiquement à la demande du mot de passe traditionnel après un court délai d'attente.

-----

## 4\. Mises en garde et meilleures pratiques

Il y a quelques particularités à considérer lors de l'utilisation de cette méthode d'authentification.

### Interférence du processus `fprintd` lors de l'enrôlement

Au moins dans certaines versions de Debian (comme Bookworm), le processus `fprintd` en arrière-plan (qui appartient à `root`) peut parfois interférer avec la capacité d'un utilisateur non-root à enregistrer de nouvelles empreintes.

Si vous rencontrez ce problème :

1.  **Tuez simplement ce processus** de démon.
2.  Si vous utilisiez une interface de paramètres graphique, **fermez et rouvrez la fenêtre d'enrôlement** après l'arrêt du processus.

Vous devriez alors être en mesure d'enregistrer correctement vos empreintes.

### Déverrouillage du trousseau de clés (Keyring/Wallet)

Si vous utilisez votre empreinte digitale pour votre **connexion initiale** à la session graphique :

  * Le système d'exploitation sera déverrouillé.
  * Cependant, le **trousseau de clés** (Keyring de GNOME ou KWallet de KDE), qui contient les mots de passe de vos applications, ne sera **pas déverrouillé automatiquement**.

Le trousseau de clés est chiffré par votre mot de passe réel pour des raisons de sécurité.

#### Meilleure pratique recommandée :

1.  **Connexion initiale :** Utilisez votre **mot de passe réel** pour vous connecter. Cela garantit que tous les services chiffrés (comme le trousseau) sont déverrouillés ou démarrés correctement.
2.  **Utilisation courante :** Utilisez ensuite votre **empreinte digitale** pour déverrouiller l'écran, exécuter des commandes avec `sudo`, et toutes les autres tâches d'authentification rapides.

-----

### Source

  * **Titre :** SecurityManagement/fingerprint authentication
  * **Lien :** [https://wiki.debian.org/SecurityManagement/fingerprint%20authentication](https://wiki.debian.org/SecurityManagement/fingerprint%20authentication)
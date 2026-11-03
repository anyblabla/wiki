---
title: >-
  Réinitialisation du Mot de Passe Root ou Utilisateur via GRUB (Debian /
  Emmabuntüs)
description: >-
  Ce guide décrit la méthode pour réinitialiser un mot de passe perdu sur un
  système Debian (ou Emmabuntüs) en éditant le menu GRUB au démarrage.
published: true
date: '2026-11-03T00:00:58.907Z'
tags:
  - password
  - user
  - root
  - emmabuntus
editor: markdown
dateCreated: '2024-08-15T15:31:02.884Z'
---

a> **Méthode testée et opérationnelle sur Emmabuntüs DE4/5, et donc valable pour Debian.**

-----

## 1\. Accéder au Menu GRUB

| Distribution | Action |
| :--- | :--- |
| **Emmabuntüs** | Le menu GRUB apparaît automatiquement pendant cinq secondes au démarrage. |
| **Debian** | Si le menu n'apparaît pas, mettez la machine sous tension tout en maintenant la touche **`Shift`** enfoncée. |

-----

## 2\. Éditer les Paramètres de Démarrage

Une fois dans le menu GRUB, vous allez forcer le système à démarrer directement sur un shell Bash avec accès en écriture.

1.  **Sélectionner la ligne :** Placez-vous sur la **première ligne** du menu GRUB (votre entrée de démarrage principale).
2.  **Passer en mode édition :** Appuyez sur la touche **`e`**.
3.  **Modifier la ligne `linux` :** Déplacez-vous jusqu'à la ligne qui commence par **`linux`** (ou `linux /boot/...`).
4.  **Ajouter la commande de réinitialisation :** À la fin de cette ligne, ajoutez l'argument suivant, séparé par un espace :
    ```plaintext
    rw init=/bin/bash
    ```
5.  **Démarrer :** Appuyez sur la touche **`F10`** pour continuer le démarrage du système. Vous arriverez sur un prompt (`shell`).

-----

## 3\. Vérifier l'Accès en Écriture

Pour vous assurer que le système de fichiers racine est monté en mode lecture/écriture (`rw`), exécutez la commande suivante :

```bash
mount | grep -w /
```

Si la commande retourne une ligne contenant **`(rw,realtime)`** (ou simplement `rw`), l'accès en écriture est correctement établi et vous pouvez procéder à la réinitialisation.

-----

## 4\. Réinitialiser le Mot de Passe

Utilisez la commande `passwd` pour changer le mot de passe du compte désiré :

### Pour le compte **root**

```bash
passwd
```

Entrez et confirmez le nouveau mot de passe pour le compte `root`.

### Pour un compte **utilisateur**

```bash
passwd <votre-nom-utilisateur>
```

Remplacez `<votre-nom-utilisateur>` par le nom d'utilisateur du compte à modifier. Entrez et confirmez le nouveau mot de passe.

-----

## 5\. Redémarrer le Système

Une fois le mot de passe réinitialisé, redémarrez le système proprement en appelant la séquence d'initialisation normale :

```bash
exec /sbin/init
```

Le système va redémarrer et vous pourrez vous connecter avec le nouveau mot de passe que vous avez défini. ✔️

> **Source :** Méthode disponible sur le site officiel Emmabuntüs : [https://yourls.blablalinux.be/emmade-reset-password](https://yourls.blablalinux.be/emmade-reset-password) 👍

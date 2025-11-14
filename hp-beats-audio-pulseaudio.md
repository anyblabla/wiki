---
title: Guide - Activer le Son Complet (Caisson de Basses) HP Beats Audio sous Ubuntu/Debian
description: HP Beats Audio sur Linux : Son incomplet sur Ubuntu/Debian? Souvent causé par un mauvais réglage des haut-parleurs et du caisson de basses (LFE). Utilisez l'outil hdajackretask pour corriger et reconfigurer la sortie audio.
published: true
date: 2025-11-14T19:04:11.130Z
tags: debian, hp, beatsaudio, pulseaudio, sound, ubuntu
editor: markdown
dateCreated: 2025-09-26T18:10:25.732Z
---

Si le son de votre PC portable HP Beats Audio (par exemple, HP Pavilion DV7) est incomplet sous Ubuntu/Debian, cela est souvent dû à une mauvaise configuration des haut-parleurs internes et du caisson de basses (**LFE**). Ce guide utilise l'outil **`hdajackretask`** pour reconfigurer la sortie audio.

-----

### Étapes de Configuration avec `hdajackretask`

#### Étape 1 : Installer l'Outil de Reconfiguration

L'outil **`hdajackretask`** fait partie du paquet **`alsa-tools-gui`**.

Ouvrez le terminal et exécutez les commandes suivantes :

1.  **Mettre à jour le système :**
    ```bash
    sudo apt update && sudo apt upgrade
    ```
2.  **Installer l'utilitaire :**
    ```bash
    sudo apt install alsa-tools-gui
    ```

#### Étape 2 : Lancer et Configurer

1.  Recherchez et lancez le programme **`hdajackretask`**.
2.  Dans le menu déroulant **"Select a codec"**, choisissez votre codec, souvent **`IDT 92HD91BXX`** (le nom peut varier).
3.  Cochez la case **"Show unconnected pins"** pour afficher toutes les broches de configuration.

#### Étape 3 : Reconfigurer les Broches Audio

Vous allez maintenant remapper les sorties physiques vers les bonnes fonctions audio.

| Broche (Pin) | Option à sélectionner dans "Remap to" | Fonction |
| :--- | :--- | :--- |
| **`0x0d`** | **"Internal speaker"** | Haut-parleur interne (face avant) |
| **`0x0f`** | **"Internal speaker"** | Haut-parleur interne (souvent sous l'écran) |
| **`0x10`** | **"Internal speaker (LFE)"** | **Caisson de basses** (Low-Frequency Effects) |

#### Étape 4 : Appliquer et Rendre les Changements Permanents

1.  Cliquez sur le bouton **"Apply now"**.
2.  **Testez immédiatement** le son avec votre lecteur audio : le son complet doit sortir, y compris les basses.
3.  Si le test est concluant, cliquez sur **"Install boot override"** pour que ces réglages soient chargés à chaque démarrage.

#### Étape 5 : Finalisation

**Redémarrez** votre ordinateur. Le son complet (y compris le caisson de basses) devrait être fonctionnel.

> **Vérification** : Assurez-vous que l'audio des haut-parleurs internes se coupe automatiquement lorsque vous branchez votre casque/vos écouteurs.
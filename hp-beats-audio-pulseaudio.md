---
title: HP Beats Audio PulseAudio
description: Beats Audio est incomplet sous Ubuntu/Debian. C'est un problème fréquent que j'ai personnellement rencontré avec mon HP Pavilion DV7-6085eb. Après de nombreuses recherches sur les forums, j'ai trouvé une solution.
published: true
date: 2025-09-26T18:29:20.116Z
tags: debian, hp, beatsaudio, pulseaudio, sound, ubuntu
editor: markdown
dateCreated: 2025-09-26T18:10:25.732Z
---

## Guide : Activer le son complet des haut-parleurs HP Beats Audio sous Ubuntu/Debian (PulseAudio)

Si vous avez un ordinateur portable HP avec des haut-parleurs Beats Audio, vous avez peut-être remarqué que le son est incomplet sous Ubuntu/Debian. C'est un problème fréquent que j'ai personnellement rencontré avec mon HP Pavilion DV7-6085eb. Après de nombreuses recherches sur les forums, j'ai trouvé une solution qui permet d'activer tous les haut-parleurs, y compris le caisson de basses, sans compromettre la fonctionnalité des écouteurs.

Ce guide utilise l'outil `hda-jack-retask` pour reconfigurer correctement la sortie audio.

### Comment procéder

**Étape 1 : Installer `hda-jack-retask`**

Ce dernier fait partie d'un pack d'utilitaires qui se nomme `alsa-tools-gui`.

Ouvrez votre terminal pour installer ce paquet :

`sudo apt update && sudo apt upgrade`

`sudo apt install alsa-tools-gui`

**Étape 2 : Lancer l'outil**

Recherchez et ouvrez le programme **hda-jack-retask**.

**Étape 3 : Sélectionner le codec**

Dans le menu déroulant "Select a codec", choisissez **IDT 92HD91BXX**. *Note : Le nom du codec peut varier selon votre modèle d'ordinateur.*

**Étape 4 : Activer l'affichage des broches non connectées**

Cochez la case **"Show unconnected pins"**. Cette étape est cruciale, car les haut-parleurs internes ne sont pas toujours détectés comme connectés par défaut.

**Étape 5 : Reconfigurer les haut-parleurs**

Vous allez maintenant remapper les broches de votre carte son. Pour cela, cliquez sur chaque ligne listée ci-dessous et sélectionnez la bonne option dans le menu déroulant "Remap to":
* Pour la broche **`0x0d`** (haut-parleur interne, face avant), sélectionnez **"Internal speaker"**.
* Pour la broche **`0x0f`** (souvent "unconnected" mais correspondant aux haut-parleurs sous l'écran), sélectionnez **"Internal speaker"**.
* Pour la broche **`0x10`** (souvent "unconnected" mais correspondant au caisson de basses), sélectionnez **"Internal speaker (LFE)"**.

**Étape 6 : Appliquer et tester les changements**

Cliquez sur le bouton **"Apply now"**. Testez immédiatement avec votre lecteur audio préféré pour vérifier que le son sort de tous les haut-parleurs, y compris le caisson de basses.

**Étape 7 : Rendre les changements permanents**

Si le test est concluant, cliquez sur **"Install boot override"**. Cela sauvegardera les paramètres pour qu'ils soient appliqués automatiquement à chaque démarrage.

**Étape 8 : Redémarrer**

Redémarrez votre ordinateur. Le son complet devrait maintenant fonctionner. N'oubliez pas de tester aussi la prise casque : l'audio des haut-parleurs internes doit se couper automatiquement lorsque vous branchez des écouteurs, comme c'est le cas avec un système fonctionnel.
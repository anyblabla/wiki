---
title: Emmabuntüs - Mot de passe perdu
description: La machine n'a plus été démarrée depuis un moment ! Vous avez oublié le mot de passe root et/ou utilisateur ? Voici une solution.
published: true
date: 2025-07-16T23:34:30.233Z
tags: password, user, root, emmabuntus
editor: markdown
dateCreated: 2024-08-15T15:31:02.884Z
---

**Méthode testée par mes soins et opérationnelle sur Emmabuntüs DE4/5, et comme Emmabuntüs est une Debian, méthode valable pour Debian 💯**

# Accéder au menu du Grub

Au démarrage, sur Emmabuntüs, pas besoins d’appeler le menu Grub. Il apparaît automatiquement durant cinq secondes.

Pour Debian, si le menu Grub n'apparaît pas au démarrage, tout en mettant la machine sous tension, laisser votre doigt appuyé sur la touche Shift.

# Éditer le menu Grub

-   Tout en étant placé sur la _première ligne_ du menu Grub, on appuie sur la touche « **_e_** » pour passer en mode édition.

On se déplace avec les touches fléchées !

-   On repère la ligne qui commence par « **linux** », et à la fin de celle-ci, on ajoute…

`rw init=/bin/bash`

-   On termine avec la touche « **F10** » pour continuer le démarrage du système et arriver sur un prompt.

# Tester l’accès au shell

-   On rentre cette commande…

`mount | grep -w /`

Si cette dernière retourne « **(rw,realtime)** » tout est ok.

# Réinitialiser le mot de passe

-   Pour réinitialiser le mot de passe du compte « **roo**t », on utilise cette commande…

 `passwd`

On entre un mot de passe, on le confirme ensuite.

-   Pour réinitialiser le mot de passe d’un compte utilisateur, on utilise cette commande…

`passwd <votre-nom-utilisateur>`

On entre un mot de passe, on le confirme ensuite.

# Redémarrer le système

-   On utilise cette commande…

`exec /sbin/init`

On arrive sur le système avec notre mot de passe ✔️

☝️ Méthode disponible sur le site officiel Emmabuntüs : [https://yourls.blablalinux.be/emmade-reset-password](https://yourls.blablalinux.be/emmade-reset-password) 👍
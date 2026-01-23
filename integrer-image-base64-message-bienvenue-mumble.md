---
title: Intégrer une image au message de bienvenue Murmur
description: Apprenez à intégrer une image au message de bienvenue de Murmur via le Base64. Méthode appliquée sur le serveur BlablaLinux pour un affichage performant sans dépendre d'un hébergement externe.
published: true
date: 2026-01-23T23:02:17.546Z
tags: debian, administration, mumble, murmur, base64, image, png
editor: markdown
dateCreated: 2026-01-23T23:02:17.545Z
---

L'utilisation d'images intégrées permet de personnaliser l'accueil des utilisateurs sans dépendre de liens externes qui pourraient devenir obsolètes. Voici la méthode complète, **comme cela a été fait sur le serveur BlablaLinux**.

## Optimisation du poids de l'image

Avant toute manipulation, il est crucial de redimensionner votre image. Une image trop volumineuse génère une chaîne Base64 immense qui peut ralentir le client Mumble ou dépasser la limite autorisée par le serveur.

**Exemple concret :**
Sur le serveur **BlablaLinux**, l'image d'origine de **1000 px** de large a été redimensionnée à **300 px** avant la conversion. Cela garantit un code compact et une intégration fluide.

## Génération du code Base64 sous Linux

Vous pouvez générer votre code directement depuis votre terminal Debian ou Ubuntu sans utiliser d'outils tiers :

```bash
base64 -w 0 mon_image.png > base64.txt

```

* **`-w 0`** : Force la sortie sur une seule ligne (indispensable pour le fichier de configuration).
* **`> base64.txt`** : Enregistre le résultat dans un fichier pour faciliter le copier-coller.

## Configuration du serveur Murmur

Dans votre fichier de configuration (par exemple `ini-personnel.ini`), vous devez insérer le code dans la variable `welcometext`.

### Syntaxe et échappement

Comme la valeur de `welcometext` est encadrée par des guillemets, vous devez impérativement ajouter un antislash (`\`) devant les guillemets internes de la balise image.

**Exemple de configuration :**

```ini
welcometext="<br />Bienvenue sur le serveur Murmur.<br />Ce service est hébergé sur un cluster <b>Proxmox VE</b>.<br /><br /><img src=\"data:image/png;base64,VOTRE_CODE_BASE64_ICI\" width=\"300\" />"

```

## Points de vigilance

* **Limite de taille** : Vérifiez que la variable `imagemessagelength` dans votre configuration (ex: `131072`) est supérieure au nombre de caractères de votre code Base64.
* **Format** : Si vous utilisez un JPEG, remplacez `image/png` par `image/jpeg` dans la balise.

![mumble-picture-welcome.png](/integrer-image-base64-message-bienvenue-mumble/mumble-picture-welcome.png)
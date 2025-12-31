---
title: Configuration de Plymouth sur Debian
description: Guide complet pour installer et configurer un splash screen (écran de démarrage) sur Debian avec Plymouth. Apprenez à changer de thème, prévisualiser et résoudre les erreurs d'alternatives.
published: true
date: 2025-12-31T01:25:56.130Z
tags: debian, personnalisation, administration système, plymouth, boot
editor: markdown
dateCreated: 2025-12-31T01:25:56.130Z
---

**Plymouth** est l'utilitaire qui permet d'afficher un écran de démarrage graphique (splash screen) lors du démarrage, masquant ainsi les messages textuels du noyau Linux.

---

## 1. Installation des paquets

Pour installer Plymouth et les thèmes officiels de Debian, exécutez la commande suivante :

```bash
sudo apt update
sudo apt install plymouth plymouth-themes plymouth-x11

```

## 2. Configuration du chargeur d'amorçage Grub

Pour que l'affichage graphique s'active dès le début du démarrage, modifiez le fichier de configuration de **Grub** :

1. Éditez le fichier : `sudo nano /etc/default/grub`
2. Modifiez la ligne suivante pour inclure `quiet splash` :
```bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"

```


3. (Optionnel) Pour un rendu net sur les écrans modernes, forcez la résolution :
```bash
GRUB_GFXMODE=1920x1080

```


4. Mettez à jour la configuration : `sudo update-grub`

## 3. Gestion des thèmes

### Sélection via l'outil Plymouth

Pour lister et appliquer un thème (par exemple : `emerald`) :

```bash
plymouth-set-default-theme -l          # Lister les thèmes
sudo plymouth-set-default-theme -R emerald  # Appliquer et régénérer l'initramfs

```

### Sélection via les alternatives Debian

Si le thème n'est pas reconnu ou pour respecter la hiérarchie **Debian** :

1. **Enregistrer le thème** (si absent de la liste) :
```bash
sudo update-alternatives --install /usr/share/plymouth/themes/default.plymouth default.plymouth /usr/share/plymouth/themes/emerald/emerald.plymouth 100

```


2. **Choisir le thème** :
```bash
sudo update-alternatives --config default.plymouth

```


3. **Appliquer les changements** :
```bash
sudo update-initramfs -u

```



## 4. Prévisualisation en direct

Pour tester le rendu visuel sans redémarrer votre machine :

```bash
sudo plymouthd ; sudo plymouth --show-splash ; sleep 5 ; sudo plymouth quit

```

*Note : Si le thème ne change pas, forcez la fermeture du service avec `sudo pkill plymouthd` avant de relancer le test.*

## 5. Thèmes recommandés

* **Emerald** : Thème par défaut de **Debian** 12.
* **Homeworld** : Thème par défaut de **Debian** 11.
* **Joy** : Thème classique bleu.
* **Spinner** : Utilise le logo du constructeur (idéal pour le reconditionnement).
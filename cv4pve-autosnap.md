---
title: Automatiser les instantanés (Snapshots) Proxmox VE avec CV4PVE-AUTOSNAP
description: Détails de l'installation et de la configuration de l'outil CV4PVE-AUTOSNAP (Corsinvest) dans un LXC pour automatiser les instantanés (Snapshots) de vos VM et CT sur Proxmox VE.
published: true
date: 2026-03-26T00:11:01.082Z
tags: 
editor: markdown
dateCreated: 2024-12-28T18:49:39.139Z
---

## 💡 Contexte et choix de l'outil

Pour la prise d'instantanés des invités sur [Proxmox VE](https://www.proxmox.com/en/proxmox-virtual-environment/overview), plusieurs méthodes existent :

* L'interface web ([GUI](https://w.wiki/CZ6F)) de Proxmox.
* La ligne de commande ([CLI](https://w.wiki/6gSf)) avec l'outil natif [PVESH](https://pve.proxmox.com/pve-docs/chapter-pvesh.html).
* Des outils tiers, tels que [CV4PVE-ADMIN](https://corsinvest.it/cv4pve-admin-proxmox/) (GUI multiclusters) qui utilise, en coulisses, **CV4PVE-AUTOSNAP**.

> **Note importante (mars 2026) :** Ce guide présente la configuration de base avec des scripts individuels. Pour une gestion optimisée sur cluster multi-nœuds, consultez la nouvelle version : [**Automatisation avancée des snapshots Proxmox (multi-nœuds)**](https://wiki.blablalinux.be/fr/automatisation-snapshots-proxmox-multi-noeuds).

> J'utilise CV4PVE-ADMIN, qui est un gestionnaire multiclusters. Mais je n'utilise que la fonction de prise automatique d'instantanés. C'est sortir la grosse artillerie pour pas grand-chose.
>
> Je vais donc vous montrer comment utiliser CV4PVE-AUTOSNAP. Non pas sur l'hôte Proxmox lui-même, mais en l'isolant sur un conteneur ([LXC](https://w.wiki/CXkQ)).
>
> Personnellement, j'ai déployé un conteneur [Debian 12 Bookworm](https://www.debian.org/releases/bookworm/).

-----

## I. Installation et préparation dans le conteneur LXC

Toutes les commandes suivantes doivent être exécutées à l'intérieur de votre conteneur LXC (Debian/Ubuntu) dédié.

### 1\. Téléchargement du binaire

Récupérez la dernière version de CV4PVE-AUTOSNAP disponible à [cette adresse](https://github.com/Corsinvest/cv4pve-autosnap/releases). *(Version 1.15.0 au moment où j'écris ces lignes)* :

```plaintext
wget https://github.com/Corsinvest/cv4pve-autosnap/releases/download/v1.15.0/cv4pve-autosnap-linux-x64.zip
```

### 2\. Décompression et installation

Installez l'utilitaire `unzip`, décompressez l'archive, et rendez le binaire exécutable.

```plaintext
apt install unzip
```

```plaintext
unzip cv4pve-autosnap-linux-x64.zip
```

```plaintext
chmod +x cv4pve-autosnap
```

### 3\. Finalisation

Déplacez le fichier binaire dans le répertoire `/bin/` pour pouvoir l'exécuter de n'importe où dans le système :

```plaintext
mv cv4pve-autosnap /bin/
```

Créez l'emplacement pour les fichiers log qui seront générés par les tâches Cron :

```plaintext
mkdir /root/autosnap-logs
```

### 4\. Vérification

Pour obtenir de l'aide sur les options disponibles :

```plaintext
cv4pve-autosnap --help
```

-----

## II. Création du script d'instantanés (Snapshots)

Nous allons maintenant créer un premier script pour la prise d'instantanés à des heures précises (9h et 21h dans cet exemple).

### 1\. Structure de la commande de base

La commande principale de `cv4pve-autosnap` utilise la structure suivante, où toutes les informations de connexion et de rétention sont fournies :

`cv4pve-autosnap --host=XXX.XXX.X.XXX --username=root@pam --password=XXXXXXXX --vmid=XXX snap --label=XXXXX --keep=X`

> **À partir d'ici, toutes les commandes sont à adapter !**

### 2\. Création du script `autosnap-921.sh`

Nous créons notre script, que l'on va nommer ici `autosnap-921.sh` :

```plaintext
nano /root/autosnap-921.sh 
```
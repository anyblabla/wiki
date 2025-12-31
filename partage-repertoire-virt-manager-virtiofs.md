---
title: Partage de répertoire hôte vers machine virtuelle avec virtio-fs
description: Guide technique pour configurer le partage de fichiers haute performance entre un hôte Linux et une VM KVM/QEMU via virtio-fs sous Virt-manager. Idéal pour Debian, Ubuntu et Linux Mint.
published: true
date: 2025-12-31T01:20:30.522Z
tags: share, debian, administration système, virtmanager, libvirt, virtiofs
editor: markdown
dateCreated: 2025-12-31T01:20:30.522Z
---

## Présentation

Le pilote **virtio-fs** est la solution de pointe pour partager des données entre un hôte et un invité. Il surpasse l'ancien protocole 9p en offrant une latence réduite et une gestion optimisée du cache disque.

## 1. Prérequis sur la machine hôte

Le moteur de rendu du système de fichiers doit être installé sur votre système physique (votre Debian ou Linux Mint habituelle).

```bash
# Installation du démon nécessaire
sudo apt update && sudo apt install virtiofsd

# Redémarrage du service de virtualisation
sudo systemctl restart libvirtd

```

## 2. Configuration dans l'interface Virt-manager

### Activation de la mémoire partagée

Cette étape est indispensable pour permettre au démon `virtiofsd` de communiquer avec la mémoire vive de la machine virtuelle.

1. Accédez aux détails de la machine virtuelle (icône ampoule).
2. Sélectionnez la section **Mémoire**.
3. Cochez l'option **Activer la mémoire partagée**.
4. Cliquez sur **Appliquer**.

### Ajout du matériel de système de fichiers

1. Cliquez sur le bouton **Ajouter un matériel**.
2. Choisissez **Système de fichiers** dans la colonne de gauche.
3. Configurez les champs suivants :
* **Pilote** : `virtiofs`
* **Chemin de la source** : Le répertoire sur votre hôte (ex: `/home/blablalinux/scripts`).
* **Chemin de la cible** : Un identifiant unique (ex: `ScriptsTests`).


4. Validez en cliquant sur **Terminer**.

## 3. Configuration dans la machine virtuelle

### Montage manuel du répertoire

Une fois la machine virtuelle démarrée, créez le point de montage et exécutez la commande de liaison.

```bash
# Création du dossier de destination
sudo mkdir -p /mnt/ScriptsTests

# Montage du partage via l'identifiant (tag)
sudo mount -t virtiofs ScriptsTests /mnt/ScriptsTests

```

### Automatisation au démarrage

Pour rendre le partage permanent, ajoutez la configuration suivante dans le fichier `/etc/fstab` de la machine virtuelle :

```text
ScriptsTests  /mnt/ScriptsTests  virtiofs  defaults  0  0

```

## 4. Résolution des problèmes courants

* **Erreur de binaire manquant** : Si la machine virtuelle refuse de démarrer, vérifiez que `virtiofsd` est bien installé sur l'hôte.
* **Module introuvable** : Si l'invité ne reconnaît pas le type `virtiofs`, assurez-vous que le noyau de la machine virtuelle est à jour (paquet `linux-image-amd64`).
* **Droits d'accès** : Le dossier sur l'hôte doit être accessible en lecture et écriture pour l'utilisateur système gérant QEMU (souvent `libvirt-qemu`).
---
title: Réduction de la taille d'un disque LXC sur ZFS
description: Apprenez à réduire la taille d'un disque LXC sur ZFS. Une manipulation simple en quelques commandes pour ajuster vos quotas et la configuration Proxmox sans risque de perte de données.
published: true
date: 2026-04-23T21:24:40.584Z
tags: lxc, proxmox, stockage, zfs
editor: markdown
dateCreated: 2026-04-23T21:24:40.584Z
---

Sur Proxmox, réduire la taille d'un disque pour un conteneur LXC stocké sur ZFS est assez rapide. Comme ZFS gère cela via des **quotas**, on ne touche pas directement aux partitions.

## 1. Avant de commencer
* **Sauvegarde :** Fais toujours un backup de ton conteneur avant de modifier le stockage.
* **Vérification :** Assure-toi que l'espace utilisé dans le LXC est bien inférieur à la nouvelle taille cible.

## 2. Procédure étape par étape

### Étape 1 : Arrêter le conteneur
Le conteneur doit être éteint pour appliquer les changements proprement.
```bash
pct stop [ID_DU_LXC]
```

### Étape 2 : Identifier le volume ZFS
Cherche le nom exact du dataset associé à ton conteneur.
```bash
zfs list | grep subvol-[ID_DU_LXC]-disk
```

### Étape 3 : Modifier le quota ZFS
On définit ici la nouvelle limite physique (ex: 10 Go).
```bash
zfs set refquota=10G [NOM_DU_POOL]/subvol-[ID_DU_LXC]-disk-0
```

### Étape 4 : Mettre à jour la configuration Proxmox
Il faut modifier manuellement le fichier de conf pour que l'interface affiche la bonne taille.

1.  Ouvre le fichier : `nano /etc/pve/lxc/[ID_DU_LXC].conf`
2.  Sur la ligne **rootfs:**, remplace l'ancienne taille par la nouvelle (ex: `size=10G`).
3.  Sauvegarde et quitte (`Ctrl+O` puis `Ctrl+X`).

### Étape 5 : Redémarrage et test
Relance le conteneur et vérifie si tout est correct.
```bash
pct start [ID_DU_LXC]
df -h
```

---

> **Auteur** : ce guide est proposé par **Amaury aka BlablaLinux**. Retrouvez l'ensemble de mes services sur [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).
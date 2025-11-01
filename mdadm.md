---
title: mdadm : RAID logiciel sous Linux
description: mdadm (Multiple Devices ADMinistration) est l'utilitaire standard sous Linux pour gérer les grappes RAID (Redundant Array of Independent Disks) logiciels.
published: true
date: 2025-11-01T00:52:15.931Z
tags: mdadm, raid
editor: markdown
dateCreated: 2025-11-01T00:26:33.056Z
---

Il permet de combiner plusieurs périphériques de stockage (disques durs, SSD, partitions) en un seul volume logique pour améliorer les performances, la redondance, ou les deux.

-----

## 1\. Concepts de base et préparation

### A. Niveaux RAID courants

| Niveau RAID | Description | Avantages | Inconvénients |
| :--- | :--- | :--- | :--- |
| **RAID 0 (Striping)** | Répartit les données pour la vitesse. | **Meilleures performances**. | **Aucune tolérance aux pannes**. |
| **RAID 1 (Miroir)** | Copie les données sur plusieurs disques. | **Redondance maximale** (tolérance à 1 panne). | Capacité réduite de moitié. |
| **RAID 5** | Striping avec parité distribuée. | Bon équilibre performance/redondance (tolérance à 1 panne). | |
| **RAID 6** | Striping avec double parité. | **Haute redondance** (tolérance à 2 pannes). | |
| **RAID 10 (1+0)** | Miroir puis Striping. | Excellentes performances et redondance. | Capacité réduite de moitié. |

### B. Préparation des disques

1.  **Installation de `mdadm` :**
    ```bash
    sudo apt update
    sudo apt install mdadm -y
    ```
2.  **Partitionnement :** Les partitions doivent être créées et marquées comme **Linux RAID auto** (code `fd` pour fdisk).
    ```bash
    # Exemple avec fdisk pour changer le type de partition /dev/sdb1
    sudo fdisk /dev/sdb
    ```

-----

## 2\. Création : RAID 0 et RAID 1

### A. RAID 0 (Performance sans redondance)

Le RAID 0 nécessite un minimum de **deux disques** et combine leur capacité pour une vitesse maximale.

```bash
# Commande de création (exemple pour /dev/md2 avec 2 disques)
sudo mdadm --create /dev/md2 \
    --level=0 \
    --raid-devices=2 \
    /dev/sdb2 /dev/sdc2
```

### B. RAID 1 (Redondance maximale)

Le RAID 1 nécessite un minimum de **deux disques** et assure que les données sont toujours dupliquées.

```bash
# Commande de création (exemple pour /dev/md0 avec 2 disques)
sudo mdadm --create /dev/md0 \
    --level=1 \
    --raid-devices=2 \
    /dev/sdb1 /dev/sdc1
```

-----

## 3\. Création : RAID 5 (Parité simple)

Le RAID 5 nécessite un **minimum de trois disques** et tolère la défaillance d'un seul disque.

### A. Commande de création RAID 5

```bash
# Exemple pour /dev/md3 avec 3 disques actifs
sudo mdadm --create /dev/md3 \
    --level=5 \
    --raid-devices=3 \
    /dev/sda1 /dev/sdb1 /dev/sdc1
```

### B. Ajouter un disque de réserve (*Hot Spare*)

Pour une protection accrue, ajoutez un disque de réserve (`--spare-devices=1`) :

```bash
sudo mdadm --create /dev/md0 \
    --level=5 \
    --raid-devices=3 \
    --spare-devices=1 \
    /dev/sdb1 /dev/sdc1 /dev/sdd1 /dev/sde1
```

-----

## 4\. Création : RAID 6 (Double parité)

Le RAID 6 nécessite un **minimum de quatre disques** et tolère la défaillance de **deux disques** simultanément.

### A. Commande de création RAID 6

```bash
# Exemple pour /dev/md4 avec 4 disques actifs
sudo mdadm --create /dev/md4 \
    --level=6 \
    --raid-devices=4 \
    /dev/sdc1 /dev/sdd1 /dev/sde1 /dev/sdf1
```

-----

## 5\. Création : RAID 10 (Hybride performance/redondance)

Le RAID 10 nécessite un **minimum de quatre disques** et est excellent pour les performances en lecture/écriture.

### A. Commande de création RAID 10

```bash
# Exemple pour /dev/md1 avec 4 disques
sudo mdadm --create /dev/md1 \
    --level=10 \
    --raid-devices=4 \
    /dev/sdb1 /dev/sdc1 /dev/sdd1 /dev/sde1
```

-----

## 6\. Gestion post-création

### A. Surveillance et synchronisation

Vérifiez toujours l'état de la grappe, en particulier la progression de la synchronisation (*resync* ou *rebuild*) pour les niveaux RAID 1, 5, 6 et 10.

```bash
cat /proc/mdstat
```

### B. Création du système de fichiers et montage

1.  **Formatage (exemple ext4) :**
    ```bash
    sudo mkfs.ext4 -F /dev/md0
    ```
2.  **Montage :**
    ```bash
    sudo mkdir /mnt/raid
    sudo mount /dev/md0 /mnt/raid
    ```

### C. Sauvegarde de la configuration

Sauvegardez la configuration pour assurer la reconnaissance de la grappe au démarrage :

```bash
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
```

-----

## 7\. Maintenance et simulation de défaillance

### A. Affichage des détails

Pour obtenir des informations détaillées :

```bash
sudo mdadm --detail /dev/md0
```

### B. Simulation de défaillance (RAID 5/6/10)

1.  **Marquer le disque comme défaillant :**

    ```bash
    sudo mdadm /dev/md0 --fail /dev/sdb1
    ```

2.  **Retirer et remplacer le disque (après reconstruction) :**

    ```bash
    # Retirer l'ancien disque défaillant
    sudo mdadm /dev/md0 --remove /dev/sdb1

    # Ajouter le nouveau disque de réserve
    sudo mdadm /dev/md0 --add /dev/sdb1
    ```


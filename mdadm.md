---
title: mdadm : RAID logiciel sous Linux
description: mdadm (Multiple Devices ADMinistration) est l'utilitaire standard sous Linux pour gérer les tableaux RAID (Redundant Array of Independent Disks) logiciels.
published: true
date: 2025-11-01T00:26:33.056Z
tags: mdadm, raid
editor: markdown
dateCreated: 2025-11-01T00:26:33.056Z
---

Il permet de combiner plusieurs périphériques de stockage (disques durs, SSD, partitions) en un seul volume logique pour améliorer les performances, la redondance, ou les deux.

-----

## 1\. Concepts de base du RAID logiciel

Le RAID logiciel est géré par le noyau Linux, ce qui le rend flexible et indépendant du matériel physique.

### A. Niveaux RAID courants

| Niveau RAID | Description | Avantages | Inconvénients |
| :--- | :--- | :--- | :--- |
| **RAID 0 (Striping)** | Regroupe les données sans redondance. | **Meilleures performances** (lecture/écriture). | **Aucune tolérance aux pannes** (perte de données si un disque échoue). |
| **RAID 1 (Miroir)** | Les données sont écrites sur deux disques simultanément (miroir). | **Redondance maximale** (tolérance à la panne d'un disque). | Capacité totale réduite de moitié. |
| **RAID 5** | Combine *striping* et parité distribuée (nécessite au moins 3 disques). | Bon équilibre entre performance et redondance (tolérance à la panne d'un disque). | Complexité accrue, performances en écriture légèrement réduites. |
| **RAID 6** | *Striping* avec double parité (nécessite au moins 4 disques). | **Très haute redondance** (tolérance à la panne de deux disques). | Capacité et performances en écriture inférieures à RAID 5. |
| **RAID 10 (1+0)** | Combinaison de RAID 1 et RAID 0 (nécessite au moins 4 disques). | Très bonnes performances et excellente redondance. | Coût élevé en disques (capacité réduite de moitié). |

-----

## 2\. Préparation des disques

Avant de créer un tableau RAID, les disques ou partitions doivent être préparés avec le type de partition Linux RAID.

### A. Partitionnement

Utilisez un outil de partitionnement (`fdisk`, `gdisk`, `parted`) pour créer les partitions qui composeront le tableau RAID.

Changez le type (ou *flag*) de la partition pour **Linux RAID auto** (code `fd` pour MBR/fdisk ou `Linux RAID` pour GPT/gdisk).

```bash
# Exemple avec fdisk pour changer le type de partition /dev/sdb1
sudo fdisk /dev/sdb
# ... (entrer 't' pour changer le type, puis 'fd' et 'w' pour écrire)
```

### B. Installation de `mdadm`

Installez l'utilitaire `mdadm` sur votre système :

```bash
sudo apt update
sudo apt install mdadm -y
```

-----

## 3\. Création du tableau RAID (exemple RAID 1)

L'exemple suivant crée un tableau RAID 1 nommé `md0` avec deux partitions : `/dev/sdb1` et `/dev/sdc1`.

### A. Commande de création

Utilisez l'option `--create` et spécifiez le niveau (`--level`), le nombre de dispositifs (`--raid-devices`) et les partitions :

```bash
sudo mdadm --create /dev/md0 \
    --level=1 \
    --raid-devices=2 \
    /dev/sdb1 /dev/sdc1
```

### B. Surveillance

La synchronisation du tableau (appelée *resync*) commence immédiatement. Vous pouvez surveiller sa progression :

```bash
cat /proc/mdstat
```

-----

## 4\. Gestion et utilisation

### A. Création d'un système de fichiers

Une fois que la synchronisation initiale est terminée (ou suffisamment avancée), vous pouvez formater le nouveau périphérique RAID (`/dev/md0`) :

```bash
# Exemple de formatage en ext4
sudo mkfs.ext4 -F /dev/md0
```

### B. Montage du tableau

Créez un point de montage et montez le volume RAID :

```bash
sudo mkdir /mnt/raid1
sudo mount /dev/md0 /mnt/raid1
```

### C. Sauvegarde de la configuration

Pour que le système reconnaisse le tableau RAID après un redémarrage, vous devez sauvegarder sa configuration dans le fichier de configuration de `mdadm` :

```bash
# Exporter la configuration actuelle et l'ajouter au fichier de conf
sudo mdadm --detail --scan | sudo tee -a /etc/mdadm/mdadm.conf
```

> Sur certains systèmes, vous devrez peut-être mettre à jour l'image `initramfs` après cela.

-----

## 5\. Maintenance et dépannage

### A. Affichage des détails

Pour obtenir des informations détaillées sur l'état et la structure d'un tableau spécifique :

```bash
sudo mdadm --detail /dev/md0
```

### B. Gestion des pannes (RAID 1)

En cas de défaillance d'un disque, le tableau passera en mode **dégradé**.

1.  **Marquer le disque comme défaillant :**

    ```bash
    sudo mdadm /dev/md0 --fail /dev/sdb1
    ```

2.  **Retirer le disque défaillant :**

    ```bash
    sudo mdadm /dev/md0 --remove /dev/sdb1
    ```

3.  **Ajouter un nouveau disque (ou partition) :**
    Une fois le disque physique remplacé et partitionné, ajoutez la nouvelle partition au tableau. La reconstruction (*rebuild*) commencera automatiquement.

    ```bash
    sudo mdadm /dev/md0 --add /dev/sdd1
    ```

La progression de la reconstruction peut être vérifiée via `cat /proc/mdstat`.
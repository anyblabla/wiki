---
title: Automatiser les instantanés (Snapshots) Proxmox VE avec CV4PVE-AUTOSNAP
description: Détails de l'installation et de la configuration de l'outil CV4PVE-AUTOSNAP (Corsinvest) dans un LXC pour automatiser les instantanés (Snapshots) de vos VM et CT sur Proxmox VE.
published: true
date: 2026-03-27T13:54:15.067Z
tags: 
editor: markdown
dateCreated: 2024-12-28T18:49:39.139Z
---

## 💡 Contexte et choix de l'outil

Pour la prise d'instantanés des invités sur [Proxmox VE](https://www.proxmox.com/en/proxmox-virtual-environment/overview), plusieurs méthodes existent :

* L'interface web ([GUI](https://w.wiki/CZ6F)) de Proxmox.
* La ligne de commande ([CLI](https://w.wiki/6gSf)) avec l'outil natif [PVESH](https://pve.proxmox.com/pve-docs/chapter-pvesh.html).
* Des outils tiers, tels que [CV4PVE-ADMIN](https://corsinvest.it/cv4pve-admin-proxmox/) (GUI multiclusters) qui utilise, en coulisses, **CV4PVE-AUTOSNAP**.

> **Note importante (mars 2026) :** Ce guide présente la configuration de base avec des scripts individuels. Pour une gestion optimisée sur cluster multi-nœuds, consultez la nouvelle version : [**Automatisation avancée des snapshots Proxmox (multi-nœuds)**](/fr/automatisation-snapshots-proxmox-multi-noeuds).

> J'utilise CV4PVE-ADMIN, qui est un gestionnaire multiclusters. Mais je n'utilise que la fonction de prise automatique d'instantanés. C'est sortir la grosse artillerie pour pas grand-chose.
>
> Je vais donc vous montrer comment utiliser CV4PVE-AUTOSNAP. Non pas sur l'hôte Proxmox lui-même, mais en l'isolant sur un conteneur ([LXC](https://w.wiki/CXkQ)).
>
> Personnellement, j'ai déployé un conteneur [Debian 12 Bookworm](https://www.debian.org/releases/bookworm/).

-----

## I. Installation et préparation dans le conteneur LXC

Toutes les commandes suivantes doivent être exécutées à l'intérieur de votre conteneur LXC (Debian/Ubuntu) dédié.

### 1. Téléchargement du binaire

Récupérez la dernière version de CV4PVE-AUTOSNAP disponible à [cette adresse](https://github.com/Corsinvest/cv4pve-autosnap/releases). *(Version 1.15.0 au moment où j'écris ces lignes)* :

```plaintext
wget https://github.com/Corsinvest/cv4pve-autosnap/releases/download/v1.15.0/cv4pve-autosnap-linux-x64.zip
```

### 2. Décompression et installation

Installez l'utilitaire `unzip`, décompressez l'archive, et rendez le binaire exécutable.

```plaintext
apt install unzip
unzip cv4pve-autosnap-linux-x64.zip
chmod +x cv4pve-autosnap
```

### 3. Finalisation

Déplacez le fichier binaire dans le répertoire `/bin/` pour pouvoir l'exécuter de n'importe où dans le système :

```plaintext
mv cv4pve-autosnap /bin/
mkdir /root/autosnap-logs
```

### 4. Vérification

Pour obtenir de l'aide sur les options disponibles :

```plaintext
cv4pve-autosnap --help
```

-----

## II. Création du script d'instantanés (Snapshots)

Nous allons maintenant créer un premier script pour la prise d'instantanés à des heures précises (9h et 21h dans cet exemple).

### 1. Structure de la commande de base

La commande principale de `cv4pve-autosnap` utilise la structure suivante :

`cv4pve-autosnap --host=XXX.XXX.X.XXX --username=root@pam --password=XXXXXXXX --vmid=XXX snap --label=XXXXX --keep=X`

> **À partir d'ici, toutes les commandes sont à adapter !**

### 2. Création du script `autosnap-921.sh`

```plaintext
nano /root/autosnap-921.sh
```

Collez le contenu suivant :

```bash
#!/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=""
############# SNAPSHOT SCRIPT ITSELF ######################
echo " " >> /root/autosnap-logs/921-$(date "+%Y-%m-%d").log
echo "###NEW JOB STARTS HERE###" >> /root/autosnap-logs/921-$(date "+%Y-%m-%d").log
cv4pve-autosnap --host=XXX.XXX.X.XXX --username=root@pam --password=XXXXXXXX --vmid=all snap --label=-921- --keep=2 >> /root/autosnap-logs/921-$(date "+%Y-%m-%d").log
echo "###NEW JOB ENDS HERE###" >> /root/autosnap-logs/921-$(date "+%Y-%m-%d").log
```

### 3. Personnalisation des arguments

| Argument | Description | Valeurs possibles |
| :--- | :--- | :--- |
| `--host=` | Adresse IP du nœud Proxmox. | `XXX.XXX.X.XXX` |
| `--username=` | Identifiant de connexion à Proxmox. | `root@pam` |
| `--password=` | Mot de passe du compte spécifié. | `XXXXXXXX` |
| `--vmid=` | ID de l'invité (ou `all`). | `all` |
| `--label=` | Nom de l'instantané (ex: `-921-`). | `-921-` |
| `--keep=` | Nombre d'instantanés à conserver. | `2` |

> **Note :** Pour un cluster, vous pouvez spécifier plusieurs IP : `--host='IP1,IP2,IP3'`.

### 4. Rendre le script exécutable

```plaintext
chmod +x /root/autosnap-921.sh
```

-----

## III. Planification avec Cron

### 1. Ouvrir le crontab

```plaintext
crontab -e
```

### 2. Ajouter la planification

Exécution tous les jours à 9 h 00 et 21 h 00 :

```plaintext
0 9,21 * * * /root/autosnap-921.sh
```

-----

## IV. Exemples de maintenance et de gestion

### 1. Suppression des instantanés (Nettoyage)

```bash
cv4pve-autosnap --host=XXX.XXX.X.XXX --username=root@pam --password=XXXXXXXX --vmid=all clean --label='-921-' --keep=0
```

### 2. Affichage du statut

```bash
cv4pve-autosnap --host=XXX.XXX.X.XXX --username=root@pam --password=XXXXXXXX --vmid=all status
```

-----

## V. Aller plus loin : stratégies de rétention avancées

Vous pouvez répliquer le modèle de rétention (horaire, quotidien, hebdomadaire, mensuel, annuel) proposé par Proxmox VE.

### 1. Création de scripts pour chaque intervalle

| Nom du Script | Fréquence | Label Ex. | Rétention |
| :--- | :--- | :--- | :--- |
| autosnap-hourly.sh | Chaque heure | -hourly- | --keep=3 |
| autosnap-daily.sh | Chaque jour | -daily- | --keep=7 |
| autosnap-weekly.sh | Chaque semaine | -weekly- | --keep=4 |
| autosnap-monthly.sh | Chaque mois | -monthly- | --keep=2 |
| autosnap-yearly.sh | Chaque année | -yearly- | --keep=1 |

### 2. Planification des tâches Cron

```cron
# Tâches planifiées
0 * * * * /root/autosnap-hourly.sh
10 0 * * * /root/autosnap-daily.sh
20 0 * * 0 /root/autosnap-weekly.sh
30 0 1 * * /root/autosnap-monthly.sh
40 0 1 1 * /root/autosnap-yearly.sh
```

-----

## VI. Exemple de fichier log

```plaintext
###NEW JOB STARTS HERE###
ACTION Snap
PVE Version: 8.3.2
VMs: all
Label: -921-
Keep: 2
Total execution 00:01:18.3608764
###NEW JOB ENDS HERE###
```

-----

## VII. Ressources et générateurs Cron

* [https://crontab.guru](https://crontab.guru)
* [https://crontab.cronhub.io](https://crontab.cronhub.io)
* [https://crontab-generator.org](https://crontab-generator.org)

---

> **Auteur** : ce guide est proposé par **Amaury aka BlablaLinux**. Retrouvez l'ensemble de mes services sur [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).
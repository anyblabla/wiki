---
title: Automatiser les instantanés (Snapshots) Proxmox VE avec CV4PVE-AUTOSNAP
description: >-
  Détails de l'installation et de la configuration de l'outil CV4PVE-AUTOSNAP
  (Corsinvest) dans un LXC pour automatiser les instantanés (Snapshots) de vos
  VM et CT sur Proxmox VE.
published: true
date: '2026-11-14T23:25:59.851Z'
tags: []
editor: markdown
dateCreated: '2024-12-28T18:49:39.139Z'
---

## 💡 Contexte et choix de l'outil

Pour la prise d'instantanés des invités sur [Proxmox VE](https://www.proxmox.com/en/proxmox-virtual-environment/overview), plusieurs méthodes existent :

- L'interface web ([GUI](https://w.wiki/CZ6F)) de Proxmox.
- La ligne de commande ([CLI](https://w.wiki/6gSf)) avec l'outil natif [PVESH](https://pve.proxmox.com/pve-docs/chapter-pvesh.html).
- Des outils tiers, tels que [CV4PVE-ADMIN](https://corsinvest.it/cv4pve-admin-proxmox/) (GUI multiclusters) qui utilise, en coulisses, **CV4PVE-AUTOSNAP**.

> **Note importante (mars 2026) :** Ce guide présente la configuration de base avec des scripts individuels. Pour une gestion optimisée sur cluster multi-nœuds, consultez la nouvelle version : [**Automatisation avancée des snapshots Proxmox (multi-nœuds)**](/fr/automatisation-snapshots-proxmox-multi-noeuds).

> J'utilise CV4PVE-ADMIN, qui est un gestionnaire multiclusters. Mais je n'utilise que la fonction de prise automatique d'instantanés. C'est sortir la grosse artillerie pour pas grand-chose.
>
> Je vais donc vous montrer comment utiliser CV4PVE-AUTOSNAP. Non pas sur l'hôte Proxmox lui-même, mais en l'isolant sur un conteneur ([LXC](https://w.wiki/CXkQ)).
>
> Personnellement, j'ai déployé un conteneur [Debian 12 Bookworm](https://www.debian.org/releases/bookworm/) (la procédure fonctionne également sous Debian 13 Trixie).

-----

## 1. Installation et préparation dans le conteneur LXC

Toutes les commandes suivantes doivent être exécutées à l'intérieur de votre conteneur LXC (Debian/Ubuntu) dédié.

### 1. Téléchargement du binaire

Récupérez la dernière version de CV4PVE-AUTOSNAP disponible à [cette adresse](https://github.com/Corsinvest/cv4pve-autosnap/releases). *(Version 1.15.0 au moment où j'écris ces lignes)* :

```bash
wget [https://github.com/Corsinvest/cv4pve-autosnap/releases/download/v1.15.0/cv4pve-autosnap-linux-x64.zip](https://github.com/Corsinvest/cv4pve-autosnap/releases/download/v1.15.0/cv4pve-autosnap-linux-x64.zip)
```

### 2. Décompression et installation

Mettez à jour vos dépôts, installez l'utilitaire `unzip`, décompressez l'archive et rendez le binaire exécutable :

```bash
apt update && apt install -y unzip
unzip cv4pve-autosnap-linux-x64.zip
chmod +x cv4pve-autosnap
```

### 3. Finalisation

Déplacez le fichier binaire dans le répertoire `/usr/local/bin/` (emplacement standard pour les outils ajoutés manuellement) afin de pouvoir l'exécuter de n'importe où dans le système :

```bash
mv cv4pve-autosnap /usr/local/bin/
mkdir -p /root/autosnap-logs
```

### 4. Vérification

Pour obtenir de l'aide sur les options disponibles :

```bash
cv4pve-autosnap --help
```

-----

## 2. Création du script d'instantanés (Snapshots)

Nous allons maintenant créer un premier script pour la prise d'instantanés à des heures précises (9h et 21h dans cet exemple).

### 1. Structure de la commande de base

La commande principale de `cv4pve-autosnap` utilise la structure suivante :

```bash
cv4pve-autosnap --host=XXX.XXX.X.XXX --username=root@pam --password=XXXXXXXX --vmid=XXX snap --label=XXXXX --keep=X
```

> **À partir d'ici, toutes les commandes sont à adapter !**
>
> **Conseil sécurité :** Dans cet exemple d'initiation, le mot de passe `root@pam` est utilisé en clair pour simplifier la mise en place. En production, il est fortement recommandé de créer un **Token API Proxmox** avec des droits restreints (`--api-token=utilisateur@pve!nom-token=valeur-secret`) plutôt que d'utiliser le compte `root`.

### 2. Création du script `autosnap-921.sh`

```bash
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

> **Note :** Pour un cluster, vous pouvez spécifier plusieurs IP : `--host="IP1,IP2,IP3"`.

### 4. Rendre le script exécutable

```bash
chmod +x /root/autosnap-921.sh
```

-----

## 3. Planification avec Cron

### 1. Ouvrir le crontab

```bash
crontab -e
```

### 2. Ajouter la planification

Exécution tous les jours à 9 h 00 et 21 h 00 :

```cron
0 9,21 * * * /root/autosnap-921.sh
```

-----

## 4. Mise à jour automatique de CV4PVE-AUTOSNAP

Comme l'outil est installé manuellement via un binaire GitHub, il n'est pas mis à jour par `apt`. Voici un petit script qui vérifie chaque semaine si une nouvelle version est disponible et l'installe automatiquement.

### 1. Création du script `update-autosnap.sh`

```bash
nano /usr/local/bin/update-autosnap.sh
```

Collez le contenu suivant :

```bash
#!/bin/bash

# Configuration
REPO="Corsinvest/cv4pve-autosnap"
INSTALL_DIR="/usr/local/bin"
BINARY_NAME="cv4pve-autosnap"
TMP_DIR="/tmp/autosnap_update"

# On s'assure que les outils nécessaires sont là
apt update && apt install curl wget unzip -y > /dev/null 2>&1

# 1. On récupère la dernière version sur GitHub
LATEST_TAG=$(curl -s [https://api.github.com/repos/$REPO/releases/latest](https://api.github.com/repos/$REPO/releases/latest) | grep tag_name | cut -d '"' -f 4)

# 2. On check la version actuelle
if [ -f "$INSTALL_DIR/$BINARY_NAME" ]; then
    CURRENT_VERSION="v$($INSTALL_DIR/$BINARY_NAME --version | grep -oE '[0-9]+\.[0-9]+\.[0-9]+')"
else
    CURRENT_VERSION="none"
fi

if [ "$LATEST_TAG" == "$CURRENT_VERSION" ]; then
    echo "✅ cv4pve-autosnap est déjà à jour ($CURRENT_VERSION)."
    exit 0
fi

echo "🚀 Nouvelle version trouvée : $LATEST_TAG (actuelle :$CURRENT_VERSION)"

# 3. Téléchargement du binaire linux-x64
mkdir -p $TMP_DIR
DOWNLOAD_URL=$(curl -s [https://api.github.com/repos/$REPO/releases/latest](https://api.github.com/repos/$REPO/releases/latest) | grep "browser_download_url" | grep "linux-x64.zip" | cut -d '"' -f 4)

wget -qO $TMP_DIR/autosnap.zip "$DOWNLOAD_URL"
unzip -qj $TMP_DIR/autosnap.zip -d$TMP_DIR
chmod +x $TMP_DIR/$BINARY_NAME
mv $TMP_DIR/$BINARY_NAME$INSTALL_DIR/

# Nettoyage
rm -rf $TMP_DIR

echo "✨ Mise à jour vers $LATEST_TAG terminée !"
```

### 2. Rendre le script exécutable et le tester

```bash
chmod +x /usr/local/bin/update-autosnap.sh
/usr/local/bin/update-autosnap.sh
```

### 3. Planification hebdomadaire dans Cron

Ajoutez cette ligne dans votre `crontab -e` pour vérifier les mises à jour tous les lundis à 4 h 00 :

```cron
# Mise à jour automatique de l'outil cv4pve-autosnap (Lundi à 4h00)
0 4 * * 1 /usr/local/bin/update-autosnap.sh > /dev/null 2>&1
```

-----

## 5. Exemples de maintenance et de gestion

### 1. Suppression des instantanés (Nettoyage)

```bash
cv4pve-autosnap --host=XXX.XXX.X.XXX --username=root@pam --password=XXXXXXXX --vmid=all clean --label='-921-' --keep=0
```

### 2. Affichage du statut

```bash
cv4pve-autosnap --host=XXX.XXX.X.XXX --username=root@pam --password=XXXXXXXX --vmid=all status
```

-----

## 6. Aller plus loin : stratégies de rétention avancées

Vous pouvez répliquer le modèle de rétention (horaire, quotidien, hebdomadaire, mensuel, annuel) proposé par Proxmox VE.

### 1. Création de scripts pour chaque intervalle

| Nom du Script | Fréquence | Label Ex. | Rétention |
| :--- | :--- | :--- | :--- |
| `autosnap-hourly.sh` | Chaque heure | `-hourly-` | `--keep=3` |
| `autosnap-daily.sh` | Chaque jour | `-daily-` | `--keep=7` |
| `autosnap-weekly.sh` | Chaque semaine | `-weekly-` | `--keep=4` |
| `autosnap-monthly.sh` | Chaque mois | `-monthly-` | `--keep=2` |
| `autosnap-yearly.sh` | Chaque année | `-yearly-` | `--keep=1` |

> **Attention :** Un instantané (snapshot) n'est pas une sauvegarde. Évitez de conserver trop d'instantanés sur le long terme (mensuels ou annuels), au risque d'alourdir votre stockage et de dégrader les performances (notamment sous ZFS). Réservez la rétention longue durée à un serveur de sauvegarde dédié comme **Proxmox Backup Server (PBS)**.

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

## 7. Exemple de fichier log

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

## 8. Ressources et générateurs Cron

- [Crontab.guru](https://crontab.guru)
- [Cronhub Crontab Generator](https://crontab.cronhub.io)
- [Crontab-generator.org](https://crontab-generator.org)

---

> **Auteur** : ce guide est proposé par **Amaury aka BlablaLinux**. Retrouvez l'ensemble de mes services sur [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).

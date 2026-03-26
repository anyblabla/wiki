---
title: Automatiser les instantanés (Snapshots) Proxmox VE avec CV4PVE-AUTOSNAP
description: Détails de l'installation et de la configuration de l'outil CV4PVE-AUTOSNAP (Corsinvest) dans un LXC pour automatiser les instantanés (Snapshots) de vos VM et CT sur Proxmox VE.
published: true
date: 2026-03-26T00:17:59.694Z
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

### 3\. Personnalisation des arguments

Vous devez adapter les arguments de la commande pour qu'elle corresponde à votre environnement Proxmox VE :

| Argument | Description | Valeurs possibles |
| :--- | :--- | :--- |
| `--host=` | Adresse IP du nœud Proxmox. | `XXX.XXX.X.XXX` |
| `--host=` | **Pour un cluster**, l'adresse IP du nœud maître, ou l'ensemble des adresses IP des nœuds. | `--host='XXX.XXX.X.XXX,XXX.XXX.X.XXX,XXX.XXX.X.XXX'` |
| `--username=` | Identifiant de connexion à Proxmox. | `--username=root@pam` **OU** `--username=XXXX@pve` |
| `--password=` | Mot de passe du compte spécifié. | `--password=XXXXXXXX` |
| `--vmid=` | ID de l'invité. Utilisez `all` pour tous les invités. | `--vmid=all` |
| `--vmid="all,-XXX"` | **Pour ignorer un invité**, utilisez cette syntaxe pour exclure l'ID spécifié. | `--vmid="all,-XXX"` |
| `--label=` | Nom de l'instantané (utilisé pour la rétention et le nettoyage). | `--label='-921-'` |
| `--keep=` | Spécifie le nombre d'instantanés avec ce label à conserver (rétention). | `--keep=X` |

> **Personnellement, je précise l'ensemble des adresses IP des nœuds ! Si une machine est inaccessible, les instantanés des invités sur les machines accessibles seront effectués.**

### 4\. Rendre le script exécutable

Sauvegardez le fichier (avec **CTRL + X**, puis **O**, puis **ENTER** sous `nano`), puis rendez le script exécutable :

```plaintext
chmod +x /root/autosnap-921.sh
```

-----

## III. Planification avec Cron

Nous allons créer une tâche Cron pour appeler ce script à des intervalles réguliers.

### 1\. Ouvrir le crontab

Ouvrez le fichier de planification des tâches Cron (utilisez `crontab -e` pour l'utilisateur actuel, qui est idéalement `root` dans le conteneur) :

```plaintext
crontab -e
```

### 2\. Ajouter la planification

Collez la ligne suivante pour exécuter le script tous les jours à 9 h 00 et 21 h 00 :

```plaintext
0 9,21 * * * /root/autosnap-921.sh
```

| Champ | Valeur | Explication |
| :--- | :--- | :--- |
| **Minute** | `0` | À la 0ème minute (début de l'heure) |
| **Heure** | `9,21` | À 9h et à 21h |
| **Jour du mois** | `*` | Tous les jours du mois |
| **Mois** | `*` | Tous les mois |
| **Jour de la semaine** | `*` | Tous les jours de la semaine |


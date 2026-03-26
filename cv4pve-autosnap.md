---
title: Automatiser les instantanés (Snapshots) Proxmox VE avec CV4PVE-AUTOSNAP
description: Détails de l'installation et de la configuration de l'outil CV4PVE-AUTOSNAP (Corsinvest) dans un LXC pour automatiser les instantanés (Snapshots) de vos VM et CT sur Proxmox VE.
published: true
date: 2026-03-26T00:23:50.125Z
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

-----

## IV. Exemples de maintenance et de gestion

L'outil permet de réaliser d'autres tâches utiles en plus de la création d'instantanés.

### 1\. Suppression des instantanés (Nettoyage)

Pour supprimer l'ensemble des instantanés qui possèdent le label `-921-` sur l'ensemble des invités (en définissant le seuil de rétention à 0), utilisez cette commande :

```bash
cv4pve-autosnap --host=XXX.XXX.X.XXX --username=root@pam --password=XXXXXXXX --vmid=all clean --label='-921-' --keep=0
```

### 2\. Affichage du statut

Pour obtenir des informations sur les instantanés existants effectués par l'outil, utilisez cette commande :

```bash
cv4pve-autosnap --host=XXX.XXX.X.XXX --username=root@pam --password=XXXXXXXX --vmid=all status
```

-----

## V. Aller plus loin : stratégies de rétention avancées

Vous pouvez répliquer le modèle de rétention (horaire, quotidien, hebdomadaire, mensuel, annuel) que propose l'outil de sauvegarde natif de Proxmox VE.

### 1. Création de scripts pour chaque intervalle

Créez des fichiers .sh dédiés pour chaque intervalle de sauvegarde, en adaptant le label et la rétention via l'argument --keep.

| Nom du Script | Fréquence | Label Ex. | Rétention |
| :--- | :--- | :--- | :--- |
| autosnap-hourly.sh | Chaque heure | -hourly- | --keep=3 |
| autosnap-daily.sh | Chaque jour | -daily- | --keep=7 |
| autosnap-weekly.sh | Chaque semaine | -weekly- | --keep=4 |
| autosnap-monthly.sh | Chaque mois | -monthly- | --keep=2 |
| autosnap-yearly.sh | Chaque année | -yearly- | --keep=1 |

### 2. Planification des tâches Cron

Vous créez une tâche Cron pour chaque fichier .sh dans votre crontab -e :

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

Le fichier log est essentiel pour valider que les opérations se sont déroulées correctement.

```plaintext
###NEW JOB STARTS HERE###
ACTION Snap
PVE Version:     8.3.2
VMs:             all
Label:           -921-
Keep:            2
...
Total execution 00:01:18.3608764
###NEW JOB ENDS HERE###
```

-----

## VII. Ressources et générateurs Cron

Pour faciliter la création de vos planifications :

* [https://crontab.guru](https://crontab.guru)
* [https://crontab.cronhub.io](https://crontab.cronhub.io)
* [https://crontab-generator.org](https://crontab-generator.org)

---

> **Auteur** : ce guide est proposé par **Amaury aka BlablaLinux**. Retrouvez l'ensemble de mes services sur [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).
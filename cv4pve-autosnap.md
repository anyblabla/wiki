---
title: Automatiser les instantanés (Snapshots) Proxmox VE avec CV4PVE-AUTOSNAP
description: Détails de l'installation et de la configuration de l'outil CV4PVE-AUTOSNAP (Corsinvest) dans un LXC pour automatiser les instantanés (Snapshots) de vos VM et CT sur Proxmox VE.
published: true
date: 2026-03-26T00:12:28.891Z
tags: 
editor: markdown
dateCreated: 2024-12-28T18:49:39.139Z
---


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

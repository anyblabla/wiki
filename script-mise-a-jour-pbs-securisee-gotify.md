---
title: Script de mise à jour Proxmox Backup Server (PBS) sécurisée via Cron et Gotify
description: Automatisation sécurisée des mises à jour de Proxmox Backup Server (PBS) via Cron et Gotify. Le script utilise apt-mark hold pour exclure le noyau et le service PBS des MAJ auto, garantissant la stabilité.
published: true
date: 2025-11-21T20:09:57.912Z
tags: proxmox, debian, apt, pbs, gotify, linux, curl, apt-mak, hold
editor: markdown
dateCreated: 2025-11-21T20:09:57.912Z
---

Ce guide explique comment mettre en place un script automatisé pour maintenir votre serveur de sauvegarde **Proxmox Backup Server (PBS)** à jour, en utilisant une méthode sécurisée qui exclut les composants critiques.

## Introduction et objectif

L'objectif est d'assurer que votre serveur de sauvegarde reste **sécurisé** et **à jour** sans intervention manuelle quotidienne.

| Composant | Description |
| :--- | :--- |
| **Script** | **`/usr/local/bin/update_pbs.sh`** |
| **Planification** | Tâche Cron exécutée par l'utilisateur `root` |
| **Fichier journal** | **`/var/log/proxmox_update.log`** |

-----

## I. Avertissement de sécurité : la stratégie du `hold`

Ce script utilise la commande **`apt-mark hold`** pour **exclure automatiquement les paquets critiques** de Proxmox Backup Server (noyau, service PBS) du processus de mise à jour automatique. Cette approche réduit le risque de rupture du système.

  * **Composants mis à jour automatiquement :** Paquets non-critiques, correctifs de sécurité non liés au noyau, utilitaires de base.
  * **Composants exclus et requérant une action manuelle :** Les paquets `proxmox-backup-server` et `pbs-kernel-*`.
  * **Action requise :** Le système de notification Gotify vous informe du succès des mises à jour non-critiques. Vous devez ensuite **vérifier manuellement** la nécessité d'installer les mises à jour critiques et de redémarrer le système.

-----

## II. Prérequis : installation et configuration des outils 🔔

### 1\. Installation de `curl`

Si ce paquet n'est pas installé sur votre hôte Proxmox Backup Server, installez-le. Il est nécessaire pour envoyer des notifications Gotify :

```bash
apt-get install curl -y
```

### 2\. Informations Gotify

Préparez les informations de votre serveur Gotify :

  * **URL Gotify** : L'URL complète de votre serveur (ex: `https://gotify.mondomaine.com`).
  * **Token Gotify** : Le jeton (Token) de l'application Gotify.

-----

## III. Création du script Bash sécurisé

### Étape 1 : créer le fichier `update_pbs.sh`

Connectez-vous à votre hôte PBS en **SSH** (en tant que `root`) et utilisez `nano` pour créer et éditer le fichier :

```bash
nano /usr/local/bin/update_pbs.sh
```

### Étape 2 : coller le contenu du script

Collez le contenu du script **Sécurisé** (celui qui inclut les étapes `apt-mark hold` et `unhold`). **⚠️ Remplacez `VOTRE_URL_GOTIFY` et `VOTRE_TOKEN_GOTIFY` par vos propres valeurs.**

*(Le contenu du script sécurisé se trouve juste avant cette section.)*

### Étape 3 : rendre le script exécutable

```bash
chmod +x /usr/local/bin/update_pbs.sh
```

-----

## IV. Configuration de la tâche Cron ⏱️

### Étape 1 : ouvrir le crontab de l'utilisateur root

```bash
crontab -e
```

### Étape 2 : ajouter la ligne de planification

Ajoutez la ligne suivante à la fin du fichier. Cet exemple planifie l'exécution du script tous les **dimanches à 3h30 du matin**.

```cron
# Mettre à jour Proxmox Backup Server (PBS) tous les dimanches à 3h30 du matin (paquets non-critiques)
30 3 * * 0 /usr/local/bin/update_pbs.sh
```

| Champ | Valeur | Description |
| :--- | :--- | :--- |
| **Minute** | `30` | 30e minute |
| **Heure** | `3` | 3h du matin |
| **Jour de la semaine** | `0` ou `7` | Dimanche |

### Étape 3 : enregistrer et quitter

-----

## V. Vérification et mises à jour manuelles 🧑‍💻

### 1\. Consulter le fichier journal

Après l'exécution planifiée, vous recevrez une notification Gotify. Vous pouvez consulter le journal pour vous assurer du succès des mises à jour non-critiques :

```bash
tail -f /var/log/proxmox_update.log
```

### 2\. Vérifier et installer les mises à jour critiques

Pour déterminer si de nouveaux paquets critiques (noyau, PBS) sont disponibles et en attente d'installation manuelle :

```bash
apt list --upgradable | grep -E 'pbs-kernel|proxmox-backup-server'
```

Si des paquets sont listés, exécutez la commande d'installation **manuellement** pendant une fenêtre de maintenance surveillée :

```bash
# Exemple d'installation manuelle (adaptez le nom des paquets)
apt-get install proxmox-backup-server pbs-kernel-X.Y.Z-pve -y
```

### 3\. Redémarrer l'hôte si nécessaire

Si vous avez installé un nouveau noyau, **un redémarrage est obligatoire** pour appliquer la mise à jour et garantir la stabilité du système. Redémarrez l'hôte PBS via l'interface Web ou en SSH :

```bash
reboot
```
---
title: Automatiser les mises à jour de Proxmox Backup Server (PBS)
description: >-
  Cette page de wiki détaille la procédure pour automatiser les mises à jour
  système de votre hôte Proxmox Backup Server (PBS) à l'aide d'un script Bash et
  de Cron.
published: true
date: '2026-11-02T23:22:36.004Z'
tags:
  - proxmox
  - cron
  - crontab
  - script
  - update
  - pbs
editor: markdown
dateCreated: '2025-10-30T13:23:36.129Z'
---

### 📝 Introduction et objectif

L'objectif est d'assurer que votre serveur de sauvegarde reste sécurisé et à jour sans intervention manuelle quotidienne.

> **Note importante :** L'automatisation des mises à jour, en particulier celles du noyau, comporte un risque. Ce script est conçu pour **ne pas redémarrer automatiquement** l'hôte.

| Composant | Description |
| :--- | :--- |
| **Script** | **`/usr/local/bin/update_pbs.sh`** |
| **Planification** | Tâche Cron exécutée par l'utilisateur `root` |
| **Fichier journal** | **`/var/log/proxmox_update.log`** |

-----

### ⚠️ Avertissement et vérification post-exécution

#### 1\. Absence de redémarrage automatique

Le script utilise les commandes `apt-get update` et `apt-get dist-upgrade` mais n'inclut **jamais** la commande `reboot`.

#### 2\. Action requise

**Après chaque exécution planifiée du script, vous devez :**

1.  **Vérifier le fichier journal** (`/var/log/proxmox_update.log`).
2.  **Redémarrer manuellement** l'hôte si un nouveau noyau a été installé, afin d'appliquer correctement les mises à jour critiques.

-----

### I. Création du script Bash

Nous allons créer un script exécutable dans le répertoire standard pour les commandes locales de l'administrateur : `/usr/local/bin/`.

#### Étape 1 : Créer le fichier `update_pbs.sh`

Connectez-vous à votre hôte PBS en SSH (en tant que `root`) et utilisez `nano` pour créer et éditer le fichier :

```bash
nano /usr/local/bin/update_pbs.sh
```

#### Étape 2 : Coller le contenu du script

Collez le contenu suivant dans le fichier. Il utilise **`dist-upgrade`**, la méthode recommandée pour les mises à jour des distributions Debian/Proxmox.

```bash
#!/bin/bash

# Fichier journal pour enregistrer le déroulement de la mise à jour
LOGFILE="/var/log/proxmox_update.log"

# Redirection de toute la sortie (standard et erreur) vers le fichier journal
exec 1>>$LOGFILE 2>&1

# --- Début du processus ---
echo "======================================================"
echo "Début de la mise à jour Proxmox Backup Server (PBS) : $(date)"
echo "======================================================"

# 1. Mise à jour de la liste des paquets (apt-get update)
echo "--- Étape 1 : apt-get update (Mise à jour des listes de paquets) ---"
apt-get update

if [ $? -ne 0 ]; then
    echo "Échec de la mise à jour des listes de paquets. Arrêt du script."
    exit 1
fi

# 2. Mise à niveau des paquets installés (apt-get dist-upgrade)
echo "--- Étape 2 : apt-get dist-upgrade (Mise à niveau des paquets) ---"
apt-get dist-upgrade -y

# 3. Suppression des dépendances inutiles (apt-get autoremove)
echo "--- Étape 3 : apt-get autoremove (Nettoyage des dépendances et anciens noyaux) ---"
apt-get autoremove -y

# 4. Nettoyage du cache APT
echo "--- Étape 4 : apt-get clean (Nettoyage du cache APT) ---"
apt-get clean

echo "======================================================"
echo "Fin de la mise à jour Proxmox Backup Server (PBS) : $(date)"
echo "======================================================"

exit 0
```

#### Étape 3 : Rendre le script exécutable

```bash
chmod +x /usr/local/bin/update_pbs.sh
```

-----

### II. Configuration de la tâche Cron

Nous allons planifier l'exécution du script via l'outil de planification de tâches `cron`, spécifiquement pour l'utilisateur `root`.

#### Étape 1 : Ouvrir le crontab de l'utilisateur root

```bash
crontab -e
```

#### Étape 2 : Ajouter la ligne de planification

Ajoutez la ligne suivante à la fin du fichier `crontab`.

Cet exemple planifie l'exécution du script **tous les dimanches à 3h30 du matin**. Ajustez les valeurs selon votre fenêtre de maintenance préférée.

```cron
# Mettre à jour Proxmox Backup Server (PBS) tous les dimanches à 3h30 du matin
30 3 * * 0 /usr/local/bin/update_pbs.sh
```

| Champ | Valeur | Description |
| :--- | :--- | :--- |
| **Minute** | `30` | 30e minute |
| **Heure** | `3` | 3h du matin |
| **Jour du mois** | `*` | Tous les jours |
| **Mois** | `*` | Tous les mois |
| **Jour de la semaine** | `0` ou `7` | Dimanche (0 et 7 sont des alias pour le dimanche) |

#### Étape 3 : Enregistrer et quitter

Enregistrez et quittez l'éditeur de `crontab`. La tâche planifiée est maintenant active.

-----

### III. Vérification de l'état du système

L'étape la plus critique est la vérification après l'exécution planifiée.

#### 1\. Consulter le fichier journal

Utilisez `tail` pour afficher les dernières entrées du journal et vérifier le bon déroulement de l'opération :

```bash
tail -f /var/log/proxmox_update.log
```

#### 2\. Vérifier la nécessité d'un redémarrage

Si le journal mentionne l'installation de paquets tels que **`pbs-kernel-*`**, un redémarrage est nécessaire.

Vous pouvez également utiliser cette commande pour vérifier si le système indique qu'un redémarrage est requis :

```bash
apt-get install -s | grep "reboot is required"
```

#### 3\. Redémarrer l'hôte si nécessaire

Si un nouveau noyau a été installé ou si la commande précédente l'indique, redémarrez l'hôte PBS via l'interface Web ou en SSH :

```bash
reboot
```

---
title: Script de mise à jour Proxmox Backup Server (PBS) sécurisée via Cron et Gotify
description: Automatisation sécurisée des mises à jour de Proxmox Backup Server (PBS) via Cron et Gotify. Le script utilise apt-mark hold pour exclure le noyau et le service PBS des MAJ auto, garantissant la stabilité.
published: true
date: 2026-04-27T10:27:21.861Z
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

### 1. Installation de `curl`

Si ce paquet n'est pas installé sur votre hôte Proxmox Backup Server, installez-le. Il est nécessaire pour envoyer des notifications Gotify :

```bash
apt-get install curl -y
```

### 2. Informations Gotify

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

Collez le contenu suivant dans le fichier. **⚠️ Remplacez `VOTRE_URL_GOTIFY` et `VOTRE_TOKEN_GOTIFY` par vos propres valeurs.**

```bash
#!/bin/bash

# --- PARAMÈTRES DE GOTIFY ---
# IMPORTANT : Remplacez ces valeurs !
GOTIFY_URL="VOTRE_URL_GOTIFY"
GOTIFY_TOKEN="VOTRE_TOKEN_GOTIFY"

# --- PARAMÈTRES DU SCRIPT ---
LOGFILE="/var/log/proxmox_update.log"
HOSTNAME=$(hostname)
UPDATE_SUCCESS=0
PACKAGES_TO_UPGRADE=0

# --- LISTE DES PAQUETS CRITIQUES À EXCLURE DE L'AUTOMATISATION ---
# Ces paquets (noyau, service PBS) seront exclus du dist-upgrade automatique.
CRITICAL_PACKAGES="proxmox-backup-server pbs-kernel-*"

# Redirection de toute la sortie vers le fichier journal
exec 1>>$LOGFILE 2>&1

# --- FONCTION DE NOTIFICATION GOTIFY (MÉTHODE FORM-DATA) ---
send_gotify_notification() {
    local title="$1"
    local message="$2"
    local priority="$3"

    curl -k -s -X POST "$GOTIFY_URL/message?token=$GOTIFY_TOKEN" \
        -F "title=$title" \
        -F "message=$message" \
        -F "priority=$priority" > /dev/null 2>&1
}

# --- DÉBUT DU PROCESSUS DE MISE À JOUR ---
echo "======================================================"
echo "Début du processus Proxmox Backup Server (PBS) sur $HOSTNAME : $(date)"
echo "======================================================"

# --- ÉTAPE 1a : MARQUER LES PAQUETS CRITIQUES EN ATTENTE (HOLD) ---
echo "--- Étape 1a : apt-mark hold (Exclusion des paquets critiques) ---"
echo "$CRITICAL_PACKAGES" | xargs -n 1 apt-mark hold

# 1. Mise à jour de la liste des paquets
echo "--- Étape 1b : apt-get update ---"
apt-get update

if [ $? -ne 0 ]; then
    echo "Échec de la mise à jour des listes. Arrêt."
    UPDATE_SUCCESS=1
else
    # 2. Simulation de mise à jour
    echo "--- Étape 2 : Simulation ---"
    PACKAGES_TO_UPGRADE=$(apt-get -s --assume-no dist-upgrade 2>/dev/null | grep -E '^(Inst|Upgr|Remv)' | wc -l)

    if [ "$PACKAGES_TO_UPGRADE" -gt 0 ]; then
        echo "Installation de $PACKAGES_TO_UPGRADE paquets non-critiques..."
        # 3, 4, 5. Installation et nettoyage
        apt-get dist-upgrade -y
        apt-get autoremove -y
        apt-get clean
    else
        echo "Aucune mise à jour non-critique disponible."
    fi
fi

# --- ÉTAPE 5b : LIBÉRER LES PAQUETS (UNHOLD) ---
echo "--- Étape 5b : apt-mark unhold ---"
echo "$CRITICAL_PACKAGES" | xargs -n 1 apt-mark unhold

echo "======================================================"
echo "Fin du processus PBS : $(date)"
echo "======================================================"

# --- ENVOI DE LA NOTIFICATION ---
if [ "$PACKAGES_TO_UPGRADE" -gt 0 ] || [ $UPDATE_SUCCESS -ne 0 ]; then
    if [ $UPDATE_SUCCESS -eq 0 ]; then
        TITLE="✅ PBS Update SUCCÈS sur $HOSTNAME"
        MESSAGE="Mise à jour de $PACKAGES_TO_UPGRADE paquet(s) terminée. Vérifiez les MAJ critiques."
        PRIORITY=4
    else
        TITLE="❌ PBS Update ÉCHEC sur $HOSTNAME"
        MESSAGE="L'opération a échoué. Consultez $LOGFILE."
        PRIORITY=8
    fi
    send_gotify_notification "$TITLE" "$MESSAGE" $PRIORITY
fi

exit $UPDATE_SUCCESS
```

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

### 1. Consulter le fichier journal

Après l'exécution planifiée, vous pouvez consulter le journal pour vous assurer du succès des opérations :

```bash
tail -f /var/log/proxmox_update.log
```

### 2. Vérifier et installer les mises à jour critiques

Pour déterminer si de nouveaux paquets critiques (noyau, PBS) sont disponibles pour une installation manuelle :

```bash
apt list --upgradable | grep -E 'pbs-kernel|proxmox-backup-server'
```

Si des paquets sont listés, exécutez l'installation manuellement :

```bash
# Exemple (adaptez le nom des paquets si nécessaire)
apt-get install proxmox-backup-server pbs-kernel-X.Y.Z-pve -y
```

### 3. Redémarrer l'hôte si nécessaire

Si vous avez installé un nouveau noyau, **un redémarrage est obligatoire** :

```bash
reboot
```

***

> **Auteur** : ce guide est proposé par **Amaury aka BlablaLinux**. Retrouvez l'ensemble de mes services sur [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).
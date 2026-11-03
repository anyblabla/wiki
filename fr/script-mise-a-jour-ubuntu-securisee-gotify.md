---
title: Script de mise à jour Ubuntu Server sécurisée via Cron et Gotify
description: >-
  Automatisation sécurisée des mises à jour d'Ubuntu Server (APT et Snap) via
  Cron et Gotify. Le script utilise apt-mark hold pour exclure les mises à jour
  du noyau, nécessitant une validation humaine.
published: true
date: '2026-11-03T00:05:31.517Z'
tags:
  - cron
  - ubuntu
  - apt
  - linux
  - snap
  - kernel
  - noyau
editor: markdown
dateCreated: '2025-11-21T20:17:24.484Z'
---

Ce guide explique comment mettre en place un script automatisé pour maintenir votre serveur Ubuntu à jour, en utilisant une méthode sécurisée qui exclut les mises à jour critiques du noyau.

## Introduction et objectif

L'objectif est d'assurer la sécurité et l'actualisation de votre serveur sans intervention manuelle quotidienne. Ce script gère les mises à jour **APT** et **Snap**, tout en vous notifiant immédiatement de l'état de l'opération.

| Composant | Description |
| :--- | :--- |
| **Script** | **`/usr/local/bin/update_ubuntu.sh`** |
| **Planification** | Tâche Cron exécutée par l'utilisateur `root` |
| **Fichier journal** | **`/var/log/ubuntu_update.log`** |

-----

## I. Avertissement de sécurité : la stratégie du `hold`

Ce script utilise la commande **`apt-mark hold`** pour **exclure automatiquement le noyau Linux** (`linux-image-*`, `linux-headers-*`) du processus de mise à jour automatique. Cette approche réduit le risque de problèmes de démarrage.

  * **Composants mis à jour automatiquement :** Paquets APT non-critiques, paquets Snap.
  * **Composants exclus et requérant une action manuelle :** Les paquets du noyau Linux.
  * **Action requise :** Le système de notification Gotify vous informe du succès des mises à jour non-critiques. **Vous devez installer manuellement les mises à jour du noyau** et planifier le redémarrage.

-----

## II. Prérequis : installation et configuration des outils 🔔

### 1\. Installation de `curl`

Si ce paquet n'est pas installé sur votre serveur Ubuntu, installez-le :

```bash
sudo apt-get install curl -y
```

### 2\. Informations Gotify

Préparez les informations de votre serveur Gotify :

  * **URL Gotify** : L'URL complète de votre serveur (ex: `https://gotify.mondomaine.com`).
  * **Token Gotify** : Le jeton (Token) de l'application Gotify.

-----

## III. Création du script Bash sécurisé

### Étape 1 : créer le fichier `update_ubuntu.sh`

1.  Connectez-vous à votre serveur Ubuntu en SSH et utilisez `nano` (ou `vi`) pour créer le fichier :

<!-- end list -->

```bash
sudo nano /usr/local/bin/update_ubuntu.sh
```

2.  Collez le contenu du script **Sécurisé**. **⚠️ Remplacez `VOTRE_URL_GOTIFY` et `VOTRE_TOKEN_GOTIFY` par vos propres valeurs.**

<!-- end list -->

```bash
#!/bin/bash

# --- PARAMÈTRES DE GOTIFY ---
# IMPORTANT : Remplacez ces valeurs !
GOTIFY_URL="VOTRE_URL_GOTIFY"
GOTIFY_TOKEN="VOTRE_TOKEN_GOTIFY"

# --- PARAMÈTRES DU SCRIPT ---
LOGFILE="/var/log/ubuntu_update.log"
HOSTNAME=$(hostname)
UPDATE_SUCCESS=0
# Compteurs pour les mises à jour
APT_UPDATED_PACKAGES=0
SNAP_UPDATED_PACKAGES=0
TOTAL_UPDATED_PACKAGES=0
UPDATES_WERE_AVAILABLE=0 # 1 si des paquets étaient à jour

# --- LISTE DES PAQUETS CRITIQUES À EXCLURE DE L'AUTOMATISATION ---
# On exclut le noyau pour éviter les problèmes au démarrage.
CRITICAL_PACKAGES="linux-image-* linux-headers-*"

# Redirection de toute la sortie vers le fichier journal
exec 1>>$LOGFILE 2>&1

# --- FONCTION DE NOTIFICATION GOTIFY (MÉTHODE FORM-DATA) ---
send_gotify_notification() {
    local title="$1"
    local message="$2"
    local priority="$3"

    # Envoi de la notification via curl
    curl -k -s -X POST "$GOTIFY_URL/message?token=$GOTIFY_TOKEN" \
        -F "title=$title" \
        -F "message=$message" \
        -F "priority=$priority" > /dev/null 2>&1
}

# --- DÉBUT DU PROCESSUS DE MISE À JOUR ---
echo "======================================================"
echo "Début du processus de mise à jour Ubuntu Server sur $HOSTNAME : $(date)"
echo "======================================================"

# --- ÉTAPE 0 : MARQUER LES PAQUETS CRITIQUES EN ATTENTE (HOLD) ---
echo "--- Étape 0 : apt-mark hold (Exclusion des noyaux critiques) ---"
echo "Mise en attente des paquets du noyau : $CRITICAL_PACKAGES"
# Note: sudo est nécessaire si cron n'est pas root
sudo echo "$CRITICAL_PACKAGES" | xargs -n 1 sudo apt-mark hold

# 1. Mise à jour de la liste des paquets APT (apt-get update)
echo "--- Étape 1 : apt-get update (Mise à jour des listes de paquets) ---"
sudo apt-get update

if [ $? -ne 0 ]; then
    echo "Échec de la mise à jour des listes de paquets. Arrêt du script."
    UPDATE_SUCCESS=1
else
    # 2. Vérification s'il y a des mises à jour APT disponibles (Simulation)
    echo "--- Étape 2 : apt-get upgrade -s (Simulation de mise à jour APT) ---"
    
    # La simulation ignore les paquets en HOLD.
    APT_UPDATED_PACKAGES=$(sudo apt-get -s --assume-no upgrade 2>/dev/null | grep -E '^(Inst|Upgr)' | wc -l)
    
    echo "$APT_UPDATED_PACKAGES paquets APT non-critiques à mettre à jour."
    
    # 3. Exécution des mises à jour APT si nécessaire
    if [ "$APT_UPDATED_PACKAGES" -gt 0 ]; then
        UPDATES_WERE_AVAILABLE=1
        echo "Démarrage de la mise à niveau APT (paquets non-critiques)..."
        sudo apt-get upgrade -y
        
        # 4. Suppression des dépendances inutiles (apt-get autoremove)
        echo "--- Étape 4 : apt-get autoremove (Nettoyage des dépendances) ---"
        sudo apt-get autoremove -y
    else
        echo "Aucune mise à jour APT non-critique disponible."
    fi

    # 5. Mise à jour des paquets Snap (spécifique à Ubuntu)
    echo "--- Étape 5 : snap refresh (Mise à jour des packages Snap) ---"
    
    SNAP_OUTPUT=$(sudo snap refresh 2>&1)
    SNAP_STATUS=$?

    if [ $SNAP_STATUS -ne 0 ]; then
        echo "Échec de la mise à jour Snap."
    elif echo "$SNAP_OUTPUT" | grep -q 'refreshed'; then
        SNAP_UPDATED_PACKAGES=1
        UPDATES_WERE_AVAILABLE=1
        echo "Au moins un paquet Snap a été mis à jour."
    else
        echo "Aucune mise à jour Snap nécessaire."
    fi
fi

# --- ÉTAPE 6 : LIBÉRER LES PAQUETS CRITIQUES (UNHOLD) ---
echo "--- Étape 6 : apt-mark unhold (Libération des paquets critiques) ---"
# On enlève le statut "hold" pour permettre une future mise à jour manuelle
sudo echo "$CRITICAL_PACKAGES" | xargs -n 1 sudo apt-mark unhold

# Calcul du total pour la notification
TOTAL_UPDATED_PACKAGES=$((APT_UPDATED_PACKAGES + SNAP_UPDATED_PACKAGES))

echo "======================================================"
echo "Fin du processus de mise à jour Ubuntu Server : $(date)"
echo "======================================================"

# --- ENVOI DE LA NOTIFICATION FINALE ---

# On notifie UNIQUEMENT si (des mises à jour ont été installées/trouvées) OU (il y a eu un ÉCHEC)
if [ "$UPDATES_WERE_AVAILABLE" -eq 1 ] || [ $UPDATE_SUCCESS -ne 0 ]; then
    if [ $UPDATE_SUCCESS -eq 0 ]; then
        # Succès (et il y avait des MAJ à faire)
        NOTIFICATION_TITLE="✅ Ubuntu Update SUCCÈS sur $HOSTNAME (Non-Critique)"
        NOTIFICATION_MESSAGE="La mise à jour Ubuntu s'est terminée. Total : $TOTAL_UPDATED_PACKAGES paquet(s) mis à jour (APT et/ou Snap). Noyau exclu. Vérifiez les MAJ noyaux manuellement."
        NOTIFICATION_PRIORITY=4 # Priorité moyenne
    else
        # Échec
        NOTIFICATION_TITLE="❌ Ubuntu Update ÉCHEC sur $HOSTNAME"
        NOTIFICATION_MESSAGE="La mise à jour d'Ubuntu a ÉCHOUÉ (apt-get update). Consultez $LOGFILE sur le serveur."
        NOTIFICATION_PRIORITY=8 # Haute priorité
    fi
    
    send_gotify_notification "$NOTIFICATION_TITLE" "$NOTIFICATION_MESSAGE" $NOTIFICATION_PRIORITY
else
    # Aucune mise à jour et pas d'échec
    echo "Aucun paquet mis à jour et aucun échec. Aucune notification Gotify envoyée."
fi

exit $UPDATE_SUCCESS
```

### Étape 2 : rendre le script exécutable

```bash
sudo chmod +x /usr/local/bin/update_ubuntu.sh
```

-----

## IV. Configuration de la tâche Cron ⏱️

### Étape 1 : ouvrir le crontab de l'utilisateur root

La tâche Cron doit être configurée dans la **crontab de l'utilisateur `root`** pour gérer les commandes `sudo` sans problème d'authentification.

```bash
sudo crontab -e
```

### Étape 2 : ajouter la ligne de planification

Ajoutez la ligne suivante à la fin du fichier. Cet exemple planifie l'exécution du script tous les **dimanches à 3h30 du matin**.

```cron
# Mettre à jour Ubuntu Server tous les dimanches à 3h30 du matin (paquets non-critiques)
30 3 * * 0 /usr/local/bin/update_ubuntu.sh
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

Après l'exécution planifiée, vous recevrez une notification Gotify. Consultez le journal pour vérifier l'état des mises à jour non-critiques :

```bash
tail -f /var/log/ubuntu_update.log
```

### 2\. Vérifier et installer les mises à jour du noyau

Pour déterminer si de nouveaux paquets du noyau sont disponibles et en attente d'installation manuelle :

```bash
apt list --upgradable | grep -E 'linux-image|linux-headers'
```

Si des paquets sont listés, exécutez la commande d'installation **manuellement** pendant une fenêtre de maintenance surveillée :

```bash
# Exemple d'installation manuelle du noyau (adaptez les noms des paquets)
sudo apt-get install linux-image-X.Y.Z linux-headers-X.Y.Z -y
```

### 3\. Redémarrer le serveur si nécessaire

Si vous avez installé un nouveau noyau, **un redémarrage est obligatoire** pour que le nouveau noyau soit utilisé.

```bash
sudo reboot
```

---
title: Script de mise à jour Proxmox VE sécurisée (via Cron et Gotify)
description: >-
  Automatisez les mises à jour non-critiques de Proxmox VE via Cron et Gotify.
  Ce script sécurisé utilise apt-mark hold pour exclure le noyau et les
  composants essentiels, assurant la stabilité tout en maintenant votre système
  à jour.
published: true
date: '2026-11-03T00:05:26.419Z'
tags:
  - proxmox
  - debian
  - pve
  - apt
  - gotify
  - linux
  - curl
  - apt-mak
  - apt-get
  - hold
editor: markdown
dateCreated: '2025-11-21T20:02:46.363Z'
---

Ce guide détaille l'installation et la configuration d'un script d'automatisation des mises à jour pour Proxmox VE. Le script inclut un mécanisme de sécurité pour mettre à jour les composants non-critiques tout en laissant la supervision humaine pour les éléments essentiels (noyau et hyperviseur).

## ⚠️ Avertissement de sécurité : la stratégie du `hold`

Ce script utilise la commande **`apt-mark hold`** pour **exclure automatiquement les paquets critiques** de Proxmox VE (noyau, hyperviseur, etc.) du processus de mise à jour automatique. Cette approche réduit significativement le risque de rupture du système due à un correctif non testé.

  * **Composants mis à jour automatiquement :** Paquets non-critiques, correctifs de sécurité non liés au noyau, utilitaires de base.
  * **Composants exclus et requérant une action manuelle :** Les paquets `pve-kernel-*`, `proxmox-ve`, et `pve-manager`.
  * **Action requise :** Le système de notification Gotify vous informe du succès des mises à jour non-critiques. Vous devez ensuite **vérifier manuellement** la nécessité d'installer les mises à jour critiques et de redémarrer le système.

-----

## I. Prérequis : installation et configuration des outils 🔔

### 1\. Installation de `curl`

Si ce paquet n'est pas installé sur votre hôte Proxmox, installez-le. Il est nécessaire pour envoyer des notifications à Gotify :

```bash
apt-get install curl -y
```

### 2\. Informations Gotify

Préparez les informations de votre serveur Gotify :

  * **URL Gotify** : L'URL complète de votre serveur (ex: `https://gotify.mondomaine.com`).
  * **Token Gotify** : Le jeton (Token) de l'application Gotify.

-----

## II. Création du script Bash sécurisé

### Étape 1 : créer le fichier `update_pve.sh`

Connectez-vous à votre hôte Proxmox en **SSH** (en tant que `root`) et utilisez `nano` pour créer et éditer le fichier de script dans le répertoire des exécutables locaux :

```bash
nano /usr/local/bin/update_pve.sh
```

### Étape 2 : coller le contenu du script

Collez le contenu suivant. **⚠️ Remplacez `VOTRE_URL_GOTIFY` et `VOTRE_TOKEN_GOTIFY` par vos propres valeurs.**

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
PACKAGES_TO_UPGRADE=0 # Nouveau compteur

# --- LISTE DES PAQUETS CRITIQUES À EXCLURE DE L'AUTOMATISATION ---
# Ces paquets (noyau, meta-package PVE, gestionnaire) seront exclus du dist-upgrade automatique.
CRITICAL_PACKAGES="proxmox-ve pve-manager pve-kernel-*"

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
echo "Début du processus Proxmox VE sur $HOSTNAME : $(date)"
echo "======================================================"

# --- ÉTAPE 1a : MARQUER LES PAQUETS CRITIQUES EN ATTENTE (HOLD) ---
echo "--- Étape 1a : apt-mark hold (Exclusion des paquets critiques) ---"
echo "Mise en attente des paquets critiques (noyau, PVE manager, etc.) : $CRITICAL_PACKAGES"
echo "$CRITICAL_PACKAGES" | xargs -n 1 apt-mark hold

# 1. Mise à jour de la liste des paquets (apt-get update)
echo "--- Étape 1b : apt-get update (Mise à jour des listes de paquets) ---"
apt-get update

if [ $? -ne 0 ]; then
    echo "Échec de la mise à jour des listes de paquets. Arrêt du script."
    UPDATE_SUCCESS=1
else
    # 2. Vérification s'il y a des mises à jour disponibles
    echo "--- Étape 2 : apt-get dist-upgrade -s (Simulation de mise à jour) ---"
    # Les paquets en HOLD sont ignorés, ne comptant que les mises à jour non-critiques.
    PACKAGES_TO_UPGRADE=$(apt-get -s --assume-no dist-upgrade 2>/dev/null | grep -E '^(Inst|Upgr|Remv)' | wc -l)

    echo "Nombre de paquets NON-CRITIQUES à mettre à jour/installer/supprimer : $PACKAGES_TO_UPGRADE"

    if [ "$PACKAGES_TO_UPGRADE" -gt 0 ]; then
        echo "Des mises à jour non-critiques sont disponibles. Démarrage de l'installation..."

        # 3. Mise à niveau des paquets installés (apt-get dist-upgrade)
        echo "--- Étape 3 : apt-get dist-upgrade (Mise à niveau des paquets) ---"
        apt-get dist-upgrade -y

        # 4. Suppression des dépendances inutiles (apt-get autoremove)
        echo "--- Étape 4 : apt-get autoremove (Nettoyage des dépendances) ---"
        apt-get autoremove -y

        # 5. Nettoyage du cache APT
        echo "--- Étape 5a : apt-get clean (Nettoyage du cache APT) ---"
        apt-get clean
    else
        echo "Aucune mise à jour de paquet non-critique disponible. Les étapes d'installation seront ignorées."
    fi
fi

# --- ÉTAPE 5b : LIBÉRER LES PAQUETS CRITIQUES (UNHOLD) ---
echo "--- Étape 5b : apt-mark unhold (Libération des paquets critiques) ---"
# On enlève le statut "hold" pour permettre une future mise à jour manuelle
echo "$CRITICAL_PACKAGES" | xargs -n 1 apt-mark unhold

echo "======================================================"
echo "Fin du processus Proxmox VE : $(date)"
echo "======================================================"

# --- ENVOI DE LA NOTIFICATION FINALE ---

if [ "$PACKAGES_TO_UPGRADE" -gt 0 ] || [ $UPDATE_SUCCESS -ne 0 ]; then
    if [ $UPDATE_SUCCESS -eq 0 ]; then
        # Succès de la mise à jour (et il y avait des MAJ à faire)
        NOTIFICATION_TITLE="✅ PVE Update SUCCÈS sur $HOSTNAME (Non-Critique)"
        NOTIFICATION_MESSAGE="La mise à jour de $PACKAGES_TO_UPGRADE paquet(s) non-critiques s'est terminée. Vérifiez les MAJ critiques manuellement."
        NOTIFICATION_PRIORITY=4 # Priorité moyenne
    else
        # Échec
        NOTIFICATION_TITLE="❌ PVE Update ÉCHEC sur $HOSTNAME"
        NOTIFICATION_MESSAGE="La mise à jour de PVE a ÉCHOUÉ (apt-get update). Consultez $LOGFILE sur l'hôte."
        NOTIFICATION_PRIORITY=8 # Haute priorité
    fi

    send_gotify_notification "$NOTIFICATION_TITLE" "$NOTIFICATION_MESSAGE" $NOTIFICATION_PRIORITY
else
    # Aucune mise à jour et pas d'échec
    echo "Aucun paquet mis à jour et aucun échec. Aucune notification Gotify envoyée."
fi

exit $UPDATE_SUCCESS
```

### Étape 3 : rendre le script exécutable

```bash
chmod +x /usr/local/bin/update_pve.sh
```

-----

## III. Configuration de la tâche Cron ⏱️

### Étape 1 : ouvrir le crontab de l'utilisateur root

```bash
crontab -e
```

### Étape 2 : ajouter la ligne de planification

Ajoutez la ligne suivante à la fin du fichier. Cet exemple planifie l'exécution du script tous les **dimanches à 3h30 du matin**.

```cron
# Mettre à jour Proxmox VE tous les dimanches à 3h30 du matin (paquets non-critiques)
30 3 * * 0 /usr/local/bin/update_pve.sh
```

| Champ | Valeur | Description |
| :--- | :--- | :--- |
| **Minute** | `30` | 30e minute |
| **Heure** | `3` | 3h du matin |
| **Jour de la semaine** | `0` ou `7` | Dimanche |

### Étape 3 : enregistrer et quitter

-----

## IV. Vérification et mises à jour manuelles 🧑‍💻

### 1\. Consulter le fichier journal

Après l'exécution planifiée, vous recevrez une notification Gotify. Vous pouvez consulter le journal pour plus de détails :

```bash
tail -f /var/log/proxmox_update.log
```

### 2\. Vérifier et installer les mises à jour critiques

Pour déterminer si de nouveaux paquets critiques (noyau, hyperviseur) sont disponibles et en attente d'installation manuelle :

```bash
apt list --upgradable | grep -E 'pve-kernel|proxmox-ve|pve-manager'
```

Si des paquets sont listés, exécutez la commande d'installation **manuellement** pendant une fenêtre de maintenance surveillée :

```bash
# Exemple d'installation manuelle (adaptez le nom des paquets)
apt-get install proxmox-ve pve-manager pve-kernel-X.Y.Z-pve -y
```

### 3\. Redémarrer l'hôte si nécessaire

Si vous avez installé un nouveau noyau, **un redémarrage est obligatoire** pour appliquer la mise à jour et garantir la stabilité du système. Redémarrez l'hôte Proxmox via l'interface Web ou en SSH :

```bash
reboot
```

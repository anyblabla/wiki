---
title: Script de mise à jour de Proxmox VE par Cron avec notification Gotify
description: >-
  Script Cron pour mise à jour Proxmox VE : Automatisez apt-get dist-upgrade et
  recevez une notification immédiate via Gotify en cas de succès ou d'échec.
  Inclut les pré-requis curl.
published: true
date: '2026-11-03T00:04:16.607Z'
tags:
  - proxmox
  - cron
  - crontab
  - script
  - bash
  - pve
  - update
  - gotify
editor: markdown
dateCreated: '2025-10-30T20:46:02.780Z'
---

a## ⚠️ Avertissement important

L'automatisation des mises à jour système (surtout celles du noyau) comporte toujours un risque. **Ce script n'inclut pas de redémarrage automatique.**

  * **Action requise :** Après l'exécution du script, **vérifiez toujours** le fichier journal (`/var/log/proxmox_update.log`). Si un nouveau noyau a été installé, vous devez **redémarrer manuellement** l'hôte pour appliquer les mises à jour du noyau et garantir la stabilité du système.
  * **Fréquence :** Il est recommandé de planifier cette tâche pendant les heures creuses (la nuit, le week-end).

-----

## I. Pré-requis : installation de `curl` et configuration Gotify 🔔

### 1\. Installation de `curl`

Si ce paquet n'est pas installé sur votre hôte Proxmox, installez-le :

```bash
apt-get install curl -y
```

### 2\. Informations Gotify

Vous aurez besoin de deux informations :

  * **URL Gotify** : L'URL complète de votre serveur (ex: `https://gotify.mondomaine.com`).
  * **Token Gotify** : Le jeton (Token) de l'application Gotify.

-----

## II. Création du script Bash avec notification Gotify (Form-Data)

### Étape 1 : créer le fichier `update_pve.sh`

Connectez-vous à votre hôte Proxmox en **SSH** (en tant que `root`) et utilisez `nano` pour créer et éditer le fichier :

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

# Redirection de toute la sortie vers le fichier journal
exec 1>>$LOGFILE 2>&1

# --- FONCTION DE NOTIFICATION GOTIFY (MÉTHODE FORM-DATA) ---
# Le token est passé dans l'URL et l'option -k est utilisée pour ignorer les erreurs SSL/TLS.
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

# 1. Mise à jour de la liste des paquets (apt-get update)
echo "--- Étape 1 : apt-get update (Mise à jour des listes de paquets) ---"
apt-get update

if [ $? -ne 0 ]; then
    echo "Échec de la mise à jour des listes de paquets. Arrêt du script."
    UPDATE_SUCCESS=1
else
    # 2. Vérification s'il y a des mises à jour disponibles
    echo "--- Étape 2 : apt-get dist-upgrade -s (Simulation de mise à jour) ---"
    # L'option -s simule. On compte le nombre de lignes avec "Inst" (Installé),
    # "Upgr" (Mis à jour) ou "Remv" (Supprimé) pour déterminer si quelque chose se passe.
    # On ajoute --assume-no pour être sûr qu'aucun prompt n'interfère, même si -s devrait suffire.
    # On utilise la sortie standard et on filtre pour compter les actions réelles sur les paquets.
    PACKAGES_TO_UPGRADE=$(apt-get -s --assume-no dist-upgrade 2>/dev/null | grep -E '^(Inst|Upgr|Remv)' | wc -l)
    
    echo "Nombre de paquets à mettre à jour/installer/supprimer : $PACKAGES_TO_UPGRADE"

    if [ "$PACKAGES_TO_UPGRADE" -gt 0 ]; then
        echo "Des mises à jour sont disponibles. Démarrage de l'installation..."
        
        # 3. Mise à niveau des paquets installés (apt-get dist-upgrade)
        echo "--- Étape 3 : apt-get dist-upgrade (Mise à niveau des paquets) ---"
        apt-get dist-upgrade -y

        # 4. Suppression des dépendances inutiles (apt-get autoremove)
        echo "--- Étape 4 : apt-get autoremove (Nettoyage des dépendances et anciens noyaux) ---"
        apt-get autoremove -y

        # 5. Nettoyage du cache APT
        echo "--- Étape 5 : apt-get clean (Nettoyage du cache APT) ---"
        apt-get clean
    else
        echo "Aucune mise à jour de paquet disponible. Les étapes d'installation seront ignorées."
    fi
fi

echo "======================================================"
echo "Fin du processus Proxmox VE : $(date)"
echo "======================================================"

# --- ENVOI DE LA NOTIFICATION FINALE ---

# On envoie la notification UNIQUEMENT si (il y a eu des MAJ) OU (il y a eu un ÉCHEC)
if [ "$PACKAGES_TO_UPGRADE" -gt 0 ] || [ $UPDATE_SUCCESS -ne 0 ]; then
    if [ $UPDATE_SUCCESS -eq 0 ]; then
        # Succès de la mise à jour (et il y avait des MAJ à faire)
        NOTIFICATION_TITLE="✅ PVE Update SUCCÈS sur $HOSTNAME"
        NOTIFICATION_MESSAGE="La mise à jour de $PACKAGES_TO_UPGRADE paquet(s) s'est terminée. Redémarrage nécessaire si nouveau noyau."
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
# Mettre à jour Proxmox VE tous les dimanches à 3h30 du matin
30 3 * * 0 /usr/local/bin/update_pve.sh
```

| Champ | Valeur | Description |
| :--- | :--- | :--- |
| **Minute** | `30` | 30e minute |
| **Heure** | `3` | 3h du matin |
| **Jour de la semaine** | `0` ou `7` | Dimanche (0 et 7 sont des alias pour le dimanche) |

### Étape 3 : enregistrer et quitter

-----

## IV. Vérification (post-exécution)

Après l'heure planifiée, vous recevrez une notification Gotify confirmant le statut.

### 1\. Consulter le fichier journal

```bash
tail -f /var/log/proxmox_update.log
```

### 2\. Vérifier la nécessité d'un redémarrage

Si le journal mentionne l'installation de paquets **`pve-kernel-*`**, ou si la commande suivante retourne des informations, un redémarrage est nécessaire :

```bash
apt-get install -s | grep "reboot is required"
```

### 3\. Redémarrer l'hôte si nécessaire

Si un nouveau noyau a été installé, redémarrez l'hôte Proxmox via l'interface Web ou en SSH :

```bash
reboot
```

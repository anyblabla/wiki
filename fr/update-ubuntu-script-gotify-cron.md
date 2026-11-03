---
title: Script de mise à jour d'Ubuntu Server par Cron avec notification Gotify
description: >-
  Script Cron pour mise à jour Ubuntu : Automatisez apt-get upgrade et snap
  refresh sur votre serveur Ubuntu. Recevez une notification immédiate via
  Gotify en cas de succès ou d'échec de la mise à jour.
published: true
date: '2026-11-03T00:04:10.051Z'
tags:
  - serveur
  - crontab
  - ubuntu
  - script
  - bash
  - update
  - gotify
editor: markdown
dateCreated: '2025-10-30T21:01:00.149Z'
---

a## ⚠️ Avertissement important

L'automatisation des mises à jour système (surtout celles du noyau) comporte toujours un risque. **Ce script n'inclut pas de redémarrage automatique.**

  * **Action requise :** Après l'exécution du script, **vérifiez toujours** le fichier journal (`/var/log/ubuntu_update.log`). Si un nouveau noyau (kernel) a été installé, vous devez **redémarrer manuellement** le serveur pour que les mises à jour prennent effet.
  * **Privilèges :** La tâche Cron doit être configurée dans la **crontab de l'utilisateur `root`** pour gérer les commandes `sudo` sans problème d'authentification.

-----

## I. Pré-requis : installation de `curl` et configuration Gotify 🔔

### 1\. Installation de `curl`

Si ce paquet n'est pas installé sur votre serveur Ubuntu, installez-le :

```bash
sudo apt-get install curl -y
```

### 2\. Informations Gotify

Vous aurez besoin de deux informations :

  * **URL Gotify** : L'URL complète de votre serveur (ex: `https://gotify.mondomaine.com`).
  * **Token Gotify** : Le jeton (Token) de l'application Gotify.

-----

## II. Création du script Bash avec notification Gotify (Form-Data)

### Étape 1 : créer le fichier `update_ubuntu.sh`

1.  Connectez-vous à votre serveur Ubuntu en SSH et utilisez `nano` (ou `vi`) pour créer le fichier.

<!-- end list -->

```bash
sudo nano /usr/local/bin/update_ubuntu.sh
```

2.  Collez le contenu suivant. **⚠️ Remplacez `VOTRE_URL_GOTIFY` et `VOTRE_TOKEN_GOTIFY` par vos propres valeurs.**

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
echo "Début du processus de mise à jour Ubuntu Server sur $HOSTNAME : $(date)"
echo "======================================================"

# 1. Mise à jour de la liste des paquets APT (apt-get update)
echo "--- Étape 1 : apt-get update (Mise à jour des listes de paquets) ---"
sudo apt-get update

if [ $? -ne 0 ]; then
    echo "Échec de la mise à jour des listes de paquets. Arrêt du script."
    UPDATE_SUCCESS=1
else
    # 2. Vérification s'il y a des mises à jour APT disponibles (Simulation)
    echo "--- Étape 2 : apt-get upgrade -s (Simulation de mise à jour APT) ---"
    
    # Utilisation de la simulation pour compter les paquets qui seraient mis à jour
    # On filtre les lignes qui commencent par 'Inst' (install) ou 'Upgr' (upgrade)
    APT_UPDATED_PACKAGES=$(sudo apt-get -s --assume-no upgrade 2>/dev/null | grep -E '^(Inst|Upgr)' | wc -l)
    
    echo "$APT_UPDATED_PACKAGES paquets APT à mettre à jour."
    
    # 3. Exécution des mises à jour APT si nécessaire
    if [ "$APT_UPDATED_PACKAGES" -gt 0 ]; then
        UPDATES_WERE_AVAILABLE=1
        echo "Démarrage de la mise à niveau APT..."
        sudo apt-get upgrade -y
        
        # 4. Suppression des dépendances inutiles (apt-get autoremove)
        echo "--- Étape 4 : apt-get autoremove (Nettoyage des dépendances et anciens noyaux) ---"
        sudo apt-get autoremove -y
    else
        echo "Aucune mise à jour APT disponible."
    fi

    # 5. Mise à jour des paquets Snap (spécifique à Ubuntu)
    echo "--- Étape 5 : snap refresh (Mise à jour des packages Snap) ---"
    
    # Exécution de la mise à jour snap et capture de la sortie
    SNAP_OUTPUT=$(sudo snap refresh 2>&1)
    SNAP_STATUS=$?

    if [ $SNAP_STATUS -ne 0 ]; then
        echo "Échec de la mise à jour Snap."
    elif echo "$SNAP_OUTPUT" | grep -q 'refreshed'; then
        # On considère qu'il y a eu une mise à jour si 'refreshed' apparaît dans la sortie
        SNAP_UPDATED_PACKAGES=1
        UPDATES_WERE_AVAILABLE=1
        echo "Au moins un paquet Snap a été mis à jour."
    else
        echo "Aucune mise à jour Snap nécessaire."
    fi
fi

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
        NOTIFICATION_TITLE="✅ Ubuntu Update SUCCÈS sur $HOSTNAME"
        NOTIFICATION_MESSAGE="La mise à jour Ubuntu s'est terminée. Total : $TOTAL_UPDATED_PACKAGES paquet(s) mis à jour (APT et/ou Snap). Redémarrage nécessaire si nouveau noyau."
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

### Étape 3 : rendre le script exécutable

```bash
sudo chmod +x /usr/local/bin/update_ubuntu.sh
```

-----

## III. Configuration de la tâche Cron (recommandé : utilisateur root) ⏱️

### Étape 1 : ouvrir le crontab de l'utilisateur root

```bash
sudo crontab -e
```

### Étape 2 : ajouter la ligne de planification

Ajoutez la ligne suivante à la fin du fichier. Cet exemple planifie l'exécution du script tous les **dimanches à 3h30 du matin**.

```cron
# Mettre à jour Ubuntu Server tous les dimanches à 3h30 du matin
30 3 * * 0 /usr/local/bin/update_ubuntu.sh
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
tail -f /var/log/ubuntu_update.log
```

### 2\. Vérifier la nécessité d'un redémarrage

Utilisez cette commande pour vérifier si un redémarrage est nécessaire :

```bash
apt-get install -s | grep "reboot is required"
```

### 3\. Redémarrer le serveur si nécessaire

```bash
sudo reboot
```

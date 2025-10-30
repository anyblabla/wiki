---
title: Script de mise à jour d'Ubuntu Server par Cron avec notification Gotify
description: Script Cron pour mise à jour Ubuntu : Automatisez apt-get upgrade et snap refresh sur votre serveur Ubuntu. Recevez une notification immédiate via Gotify en cas de succès ou d'échec de la mise à jour.
published: true
date: 2025-10-30T21:51:09.053Z
tags: serveur, crontab, ubuntu, script, bash, update, gotify
editor: markdown
dateCreated: 2025-10-30T21:01:00.149Z
---

## ⚠️ Avertissement important

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
echo "Début de la mise à jour Ubuntu Server sur $HOSTNAME : $(date)"
echo "======================================================"

# 1. Mise à jour de la liste des paquets APT (apt-get update)
echo "--- Étape 1 : apt-get update (Mise à jour des listes de paquets) ---"
sudo apt-get update

if [ $? -ne 0 ]; then
    echo "Échec de la mise à jour des listes de paquets. Arrêt du script."
    UPDATE_SUCCESS=1
else
    # 2. Mise à niveau des paquets installés (apt-get upgrade)
    echo "--- Étape 2 : apt-get upgrade (Mise à niveau des paquets) ---"
    sudo apt-get upgrade -y

    # 3. Suppression des dépendances inutiles (apt-get autoremove)
    echo "--- Étape 3 : apt-get autoremove (Nettoyage des dépendances et anciens noyaux) ---"
    sudo apt-get autoremove -y

    # 4. Mise à jour des paquets Snap (spécifique à Ubuntu)
    echo "--- Étape 4 : snap refresh (Mise à jour des packages Snap) ---"
    sudo snap refresh
fi

echo "======================================================"
echo "Fin de la mise à jour Ubuntu Server : $(date)"
echo "======================================================"

# --- ENVOI DE LA NOTIFICATION FINALE ---

if [ $UPDATE_SUCCESS -eq 0 ]; then
    # Succès
    NOTIFICATION_TITLE="✅ Ubuntu Update SUCCÈS sur $HOSTNAME"
    NOTIFICATION_MESSAGE="La mise à jour Ubuntu s'est terminée. Vérifiez si un redémarrage est nécessaire."
    NOTIFICATION_PRIORITY=4 # Priorité moyenne
else
    # Échec
    NOTIFICATION_TITLE="❌ Ubuntu Update ÉCHEC sur $HOSTNAME"
    NOTIFICATION_MESSAGE="La mise à jour d'Ubuntu a ÉCHOUÉ (apt-get update). Consultez $LOGFILE sur le serveur."
    NOTIFICATION_PRIORITY=8 # Haute priorité
fi

send_gotify_notification "$NOTIFICATION_TITLE" "$NOTIFICATION_MESSAGE" $NOTIFICATION_PRIORITY

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
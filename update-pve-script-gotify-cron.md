---
title: Script de mise à jour de Proxmox VE par Cron avec notification Gotify
description: Script Cron pour mise à jour Proxmox VE : Automatisez apt-get dist-upgrade et recevez une notification immédiate via Gotify en cas de succès ou d'échec. Inclut les pré-requis curl.
published: true
date: 2025-10-30T20:53:21.989Z
tags: proxmox, cron, crontab, script, bash, pve, update, gotify
editor: markdown
dateCreated: 2025-10-30T20:46:02.780Z
---

## ⚠️ Avertissement important

L'automatisation des mises à jour système (surtout celles du noyau) comporte toujours un risque. **Ce script n'inclut pas de redémarrage automatique.**

  * **Action requise :** Après l'exécution du script, **vérifiez toujours** le fichier journal (`/var/log/proxmox_update.log`). Si un nouveau noyau a été installé, vous devez **redémarrer manuellement** l'hôte pour appliquer les mises à jour du noyau et garantir la stabilité du système.
  * **Fréquence :** Il est recommandé de planifier cette tâche pendant les heures creuses (la nuit, le week-end).

-----

## I. Pré-requis : installation de `curl` et configuration Gotify 🔔

Le script utilise l'outil **`curl`** pour envoyer la notification au service **Gotify**. Vous devez vous assurer que `curl` est installé et que vous avez vos identifiants Gotify.

### 1\. Installation de `curl`

Si ce paquet n'est pas installé sur votre hôte Proxmox, installez-le :

```bash
apt-get install curl -y
```

### 2\. Informations Gotify

Vous aurez besoin de deux informations :

  * **URL Gotify** : L'URL complète de votre serveur (ex: `https://gotify.mondomaine.com`).
  * **Token Gotify** : Le jeton (Token) de l'application Gotify que vous souhaitez utiliser pour la notification.

-----

## II. Création du script Bash avec notification Gotify

### Étape 1 : créer le fichier `update_pve.sh`

Connectez-vous à votre hôte Proxmox en **SSH** (en tant que `root`) et utilisez `nano` pour créer et éditer le fichier :

```bash
nano /usr/local/bin/update_pve.sh
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
UPDATE_SUCCESS=0 # Statut de la mise à jour (0 = succès, 1 = échec)

# Redirection de toute la sortie vers le fichier journal
exec 1>>$LOGFILE 2>&1

# --- FONCTION DE NOTIFICATION GOTIFY ---
# Envoie un message à Gotify avec titre, corps et priorité
send_gotify_notification() {
    local title="$1"
    local message="$2"
    local priority="$3" # 1 (normal) à 10 (critique)
    
    # Envoi de la notification via curl
    # Note: Le résultat de curl est redirigé vers /dev/null pour éviter de polluer le journal
    curl -s -X POST "$GOTIFY_URL/message" \
        -H "Content-Type: application/json" \
        -d "{
              \"title\": \"$title\",
              \"message\": \"$message\",
              \"priority\": $priority\",
              \"token\": \"$GOTIFY_TOKEN\"
            }" > /dev/null 2>&1
}

# --- DÉBUT DU PROCESSUS DE MISE À JOUR ---
echo "======================================================"
echo "Début de la mise à jour Proxmox VE sur $HOSTNAME : $(date)"
echo "======================================================"

# 1. Mise à jour de la liste des paquets (apt-get update)
echo "--- Étape 1 : apt-get update (Mise à jour des listes de paquets) ---"
apt-get update

if [ $? -ne 0 ]; then
    # Si apt-get update échoue, on marque l'échec et on sort
    echo "Échec de la mise à jour des listes de paquets. Arrêt du script."
    UPDATE_SUCCESS=1
else
    # 2. Mise à niveau des paquets installés (apt-get dist-upgrade)
    echo "--- Étape 2 : apt-get dist-upgrade (Mise à niveau des paquets) ---"
    # dist-upgrade est la méthode recommandée pour les mises à jour d'hôtes Debian/Proxmox
    apt-get dist-upgrade -y

    # 3. Suppression des dépendances inutiles (apt-get autoremove)
    echo "--- Étape 3 : apt-get autoremove (Nettoyage des dépendances et anciens noyaux) ---"
    apt-get autoremove -y

    # 4. Nettoyage du cache APT
    echo "--- Étape 4 : apt-get clean (Nettoyage du cache APT) ---"
    apt-get clean
fi

echo "======================================================"
echo "Fin de la mise à jour Proxmox VE : $(date)"
echo "======================================================"

# --- ENVOI DE LA NOTIFICATION FINALE ---

if [ $UPDATE_SUCCESS -eq 0 ]; then
    # Succès
    NOTIFICATION_TITLE="✅ PVE Update SUCCÈS sur $HOSTNAME"
    NOTIFICATION_MESSAGE="La mise à jour Proxmox VE s'est terminée avec succès. Vérifiez si un redémarrage est nécessaire."
    NOTIFICATION_PRIORITY=4 # Priorité moyenne
else
    # Échec
    NOTIFICATION_TITLE="❌ PVE Update ÉCHEC sur $HOSTNAME"
    NOTIFICATION_MESSAGE="La mise à jour de Proxmox VE a ÉCHOUÉ (apt-get update). Consultez $LOGFILE sur l'hôte."
    NOTIFICATION_PRIORITY=8 # Haute priorité
fi

send_gotify_notification "$NOTIFICATION_TITLE" "$NOTIFICATION_MESSAGE" $NOTIFICATION_PRIORITY

exit $UPDATE_SUCCESS
```

### Étape 3 : rendre le script exécutable

```bash
chmod +x /usr/local/bin/update_pve.sh
```

-----

## III. Configuration de la tâche Cron ⏱️

Nous allons ajouter une entrée au `crontab` de l'utilisateur `root` pour planifier l'exécution du script.

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

Après l'heure planifiée, vous recevrez une notification Gotify confirmant le statut. Vous devez ensuite vérifier le journal et la nécessité d'un redémarrage.

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
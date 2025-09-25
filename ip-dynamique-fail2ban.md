---
title: Gestion d’IP Dynamique Fail2Ban (LXC/Docker)
description: Guide pour configurer un script qui met à jour automatiquement votre adresse IP publique dans le fichier /root/f2b/data/jail.d/npm.local de Fail2Ban toutes les 5 minutes, en gérant la rotation et en préservant toutes les autres adresses statiques.
published: true
date: 2025-09-25T22:01:24.189Z
tags: docker, lxc, proxmox, fail2ban, ip
editor: markdown
dateCreated: 2025-09-25T21:59:07.073Z
---

Mon fournisseur d'accès internet ne m'alloue **ni** une adresse IP publique statique **ni** une adresse IP publique dynamique au sens strict.

Je m'explique.

Mon adresse IP publique est **quasi-statique** : elle reste la même, sauf si je demande **explicitement** un changement via le routeur, ou dans de **rares** cas lors d'un redémarrage de celui-ci.

Bien que mon adresse change rarement, elle n'en reste pas moins **variable**.

Chez **OVH**, l'entrée **A** de mon domaine "**blablalinux**" est automatiquement mise à jour par le logiciel **ddclient**, installé sur un serveur local.

Le seul souci se produit lorsqu'un changement d'adresse IP publique a lieu : **fail2ban** me bannit, car ma nouvelle adresse ne figure pas dans la directive "**ignoreip**" section **[npm]** de mon fichier "**jail.local**".

La dernière fois, j'ai cherché **longtemps** avant d'identifier le problème. C'est un scénario qui se répète.

Pour information, mon infrastructure repose sur **Proxmox VE**, où j'ai un conteneur Linux (LXC) hébergeant trois conteneurs Docker : **Nginx Proxy Manager**, **Fail2Ban** et **GoAccess**.

Mais ce problème ne se reproduira plus : voici la solution que j'ai trouvée !

-----

## Guide Complet : Gestion d'IP Dynamique Fail2Ban (LXC/Docker)

Ce guide résume toutes les étapes pour configurer un script qui met à jour automatiquement votre adresse IP publique dans le fichier `/root/f2b/data/jail.d/npm.local` de Fail2Ban toutes les 5 minutes, en gérant la rotation et en préservant toutes les autres adresses statiques.

### 1\. Le Script Final et Sécurisé

Le script gère la suppression des anciennes adresses IP et l'ajout de la nouvelle, en ciblant le conteneur `fail2ban`. **Vérifiez le chemin `FAIL2BAN_CONFIG` et `FAIL2BAN_CONTAINER`.**

```bash
#!/bin/bash

# --- ⚠️ PARAMÈTRES OBLIGATOIRES À VÉRIFIER ⚠️ ---
# Le chemin vers le fichier npm.local dans votre volume monté
FAIL2BAN_CONFIG="/root/f2b/data/jail.d/npm.local"
# Le nom de votre conteneur Docker Fail2Ban
FAIL2BAN_CONTAINER="fail2ban" 
# Fichier de suivi dans le LXC pour stocker l'IP gérée
IP_FILE="/var/run/current_public_ip.txt"
# ------------------------------------------------

# 1. Récupérer l'ancienne IP enregistrée (l'IP gérée par le script)
if [ -f "$IP_FILE" ]; then
    OLD_IP=$(cat "$IP_FILE" | head -n 1)
else
    OLD_IP="" 
fi

# 2. Récupérer la nouvelle IP publique actuelle
NEW_IP=$(curl -s https://api.ipify.org)

if [ -z "$NEW_IP" ]; then
    echo "$(date) - Erreur: Impossible de récupérer la nouvelle adresse IP publique." >> /var/log/fail2ban_ip_update.log
    exit 1
fi

# Préparer les formats CIDR
NEW_IP_CIDR="$NEW_IP/32"
OLD_IP_CIDR="$OLD_IP/32"

# Sécurité : Échapper les points des IPs pour la commande sed
OLD_IP_ESCAPED=$(echo "$OLD_IP_CIDR" | sed 's/\./\\./g')
NEW_IP_ESCAPED=$(echo "$NEW_IP_CIDR" | sed 's/\./\\./g')

# 3. Comparer les adresses IP
if [ "$OLD_IP" = "$NEW_IP" ]; then
    echo "$(date) - IP inchangée ($NEW_IP). Aucune action requise." >> /var/log/fail2ban_ip_update.log
    exit 0
fi

# 4. Traitement des changements
echo "$(date) - Changement d'IP détecté. Ancienne: $OLD_IP, Nouvelle: $NEW_IP" >> /var/log/fail2ban_ip_update.log

# 4.1. Suppression de toutes les anciennes occurrences
if [ ! -z "$OLD_IP" ]; then
    # Nettoie l'IP que ce script a précédemment insérée (avec ou sans espace de tête)
    sed -i "s| $OLD_IP_ESCAPED||g" "$FAIL2BAN_CONFIG"
    sed -i "s|$OLD_IP_ESCAPED||g" "$FAIL2BAN_CONFIG"
    echo "$(date) - IP $OLD_IP_CIDR nettoyée de ignoreip." >> /var/log/fail2ban_ip_update.log
fi

# 4.2. Ajout de la nouvelle IP
# Utilisation du délimiteur '#' et de l'IP échappée pour la substitution.
sed -i "/\[npm\]/,/ignoreip/ s#^\(ignoreip\s*=\s*.*\)\$#\1 $NEW_IP_ESCAPED#" "$FAIL2BAN_CONFIG"
echo "$(date) - Nouvelle IP $NEW_IP_CIDR ajoutée à ignoreip." >> /var/log/fail2ban_ip_update.log

# 5. Mettre à jour le fichier de suivi
echo "$NEW_IP" > "$IP_FILE"

# 6. Rechargement de Fail2Ban
if docker exec "$FAIL2BAN_CONTAINER" fail2ban-client reload; then
    echo "$(date) - Fail2Ban rechargé avec succès." >> /var/log/fail2ban_ip_update.log
else
    echo "$(date) - Échec du rechargement. Tentative de redémarrage du conteneur..." >> /var/log/fail2ban_ip_update.log
    docker restart "$FAIL2BAN_CONTAINER"
    echo "$(date) - Conteneur Fail2Ban redémarré." >> /var/log/fail2ban_ip_update.log
fi

exit 0
```

-----

### 2\. Procédure d'Installation Complète (Dans le LXC)

#### Étape 1 : Préparation du Script

1.  **Créez et collez** le script ci-dessus dans le fichier :
    ```bash
    nano /usr/local/bin/update_fail2ban_ip.sh
    ```
2.  **Rendez-le exécutable** :
    ```bash
    chmod +x /usr/local/bin/update_fail2ban_ip.sh
    ```
3.  **Créez le fichier de log** :
    ```bash
    touch /var/log/fail2ban_ip_update.log
    ```

#### Étape 2 : Configuration de la Tâche Cron

1.  **Ouvrez la table cron** (en tant que `root` est recommandé) :
    ```bash
    crontab -e
    ```
2.  **Ajoutez cette ligne** pour une exécution toutes les **5 minutes** :
    ```cron
    */5 * * * * /usr/local/bin/update_fail2ban_ip.sh >> /var/log/fail2ban_ip_update.log 2>&1
    ```

#### Étape 3 : Validation Manuelle (Recommandé)

1.  **Exécutez le script manuellement** pour vérifier le premier ajout :
    ```bash
    /usr/local/bin/update_fail2ban_ip.sh
    ```
2.  **Vérifiez le journal** pour confirmer le succès :
    ```bash
    tail /var/log/fail2ban_ip_update.log
    ```
3.  **Vérifiez le fichier de configuration** (`/root/f2b/data/jail.d/npm.local`) : Votre IP doit y figurer, ajoutée à la fin de la ligne `ignoreip`.
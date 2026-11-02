---
title: Gestion de l'IP publique quasi-statique dans Fail2Ban (LXC/Docker)
description: >-
  Solution pour IP quasi-statique : Un script met à jour automatiquement votre
  IP publique dans la config Fail2Ban (jail NPM) via Cron pour éviter un
  bannissement lors d'un changement d'IP. Fini les bannissements intempestifs !
published: true
date: '2026-11-02T23:38:28.766Z'
tags:
  - docker
  - lxc
  - proxmox
  - fail2ban
  - ip
editor: markdown
dateCreated: '2025-09-25T21:59:07.073Z'
---

a## 1\. Contexte du Problème

Votre fournisseur d'accès Internet vous alloue une adresse IP publique **variable** mais **quasi-statique** (elle change rarement, souvent suite à un redémarrage du routeur ou sur demande explicite).

Le problème survient lorsque cette adresse change : **Fail2Ban**, hébergé dans un conteneur Docker sur votre LXC, vous bannit car votre nouvelle IP ne figure plus dans la directive `ignoreip` de la section `[npm]` de votre fichier `jail.local` (ou `npm.local`).

La solution ci-dessous automatise la mise à jour de cette directive.

-----

## 2\. Le Script d'Automatisation Sécurisé

Ce script gère la récupération de la nouvelle IP, la comparaison, la suppression de l'ancienne IP dans le fichier de configuration Fail2Ban, l'ajout de la nouvelle, et le rechargement conditionnel du service Fail2Ban (ou le redémarrage du conteneur en cas d'échec).

> **⚠️ Vérifiez les chemins et noms de conteneur ci-dessous avant toute exécution \!**

```bash
#!/bin/bash

# --- ⚠️ PARAMÈTRES OBLIGATOIRES À VÉRIFIER ⚠️ ---
FAIL2BAN_CONFIG="/root/f2b/data/jail.d/npm.local"
FAIL2BAN_CONTAINER="fail2ban" 
IP_FILE="/var/run/current_public_ip.txt"
# ------------------------------------------------

# 1. Récupérer l'ancienne IP enregistrée
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

# Préparer les formats CIDR et échappés
NEW_IP_CIDR="$NEW_IP/32"
OLD_IP_CIDR="$OLD_IP/32"
OLD_IP_ESCAPED=$(echo "$OLD_IP_CIDR" | sed 's/\./\\./g')
NEW_IP_ESCAPED=$(echo "$NEW_IP_CIDR" | sed 's/\./\\./g')

# 3. Comparer les adresses IP
if [ "$OLD_IP" = "$NEW_IP" ]; then
    echo "$(date) - IP inchangée ($NEW_IP). Aucune action requise." >> /var/log/fail2ban_ip_update.log
    exit 0
fi

# 4. Traitement des changements
echo "$(date) - Changement d'IP détecté. Ancienne: $OLD_IP, Nouvelle: $NEW_IP" >> /var/log/fail2ban_ip_update.log
# 🚨 Initialisation de la variable de contrôle
SHOULD_RELOAD=false

# 4.1. Suppression de toutes les anciennes occurrences
if [ ! -z "$OLD_IP" ]; then
    sed -i "s| $OLD_IP_ESCAPED||g" "$FAIL2BAN_CONFIG"
    sed -i "s|$OLD_IP_ESCAPED||g" "$FAIL2BAN_CONFIG"
    echo "$(date) - IP $OLD_IP_CIDR nettoyée de ignoreip." >> /var/log/fail2ban_ip_update.log
    SHOULD_RELOAD=true
fi

# 4.2. Suppression préventive des doublons de la NOUVELLE IP
# Cela gère le cas où l'IP n'a pas changé, mais un doublon existe déjà.
if grep -E '^\s*ignoreip\s*=' "$FAIL2BAN_CONFIG" | grep -q "$NEW_IP_ESCAPED"; then
    sed -i "s| $NEW_IP_ESCAPED||g" "$FAIL2BAN_CONFIG"
    sed -i "s|$NEW_IP_ESCAPED||g" "$FAIL2BAN_CONFIG"
    SHOULD_RELOAD=true
fi

# 4.3. Ajout de la nouvelle IP (une seule fois, après nettoyage)
sed -i "/\[npm\]/,/ignoreip/ s#^\(ignoreip\s*=\s*.*\)\$#\1 $NEW_IP_ESCAPED#" "$FAIL2BAN_CONFIG"
echo "$(date) - Nouvelle IP $NEW_IP_CIDR ajoutée à ignoreip." >> /var/log/fail2ban_ip_update.log
SHOULD_RELOAD=true

# 5. Mettre à jour le fichier de suivi
echo "$NEW_IP" > "$IP_FILE"

# 6. Rechargement de Fail2Ban (conditionnel)
# 🚨 Utilisation de la variable de contrôle
if $SHOULD_RELOAD; then
    if docker exec "$FAIL2BAN_CONTAINER" fail2ban-client reload; then
        echo "$(date) - Fail2Ban rechargé avec succès." >> /var/log/fail2ban_ip_update.log
    else
        echo "$(date) - Échec du rechargement. Tentative de redémarrage du conteneur..." >> /var/log/fail2ban_ip_update.log
        docker restart "$FAIL2BAN_CONTAINER"
        echo "$(date) - Conteneur Fail2Ban redémarré." >> /var/log/fail2ban_ip_update.log
    fi
fi
# 🚨 Fin du script
exit 0
```

-----

## 3\. Procédure d'Installation Complète (Dans le LXC)

Les étapes suivantes sont à effectuer **à l'intérieur** de votre conteneur LXC hébergeant Docker.

### Étape 1 : Préparation du Script

1.  **Créer le fichier script** et coller le code ci-dessus :
    ```bash
    nano /usr/local/bin/update_fail2ban_ip.sh
    ```
2.  **Rendre le script exécutable** :
    ```bash
    chmod +x /usr/local/bin/update_fail2ban_ip.sh
    ```
3.  **Créer le fichier de log** (pour le suivi des opérations) :
    ```bash
    touch /var/log/fail2ban_ip_update.log
    ```

### Étape 2 : Configuration de la Tâche Cron

1.  **Ouvrir la table cron** (en tant que `root` est fortement recommandé pour les commandes `docker`) :
    ```bash
    crontab -e
    ```
2.  **Ajouter la ligne suivante** pour que le script s'exécute **toutes les 5 minutes** :
    ```cron
    */5 * * * * /usr/local/bin/update_fail2ban_ip.sh >> /var/log/fail2ban_ip_update.log 2>&1
    ```

### Étape 3 : Validation Manuelle (Recommandé)

1.  **Exécutez le script manuellement** une première fois pour initialiser le processus et vérifier l'ajout de votre IP actuelle :
    ```bash
    /usr/local/bin/update_fail2ban_ip.sh
    ```
2.  **Vérifiez le journal** pour confirmer le succès du rechargement et de l'opération :
    ```bash
    tail /var/log/fail2ban_ip_update.log
    ```
3.  **Vérifiez le fichier de configuration** (`/root/f2b/data/jail.d/npm.local`) pour vous assurer que votre adresse IP figure bien dans la directive `ignoreip`.

Votre Fail2Ban est désormais **auto-adaptatif** aux changements d'adresse IP publique de votre routeur.

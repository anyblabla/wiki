---
title: Script de nettoyage LXC Proxmox avec Gotify
description: >-
  Automatisez le nettoyage de vos LXC (Debian/Ubuntu/Alpine) sur Proxmox. Ce
  script supprime caches, logs et paquets inutiles, puis envoie un rapport
  détaillé via Gotify.
published: true
date: '2026-11-03T00:05:35.543Z'
tags:
  - lxc
  - proxmox
  - debian
  - ubuntu
  - script
  - bash
  - nettoyage
  - cleanup
  - alpine
  - gotify
editor: markdown
dateCreated: '2025-11-22T21:55:39.053Z'
---

Ce script Bash est conçu pour automatiser le nettoyage régulier des conteneurs LXC sur Proxmox VE. Il cible la suppression des caches de paquets, des journaux (`logs`) et des fichiers temporaires pour économiser de l'espace disque. Le script gère les distributions **Debian**, **Ubuntu** et **Alpine** et envoie un rapport détaillé via **Gotify**.

-----

## 📋 Prérequis

1.  **Hôte Proxmox VE :** Accès SSH au nœud Proxmox.
2.  **Gotify :** Un serveur Gotify fonctionnel et une application de jeton (Token) associée.
3.  **Permissions :** Le script doit être exécuté par l'utilisateur `root` ou un utilisateur ayant les permissions `sudo` pour utiliser les commandes `pct`.

-----

## 📥 Le Script (`clean_lxcs.sh`)

Créez le fichier `/root/scripts/clean_lxcs.sh` et copiez-y le contenu suivant.

```bash
#!/bin/bash
#
# SCRIPT : clean_lxcs.sh
# OBJECTIF : Nettoyer les caches, logs et paquets inutiles des conteneurs LXC
#            Debian/Ubuntu et Alpine en cours d'exécution.
#
# ==============================================================================

# --- PARAMÈTRES DE GOTIFY (À MODIFIER !) ---
GOTIFY_URL="VOTRE_URL_GOTIFY" 
GOTIFY_TOKEN="VOTRE_TOKEN_GOTIFY"

# --- PARAMÈTRES DU SCRIPT ---
LOGFILE="/var/log/clean_lxcs_cron.log"
# CTIDs à exclure (séparés par des espaces, ex: "101 105 110")
EXCLUDED_CTIDS="" 
SUCCESS_COUNT=0
FAILURE_COUNT=0
CLEANED_CT_COUNT=0 
UNSUPPORTED_CT_COUNT=0
HOSTNAME=$(hostname)

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

echo "=================================================="
echo "Démarrage du nettoyage des LXC le $(date)"
echo "=================================================="

# Boucle sur les conteneurs en cours d'exécution
for CTID in $(/usr/sbin/pct list | grep running | awk '{print $1}')
do
    if [[ " ${EXCLUDED_CTIDS} " =~ " ${CTID} " ]]; then
        echo "    [SKIP] Conteneur $CTID exclu manuellement."
        continue
    fi

    echo "--> Traitement du conteneur CTID $CTID..."

    # Déterminer le type d'OS via la configuration Proxmox
    OS_TYPE=$(/usr/sbin/pct config $CTID | awk '/^ostype/ {print $2}')
    
    # 1. Définition des commandes de nettoyage
    CLEAN_CMD=""
    
    if [ "$OS_TYPE" == "debian" ] || [ "$OS_TYPE" == "ubuntu" ]; then
        echo "    - Préparation du nettoyage (Debian/Ubuntu)..."
        CLEAN_CMD="
            echo 'Nettoyage des caches (Debian/Ubuntu)...';
            # Nettoyage des caches divers (/var/cache), logs et fichiers temporaires
            find /var/cache -type f -delete 2>/dev/null;
            find /var/log -type f -delete 2>/dev/null;
            find /tmp -mindepth 1 -delete 2>/dev/null;
            
            # Suppression des paquets inutiles et nettoyage des caches APT
            export DEBIAN_FRONTEND=noninteractive;
            apt-get update -y >/dev/null 2>&1 || true;
            apt-get -y --purge autoremove;
            apt-get -y autoclean;
            apt-get clean;
            
            # Suppression du cache des listes APT
            rm -rf /var/lib/apt/lists/*;
            echo 'Nettoyage terminé.'
        "
        
    elif [ "$OS_TYPE" == "alpine" ]; then
        echo "    - Préparation du nettoyage (Alpine)..."
        CLEAN_CMD="
            echo 'Nettoyage des caches (Alpine)...';
            # Nettoyage du cache APK, logs et fichiers temporaires
            apk cache clean;
            find /var/log -type f -delete 2>/dev/null;
            find /tmp -mindepth 1 -delete 2>/dev/null;
            echo 'Nettoyage terminé.'
        "
    else
        echo "    [SKIP] Conteneur $CTID est de type OS $OS_TYPE : non pris en charge."
        UNSUPPORTED_CT_COUNT=$((UNSUPPORTED_CT_COUNT + 1))
        continue
    fi

    # 2. Exécution du nettoyage
    echo "    - Exécution des commandes de nettoyage..."
    /usr/sbin/pct exec $CTID -- bash -c "$CLEAN_CMD"

    if [ $? -ne 0 ]; then
        echo "    [ERREUR CRITIQUE] Le nettoyage du conteneur $CTID a échoué."
        FAILURE_COUNT=$((FAILURE_COUNT + 1))
        continue
    else
        echo "    [OK] Nettoyage du conteneur $CTID réussi."
        SUCCESS_COUNT=$((SUCCESS_COUNT + 1))
        CLEANED_CT_COUNT=$((CLEANED_CT_COUNT + 1))
    fi

done

echo "=================================================="
echo "Fin du nettoyage des LXC le $(date)"
echo "=================================================="

# --- ENVOI DE LA NOTIFICATION FINALE CONDITIONNELLE ---
if [ $CLEANED_CT_COUNT -gt 0 ] || [ $FAILURE_COUNT -gt 0 ] || [ $UNSUPPORTED_CT_COUNT -gt 0 ]; then
    
    if [ $FAILURE_COUNT -gt 0 ]; then
        TITLE="❌ LXC Nettoyage ÉCHEC(s) sur $HOSTNAME"
        MESSAGE="$FAILURE_COUNT LXC ont rencontré une ERREUR. $CLEANED_CT_COUNT LXC nettoyés. $UNSUPPORTED_CT_COUNT ignorés (OS non supporté)."
        PRIORITY=8
    else
        TITLE="✅ LXC Nettoyage SUCCÈS sur $HOSTNAME"
        MESSAGE="$CLEANED_CT_COUNT LXC nettoyés avec succès. $UNSUPPORTED_CT_COUNT ignorés (OS non supporté)."
        PRIORITY=4
    fi

    send_gotify_notification "$TITLE" "$MESSAGE" $PRIORITY
else
    echo "Aucun conteneur n'a été nettoyé. Aucune notification Gotify envoyée."
fi

exec 1>&- 2>&-
exit 0
```

-----

## 🛠️ Configuration et déploiement

### 1\. Adaptation des paramètres

Modifiez les deux premières lignes du script :

| Variable | Description | Exemple |
| :--- | :--- | :--- |
| `GOTIFY_URL` | URL de base de votre serveur Gotify (sans `/message`). | `https://gotify.mondomaine.com` |
| `GOTIFY_TOKEN` | Jeton d'application que le script utilisera pour envoyer le message. | `AzErTy1234` |
| `EXCLUDED_CTIDS` | Liste des CTID à ignorer, séparés par des espaces. | `"105 112"` |

### 2\. Permissions

Rendez le script exécutable :

```bash
chmod +x /root/scripts/clean_lxcs.sh
```

### 3\. Automatisation (Cron)

Il est recommandé d'exécuter le nettoyage **mensuellement** ou **bimensuellement**.

Pour une exécution le **premier jour du mois à 03h30**, éditez votre `crontab` en tant que `root` :

```bash
crontab -e
```

Ajoutez la ligne suivante :

```cron
30 3 1 * * /root/scripts/clean_lxcs.sh
```

### 4\. Vérification des logs

Après chaque exécution, vous pouvez consulter les détails dans le fichier de log :

```bash
cat /var/log/clean_lxcs_cron.log
```

-----

## 🔍 Détails du nettoyage

Le script utilise les commandes internes les plus efficaces pour chaque OS supporté :

| OS | Commandes de nettoyage principales | Objectif |
| :--- | :--- | :--- |
| **Debian/Ubuntu** | `apt-get --purge autoremove` | Suppression des paquets orphelins. |
| | `apt-get autoclean; apt-get clean` | Vidange complète du cache des fichiers `.deb`. |
| | `find /var/cache; find /var/log; find /tmp` | Suppression des fichiers de cache divers, des logs et des fichiers temporaires. |
| **Alpine** | `apk cache clean` | Suppression du cache des paquets APK. |
| | `find /var/log; find /tmp` | Suppression des logs et des fichiers temporaires. |

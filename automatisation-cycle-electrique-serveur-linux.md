---
title: Automatisation du cycle électrique (cold boot) pour serveurs Linux
description: Apprenez à automatiser proprement l'arrêt et le redémarrage électrique de vos serveurs Linux via SSH et Wake-on-LAN. Un guide complet pour sécuriser vos cycles de maintenance avec notifications Gotify.
published: true
date: 2026-04-06T07:38:52.067Z
tags: proxmox, ssh, script, gotify, linux, maintenance, automatisation, wake-on-lan
editor: markdown
dateCreated: 2026-04-06T07:15:43.821Z
---

Ce guide explique comment automatiser proprement l'arrêt, la pause électrique et le redémarrage (via Wake-on-LAN) d'un serveur distant. Cette méthode est idéale pour réinitialiser des contrôleurs matériels (USB, disques) ou économiser de l'énergie.

> **Le petit mot de BlablaLinux** : Dans mon propre labo, j'utilise ce script pour piloter mon serveur **Ubuntu Server** qui fait tourner **Nextcloud**. C'est mon nœud **Proxmox** (Debian) principal qui envoie les commandes. Ça me permet d'être sûr que mes disques de sauvegarde USB sont bien réinitialisés électriquement chaque semaine !

## 1. Prérequis techniques

### Côté serveur (la machine à piloter, ex : Ubuntu Server)
1.  **Dans le BIOS/UEFI** : activez l'option **Wake on LAN (WOL)**.
2.  **Interface réseau** : vérifiez le support WOL avec la commande `ethtool <NOM_INTERFACE>`. La ligne `Wake-on: g` doit être présente.
3.  **Adresse MAC** : notez l'adresse MAC du serveur (via la commande `ip link`).

### Côté machine d'administration (le pilote, ex : nœud Proxmox)
Installez l'utilitaire de réveil (en tant que root) :
```bash
apt update && apt install wakeonlan -y
```

---

## 2. Configuration de la sécurité (SSH et sudoers)

Pour que l'automate puisse agir sans intervention, il faut configurer l'accès et les permissions. **Attention à bien remplacer les éléments entre crochets.**

### Étape A : authentification par clé SSH
Générez une clé sur votre machine de contrôle (sans passphrase) et envoyez-la sur le serveur cible :
```bash
ssh-keygen -t ed25519
ssh-copy-id <VOTRE_UTILISATEUR>@<IP_DU_SERVEUR>
```

### Étape B : permission d'extinction sans mot de passe
Sur le serveur à piloter, l'utilisateur doit pouvoir éteindre la machine sans saisir de mot de passe.
1.  Éditez le fichier sudoers : `sudo visudo`
2.  Ajoutez cette ligne tout à la fin du fichier en remplaçant `<VOTRE_UTILISATEUR>` par le nom de votre compte (ex: `any` ou `admin`) :
```text
<VOTRE_UTILISATEUR> ALL=(ALL) NOPASSWD: /sbin/poweroff, /usr/sbin/poweroff
```

---

## 3. Mise en place du script intelligent

### Comment fonctionne ce script ?
Le script suit une logique de sécurité en 5 étapes :
1.  **Extinction** : il se connecte au serveur distant pour lui dire de s'éteindre proprement.
2.  **Vérification** : il teste le réseau (ping) jusqu'à ce que le serveur ne réponde plus.
3.  **Repos électrique** : il attend 90 secondes pour laisser les disques USB s'arrêter et les composants se décharger.
4.  **Relance** : il envoie un "paquet magique" (WOL) pour réveiller le serveur.
5.  **Confirmation** : il attend que le serveur revienne en ligne pour vous envoyer une notification de succès (ou une alerte en cas de problème).

```bash
#!/bin/bash
# Script de cycle électrique intelligent (BlablaLinux)

# --- CONFIGURATION À PERSONNALISER ---
TARGET_IP="<IP_DU_SERVEUR_CIBLE>"
TARGET_MAC="<MAC_DU_SERVEUR_CIBLE>"
TARGET_USER="<VOTRE_UTILISATEUR_SSH>"
USE_GOTIFY="true" 
GOTIFY_URL="https://<VOTRE_URL_GOTIFY>"
GOTIFY_TOKEN="<VOTRE_TOKEN_APPLICATION>"

# --- FONCTION DE NOTIFICATION ---
send_notification() {
    local title="$1"
    local message="$2"
    local priority="$3"
    
    if [ "$USE_GOTIFY" = "true" ]; then
        curl -k -s -X POST "$GOTIFY_URL/message?token=$GOTIFY_TOKEN" \
            -F "title=$title" -F "message=$message" -F "priority=$priority" > /dev/null 2>&1
    else
        echo "[$title] $message (Priorité: $priority)"
    fi
}

# 1. Extinction propre via SSH
echo "Envoi de l'ordre d'extinction à $TARGET_IP..."
ssh $TARGET_USER@$TARGET_IP "sudo /sbin/poweroff"

# 2. Attente de l'extinction réelle (boucle ping)
echo "Attente de l'arrêt du réseau..."
while ping -c 1 -W 1 $TARGET_IP > /dev/null; do
    sleep 5
done

# 3. Pause de sécurité électrique (90s)
echo "Serveur éteint. Pause électrique de 90s..."
sleep 90

# 4. Réveil via Wake-on-LAN
/usr/bin/wakeonlan $TARGET_MAC

# 5. Surveillance du redémarrage (timeout 10 min)
echo "Attente du redémarrage..."
MAX_RETRIES=60
COUNT=0
while ! ping -c 1 -W 1 $TARGET_IP > /dev/null; do
    if [ $COUNT -ge $MAX_RETRIES ]; then
        send_notification "🚨 ERREUR : $TARGET_IP" "Le serveur n'a pas redémarré après 10 minutes !" "10"
        exit 1
    fi
    sleep 10
    COUNT=$((COUNT + 1))
done

send_notification "✅ Succès : $TARGET_IP" "Le cycle électrique est terminé. Serveur en ligne." "5"
```

---

## 4. Test direct et vérification

Avant d'automatiser, lancez le script manuellement pour valider la configuration (SSH, Sudoers et Wake-on-LAN).

### Lancement manuel
Depuis votre machine pilote (ex: Proxmox), exécutez le script :
```bash
/bin/bash /root/scripts/cycle-nextcloud.sh
```

### Vérification des logs
Si vous avez configuré la redirection vers un fichier (voir étape 6), vous pouvez suivre l'avancement en temps réel :
```bash
tail -f /var/log/nc-cycle.log
```

---

## 5. Récapitulatif des personnalisations

Voici les points critiques à modifier pour que le système fonctionne chez vous :

| Élément | Où le modifier ? | Description |
| :--- | :--- | :--- |
| **Utilisateur** | Étape 2 (B) et Script | Le nom d'utilisateur qui possède la clé SSH. |
| **IP cible** | Étape 2 (A) et Script | L'adresse IP locale du serveur à éteindre. |
| **Adresse MAC** | Script (TARGET_MAC) | L'identifiant physique de la carte réseau. |
| **Gotify** | Script (URL/TOKEN) | Vos identifiants Gotify pour recevoir l'alerte sur smartphone. |

---

## 6. Automatisation avec crontab

Pour lancer ce cycle automatiquement (par exemple, tous les lundis à 04h00), ajoutez cette ligne au crontab de l'utilisateur **root** (`crontab -e`) de votre machine pilote :

```bash
0 4 * * 1 /bin/bash /root/scripts/cycle-nextcloud.sh >> /var/log/nc-cycle.log 2>&1
```

### Pourquoi cette méthode ?
Contrairement à une coupure brutale via une prise connectée, ce script assure une **extinction logicielle propre**. Cela protège l'intégrité de vos bases de données et de vos systèmes de fichiers tout en garantissant une réinitialisation matérielle complète.

---

> **Auteur** : ce guide est proposé par **Amaury aka BlablaLinux**. Retrouvez l'ensemble de mes services sur [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).
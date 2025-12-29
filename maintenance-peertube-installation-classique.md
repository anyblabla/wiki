---
title: Maintenance PeerTube (installation classique)
description: Maintenance de PeerTube en installation classique (bare-metal) : automatisation du nettoyage du stockage, des fichiers distants et optimisation RAM avec notifications Gotify.
published: true
date: 2025-12-29T12:24:30.442Z
tags: serveur, debian, script, gotify, administration système, maintenance, peertube, logiciel libre
editor: markdown
dateCreated: 2025-12-28T18:39:33.530Z
---

Bien que PeerTube gère une partie de sa rétention via l'interface d'administration, certaines opérations manuelles sont nécessaires pour supprimer les résidus de transcodage ou les fichiers temporaires qui finissent par saturer l'espace disque.

> 💡 **Pourquoi ce script ?**
> Sur une installation classique, il est crucial de lancer les commandes avec l'utilisateur `peertube` et de charger les bonnes variables d'environnement. Ce script automatise ces tâches répétitives pour maintenir votre instance propre sans risque d'erreur de permissions ou d'oubli de paramètres.

---

## 1. Prérequis : chemins par défaut

Ce guide considère que votre installation suit la structure standard de la documentation :

* **Dossier de l'instance :** `/var/www/peertube/peertube-latest`
* **Dossier de configuration :** `/var/www/peertube/config`
* **Utilisateur système :** `peertube`

---

## 2. Commandes de nettoyage manuel

Ces commandes doivent être lancées depuis le dossier `peertube-latest` pour fonctionner correctement.

| Action | Commande officielle |
| --- | --- |
| **Prune (fichiers orphelins)** | `sudo -u peertube NODE_CONFIG_DIR=/var/www/peertube/config NODE_ENV=production npm run prune-storage` |
| **Nettoyage fichiers distants** | `sudo -u peertube NODE_CONFIG_DIR=/var/www/peertube/config NODE_ENV=production npm run house-keeping -- --delete-remote-files` |
| **Régénérer les miniatures** | `sudo -u peertube NODE_CONFIG_DIR=/var/www/peertube/config NODE_ENV=production npm run regenerate-thumbnails` |

---

## 3. Automatisation par script (avec Gotify optionnel)

Ce script centralise les commandes de maintenance et peut vous informer via **Gotify**.

> 🔴 **Vous n'utilisez pas Gotify ?**
> Laissez simplement les variables `GOTIFY_URL` et `GOTIFY_TOKEN` vides. Le script détectera l'absence de configuration et ignorera l'envoi des notifications sans générer d'erreur.

### Contenu du script : `peertube-cleanup-classic.sh`

```bash
#!/bin/bash
# Script de maintenance PeerTube (installation classique)
# S'aligne sur les outils officiels (Server tools) de PeerTube >= 6.2
# Auteur : Amaury aka BlablaLinux

# --- PARAMÈTRES DE GOTIFY (Optionnel) ---
GOTIFY_URL=""
GOTIFY_TOKEN=""

# --- PARAMÈTRES DE MAINTENANCE ---
PT_DIR="/var/www/peertube/peertube-latest"
export NODE_CONFIG_DIR="/var/www/peertube/config"
export NODE_ENV="production"
LOGFILE="/var/log/peertube-cleanup-classic.log"
HOSTNAME=$(hostname)

# Redirection de toute la sortie vers le fichier journal
exec 1>>$LOGFILE 2>&1

# --- FONCTION DE NOTIFICATION GOTIFY ---
send_gotify_notification() {
    if [ -n "$GOTIFY_URL" ] && [ -n "$GOTIFY_TOKEN" ]; then
        local title="$1"
        local message="$2"
        local priority="$3"

        curl -k -s -X POST "$GOTIFY_URL/message?token=$GOTIFY_TOKEN" \
            -F "title=$title" \
            -F "message=$message" \
            -F "priority=$priority" > /dev/null 2>&1
    fi
}

echo "======================================================"
echo "Début de la maintenance PeerTube sur $HOSTNAME : $(date)"
echo "======================================================"

# On se déplace dans le dossier de l'instance
cd $PT_DIR || { 
    echo "Erreur : dossier PeerTube introuvable"; 
    send_gotify_notification "❌ PeerTube Classic Cleanup ÉCHEC" "Dossier $PT_DIR introuvable sur $HOSTNAME." 8
    exit 1; 
}

# 1. Nettoyage du stockage (vidéos transcodées inutilisées ou fichiers orphelins)
# Ref: https://docs.joinpeertube.org/maintain/tools#prune-filesystem-object-storage
echo "--- Étape 1 : Nettoyage du stockage (Prune) ---"
sudo -u peertube NODE_CONFIG_DIR=$NODE_CONFIG_DIR NODE_ENV=$NODE_ENV npm run prune-storage

# 2. Suppression des fichiers distants (vignettes, avatars d'autres instances)
# Ref: https://docs.joinpeertube.org/maintain/tools#cleanup-remote-files
echo "--- Étape 2 : Nettoyage des fichiers distants ---"
sudo -u peertube NODE_CONFIG_DIR=$NODE_CONFIG_DIR NODE_ENV=$NODE_ENV npm run house-keeping -- --delete-remote-files

# 3. Optimisation RAM : libérer le cache système
echo "--- Étape 3 : Libération du cache RAM ---"
sync; echo 3 > /proc/sys/vm/drop_caches

echo "======================================================"
echo "Maintenance terminée avec succès : $(date)"
echo "======================================================"

# Envoi de la notification de succès
send_gotify_notification "✅ PeerTube Classic Cleanup SUCCÈS" "La maintenance hebdomadaire sur $HOSTNAME s'est terminée correctement." 4

exit 0

```

---

## 4. Conseils supplémentaires

* **Permissions :** ne lancez jamais ces commandes directement en tant qu'utilisateur `root` sans spécifier `sudo -u peertube`. Cela pourrait corrompre les permissions de vos fichiers de données.
* **Version Docker :** pour ceux qui utilisent PeerTube via Docker, consultez la page dédiée : [Maintenance et nettoyage de PeerTube sous Docker](https://wiki.blablalinux.be/fr/maintenance-peertube-docker).
* **Mises à jour :** pensez à vérifier régulièrement la documentation officielle si vous effectuez des montées de version majeures de PeerTube.

[https://peertube.blablalinux.be/a/blablalinux/video-channels](https://peertube.blablalinux.be/a/blablalinux/video-channels)
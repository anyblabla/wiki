---
title: Maintenance PeerTube (installation classique)
description: Maintenance de PeerTube en installation classique (bare-metal) : automatisation du nettoyage du stockage, des fichiers distants et optimisation RAM avec notifications Gotify.
published: true
date: 2025-12-29T13:23:22.430Z
tags: serveur, debian, script, gotify, administration système, maintenance, peertube, logiciel libre
editor: markdown
dateCreated: 2025-12-28T18:39:33.530Z
---

Bien que PeerTube gère une partie de sa rétention via l'interface d'administration, certaines opérations manuelles sont nécessaires pour supprimer les résidus de transcodage ou les fichiers temporaires qui finissent par saturer l'espace disque.

> 💡 **Pourquoi ce script ?**
> Sur une installation classique, il est crucial de lancer les commandes avec l'utilisateur `peertube` et de charger les bonnes variables d'environnement. Ce script automatise ces tâches répétitives pour maintenir votre instance propre sans risque d'erreur de permissions ou d'oubli de paramètres.

---

## 1. Prérequis : chemins par défaut

Ce guide considère que votre installation suit la structure standard de la documentation officielle :

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

        curl -k -s -X POST "${GOTIFY_URL}/message?token=${GOTIFY_TOKEN}" \
            -F "title=${title}" \
            -F "message=${message}" \
            -F "priority=${priority}" > /dev/null 2>&1
    fi
}

echo "======================================================"
echo "Début de la maintenance PeerTube sur $HOSTNAME : $(date)"
echo "======================================================"

# On se déplace dans le dossier de l'instance
cd $PT_DIR || { 
    MSG="Erreur : dossier PeerTube introuvable."
    echo "$MSG"
    send_gotify_notification "❌ PeerTube Classic Cleanup ÉCHEC" "$MSG sur $HOSTNAME." 8
    exit 1
}

# 1. Nettoyage du stockage (vidéos transcodées inutilisées ou fichiers orphelins)
echo "--- Étape 1 : Nettoyage du stockage (Prune) ---"
sudo -u peertube NODE_CONFIG_DIR=$NODE_CONFIG_DIR NODE_ENV=$NODE_ENV npm run prune-storage
if [ $? -ne 0 ]; then
    send_gotify_notification "⚠️ PeerTube Classic ALERTE" "Échec partiel du nettoyage Prune sur $HOSTNAME." 5
fi

# 2. Suppression des fichiers distants (vignettes, avatars d'instances distantes)
echo "--- Étape 2 : Nettoyage des fichiers distants ---"
sudo -u peertube NODE_CONFIG_DIR=$NODE_CONFIG_DIR NODE_ENV=$NODE_ENV npm run house-keeping -- --delete-remote-files
if [ $? -ne 0 ]; then
    send_gotify_notification "⚠️ PeerTube Classic ALERTE" "Échec du nettoyage des fichiers distants sur $HOSTNAME." 5
fi

# 3. Optimisation RAM (Protection LXC)
echo "--- Étape 3 : Libération du cache RAM ---"
sync
if [ -w /proc/sys/vm/drop_caches ]; then
    echo 3 > /proc/sys/vm/drop_caches
    echo "Cache RAM libéré avec succès."
else
    echo "Note : Droits insuffisants pour drop_caches (LXC), ignoré."
fi

echo "======================================================"
echo "Maintenance terminée : $(date)"
echo "======================================================"

# 4. Envoi de la notification de succès final
send_gotify_notification "✅ PeerTube Classic Cleanup TERMINÉ" "La maintenance PeerTube sur $HOSTNAME est terminée avec succès." 4

exit 0

```

---

## 4. Conseils supplémentaires

1. **Permissions :** Ne lancez jamais ces commandes directement en tant qu'utilisateur `root` sans spécifier `sudo -u peertube`. Cela pourrait corrompre les permissions de vos fichiers de données.
2. **Sécurité du script :** Appliquez des permissions restrictives pour protéger vos tokens Gotify :
`chmod 700 /root/scripts/peertube-cleanup-classic.sh`
3. **Planification (Crontab) :** `00 04 * * 0 /bin/bash /root/scripts/peertube-cleanup-classic.sh >> /var/log/peertube-cleanup-classic.log 2>&1`

[https://peertube.blablalinux.be/a/blablalinux/video-channels](https://peertube.blablalinux.be/a/blablalinux/video-channels)
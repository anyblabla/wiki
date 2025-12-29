---
title: Maintenance et nettoyage de PeerTube sous Docker
description: Maintenance de PeerTube sous Docker : automatisation du nettoyage du stockage, des fichiers distants et optimisation RAM avec notifications Gotify optionnelles.
published: true
date: 2025-12-29T12:22:08.258Z
tags: docker, lxc, proxmox, gotify, linux, maintenance, peertube
editor: markdown
dateCreated: 2025-12-26T16:55:44.444Z
---

Bien que PeerTube gère une partie de sa rétention via l'interface d'administration, certaines opérations manuelles sont nécessaires pour supprimer les résidus de transcodage ou les fichiers temporaires qui finissent par saturer l'espace disque.

> 💡
> **Pourquoi ce script ?**
> PeerTube possède son propre système de rétention, mais il fournit également des outils de maintenance officiels (accessibles via `npm run`) pour les tâches lourdes ou spécifiques. Mon script ne remplace pas le code des développeurs : il automatise simplement le lancement de ces outils internes à des heures creuses. C'est un complément d'administration pour garder une machine propre et réactive sans intervention manuelle.

---

## 1. Identifier votre conteneur PeerTube

Dans la configuration Docker par défaut, PeerTube n'a pas de nom fixe. Il est nommé selon votre dossier (souvent `peertube-peertube-1`).

Pour connaître le nom exact sur votre système, lancez :

```bash
docker ps --format "{{.Names}}" | grep peertube

```

> ⚠️ Dans la suite de ce guide et dans le script, j'utiliserai le nom **`peertube-peertube-1`**. Si vous avez personnalisé votre fichier `docker-compose.yml` avec un `container_name: peertube`, pensez à adapter le nom dans les commandes.

---

## 2. Commandes de nettoyage manuel

Voici les commandes officielles pour un nettoyage ponctuel. Elles s'exécutent via `docker exec`.

| Action | Commande officielle |
| --- | --- |
| **Prune (fichiers orphelins)** | `docker exec -u peertube peertube-peertube-1 npm run prune-storage` |
| **Nettoyage fichiers distants** | `docker exec -u peertube peertube-peertube-1 npm run house-keeping -- --delete-remote-files` |
| **Régénérer les miniatures** | `docker exec -u peertube peertube-peertube-1 npm run regenerate-thumbnails` |

---

## 3. Automatisation par script (avec Gotify optionnel)

Ce script automatise les tâches de nettoyage recommandées et peut vous envoyer une notification via **Gotify**.

> 🔴
> **Vous n'utilisez pas Gotify ?**
> Laissez simplement les variables `GOTIFY_URL` et `GOTIFY_TOKEN` vides. Le script détectera l'absence de configuration et ignorera l'envoi des notifications sans générer d'erreur.

### Contenu du script : `peertube-cleanup.sh`

```bash
#!/bin/bash
# Script de maintenance PeerTube pour Docker
# S'aligne sur les outils officiels (Server tools) de PeerTube >= 6.2
# Auteur : Amaury aka BlablaLinux

# --- PARAMÈTRES DE GOTIFY (Optionnel) ---
GOTIFY_URL=""
GOTIFY_TOKEN=""

# --- PARAMÈTRES DE MAINTENANCE ---
CONTAINER_NAME="peertube-peertube-1"
LOGFILE="/var/log/peertube-cleanup.log"
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

# 1. Vérification de la présence du conteneur
if [ ! "$(docker ps -q -f name=$CONTAINER_NAME)" ]; then
    echo "Erreur : Le conteneur $CONTAINER_NAME est introuvable."
    send_gotify_notification "❌ PeerTube Cleanup ÉCHEC" "Le conteneur $CONTAINER_NAME est introuvable sur $HOSTNAME." 8
    exit 1
fi

# 2. Nettoyage du stockage (vidéos transcodées inutilisées ou fichiers orphelins)
# Ref: https://docs.joinpeertube.org/maintain/tools#prune-filesystem-object-storage
echo "--- Étape 1 : Nettoyage du stockage (Prune) ---"
docker exec -u peertube $CONTAINER_NAME npm run prune-storage

# 3. Suppression des fichiers distants (vignettes, avatars d'autres instances)
# Ref: https://docs.joinpeertube.org/maintain/tools#cleanup-remote-files
echo "--- Étape 2 : Nettoyage des fichiers distants ---"
docker exec -u peertube $CONTAINER_NAME npm run house-keeping -- --delete-remote-files

# 4. Optimisation RAM : libérer le cache système
echo "--- Étape 3 : Libération du cache RAM ---"
sync; echo 3 > /proc/sys/vm/drop_caches

echo "======================================================"
echo "Maintenance terminée avec succès : $(date)"
echo "======================================================"

# Envoi de la notification de succès
send_gotify_notification "✅ PeerTube Cleanup SUCCÈS" "La maintenance hebdomadaire sur $HOSTNAME s'est terminée correctement." 4

exit 0

```

---

## 4. Conseils supplémentaires

* **Stockage :** si vous fédérez beaucoup d'instances, surveillez votre dossier `./docker-volume/data`.
* **Redondance :** pensez à limiter l'espace alloué aux vidéos des autres instances dans l'interface d'administration de PeerTube (Configuration > VOD > Redondance).
* **Installation classique :** pour ceux qui n'utilisent pas Docker, consultez la page dédiée : [Maintenance PeerTube (installation classique)](https://wiki.blablalinux.be/fr/maintenance-peertube-installation-classique).

[https://peertube.blablalinux.be/a/blablalinux/video-channels](https://peertube.blablalinux.be/a/blablalinux/video-channels)
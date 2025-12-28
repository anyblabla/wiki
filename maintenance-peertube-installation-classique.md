---
title: Maintenance PeerTube (installation classique)
description: Apprenez à maintenir votre instance PeerTube en installation classique (bare-metal) : nettoyage automatique du stockage, des fichiers distants et optimisation système pour Debian et Ubuntu.
published: true
date: 2025-12-28T18:46:07.345Z
tags: serveur, debian, script, administration système, maintenance, peertube, logiciel libre
editor: markdown
dateCreated: 2025-12-28T18:39:33.530Z
---

Bien que PeerTube gère une partie de sa rétention via l'interface d'administration, certaines opérations manuelles sont nécessaires pour supprimer les résidus de transcodage ou les fichiers temporaires qui finissent par saturer l'espace disque.

> [!TIP]
> **Pourquoi ce script ?**
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

## 3. Automatisation par script

Nous allons créer un script qui centralise ces commandes de maintenance recommandées par les développeurs.

### Étape A : créer le fichier

```bash
mkdir -p /root/scripts
nano /root/scripts/peertube-cleanup-classic.sh

```

### Étape B : contenu du script

```bash
#!/bin/bash
# Script de maintenance PeerTube (installation classique)
# S'aligne sur les outils officiels (Server tools) de PeerTube >= 6.2
# Auteur : Amaury Libert (Blabla Linux)

# Configuration des chemins (à adapter selon votre installation)
PT_DIR="/var/www/peertube/peertube-latest"
export NODE_CONFIG_DIR="/var/www/peertube/config"
export NODE_ENV="production"

echo "--- Déun de la maintenance PeerTube : $(date) ---"

# On se déplace dans le dossier de l'instance
cd $PT_DIR || { echo "Erreur : dossier PeerTube introuvable"; exit 1; }

# 1. Nettoyage du stockage (vidéos transcodées inutilisées ou fichiers orphelins)
# Ref: https://docs.joinpeertube.org/maintain/tools#prune-filesystem-object-storage
sudo -u peertube NODE_CONFIG_DIR=$NODE_CONFIG_DIR NODE_ENV=$NODE_ENV npm run prune-storage

# 2. Suppression des fichiers distants (vignettes, avatars d'autres instances)
# Ref: https://docs.joinpeertube.org/maintain/tools#cleanup-remote-files
sudo -u peertube NODE_CONFIG_DIR=$NODE_CONFIG_DIR NODE_ENV=$NODE_ENV npm run house-keeping -- --delete-remote-files

# 3. Optimisation RAM : libérer le cache système
sync; echo 3 > /proc/sys/vm/drop_caches

echo "--- Maintenance terminée : $(date) ---"

```

### Étape C : planification

1. Rendre le script exécutable : `chmod 700 /root/scripts/peertube-cleanup-classic.sh`
2. Ouvrir la crontab : `crontab -e`
3. Ajouter l'exécution automatique (par exemple, le dimanche à 3h30) :

```cron
30 03 * * 0 /bin/bash /root/scripts/peertube-cleanup-classic.sh >> /var/log/peertube-cleanup.log 2>&1

```

---

## 4. Conseils supplémentaires

* **Permissions :** ne lancez jamais ces commandes directement en tant qu'utilisateur `root` sans spécifier `sudo -u peertube`. Cela pourrait corrompre les permissions de vos fichiers de données.
* **Version Docker :** pour ceux qui utilisent PeerTube via Docker, consultez la page dédiée : [Maintenance et nettoyage de PeerTube sous Docker](https://wiki.blablalinux.be/fr/maintenance-peertube-docker).
* **Mises à jour :** pensez à vérifier régulièrement la documentation officielle si vous effectuez des montées de version majeures de PeerTube.

[https://peertube.blablalinux.be/a/blablalinux/video-channels](https://peertube.blablalinux.be/a/blablalinux/video-channels)
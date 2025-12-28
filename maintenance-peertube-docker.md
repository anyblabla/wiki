---
title: Maintenance et nettoyage de PeerTube sous Docker
description: Comment libérer de l'espace disque sur votre instance PeerTube Docker : nettoyage des fichiers temporaires, des transcodages échoués et des caches.
published: true
date: 2025-12-28T18:40:13.515Z
tags: docker, lxc, proxmox, linux, maintenance, peertube
editor: markdown
dateCreated: 2025-12-26T16:55:44.444Z
---

Bien que PeerTube gère une partie de sa rétention via l'interface d'administration, certaines opérations manuelles sont nécessaires pour supprimer les résidus de transcodage ou les fichiers temporaires qui finissent par saturer l'espace disque.

> [!TIP]
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

## 2. Les commandes de nettoyage manuel

Voici les commandes officielles pour un nettoyage ponctuel. Elles s'exécutent via `docker exec`.

| Action | Commande officielle |
| --- | --- |
| **Prune (Fichiers orphelins)** | `docker exec -u peertube peertube-peertube-1 npm run prune-storage` |
| **Nettoyage fichiers distants** | `docker exec -u peertube peertube-peertube-1 npm run house-keeping -- --delete-remote-files` |
| **Régénérer les miniatures** | `docker exec -u peertube peertube-peertube-1 npm run regenerate-thumbnails` |

---

## 3. Automatisation par script

Pour ne plus y penser, nous allons créer un script qui nettoie les fichiers et libère la RAM du système.

### Étape A : Créer le fichier

```bash
mkdir -p /root/scripts
nano /root/scripts/peertube-cleanup.sh

```

### Étape B : Contenu du script

Copiez ce code. **Note :** Si votre conteneur ne s'appelle pas `peertube-peertube-1`, modifiez la deuxième ligne.

```bash
#!/bin/bash
# Script de maintenance PeerTube pour Docker
# S'aligne sur les outils officiels (Server Tools) de PeerTube >= 6.2
# Auteur : Amaury Libert (Blabla Linux)

CONTAINER_NAME="peertube-peertube-1"

echo "--- Début de la maintenance PeerTube : $(date) ---"

# 1. Nettoyage du stockage (vidéos transcodées inutilisées ou fichiers orphelins)
# Ref: https://docs.joinpeertube.org/maintain/tools#prune-filesystem-object-storage
docker exec -u peertube $CONTAINER_NAME npm run prune-storage

# 2. Suppression des fichiers distants (vignettes, avatars d'autres instances)
# Ref: https://docs.joinpeertube.org/maintain/tools#cleanup-remote-files
docker exec -u peertube $CONTAINER_NAME npm run house-keeping -- --delete-remote-files

# 3. Optimisation RAM : Libérer le cache système
sync; echo 3 > /proc/sys/vm/drop_caches

echo "--- Maintenance terminée : $(date) ---"

```

### Étape C : Planification

1. Rendre le script exécutable : `chmod 700 /root/scripts/peertube-cleanup.sh`
2. Ouvrir la crontab : `crontab -e`
3. Ajouter la ligne suivante (exécution le dimanche à 3h30) :

```cron
30 03 * * 0 /bin/bash /root/scripts/peertube-cleanup.sh >> /var/log/peertube-cleanup.log 2>&1

```

---

## 4. Conseils supplémentaires

* **Stockage :** Si vous fédérez beaucoup d'instances, surveillez votre dossier `./docker-volume/data`.
* **Redondance :** Pensez à limiter l'espace alloué aux vidéos des autres instances dans l'interface d'administration de PeerTube (Configuration > VOD > Redondance).
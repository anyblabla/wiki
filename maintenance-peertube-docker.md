---
title: Maintenance et nettoyage de PeerTube sous Docker
description: Comment libérer de l'espace disque sur votre instance PeerTube Docker : nettoyage des fichiers temporaires, des transcodages échoués et des caches.
published: false
date: 2025-12-26T16:55:44.444Z
tags: docker, lxc, proxmox, linux, maintenance, peertube
editor: markdown
dateCreated: 2025-12-26T16:55:44.444Z
---

Bien que PeerTube gère une partie de sa rétention via l'interface d'administration, certaines opérations de maintenance manuelle sont nécessaires pour supprimer les fichiers "fantômes" (résidus de transcodage ou téléchargements interrompus) qui finissent par saturer l'espace disque de votre conteneur ou de votre LXC.

---

## 1. Identifier le nom de votre conteneur

Avant de lancer les commandes ou d'installer le script, vous devez connaître le nom exact de votre conteneur PeerTube. Dans ma configuration, il s'appelle `peertube`, mais selon votre installation, il peut s'appeler `peertube_peertube_1`.

Pour le vérifier, lancez :

```bash
docker ps --format "{{.Names}}" | grep peertube

```

---

## 2. Les commandes de nettoyage manuel

Si vous souhaitez effectuer un nettoyage ponctuel, voici les commandes à exécuter (en remplaçant `peertube` par le nom de votre conteneur si nécessaire) :

| Type de nettoyage | Commande (via docker exec) |
| --- | --- |
| **Vidéos temporaires** | `docker exec -u peertube peertube npm run command -- clean-tmp-videos` |
| **Miniatures (previews)** | `docker exec -u peertube peertube npm run command -- clean-previews` |
| **Fichiers orphelins** | `docker exec -u peertube peertube npm run command -- check-storage --delete-unreferenced` |

---

## 3. Ma méthode d'automatisation (pas à pas)

Pour garantir une instance toujours propre, j'utilise un script qui automatise ces purges et optimise la RAM du système.

### Étape A : Créer le dossier pour vos scripts

Sur un LXC où vous êtes root par défaut :

```bash
mkdir -p /root/scripts

```

### Étape B : Créer le fichier du script

```bash
nano /root/scripts/peertube-cleanup.sh

```

### Étape C : Contenu du script (avec détection automatique)

Ce script détecte tout seul le nom de votre conteneur et libère le cache RAM à la fin.

```bash
#!/bin/bash
# Script de maintenance PeerTube pour Docker
# Auteur : Amaury Libert (Blabla Linux)

# Détection automatique du nom du conteneur PeerTube
CONTAINER_NAME=$(docker ps --format "{{.Names}}" | grep -m 1 "peertube")

if [ -z "$CONTAINER_NAME" ]; then
    echo "Erreur : Conteneur PeerTube introuvable."
    exit 1
fi

echo "--- Début de la maintenance PeerTube ($CONTAINER_NAME) : $(date) ---"

# 1. Nettoyage des vidéos temporaires (résidus de transcodages plantés)
docker exec -u peertube $CONTAINER_NAME npm run command -- clean-tmp-videos

# 2. Nettoyage des miniatures distantes (previews)
docker exec -u peertube $CONTAINER_NAME npm run command -- clean-previews

# 3. Optimisation RAM : Libérer le cache système
sync; echo 3 > /proc/sys/vm/drop_caches

echo "--- Maintenance terminée : $(date) ---"

```

### Étape D : Permissions et planification

```bash
chmod 700 /root/scripts/peertube-cleanup.sh
crontab -e

```

Ajoutez cette ligne pour une exécution le dimanche à 3h30 (pour ne pas interférer avec le nettoyage de Mastodon à 3h00) :

```cron
30 03 * * 0 /bin/bash /root/scripts/peertube-cleanup.sh >> /var/log/peertube-cleanup.log 2>&1

```

---

## 4. Recommandation : La redondance

N'oubliez pas que PeerTube peut aussi remplir votre disque avec les vidéos des autres instances si la **Redondance** est activée. Pensez à limiter cet espace dans :
**Administration** > **Configuration** > **VOD** > **Redondance**.

Fixer une limite (ex: 50 Go ou 100 Go) permet d'éviter que votre LXC ne sature de manière imprévue.

[https://mastodon.blablalinux.be/@blablalinux](https://mastodon.blablalinux.be/@blablalinux)
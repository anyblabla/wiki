---
title: Maintenance et nettoyage de Mastodon sous Docker
description: Guide complet pour automatiser le nettoyage de Mastodon sous Docker. Tutoriel pas à pas : commandes tootctl, script de maintenance, cron et optimisation de la RAM.
published: true
date: 2025-12-26T17:15:20.767Z
tags: mastodon, docker, lxc, proxmox, cron, crontab, script, bash, pve, maintenance, automatisation
editor: markdown
dateCreated: 2025-12-25T13:00:52.896Z
---

> ⚠️ Ce guide est spécifiquement conçu pour une installation de Mastodon tournant sous **Docker** (souvent via un conteneur LXC sur Proxmox). Si vous utilisez une installation "classique" (non Docker) sur base Debian, je vous invite à consulter mon autre page dédiée : [Maintenance Mastodon (Installation classique)](https://wiki.blablalinux.be/fr/mastodon-cache).

Dans cet article, je partage ma méthode pour entretenir mon instance Mastodon lorsqu'elle fonctionne sous Docker. Le nettoyage régulier est indispensable pour éviter que le cache des médias et la base de données ne saturent l'espace disque de mon système.

---

## 1. Pourquoi utiliser un script plutôt que les réglages natifs ?

Il existe des options de nettoyage directement dans l'interface d'administration sous :
**Préférences** > **Administration** > **Paramètres du serveur** > **Rétention du contenu**.

Cependant, je privilégie l'automatisation externe pour plusieurs raisons :

* **La performance :** Mon script utilise l'option `--concurrency`, permettant d'utiliser plusieurs cœurs du processeur.
* **Le nettoyage complet :** L'interface ne permet pas de nettoyer les **cartes de prévisualisation** (miniatures de liens) ni de purger les **comptes distants inactifs**.
* **Le contrôle :** Je maîtrise exactement ce qui est supprimé et à quel moment (heures creuses).

---

## 2. Ma recommandation : la stratégie de nettoyage hybride

Faut-il désactiver les réglages de l'interface si on utilise un script ? Ma réponse est **non**. Je conseille une approche hybride pour une sécurité maximale :

1. **Le script (Prioritaire) :** Il effectue un nettoyage en profondeur et rapide tous les dimanches (rétention de 7 jours).
2. **L'interface (Sécurité) :** Je règle la rétention des médias sur **14 ou 30 jours** dans l'interface. Ainsi, si mon script échoue pour une raison technique, Mastodon dispose d'un filet de sécurité automatique.
3. **La zone de danger :** Je laisse la "Durée de rétention du contenu distant" sur **0 (désactivé)**. Je laisse mon script gérer la purge des messages via `statuses remove`, car c'est beaucoup plus respectueux pour les utilisateurs.

---

## 3. Les commandes de nettoyage (tootctl)

Voici les commandes que j'utilise. Sous Docker, elles s'exécutent via l'outil `docker exec`.

| Nettoyage | Commande (via docker exec) | Impact |
| --- | --- | --- |
| **Vignettes** | `bin/tootctl cache clear` | Supprime les petites images de prévisualisation. |
| **Vieux médias** | `bin/tootctl media remove` | Supprime les images et vidéos distantes en cache. |
| **Médias orphelins** | `bin/tootctl media remove-orphans` | Nettoie les fichiers qui ne sont plus liés à rien. |
| **Comptes inactifs** | `bin/tootctl accounts prune` | Supprime les profils distants sans interaction. |
| **Vieux statuts** | `bin/tootctl statuses remove` | Supprime les messages distants obsolètes. |

---

## 4. Exécution manuelle sous Docker

Idéal pour un nettoyage ponctuel. J'envoie directement la commande au conteneur web.

1. **Identifier le conteneur :** Dans la configuration officielle, le conteneur se nomme souvent **`mastodon-web-1`**. Pour vérifier le vôtre :

```bash
docker ps --format "{{.Names}}" | grep web

```

2. **Lancer le nettoyage (Exemples complets) :** J'utilise toujours l'utilisateur `mastodon`.

**Nettoyer les vignettes en cache :**

```bash
docker exec -u mastodon mastodon-web-1 bin/tootctl cache clear

```

**Supprimer les médias distants de plus de 7 jours :**

```bash
docker exec -u mastodon mastodon-web-1 bin/tootctl media remove --days 7

```

**Supprimer les fichiers médias orphelins :**

```bash
docker exec -u mastodon mastodon-web-1 bin/tootctl media remove-orphans

```

**Purger les comptes distants inactifs :**

```bash
docker exec -u mastodon mastodon-web-1 bin/tootctl accounts prune

```

**Supprimer les anciens statuts (messages) distants :**

```bash
docker exec -u mastodon mastodon-web-1 bin/tootctl statuses remove

```

---

## 5. Ma méthode d'automatisation (pas à pas)

### Étape A : Créer le dossier pour vos scripts

Sur un LXC où vous êtes root par défaut :

```bash
mkdir -p /root/scripts

```

### Étape B : Créer le fichier du script

```bash
nano /root/scripts/mastodon-cleanup.sh

```

### Étape C : Contenu du script (avec optimisation RAM)

Ce script inclut désormais une commande pour libérer le cache mémoire après le nettoyage intensif de la base de données. **Note :** Adaptez `CONTAINER_NAME` si votre conteneur porte un nom différent.

```bash
#!/bin/bash
# Script de maintenance Mastodon pour Docker
# Auteur : Amaury Libert (Blabla Linux)

CONTAINER_NAME="mastodon-web-1"
DAYS_MEDIA=7
THREADS=4 

echo "--- Début de la maintenance : $(date) ---"

# Vérification de la présence du conteneur
if [ ! "$(docker ps -q -f name=$CONTAINER_NAME)" ]; then
    echo "Erreur : Le conteneur $CONTAINER_NAME est introuvable."
    exit 1
fi

# Nettoyage des médias et miniatures de liens
docker exec -u mastodon $CONTAINER_NAME bin/tootctl media remove --days=$DAYS_MEDIA --concurrency=$THREADS
docker exec -u mastodon $CONTAINER_NAME bin/tootctl preview_cards remove --days=$DAYS_MEDIA

# Nettoyage des anciens statuts et comptes inactifs
docker exec -u mastodon $CONTAINER_NAME bin/tootctl statuses remove --days=30
docker exec -u mastodon $CONTAINER_NAME bin/tootctl accounts prune

# Optimisation RAM : Libérer le cache système (PageCache, dentries et inodes)
sync; echo 3 > /proc/sys/vm/drop_caches

echo "--- Maintenance terminée : $(date) ---"

```

### Étape D : Rendre le script exécutable (sécurisé)

```bash
chmod 700 /root/scripts/mastodon-cleanup.sh

```

### Étape E : Planification avec Crontab

```bash
crontab -e

```

Ajoutez cette ligne tout en bas (exécution le dimanche à 3h00) :

```cron
00 03 * * 0 /bin/bash /root/scripts/mastodon-cleanup.sh >> /var/log/mastodon-cleanup.log 2>&1

```

---

## 6. Cas particulier d'Elasticsearch

Si vous utilisez le moteur de recherche **Elasticsearch**, lancez ponctuellement cette commande pour réindexer le contenu :

```bash
docker exec -u mastodon mastodon-web-1 bin/tootctl search deploy

```

---

## 7. Mes recommandations finales

**Libération de la RAM :** La commande `drop_caches` ajoutée au script permet de récupérer la mémoire vive "grignotée" par le cache de la base de données pendant le nettoyage. C'est une sécurité indispensable pour les instances tournant avec peu de RAM.

**Note sur le SWAP :** Si vous êtes sur Proxmox, ne videz pas le SWAP via le script (commande `swapoff`). Gérez plutôt l'allocation du SWAP directement dans les ressources de votre LXC (1 ou 2 Go suffisent largement pour 4 Go de RAM).

[https://mastodon.blablalinux.be/@blablalinux](https://mastodon.blablalinux.be/@blablalinux)
---
title: Maintenance et nettoyage de Mastodon sous Docker
description: Apprenez à automatiser le nettoyage de votre instance Mastodon sous Docker. Script de maintenance, configuration Cron et commandes tootctl pour optimiser l'espace disque de votre serveur.
published: false
date: 2025-12-25T13:04:37.877Z
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

## 2. Les commandes de nettoyage (tootctl)

Voici les commandes que j'utilise. Sous Docker, elles s'exécutent via l'outil `docker exec`.

| Nettoyage | Commande (via docker exec) | Impact |
| --- | --- | --- |
| **Vignettes** | `bin/tootctl cache clear` | Supprime les petites images de prévisualisation. |
| **Vieux médias** | `bin/tootctl media remove` | Supprime les images et vidéos distantes en cache. |
| **Médias orphelins** | `bin/tootctl media remove-orphans` | Nettoie les fichiers qui ne sont plus liés à rien. |
| **Comptes inactifs** | `bin/tootctl accounts prune` | Supprime les profils distants sans interaction. |
| **Vieux statuts** | `bin/tootctl statuses remove` | Supprime les messages distants obsolètes. |

---

## 3. Exécution manuelle sous Docker

Idéal pour un nettoyage ponctuel. J'envoie directement la commande au conteneur web.

1. **Identifier le conteneur :** Pour connaître le nom exact de mon conteneur :
```bash
docker ps --format "{{.Names}}"

```


2. **Lancer le nettoyage (Exemples complets) :** J'utilise toujours l'utilisateur `mastodon`.

**Nettoyer les vignettes en cache :**

```bash
docker exec -u mastodon mastodon-web bin/tootctl cache clear

```

**Supprimer les médias distants de plus de 7 jours :**

```bash
docker exec -u mastodon mastodon-web bin/tootctl media remove --days 7

```

**Supprimer les fichiers médias orphelins :**

```bash
docker exec -u mastodon mastodon-web bin/tootctl media remove-orphans

```

**Purger les comptes distants inactifs :**

```bash
docker exec -u mastodon mastodon-web bin/tootctl accounts prune

```

**Supprimer les anciens statuts (messages) distants :**

```bash
docker exec -u mastodon mastodon-web bin/tootctl statuses remove

```

---

## 4. Ma méthode d'automatisation (pas à pas)

### Étape A : Créer le dossier pour vos scripts

```bash
mkdir -p /home/scripts

```

### Étape B : Créer le fichier du script

```bash
nano /home/scripts/mastodon-cleanup.sh

```

### Étape C : Copier le contenu du script

```bash
#!/bin/bash
# Script de maintenance Mastodon pour Docker
# Auteur : Amaury Libert (Blabla Linux)

CONTAINER_NAME="mastodon-web"
DAYS_MEDIA=7
THREADS=4 

echo "--- Début de la maintenance : $(date) ---"

# Nettoyage des médias et miniatures de liens
docker exec -u mastodon $CONTAINER_NAME bin/tootctl media remove --days=$DAYS_MEDIA --concurrency=$THREADS
docker exec -u mastodon $CONTAINER_NAME bin/tootctl preview_cards remove --days=$DAYS_MEDIA

# Nettoyage des anciens statuts et comptes inactifs
docker exec -u mastodon $CONTAINER_NAME bin/tootctl statuses remove --days=30
docker exec -u mastodon $CONTAINER_NAME bin/tootctl accounts prune

echo "--- Maintenance terminée : $(date) ---"

```

### Étape D : Rendre le script exécutable

```bash
chmod +x /home/scripts/mastodon-cleanup.sh

```

### Étape E : Planification avec Crontab

Pour un lancement automatique chaque dimanche à 3h00 du matin :

```bash
crontab -e

```

Ajoutez cette ligne tout en bas :

```cron
00 03 * * 0 /bin/bash /home/scripts/mastodon-cleanup.sh >> /var/log/mastodon-cleanup.log 2>&1

```

---

## 5. Cas particulier d'Elasticsearch

Si vous utilisez le moteur de recherche **Elasticsearch**, lancez ponctuellement cette commande pour réindexer le contenu :

```bash
docker exec -u mastodon mastodon-web bin/tootctl search deploy

```

---

## 6. Mes recommandations finales

**Fréquence de maintenance :** Si vous effectuez les nettoyages manuellement, je vous recommande d'exécuter `media remove` et `accounts prune` au moins une fois par semaine ou par mois.

**Impact de l'automatisation :** Avec le script et la tâche Cron que je vous propose, ce nettoyage devient **hebdomadaire**. C'est le compromis idéal pour maintenir un espace disque stable sur une petite instance sans solliciter inutilement les ressources CPU tous les jours.

**Horaires :** Je planifie toujours l'exécution à 3h du matin. Les commandes peuvent être gourmandes en ressources ; il est donc préférable de les lancer durant les heures creuses de votre instance.

[https://mastodon.blablalinux.be/@blablalinux](https://mastodon.blablalinux.be/@blablalinux)
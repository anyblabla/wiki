---
title: Maintenance Mastodon - Nettoyage et optimisation des caches avec tootctl
description: Mastodon accumule divers types de données et de caches au fil du temps (images, médias, comptes distants, etc.). L'utilisation régulière des commandes tootctl est essentielle pour libérer de l'espace disque et maintenir la performance de votre instance.
published: true
date: 2025-12-26T16:33:03.350Z
tags: mastodon, cache, delete
editor: markdown
dateCreated: 2024-05-06T22:29:10.684Z
---

> ⚠️ Ce guide concerne une installation "classique" (non Docker) de Mastodon sur base Debian. Si vous utilisez une installation tournant sous **Docker**, je vous invite à consulter cette page dédiée : [Maintenance Mastodon sous Docker](https://wiki.blablalinux.be/fr/maintenance-mastodon-docker).

Mastodon accumule divers types de données et de caches au fil du temps (images, médias, comptes distants, etc.). L'utilisation régulière des commandes `tootctl` est essentielle pour libérer de l'espace disque et maintenir la performance de votre instance.

---

## I. Les principaux niveaux de nettoyage

Voici les commandes de base pour vider les différents niveaux de cache et supprimer les données obsolètes :

| Nettoyage | Commande `tootctl` | Description | Poids / Impact |
| --- | --- | --- | --- |
| **Vignettes en cache** | `tootctl cache clear` | Supprime les petites vignettes mises en cache (provenant d'autres serveurs). | Léger |
| **Vieux médias** | `tootctl media remove` | Supprime les médias (images, vidéos) plus anciens que la période de rétention configurée. | Lourd |
| **Médias non référencés** | `tootctl media remove-orphans` | Supprime les fichiers médias qui ne sont plus liés à aucun statut ou profil. | Variable |
| **Comptes distants obsolètes** | `tootctl accounts cull` | Supprime les comptes distants qui n'existent plus ou qui n'ont pas été actifs/utilisés depuis longtemps. | Lourd |
| **Statuts obsolètes** | `tootctl statuses remove` | Supprime les statuts (toots) qui ont expiré ou qui n'existent plus sur la fédération. | Très lourd |

* Les principaux niveaux de nettoyage :

1. Vignettes en cache : **tootctl cache clear**
2. Vieux média : **tootctl media remove**
3. Médias non référencés : **tootctl media remove-orphans**
4. Comptes distants qui n’existent plus (lourd) : **tootctl accounts cull**
5. Statuts qui n’existent plus (très lourd) : **tootctl statuses remove**

---

## II. Exécution des commandes de nettoyage

Toutes les commandes de nettoyage `tootctl` doivent être exécutées en tant qu'utilisateur `mastodon` depuis le répertoire `live` de votre instance.

1. **Passer en mode super-utilisateur (root) :**

```bash
su -

```

2. **Se déplacer dans le répertoire de l'instance Mastodon :**

```bash
cd /var/www/mastodon/live

```

3. **Exécuter chaque commande de nettoyage :**
On utilise `sudo -u mastodon` avec les variables d'environnement nécessaires.

```bash
sudo -u mastodon RAILS_ENV=production PATH=/opt/rbenv/versions/mastodon/bin bin/tootctl cache clear
sudo -u mastodon RAILS_ENV=production PATH=/opt/rbenv/versions/mastodon/bin bin/tootctl media remove
sudo -u mastodon RAILS_ENV=production PATH=/opt/rbenv/versions/mastodon/bin bin/tootctl media remove-orphans
sudo -u mastodon RAILS_ENV=production PATH=/opt/rbenv/versions/mastodon/bin bin/tootctl accounts cull 
sudo -u mastodon RAILS_ENV=production PATH=/opt/rbenv/versions/mastodon/bin bin/tootctl statuses remove

```

---

## III. Ma méthode d'automatisation (pas à pas)

### Étape A : Créer le dossier et le script

```bash
mkdir -p /root/scripts
nano /root/scripts/mastodon-cleanup.sh

```

### Étape B : Contenu du script (avec optimisation RAM/SWAP)

Ce script inclut désormais la libération du cache RAM et la purge du SWAP pour garantir que le système retrouve toute sa réactivité après le nettoyage.

```bash
#!/bin/bash
# Script de maintenance Mastodon (Installation Classique)
# Auteur : Amaury Libert (Blabla Linux)

MASTODON_DIR="/var/www/mastodon/live"
RBENV_PATH="/opt/rbenv/versions/mastodon/bin"
DAYS_MEDIA=7

echo "--- Début de la maintenance : $(date) ---"

cd $MASTODON_DIR

# Nettoyage des médias et vignettes
sudo -u mastodon RAILS_ENV=production PATH=$RBENV_PATH bin/tootctl media remove --days=$DAYS_MEDIA
sudo -u mastodon RAILS_ENV=production PATH=$RBENV_PATH bin/tootctl cache clear

# Nettoyage des comptes et statuts
sudo -u mastodon RAILS_ENV=production PATH=$RBENV_PATH bin/tootctl accounts cull
sudo -u mastodon RAILS_ENV=production PATH=$RBENV_PATH bin/tootctl statuses remove

# Optimisation avancée de la mémoire
echo "Optimisation de la RAM et du SWAP..."
sync; echo 3 > /proc/sys/vm/drop_caches
swapoff -a && swapon -a

echo "--- Maintenance terminée : $(date) ---"

```

### Étape C : Permissions et planification

```bash
chmod 700 /root/scripts/mastodon-cleanup.sh
crontab -e

```

Ajoutez cette ligne (dimanche à 3h00) :

```cron
00 03 * * 0 /bin/bash /root/scripts/mastodon-cleanup.sh >> /var/log/mastodon-cleanup.log 2>&1

```

---

## IV. Notes importantes

**Calendrier de maintenance :** Les commandes `media remove` et `accounts cull` sont recommandées pour une exécution régulière. La commande `statuses remove` peut prendre beaucoup de temps.

**Optimisation de la mémoire :** La commande `drop_caches` et la réinitialisation du SWAP en fin de script permettent de libérer les ressources mobilisées par PostgreSQL et Ruby pendant le nettoyage. Assurez-vous d'avoir assez de RAM disponible pour vider le SWAP.

**Stratégie hybride :** Gardez les réglages de l'interface (Rétention du contenu) actifs avec des valeurs de sécurité (14 ou 30 jours) comme filet de sécurité.

[https://mastodon.blablalinux.be/@blablalinux](https://mastodon.blablalinux.be/@blablalinux)
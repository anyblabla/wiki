---
title: Nettoyage automatique des fichiers temporaires de compression Nextcloud
description: Mise en place d'un cron pour supprimer automatiquement les fichiers .tmp.mp4 orphelins laissés par le script de compression, couvrant tous les utilisateurs actuels et futurs.
published: false
date: 2026-08-15T11:38:20.774Z
tags: cron, bash, ffmpeg, administration, nextcloud
editor: markdown
dateCreated: 2026-08-15T11:38:20.774Z
---

Ce guide complète la mise en place du script de compression automatique des médias Nextcloud. Il explique comment configurer une tâche cron pour supprimer automatiquement les fichiers temporaires orphelins (`.tmp.mp4`) qui peuvent rester sur le serveur en cas d'interruption du script de compression.

---

## 📋 Contexte

Lors d'une compression vidéo, le script crée un fichier temporaire avec l'extension `.tmp.mp4`. Si la compression se termine normalement, ce fichier est renommé en `.mp4` final. Mais si le script est interrompu brutalement (coupure réseau, redémarrage du serveur, erreur inattendue), le fichier temporaire reste sur le disque sans jamais être nettoyé.

Ce fichier orphelin est visible dans l'interface Nextcloud et dans les applications mobiles, ce qui peut être source de confusion.

---

## 🛠️ La solution : une tâche cron de nettoyage

La commande de nettoyage utilise `find` avec plusieurs paramètres clés :

- **`-mindepth 3`** : commence la recherche à l'intérieur des dossiers `files/` des utilisateurs, en ignorant les dossiers système de Nextcloud (`appdata_xxx`, `__groupfolders`, etc.)
- **`-maxdepth 4`** : limite la profondeur de recherche pour rester dans les sous-dossiers directs de `files/`
- **`-mmin +60`** : ne supprime que les fichiers de plus de 60 minutes, pour ne jamais interrompre une compression en cours

Cette approche est **universelle** : elle couvre automatiquement tous les utilisateurs actuels et futurs sans aucune modification à faire lors de l'ajout d'un nouvel utilisateur.

L'arborescence parcourue ressemble à ceci :

```
/srv/nextcloud/data/
    utilisateur1/files/...   ✅ parcouru
    utilisateur2/files/...   ✅ parcouru
    NouvelUtilisateur/...    ✅ parcouru automatiquement
    appdata_ocpg7fho3f71/    ❌ ignoré
    __groupfolders/          ❌ ignoré
    nextcloud.log            ❌ ignoré
```

---

## ⚙️ Configuration du cron

Le nettoyage doit être ajouté dans le crontab de l'utilisateur `www-data`, qui est le propriétaire des fichiers Nextcloud.

```bash
crontab -u www-data -e
```

### Pour une installation dans `/srv/nextcloud/`

Ajoutez cette ligne :

```
# Supprimer les fichiers de compression temporaires orphelins de plus de 60 minutes (quotidien à 04h00)
0 4 * * * find /srv/nextcloud/data -mindepth 3 -maxdepth 4 -name "*.tmp.mp4" -mmin +60 -delete
```

### Pour une installation dans `/var/www/`

Ajoutez cette ligne :

```
# Supprimer les fichiers de compression temporaires orphelins de plus de 60 minutes (quotidien à 04h00)
0 4 * * * find /var/www/nextcloud-data -mindepth 3 -maxdepth 4 -name "*.tmp.mp4" -mmin +60 -delete
```

> **Note** : adaptez le chemin selon votre installation. La logique reste identique dans les deux cas. Choisissez une heure creuse pour éviter de cumuler plusieurs tâches au même moment.

---

## 🔍 Tester manuellement

Avant d'activer le cron, vous pouvez vérifier si des fichiers orphelins sont présents sans les supprimer :

```bash
sudo find /srv/nextcloud/data -mindepth 3 -maxdepth 4 -name "*.tmp.mp4" -mmin +60
```

Si des fichiers apparaissent, vous pouvez les supprimer immédiatement :

```bash
sudo find /srv/nextcloud/data -mindepth 3 -maxdepth 4 -name "*.tmp.mp4" -mmin +60 -delete
```

Si vous voulez forcer la suppression sans attendre les 60 minutes (par exemple pour nettoyer un fichier orphelin récent) :

```bash
sudo find /srv/nextcloud/data -mindepth 3 -maxdepth 4 -name "*.tmp.mp4" -delete
```

---

## 🔄 Mettre à jour l'index Nextcloud après suppression

Après suppression manuelle d'un fichier temporaire, Nextcloud ne le sait pas encore. Le fichier peut encore apparaître dans l'interface ou dans l'application mobile. Pour forcer la mise à jour immédiate :

```bash
sudo -u www-data php8.3 /srv/nextcloud/occ files:scan --all
```

Ou pour ne scanner qu'un utilisateur spécifique :

```bash
sudo -u www-data php8.3 /srv/nextcloud/occ files:scan --path="nom_utilisateur/files/"
```

Le cron de scan existant (`files:scan --all` toutes les 3 heures) se chargera de cette mise à jour automatiquement si vous n'êtes pas pressé.

---

## 💡 Conseil

Évitez de planifier le nettoyage à la même heure que d'autres tâches cron lourdes (scan de fichiers, génération d'aperçus, nettoyage de logs). Répartissez les tâches sur des heures différentes pour garantir la réactivité de votre serveur, surtout sur du matériel modeste ou reconditionné.
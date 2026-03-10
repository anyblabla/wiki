---
title: Sauvegarde des données avec un script Rsync universel
description: Apprenez à utiliser un script Rsync universel pour sauvegarder vos données (home, sites web, clouds). Un guide pour automatiser vos copies de fichiers en complément d'une sauvegarde système.
published: true
date: 2026-03-10T19:46:38.622Z
tags: sauvegarde, rsync, script, bash, linux, automatisation, données
editor: markdown
dateCreated: 2026-03-10T18:23:56.364Z
---

Ce guide explique comment mettre en place une solution de sauvegarde pour vos données personnelles ou applicatives (fichiers, sites web, bases de données) à l'aide d'un script flexible et automatisé.

> 💡 **Complémentarité des sauvegardes**
> Pour une sécurité maximale, ce script doit être utilisé en complément d'une sauvegarde complète du système (OS).
> * **Pour le système :** utilisez **[Timeshift](https://wiki.blablalinux.be/fr/timeshift-serveur)**.
> * **Pour les données :** utilisez ce guide **Rsync**.
> 
> 

---

## 1. Pourquoi utiliser ce script ?

Alors que Timeshift gère des "instantanés" de l'ensemble du système, ce script utilise la puissance de **Rsync** pour copier vos dossiers de données de manière lisible, granulaire et indépendante. Il gère intelligemment la destination pour éviter tout conflit entre les sauvegardes.

> 📝 Pour approfondir le fonctionnement de la commande de base, consultez :
> 👉 **[Rsync : synchronisation et sauvegarde](https://wiki.blablalinux.be/fr/rsync-synchronisation-sauvegarde)**

---

## 2. Préparation de l'environnement

Créez un répertoire pour stocker vos scripts et vos rapports d'exécution (logs) :

```bash
mkdir -p ~/scripts/logs

```

---

## 3. Création du script universel

Créez le fichier : `nano ~/scripts/backup-rsync.sh` et collez-y le contenu suivant.

```bash
#!/bin/bash
# Script de sauvegarde universel - BlablaLinux
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

# ==============================================================================
# CONFIGURATION À PERSONNALISER
# ==============================================================================
# 1. Chemin vers le point de montage de votre disque de sauvegarde
DEST_BASE="/media/VOTRE_UTILISATEUR/NOM_DU_DISQUE/rsync"

# 2. Chemin vers le dossier où vous souhaitez stocker les rapports (logs)
LOG_DIR="/home/VOTRE_UTILISATEUR/scripts/logs"
# ==============================================================================

DATE=$(date +%Y-%m-%d_%Hh%M)
SOURCE=$1

# Création automatique des répertoires si nécessaire
mkdir -p "$LOG_DIR"
mkdir -p "$DEST_BASE"

# Vérification si une source a été renseignée
if [ -z "$SOURCE" ]; then
    echo "Usage: $0 /chemin/a/sauvegarder"
    exit 1
fi

# Gestion du nom du dossier de destination (gère le cas de la racine /)
if [ "$SOURCE" == "/" ]; then
    FOLDER_NAME="SYSTEM_ROOT"
else
    FOLDER_NAME=$(basename "$SOURCE")
fi

DEST_FINAL="$DEST_BASE/$FOLDER_NAME"
LOG_FILE="$LOG_DIR/backup_${FOLDER_NAME}_${DATE}.log"

# Création du dossier de destination final
mkdir -p "$DEST_FINAL"

echo "--- Démarrage de la sauvegarde de $SOURCE vers $DEST_FINAL le $(date) ---" | tee -a "$LOG_FILE"

# Exécution de Rsync avec exclusions de sécurité
# --exclude : évite les boucles infinies sur le disque externe et les dossiers virtuels
sudo rsync -av --delete \
  --exclude={"/dev/*","/proc/*","/sys/*","/tmp/*","/run/*","/mnt/*","/media/*","/lost+found"} \
  "$SOURCE/" "$DEST_FINAL/" 2>&1 | tee -a "$LOG_FILE"

# Nettoyage automatique des logs de plus de 2 jours
find "$LOG_DIR" -name "*.log" -mtime +2 -delete

echo "--- Fin de la sauvegarde le $(date) ---" | tee -a "$LOG_FILE"

```

Rendez le script exécutable : `chmod +x ~/scripts/backup-rsync.sh`.

---

## 4. Utilisation simplifiée (alias)

Pour lancer une sauvegarde ou une restauration rapidement, ajoutez ces raccourcis dans votre fichier `~/.bash_aliases` :

```bash
# Sauvegardes manuelles
alias rsyncsystem='sudo ~/scripts/backup-rsync.sh /'
alias rsynchome='sudo ~/scripts/backup-rsync.sh /home'
alias rsyncwww='sudo ~/scripts/backup-rsync.sh /var/www/html'

# Restaurations (Exemples)
alias rsyncrestoresystem='sudo rsync -av --stats /media/VOTRE_UTILISATEUR/NOM_DU_DISQUE/rsync/SYSTEM_ROOT/ /'
alias rsyncrestorehome='sudo rsync -av --stats /media/VOTRE_UTILISATEUR/NOM_DU_DISQUE/rsync/home/ /home/'

```

---

## 5. Automatisation avec Crontab

Éditez la crontab de l'utilisateur **root** pour gérer les droits d'accès aux dossiers système :

```bash
sudo crontab -e

```

**Exemple de planification (chaque mardi à minuit) :**

```cron
0 0 * * 2 /home/VOTRE_UTILISATEUR/scripts/backup-rsync.sh /home

```

---

## 💡 Notes de sécurité

* **Slash de fin** : Le script utilise la syntaxe `"$SOURCE/"` pour s'assurer que c'est le **contenu** du dossier qui est synchronisé, garantissant une structure propre dans votre backup.
* **Exclusions** : La ligne `--exclude` est vitale. Elle empêche `rsync` de tenter de copier des fichiers éphémères ou, pire, de copier récursivement votre disque de sauvegarde sur lui-même s'il est monté dans `/media` ou `/run`.
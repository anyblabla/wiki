---
title: Sauvegarde des données avec un script Rsync universel
description: Apprenez à utiliser un script Rsync universel pour sauvegarder vos données (home, sites web, clouds). Un guide pour automatiser vos copies de fichiers en complément d'une sauvegarde système.
published: true
date: 2026-03-10T18:23:56.364Z
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

Alors que Timeshift gère des "instantanés" de l'ensemble du système, ce script utilise la puissance de **Rsync** pour copier vos dossiers de données de manière lisible, granulaire et indépendante.

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

# ==============================================================================
# CONFIGURATION À PERSONNALISER (PARTIES ANONYMISÉES)
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
FOLDER_NAME=$(basename "$SOURCE")
[ "$SOURCE" == "/" ] && FOLDER_NAME="SYSTEM_ROOT"
LOG_FILE="$LOG_DIR/backup_${FOLDER_NAME}_${DATE}.log"

echo "--- Démarrage de la sauvegarde de $SOURCE le $(date) ---" | tee -a "$LOG_FILE"

# Exécution de Rsync avec exclusions de sécurité
sudo rsync -av --delete \
  --exclude={"/dev/*","/proc/*","/sys/*","/tmp/*","/run/*","/mnt/*","/media/*","/lost+found"} \
  "$SOURCE" "$DEST_BASE/" 2>&1 | tee -a "$LOG_FILE"

# Nettoyage automatique des logs de plus de 2 jours
find "$LOG_DIR" -name "*.log" -mtime +2 -delete

echo "--- Fin de la sauvegarde le $(date) ---" | tee -a "$LOG_FILE"

```

Rendez le script exécutable : `chmod +x ~/scripts/backup-rsync.sh`.

> 📝 Le terme `/chemin/a/sauvegarder` dans le script est un **exemple d'usage**. Lors de l'exécution, vous devez le remplacer par le dossier réel que vous souhaitez copier (ex: `/home` ou `/var/www`).

---

## 4. Utilisation simplifiée (alias)

Pour lancer une sauvegarde manuelle rapidement, ajoutez des raccourcis dans votre fichier `~/.bash_aliases` :

```bash
# Sauvegardes manuelles (nécessite sudo pour les droits d'accès)
alias rsynchome='sudo ~/scripts/backup-rsync.sh /home'
alias rsyncwww='sudo ~/scripts/backup-rsync.sh /var/www/html'

# Restauration (Exemple pour le dossier /home)
alias rsyncrestorehome='sudo rsync -av --stats /media/VOTRE_UTILISATEUR/NOM_DU_DISQUE/rsync/home /'

```

---

## 5. Automatisation avec Crontab

Pour que vos sauvegardes s'exécutent automatiquement, utilisez le planificateur de tâches de l'utilisateur root.

> ⚠️ **Pourquoi utiliser `sudo crontab -e` ?**
> Pour sauvegarder des fichiers appartenant au système ou à d'autres utilisateurs, le script doit avoir les privilèges de **root**.

**Éditez la crontab de root :**

```bash
sudo crontab -e

```

**Ajoutez votre planification (exemple pour chaque mardi à minuit) :**

```cron
0 0 * * 2 /home/VOTRE_UTILISATEUR/scripts/backup-rsync.sh /home

```

---

## 💡 Notes de personnalisation

* **Organisation du disque** : Il est recommandé de créer un dossier `rsync/` à la racine de votre support externe pour bien séparer ces fichiers des instantanés produits par Timeshift.
* **Exclusions** : La ligne `--exclude` du script est essentielle pour éviter de copier des dossiers systèmes temporaires ou de créer une boucle infinie si vous ciblez la racine (`/`).
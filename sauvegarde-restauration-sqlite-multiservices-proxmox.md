---
title: Sauvegarde et restauration SQLite multiservices
description: Protégez vos bases de données SQLite avec un LXC dédié nommé sqlite-backup. Une solution maison pour automatiser vos sauvegardes Vaultwarden sans coupure et garantir une restauration fiable à 100%.
published: true
date: 2026-03-27T13:51:49.706Z
tags: proxmox, bash, linux, automatisation, vaultwarden, sqlite, backup
editor: markdown
dateCreated: 2026-03-26T11:59:23.576Z
---

> **L'approche BlablaLinux** : Pour mes bases de données externes classiques, je fais confiance à **Databasus**. Mais quand il s'agit de services utilisant des bases de type **SQLite** (comme Vaultwarden), j'utilise ma **solution maison**. Pourquoi ? Parce que SQLite est un fichier plat qui nécessite une manipulation spécifique (notamment la gestion du mode WAL) pour garantir une restauration 100% fiable sans corruption. Voici comment je sécurise mes instances.

---

## 🏗️ Étape 1 : préparation du LXC dédié `sqlite-backup`
Pour centraliser vos sauvegardes, nous allons utiliser un conteneur LXC (Debian ou Ubuntu) nommé **`sqlite-backup`**.

### Installation des dépendances
Sur ce LXC `sqlite-backup`, installez les outils de base :
```bash
apt update && apt install sqlite3 rsync -y
```

### Établir la confiance (SSH sans mot de passe)
1. **Générer une clé** (si besoin) : `ssh-keygen -t ed25519`
2. **Envoyer la clé vers la cible** (ex: Vaultwarden) : `ssh-copy-id root@IP_DU_SERVEUR`
   > **⚠️ Attention** : Le mot de passe demandé ici est celui de l'utilisateur **root du serveur cible**. Une fois validé, le script pourra se connecter tout seul la nuit.

---

## 📜 Étape 2 : le script de sauvegarde (`backup_sqlite.sh`)
Ce script est capable de gérer plusieurs services à la chaîne.

1. Créez le fichier : `nano /root/backup_sqlite.sh`
2. Collez le code ci-dessous. **La partie à modifier pour vos propres services se trouve dans le tableau `TARGETS` (lignes d'exemple fournies).**

```bash
#!/bin/bash

# =========================================================================
# CONFIGURATION DES CIBLES (À MODIFIER SELON VOS BESOINS)
# Ajoutez une ligne par service sous le format : "Nom|IP|Chemin_DB"
# =========================================================================
TARGETS=(
    "vaultwarden|192.168.2.156|/root/vaultwarden/data/db.sqlite3"
    "freshrss|192.168.2.160|/var/lib/docker/volumes/freshrss/_data/db.sqlite"
)
# =========================================================================

BACKUP_ROOT="/root/backups"
DATE=$(date +%Y-%m-%d_%Hh%M)

echo "--- Démarrage de la sauvegarde $(date) ---"

for target in "${TARGETS[@]}"; do
    IFS="|" read -r NAME IP DB_PATH <<< "$target"
    DEST_DIR="$BACKUP_ROOT/$NAME"
    mkdir -p "$DEST_DIR"

    echo "[+] Sauvegarde de $NAME ($IP)..."
    ssh root@$IP "sqlite3 $DB_PATH '.backup /tmp/db_temp.bak'"
    rsync -avz root@$IP:/tmp/db_temp.bak "$DEST_DIR/${NAME}_${DATE}.sqlite3"
    ssh root@$IP "rm /tmp/db_temp.bak"
    find "$DEST_DIR" -type f -mtime +30 -delete
done
echo "--- Sauvegardes terminées avec succès ---"
```
3. Rendez le script exécutable : `chmod +x /root/backup_sqlite.sh`

---

## 🚀 Étape 3 : le script de restauration (un script par service)
La restauration étant une action critique, **utilisez un nom de script spécifique par service** (ex: `restore_vaultwarden.sh`) pour éviter toute erreur de destination.

1. Créez votre script : `nano /root/restore_vaultwarden.sh`
2. **Modifiez les 4 variables du haut** pour correspondre à votre installation.

```bash
#!/bin/bash

# =========================================================================
# CONFIGURATION CIBLE (À MODIFIER POUR CHAQUE NOUVEAU SERVICE)
# =========================================================================
REMOTE_IP="192.168.2.156"
REMOTE_DB_PATH="/root/vaultwarden/data/db.sqlite3"
CONTAINER_NAME="vaultwarden"
LOCAL_BACKUP_DIR="/root/backups/vaultwarden"
# =========================================================================

echo "--- RESTAURATION DU SERVICE : $CONTAINER_NAME ---"
ls -1 $LOCAL_BACKUP_DIR
echo ""
read -p "Copie-colle le nom du fichier à restaurer : " FILE

if [ ! -f "$LOCAL_BACKUP_DIR/$FILE" ]; then echo "Erreur : Fichier introuvable."; exit 1; fi

rsync -avz "$LOCAL_BACKUP_DIR/$FILE" root@$REMOTE_IP:/tmp/db_restore.sqlite3

ssh root@$REMOTE_IP << EOF
    echo "[1/3] Arrêt du conteneur $CONTAINER_NAME..."
    docker stop $CONTAINER_NAME
    
    echo "[2/3] Sécurisation et nettoyage WAL/SHM..."
    cp $REMOTE_DB_PATH ${REMOTE_DB_PATH}.old
    rm -f ${REMOTE_DB_PATH}-wal ${REMOTE_DB_PATH}-shm
    mv /tmp/db_restore.sqlite3 $REMOTE_DB_PATH
    
    echo "[3/3] Relance du conteneur..."
    docker start $CONTAINER_NAME
EOF
echo "--- Restauration terminée ! ---"
```
3. Rendez le script exécutable : `chmod +x /root/restore_vaultwarden.sh`

---

## ⏰ Étape 4 : automatisation avec crontab
Ajoutez la tâche dans votre planificateur : `crontab -e`.
```bash
# Sauvegarde quotidienne SQLite à 03h00 du matin
00 3 * * * /root/backup_sqlite.sh > /root/backup_log.txt 2>&1
```

---

## ➕ Comment ajouter un nouveau service ?
1. **Pour le backup** : Ajoutez simplement une ligne dans le tableau `TARGETS` du script `/root/backup_sqlite.sh`.
2. **Pour la restauration** : 
   - Copiez votre premier script : `cp restore_vaultwarden.sh restore_monservice.sh`
   - Modifiez les variables de configuration en haut du fichier.

---

## 🔗 Liens utiles et articles complémentaires
Pour aller plus loin dans la sécurisation et l'administration de vos services, n'hésitez pas à consulter ces guides :
* [Héberger Vaultwarden avec Docker : guide complet](/fr/heberger-vaultwarden-docker-guide-complet)
* [Automatisation des snapshots Proxmox avec CV4PVE-Autosnap](/fr/cv4pve-autosnap)
* [Gestion des snapshots sur plusieurs nœuds Proxmox](/fr/automatisation-snapshots-proxmox-multi-noeuds)

> **Note** : si vous cherchez d'autres tutoriels, n'hésitez pas à utiliser la barre de recherche du wiki pour découvrir de nombreuses autres pages intéressantes !

---
> **Auteur** : ce guide est proposé par **Amaury aka BlablaLinux**. Retrouvez l'ensemble de mes services sur [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).
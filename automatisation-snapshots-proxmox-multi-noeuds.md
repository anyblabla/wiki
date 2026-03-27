---
title: Automatisation avancée des snapshots Proxmox (multi-nœuds)
description: Découvrez comment optimiser vos snapshots Proxmox sur un cluster multi-nœuds avec cv4pve-autosnap. Un guide complet pour réduire le temps d'exécution et supprimer les erreurs de timeout API.
published: true
date: 2026-03-27T13:55:10.897Z
tags: proxmox, sauvegarde, snapshot, debian, cluster, virtualisation, automatisation
editor: markdown
dateCreated: 2026-03-25T23:01:20.016Z
---

Ce guide détaille comment mettre en place une stratégie de snapshots performante pour un cluster Proxmox. L'utilisation du **ciblage direct** par IP permet de réduire le temps d'exécution de plusieurs minutes à quelques secondes.

> 🗒️ Ce système repose sur l'outil open-source **cv4pve-autosnap**. 
> * **Projet GitHub officiel** : [Corsinvest/cv4pve-autosnap](https://github.com/Corsinvest/cv4pve-autosnap)
> * **Ancienne version du Wiki** : [cv4pve-autosnap (version simple)](/fr/cv4pve-autosnap)

---

## 🏗️ Recommandation : LXC dédié

Bien que vous puissiez installer ces scripts sur n'importe quelle machine Linux (ou même directement sur un nœud Proxmox), **il est vivement recommandé de dédier un petit conteneur LXC (Debian ou Ubuntu)** à cette tâche. 
* Cela isole les outils de sauvegarde du reste de votre infrastructure.
* Cela facilite la maintenance et les mises à jour sans impacter vos services de production.

---

## 📦 Étape 1 : installation de l'outil

Une fois dans votre LXC, copiez et collez ce bloc pour installer les dépendances et la dernière version de l'exécutable :

```bash
apt update && apt install wget unzip curl -y
LATEST_URL=$(curl -s https://api.github.com/repos/Corsinvest/cv4pve-autosnap/releases/latest | grep "browser_download_url.*linux-x64.zip" | cut -d '"' -f 4)
cd /tmp && wget -O autosnap-latest.zip "$LATEST_URL"
unzip -o autosnap-latest.zip && mv cv4pve-autosnap /usr/local/bin/
chmod +x /usr/local/bin/cv4pve-autosnap && rm autosnap-latest.zip
```

---

## 🏗️ Étape 2 : le fichier de configuration

Ce fichier centralise vos accès et définit vos nœuds Proxmox.

1. **Créez le fichier :**
   ```bash
   nano /root/autosnap.conf
   ```
2. **Collez et adaptez le contenu suivant :**
   ```bash
   # Liste complète de vos nœuds (séparés par des virgules) pour les backups globaux
   PVE_HOSTS="192.168.1.10,192.168.1.11,192.168.1.12"

   # Token API Proxmox (Format: utilisateur@pve!ID=SECRET)
   PVE_TOKEN="votre_user@pve!votre_id=votre-secret-uuid"

   # Dossier de stockage des logs
   LOG_DIR="/root/autosnap-logs"

   # RACCOURCIS NOEUDS (Ciblage direct pour la performance)
   NODE1="192.168.1.10"
   NODE2="192.168.1.11"
   NODE3="192.168.1.12"
   ```

---

## 🛠️ Étape 3 : le script maître

C'est lui qui pilote les snapshots et génère les logs de manière structurée.

1. **Créez le script :**
   ```bash
   nano /usr/local/bin/autosnap-master.sh
   ```
2. **Collez le contenu suivant :**
   ```bash
   #!/bin/bash
   source /root/autosnap.conf

   HOST=$1    # IP du nœud cible
   LABEL=$2   # Label (ex: hourly)
   KEEP=$3    # Rétention
   ID=${4:-all} # VMID ou "all"

   if [ -z "$HOST" ] || [ -z "$LABEL" ] || [ -z "$KEEP" ]; then
       echo "Usage: $0 [HOST] [label] [keep] [vmid(optionnel)]"
       exit 1
   fi

   mkdir -p "$LOG_DIR"
   LOG_FILE="$LOG_DIR/${LABEL}-${ID}-$(date +%Y-%m-%d).log"

   echo "--- START JOB $LABEL pour ID $ID sur $HOST ($(date '+%H:%M:%S')) ---" >> "$LOG_FILE"
   /usr/local/bin/cv4pve-autosnap --host="$HOST" --api-token="$PVE_TOKEN" --vmid="$ID" snap --label="-$LABEL-" --keep="$KEEP" >> "$LOG_FILE" 2>&1
   echo "--- END JOB $LABEL ---" >> "$LOG_FILE"
   ```
3. **Rendez-le exécutable :**
   ```bash
   chmod +x /usr/local/bin/autosnap-master.sh
   ```

---

## 🔄 Étape 4 : scripts de maintenance

1. **Mise à jour automatique (`nano /usr/local/bin/update-autosnap.sh`) :**
   ```bash
   #!/bin/bash
   LATEST_URL=$(curl -s https://api.github.com/repos/Corsinvest/cv4pve-autosnap/releases/latest | grep "browser_download_url.*linux-x64.zip" | cut -d '"' -f 4)
   cd /tmp && wget -q "$LATEST_URL" -O autosnap-latest.zip
   unzip -o autosnap-latest.zip && mv cv4pve-autosnap /usr/local/bin/
   chmod +x /usr/local/bin/cv4pve-autosnap && rm autosnap-latest.zip
   ```

2. **Purge des logs (`nano /usr/local/bin/autosnap-purge-logs.sh`) :**
   ```bash
   #!/bin/bash
   source /root/autosnap.conf
   find "$LOG_DIR" -name "*.log" -type f -mtime +30 -delete
   ```
3. **Permission :** `chmod +x /usr/local/bin/update-autosnap.sh /usr/local/bin/autosnap-purge-logs.sh`

---

## 📅 Étape 5 : automatisation (Crontab)

1. **Ouvrez le crontab :** `crontab -e`
2. **Ajoutez vos tâches :**
   ```text
   # Maintenance
   0 4 * * 1 /usr/local/bin/update-autosnap.sh > /dev/null 2>&1
   0 5 * * * /usr/local/bin/autosnap-purge-logs.sh

   # Snapshots horaires (ciblés par nœud pour la vitesse)
   # Important : utilisez ". /root/autosnap.conf &&" pour charger vos variables
   0 * * * * . /root/autosnap.conf && /usr/local/bin/autosnap-master.sh "$NODE1" hourly-gitea 24 4060
   5 * * * * . /root/autosnap.conf && /usr/local/bin/autosnap-master.sh "$NODE2" hourly-vaultwarden 24 138

   # Sauvegardes globales (tous les nœuds à 00h30)
   30 0 * * * . /root/autosnap.conf && /usr/local/bin/autosnap-master.sh "$PVE_HOSTS" daily-all 7 all
   ```

---

## 🧠 Pourquoi cibler un nœud spécifique ?

Sur un cluster multi-nœuds, scanner l'ensemble des hôtes pour trouver une seule VM peut provoquer des ralentissements importants et des erreurs de timeout ("Response ended prematurely"). En ciblant l'IP directe du nœud via les variables `$NODE_X`, l'opération est **instantanée**.

> **Attention** : en cas de migration (vMotion) d'une VM vers un autre nœud, pensez à mettre à jour la variable `$NODE_X` dans votre crontab.

---

> **Auteur** : ce guide est proposé par **Amaury aka BlablaLinux**. Retrouvez l'ensemble de mes services sur [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).
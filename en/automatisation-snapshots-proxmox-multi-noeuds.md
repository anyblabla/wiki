---
title: Advanced Proxmox Snapshot Automation (Multi-Node)
description: >-
  Learn how to optimize your Proxmox snapshots on a multi-node cluster using
  cv4pve-autosnap. A complete guide to reduce execution time and eliminate API
  timeout errors.
published: true
date: '2026-11-21T21:53:23.574Z'
tags:
  - proxmox
  - backup
  - snapshot
  - debian
  - cluster
  - virtualization
  - automation
editor: markdown
dateCreated: '2026-11-21T21:53:23.574Z'
---

This guide details how to set up an efficient snapshot strategy for a Proxmox cluster. Using **direct targeting** by IP reduces execution time from several minutes to just a few seconds.

> 🗒️ This system relies on the open-source tool **cv4pve-autosnap**.
> - **Official GitHub Project**: [Corsinvest/cv4pve-autosnap](https://github.com/Corsinvest/cv4pve-autosnap)
> - **Previous Wiki Version**: [cv4pve-autosnap (simple version)](/en/cv4pve-autosnap)

---

## 🏗️ Recommendation: Dedicated LXC

Although you can install these scripts on any Linux machine (or even directly on a Proxmox node), **it is strongly recommended to dedicate a small LXC container (Debian or Ubuntu)** to this task.

- This isolates the backup tools from the rest of your infrastructure.
- It makes maintenance and updates easier without impacting your production services.

---

## 📦 Step 1: Installing the Tool

Once inside your LXC container, copy and paste this block to install dependencies and the latest version of the executable:

```bash
apt update && apt install wget unzip curl -y
LATEST_URL=$(curl -s https://api.github.com/repos/Corsinvest/cv4pve-autosnap/releases/latest | grep "browser_download_url.*linux-x64.zip" | cut -d '"' -f 4)
cd /tmp && wget -O autosnap-latest.zip "$LATEST_URL"
unzip -o autosnap-latest.zip && mv cv4pve-autosnap /usr/local/bin/
chmod +x /usr/local/bin/cv4pve-autosnap && rm autosnap-latest.zip
```

---

## 🏗️ Step 2: Configuration File

This file centralizes your access credentials and defines your Proxmox nodes.

1. **Create the file:**
   ```bash
   nano /root/autosnap.conf
   ```
2. **Paste and adapt the following content:**
   ```bash
   # Complete list of your nodes (comma-separated) for global backups
   PVE_HOSTS="192.168.1.10,192.168.1.11,192.168.1.12"

   # Proxmox API Token (Format: user@pve!ID=SECRET)
   PVE_TOKEN="your_user@pve!your_id=your-secret-uuid"

   # Log storage directory
   LOG_DIR="/root/autosnap-logs"

   # NODE SHORTCUTS (Direct targeting for performance)
   NODE1="192.168.1.10"
   NODE2="192.168.1.11"
   NODE3="192.168.1.12"
   ```

---

## 🛠️ Step 3: Master Script

This script manages snapshots and generates structured logs.

1. **Create the script:**
   ```bash
   nano /usr/local/bin/autosnap-master.sh
   ```
2. **Paste the following content:**
   ```bash
   #!/bin/bash
   source /root/autosnap.conf

   HOST=$1      # Target node IP
   LABEL=$2     # Label (e.g., hourly)
   KEEP=$3      # Retention count
   ID=${4:-all} # VMID or "all"

   if [ -z "$HOST" ] || [ -z "$LABEL" ] || [ -z "$KEEP" ]; then
       echo "Usage: $0 [HOST] [label] [keep] [vmid(optional)]"
       exit 1
   fi

   mkdir -p "$LOG_DIR"
   LOG_FILE="$LOG_DIR/${LABEL}-${ID}-$(date +%Y-%m-%d).log"

   echo "--- START JOB $LABEL for ID $ID on $HOST ($(date '+%H:%M:%S')) ---" >> "$LOG_FILE"
   /usr/local/bin/cv4pve-autosnap --host="$HOST" --api-token="$PVE_TOKEN" --vmid="$ID" snap --label="-$LABEL-" --keep="$KEEP" >> "$LOG_FILE" 2>&1
   echo "--- END JOB $LABEL ---" >> "$LOG_FILE"
   ```
3. **Make it executable:**
   ```bash
   chmod +x /usr/local/bin/autosnap-master.sh
   ```

---

## 🔄 Step 4: Maintenance Scripts

1. **Automatic Update (`nano /usr/local/bin/update-autosnap.sh`):**
   This script checks GitHub weekly for a new version and updates the binary only if necessary.
   ```bash
   #!/bin/bash

   # Configuration
   REPO="Corsinvest/cv4pve-autosnap"
   INSTALL_DIR="/usr/local/bin"
   BINARY_NAME="cv4pve-autosnap"
   TMP_DIR="/tmp/autosnap_update"

   # Ensure required tools are present
   apt update && apt install curl wget unzip -y > /dev/null 2>&1

   # 1. Fetch latest version from GitHub
   LATEST_TAG=$(curl -s https://api.github.com/repos/$REPO/releases/latest | grep tag_name | cut -d '"' -f 4)

   # 2. Check current version
   if [ -f "$INSTALL_DIR/$BINARY_NAME" ]; then
       CURRENT_VERSION="v$($INSTALL_DIR/$BINARY_NAME --version | grep -oE '[0-9]+\.[0-9]+\.[0-9]+')"
   else
       CURRENT_VERSION="none"
   fi

   if [ "$LATEST_TAG" == "$CURRENT_VERSION" ]; then
       echo "✅ cv4pve-autosnap is already up to date ($CURRENT_VERSION)."
       exit 0
   fi

   echo "🚀 New version found: $LATEST_TAG (current: $CURRENT_VERSION)"

   # 3. Download linux-x64 binary
   mkdir -p $TMP_DIR
   DOWNLOAD_URL=$(curl -s https://api.github.com/repos/$REPO/releases/latest | grep "browser_download_url" | grep "linux-x64.zip" | cut -d '"' -f 4)

   wget -qO $TMP_DIR/autosnap.zip "$DOWNLOAD_URL"
   unzip -qj $TMP_DIR/autosnap.zip -d $TMP_DIR
   chmod +x $TMP_DIR/$BINARY_NAME
   mv $TMP_DIR/$BINARY_NAME $INSTALL_DIR/

   # Cleanup
   rm -rf $TMP_DIR

   echo "✨ Update to $LATEST_TAG completed!"
   ```

2. **Log Purge (`nano /usr/local/bin/autosnap-purge-logs.sh`):**
   ```bash
   #!/bin/bash
   source /root/autosnap.conf
   find "$LOG_DIR" -name "*.log" -type f -mtime +30 -delete
   ```

3. **Make both scripts executable:**
   ```bash
   chmod +x /usr/local/bin/update-autosnap.sh /usr/local/bin/autosnap-purge-logs.sh
   ```

---

## 📅 Step 5: Automation (Crontab)

1. **Open the crontab:**
   ```bash
   crontab -e
   ```
2. **Add your tasks:**
   ```cron
   # Maintenance
   0 4 * * 1 /usr/local/bin/update-autosnap.sh > /dev/null 2>&1
   0 5 * * * /usr/local/bin/autosnap-purge-logs.sh

   # Critical services snapshots (every 4h, keep 24h - targeted per node)
   # Important: use ". /root/autosnap.conf &&" to load your variables
   0 */4 * * * . /root/autosnap.conf && /usr/local/bin/autosnap-master.sh "$NODE1" hourly-gitea 6 4060
   5 */4 * * * . /root/autosnap.conf && /usr/local/bin/autosnap-master.sh "$NODE2" hourly-vaultwarden 6 138

   # Global snapshots (all nodes every night, keep 2 days)
   10 0 * * * . /root/autosnap.conf && /usr/local/bin/autosnap-master.sh "$PVE_HOSTS" daily-all 2 all
   ```

> **Tip:** A snapshot is not a backup. Keep a short local retention period (to avoid bloating your ZFS pools) and leave long-term archiving to **Proxmox Backup Server (PBS)**.

---

## 🧠 Why target a specific node?

On a multi-node cluster, scanning all hosts to find a single VM can cause significant slowdowns and timeout errors ("Response ended prematurely"). By directly targeting the node's IP via `$NODE_X` variables, the operation is **instantaneous**.

> **Warning:** In case of migration (live migration or manual move) of a VM to another node, remember to update the corresponding `$NODE_X` variable in your crontab.

---

> **Author**: This guide is provided by **Amaury aka BlablaLinux**. Find all my services at [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).

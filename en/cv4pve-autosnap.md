---
title: Automating Proxmox VE Snapshots with CV4PVE-AUTOSNAP
description: >-
  Details on installing and configuring the CV4PVE-AUTOSNAP tool (Corsinvest) in
  an LXC container to automate snapshots of your VMs and CTs on Proxmox VE.
published: true
date: '2026-11-21T21:53:38.788Z'
tags:
  - proxmox
  - snapshot
  - backup
  - debian
  - lxc
  - automation
editor: markdown
dateCreated: '2026-11-21T21:53:38.788Z'
---

## 💡 Context and Tool Selection

For taking snapshots of guests on [Proxmox VE](https://www.proxmox.com/en/proxmox-virtual-environment/overview), several methods exist:

- The Proxmox web interface ([GUI](https://w.wiki/CZ6F)).
- The command line ([CLI](https://w.wiki/6gSf)) using the native [PVESH](https://pve.proxmox.com/pve-docs/chapter-pvesh.html) tool.
- Third-party tools, such as [CV4PVE-ADMIN](https://corsinvest.it/cv4pve-admin-proxmox/) (multi-cluster GUI) which uses **CV4PVE-AUTOSNAP** under the hood.

> **Important Note (March 2026):** This guide presents the basic configuration with individual scripts. For optimized management on a multi-node cluster, check out the newer version: [**Advanced Proxmox Snapshot Automation (Multi-Node)**](/en/automatisation-snapshots-proxmox-multi-noeuds).

> I use CV4PVE-ADMIN, which is a multi-cluster manager. However, I only use the automatic snapshot feature, which is like using a sledgehammer to crack a nut.
>
> Therefore, I will show you how to use CV4PVE-AUTOSNAP, not directly on the Proxmox host, but isolated inside an [LXC](https://w.wiki/CXkQ) container.
>
> Personally, I deployed a [Debian 12 Bookworm](https://www.debian.org/releases/bookworm/) container (this procedure also works on Debian 13 Trixie).

---

## 1. Installation and Preparation in the LXC Container

All of the following commands must be executed inside your dedicated Debian/Ubuntu LXC container.

### 1. Downloading the Binary

Get the latest version of CV4PVE-AUTOSNAP available at [this address](https://github.com/Corsinvest/cv4pve-autosnap/releases). *(Version 1.15.0 at the time of writing)*:

```bash
wget https://github.com/Corsinvest/cv4pve-autosnap/releases/download/v1.15.0/cv4pve-autosnap-linux-x64.zip
```

### 2. Extraction and Installation

Update your repositories, install the `unzip` utility, extract the archive, and make the binary executable:

```bash
apt update && apt install -y unzip
unzip cv4pve-autosnap-linux-x64.zip
chmod +x cv4pve-autosnap
```

### 3. Finalization

Move the binary file to the `/usr/local/bin/` directory (standard location for manually added tools) so it can be executed from anywhere in the system:

```bash
mv cv4pve-autosnap /usr/local/bin/
mkdir -p /root/autosnap-logs
```

### 4. Verification

To get help on available options:

```bash
cv4pve-autosnap --help
```

---

## 2. Creating the Snapshot Script

We will now create a first script for taking snapshots at specific times (9 AM and 9 PM in this example).

### 1. Base Command Structure

The main command for `cv4pve-autosnap` uses the following structure:

```bash
cv4pve-autosnap --host=XXX.XXX.X.XXX --username=root@pam --password=XXXXXXXX --vmid=XXX snap --label=XXXXX --keep=X
```

> **From here on, all commands must be adapted!**
>
> **Security Advice:** In this introductory example, the `root@pam` password is used in plain text to simplify setup. In production, it is strongly recommended to create a **Proxmox API Token** with restricted rights (`--api-token=user@pve!token-name=secret-value`) rather than using the `root` account.

### 2. Creating the `autosnap-921.sh` Script

```bash
nano /root/autosnap-921.sh
```

Paste the following content:

```bash
#!/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=""
############# SNAPSHOT SCRIPT ITSELF ######################
echo " " >> /root/autosnap-logs/921-$(date "+%Y-%m-%d").log
echo "###NEW JOB STARTS HERE###" >> /root/autosnap-logs/921-$(date "+%Y-%m-%d").log
cv4pve-autosnap --host=XXX.XXX.X.XXX --username=root@pam --password=XXXXXXXX --vmid=all snap --label=-921- --keep=2 >> /root/autosnap-logs/921-$(date "+%Y-%m-%d").log
echo "###NEW JOB ENDS HERE###" >> /root/autosnap-logs/921-$(date "+%Y-%m-%d").log
```

### 3. Customizing Arguments

| Argument | Description | Possible Values |
| :--- | :--- | :--- |
| `--host=` | Proxmox node IP address. | `XXX.XXX.X.XXX` |
| `--username=` | Proxmox login username. | `root@pam` |
| `--password=` | Password for the specified account. | `XXXXXXXX` |
| `--vmid=` | Guest ID (or `all`). | `all` |
| `--label=` | Snapshot name (e.g., `-921-`). | `-921-` |
| `--keep=` | Number of snapshots to retain. | `2` |

> **Note:** For a cluster, you can specify multiple IPs: `--host="IP1,IP2,IP3"`.

### 4. Making the Script Executable

```bash
chmod +x /root/autosnap-921.sh
```

---

## 3. Scheduling with Cron

### 1. Open Crontab

```bash
crontab -e
```

### 2. Add Schedule

Execution every day at 9:00 AM and 9:00 PM:

```cron
0 9,21 * * * /root/autosnap-921.sh
```

---

## 4. Automatic Update of CV4PVE-AUTOSNAP

Since the tool is installed manually via a GitHub binary, it is not updated by `apt`. Here is a small script that checks weekly if a new version is available and installs it automatically.

### 1. Creating the `update-autosnap.sh` Script

```bash
nano /usr/local/bin/update-autosnap.sh
```

Paste the following content:

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

### 2. Make the Script Executable and Test It

```bash
chmod +x /usr/local/bin/update-autosnap.sh
/usr/local/bin/update-autosnap.sh
```

### 3. Weekly Schedule in Cron

Add this line in your `crontab -e` to check for updates every Monday at 4:00 AM:

```cron
# Automatic update of cv4pve-autosnap tool (Monday at 4:00 AM)
0 4 * * 1 /usr/local/bin/update-autosnap.sh > /dev/null 2>&1
```

---

## 5. Maintenance and Management Examples

### 1. Deleting Snapshots (Cleanup)

```bash
cv4pve-autosnap --host=XXX.XXX.X.XXX --username=root@pam --password=XXXXXXXX --vmid=all clean --label='-921-' --keep=0
```

### 2. Displaying Status

```bash
cv4pve-autosnap --host=XXX.XXX.X.XXX --username=root@pam --password=XXXXXXXX --vmid=all status
```

---

## 6. Going Further: Advanced Retention Strategies

You can replicate the retention model (hourly, daily, weekly, monthly, yearly) offered by Proxmox VE.

### 1. Creating Scripts for Each Interval

| Script Name | Frequency | Ex. Label | Retention |
| :--- | :--- | :--- | :--- |
| `autosnap-hourly.sh` | Every hour | `-hourly-` | `--keep=3` |
| `autosnap-daily.sh` | Every day | `-daily-` | `--keep=7` |
| `autosnap-weekly.sh` | Every week | `-weekly-` | `--keep=4` |
| `autosnap-monthly.sh` | Every month | `-monthly-` | `--keep=2` |
| `autosnap-yearly.sh` | Every year | `-yearly-` | `--keep=1` |

> **Warning:** A snapshot is not a backup. Avoid keeping too many long-term snapshots (monthly or yearly), as this may bloat your storage and degrade performance (especially under ZFS). Reserve long-term retention for a dedicated backup server like **Proxmox Backup Server (PBS)**.

### 2. Cron Task Scheduling

```cron
# Scheduled tasks
0 * * * * /root/autosnap-hourly.sh
10 0 * * * /root/autosnap-daily.sh
20 0 * * 0 /root/autosnap-weekly.sh
30 0 1 * * /root/autosnap-monthly.sh
40 0 1 1 * /root/autosnap-yearly.sh
```

---

## 7. Sample Log File

```plaintext
###NEW JOB STARTS HERE###
ACTION Snap
PVE Version: 8.3.2
VMs: all
Label: -921-
Keep: 2
Total execution 00:01:18.3608764
###NEW JOB ENDS HERE###
```

---

## 8. Resources and Cron Generators

- [Crontab.guru](https://crontab.guru)
- [Cronhub Crontab Generator](https://crontab.cronhub.io)
- [Crontab-generator.org](https://crontab-generator.org)

---

> **Author**: This guide is provided by **Amaury aka BlablaLinux**. Find all my services at [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).

---
title: Automating Phanpy updates
description: Learn how to automate Phanpy updates on a Debian server. This guide provides a Bash script to check GitHub releases, manage installation, and configure a Cron job for daily updates.
published: true
date: 2026-01-08T23:14:38.620Z
tags: mastodon, lxc, debian, script, bash, phanpy, automation
editor: markdown
dateCreated: 2026-01-08T23:10:31.448Z
---

This page documents how to set up a Bash script to keep your **Phanpy** instance (web client for Mastodon) up to date by monitoring official releases on GitHub.

## Useful links

* **Official repository:** [cheeaun/phanpy](https://github.com/cheeaun/phanpy){:target="_blank"}
* **GitHub releases:** [Phanpy Releases](https://github.com/cheeaun/phanpy/releases){:target="_blank"}
* **Demo instance:** [Phanpy by Blabla Linux](https://phanpy.blablalinux.be){:target="_blank"}

---

## Prerequisites

The script uses `curl` to query the GitHub API. On minimal installations (such as Debian LXC templates), this dependency might be missing.

1. **Update repositories and install curl:**
```bash
apt update && apt install curl -y

```



> **Note:** While there is no official Docker image for Phanpy (as it consists of static files), you can find my Docker guides for other services here: [Installing Docker, Portainer, and LXC on Debian and Proxmox](https://wiki.blablalinux.be/fr/docker-portainer-lxc-debian-proxmox).

---

## Environment preparation

You need to create a dedicated folder for your scripts and initialize a version file so the script can compare your local installation with the latest GitHub release.

1. **Create the scripts folder:**
```bash
mkdir -p /root/scripts

```


2. **Initialize the version file:**
Identify your current version and save it into the tracker file to enable the first comparison:
```bash
echo "2025.11.26.ac85274" > /var/www/html/.phanpy_version

```



---

## Creating the automation script

1. **Create the script file:**
```bash
nano /root/scripts/update_phanpy.sh

```


2. **Paste the following content:**

```bash
#!/bin/bash
# Automatic update script for Phanpy
# Author: Amaury aka BlablaLinux

REPO="cheeaun/phanpy"
INSTALL_DIR="/var/www/html"
VERSION_FILE="$INSTALL_DIR/.phanpy_version"
FILENAME="phanpy-dist.tar.gz"

echo "--- Starting Phanpy update check ---"

# Fetch the latest stable release tag via GitHub API
LATEST_RELEASE=$(curl -s https://api.github.com/repos/$REPO/releases/latest | grep '"tag_name":' | sed -E 's/.*"([^"]+)".*/\1/')

if [ -z "$LATEST_RELEASE" ]; then
    echo "Error: Unable to reach GitHub API."
    exit 1
fi

# Fetch the local version
CURRENT_VERSION=$([ -f "$VERSION_FILE" ] && cat "$VERSION_FILE" || echo "none")

if [ "$LATEST_RELEASE" == "$CURRENT_VERSION" ]; then
    echo "Phanpy is already up to date ($CURRENT_VERSION)."
    exit 0
fi

echo "New version detected: $LATEST_RELEASE. Updating..."

TEMP_DIR=$(mktemp -d)
URL="https://github.com/$REPO/releases/download/$LATEST_RELEASE/$FILENAME"

if curl -L -s -o "$TEMP_DIR/$FILENAME" "$URL"; then
    # Preventive cleanup of old files to avoid asset conflicts
    # Deletes everything in /var/www/html except the version file itself
    find "$INSTALL_DIR" -mindepth 1 -not -name '.phanpy_version' -delete
    
    # Extract the new version
    tar -xzf "$TEMP_DIR/$FILENAME" -C "$INSTALL_DIR"
    
    # Update the local version tag
    echo "$LATEST_RELEASE" > "$VERSION_FILE"
    echo "Update to $LATEST_RELEASE completed successfully."
else
    echo "Error while downloading the archive."
    exit 1
fi

rm -rf "$TEMP_DIR"

```

3. **Make the script executable:**
```bash
chmod +x /root/scripts/update_phanpy.sh

```



---

## Manual validation and verification

Run the script manually once to ensure the deployment works as expected:

```bash
/root/scripts/update_phanpy.sh

```

Once the script has finished, verify that the version file has been updated with the latest GitHub tag:

```bash
cat /var/www/html/.phanpy_version

```

---

## Cron job configuration

To automate the check every night at 04:15 AM and generate a log file:

1. **Open the root crontab:**
```bash
crontab -e

```


2. **Add the following line at the end of the file:**
```cron
15 4 * * * /bin/bash /root/scripts/update_phanpy.sh > /var/log/phanpy_update.log 2>&1

```
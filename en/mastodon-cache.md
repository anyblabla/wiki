---
title: Maintenance Mastodon - Cleaning and optimizing caches with tootctl
description: Mastodon maintenance in classic installation (bare-metal): automation of cache cleaning, media and inactive accounts with Gotify notifications.
published: true
date: 2026-01-10T23:07:02.779Z
tags: mastodon, cache, delete, gotify
editor: markdown
dateCreated: 2026-01-10T23:07:02.779Z
---

>  This guide refers to a "classical" (non-Docker) installation of Mastodon based on Debian. If you are using an installation running under **Docker**, please consult this dedicated page: [Maintenance Mastodon under Docker](https://wiki.blablalinux.be/en/maintenance-mastodon-docker).

Mastodon accumulates various types of data and caches over time (images, media, remote accounts, etc.). Regular use of `tootctl` commands is essential to free up disk space and maintain the performance of your instance.

---

##1. Main cleaning levels

Here are the basic commands to clear the different cache levels and remove obsolete data:

- Cleaning - Order tootctl
- ...-------------------------------------
**Cache Vignettes** - Yeah.
**Oldmedia** - Yeah.
**Orphaned media** - Yeah.
**Obsolete accounts** - Yeah.
**Obsolete status** - Yeah.

---

##2. Script automation (with Gotify optional)

This script centralizes cleaning tasks for your bare-metal installation. It can send you a notification via **Gotify** if it is configured.

> 
> Simply leave empty `GOTIFY_URL` and `GOTIFY_TOKEN` variables. The script will detect the lack of configuration and ignore sending messages without making an error.

### Contentof the script: `mastodon-cleanup.sh`

``bash
#!/bin/bash
# Mastodon Maintenance Script (classical installation)
# Author: Amaury aka BlablaLinux

# --- GOTIFY PARAMETERS (Optional) ---
GOTIFY_URL=""
GOTIFY_TOKEN=""

# --- MAINTENANCE PARAMETERS ---
MASTODON_DIR="/var/www/mastodon/live"
RBENV_PATH="/opt/rbenv/versions/mastodon/bin"
LOGFILE="/var/log/mastodon-cleanup.log"
HOSTNAME=$(hostname)
DAYS_MEDIA=7
THREADS=4

# Redirect all output to log file
exec 1>>$LOGFILE 2>&1

# --- GOTIFY NOTIFICATION FUNCTIONS ---
send_gotify_notification() {
if [ -n "$GOTIFY_URL" ] && [ -n "$GOTIFY_TOKEN" ]; then
local title="$1"
local message="$2"
local priority="$3"

curl -k -s -X POST "${GOTIFY_URL}/message?token=${GOTIFY_TOKEN}" \
-F "title=${title}" \
-F "message=${message}" \
-F Priority=${priority}" > /dev/null 2>&1
fi
}

echo "===============================================================================================
echo "Start of maintenance Mastodon on $HOSTNAME: $(date)"
echo "===============================================================================================

cd $MASTODON_DIR
MSG="Error: Mastodon folder not found. "
echo "MSG$"
send_gotify_notification "
exit 1
}

# 1. Media cleaning and thumbnails
echo "--- Step 1: Media cleaning and thumbnails ---"
sudo -u mastodon RAILS_ENV=production PATH=$RBENV_PATH bin/tootctl media remove --days=$DAYS_MEDIA --concurrence=$THREADS
sudo -u mastodon RAILS_ENV=production PATH=$RBENV_PATH bin/tootctl cache clean

#2. Cleaning of accounts and statutes
echo "--- Step 2: Cleaning of accounts and status ---"
sudo -u mastodon RAILS_ENV=production PATH=$RBENV_PATH bin/tootctl accounts prune
sudo -u mastodon RAILS_ENV=production PATH=$RBENV_PATH bin/tootctl status removes
if [ $? -ne 0 ]; then
send_gotify_notification "
fi

#3. RAM Optimization (LXC Protection)
echo "--- Step 3: Release of RAM cache ---"
sync
if [ -w /proc/sys/vm/drop_caches ]; then
echo 3 > /proc/sys/vm/drop_caches
echo "Cache RAM released successfully."
Else
echo "Note: Insufficient rights for drop_caches (LXC), ignored."
fi

echo "===============================================================================================
echo "Maintenance completed: $(date)"
echo "===============================================================================================

#4. Final success notification sent
send_gotify_notification "

exit 0

````

---

##3. Planning with Crontab

1. Rendre the executable script (root only): `chmod 700 /root/scripts/mastodon-cleanup.sh`
2. Add to the crontab (`crontab -e`) for an execution on Sunday at 3:00 am :

``ron
00 03 * * 0 /bin/bash /root/scripts/mastodon-cleanup.sh >> /var/log/mastodon-cleanup.log 2>&1

````

---

##4. Important notes

**Optimization of memory:** The `drop_caches' command included in the script frees resources mobilized by PostgreSQL and Ruby. If your server is an LXC container, the script automatically manages rights restrictions to avoid crashing.

**Hybrid strategy:** Keep the interface settings (Content retention) active with security values (14 or 30 days) as a safety net if the script does not run.

(https://mastodon.blablalinux.be/@blablalinux)
---
title: Mastodon maintenance - Cleaning and optimising caches with tootctl
description: Mastodon maintenance in bare-metal installation: automated cleaning of cache, media and inactive accounts with Gotify notifications.
published: true
date: 2026-01-10T23:49:52.315Z
tags: mastodon, cache, delete, gotify
editor: markdown
dateCreated: 2026-01-10T23:49:52.315Z
---

&gt; ⚠️ This guide concerns a "classic" (non-Docker) installation of Mastodon on a Debian base. If you are using an installation running under **Docker**, please consult this dedicated page: [Mastodon maintenance under Docker](https://wiki.blablalinux.be/fr/maintenance-mastodon-docker).

Mastodon accumulates various types of data and caches over time (images, media, remote accounts, etc.). Regular use of the `tootctl` commands is essential to free up disk space and maintain the performance of your instance.

---

## 1. Main cleanup levels

Here are the basic commands for clearing the various cache levels and deleting obsolete data:

| Cleaning | Command `tootctl` | Description |
| --- | --- | --- |
| Cached thumbnails** | `tootctl cache clear` | Removes small thumbnails from other servers. |
|tootctl media remove` | Removes images/videos older than the retention period. |
| Orphaned media** | `tootctl media remove-orphans` | Removes media files that are no longer linked to anything. |
| Obsolete accounts** | `tootctl accounts prune` | Removes inactive remote profiles (heavy). |
|tootctl statuses remove` | Removes toots that no longer exist on the federation (very heavy). |

---

## 2. Script automation (with optional Gotify)

This script centralises cleaning tasks for your bare-metal installation. It can send you a notification via **Gotify** if configured.

&gt; You don't use Gotify?
&gt; Simply leave the `GOTIFY_URL` and `GOTIFY_TOKEN` variables empty. The script will detect the absence of configuration and will ignore the sending of messages without making an error.

### Contents of script: `mastodon-cleanup.sh`

```bash
#!/bin/bash
# Script de maintenance Mastodon (installation classique)
# Auteur : Amaury aka BlablaLinux

# --- PARAMÈTRES DE GOTIFY (Optionnel) ---
GOTIFY_URL=""
GOTIFY_TOKEN=""

# --- PARAMÈTRES DE MAINTENANCE ---
MASTODON_DIR="/var/www/mastodon/live"
RBENV_PATH="/opt/rbenv/versions/mastodon/bin"
LOGFILE="/var/log/mastodon-cleanup.log"
HOSTNAME=$(hostname)
DAYS_MEDIA=7
THREADS=4

# Redirection de toute la sortie vers le fichier journal
exec 1>>$LOGFILE 2>&1

# --- FONCTION DE NOTIFICATION GOTIFY ---
send_gotify_notification() {
    if [ -n "$GOTIFY_URL" ] && [ -n "$GOTIFY_TOKEN" ]; then
        local title="$1"
        local message="$2"
        local priority="$3"

        curl -k -s -X POST "${GOTIFY_URL}/message?token=${GOTIFY_TOKEN}" \
            -F "title=${title}" \
            -F "message=${message}" \
            -F "priority=${priority}" > /dev/null 2>&1
    fi
}

echo "======================================================"
echo "Début de la maintenance Mastodon sur $HOSTNAME : $(date)"
echo "======================================================"

cd $MASTODON_DIR || {
    MSG="Erreur : dossier Mastodon introuvable."
    echo "$MSG"
    send_gotify_notification "❌ Mastodon Cleanup ÉCHEC" "$MSG sur $HOSTNAME." 8
    exit 1
}

# 1. Nettoyage des médias et vignettes
echo "--- Étape 1 : Nettoyage des médias et vignettes ---"
sudo -u mastodon RAILS_ENV=production PATH=$RBENV_PATH bin/tootctl media remove --days=$DAYS_MEDIA --concurrency=$THREADS
sudo -u mastodon RAILS_ENV=production PATH=$RBENV_PATH bin/tootctl cache clear

# 2. Nettoyage des comptes et statuts
echo "--- Étape 2 : Nettoyage des comptes et statuts ---"
sudo -u mastodon RAILS_ENV=production PATH=$RBENV_PATH bin/tootctl accounts prune
sudo -u mastodon RAILS_ENV=production PATH=$RBENV_PATH bin/tootctl statuses remove
if [ $? -ne 0 ]; then
    send_gotify_notification "⚠️ Mastodon Cleanup ALERTE" "Le nettoyage des statuts a rencontré une erreur sur $HOSTNAME." 5
fi

# 3. Optimisation de la mémoire RAM (Protection LXC)
echo "--- Étape 3 : Libération du cache RAM ---"
sync
if [ -w /proc/sys/vm/drop_caches ]; then
    echo 3 > /proc/sys/vm/drop_caches
    echo "Cache RAM libéré avec succès."
else
    echo "Note : Droits insuffisants pour drop_caches (LXC), ignoré."
fi

echo "======================================================"
echo "Maintenance terminée : $(date)"
echo "======================================================"

# 4. Envoi de la notification de succès final
send_gotify_notification "✅ Mastodon Cleanup TERMINÉ" "La maintenance sur $HOSTNAME est terminée avec succès." 4

exit 0

```

---

## 3. Planning with Crontab

1. Make the script executable (for root only): `chmod 700 /root/scripts/mastodon-cleanup.sh`
2. Add to the crontab (`crontab -e`) to run on Sunday at 3:00:

```cron
00 03 * * 0 /bin/bash /root/scripts/mastodon-cleanup.sh >> /var/log/mastodon-cleanup.log 2>&1

```

---

## 4. Important notes

**Memory optimisation:** The `drop_caches` command included in the script frees up resources used by PostgreSQL and Ruby. If your server is an LXC container, the script automatically manages rights restrictions to avoid crashing.

**Hybrid strategy:** Keep the interface settings (Content retention) active with security values (14 or 30 days) as a safety net if the script does not run.

[https://mastodon.blablalinux.be/@blablalinux](https://mastodon.blablalinux.be/@blablalinux)
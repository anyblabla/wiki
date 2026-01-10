---
title: Mastodon maintenance - Cleaning and optimising caches with tootctl
description: Mastodon maintenance in bare-metal installation: automated cleaning of cache, media and inactive accounts with Gotify notifications.
published: true
date: 2026-01-10T23:53:51.788Z
tags: mastodon, cache, delete, gotify
editor: markdown
dateCreated: 2026-01-10T23:53:51.788Z
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

This script centralises the cleaning tasks for your bare-metal installation. It can send you a notification via **Gotify** if configured.

&gt; You don't use Gotify?
&gt; Simply leave the `GOTIFY_URL` and `GOTIFY_TOKEN` variables empty. The script will detect the absence of configuration and will ignore the sending of messages without making an error.

### Contents of script: `mastodon-cleanup.sh`

[PROTECT_0]]

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
---
title: Managing Watchtower in LXC containers
description: This page describes the script used to manage Watchtower in LXC containers running on Proxmox VE. It allows you to check the status, start, stop, restart Watchtower, and update its configurations automatically.
published: true
date: 2026-08-01T23:28:45.685Z
tags: docker, lxc, proxmox, script, watchtower, pve, compose
editor: markdown
dateCreated: 2026-03-13T23:45:40.354Z
---

## Introduction

**Watchtower** is a tool that monitors your Docker containers and updates them automatically. This page documents two methods for managing your Watchtower instances deployed in LXC containers directly from the Proxmox host.

📺 **Demo:** Check out the demo video for this script on Blabla Linux's Mastodon instance: [https://mastodon.blablalinux.be/@blablalinux/115826788636738220](https://mastodon.blablalinux.be/@blablalinux/115826788636738220)

These scripts let you:

* Identify **LXCs** running **Docker**.
* Find Watchtower's `docker-compose.yml` files.
* **View and modify the essential options** (schedule, cleanup, notification URL, image, etc.).
* **Automatically migrate** legacy Gotify notifications to the modern Shoutrrr format (`WATCHTOWER_NOTIFICATION_URL`).
* **Clean up** unused Docker images (`prune`) and old compose backups.

> **Technical note:** These scripts run on the **Proxmox host**. They use the `pct exec [ID]` command to drive Docker inside the containers without having to connect to each instance individually.

> **⚠️ About the Gotify deprecation:** as of Watchtower v1.20+, the `WATCHTOWER_NOTIFICATION_GOTIFY_URL`, `WATCHTOWER_NOTIFICATION_GOTIFY_TOKEN`, and `WATCHTOWER_NOTIFICATION_GOTIFY_TLS_SKIP_VERIFY` variables are deprecated in favor of a single Shoutrrr URL (`WATCHTOWER_NOTIFICATION_URL`). They will be removed with the release of Watchtower v2. **Script 2** includes an automatic migration option (see below).

---

## 🔗 Official Repositories

Find the source code and contribute to the project on our repositories (external links):

* **GitHub:** [https://github.com/anyblabla/proxmox-watchtower-manager](https://github.com/anyblabla/proxmox-watchtower-manager)
* **Gitea:** [https://gitea.blablalinux.be/blablalinux/proxmox-watchtower-manager](https://gitea.blablalinux.be/blablalinux/proxmox-watchtower-manager)

---

## 🛠️ Installation and Setup

These steps assume you are connected via **SSH** to your **Proxmox host**.

### 1. Create the scripts directory

```bash
sudo mkdir -p /root/scripts
```

### 2. Make the scripts executable

Once you've created the files (see the source code below), don't forget to set the right permissions:

```bash
sudo chmod +x /root/scripts/manage_watchtower.sh
sudo chmod +x /root/scripts/manage_watchtower_all.sh
```

---

## 🧭 Comparing the two scripts

| Feature | **Script 1: Standard** | **Script 2: Full Maintenance** |
| --- | --- | --- |
| **Target** | **Running** LXCs only. | **All** LXCs (running & stopped). |
| **Filtering** | Auto-detection (if Docker is running). | Only if the exact **`watchtower`** tag is present. |
| **State handling** | Doesn't change anything. | Starts the LXC, acts, then **shuts it back down**. |
| **Reliability** | Immediate execution. | Waits for Docker to be ready (wait-loop) + post-boot stabilization pause. |
| **Compose lookup** | Fixed path `*/watchtower/docker-compose.yml`. | Two-step search (targeted path, then content-based fallback), tolerates `.yaml` files and folders not named `watchtower`. |
| **Gotify notifications** | Manual editing of the legacy variable only. | Automatic migration to Shoutrrr (`WATCHTOWER_NOTIFICATION_URL`) with a `.bak` backup. |
| **Backup cleanup** | Not available. | Targeted removal of leftover `.bkp` files. |

---

## 📜 Script 1: Standard (`manage_watchtower.sh`)

This script is ideal for quick changes on services that are currently running in production.

```bash
#!/bin/bash
# ==============================================================================
# Script: manage_watchtower.sh
# Description: Centralized Watchtower management for LXCs (running only).
# Author: Amaury aka BlablaLinux
# Website: https://blablalinux.be
# Wiki: https://wiki.blablalinux.be/fr/script-gestion-watchtower
# License: GPL-3.0
# Version: 1.1.0
# ==============================================================================

MENU="
===============================================
    Watchtower Management in LXC Containers
===============================================
 [1] 🔍 View current Watchtower status
 [2] 🚀 Start Watchtower
 [3] 🛑 Stop Watchtower
 [4] 🔁 Restart Watchtower
 [5] 📂 View editable docker-compose.yml content
 [6] 🔄 Set restart policy (always/none)
 [7] ✏️  Edit WATCHTOWER_NO_STARTUP_MESSAGE
 [8] ✏️  Edit WATCHTOWER_CLEANUP
 [9] 📅 Set random schedule (2pm-8pm)
 [10] 📅 Set the same schedule for all
 [11] ✏️  Edit WATCHTOWER_TIMEOUT
 [12] ✏️  Edit WATCHTOWER_NOTIFICATION_GOTIFY_URL
 [13] 🖼️  Edit the Docker image
 [14] 🧹 Clean up all images (prune -a)
 [Q] ❌ Quit
"

get_running_docker_lxc() {
    pct list | awk 'NR>1 && $2=="running"{print $1}' | while read lxc; do
        if pct exec "$lxc" -- docker ps >/dev/null 2>&1; then
            echo "$lxc"
        fi
    done
}

find_watchtower_compose() {
    timeout 5s pct exec "$1" -- find /root -type f -path "*/watchtower/docker-compose.yml" 2>/dev/null | head -n1
}

status_watchtower() {
    for lxc_id in $(get_running_docker_lxc); do
        compose_file=$(find_watchtower_compose "$lxc_id")
        echo "→ LXC $lxc_id"
        [ -n "$compose_file" ] && pct exec "$lxc_id" -- docker ps --filter name=watchtower || echo "Not found."
    done
    read -rp "Press [Enter]..."
}

start_watchtower() {
    for lxc_id in $(get_running_docker_lxc); do
        compose_file=$(find_watchtower_compose "$lxc_id")
        if [ -n "$compose_file" ]; then
            dir=$(dirname "$compose_file")
            pct exec "$lxc_id" -- sh -c "cd $dir && docker compose up -d"
            echo "🚀 Started in LXC $lxc_id"
        fi
    done
    read -rp "Done. [Enter]..."
}

stop_watchtower() {
    for lxc_id in $(get_running_docker_lxc); do
        pct exec "$lxc_id" -- docker stop watchtower >/dev/null 2>&1 && echo "🛑 Stopped in LXC $lxc_id"
    done
    read -rp "Done. [Enter]..."
}

restart_watchtower() {
    for lxc_id in $(get_running_docker_lxc); do
        compose_file=$(find_watchtower_compose "$lxc_id")
        if [ -n "$compose_file" ]; then
            dir=$(dirname "$compose_file")
            pct exec "$lxc_id" -- sh -c "cd $dir && docker compose down && docker compose up -d"
            echo "🔁 Restarted in LXC $lxc_id"
        fi
    done
    read -rp "Done. [Enter]..."
}

view_compose() {
    for lxc_id in $(get_running_docker_lxc); do
        compose_file=$(find_watchtower_compose "$lxc_id")
        echo "→ LXC $lxc_id"
        [ -n "$compose_file" ] && pct exec "$lxc_id" -- sh -c "grep -E 'image:|restart:|WATCHTOWER_NO_STARTUP_MESSAGE|WATCHTOWER_CLEANUP|WATCHTOWER_SCHEDULE|WATCHTOWER_TIMEOUT|WATCHTOWER_NOTIFICATION_GOTIFY_URL' $compose_file"
    done
    read -rp "Press [Enter]..."
}

modify_key_restart() {
    key=$1
    val=$2
    for lxc_id in $(get_running_docker_lxc); do
        compose_file=$(find_watchtower_compose "$lxc_id")
        if [ -n "$compose_file" ]; then
            pct exec "$lxc_id" -- sed -i "s|^\s*-\s*$key=.*|      - $key=$val|" "$compose_file"
            dir=$(dirname "$compose_file")
            pct exec "$lxc_id" -- sh -c "cd $dir && docker compose down && docker compose up -d"
            echo "✅ $key updated in LXC $lxc_id"
        fi
    done
}

set_restart_policy() {
    read -rp "Policy (always/none): " new_policy
    for lxc_id in $(get_running_docker_lxc); do
        compose_file=$(find_watchtower_compose "$lxc_id")
        if [ -n "$compose_file" ]; then
            pct exec "$lxc_id" -- sed -i "s/^[[:space:]]*restart: .*/    restart: $new_policy/" "$compose_file"
            dir=$(dirname "$compose_file")
            pct exec "$lxc_id" -- sh -c "cd $dir && docker compose down && docker compose up -d"
            echo "✅ Policy $new_policy in LXC $lxc_id"
        fi
    done
}

random_schedule() {
    for lxc_id in $(get_running_docker_lxc); do
        compose_file=$(find_watchtower_compose "$lxc_id")
        if [ -n "$compose_file" ]; then
            hour=$((RANDOM % 7 + 14))
            minute=$((RANDOM % 12 * 5))
            schedule="0 $minute $hour ? * 5"
            pct exec "$lxc_id" -- sed -i "s|^\s*-\s*WATCHTOWER_SCHEDULE=.*|      - WATCHTOWER_SCHEDULE=$schedule|" "$compose_file"
            dir=$(dirname "$compose_file")
            pct exec "$lxc_id" -- sh -c "cd $dir && docker compose down && docker compose up -d"
            echo "✅ Schedule $schedule for LXC $lxc_id"
        fi
    done
}

set_watchtower_image() {
    read -rp "Image (e.g. containrrr/watchtower:latest): " img
    for lxc_id in $(get_running_docker_lxc); do
        compose_file=$(find_watchtower_compose "$lxc_id")
        if [ -n "$compose_file" ]; then
            pct exec "$lxc_id" -- sed -i "s#^[[:space:]]*image: .*#    image: $img#" "$compose_file"
            dir=$(dirname "$compose_file")
            pct exec "$lxc_id" -- sh -c "cd $dir && docker compose down && docker compose up -d"
        fi
    done
}

prune_docker_images() {
    read -rp "Confirm prune -a on ALL running LXCs? (yes/no): " conf
    if [[ "$conf" =~ ^[Yy][Ee][Ss]$ ]]; then
        for lxc_id in $(get_running_docker_lxc); do
            pct exec "$lxc_id" -- docker image prune -a -f
        done
    fi
}

while true; do
    clear ; echo "$MENU" ; read -rp "Choice: " choice
    case $choice in
        1) status_watchtower ;;
        2) start_watchtower ;;
        3) stop_watchtower ;;
        4) restart_watchtower ;;
        5) view_compose ;;
        6) set_restart_policy ;;
        7) read -rp "true/false: " v; modify_key_restart "WATCHTOWER_NO_STARTUP_MESSAGE" "$v" ;;
        8) read -rp "true/false: " v; modify_key_restart "WATCHTOWER_CLEANUP" "$v" ;;
        9) random_schedule ;;
        10) read -rp "Cron: " v; modify_key_restart "WATCHTOWER_SCHEDULE" "$v" ;;
        11) read -rp "Timeout: " v; modify_key_restart "WATCHTOWER_TIMEOUT" "$v" ;;
        12) read -rp "New Gotify URL: " v; modify_key_restart "WATCHTOWER_NOTIFICATION_GOTIFY_URL" "$v" ;;
        13) set_watchtower_image ;;
        14) prune_docker_images ;;
        [Qq]) exit ;;
    esac
done
```

---

## 🚀 Script 2: Full Maintenance (`manage_watchtower_all.sh`)

This script is designed for bulk maintenance. **It will start stopped containers** that carry the `watchtower` tag exclusively, apply your changes, then shut them back down.

### Using Tags

For this script to process a container, you need to add the **watchtower** tag in the Proxmox UI (or via `pct set ID --tags watchtower`). The script deliberately ignores derived tags such as `watchtower-custom`.

### 🆕 What's new in v2.0.0

* **Gotify → Shoutrrr migration (option 15):** reads the legacy `WATCHTOWER_NOTIFICATION_GOTIFY_URL` / `_TOKEN` variables, automatically builds the corresponding Shoutrrr URL (`gotify://host[:port]/token`, with `?disabletls=yes` appended if the original URL was `http://`), removes the deprecated keys, inserts `WATCHTOWER_NOTIFICATION_URL`, and restarts the service. A `.bak` backup of the compose file is created before any changes. If `WATCHTOWER_NOTIFICATIONS` references another notifier alongside `gotify`, that line is left untouched (a warning is shown so you can check it manually).
* **Backup cleanup (option 16):** removes leftover `*.bkp` files in the compose file's directory (no confirmation prompt). The `.bak` files generated by the migration are deliberately left alone.
* **More reliable compose lookup:** `find_watchtower_compose` first tries a quick path-based search (`*watchtower*`, depth limited to 4, `.yml`/`.yaml`), then falls back to a content-based search (`grep -r` for the `containrrr/watchtower` or `nickfedor/watchtower` image) if nothing is found — covering both folders with different names and LXCs that are slower to respond.
* **Post-boot stabilization pause:** after starting a stopped LXC, a `sleep 3` is added before scanning files, to avoid false negatives caused by an operating system that's still booting.
* **Explicit error messages:** every action now clearly shows `🚫 Watchtower compose not found for this LXC.` on failure, instead of staying silent.
* **Option 12 refocused:** now edits `WATCHTOWER_NOTIFICATION_URL` directly (the modern Shoutrrr format) instead of the old Gotify variable.

```bash
#!/bin/bash
# ==============================================================================
# Script: manage_watchtower_all.sh
# Description: Watchtower management for all LXCs (all states) via Tags.
# Features: Auto-start/stop LXC, Tag filtering (watchtower), Docker wait-loop,
#           legacy Gotify -> Shoutrrr migration, .bkp backup cleanup.
# Author: Amaury aka BlablaLinux
# Website: https://blablalinux.be
# Wiki: https://wiki.blablalinux.be/fr/script-gestion-watchtower
# License: GPL-3.0
# Version: 2.0.0
# ==============================================================================

MENU="
===============================================
   Watchtower Management - 'watchtower' TAG ONLY
===============================================
 [1] 🔍 View current Watchtower status
 [2] 🚀 Start Watchtower
 [3] 🛑 Stop Watchtower
 [4] 🔁 Restart Watchtower
 [5] 📂 View docker-compose.yml content
 [6] 🔄 Set restart policy (always/none)
 [7] ✏️  Edit WATCHTOWER_NO_STARTUP_MESSAGE
 [8] ✏️  Edit WATCHTOWER_CLEANUP
 [9] 📅 Set random schedule (2pm-8pm on Saturdays)
 [10] 📅 Set the same schedule for all
 [11] ✏️  Edit WATCHTOWER_TIMEOUT
 [12] ✏️  Edit WATCHTOWER_NOTIFICATION_URL (Shoutrrr)
 [13] 🖼️  Edit the Docker image
 [14] 🧹 Clean up all images (prune -a)
 [15] 🔀 Migrate legacy Gotify -> Shoutrrr (WATCHTOWER_NOTIFICATION_URL)
 [16] 🗑️  Remove old backups (*.bkp)
 [Q] ❌ Quit
"

# --- MASTER FUNCTION (Tag filtering and state handling) ---

run_action_on_all() {
    local action_func=$1

    for lxc_id in $(pct list | awk 'NR>1{print $1}'); do
        # Get the LXC's tags
        tags=$(pct config "$lxc_id" | grep "^tags:" | awk '{print $2}')

        # --- WHOLE-WORD TAG MATCH FIX ---
        if echo "$tags" | grep -qE "(^|;)watchtower(;|$)" ; then
            initial_status=$(pct status "$lxc_id" | awk '{print $2}')
            hostname=$(pct config "$lxc_id" | grep "^hostname:" | awk '{print $2}')
            echo "--- Processing LXC $lxc_id ($hostname) ---"
        else
            continue
        fi

        was_stopped=false
        if [ "$initial_status" == "stopped" ]; then
            echo "⚡ Starting the LXC..."
            pct start "$lxc_id"
            was_stopped=true

            # Docker wait-loop
            echo -n "⏳ Waiting for Docker..."
            success=false
            for i in {1..15}; do
                if pct exec "$lxc_id" -- docker ps >/dev/null 2>&1; then
                    echo " OK!"
                    success=true
                    break
                fi
                echo -n "."
                sleep 1
            done

            if [ "$success" = false ]; then
                echo -e "\n❌ Docker unreachable. Moving to the next one."
                pct stop "$lxc_id"
                continue
            fi
        fi

        # Run the action
        if pct exec "$lxc_id" -- docker ps >/dev/null 2>&1; then
            if [ "$was_stopped" = true ]; then
                # Small pause: right after boot, the FS/IO can still be busy
                # (services starting up), so we let the system settle before
                # scanning files.
                sleep 3
            fi
            $action_func "$lxc_id"
        else
            echo "🚫 Error: Docker not ready."
        fi

        # Return to the initial state
        if [ "$was_stopped" = true ]; then
            echo "💤 Shutting back down..."
            pct stop "$lxc_id"
        fi
    done
    read -rp "Done. Press [Enter]..."
}

# --- INDIVIDUAL FUNCTIONS ---

_status() {
    compose_file=$(find_watchtower_compose "$1")
    if [ -n "$compose_file" ]; then
        pct exec "$1" -- docker ps --filter name=watchtower
    else
        echo "🚫 Watchtower compose not found for this LXC."
    fi
}
_start() {
    compose_file=$(find_watchtower_compose "$1")
    if [ -n "$compose_file" ]; then
        dir=$(dirname "$compose_file")
        pct exec "$1" -- sh -c "cd $dir && docker compose up -d"
        echo "🚀 Started."
    else
        echo "🚫 Watchtower compose not found for this LXC."
    fi
}
_stop() { pct exec "$1" -- docker stop watchtower >/dev/null 2>&1 && echo "🛑 Stopped." || echo "🚫 watchtower container not found/already stopped."; }
_restart() {
    compose_file=$(find_watchtower_compose "$1")
    if [ -n "$compose_file" ]; then
        dir=$(dirname "$compose_file")
        pct exec "$1" -- sh -c "cd $dir && docker compose down && docker compose up -d"
        echo "🔁 Restarted."
    else
        echo "🚫 Watchtower compose not found for this LXC."
    fi
}
_view() {
    compose_file=$(find_watchtower_compose "$1")
    if [ -n "$compose_file" ]; then
        echo "📄 $compose_file"
        pct exec "$1" -- sh -c "grep -E 'image:|restart:|WATCHTOWER_NO_STARTUP_MESSAGE|WATCHTOWER_CLEANUP|WATCHTOWER_SCHEDULE|WATCHTOWER_TIMEOUT|WATCHTOWER_NOTIFICATION_GOTIFY_URL|WATCHTOWER_NOTIFICATION_GOTIFY_TOKEN|WATCHTOWER_NOTIFICATIONS|WATCHTOWER_NOTIFICATION_URL' $compose_file"
    else
        echo "🚫 Watchtower compose not found for this LXC."
    fi
}
_modify_key() {
    compose_file=$(find_watchtower_compose "$1")
    if [ -n "$compose_file" ]; then
        pct exec "$1" -- sed -i "s|^\s*-\s*$GLOBAL_KEY=.*|      - $GLOBAL_KEY=$GLOBAL_VAL|" "$compose_file"
        dir=$(dirname "$compose_file")
        pct exec "$1" -- sh -c "cd $dir && docker compose down && docker compose up -d"
        echo "✅ $GLOBAL_KEY updated."
    else
        echo "🚫 Watchtower compose not found for this LXC."
    fi
}
_set_image() {
    compose_file=$(find_watchtower_compose "$1")
    if [ -n "$compose_file" ]; then
        pct exec "$1" -- sed -i "s#^[[:space:]]*image: .*#    image: $GLOBAL_VAL#" "$compose_file"
        dir=$(dirname "$compose_file")
        pct exec "$1" -- sh -c "cd $dir && docker compose down && docker compose up -d"
        echo "✅ Image updated."
    else
        echo "🚫 Watchtower compose not found for this LXC."
    fi
}
_random_sched() {
    compose_file=$(find_watchtower_compose "$1")
    if [ -n "$compose_file" ]; then
        hour=$((RANDOM % 7 + 14)) ; minute=$((RANDOM % 12 * 5)) ; schedule="0 $minute $hour ? * 6"
        pct exec "$1" -- sed -i "s|^\s*-\s*WATCHTOWER_SCHEDULE=.*|      - WATCHTOWER_SCHEDULE=$schedule|" "$compose_file"
        dir=$(dirname "$compose_file")
        pct exec "$1" -- sh -c "cd $dir && docker compose down && docker compose up -d"
        echo "✅ Schedule set: $schedule"
    else
        echo "🚫 Watchtower compose not found for this LXC."
    fi
}
_prune() { echo "🧹 Pruning images..."; pct exec "$1" -- docker image prune -a -f; }
_set_policy() {
    compose_file=$(find_watchtower_compose "$1")
    if [ -n "$compose_file" ]; then
        pct exec "$1" -- sed -i "s/^[[:space:]]*restart: .*/    restart: $GLOBAL_VAL/" "$compose_file"
        dir=$(dirname "$compose_file")
        pct exec "$1" -- sh -c "cd $dir && docker compose down && docker compose up -d"
        echo "✅ Restart policy updated."
    else
        echo "🚫 Watchtower compose not found for this LXC."
    fi
}
# Two-step search: fast path-based lookup, then content-based fallback.
find_watchtower_compose() {
    local lxc_id="$1"
    local result

    # 1) Quick, targeted search: path containing "watchtower", limited depth.
    result=$(timeout 8s pct exec "$lxc_id" -- sh -c "find /root /opt /home -maxdepth 4 -type f \( -iname 'docker-compose.yml' -o -iname 'docker-compose.yaml' \) -path '*watchtower*' 2>/dev/null | head -n1")
    if [ -n "$result" ]; then
        echo "$result"
        return
    fi

    # 2) Fallback: broad content-based scan (slower), in case the folder doesn't
    #    contain "watchtower" in its name. More generous timeout.
    timeout 20s pct exec "$lxc_id" -- sh -c "grep -rlE 'image:[[:space:]]*(containrrr|nickfedor)/watchtower' /root /opt /home 2>/dev/null | head -n1"
}

# --- LEGACY GOTIFY -> SHOUTRRR MIGRATION ---
#
# Reads WATCHTOWER_NOTIFICATION_GOTIFY_URL / _TOKEN, builds a
# gotify://host[:port]/token URL (with ?disabletls=yes if the original URL was
# http://), removes the 3 old keys (URL, TOKEN, TLS_SKIP_VERIFY) and the
# "gotify" entry in WATCHTOWER_NOTIFICATIONS if it's the only one, then inserts
# WATCHTOWER_NOTIFICATION_URL. The file is backed up as .bak before any change.

_migrate_gotify() {
    lxc_id="$1"
    compose_file=$(find_watchtower_compose "$lxc_id")
    if [ -z "$compose_file" ]; then
        echo "🚫 No compose found, skipping this LXC."
        return
    fi

    url_line=$(pct exec "$lxc_id" -- grep -E 'WATCHTOWER_NOTIFICATION_GOTIFY_URL=' "$compose_file" 2>/dev/null)
    token_line=$(pct exec "$lxc_id" -- grep -E 'WATCHTOWER_NOTIFICATION_GOTIFY_TOKEN=' "$compose_file" 2>/dev/null)

    if [ -z "$url_line" ] || [ -z "$token_line" ]; then
        echo "ℹ️  No legacy Gotify config found (already migrated or not configured). Skipping this LXC."
        return
    fi

    # Detect whether another notifier is declared alongside gotify (more complex config -> no auto-removal)
    notif_line=$(pct exec "$lxc_id" -- grep -E 'WATCHTOWER_NOTIFICATIONS=' "$compose_file" 2>/dev/null)
    notif_val=$(echo "$notif_line" | sed -E 's/.*WATCHTOWER_NOTIFICATIONS=//; s/"//g' | tr -d '[:space:]')

    old_url=$(echo "$url_line" | sed -E 's/.*WATCHTOWER_NOTIFICATION_GOTIFY_URL=//; s/"//g' | tr -d '[:space:]')
    old_token=$(echo "$token_line" | sed -E 's/.*WATCHTOWER_NOTIFICATION_GOTIFY_TOKEN=//; s/"//g' | tr -d '[:space:]')

    if echo "$old_url" | grep -qE '^https://'; then
        host_port=$(echo "$old_url" | sed -E 's#^https://##; s#/$##')
        tls_suffix=""
    else
        host_port=$(echo "$old_url" | sed -E 's#^http://##; s#/$##')
        tls_suffix="?disabletls=yes"
    fi

    new_url="gotify://${host_port}/${old_token}${tls_suffix}"

    echo "🔎 LXC $lxc_id: old URL = $old_url"
    echo "   -> New Shoutrrr URL: $new_url"

    # Backup
    pct exec "$lxc_id" -- cp "$compose_file" "${compose_file}.bak"

    # Remove old keys
    pct exec "$lxc_id" -- sed -i '/WATCHTOWER_NOTIFICATION_GOTIFY_URL=/d' "$compose_file"
    pct exec "$lxc_id" -- sed -i '/WATCHTOWER_NOTIFICATION_GOTIFY_TOKEN=/d' "$compose_file"
    pct exec "$lxc_id" -- sed -i '/WATCHTOWER_NOTIFICATION_GOTIFY_TLS_SKIP_VERIFY=/d' "$compose_file"

    if [ "$notif_val" = "gotify" ]; then
        pct exec "$lxc_id" -- sed -i '/WATCHTOWER_NOTIFICATIONS=/d' "$compose_file"
    elif [ -n "$notif_val" ]; then
        echo "⚠️  WATCHTOWER_NOTIFICATIONS contains '$notif_val' (not just gotify)."
        echo "    This line was NOT removed automatically, please check it manually."
    fi

    # Insert the new key right after the "environment:" line
    pct exec "$lxc_id" -- sed -i "/^\s*environment:/a\\      - WATCHTOWER_NOTIFICATION_URL=${new_url}" "$compose_file"

    dir=$(dirname "$compose_file")
    pct exec "$lxc_id" -- sh -c "cd $dir && docker compose down && docker compose up -d"
    echo "✅ Migration completed for LXC $lxc_id (backup: ${compose_file}.bak)"
}

# --- BACKUP CLEANUP (.bkp) ---
#
# Looks, in the same folder as the found docker-compose.yml, for any file
# ending in .bkp (old manual backups) and removes them. The .bak files
# generated by option 15 are deliberately left untouched.

_cleanup_backups() {
    lxc_id="$1"
    compose_file=$(find_watchtower_compose "$lxc_id")
    if [ -z "$compose_file" ]; then
        echo "🚫 Watchtower compose not found for this LXC."
        return
    fi

    dir=$(dirname "$compose_file")
    backups=$(pct exec "$lxc_id" -- find "$dir" -maxdepth 1 -type f -name "*.bkp" 2>/dev/null)

    if [ -z "$backups" ]; then
        echo "ℹ️  No .bkp backup found in $dir."
        return
    fi

    echo "🗂️  .bkp backups found in $dir:"
    echo "$backups" | sed 's/^/   - /'

    pct exec "$lxc_id" -- find "$dir" -maxdepth 1 -type f -name "*.bkp" -delete
    echo "✅ .bkp backups removed."
}

# --- MENU LOOP ---

while true; do
    clear ; echo "$MENU" ; read -rp "Your choice: " choice
    case $choice in
        1) run_action_on_all _status ;;
        2) run_action_on_all _start ;;
        3) run_action_on_all _stop ;;
        4) run_action_on_all _restart ;;
        5) run_action_on_all _view ;;
        6) read -rp "Policy (always/none): " GLOBAL_VAL ; run_action_on_all _set_policy ;;
        7) GLOBAL_KEY="WATCHTOWER_NO_STARTUP_MESSAGE" ; read -rp "true/false: " GLOBAL_VAL ; run_action_on_all _modify_key ;;
        8) GLOBAL_KEY="WATCHTOWER_CLEANUP" ; read -rp "true/false: " GLOBAL_VAL ; run_action_on_all _modify_key ;;
        9) run_action_on_all _random_sched ;;
        10) GLOBAL_KEY="WATCHTOWER_SCHEDULE" ; read -rp "Cron: " GLOBAL_VAL ; run_action_on_all _modify_key ;;
        11) GLOBAL_KEY="WATCHTOWER_TIMEOUT" ; read -rp "Value: " GLOBAL_VAL ; run_action_on_all _modify_key ;;
        12) GLOBAL_KEY="WATCHTOWER_NOTIFICATION_URL" ; read -rp "New Shoutrrr URL (e.g. gotify://host:port/token?disabletls=yes): " GLOBAL_VAL ; run_action_on_all _modify_key ;;
        13) read -rp "Image: " GLOBAL_VAL ; run_action_on_all _set_image ;;
        14) read -rp "Confirm prune (yes/no): " conf ; [[ "$conf" =~ ^[Yy][Ee][Ss]$ ]] && run_action_on_all _prune ;;
        15) read -rp "⚠️  Migrate all 'watchtower' LXCs to Shoutrrr? (yes/no): " conf ; [[ "$conf" =~ ^[Yy][Ee][Ss]$ ]] && run_action_on_all _migrate_gotify ;;
        16) run_action_on_all _cleanup_backups ;;
        [Qq]) exit ;;
        *) echo "Invalid option." ; sleep 1 ;;
    esac
done
```

---

## 🔀 Using the Gotify → Shoutrrr migration

Recommended order to migrate an LXC (or check ones already migrated):

1. **[5]** — status check: shows the current variables (legacy Gotify or already Shoutrrr).
2. **[15]** — migration: switches to `WATCHTOWER_NOTIFICATION_URL`, removes the old keys, restarts the container, creates a `.bak` safety backup.
3. **[5]** — verification: confirms `WATCHTOWER_NOTIFICATION_URL` is in place and the old keys are gone.
4. **[16]** — cleanup: removes any leftover `.bkp` files (older manual backups). The `.bak` created in step 2 is kept.

---

## 📘 Additional Notes

* To easily install Docker on your LXCs, check out our dedicated page: [docker-portainer-lxc-debian-proxmox](/fr/docker-portainer-lxc-debian-proxmox).
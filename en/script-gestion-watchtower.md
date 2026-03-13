---
title: Managing Watchtower in LXC containers
description: This page describes the script used to manage Watchtower in LXC containers running on Proxmox VE. It allows you to check the status, start, stop, restart Watchtower, and update its configurations automatically.
published: true
date: 2026-03-13T23:45:40.354Z
tags: docker, lxc, proxmox, script, watchtower, pve, compose
editor: markdown
dateCreated: 2026-03-13T23:45:40.354Z
---

## Introduction

**Watchtower** is a tool that monitors your Docker containers and updates them automatically. This page documents two methods to manage your Watchtower instances deployed in LXC containers directly from the Proxmox host.

📺 **Demo:** You can find the demo video of this script on the <a href="[https://mastodon.blablalinux.be/about](https://mastodon.blablalinux.be/about)" target="_blank">Blabla Linux Mastodon</a> instance: <a href="[https://mastodon.blablalinux.be/@blablalinux/115826788636738220](https://mastodon.blablalinux.be/@blablalinux/115826788636738220)" target="_blank">[https://mastodon.blablalinux.be/@blablalinux/115826788636738220](https://mastodon.blablalinux.be/@blablalinux/115826788636738220)</a>

These scripts allow you to:

* Identify **LXCs** containing **Docker**.
* Locate Watchtower `docker-compose.yml` files.
* **View and modify essential options** (Schedule, Cleanup, Image, etc.).
* **Clean up** unused Docker images (`prune`).

> **Technical Note:** These scripts run on the **Proxmox host**. They use the `pct exec [ID]` command to manage Docker inside the containers without having to log into each one individually.

---

## 🔗 Official Repositories

Find the source code and contribute to the project on our repositories (external links):

* **GitHub:** <a href="[https://github.com/anyblabla/proxmox-watchtower-manager](https://github.com/anyblabla/proxmox-watchtower-manager)" target="_blank">anyblabla/proxmox-watchtower-manager</a>
* **Gitea:** <a href="[https://gitea.blablalinux.be/blablalinux/proxmox-watchtower-manager](https://gitea.blablalinux.be/blablalinux/proxmox-watchtower-manager)" target="_blank">gitea.blablalinux.be/blablalinux/proxmox-watchtower-manager</a>

---

## 🛠️ Installation and Setup

These steps assume you are connected via **SSH** to your **Proxmox host**.

### 1. Create the scripts directory

```bash
sudo mkdir -p /root/scripts

```

### 2. Make the scripts executable

After creating the files (see source codes below), don't forget to apply the permissions:

```bash
sudo chmod +x /root/scripts/manage_watchtower.sh
sudo chmod +x /root/scripts/manage_watchtower_all.sh

```

---

## 🧭 Scripts Comparison

| Feature | **Script 1: Standard** | **Script 2: Full Maintenance** |
| --- | --- | --- |
| **Target** | **Running** LXCs only. | **All** LXCs (Running & Stopped). |
| **Filtering** | Auto-detection (if Docker is running). | Only if the **`watchtower`** tag is present. |
| **State Management** | Does not change state. | Starts the LXC, acts, then **shuts it down again**. |
| **Reliability** | Immediate execution. | Waits for Docker to be ready (Wait-loop). |

---

## 📜 Script 1: Standard (`manage_watchtower.sh`)

This script is ideal for quick changes on production services currently online.

<details>
<summary>👉 Click to view the Standard Script source code</summary>

```bash
#!/bin/bash
# ==============================================================================
# Script: manage_watchtower.sh
# Description: Centralized Watchtower management for LXC (running only).
# Author: Amaury aka BlablaLinux
# Website: https://blablalinux.be
# Wiki: https://wiki.blablalinux.be/fr/script-gestion-watchtower
# License: GPL-3.0
# Version: 1.0.0
# ==============================================================================

MENU="
===============================================
    Managing Watchtower in LXC containers
===============================================
 [1] 🔍 View current Watchtower status
 [2] 🚀 Start Watchtower
 [3] 🛑 Stop Watchtower
 [4] 🔁 Restart Watchtower
 [5] 📂 View editable content of docker-compose.yml
 [6] 🔄 Set restart policy (always/none)
 [7] ✏️  Modify WATCHTOWER_NO_STARTUP_MESSAGE
 [8] ✏️  Modify WATCHTOWER_CLEANUP
 [9] 📅 Modify random schedule (2 PM-8 PM)
 [10] 📅 Fix the same schedule for all
 [11] ✏️  Modify WATCHTOWER_TIMEOUT
 [12] 🖼️  Modify Docker image
 [13] 🧹 Clean all images (prune -a)
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
        [ -n "$compose_file" ] && pct exec "$lxc_id" -- sh -c "grep -E 'image:|restart:|WATCHTOWER_NO_STARTUP_MESSAGE|WATCHTOWER_CLEANUP|WATCHTOWER_SCHEDULE|WATCHTOWER_TIMEOUT' $compose_file"
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
    read -rp "Image (e.g., containrrr/watchtower:latest): " img
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
    read -rp "Confirm prune -a on ALL active LXCs? (yes/no): " conf
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
        12) set_watchtower_image ;;
        13) prune_docker_images ;;
        [Qq]) exit ;;
    esac
done

```

</details>

---

## 🚀 Script 2: Full Maintenance (`manage_watchtower_all.sh`)

This script is designed for mass maintenance. **It will start shut down containers** that have the `watchtower` tag, apply your changes, and then shut them down again.

### Using Tags

For this script to process a container, you must add the **watchtower** tag in the Proxmox interface (or via `pct set ID --tags watchtower`).

<details>
<summary>👉 Click to view the Maintenance Script source code</summary>

```bash
#!/bin/bash
# ==============================================================================
# Script: manage_watchtower_all.sh
# Description: Watchtower management for all LXCs (All states) via Tags.
# Features: Auto-start/stop LXC, Tag filtering (watchtower), Docker wait-loop.
# Author: Amaury aka BlablaLinux
# Website: https://blablalinux.be
# Wiki: https://wiki.blablalinux.be/fr/script-gestion-watchtower
# License: GPL-3.0
# Version: 1.1.0
# ==============================================================================

MENU="
===============================================
    Watchtower Management - TOTAL MAINTENANCE (Tags)
===============================================
 [1] 🔍 View current Watchtower status
 [2] 🚀 Start Watchtower
 [3] 🛑 Stop Watchtower
 [4] 🔁 Restart Watchtower
 [5] 📂 View docker-compose.yml content
 [6] 🔄 Set restart policy (always/none)
 [7] ✏️  Modify WATCHTOWER_NO_STARTUP_MESSAGE
 [8] ✏️  Modify WATCHTOWER_CLEANUP
 [9] 📅 Modify random schedule (2 PM-8 PM)
 [10] 📅 Fix the same schedule for all
 [11] ✏️  Modify WATCHTOWER_TIMEOUT
 [12] 🖼️  Modify Docker image
 [13] 🧹 Clean all images (prune -a)
 [Q] ❌ Quit
"

run_action_on_all() {
    local action_func=$1
    for lxc_id in $(pct list | awk 'NR>1{print $1}'); do
        tags=$(pct config "$lxc_id" | grep "^tags:" | awk '{print $2}')
        if [[ "$tags" =~ "watchtower" ]]; then
            initial_status=$(pct status "$lxc_id" | awk '{print $2}')
            hostname=$(pct config "$lxc_id" | grep "^hostname:" | awk '{print $2}')
            echo "--- Processing LXC $lxc_id ($hostname) ---"
        else
            continue
        fi

        was_stopped=false
        if [ "$initial_status" == "stopped" ]; then
            echo "⚡ Starting LXC..."
            pct start "$lxc_id"
            was_stopped=true
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
                echo -e "\n❌ Docker unreachable. Skipping to next."
                pct stop "$lxc_id"
                continue
            fi
        fi

        if pct exec "$lxc_id" -- docker ps >/dev/null 2>&1; then
            $action_func "$lxc_id"
        else
            echo "🚫 Error: Docker not ready."
        fi

        if [ "$was_stopped" = true ]; then
            echo "💤 Returning to stopped state..."
            pct stop "$lxc_id"
        fi
    done
    read -rp "Done. Press [Enter]..."
}

_status() {
    compose_file=$(find_watchtower_compose "$1")
    [ -n "$compose_file" ] && pct exec "$1" -- docker ps --filter name=watchtower || echo "No compose file."
}
_start() {
    compose_file=$(find_watchtower_compose "$1")
    [ -n "$compose_file" ] && { dir=$(dirname "$compose_file"); pct exec "$1" -- sh -c "cd $dir && docker compose up -d"; echo "🚀 Launched."; }
}
_stop() { pct exec "$1" -- docker stop watchtower >/dev/null 2>&1 && echo "🛑 Stopped."; }
_restart() {
    compose_file=$(find_watchtower_compose "$1")
    [ -n "$compose_file" ] && { dir=$(dirname "$compose_file"); pct exec "$1" -- sh -c "cd $dir && docker compose down && docker compose up -d"; echo "🔁 Restarted."; }
}
_view() {
    compose_file=$(find_watchtower_compose "$1")
    [ -n "$compose_file" ] && pct exec "$1" -- sh -c "grep -E 'image:|restart:|WATCHTOWER_NO_STARTUP_MESSAGE|WATCHTOWER_CLEANUP|WATCHTOWER_SCHEDULE|WATCHTOWER_TIMEOUT' $compose_file"
}
_modify_key() {
    compose_file=$(find_watchtower_compose "$1")
    if [ -n "$compose_file" ]; then
        pct exec "$1" -- sed -i "s|^\s*-\s*$GLOBAL_KEY=.*|      - $GLOBAL_KEY=$GLOBAL_VAL|" "$compose_file"
        dir=$(dirname "$compose_file")
        pct exec "$1" -- sh -c "cd $dir && docker compose down && docker compose up -d"
        echo "✅ $GLOBAL_KEY updated."
    fi
}
_set_image() {
    compose_file=$(find_watchtower_compose "$1")
    if [ -n "$compose_file" ]; then
        pct exec "$1" -- sed -i "s#^[[:space:]]*image: .*#    image: $GLOBAL_VAL#" "$compose_file"
        dir=$(dirname "$compose_file")
        pct exec "$1" -- sh -c "cd $dir && docker compose down && docker compose up -d"
    fi
}
_random_sched() {
    compose_file=$(find_watchtower_compose "$1")
    if [ -n "$compose_file" ]; then
        hour=$((RANDOM % 7 + 14)) ; minute=$((RANDOM % 12 * 5)) ; schedule="0 $minute $hour ? * 5"
        pct exec "$1" -- sed -i "s|^\s*-\s*WATCHTOWER_SCHEDULE=.*|      - WATCHTOWER_SCHEDULE=$schedule|" "$compose_file"
        dir=$(dirname "$compose_file")
        pct exec "$1" -- sh -c "cd $dir && docker compose down && docker compose up -d"
        echo "✅ Schedule set: $schedule"
    fi
}
_prune() { echo "🧹 Pruning images..."; pct exec "$1" -- docker image prune -a -f; }
_set_policy() {
    compose_file=$(find_watchtower_compose "$1")
    if [ -n "$compose_file" ]; then
        pct exec "$1" -- sed -i "s/^[[:space:]]*restart: .*/    restart: $GLOBAL_VAL/" "$compose_file"
        dir=$(dirname "$compose_file")
        pct exec "$1" -- sh -c "cd $dir && docker compose down && docker compose up -d"
    fi
}
find_watchtower_compose() { timeout 5s pct exec "$1" -- find /root -type f -path "*/watchtower/docker-compose.yml" 2>/dev/null | head -n1; }

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
        12) read -rp "Image: " GLOBAL_VAL ; run_action_on_all _set_image ;;
        13) read -rp "Confirm prune (yes/no): " conf ; [[ "$conf" =~ ^[Yy][Ee][Ss]$ ]] && run_action_on_all _prune ;;
        [Qq]) exit ;;
    esac
done

```

</details>

---

## 📘 Additional Notes

* To easily install Docker on your LXCs, check out our dedicated page: <a href="[https://wiki.blablalinux.be/fr/docker-portainer-lxc-debian-proxmox](https://wiki.blablalinux.be/fr/docker-portainer-lxc-debian-proxmox)" target="_blank">Docker/Portainer on Debian/Proxmox</a>.
---
title: Gestion de Watchtower dans les conteneurs LXC
description: Cette page décrit le script utilisé pour gérer Watchtower dans des conteneurs LXC fonctionnant sur Proxmox VE. Il permet de vérifier l’état, démarrer, arrêter, redémarrer Watchtower et modifier ses configurations automatiquement.
published: true
date: 2026-05-23T15:02:35.281Z
tags: docker, lxc, proxmox, script, watchtower, pve, compose
editor: markdown
dateCreated: 2025-11-06T18:26:43.925Z
---

## Introduction

**Watchtower** est un outil qui surveille vos conteneurs Docker et les met à jour automatiquement. Cette page documente deux méthodes pour gérer vos instances Watchtower déployées dans des conteneurs LXC directement depuis l'hôte Proxmox.

📺 **Démonstration :** Retrouvez la vidéo de démonstration de ce script sur l'instance Mastodon de Blabla Linux : [https://mastodon.blablalinux.be/@blablalinux/115826788636738220](https://mastodon.blablalinux.be/@blablalinux/115826788636738220)

Ces scripts permettent de :

* Identifier les **LXC** contenant **Docker**.
* Trouver les fichiers `docker-compose.yml` de Watchtower.
* **Voir et modifier les options essentielles** (Planning, Nettoyage, URL de notification, Image, etc.).
* **Nettoyer les images** Docker non utilisées (`prune`).

> **Note technique :** Ces scripts s'exécutent sur l'**hôte Proxmox**. Ils utilisent la commande `pct exec [ID]` pour piloter Docker à l'intérieur des conteneurs sans avoir à s'y connecter individuellement.

---

## 🔗 Dépôts Officiels

Retrouvez les sources et contribuez au projet sur nos dépôts (liens externes) :

* **GitHub :** <a href="[https://github.com/anyblabla/proxmox-watchtower-manager](https://github.com/anyblabla/proxmox-watchtower-manager)" target="_blank">anyblabla/proxmox-watchtower-manager</a>
* **Gitea :** <a href="[https://gitea.blablalinux.be/blablalinux/proxmox-watchtower-manager](https://gitea.blablalinux.be/blablalinux/proxmox-watchtower-manager)" target="_blank">gitea.blablalinux.be/blablalinux/proxmox-watchtower-manager</a>

---

## 🛠️ Installation et Préparation

Ces étapes supposent que vous êtes connecté en **SSH** à votre **hôte Proxmox**.

### 1. Créer le répertoire de scripts

```bash
sudo mkdir -p /root/scripts

```

### 2. Rendre les scripts exécutables

Après avoir créé les fichiers (voir les codes sources plus bas), n'oubliez pas d'appliquer les permissions :

```bash
sudo chmod +x /root/scripts/manage_watchtower.sh
sudo chmod +x /root/scripts/manage_watchtower_all.sh

```

---

## 🧭 Comparaison des deux scripts

| Fonctionnalité | **Script 1 : Standard** | **Script 2 : Maintenance Intégrale** |
| --- | --- | --- |
| **Cible** | LXC **allumés** uniquement. | **Tous** les LXC (Allumés & Éteints). |
| **Filtrage** | Détection auto (si Docker tourne). | Uniquement si l'étiquette exacte **`watchtower`** est présente. |
| **Gestion d'état** | Ne change rien. | Allume le LXC, agit, puis **le rééteint**. |
| **Fiabilité** | Exécution immédiate. | Attend que Docker soit prêt (Wait-loop). |

---

## 📜 Script 1 : Standard (`manage_watchtower.sh`)

Ce script est idéal pour des modifications rapides sur vos services en production actuellement en ligne.

```bash
#!/bin/bash
# ==============================================================================
# Script: manage_watchtower.sh
# Description: Gestion centralisée de Watchtower pour LXC (allumés uniquement).
# Author: Amaury aka BlablaLinux
# Website: https://blablalinux.be
# Wiki: https://wiki.blablalinux.be/fr/script-gestion-watchtower
# License: GPL-3.0
# Version: 1.1.0
# ==============================================================================

MENU="
===============================================
    Gestion de Watchtower dans les conteneurs LXC
===============================================
 [1] 🔍 Voir l’état actuel de Watchtower
 [2] 🚀 Démarrer Watchtower
 [3] 🛑 Arrêter Watchtower
 [4] 🔁 Redémarrer Watchtower
 [5] 📂 Voir le contenu modifiable du docker-compose.yml
 [6] 🔄 Définir restart policy (always/none)
 [7] ✏️  Modifier WATCHTOWER_NO_STARTUP_MESSAGE
 [8] ✏️  Modifier WATCHTOWER_CLEANUP
 [9] 📅 Modifier le schedule aléatoire (14h-20h)
 [10] 📅 Fixer le même schedule pour tous
 [11] ✏️  Modifier WATCHTOWER_TIMEOUT
 [12] ✏️  Modifier WATCHTOWER_NOTIFICATION_GOTIFY_URL
 [13] 🖼️  Modifier l'image Docker
 [14] 🧹 Nettoyer toutes les images (prune -a)
 [Q] ❌ Quitter
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
        [ -n "$compose_file" ] && pct exec "$lxc_id" -- docker ps --filter name=watchtower || echo "Non trouvé."
    done
    read -rp "Appuyez sur [Entrée]..."
}

start_watchtower() {
    for lxc_id in $(get_running_docker_lxc); do
        compose_file=$(find_watchtower_compose "$lxc_id")
        if [ -n "$compose_file" ]; then
            dir=$(dirname "$compose_file")
            pct exec "$lxc_id" -- sh -c "cd $dir && docker compose up -d"
            echo "🚀 Démarré dans LXC $lxc_id"
        fi
    done
    read -rp "Terminé. [Entrée]..."
}

stop_watchtower() {
    for lxc_id in $(get_running_docker_lxc); do
        pct exec "$lxc_id" -- docker stop watchtower >/dev/null 2>&1 && echo "🛑 Arrêté dans LXC $lxc_id"
    done
    read -rp "Terminé. [Entrée]..."
}

restart_watchtower() {
    for lxc_id in $(get_running_docker_lxc); do
        compose_file=$(find_watchtower_compose "$lxc_id")
        if [ -n "$compose_file" ]; then
            dir=$(dirname "$compose_file")
            pct exec "$lxc_id" -- sh -c "cd $dir && docker compose down && docker compose up -d"
            echo "🔁 Redémarré dans LXC $lxc_id"
        fi
    done
    read -rp "Terminé. [Entrée]..."
}

view_compose() {
    for lxc_id in $(get_running_docker_lxc); do
        compose_file=$(find_watchtower_compose "$lxc_id")
        echo "→ LXC $lxc_id"
        [ -n "$compose_file" ] && pct exec "$lxc_id" -- sh -c "grep -E 'image:|restart:|WATCHTOWER_NO_STARTUP_MESSAGE|WATCHTOWER_CLEANUP|WATCHTOWER_SCHEDULE|WATCHTOWER_TIMEOUT|WATCHTOWER_NOTIFICATION_GOTIFY_URL' $compose_file"
    done
    read -rp "Appuyez sur [Entrée]..."
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
            echo "✅ $key mis à jour dans LXC $lxc_id"
        fi
    done
}

set_restart_policy() {
    read -rp "Policy (always/none) : " new_policy
    for lxc_id in $(get_running_docker_lxc); do
        compose_file=$(find_watchtower_compose "$lxc_id")
        if [ -n "$compose_file" ]; then
            pct exec "$lxc_id" -- sed -i "s/^[[:space:]]*restart: .*/    restart: $new_policy/" "$compose_file"
            dir=$(dirname "$compose_file")
            pct exec "$lxc_id" -- sh -c "cd $dir && docker compose down && docker compose up -d"
            echo "✅ Policy $new_policy dans LXC $lxc_id"
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
            echo "✅ Schedule $schedule pour LXC $lxc_id"
        fi
    done
}

set_watchtower_image() {
    read -rp "Image (ex: containrrr/watchtower:latest): " img
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
    read -rp "Confirmer prune -a sur TOUS les LXC actifs? (oui/non): " conf
    if [[ "$conf" =~ ^[Oo][Uu][Ii]$ ]]; then
        for lxc_id in $(get_running_docker_lxc); do
            pct exec "$lxc_id" -- docker image prune -a -f
        done
    fi
}

while true; do
    clear ; echo "$MENU" ; read -rp "Choix : " choice
    case $choice in
        1) status_watchtower ;;
        2) start_watchtower ;;
        3) stop_watchtower ;;
        4) restart_watchtower ;;
        5) view_compose ;;
        6) set_restart_policy ;;
        7) read -rp "true/false : " v; modify_key_restart "WATCHTOWER_NO_STARTUP_MESSAGE" "$v" ;;
        8) read -rp "true/false : " v; modify_key_restart "WATCHTOWER_CLEANUP" "$v" ;;
        9) random_schedule ;;
        10) read -rp "Cron : " v; modify_key_restart "WATCHTOWER_SCHEDULE" "$v" ;;
        11) read -rp "Timeout : " v; modify_key_restart "WATCHTOWER_TIMEOUT" "$v" ;;
        12) read -rp "Nouvelle URL Gotify : " v; modify_key_restart "WATCHTOWER_NOTIFICATION_GOTIFY_URL" "$v" ;;
        13) set_watchtower_image ;;
        14) prune_docker_images ;;
        [Qq]) exit ;;
    esac
done

```

---

## 🚀 Script 2 : Maintenance Intégrale (`manage_watchtower_all.sh`)

Ce script est conçu pour la maintenance de masse. **Il démarrera les conteneurs éteints** portant exclusivement le tag `watchtower`, appliquera vos changements, puis les éteindra à nouveau.

### Utilisation des Tags

Pour que ce script traite un conteneur, vous devez lui ajouter le tag **watchtower** dans l'interface Proxmox (ou via `pct set ID --tags watchtower`). Le script ignore volontairement les tags dérivés comme `watchtower-custom`.

```bash
#!/bin/bash
# ==============================================================================
# Script: manage_watchtower_all.sh
# Description: Gestion de Watchtower pour tous les LXC (All states) via Tags.
# Features: Auto-start/stop LXC, Tag filtering (watchtower), Docker wait-loop.
# Author: Amaury aka BlablaLinux
# Website: https://blablalinux.be
# Wiki: https://wiki.blablalinux.be/fr/script-gestion-watchtower
# License: GPL-3.0
# Version: 1.2.0
# ==============================================================================

MENU="
===============================================
   Gestion Watchtower - MAINTENANCE TOTALE (Tags)
===============================================
 [1] 🔍 Voir l’état actuel de Watchtower
 [2] 🚀 Démarrer Watchtower
 [3] 🛑 Arrêter Watchtower
 [4] 🔁 Redémarrer Watchtower
 [5] 📂 Voir le contenu du docker-compose.yml
 [6] 🔄 Définir restart policy (always/none)
 [7] ✏️  Modifier WATCHTOWER_NO_STARTUP_MESSAGE
 [8] ✏️  Modifier WATCHTOWER_CLEANUP
 [9] 📅 Modifier le schedule aléatoire (14h-20h)
 [10] 📅 Fixer le même schedule pour tous
 [11] ✏️  Modifier WATCHTOWER_TIMEOUT
 [12] ✏️  Modifier WATCHTOWER_NOTIFICATION_GOTIFY_URL
 [13] 🖼️  Modifier l'image Docker
 [14] 🧹 Nettoyer toutes les images (prune -a)
 [Q] ❌ Quitter
"

run_action_on_all() {
    local action_func=$1
    for lxc_id in $(pct list | awk 'NR>1{print $1}'); do
        tags=$(pct config "$lxc_id" | grep "^tags:" | awk '{print $2}')
        
        # --- CORRECTION FILTRAGE PAR MOT ENTIER ---
        # On ne traite QUE si le tag exact "watchtower" est présent, évite d'inclure "watchtower-custom"
        if [[ " $tags " =~ " watchtower " ]]; then
            initial_status=$(pct status "$lxc_id" | awk '{print $2}')
            hostname=$(pct config "$lxc_id" | grep "^hostname:" | awk '{print $2}')
            echo "--- Traitement LXC $lxc_id ($hostname) ---"
        else
            continue
        fi

        was_stopped=false
        if [ "$initial_status" == "stopped" ]; then
            echo "⚡ Démarrage du LXC..."
            pct start "$lxc_id"
            was_stopped=true
            echo -n "⏳ Attente Docker..."
            success=false
            for i in {1..15}; do
                if pct exec "$lxc_id" -- docker ps >/dev/null 2>&1; then
                    echo " OK !"
                    success=true
                    break
                fi
                echo -n "."
                sleep 1
            done
            if [ "$success" = false ]; then
                echo -e "\n❌ Docker injoignable. Passage au suivant."
                pct stop "$lxc_id"
                continue
            fi
        fi

        if pct exec "$lxc_id" -- docker ps >/dev/null 2>&1; then
            $action_func "$lxc_id"
        else
            echo "🚫 Erreur : Docker non prêt."
        fi

        if [ "$was_stopped" = true ]; then
            echo "💤 Retour à l'état éteint..."
            pct stop "$lxc_id"
        fi
    done
    read -rp "Terminé. Appuyez sur [Entrée]..."
}

_status() {
    compose_file=$(find_watchtower_compose "$1")
    [ -n "$compose_file" ] && pct exec "$1" -- docker ps --filter name=watchtower || echo "Pas de compose."
}
_start() {
    compose_file=$(find_watchtower_compose "$1")
    [ -n "$compose_file" ] && { dir=$(dirname "$compose_file"); pct exec "$1" -- sh -c "cd $dir && docker compose up -d"; echo "🚀 Lancé."; }
}
_stop() { pct exec "$1" -- docker stop watchtower >/dev/null 2>&1 && echo "🛑 Arrêté."; }
_restart() {
    compose_file=$(find_watchtower_compose "$1")
    [ -n "$compose_file" ] && { dir=$(dirname "$compose_file"); pct exec "$1" -- sh -c "cd $dir && docker compose down && docker compose up -d"; echo "🔁 Redémarré."; }
}
_view() {
    compose_file=$(find_watchtower_compose "$1")
    [ -n "$compose_file" ] && pct exec "$1" -- sh -c "grep -E 'image:|restart:|WATCHTOWER_NO_STARTUP_MESSAGE|WATCHTOWER_CLEANUP|WATCHTOWER_SCHEDULE|WATCHTOWER_TIMEOUT|WATCHTOWER_NOTIFICATION_GOTIFY_URL' $compose_file"
}
_modify_key() {
    compose_file=$(find_watchtower_compose "$1")
    if [ -n "$compose_file" ]; then
        pct exec "$1" -- sed -i "s|^\s*-\s*$GLOBAL_KEY=.*|      - $GLOBAL_KEY=$GLOBAL_VAL|" "$compose_file"
        dir=$(dirname "$compose_file")
        pct exec "$1" -- sh -c "cd $dir && docker compose down && docker compose up -d"
        echo "✅ $GLOBAL_KEY mis à jour."
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
        echo "✅ Schedule fixé : $schedule"
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
    clear ; echo "$MENU" ; read -rp "Votre choix : " choice
    case $choice in
        1) run_action_on_all _status ;;
        2) run_action_on_all _start ;;
        3) run_action_on_all _stop ;;
        4) run_action_on_all _restart ;;
        5) run_action_on_all _view ;;
        6) read -rp "Policy (always/none) : " GLOBAL_VAL ; run_action_on_all _set_policy ;;
        7) GLOBAL_KEY="WATCHTOWER_NO_STARTUP_MESSAGE" ; read -rp "true/false : " GLOBAL_VAL ; run_action_on_all _modify_key ;;
        8) GLOBAL_KEY="WATCHTOWER_CLEANUP" ; read -rp "true/false : " GLOBAL_VAL ; run_action_on_all _modify_key ;;
        9) run_action_on_all _random_sched ;;
        10) GLOBAL_KEY="WATCHTOWER_SCHEDULE" ; read -rp "Cron : " GLOBAL_VAL ; run_action_on_all _modify_key ;;
        11) GLOBAL_KEY="WATCHTOWER_TIMEOUT" ; read -rp "Valeur : " GLOBAL_VAL ; run_action_on_all _modify_key ;;
        12) GLOBAL_KEY="WATCHTOWER_NOTIFICATION_GOTIFY_URL" ; read -rp "Nouvelle URL Gotify : " GLOBAL_VAL ; run_action_on_all _modify_key ;;
        13) read -rp "Image : " GLOBAL_VAL ; run_action_on_all _set_image ;;
        14) read -rp "Confirmer prune (oui/non) : " conf ; [[ "$conf" =~ ^[Oo][Uu][Ii]$ ]] && run_action_on_all _prune ;;
        [Qq]) exit ;;
    esac
done

```

---

## 📘 Notes Additionnelles

* Pour installer Docker facilement sur vos LXC, consultez notre page dédiée : <a href="[https://wiki.blablalinux.be/fr/docker-portainer-lxc-debian-proxmox](https://wiki.blablalinux.be/fr/docker-portainer-lxc-debian-proxmox)" target="_blank">Docker/Portainer sur Debian/Proxmox</a>.
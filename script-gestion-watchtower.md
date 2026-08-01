---
title: Gestion de Watchtower dans les conteneurs LXC
description: Cette page décrit le script utilisé pour gérer Watchtower dans des conteneurs LXC fonctionnant sur Proxmox VE. Il permet de vérifier l’état, démarrer, arrêter, redémarrer Watchtower et modifier ses configurations automatiquement.
published: true
date: 2026-08-01T16:57:19.079Z
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
* **Migrer automatiquement** les notifications Gotify legacy vers le format Shoutrrr moderne (`WATCHTOWER_NOTIFICATION_URL`).
* **Nettoyer les images** Docker non utilisées (`prune`) et les anciennes sauvegardes de compose.

> **Note technique :** Ces scripts s'exécutent sur l'**hôte Proxmox**. Ils utilisent la commande `pct exec [ID]` pour piloter Docker à l'intérieur des conteneurs sans avoir à s'y connecter individuellement.

> **⚠️ À propos de la dépréciation Gotify :** depuis Watchtower v1.20+, les variables `WATCHTOWER_NOTIFICATION_GOTIFY_URL`, `WATCHTOWER_NOTIFICATION_GOTIFY_TOKEN` et `WATCHTOWER_NOTIFICATION_GOTIFY_TLS_SKIP_VERIFY` sont dépréciées au profit d'une URL Shoutrrr unique (`WATCHTOWER_NOTIFICATION_URL`). Elles seront supprimées avec la sortie de Watchtower v2. Le **Script 2** intègre une option de migration automatique (voir plus bas).

---

## 🔗 Dépôts Officiels

Retrouvez les sources et contribuez au projet sur nos dépôts (liens externes) :

* **GitHub :** [https://github.com/anyblabla/proxmox-watchtower-manager](https://github.com/anyblabla/proxmox-watchtower-manager)
* **Gitea :** [https://gitea.blablalinux.be/blablalinux/proxmox-watchtower-manager](https://gitea.blablalinux.be/blablalinux/proxmox-watchtower-manager)

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
| **Fiabilité** | Exécution immédiate. | Attend que Docker soit prêt (Wait-loop) + pause de stabilisation post-boot. |
| **Recherche du compose** | Chemin fixe `*/watchtower/docker-compose.yml`. | Recherche en deux temps (chemin ciblé puis repli par contenu), tolère les `.yaml` et les dossiers non nommés `watchtower`. |
| **Notifications Gotify** | Édition manuelle de l'ancienne variable uniquement. | Migration automatique vers Shoutrrr (`WATCHTOWER_NOTIFICATION_URL`) avec sauvegarde `.bak`. |
| **Nettoyage sauvegardes** | Non disponible. | Suppression ciblée des `.bkp` résiduels. |

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

### 🆕 Nouveautés de la v2.0.0

* **Migration Gotify → Shoutrrr (option 15)** : lit les anciennes variables `WATCHTOWER_NOTIFICATION_GOTIFY_URL` / `_TOKEN`, construit automatiquement l'URL Shoutrrr correspondante (`gotify://host[:port]/token`, avec `?disabletls=yes` ajouté si l'ancienne URL était en `http://`), supprime les clés dépréciées, insère `WATCHTOWER_NOTIFICATION_URL`, et redémarre le service. Une sauvegarde `.bak` du compose est créée avant toute modification. Si `WATCHTOWER_NOTIFICATIONS` référence un autre notifieur en plus de `gotify`, la ligne n'est pas touchée automatiquement (avertissement affiché pour vérification manuelle).
* **Nettoyage des sauvegardes (option 16)** : supprime les fichiers `*.bkp` résiduels dans le dossier du compose (sans confirmation). Les `.bak` générés par la migration ne sont volontairement pas concernés.
* **Recherche du compose plus fiable** : `find_watchtower_compose` tente d'abord une recherche rapide par chemin (`*watchtower*`, profondeur limitée à 4, `.yml`/`.yaml`), puis se replie sur une recherche par contenu (`grep -r` sur l'image `containrrr/watchtower` ou `nickfedor/watchtower`) si rien n'est trouvé — ce qui couvre aussi bien les dossiers nommés différemment que les LXC plus lents à répondre.
* **Pause de stabilisation post-boot** : après le démarrage d'un LXC éteint, un `sleep 3` est ajouté avant de scanner les fichiers, pour éviter les faux négatifs liés à un système d'exploitation encore en cours de démarrage.
* **Messages d'erreur explicites** : chaque action affiche désormais clairement `🚫 Compose Watchtower introuvable pour ce LXC.` en cas d'échec, au lieu de rester silencieuse.
* **Option 12 recentrée** : édite directement `WATCHTOWER_NOTIFICATION_URL` (le format Shoutrrr moderne) plutôt que l'ancienne variable Gotify.

```bash
#!/bin/bash
# ==============================================================================
# Script: manage_watchtower_all.sh
# Description: Gestion de Watchtower pour tous les LXC (All states) via Tags.
# Features: Auto-start/stop LXC, Tag filtering (watchtower), Docker wait-loop,
#           migration Gotify legacy -> Shoutrrr, nettoyage des sauvegardes .bkp.
# Author: Amaury aka BlablaLinux
# Website: https://blablalinux.be
# Wiki: https://wiki.blablalinux.be/fr/script-gestion-watchtower
# License: GPL-3.0
# Version: 2.0.0
# ==============================================================================

MENU="
===============================================
   Gestion Watchtower - TAG 'watchtower' UNIQUE
===============================================
 [1] 🔍 Voir l'état actuel de Watchtower
 [2] 🚀 Démarrer Watchtower
 [3] 🛑 Arrêter Watchtower
 [4] 🔁 Redémarrer Watchtower
 [5] 📂 Voir le contenu du docker-compose.yml
 [6] 🔄 Définir restart policy (always/none)
 [7] ✏️  Modifier WATCHTOWER_NO_STARTUP_MESSAGE
 [8] ✏️  Modifier WATCHTOWER_CLEANUP
 [9] 📅 Modifier le schedule aléatoire (14h-20h le samedi)
 [10] 📅 Fixer le même schedule pour tous
 [11] ✏️  Modifier WATCHTOWER_TIMEOUT
 [12] ✏️  Modifier WATCHTOWER_NOTIFICATION_URL (Shoutrrr)
 [13] 🖼️  Modifier l'image Docker
 [14] 🧹 Nettoyer toutes les images (prune -a)
 [15] 🔀 Migrer Gotify legacy -> Shoutrrr (WATCHTOWER_NOTIFICATION_URL)
 [16] 🗑️  Supprimer les anciennes sauvegardes (*.bkp)
 [Q] ❌ Quitter
"

# --- FONCTION MAÎTRESSE (Filtrage par Tag et Gestion d'état) ---

run_action_on_all() {
    local action_func=$1

    for lxc_id in $(pct list | awk 'NR>1{print $1}'); do
        # Récupération des tags du LXC
        tags=$(pct config "$lxc_id" | grep "^tags:" | awk '{print $2}')

        # --- CORRECTION FILTRAGE PAR MOT ENTIER ---
        if echo "$tags" | grep -qE "(^|;)watchtower(;|$)" ; then
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

            # Boucle d'attente Docker
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

        # Exécution de la commande
        if pct exec "$lxc_id" -- docker ps >/dev/null 2>&1; then
            if [ "$was_stopped" = true ]; then
                # Petite pause : juste après le boot, le FS/IO peut encore être chargé
                # (démarrage des services), on laisse le système se stabiliser avant
                # de scanner les fichiers.
                sleep 3
            fi
            $action_func "$lxc_id"
        else
            echo "🚫 Erreur : Docker non prêt."
        fi

        # Retour à l'état initial
        if [ "$was_stopped" = true ]; then
            echo "💤 Retour à l'état éteint..."
            pct stop "$lxc_id"
        fi
    done
    read -rp "Terminé. Appuyez sur [Entrée]..."
}

# --- FONCTIONS UNITAIRES ---

_status() {
    compose_file=$(find_watchtower_compose "$1")
    if [ -n "$compose_file" ]; then
        pct exec "$1" -- docker ps --filter name=watchtower
    else
        echo "🚫 Compose Watchtower introuvable pour ce LXC."
    fi
}
_start() {
    compose_file=$(find_watchtower_compose "$1")
    if [ -n "$compose_file" ]; then
        dir=$(dirname "$compose_file")
        pct exec "$1" -- sh -c "cd $dir && docker compose up -d"
        echo "🚀 Lancé."
    else
        echo "🚫 Compose Watchtower introuvable pour ce LXC."
    fi
}
_stop() { pct exec "$1" -- docker stop watchtower >/dev/null 2>&1 && echo "🛑 Arrêté." || echo "🚫 Conteneur watchtower introuvable/déjà arrêté."; }
_restart() {
    compose_file=$(find_watchtower_compose "$1")
    if [ -n "$compose_file" ]; then
        dir=$(dirname "$compose_file")
        pct exec "$1" -- sh -c "cd $dir && docker compose down && docker compose up -d"
        echo "🔁 Redémarré."
    else
        echo "🚫 Compose Watchtower introuvable pour ce LXC."
    fi
}
_view() {
    compose_file=$(find_watchtower_compose "$1")
    if [ -n "$compose_file" ]; then
        echo "📄 $compose_file"
        pct exec "$1" -- sh -c "grep -E 'image:|restart:|WATCHTOWER_NO_STARTUP_MESSAGE|WATCHTOWER_CLEANUP|WATCHTOWER_SCHEDULE|WATCHTOWER_TIMEOUT|WATCHTOWER_NOTIFICATION_GOTIFY_URL|WATCHTOWER_NOTIFICATION_GOTIFY_TOKEN|WATCHTOWER_NOTIFICATIONS|WATCHTOWER_NOTIFICATION_URL' $compose_file"
    else
        echo "🚫 Compose Watchtower introuvable pour ce LXC."
    fi
}
_modify_key() {
    compose_file=$(find_watchtower_compose "$1")
    if [ -n "$compose_file" ]; then
        pct exec "$1" -- sed -i "s|^\s*-\s*$GLOBAL_KEY=.*|      - $GLOBAL_KEY=$GLOBAL_VAL|" "$compose_file"
        dir=$(dirname "$compose_file")
        pct exec "$1" -- sh -c "cd $dir && docker compose down && docker compose up -d"
        echo "✅ $GLOBAL_KEY mis à jour."
    else
        echo "🚫 Compose Watchtower introuvable pour ce LXC."
    fi
}
_set_image() {
    compose_file=$(find_watchtower_compose "$1")
    if [ -n "$compose_file" ]; then
        pct exec "$1" -- sed -i "s#^[[:space:]]*image: .*#    image: $GLOBAL_VAL#" "$compose_file"
        dir=$(dirname "$compose_file")
        pct exec "$1" -- sh -c "cd $dir && docker compose down && docker compose up -d"
        echo "✅ Image mise à jour."
    else
        echo "🚫 Compose Watchtower introuvable pour ce LXC."
    fi
}
_random_sched() {
    compose_file=$(find_watchtower_compose "$1")
    if [ -n "$compose_file" ]; then
        hour=$((RANDOM % 7 + 14)) ; minute=$((RANDOM % 12 * 5)) ; schedule="0 $minute $hour ? * 6"
        pct exec "$1" -- sed -i "s|^\s*-\s*WATCHTOWER_SCHEDULE=.*|      - WATCHTOWER_SCHEDULE=$schedule|" "$compose_file"
        dir=$(dirname "$compose_file")
        pct exec "$1" -- sh -c "cd $dir && docker compose down && docker compose up -d"
        echo "✅ Schedule fixé : $schedule"
    else
        echo "🚫 Compose Watchtower introuvable pour ce LXC."
    fi
}
_prune() { echo "🧹 Pruning images..."; pct exec "$1" -- docker image prune -a -f; }
_set_policy() {
    compose_file=$(find_watchtower_compose "$1")
    if [ -n "$compose_file" ]; then
        pct exec "$1" -- sed -i "s/^[[:space:]]*restart: .*/    restart: $GLOBAL_VAL/" "$compose_file"
        dir=$(dirname "$compose_file")
        pct exec "$1" -- sh -c "cd $dir && docker compose down && docker compose up -d"
        echo "✅ Restart policy mise à jour."
    else
        echo "🚫 Compose Watchtower introuvable pour ce LXC."
    fi
}
# Recherche en deux temps : rapide par chemin, puis repli par contenu.
find_watchtower_compose() {
    local lxc_id="$1"
    local result

    # 1) Recherche rapide et ciblée : chemin contenant "watchtower", profondeur limitée.
    result=$(timeout 8s pct exec "$lxc_id" -- sh -c "find /root /opt /home -maxdepth 4 -type f \( -iname 'docker-compose.yml' -o -iname 'docker-compose.yaml' \) -path '*watchtower*' 2>/dev/null | head -n1")
    if [ -n "$result" ]; then
        echo "$result"
        return
    fi

    # 2) Repli : scan large par contenu (plus lent), au cas où le dossier ne contient
    #    pas "watchtower" dans son nom. Timeout plus généreux.
    timeout 20s pct exec "$lxc_id" -- sh -c "grep -rlE 'image:[[:space:]]*(containrrr|nickfedor)/watchtower' /root /opt /home 2>/dev/null | head -n1"
}

# --- MIGRATION GOTIFY LEGACY -> SHOUTRRR ---
#
# Lit WATCHTOWER_NOTIFICATION_GOTIFY_URL / _TOKEN, construit une URL
# gotify://host[:port]/token (avec ?disabletls=yes si l'URL d'origine est en http://),
# supprime les 3 anciennes clés (URL, TOKEN, TLS_SKIP_VERIFY) et l'entrée
# "gotify" de WATCHTOWER_NOTIFICATIONS si elle est seule, puis insère
# WATCHTOWER_NOTIFICATION_URL. Le fichier est sauvegardé en .bak avant modif.

_migrate_gotify() {
    lxc_id="$1"
    compose_file=$(find_watchtower_compose "$lxc_id")
    if [ -z "$compose_file" ]; then
        echo "🚫 Pas de compose trouvé, LXC ignoré."
        return
    fi

    url_line=$(pct exec "$lxc_id" -- grep -E 'WATCHTOWER_NOTIFICATION_GOTIFY_URL=' "$compose_file" 2>/dev/null)
    token_line=$(pct exec "$lxc_id" -- grep -E 'WATCHTOWER_NOTIFICATION_GOTIFY_TOKEN=' "$compose_file" 2>/dev/null)

    if [ -z "$url_line" ] || [ -z "$token_line" ]; then
        echo "ℹ️  Aucune config Gotify legacy trouvée (déjà migré ou non configuré). LXC ignoré."
        return
    fi

    # Détecter si un notifier autre que gotify est aussi déclaré (config plus complexe -> pas d'auto-suppression)
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

    echo "🔎 LXC $lxc_id : ancienne URL = $old_url"
    echo "   -> Nouvelle URL Shoutrrr : $new_url"

    # Sauvegarde
    pct exec "$lxc_id" -- cp "$compose_file" "${compose_file}.bak"

    # Suppression des anciennes clés
    pct exec "$lxc_id" -- sed -i '/WATCHTOWER_NOTIFICATION_GOTIFY_URL=/d' "$compose_file"
    pct exec "$lxc_id" -- sed -i '/WATCHTOWER_NOTIFICATION_GOTIFY_TOKEN=/d' "$compose_file"
    pct exec "$lxc_id" -- sed -i '/WATCHTOWER_NOTIFICATION_GOTIFY_TLS_SKIP_VERIFY=/d' "$compose_file"

    if [ "$notif_val" = "gotify" ]; then
        pct exec "$lxc_id" -- sed -i '/WATCHTOWER_NOTIFICATIONS=/d' "$compose_file"
    elif [ -n "$notif_val" ]; then
        echo "⚠️  WATCHTOWER_NOTIFICATIONS contient '$notif_val' (pas seulement gotify)."
        echo "    Cette ligne n'a PAS été supprimée automatiquement, vérifie-la manuellement."
    fi

    # Insertion de la nouvelle clé juste après la ligne "environment:"
    pct exec "$lxc_id" -- sed -i "/^\s*environment:/a\\      - WATCHTOWER_NOTIFICATION_URL=${new_url}" "$compose_file"

    dir=$(dirname "$compose_file")
    pct exec "$lxc_id" -- sh -c "cd $dir && docker compose down && docker compose up -d"
    echo "✅ Migration terminée pour LXC $lxc_id (sauvegarde : ${compose_file}.bak)"
}

# --- NETTOYAGE DES SAUVEGARDES (.bkp) ---
#
# Cherche, dans le même dossier que le docker-compose.yml trouvé, tout fichier
# se terminant par .bkp (anciennes sauvegardes manuelles) et les supprime.
# Les .bak (générés par l'option 15) ne sont volontairement PAS touchés.

_cleanup_backups() {
    lxc_id="$1"
    compose_file=$(find_watchtower_compose "$lxc_id")
    if [ -z "$compose_file" ]; then
        echo "🚫 Compose Watchtower introuvable pour ce LXC."
        return
    fi

    dir=$(dirname "$compose_file")
    backups=$(pct exec "$lxc_id" -- find "$dir" -maxdepth 1 -type f -name "*.bkp" 2>/dev/null)

    if [ -z "$backups" ]; then
        echo "ℹ️  Aucune sauvegarde .bkp trouvée dans $dir."
        return
    fi

    echo "🗂️  Sauvegardes .bkp trouvées dans $dir :"
    echo "$backups" | sed 's/^/   - /'

    pct exec "$lxc_id" -- find "$dir" -maxdepth 1 -type f -name "*.bkp" -delete
    echo "✅ Sauvegardes .bkp supprimées."
}

# --- BOUCLE MENU ---

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
        10) GLOBAL_KEY="WATCHTOWER_SCHEDULE" ; read -rp "Spring Cron : " GLOBAL_VAL ; run_action_on_all _modify_key ;;
        11) GLOBAL_KEY="WATCHTOWER_TIMEOUT" ; read -rp "Valeur : " GLOBAL_VAL ; run_action_on_all _modify_key ;;
        12) GLOBAL_KEY="WATCHTOWER_NOTIFICATION_URL" ; read -rp "Nouvelle URL Shoutrrr (ex: gotify://host:port/token?disabletls=yes) : " GLOBAL_VAL ; run_action_on_all _modify_key ;;
        13) read -rp "Image : " GLOBAL_VAL ; run_action_on_all _set_image ;;
        14) read -rp "Confirmer prune (oui/non) : " conf ; [[ "$conf" =~ ^[Oo][Uu][Ii]$ ]] && run_action_on_all _prune ;;
        15) read -rp "⚠️  Migrer tous les LXC 'watchtower' vers Shoutrrr ? (oui/non) : " conf ; [[ "$conf" =~ ^[Oo][Uu][Ii]$ ]] && run_action_on_all _migrate_gotify ;;
        16) run_action_on_all _cleanup_backups ;;
        [Qq]) exit ;;
        *) echo "Option invalide." ; sleep 1 ;;
    esac
done
```

---

## 🔀 Utilisation de la migration Gotify → Shoutrrr

Ordre recommandé pour migrer un LXC (ou vérifier ceux déjà migrés) :

1. **[5]** — état des lieux : affiche les variables actuelles (legacy Gotify ou déjà en Shoutrrr).
2. **[15]** — migration : bascule vers `WATCHTOWER_NOTIFICATION_URL`, supprime les anciennes clés, redémarre le conteneur, crée un `.bak` de sécurité.
3. **[5]** — vérification : confirme que `WATCHTOWER_NOTIFICATION_URL` est en place et que les anciennes clés ont disparu.
4. **[16]** — nettoyage : supprime les éventuels `.bkp` résiduels (sauvegardes manuelles antérieures). Le `.bak` créé à l'étape 2 est conservé.

---

## 📘 Notes Additionnelles

* Pour installer Docker facilement sur vos LXC, consultez notre page dédiée : [docker-portainer-lxc-debian-proxmox](/fr/docker-portainer-lxc-debian-proxmox).
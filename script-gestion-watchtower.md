---
title: Gestion de Watchtower dans les conteneurs LXC
description: Cette page décrit le script utilisé pour gérer Watchtower dans des conteneurs LXC fonctionnant sur Proxmox VE. Il permet de vérifier l’état, démarrer, arrêter, redémarrer Watchtower et modifier ses configurations automatiquement.
published: true
date: 2025-11-07T14:23:26.528Z
tags: docker, lxc, proxmox, script, watchtower, pve, compose
editor: markdown
dateCreated: 2025-11-06T18:26:43.925Z
---

## Introduction

Watchtower est un outil qui surveille vos conteneurs Docker et les met à jour automatiquement.
Ce script permet de :

* Identifier les LXC en ligne qui contiennent Docker.
* Trouver les fichiers `docker-compose.yml` de Watchtower.
* Voir et modifier les options essentielles de Watchtower.
* Redémarrer automatiquement les containers après modification.

Le script est adapté pour des LXC dont le répertoire Watchtower se trouve dans `/root` ou un sous-répertoire de `/root`.

---

## Menu du script

Lorsque vous lancez le script, le menu suivant apparaît :

```
===============================================
   Gestion de Watchtower dans les conteneurs LXC
===============================================
 [1] 🔍 Voir l’état actuel de Watchtower
 [2] 🚀 Démarrer Watchtower
 [3] 🛑 Arrêter Watchtower
 [4] 🔁 Redémarrer Watchtower
 [5] 📂 Voir le contenu modifiable du docker-compose.yml de Watchtower
 [6] 🔄 Basculer restart policy (always ↔ none)
 [7] ✏️  Modifier WATCHTOWER_NO_STARTUP_MESSAGE (true/false)
 [8] ✏️  Modifier WATCHTOWER_CLEANUP (true/false)
 [9] 📅 Modifier le schedule aléatoire (14h-20h, min multiples de 5)
 [10] 📅 Fixer le même schedule pour tous (6 champs, Spring Cron)
 [11] ✏️  Modifier WATCHTOWER_TIMEOUT
 [Q] ❌ Quitter
```

---

## Description des options

### [1] Voir l’état actuel de Watchtower

Affiche l’état des conteneurs Watchtower pour chaque LXC en ligne. Seuls les conteneurs actifs sont affichés.

### [2] Démarrer Watchtower

Démarre le conteneur Watchtower dans chaque LXC identifié.

### [3] Arrêter Watchtower

Arrête le conteneur Watchtower.

### [4] Redémarrer Watchtower

Redémarre le conteneur Watchtower pour appliquer d’éventuelles modifications.

### [5] Voir le contenu modifiable du `docker-compose.yml`

Affiche uniquement les lignes suivantes pour chaque LXC :

```yaml
restart: always
WATCHTOWER_NO_STARTUP_MESSAGE=false
WATCHTOWER_CLEANUP=true
WATCHTOWER_SCHEDULE=0 10 15 ? * 5
WATCHTOWER_TIMEOUT=30s
```

### [6] Basculer restart policy

Change la valeur `restart:` entre `always` et `none` et redémarre le conteneur automatiquement.

### [7] Modifier `WATCHTOWER_NO_STARTUP_MESSAGE`

Permet de définir `true` ou `false`. Après modification, le conteneur est redémarré automatiquement.

### [8] Modifier `WATCHTOWER_CLEANUP`

Permet de définir `true` ou `false`. Après modification, le conteneur est redémarré automatiquement.

### [9] Modifier le schedule aléatoire

Génère un schedule aléatoire unique pour chaque LXC (heures entre 14h et 20h, minutes multiples de 5) et redémarre le conteneur.

### [10] Fixer le même schedule pour tous

Permet de saisir un schedule au format Spring Cron (6 champs). Exemple :

```
0 0 16 ? * 5
```

Après modification, le conteneur est redémarré automatiquement.

### [11] Modifier `WATCHTOWER_TIMEOUT`

Permet de définir une valeur comme `30s`, `60s`, etc. Après modification, le conteneur est redémarré automatiquement.

---

## Script complet

```bash
#!/bin/bash
# Gestion complète de Watchtower dans LXC

MENU="
===============================================
   Gestion de Watchtower dans les conteneurs LXC
===============================================
 [1] 🔍 Voir l’état actuel de Watchtower
 [2] 🚀 Démarrer Watchtower
 [3] 🛑 Arrêter Watchtower
 [4] 🔁 Redémarrer Watchtower
 [5] 📂 Voir le contenu modifiable du docker-compose.yml de Watchtower
 [6] 🔄 Basculer restart policy (always ↔ none)
 [7] ✏️  Modifier WATCHTOWER_NO_STARTUP_MESSAGE (true/false)
 [8] ✏️  Modifier WATCHTOWER_CLEANUP (true/false)
 [9] 📅 Modifier le schedule aléatoire (14h-20h, min multiples de 5)
 [10] 📅 Fixer le même schedule pour tous (6 champs, Spring Cron)
 [11] ✏️  Modifier WATCHTOWER_TIMEOUT
 [Q] ❌ Quitter
"

# Obtenir les LXC en ligne avec Docker
get_running_docker_lxc() {
    pct list | awk 'NR>1 && $2=="running"{print $1}' | while read lxc; do
        if pct exec "$lxc" -- docker ps >/dev/null 2>&1; then
            echo "$lxc"
        fi
    done
}

# Trouver docker-compose.yml de watchtower avec timeout (5s)
find_watchtower_compose() {
    lxc_id=$1
    timeout 5s pct exec "$lxc_id" -- find /root -type f -path "*/watchtower/docker-compose.yml" 2>/dev/null | head -n1
}

# Afficher état Watchtower
status_watchtower() {
    for lxc_id in $(get_running_docker_lxc); do
        compose_file=$(find_watchtower_compose "$lxc_id")
        echo "→ LXC $lxc_id"
        if [ -n "$compose_file" ]; then
            pct exec "$lxc_id" -- docker ps --filter name=watchtower
        else
            echo "Pas de docker-compose.yml trouvé ou recherche expirée."
        fi
    done
    read -rp "Appuyez sur [Entrée] pour revenir au menu..."
}

# Démarrer Watchtower
start_watchtower() {
    for lxc_id in $(get_running_docker_lxc); do
        compose_file=$(find_watchtower_compose "$lxc_id")
        if [ -n "$compose_file" ]; then
            dir=$(dirname "$compose_file")
            pct exec "$lxc_id" -- sh -c "cd $dir && docker compose up -d"
            echo "🚀 Watchtower démarré dans LXC $lxc_id"
        else
            echo "Pas de docker-compose.yml trouvé ou recherche expirée pour LXC $lxc_id."
        fi
    done
    read -rp "Appuyez sur [Entrée] pour revenir au menu..."
}

# Arrêter Watchtower
stop_watchtower() {
    for lxc_id in $(get_running_docker_lxc); do
        compose_file=$(find_watchtower_compose "$lxc_id")
        if [ -n "$compose_file" ]; then
            pct exec "$lxc_id" -- docker stop watchtower >/dev/null 2>&1
            echo "🛑 Watchtower arrêté dans LXC $lxc_id"
        else
            echo "Pas de docker-compose.yml trouvé ou recherche expirée pour LXC $lxc_id."
        fi
    done
    read -rp "Appuyez sur [Entrée] pour revenir au menu..."
}

# Redémarrer Watchtower
restart_watchtower() {
    for lxc_id in $(get_running_docker_lxc); do
        compose_file=$(find_watchtower_compose "$lxc_id")
        if [ -n "$compose_file" ]; then
            dir=$(dirname "$compose_file")
            pct exec "$lxc_id" -- sh -c "cd $dir && docker compose down && docker compose up -d"
            echo "🔁 Watchtower redémarré dans LXC $lxc_id"
        else
            echo "Pas de docker-compose.yml trouvé ou recherche expirée pour LXC $lxc_id."
        fi
    done
    read -rp "Appuyez sur [Entrée] pour revenir au menu..."
}

# Voir le contenu modifiable du docker-compose.yml
view_compose() {
    for lxc_id in $(get_running_docker_lxc); do
        compose_file=$(find_watchtower_compose "$lxc_id")
        echo "→ LXC $lxc_id"
        if [ -n "$compose_file" ]; then
            pct exec "$lxc_id" -- sh -c "grep -E 'restart:|WATCHTOWER_NO_STARTUP_MESSAGE|WATCHTOWER_CLEANUP|WATCHTOWER_SCHEDULE|WATCHTOWER_TIMEOUT' $compose_file"
        else
            echo "Pas de docker-compose.yml trouvé ou recherche expirée pour LXC $lxc_id."
        fi
    done
    read -rp "Appuyez sur [Entrée] pour revenir au menu..."
}

# Modifier une clé dans docker-compose.yml et redémarrer
modify_key_restart() {
    key=$1
    new_value=$2
    for lxc_id in $(get_running_docker_lxc); do
        compose_file=$(find_watchtower_compose "$lxc_id")
        if [ -n "$compose_file" ]; then
            pct exec "$lxc_id" -- sed -i "s|^\s*-\s*$key=.*|      - $key=$new_value|" "$compose_file"
            dir=$(dirname "$compose_file")
            pct exec "$lxc_id" -- sh -c "cd $dir && docker compose down && docker compose up -d"
            echo "✅ $key mis à jour et Watchtower redémarré pour LXC $lxc_id"
        else
            echo "Pas de docker-compose.yml trouvé ou recherche expirée pour LXC $lxc_id."
        fi
    done
    read -rp "Appuyez sur [Entrée] pour revenir au menu..."
}

# Basculer restart policy
toggle_restart() {
    for lxc_id in $(get_running_docker_lxc); do
        compose_file=$(find_watchtower_compose "$lxc_id")
        if [ -n "$compose_file" ]; then
            current=$(pct exec "$lxc_id" -- grep "restart:" "$compose_file" | awk '{print $2}')
            if [ "$current" = "always" ]; then new="none"; else new="always"; fi
            pct exec "$lxc_id" -- sed -i "s/^restart:.*/restart: $new/" "$compose_file"
            dir=$(dirname "$compose_file")
            pct exec "$lxc_id" -- sh -c "cd $dir && docker compose down && docker compose up -d"
            echo "🔄 Restart policy basculée et Watchtower redémarré dans LXC $lxc_id : $new"
        else
            echo "Pas de docker-compose.yml trouvé ou recherche expirée pour LXC $lxc_id."
        fi
    done
    read -rp "Appuyez sur [Entrée] pour revenir au menu..."
}

# Schedule aléatoire (14h-20h, minutes multiples de 5) pour chaque LXC
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
            echo "✅ WATCHTOWER_SCHEDULE mis à jour et Watchtower redémarré pour LXC $lxc_id : $schedule"
        else
            echo "Pas de docker-compose.yml trouvé ou recherche expirée pour LXC $lxc_id."
        fi
    done
    read -rp "Appuyez sur [Entrée] pour revenir au menu..."
}

# Schedule fixe pour tous (Spring cron, 6 champs)
fixed_schedule() {
    read -rp "Entrez la valeur du schedule (ex: 0 0 16 ? * 5) : " schedule
    modify_key_restart "WATCHTOWER_SCHEDULE" "$schedule"
}

# Menu principal
while true; do
    clear
    echo "$MENU"
    read -rp "Votre choix : " choice
    case $choice in
        1) status_watchtower ;;
        2) start_watchtower ;;
        3) stop_watchtower ;;
        4) restart_watchtower ;;
        5) view_compose ;;
        6) toggle_restart ;;
        7) read -rp "Entrez true ou false pour WATCHTOWER_NO_STARTUP_MESSAGE : " val; modify_key_restart "WATCHTOWER_NO_STARTUP_MESSAGE" "$val" ;;
        8) read -rp "Entrez true ou false pour WATCHTOWER_CLEANUP : " val; modify_key_restart "WATCHTOWER_CLEANUP" "$val" ;;
        9) random_schedule ;;
        10) fixed_schedule ;;
        11) read -rp "Entrez la valeur pour WATCHTOWER_TIMEOUT (ex: 30s) : " val; modify_key_restart "WATCHTOWER_TIMEOUT" "$val" ;;
        [Qq]) exit ;;
        *) echo "Option invalide." ; read -rp "Appuyez sur [Entrée] pour continuer..." ;;
    esac
done
```

---

## Vidéo

Une vidéo de démonstration existe, et celle-ci à été publié sur les réseaux-sociaux Blabla Linux 😎

- [Sur Facebook](https://www.facebook.com/share/v/1BZNkP7kk5/)
- [Sur Twitter (X)](https://x.com/BlablaLinux/status/1986574526365507694)
- [Sur Bluesky](https://bsky.app/profile/blablalinux.be/post/3m4ypfgobds2a)
- [Sur Mastodon](https://mastodon.blablalinux.be/@blablalinux/115505322559841251)

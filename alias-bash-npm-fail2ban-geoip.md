---
title: Mon fichier d'alias Bash pour Nginx Proxy Manager, Fail2Ban et GeoIP
description: Découvrez mon fichier d'alias Bash surpuissant pour administrer et surveiller un serveur Nginx Proxy Manager couplé à Fail2Ban et GeoIP. Simplifiez la gestion de vos logs et sécurisez vos services.
published: true
date: 2026-05-30T17:59:35.092Z
tags: docker, lxc, fail2ban, bash, alias, sécurité, administration, nginxproxymanager, geoip
editor: markdown
dateCreated: 2026-05-30T17:59:35.092Z
---

Voici la documentation complète de mon fichier d'alias, conçu pour gagner un temps précieux lors de l'administration d'une infrastructure d'auto-hébergement.

## Introduction et architecture type

Ce fichier est pensé pour une architecture spécifique, mais très courante :

* **Nginx Proxy Manager (NPM)** et **Fail2Ban** tournent dans des conteneurs.


* Les journaux de Nginx sont consolidés dans un fichier global (ici nommé `global_access_geo.log`) qui est monté dans Fail2Ban.


* Un système de filtrage GeoIP ajoute des balises `[yes]` ou `[no]` directement dans les requêtes.



> **Note de sécurité :** Dans le script complet en bas de page, les chemins vers les dossiers de configuration (`/chemin/vers/...`), les adresses IP d'exemple et le nom de domaine de base (`example.com`) ont été modifiés par rapport à mon infrastructure réelle pour des raisons évidentes de sécurité. Pensez à les adapter à votre propre environnement !

## Contrôle de Fail2Ban et gestion des prisons

La première partie du script permet d'avoir un œil constant sur l'état des défenses. La commande `f2ball` est particulièrement pratique puisqu'elle affiche un audit rapide des quatre prisons principales : `npm` (trafic autorisé), `npm-geo-abuse` (trafic refusé géographiquement), `npm-403-abuse` (erreurs 403 globales) et `npm-wp-login` (attaques sur WordPress).

Des commandes directes comme `f2b-ban` et `f2b-unban` permettent de bloquer ou débloquer une adresse IP (IPv4 ou IPv6) sur l'ensemble de ces prisons en une seule pression de touche.

## Actions rapides sur Nginx Proxy Manager

Inutile d'entrer dans le conteneur pour tester la syntaxe (`npm-t`) ou recharger la configuration à chaud (`npm-reload`).
Le petit bijou de cette section est la commande `npm-domains`. Plutôt que de lire des fichiers de configuration, elle interroge directement la base de données SQLite de Nginx Proxy Manager (`database.sqlite`) pour extraire les noms de domaine actifs (où `is_deleted=0`), les nettoie et les trie par ordre alphabétique.

## Surveillance du trafic en temps réel (logs)

C'est ici que la magie opère pour l'analyse en direct. Grâce à une série d'alias basés sur la commande `tail -f` :

* `live-yes` et `live-no` permettent de filtrer le trafic selon l'autorisation géographique.


* `live-site <domaine>` cible instantanément les requêtes entrantes pour un service spécifique.


* La fonction de corrélation `live-ban` affiche simultanément les erreurs 403 de Nginx et les journaux du conteneur Fail2Ban, ce qui est redoutable pour voir une attaque et son blocage en temps réel.



## Corrélation, historique et statistiques

Pour comprendre ce qui se passe sur le serveur, le script intègre des outils d'analyse puissants :

* `f2b-top-ip`, `f2b-top-403`, et `f2b-top-no` génèrent des classements des adresses IP les plus actives ou les plus bloquées en lisant directement la réalité du trafic dans les journaux Nginx.


* `f2b-ip-history <IP>` fouille dans la base de données `fail2ban.sqlite3` pour vous donner l'historique exact des heures et dates de bannissement d'une adresse.



## L'outil d'analyse GeoIP multi-sources

C'est la version "Pro" du script : la fonction `geoip` lance un menu interactif permettant d'analyser n'importe quelle adresse IP. Vous pouvez choisir votre source de données entre la base MaxMind (`GeoLite2`) ou la base `ip66.dev` (via `mmdbinspect`). L'outil renvoie l'ASN (le fournisseur d'accès), la ville, le pays, ou un dump JSON complet de la localisation.

---

## Le script complet prêt à l'emploi

Voici le fichier complet à coller dans votre `~/.bash_aliases`. N'oubliez pas d'ajuster la variable `LOG` ainsi que les chemins de vos montages Docker/LXC.

```bash
# ============================================================
# FAIL2BAN + NPM + GEOIP (DOCKER / LXC)
# ============================================================
# Architecture réelle :
# - Fail2Ban : container (logs = docker stdout)
# - Nginx logs : monté dans fail2ban
# - GeoIP : basé sur tag [yes|no]
#
# Lecture :
# - live-* → trafic
# - f2b-* → décisions Fail2Ban
# - top-* → analyse
# ============================================================

LOG="/chemin/vers/npm/data/logs_geo/global_access_geo.log"


# ============================================================
# FAIL2BAN - STATUS
# ============================================================

# Vue globale
# Usage : f2bs
alias f2bs='docker exec -t fail2ban fail2ban-client status'

# Prisons
alias f2bnpm='docker exec -t fail2ban fail2ban-client status npm'
alias f2bgeo='docker exec -t fail2ban fail2ban-client status npm-geo-abuse'
alias f2b403='docker exec -t fail2ban fail2ban-client status npm-403-abuse'
alias f2bwp='docker exec -t fail2ban fail2ban-client status npm-wp-login'

# Vue complète (audit rapide)
# Usage : f2ball
f2ball() {
    echo "=== NPM (YES) ==="
    docker exec -t fail2ban fail2ban-client status npm
    echo ""
    echo "=== GEO (NO) ==="
    docker exec -t fail2ban fail2ban-client status npm-geo-abuse
    echo ""
    echo "=== 403 ABUSE ==="
    docker exec -t fail2ban fail2ban-client status npm-403-abuse
    echo ""
    echo "=== WP LOGIN ==="
    docker exec -t fail2ban fail2ban-client status npm-wp-login
}

# Reload config
alias f2breload='docker exec -t fail2ban fail2ban-client reload'


# ============================================================
# ACTIONS IP (SAFE)
# ============================================================

# Bannir une IP partout
# Usage : f2b-ban 1.2.3.4 ou 2001:db8::1
# Cas : IP malveillante confirmée
f2b-ban() {
    [ -z "$1" ] && echo "Usage: f2b-ban <IP>" && return 1
    
    # Validation hybride IPv4 et IPv6
    if echo "$1" | grep -Eq '^([0-9]{1,3}\.){3}[0-9]{1,3}$' || echo "$1" | grep -q ':'; then
        echo "[BAN] $1"
        for jail in npm npm-geo-abuse npm-403-abuse npm-wp-login; do
            docker exec -t fail2ban fail2ban-client set "$jail" banip "$1"
        done
    else
        echo "IP invalide (ni IPv4, ni IPv6 valide)"
        return 1
    fi
}

# Débannir
# Usage : f2b-unban 1.2.3.4 ou 2001:db8::1
# Cas : faux positif
f2b-unban() {
    [ -z "$1" ] && echo "Usage: f2b-unban <IP>" && return 1

    echo "[UNBAN] $1"
    for jail in npm npm-geo-abuse npm-403-abuse npm-wp-login; do
        docker exec -t fail2ban fail2ban-client set "$jail" unbanip "$1"
    done
}

# ============================================================
# NGINX PROXY MANAGER - ACTIONS
# ============================================================

# Tester la syntaxe de configuration Nginx
# Usage : npm-t
alias npm-t='docker exec -it npm nginx -t'

# Recharger la configuration Nginx à chaud (sans coupure)
# Usage : npm-reload
alias npm-reload='docker exec -it npm nginx -s reload'

# Lister tous les domaines gérés par Nginx Proxy Manager (via DB)
# Usage : npm-domains
npm-domains() {
    echo "=== Domaines actifs dans Nginx Proxy Manager ==="
    sqlite3 /chemin/vers/npm/data/database.sqlite "SELECT domain_names FROM proxy_host WHERE is_deleted=0;" 2>/dev/null | tr -d '[]"' | tr ',' '\n' | sort -u
    echo "================================================="
}


# ============================================================
# LIVE TRAFIC (NGINX)
# ============================================================

# Full log
# Usage : live
alias live='tail -f "$LOG"'

# YES
# Usage : live-yes
alias live-yes='tail -f "$LOG" | grep --line-buffered "\[yes "'

# NO
# Usage : live-no
alias live-no='tail -f "$LOG" | grep --line-buffered "\[no "'

# 403 global
# Usage : live-403
alias live-403='tail -f "$LOG" | grep --line-buffered " 403$"'

# YES + 403
# Usage : live-yes-403
# Cas : bot autorisé géo
alias live-yes-403='tail -f "$LOG" | grep --line-buffered "\[yes " | grep --line-buffered " 403$"'

# NO + 403
# Usage : live-no-403
# Cas : scan bloqué GEO
alias live-no-403='tail -f "$LOG" | grep --line-buffered "\[no " | grep --line-buffered " 403$"'

# WP-Login
# Usage : live-wp
# Cas : requêtes sur la page de connexion WordPress
alias live-wp='tail -f "$LOG" | grep --line-buffered "wp-login\.php"'

# Surveillance d'un domaine spécifique en direct
# Usage : live-site mondomaine
# Exemple : live-site nextcloud
live-site() {
    [ -z "$1" ] && echo "Usage: live-site <domaine>" && return 1
    tail -f "$LOG" | grep --line-buffered "$1.example.com"
}

# ============================================================
# CORRÉLATION (TRÈS PUISSANT)
# ============================================================

# Voir attaque + ban en live
# Usage : live-ban
# Cas : debug comportement Fail2Ban
live-ban() {
    docker logs -f fail2ban &
    tail -f "$LOG" | grep --line-buffered " 403$"
}

# Stop propre
alias live-stop='killall tail docker 2>/dev/null'


# ============================================================
# FAIL2BAN LOGS (SOURCE RÉELLE)
# ============================================================

# Logs Fail2Ban
# Usage : f2b-log
# Cas : voir bans en direct
alias f2b-log='docker logs -f fail2ban'


# ============================================================
# SOURCE OF TRUTH (BANS RÉELS)
# ============================================================

# IP bannies par prison
# Usage : f2b-banned npm
f2b-banned() {
    docker exec fail2ban fail2ban-client get "$1" banip
}

# Vue globale
# Usage : f2b-banned-list
# Cas : audit réel
f2b-banned-list() {
    for jail in npm npm-geo-abuse npm-403-abuse npm-wp-login; do
        echo "=== $jail ==="
        docker exec fail2ban fail2ban-client get "$jail" banip
        echo ""
    done
}

# Voir l'historique des dates et heures des 20 derniers bannissements réels
# Usage : f2b-dates
alias f2b-dates='sqlite3 /chemin/vers/f2b/data/db/fail2ban.sqlite3 "SELECT jail, ip, datetime(timeofban, '\''unixepoch'\'', '\''localtime'\'') FROM bans ORDER BY timeofban DESC LIMIT 20;"'

# Rechercher l'historique de bannissement d'une IP précise dans la base
# Usage : f2b-ip-history 1.2.3.4 ou 2001:db8::1
f2b-ip-history() {
    [ -z "$1" ] && echo "Usage: f2b-ip-history <IP>" && return 1
    sqlite3 /chemin/vers/f2b/data/db/fail2ban.sqlite3 "SELECT jail, ip, datetime(timeofban, 'unixepoch', 'localtime') FROM bans WHERE ip='$1' ORDER BY timeofban DESC;"
}


# ============================================================
# STATISTIQUES (LOG = RÉALITÉ TRAFIC)
# ============================================================

# Top IP global (IPv4 et IPv6)
# Usage : f2b-top-ip
f2b-top-ip() {
    grep -oP "Client \K[0-9a-fA-F.:]+" "$LOG" | sort | uniq -c | sort -rn | head
}

# Top 403 (IPv4 et IPv6)
# Usage : f2b-top-403
f2b-top-403() {
    grep ' 403$' "$LOG" | grep -oP "Client \K[0-9a-fA-F.:]+" | sort | uniq -c | sort -rn | head
}

# Top GEO refusé (IPv4 et IPv6)
# Usage : f2b-top-no
f2b-top-no() {
    grep "\[no " "$LOG" | grep -oP "Client \K[0-9a-fA-F.:]+" | sort | uniq -c | sort -rn | head
}

# Top trafic autorisé (IPv4 et IPv6)
# Usage : f2b-top-yes
f2b-top-yes() {
    grep "\[yes " "$LOG" | grep -oP "Client \K[0-9a-fA-F.:]+" | sort | uniq -c | sort -rn | head
}


# ============================================================
# DEBUG
# ============================================================

# Vérifier log
# Usage : f2b-check-log
f2b-check-log() {
    ls -lah "$LOG"
    tail -n 10 "$LOG"
}


# ============================================================
# FIREWALL
# ============================================================

# Blocage direct
# Usage : block-ip 1.2.3.4 ou 2001:db8::1
block-ip() {
    [ -z "$1" ] && echo "Usage: block-ip <IP>" && return 1
    
    if echo "$1" | grep -q ':'; then
        echo "[FIREWALL IPv6] Blocage direct de $1"
        ip6tables -I DOCKER-USER -s "$1" -j DROP
    else
        echo "[FIREWALL IPv4] Blocage direct de $1"
        iptables -I DOCKER-USER -s "$1" -j DROP
    fi
}

# Vue globale iptables (f2b + DROP + REJECT)
# Usage : f2b-list
alias f2b-list='echo "=== IPv4 ===" && iptables -L -n | grep -E "f2b|DROP|REJECT"; echo "=== IPv6 ===" && ip6tables -L -n | grep -E "f2b|DROP|REJECT"'

# Liste brute de ta chaîne personnalisée avec compteurs d'IP (Packets / Bytes)
# Usage : f2b-ipt-f2b
alias f2b-ipt-f2b='echo "=== IPv4 ===" && iptables -L f2b-npm-docker -n -v; echo "=== IPv6 ===" && ip6tables -L f2b-npm-docker -n -v 2>/dev/null || echo "Pas de chaîne f2b-npm-docker en IPv6"'

# Recherche brute de l'historique d'une IP dans les logs de bannissement
# Usage : ipt-search 1.2.3.4
ipt-search() {
    [ -z "$1" ] && echo "Usage: ipt-search <IP>" && return 1
    docker logs fail2ban 2>&1 | grep "Ban $1"
}

# Compter le nombre total d'adresses IPv4 et IPv6 actuellement bloquées dans le pare-feu
alias f2b-count='echo -n "Total IPv4 bloquées : " && iptables -L -n | grep -E "DROP|REJECT" | wc -l && echo -n "Total IPv6 bloquées : " && ip6tables -L -n | grep -E "DROP|REJECT" | wc -l'


# ============================================================
# MAINTENANCE
# ============================================================

alias update='apt update && apt upgrade -y'
alias clean='apt autoremove -y && apt autoclean'
alias alias-edit='nano ~/.bash_aliases && source ~/.bash_aliases'
alias alias-reload='source ~/.bash_aliases'


# ============================================================
# GEOIP ULTIME (MAXMIND + IP66 + FULL DUMP) - VERSION PRO
# ============================================================

geoip() {
    local PATH_MM="/chemin/vers/npm/data/geoip2"
    local PATH_I66="/chemin/vers/npm/data/ip66/ip66.mmdb"
    
    local C_CYAN='\033[0;36m'
    local C_RED='\033[0;31m'
    local C_BOLD='\033[1m'
    local C_RESET='\033[0m'
    
    # Message d'accueil et instruction de sortie
    echo -e "${C_BOLD}${C_CYAN}Lancement de l'outil GeoIP Multi-Sources...${C_RESET}"
    echo -e "Note : Vous pouvez quitter cet outil à tout moment avec ${C_BOLD}CTRL+C${C_RESET}."

    while true; do
        echo ""
        echo "--- SOURCE DE DONNÉES ---"
        echo "1) MaxMind (GeoLite2)"
        echo "2) ip66.dev (MMDBinspect)"
        echo "q) Quitter"
        echo "Votre choix de source : "
        read source
        [[ "$source" == "q" ]] && break

        if [[ ! "$source" =~ ^[1-2]$ ]]; then
            printf "${C_RED}Erreur : Veuillez choisir 1, 2 ou q.${C_RESET}\n"
            continue
        fi

        while true; do
            echo ""
            echo "--- MENU INFO ---"
            echo "1) ASN (Fournisseur)"
            echo "2) Ville"
            echo "3) Pays"
            echo "4) DUMP COMPLET (Bloc texte)"
            echo "b) Retour au choix de la source"
            echo "Votre choix d'info : "
            read choice
            
            [[ "$choice" == "b" ]] && break
            
            # Anti-IP dans le menu (Supporte les IPV4 et IPV6)
            if [[ "$choice" =~ ^[0-9a-fA-F.:]+$ ]] && [[ "$choice" =~ [.:] ]]; then
                printf "${C_RED}Attention : Tu as entré une IP au lieu d'une option du menu (1-4).${C_RESET}\n"
                continue
            fi
            
            if [[ ! "$choice" =~ ^[1-4]$ ]]; then
                printf "${C_RED}Choix non valide. Entre un chiffre de 1 à 4 ou 'b'.${C_RESET}\n"
                continue
            fi

            # --- VÉRIFICATION DE L'IP ---
            while true; do
                echo "Entrez l'IP : "
                read ip
                [[ -z "$ip" ]] && { echo "IP manquante."; continue; }
                
                # Détection si l'utilisateur entre un chiffre de menu au lieu d'une IP
                if [[ "$ip" =~ ^[0-9]{1,2}$ ]]; then
                    printf "${C_RED}Attention : Tu as entré un numéro de menu au lieu d'une adresse IP.${C_RESET}\n"
                    continue
                fi
                break
            done

            if [ "$source" == "1" ]; then
                case $choice in
                    1)
                        res=$(mmdblookup --file "$PATH_MM/GeoLite2-ASN.mmdb" --ip "$ip" autonomous_system_organization 2>/dev/null | grep -oP '".*?"' | tr -d '"')
                        printf ">> ASN : ${C_BOLD}${C_CYAN}%s${C_RESET}\n" "${res:-Inconnu}" ;;
                    2)
                        res=$(mmdblookup --file "$PATH_MM/GeoLite2-City.mmdb" --ip "$ip" city names en 2>/dev/null | grep -oP '".*?"' | tr -d '"')
                        printf ">> Ville : ${C_BOLD}${C_CYAN}%s${C_RESET}\n" "${res:-Inconnue}" ;;
                    3)
                        res=$(mmdblookup --file "$PATH_MM/GeoLite2-Country.mmdb" --ip "$ip" country names fr 2>/dev/null | grep -oP '".*?"' | tr -d '"')
                        printf ">> Pays : ${C_BOLD}${C_CYAN}%s${C_RESET}\n" "${res:-Inconnu}" ;;
                    4)
                        echo -e "${C_CYAN}--- DUMP COMPLET MAXMIND (CITY) ---${C_RESET}"
                        mmdblookup --file "$PATH_MM/GeoLite2-City.mmdb" --ip "$ip" 2>/dev/null ;;
                esac

            elif [ "$source" == "2" ]; then
                case $choice in
                    1)
                        res=$(mmdbinspect -db "$PATH_I66" "$ip" 2>/dev/null | grep "autonomous_system_organization" | head -n 1 | grep -oP '(?<=": ").*(?=")')
                        printf ">> ASN : ${C_BOLD}${C_CYAN}%s${C_RESET}\n" "${res:-Inconnu}" ;;
                    2)
                        res=$(mmdbinspect -db "$PATH_I66" "$ip" 2>/dev/null | sed -n '/"city":/,/}/p' | grep '"en":' | grep -oP '(?<=": ").*(?=")')
                        printf ">> Ville : ${C_BOLD}${C_CYAN}%s${C_RESET}\n" "${res:-Inconnue (non listée)}" ;;
                    3)
                        res=$(mmdbinspect -db "$PATH_I66" "$ip" 2>/dev/null | sed -n '/"country":/,/}/p' | grep '"en":' | head -n 1 | grep -oP '(?<=": ").*(?=")')
                        printf ">> Pays : ${C_BOLD}${C_CYAN}%s${C_RESET}\n" "${res:-Inconnu}" ;;
                    4)
                        echo -e "${C_CYAN}--- DUMP COMPLET IP66 (JSON) ---${C_RESET}"
                        mmdbinspect -db "$PATH_I66" "$ip" 2>/dev/null ;;
                esac
            fi
        done
    done
}


# ============================================================
# SAUVEGARDE ET RESTAURATION (PROXMOX / LXC)
# ============================================================

# Sauvegarder toute la configuration Nginx Custom
# Usage : npm-backup
npm-backup() {
    echo "[BACKUP] Sauvegarde globale du dossier Nginx custom..."
    rm -rf /chemin/vers/npm/backups_custom/data/nginx/custom/*
    cp -r /chemin/vers/npm/data/nginx/custom/* /chemin/vers/npm/backups_custom/data/nginx/custom/
    echo "[OK] Terminé."
}

# Restaurer toute la configuration Nginx Custom depuis le backup
# Usage : npm-restore
npm-restore() {
    echo "[RESTORE] Restauration globale du dossier Nginx custom..."
    rm -rf /chemin/vers/npm/data/nginx/custom/*
    cp -r /chemin/vers/npm/backups_custom/data/nginx/custom/* /chemin/vers/npm/data/nginx/custom/
    echo "[TEST] Vérification de la syntaxe Nginx..."
    npm-t
    if [ $? -eq 0 ]; then
        echo "[RELOAD] Application de la configuration..."
        npm-reload
        echo "[OK] Restauration complète et réussie."
    else
        echo " Erreur de syntaxe détectée dans les fichiers restaurés ! Nginx n'a pas été rechargé."
    fi
}

# Sauvegarder toute la configuration Fail2Ban
# Usage : f2b-backup
f2b-backup() {
    echo "[BACKUP] Sauvegarde globale du dossier de config Fail2Ban..."
    rm -rf /chemin/vers/f2b/backups/data/*
    cp -r /chemin/vers/f2b/data/* /chemin/vers/f2b/backups/data/
    echo "[OK] Terminé."
}

# Restaurer toute la configuration Fail2Ban depuis le backup
# Usage : f2b-restore
f2b-restore() {
    echo "[RESTORE] Restauration globale du dossier de config Fail2Ban..."
    rm -rf /chemin/vers/f2b/data/*
    cp -r /chemin/vers/f2b/backups/data/* /chemin/vers/f2b/data/
    echo "[RELOAD] Rechargement de la configuration Fail2Ban..."
    f2breload
    echo "[OK] Restauration complète et réussie."
}

```

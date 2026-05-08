---
title: L'alias GeoIP ultime (MaxMind + ip66.dev)
description: Apprenez à installer un alias GeoIP puissant combinant MaxMind et ip66.dev. Un guide pas à pas pour diagnostiquer vos IP, détecter les VPN et serveurs cloud directement dans votre terminal.
published: false
date: 2026-05-08T18:14:56.163Z
tags: bash, sécurité, linux, terminal, geoip, maxmind, ip66.dev
editor: markdown
dateCreated: 2026-05-08T18:09:39.102Z
---

Ce guide vous explique comment transformer votre terminal en un véritable centre de diagnostic réseau. Nous allons coupler la base **MaxMind** (géographie) et la base **ip66.dev** (détection de VPN, Proxy, et serveurs Cloud).

## 🛠 1. Prérequis et liens utiles

Pour que ce guide fonctionne, vous devez :

1. Avoir un conteneur **GeoIP Update** fonctionnel (voir le guide [Sécurisation NPM avec Fail2Ban et GeoIP2](https://wiki.blablalinux.be/fr/securisation-npm-fail2ban-geoip2)) ou au moins le conteneur officiel de mise à jour MaxMind qui tourne sur votre infrastructure.
2. Disposer d'un compte chez [MaxMind](https://www.maxmind.com) (gratuit pour les bases GeoLite2).
3. Découvrir [ip66.dev](https://ip66.dev), un excellent service complémentaire pour identifier les types de connexions.

---

## 📂 2. Organisation des dossiers

Nous allons utiliser des chemins d'exemple généralistes. **Important :** Partout où vous verrez `/votre/chemin/`, remplacez-le par le chemin réel sur votre serveur (par exemple : `/root/npm/data` ou `/home/user/data`).

Créez la structure pour accueillir la base complémentaire ip66 :

```bash
# Remplacez /votre/chemin/ par votre dossier de données réel
mkdir -p /votre/chemin/geoip2
mkdir -p /votre/chemin/ip66

```

---

## ⚙️ 3. Installation de mmdbinspect

L'outil standard `mmdblookup` est parfait pour MaxMind, mais pour lire les données plus complexes (JSON) de la base **ip66**, nous avons besoin de `mmdbinspect`. Voici comment l'installer étape par étape :

1. **cd /tmp** : on se place dans le dossier temporaire du système pour ne pas encombrer vos dossiers personnels.
2. **wget [URL]** : on télécharge l'archive officielle depuis le dépôt GitHub de MaxMind.
3. **tar xvf [Fichier]** : on décompresse l'archive. Cela va créer un dossier contenant le programme binaire (le fichier exécutable).
4. **mv ... /usr/local/bin/** : on déplace le fichier exécutable dans un dossier système reconnu. Cela permet de lancer la commande `mmdbinspect` depuis n'importe où dans le terminal.
5. **chmod +x ...** : on rend le fichier exécutable par le système.
6. **rm -rf ...** : on nettoie les fichiers d'installation devenus inutiles.

```bash
cd /tmp
wget https://github.com/maxmind/mmdbinspect/releases/download/v0.2.0/mmdbinspect_0.2.0_linux_amd64.tar.gz
tar xvf mmdbinspect_0.2.0_linux_amd64.tar.gz
mv mmdbinspect_0.2.0_linux_amd64/mmdbinspect /usr/local/bin/
chmod +x /usr/local/bin/mmdbinspect
rm -rf mmdbinspect_0.2.0_linux_amd64*

```

---

## 🔄 4. Script de mise à jour ip66.dev

Contrairement à MaxMind qui possède son propre conteneur Docker de mise à jour, nous allons créer un petit script léger pour récupérer la base ip66 une fois par semaine.

Créez le fichier : `nano /votre/chemin/ip66/update_ip66.sh`

```bash
#!/bin/bash
# Script de mise à jour de la base ip66.dev
DEST_DIR="/votre/chemin/ip66"
URL="https://downloads.ip66.dev/db/ip66.mmdb"

cd "$DEST_DIR" || exit

# Téléchargement vers un fichier temporaire pour ne pas corrompre l'original
wget -q --no-check-certificate --user-agent="Mozilla/5.0" "$URL" -O ip66.mmdb.tmp

# Si le fichier reçu n'est pas vide, on remplace l'ancienne base
if [ -s "ip66.mmdb.tmp" ]; then
    mv ip66.mmdb.tmp ip66.mmdb
    chmod 644 ip66.mmdb
    echo "Mise à jour ip66.dev terminée avec succès."
else
    rm -f ip66.mmdb.tmp
    echo "Erreur lors du téléchargement (vérifiez votre connexion)."
fi

```

**Rendez-le exécutable** : `chmod +x /votre/chemin/ip66/update_ip66.sh`

---

## 📜 5. L'alias GeoIP ultime

Voici le cœur du système. Cet alias est à copier dans votre fichier `~/.bash_aliases` (ou `~/.bashrc`). Il inclut des sécurités pour éviter les erreurs de saisie (comme entrer une IP au lieu d'un numéro de menu).

**⚠️ Pensez à bien modifier les deux premières variables (PATH_MM et PATH_I66) avec vos chemins réels !**

```bash
geoip() {
    # --- CONFIGURATION DES CHEMINS ---
    local PATH_MM="/votre/chemin/geoip2"
    local PATH_I66="/votre/chemin/ip66/ip66.mmdb"
    
    # Couleurs pour la lisibilité
    local C_CYAN='\033[0;36m'
    local C_RED='\033[0;31m'
    local C_BOLD='\033[1m'
    local C_RESET='\033[0m'
    
    echo -e "${C_BOLD}${C_CYAN}Lancement de l'outil GeoIP Multi-Sources...${C_RESET}"
    echo -e "Note : Vous pouvez quitter à tout moment avec ${C_BOLD}CTRL+C${C_RESET}."

    while true; do
        echo -e "\n--- SOURCE DE DONNÉES ---"
        echo "1) MaxMind (GeoLite2)"
        echo "2) ip66.dev (MMDBinspect)"
        echo "q) Quitter"
        read -p "Votre choix : " source
        [[ "$source" == "q" ]] && break
        [[ ! "$source" =~ ^[1-2]$ ]] && { printf "${C_RED}Erreur : Choisissez 1 ou 2.${C_RESET}\n"; continue; }

        while true; do
            echo -e "\n--- MENU INFO ---"
            echo "1) ASN | 2) Ville | 3) Pays | 4) DUMP COMPLET | b) Retour"
            read -p "Votre choix : " choice
            [[ "$choice" == "b" ]] && break
            
            # Sécurité : si l'utilisateur entre une IP au lieu du chiffre du menu
            [[ "$choice" =~ ^[0-9]{1,3}(\.[0-9]{1,3}){3}$ ]] && { printf "${C_RED}Attention : Entrez un chiffre (1-4), pas l'IP ici !${C_RESET}\n"; continue; }
            [[ ! "$choice" =~ ^[1-4]$ ]] && { printf "${C_RED}Choix invalide.${C_RESET}\n"; continue; }

            # Saisie de l'IP avec sécurité anti-chiffre seul
            while true; do
                read -p "Entrez l'IP : " ip
                [[ -z "$ip" ]] && { echo "IP manquante."; continue; }
                [[ "$ip" =~ ^[0-9]{1,2}$ ]] && { printf "${C_RED}Erreur : Entrez une IP valide, pas un numéro de menu !${C_RESET}\n"; continue; }
                break
            done

            if [ "$source" == "1" ]; then
                case $choice in
                    1) res=$(mmdblookup --file "$PATH_MM/GeoLite2-ASN.mmdb" --ip "$ip" autonomous_system_organization 2>/dev/null | grep -oP '".*?"' | tr -d '"')
                       printf ">> ASN : ${C_BOLD}${C_CYAN}%s${C_RESET}\n" "${res:-Inconnu}" ;;
                    2) res=$(mmdblookup --file "$PATH_MM/GeoLite2-City.mmdb" --ip "$ip" city names en 2>/dev/null | grep -oP '".*?"' | tr -d '"')
                       printf ">> Ville : ${C_BOLD}${C_CYAN}%s${C_RESET}\n" "${res:-Inconnue}" ;;
                    3) res=$(mmdblookup --file "$PATH_MM/GeoLite2-Country.mmdb" --ip "$ip" country names fr 2>/dev/null | grep -oP '".*?"' | tr -d '"')
                       printf ">> Pays : ${C_BOLD}${C_CYAN}%s${C_RESET}\n" "${res:-Inconnu}" ;;
                    4) mmdblookup --file "$PATH_MM/GeoLite2-City.mmdb" --ip "$ip" 2>/dev/null ;;
                esac
            else
                case $choice in
                    1) res=$(mmdbinspect -db "$PATH_I66" "$ip" 2>/dev/null | grep "autonomous_system_organization" | head -n 1 | grep -oP '(?<=": ").*(?=")')
                       printf ">> ASN : ${C_BOLD}${C_CYAN}%s${C_RESET}\n" "${res:-Inconnu}" ;;
                    2) res=$(mmdbinspect -db "$PATH_I66" "$ip" 2>/dev/null | sed -n '/"city":/,/}/p' | grep '"en":' | grep -oP '(?<=": ").*(?=")')
                       printf ">> Ville : ${C_BOLD}${C_CYAN}%s${C_RESET}\n" "${res:-Inconnue}" ;;
                    3) res=$(mmdbinspect -db "$PATH_I66" "$ip" 2>/dev/null | sed -n '/"country":/,/}/p' | grep '"en":' | head -n 1 | grep -oP '(?<=": ").*(?=")')
                       printf ">> Pays : ${C_BOLD}${C_CYAN}%s${C_RESET}\n" "${res:-Inconnu}" ;;
                    4) mmdbinspect -db "$PATH_I66" "$ip" 2>/dev/null ;;
                esac
            fi
        done
    done
}

```

---

## 🛡️ 6. Petit conseil « BlablaLinux » (AdGuard Home)

Si vous utilisez **AdGuard Home** ou un autre bloqueur DNS, il est possible que le domaine `download.maxmind.com` soit bloqué par défaut par certaines listes de filtrage. Si vos bases ne se mettent pas à jour, pensez à ajouter ce domaine à votre **liste d'autorisation** (Whitelist).

---

## 📽️ 7. Démonstration de l'outil en action

Rien de tel qu'une petite démonstration visuelle pour voir comment l'alias `geoip` se comporte « dans la vraie vie ». On y voit la bascule entre les sources, la sécurité anti-erreur et l'affichage des données.

![installation-alias-geoip-multisources.gif](/installation-alias-geoip-multisources/installation-alias-geoip-multisources.gif){width=100%}

---

> **Auteur** : ce guide est proposé par **Amaury aka BlablaLinux**. Retrouvez l'ensemble de mes services sur [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).

---
title: Automatiser la mise à jour complète de Debian 13 et Ubuntu Server 24.04
description: >-
  Automatiser la mise à jour complète et le nettoyage de Debian 13 (GNOME) et
  Ubuntu Server 24.04 grâce à deux scripts Bash et un alias dédié.
published: true
date: '2026-11-14T17:28:17.404Z'
tags:
  - linux
  - debian
  - debian 13
  - ubuntu
  - ubuntu 24.04
  - bash
  - administration
  - maintenance
editor: markdown
dateCreated: '2026-11-14T17:12:46.371Z'
---

Maintenir un système Linux à jour demande souvent d'enchaîner plusieurs commandes : mise à jour des dépôts, mise à niveau des paquets, nettoyage des dépendances inutiles, mise à jour des paquets Flatpak ou Snap, puis vérification d'un éventuel redémarrage.

Ce guide propose deux scripts Bash prêts à l'emploi ainsi qu'un alias `maj` pour lancer l'ensemble de ces opérations en une seule commande :

* **Une version Desktop** pensée pour **Debian 13 (GNOME)** qui gère APT, Flatpak et Snap.
* **Une version Serveur** adaptée à **Ubuntu Server 24.04 LTS** qui force le mode non-interactif (pour éviter les blocages liés à `needrestart`) et nettoie les anciennes révisions Snap.

---

## 1. Script pour Debian 13 (Desktop GNOME)

Ce script met à jour les paquets APT, purge les dépendances orphelines, actualise les applications Flatpak et Snap si elles sont présentes, puis vérifie si le noyau ou un service critique demande un redémarrage.

### Création du script

Crée le dossier `~/Scripts` (s'il n'existe pas encore) et enregistre le fichier `update-system.sh` :

```bash
mkdir -p ~/Scripts

cat << 'EOF' > ~/Scripts/update-system.sh
#!/usr/bin/env bash
#
# update-system.sh - Script de mise à jour et nettoyage complet du système Debian
#

set -euo pipefail

# Couleurs
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'

log_info()    { echo -e "${BLUE}[INFO]${NC} $1"; }
log_success() { echo -e "${GREEN}[OK]${NC} $1"; }
log_warn()    { echo -e "${YELLOW}[ATTENTION]${NC} $1"; }
log_error()   { echo -e "${RED}[ERREUR]${NC} $1"; }

# 1. Vérification des droits root
if [[ $EUID -ne 0 ]]; then
   log_error "Ce script doit être exécuté avec sudo ou en tant que root."
   exit 1
fi

log_info "--- Démarrage de la mise à jour globale du système ---"

# 2. Paquets Debian (APT)
log_info "Mise à jour des catalogues de paquets (apt update)..."
apt update

log_info "Mise à niveau des paquets DEB (apt full-upgrade)..."
apt full-upgrade -y

log_info "Nettoyage des paquets DEB inutilisés..."
apt autoremove --purge -y
apt autoclean

# 3. Paquets Flatpak (si installés)
if command -v flatpak &> /dev/null; then
    log_info "Mise à jour des paquets Flatpak..."
    flatpak update -y
    log_info "Nettoyage des paquets Flatpak inutilisés..."
    flatpak uninstall --unused -y
fi

# 4. Paquets Snap (si installés)
if command -v snap &> /dev/null; then
    log_info "Mise à jour des paquets Snap..."
    snap refresh
fi

# 5. Détection de redémarrage requis
if [ -f /var/run/reboot-required ]; then
    log_warn "Un redémarrage du système est nécessaire !"
    if [ -f /var/run/reboot-required.pkgs ]; then
        log_info "Paquets ayant demandé le redémarrage :"
        cat /var/run/reboot-required.pkgs
    fi
else
    log_success "Système à jour. Aucun redémarrage requis."
fi

log_success "Toutes les opérations de mise à jour sont terminées."
EOF

chmod +x ~/Scripts/update-system.sh
```

### Configuration de l'alias `maj`

Ajoute l'alias dans ton fichier `~/.bashrc` et recharge la configuration :

```bash
echo "alias maj='sudo ~/Scripts/update-system.sh'" >> ~/.bashrc
source ~/.bashrc
```

---

## 2. Script pour Ubuntu Server 24.04 LTS

Sur Ubuntu Server, l'outil `needrestart` affiche par défaut un écran interactif lors de la mise à jour de certaines bibliothèques ou du noyau, ce qui interrompt l'automatisation. De plus, Snap conserve plusieurs anciennes révisions désactivées qui finissent par occuper inutilement de l'espace disque.

Cette version corrige ces deux points :

* `DEBIAN_FRONTEND=noninteractive` et `NEEDRESTART_MODE=a` permettent de redémarrer les services automatiquement sans poser de question.
* Les options `Dpkg::Options` conservent tes fichiers de configuration déjà modifiés.
* Une boucle supprime les anciennes révisions Snap désactivées après chaque mise à jour.

### Création du script

```bash
mkdir -p ~/Scripts

cat << 'EOF' > ~/Scripts/update-system.sh
#!/usr/bin/env bash
#
# update-system.sh - Script de mise à jour et nettoyage complet pour Ubuntu Server 24.04
#

set -euo pipefail

# Évite les fenêtres interactives (notamment needrestart sur Ubuntu Server)
export DEBIAN_FRONTEND=noninteractive
export NEEDRESTART_MODE=a

# Couleurs
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'

log_info()    { echo -e "${BLUE}[INFO]${NC} $1"; }
log_success() { echo -e "${GREEN}[OK]${NC} $1"; }
log_warn()    { echo -e "${YELLOW}[ATTENTION]${NC} $1"; }
log_error()   { echo -e "${RED}[ERREUR]${NC} $1"; }

# 1. Vérification des droits root
if [[ $EUID -ne 0 ]]; then
    log_error "Ce script doit être exécuté avec sudo ou en tant que root."
    exit 1
fi

log_info "--- Démarrage de la mise à jour globale d'Ubuntu Server ---"

# 2. Paquets Ubuntu (APT)
log_info "Mise à jour des catalogues de paquets (apt update)..."
apt update

log_info "Mise à niveau complète des paquets (apt full-upgrade)..."
apt full-upgrade -y \
    -o Dpkg::Options::="--force-confdef" \
    -o Dpkg::Options::="--force-confold"

log_info "Nettoyage des paquets inutilisés..."
apt autoremove --purge -y
apt autoclean

# 3. Paquets Snap (présents par défaut sur Ubuntu Server)
if command -v snap &> /dev/null; then
    log_info "Mise à jour des paquets Snap..."
    snap refresh

    # Suppression des anciennes révisions Snap désactivées
    log_info "Nettoyage des anciennes révisions Snap..."
    snap list --all | awk '/disabled/{print $1, $3}' | while read -r snapname revision; do
        snap remove "$snapname" --revision="$revision"
    done
fi

# 4. Détection de redémarrage requis
if [ -f /var/run/reboot-required ]; then
    log_warn "Un redémarrage du serveur est nécessaire !"
    if [ -f /var/run/reboot-required.pkgs ]; then
        log_info "Paquets ayant demandé le redémarrage :"
        cat /var/run/reboot-required.pkgs
    fi
else
    log_success "Système à jour. Aucun redémarrage requis."
fi

log_success "Toutes les opérations de mise à jour sont terminées."
EOF

chmod +x ~/Scripts/update-system.sh
```

### Configuration de l'alias `maj`

Ajoute ou remplace l'alias `maj` dans ton fichier `~/.bashrc`, puis recharge-le :

```bash
sed -i "s|^alias maj=.*|alias maj='sudo ~/Scripts/update-system.sh'|" ~/.bashrc ~/.bash_aliases 2>/dev/null || true
grep -q "^alias maj=" ~/.bashrc ~/.bash_aliases 2>/dev/null || echo "alias maj='sudo ~/Scripts/update-system.sh'" >> ~/.bashrc
source ~/.bashrc
```

---

## Utilisation

Une fois l'alias en place, il suffit de taper la commande suivante dans ton terminal pour lancer la mise à jour complète et le nettoyage :

```bash
maj
```

> **Auteur** : ce guide est proposé par **Amaury aka BlablaLinux**. Retrouvez tous mes services sur [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).

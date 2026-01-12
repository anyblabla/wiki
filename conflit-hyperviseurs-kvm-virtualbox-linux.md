---
title: Gestion des conflits entre VirtualBox et KVM (Virt-Manager)
description: Comprendre et résoudre l'erreur "Un autre hyperviseur est déjà installé" lors du lancement de VirtualBox sur Debian/Ubuntu/Mint. Guide de bascule dynamique entre KVM, VMware et VirtualBox.
published: true
date: 2026-01-12T01:04:10.601Z
tags: virtualisation, virtmanager, virtualbox, kvm, vmware
editor: markdown
dateCreated: 2026-01-12T01:04:10.601Z
---

Il arrive fréquemment, notamment sur **Debian 13**, **Ubuntu 24.04** ou **Linux Mint 22**, que VirtualBox refuse de démarrer une machine virtuelle en affichant une erreur indiquant qu'un autre hyperviseur est déjà actif.

## Pourquoi ce problème apparaît ?

Le processeur de votre ordinateur possède des instructions de virtualisation matérielle (**VT-x** pour Intel, **AMD-V** pour AMD). Ces instructions ne peuvent être pilotées que par **un seul logiciel à la fois**.

Sur Linux, si vous avez installé **Virt-Manager**, le noyau charge automatiquement les modules **KVM** au démarrage. KVM prend alors l'exclusivité du processeur, empêchant VirtualBox d'accéder aux ressources nécessaires pour lancer ses propres VM.

---

## La solution : Bascule dynamique (hv-switch)

Plutôt que de désinstaller l'un ou l'autre, la solution consiste à "décharger" les modules de l'hyperviseur dont on n'a pas besoin pour laisser la place à l'autre.

### 1. Le script de bascule

Voici le script `hv-switch.sh` que nous utilisons pour automatiser cette opération. Il permet de passer proprement d'un mode à l'autre sans redémarrer.

```bash
#!/bin/bash
# hv-switch.sh : Bascule entre VirtualBox et KVM/VMware

set -euo pipefail

# Couleurs pour la lisibilité
VERT='\033[0;32m'
ROUGE='\033[0;31m'
CYAN='\033[0;36m'
FIN='\033[0m'

if [ "$(id -u)" -ne 0 ]; then
    echo -e "${ROUGE}ERREUR : Ce script doit être exécuté avec 'sudo'.${FIN}"
    exit 1
fi

echo -e "${CYAN}Gestionnaire de Conflit Hyperviseur${FIN}"
echo "1) Mode VirtualBox (Libère le CPU)"
echo "2) Mode KVM / VMware (Réactive tout)"
read -rp "Votre choix : " CHOIX

case "$CHOIX" in
    1)
        systemctl stop vmware 2>/dev/null || true
        modprobe -r kvm_intel 2>/dev/null || modprobe -r kvm_amd 2>/dev/null || true
        modprobe -r kvm 2>/dev/null || true
        echo -e "${VERT}KVM désactivé. VirtualBox peut démarrer.${FIN}"
        ;;
    2)
        if grep -q vmx /proc/cpuinfo; then modprobe kvm_intel; 
        elif grep -q svm /proc/cpuinfo; then modprobe kvm_amd; fi
        systemctl start vmware 2>/dev/null || true
        echo -e "${VERT}KVM/VMware réactivés.${FIN}"
        ;;
esac

```

### 2. Création d'un alias pour un accès rapide

Pour ne pas avoir à chercher le script dans vos dossiers, ajoutez un alias dans votre fichier `~/.bash_aliases`.

Exemple si votre script est dans `~/Scrypts/` :

```bash
# hv-switch : Bascule entre VirtualBox et KVM/VMware
alias hv-switch='sudo ~/Scrypts/hv-switch.sh'

```

Après avoir rechargé votre terminal (`source ~/.bashrc`), il vous suffira de taper **`hv-switch`** pour choisir votre mode de travail.

---

## Résumé du fonctionnement

* **Mode 1 (VirtualBox) :** Tue les processus KVM et retire les modules noyau `kvm`. Le processeur devient libre pour le pilote `vboxdrv`.
* **Mode 2 (KVM/VMware) :** Recharge les modules `kvm_intel` ou `kvm_amd` et relance les services VMware.

**Note importante :** Vous ne pouvez pas faire tourner une VM VirtualBox et une VM KVM **simultanément** avec l'accélération matérielle activée.
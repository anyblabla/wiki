---
title: Gestion à distance de l'alimentation d'un parc Proxmox
description: Apprenez à piloter à distance le démarrage et l'arrêt de vos serveurs Proxmox. Ce guide détaille l'usage du Wake-on-LAN et la configuration des clés SSH pour une gestion automatisée du cluster.
published: true
date: 2026-03-31T23:43:35.130Z
tags: proxmox, ssh, bash, sécurité, wol, wake-on-lan, énergie
editor: markdown
dateCreated: 2026-03-31T23:43:35.130Z
---

Ce guide explique comment automatiser le démarrage (via Wake-on-LAN) ainsi que l'arrêt ou le redémarrage (via SSH) de vos nœuds Proxmox à partir d'un script centralisé.

## 1. Prérequis techniques

Pour que ce script fonctionne, plusieurs configurations préalables sont nécessaires sur vos serveurs et votre machine d'administration.

### Configuration du Wake-on-LAN (WOL)
1.  **Dans le BIOS/UEFI** : activez l'option "Wake on LAN", "Remote Wake Up" ou "Power on by PCI-E".
2.  **Dans Proxmox** : vérifiez que votre interface réseau supporte le WOL avec la commande `ethtool eth0` (remplacez `eth0` par le nom de votre interface). La ligne `Wake-on: g` doit apparaître.
3.  **Côté client** : installez l'utilitaire de réveil sur votre machine d'administration :
    ```bash
    sudo apt install wakeonlan
    ```

## 2. Mise en place de l'authentification par clé SSH

L'automatisation repose sur la capacité de votre machine à se connecter aux serveurs sans saisir de mot de passe à chaque commande.

### Étape A : créer une paire de clés SSH
Si vous n'avez pas encore de clé SSH sur votre machine locale, vous devez en générer une. L'algorithme **ED25519** est recommandé pour sa sécurité et sa rapidité.

1.  Ouvrez votre terminal et tapez :
    ```bash
    ssh-keygen -t ed25519 -C "votre_email@exemple.com"
    ```
2.  Appuyez sur **Entrée** pour accepter l'emplacement par défaut.
3.  Pour un script totalement automatisé, ne définissez pas de "passphrase" (appuyez deux fois sur Entrée).

### Étape B : envoyer la clé sur chaque serveur
Une fois la clé générée, vous devez copier la "clé publique" sur chaque nœud Proxmox. Cela autorisera votre machine à se connecter.

Utilisez la commande suivante pour chaque serveur :
```bash
ssh-copy-id root@IP_DU_SERVEUR
```
*Le système vous demandera le mot de passe root du serveur une dernière fois pour installer la clé. Une fois cela fait, la connexion se fera automatiquement.*

## 3. Création du script de gestion

Créez un fichier nommé `pve_power.sh` dans votre dossier de scripts :
```bash
nano ~/Scripts/pve_power.sh
```

Copiez le code suivant en adaptant la section de configuration (noms, adresses IP et adresses MAC) :

```bash
#!/bin/bash

# --- CONFIGURATION ---
# Liste des serveurs : "NOM" "IP" "ADRESSE_MAC"
SERVERS=(
    "PVE-01" "192.168.1.10" "AA:BB:CC:DD:EE:01"
    "PVE-02" "192.168.1.11" "AA:BB:CC:DD:EE:02"
    "PBS-01" "192.168.1.20" "AA:BB:CC:DD:EE:03"
)

ACTION=$1

case $ACTION in
    start)
        echo "--- Démarrage des serveurs (Wake-on-LAN) ---"
        for ((i=0; i<${#SERVERS[@]}; i+=3)); do
            echo "Envoi du paquet magique à ${SERVERS[i]} (${SERVERS[i+2]})..."
            wakeonlan ${SERVERS[i+2]}
        done
        ;;
    stop)
        echo "--- Arrêt des serveurs via SSH ---"
        for ((i=0; i<${#SERVERS[@]}; i+=3)); do
            echo "Arrêt de ${SERVERS[i]} (${SERVERS[i+1]})..."
            ssh root@${SERVERS[i+1]} "poweroff"
        done
        ;;
    reboot)
        echo "--- Redémarrage des serveurs via SSH ---"
        for ((i=0; i<${#SERVERS[@]}; i+=3)); do
            echo "Redémarrage de ${SERVERS[i]} (${SERVERS[i+1]})..."
            ssh root@${SERVERS[i+1]} "reboot"
        done
        ;;
    *)
        echo "Usage: $0 {start|stop|reboot}"
        exit 1
        ;;
esac
```

## 4. Mise en service et utilisation

1.  **Rendre le script exécutable** :
    ```bash
    chmod +x ~/Scripts/pve_power.sh
    ```
2.  **Créer des alias pratiques** (à ajouter dans votre fichier `~/.bash_aliases` ou `~/.bashrc`) :
    ```bash
    alias pve-on='~/Scripts/pve_power.sh start'
    alias pve-off='~/Scripts/pve_power.sh stop'
    alias pve-reboot='~/Scripts/pve_power.sh reboot'
    ```
3.  **Prendre en compte les alias** :
    ```bash
    source ~/.bashrc
    ```

### Pourquoi utiliser ce script ?
L'utilisation combinée du **Wake-on-LAN** et du **SSH** permet de gérer dynamiquement la consommation électrique de votre infrastructure. C'est une solution idéale pour les environnements de test qui ne nécessitent pas de fonctionner 24 heures sur 24.

---

> **Auteur** : ce guide est proposé par **Amaury aka BlablaLinux**. Retrouvez l'ensemble de mes services sur [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).
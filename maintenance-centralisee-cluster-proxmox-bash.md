---
title: Maintenance centralisée d'un parc Proxmox (PVE et PBS)
description: Apprenez à créer un script Bash pour automatiser la maintenance de vos serveurs Proxmox VE et PBS. Centralisez les mises à jour et le nettoyage de votre cluster depuis une seule machine Linux.
published: true
date: 2026-03-31T23:30:22.756Z
tags: proxmox, ssh, debian, bash, linux, sysadmin, automatisation
editor: markdown
dateCreated: 2026-03-31T23:30:22.756Z
---

Ce guide explique comment concevoir un script Bash pour mettre à jour automatiquement plusieurs serveurs Proxmox VE et Proxmox Backup Server à partir d'une seule machine d'administration (ordinateur portable ou station de travail sous Linux).

## 1. Prérequis indispensables

Avant de déployer le script, votre machine d'administration doit pouvoir communiquer avec les serveurs cibles de manière fluide et sécurisée.

### Authentification par clé SSH (sans mot de passe)
C'est le point le plus important : le script utilise SSH. Pour qu'il ne s'arrête pas à chaque nœud pour demander un mot de passe, vous devez copier votre clé publique sur chaque serveur :

1.  **Génération d'une clé** (si vous n'en avez pas encore) : `ssh-keygen -t ed25519`
2.  **Copie de la clé sur chaque serveur** :
    ```bash
    ssh-copy-id root@IP_DU_SERVEUR
    ```
    *Répétez l'opération pour chaque nœud de votre parc.*

### Organisation des fichiers
Assurez-vous de disposer d'un dossier pour vos scripts personnels, par exemple `~/Scripts/`, et vérifiez que vous avez les droits d'écriture dedans.

## 2. Création du script de maintenance

Créez un nouveau fichier nommé `update_proxmox.sh` :
```bash
nano ~/Scripts/update_proxmox.sh
```

Copiez le code suivant en **adaptant les adresses IP** dans la section de configuration :

```bash
#!/bin/bash

# --- CONFIGURATION ---
# Remplacez les adresses IP ci-dessous par celles de vos serveurs (PVE et PBS)
NODES=(
    "192.168.1.10"
    "192.168.1.11"
    "192.168.1.20"
)

echo "--- Lancement de la maintenance complète ---"

for IP in "${NODES[@]}"
do
    echo "----------------------------------------------------"
    echo "Connexion à : $IP..."
    
    # Exécution des commandes de maintenance
    # On force le mode non-interactif pour éviter les blocages sur les questions système
    ssh root@$IP "export DEBIAN_FRONTEND=noninteractive; \
                  apt-get update && \
                  apt-get dist-upgrade -y && \
                  apt-get autoremove -y && \
                  apt-get autoclean"
    
    if [ $? -eq 0 ]; then
        echo "[OK] Terminé avec succès sur $IP"
    else
        echo "[ERREUR] Problème rencontré sur $IP"
    fi
done

echo "----------------------------------------------------"
echo "--- Maintenance terminée sur l'ensemble du parc ---"
```

## 3. Explication des commandes utilisées

* **`DEBIAN_FRONTEND=noninteractive`** : Cette variable indique au système de ne pas poser de questions lors des mises à jour (comme la validation d'un nouveau fichier de configuration) et d'utiliser l'option par défaut.
* **`apt-get dist-upgrade -y`** : Cette commande est cruciale pour Proxmox. Contrairement à un simple `upgrade`, elle gère les changements de dépendances et l'installation des nouveaux noyaux spécifiques à Proxmox (`proxmox-kernel`).
* **`apt-get autoremove -y`** : Cette ligne permet de supprimer automatiquement les anciens noyaux et les paquets devenus inutiles, évitant ainsi de saturer inutilement l'espace disque.

## 4. Mise en service et automatisation

1.  **Rendre le script exécutable** :
    ```bash
    chmod +x ~/Scripts/update_proxmox.sh
    ```
2.  **Créer un alias pour un lancement rapide** (à ajouter dans votre fichier `~/.bashrc` ou `~/.bash_aliases`) :
    ```bash
    alias pve-update='~/Scripts/update_proxmox.sh'
    ```
3.  **Prendre en compte les changements** :
    ```bash
    source ~/.bashrc
    ```

Désormais, la commande simple **`pve-update`** suffit pour maintenir l'ensemble de votre infrastructure à jour et propre.

---

> **Auteur** : ce guide est proposé par **Amaury aka BlablaLinux**. Retrouvez l'ensemble de mes services sur [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).
---
title: Installation de Docker et Portainer (Environnement Debian/LXC)
description: Ce guide fournit les commandes d'installation et de désinstallation pour Docker Engine et Portainer sur une machine hôte ou invitée utilisant une distribution basée sur Debian (comme un Container LXC).
published: true
date: 2025-10-28T13:38:14.310Z
tags: docker, lxc, proxmox, container, debian
editor: markdown
dateCreated: 2025-02-24T22:06:12.187Z
---

> **IMPORTANT :** Toutes les commandes Docker et Portainer ci-dessous (sauf le script Proxmox) doivent être exécutées **à l'intérieur** de votre [container LXC](https://fr.wikipedia.org/wiki/LXC), via le [shell](https://fr.wikipedia.org/wiki/Shell_Unix) du container ou via [SSH](https://fr.wikipedia.org/wiki/Secure_Shell).

-----

## 1\. Installation de Docker Engine

Vous avez le choix entre l'installation via le dépôt officiel APT de Docker (recommandé pour la stabilité) ou via le script d'installation rapide.

### Option A : Installation via le Dépôt Officiel APT (Recommandé)

Cette méthode configure les sources APT officielles de Docker pour une gestion et des mises à jour standardisées.

1.  **Rafraîchir les dépôts et installer les dépendances :**
    ```plaintext
    apt update
    apt install ca-certificates curl -y
    ```
2.  **Ajouter la clé GPG officielle de Docker :**
    ```plaintext
    install -m 0755 -d /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
    chmod a+r /etc/apt/keyrings/docker.asc
    ```
3.  **Ajouter le dépôt aux sources APT :**
    ```plaintext
    echo \
      "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
      $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
      tee /etc/apt/sources.list.d/docker.list > /dev/null
    apt update
    ```
4.  **Installer les paquets Docker :**
    ```plaintext
    apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
    ```

### Option B : Installation via le Script Officiel Docker (Méthode rapide)

Cette méthode utilise un script pour effectuer la détection du système et l'installation.

1.  **Préparer et télécharger le script :**
    ```plaintext
    apt update
    apt install curl -y
    curl -fsSL https://get.docker.com -o get-docker.sh
    ```
2.  **Tester ce que fera le script (Optionnel) :**
    ```plaintext
    sh get-docker.sh --dry-run
    ```
3.  **Installer Docker :**
    ```plaintext
    sh get-docker.sh
    ```

-----

## 2\. Gestion de Docker

### Désinstallation complète de Docker

Pour désinstaller Docker et supprimer toutes les données persistantes :

1.  **Désinstaller les paquets :**
    ```plaintext
    apt purge docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-ce-rootless-extras -y
    ```
2.  **Supprimer les images, conteneurs et volumes/données persistantes :**
    ```plaintext
    rm -rf /var/lib/docker
    rm -rf /var/lib/containerd
    ```
3.  **Nettoyer les sources APT et les trousseaux de clés :**
    ```plaintext
    rm /etc/apt/sources.list.d/docker.list
    rm /etc/apt/keyrings/docker.asc
    ```

-----

## 3\. Installation et Gestion de Portainer

[Portainer](https://www.portainer.io) est une interface graphique de gestion pour Docker, facilitant le déploiement de conteneurs et de piles.

### Installation de Portainer

1.  **Installer le paquet Docker de la distribution (si non déjà fait) :**

    ```plaintext
    apt install docker.io -y
    ```

2.  **Lancer le conteneur Portainer CE :**

    ```plaintext
    docker run -d \
    --name="portainer" \
    --restart always \
    -p 9000:9000 \
    -p 8000:8000 \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v portainer_data:/data \
    portainer/portainer-ce:latest
    ```

    *Le conteneur est configuré pour écouter sur le port **9000** (interface web) et le port **8000** (port d'Edge Agent, optionnel).*

3.  **Accès à l'interface :** Après quelques secondes, accédez à Portainer en visitant **`http://<IP_du_LXC>:9000`** pour la première configuration.

### Désinstallation de Portainer

Pour arrêter et supprimer le conteneur Portainer et ses images :

```plaintext
docker stop portainer && docker rm portainer && docker image prune -a
```

-----

## 4\. BONUS : Proxmox VE Helper-Scripts

Si vous utilisez **Proxmox VE** comme hyperviseur, vous pouvez automatiser la création d'un container LXC Debian avec Docker et Portainer préinstallés.

> **ATTENTION :** Cette commande doit être exécutée **dans le shell Proxmox VE (l'hôte)**, et non pas dans un container.

```plaintext
bash -c "$(wget -qLO - https://github.com/community-scripts/ProxmoxVE/raw/main/ct/docker.sh)"
```

Le script vous proposera différentes options d'installation, y compris Portainer.

-----

## Liens utiles

  - [Proxmox VE](https://www.proxmox.com/en/products/proxmox-virtual-environment/overview)
  - [Linux Containers](https://linuxcontainers.org)
  - [Docker](https://www.docker.com)
  - [Portainer](https://www.portainer.io)
  - [Proxmox VE Helper-Scripts](https://community-scripts.github.io/ProxmoxVE/)
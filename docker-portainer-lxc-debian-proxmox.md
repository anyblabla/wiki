---
title: Docker/Portainer LXC Debian Proxmox
description: Plusieurs méthodes d'installation de Docker (Portainer) dans un container LXC Debian sur Proxmox VE.
published: true
date: 2025-04-19T22:49:01.218Z
tags: docker, lxc, proxmox, container, debian
editor: markdown
dateCreated: 2025-02-24T22:06:12.187Z
---

# Docker

## Installation

**Les commandes ci-dessous sont à effectuer _à l'intérieur_ de votre** [**container LXC**](https://fr.wikipedia.org/wiki/LXC)**, soit via le** [**shell**](https://fr.wikipedia.org/wiki/Shell_Unix) **du container, ou via** [**SSH**](https://fr.wikipedia.org/wiki/Secure_Shell)**.**

### À partir des dépôts APT officiels Docker

Avant d'installer [Docker](https://fr.wikipedia.org/wiki/Docker_(logiciel)) Engine (moteur) pour la première fois sur une nouvelle machine hôte ou invitée, vous devez configurer le [dépôt APT](https://wiki.debian.org/fr/SourcesList) [Docker](https://www.docker.com). Vous pouvez ensuite installer et mettre à jour Docker à partir des dépôts :

-   Rafraîchir les dépôts :

```plaintext
apt update
```

-   Configurer le dépôt APT de Docker :

```plaintext
# Add Docker's official GPG key:
apt update
apt install ca-certificates curl -y
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null
apt update
```

-   Installez les paquets Docker :

```plaintext
apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```

### À partir du script officiel Docker

-   Rafraîchir les dépôts :

```plaintext
apt update
```

-   Installation de “[curl](https://fr.wikipedia.org/wiki/CURL)” :

```plaintext
apt install curl -y
```

-   Téléchargement du [script](https://fr.wikipedia.org/wiki/Script) :

```plaintext
curl -fsSL https://get.docker.com -o get-docker.sh
```

-   Tester ce que va faire le script :

```plaintext
sh get-docker.sh --dry-run
```

-   Installer Docker avec le script :

```plaintext
sh get-docker.sh
```

Script Docker également disponible sur [ByteStash Blabla Linux](https://bytestash.blablalinux.be/public/snippets).

## Désinstallation

-   Désinstaller les paquets Docker :

```plaintext
apt purge docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-ce-rootless-extras -y
```

-   Les images, les conteneurs, volumes ou fichiers de configuration personnalisés sur votre système ne sont pas automatiquement supprimés. Pour supprimer toutes les images, conteneurs et volumes :

```plaintext
rm -rf /var/lib/docker
rm -rf /var/lib/containerd
```

-   Supprimer la liste des sources et les trousseaux de clés :

```plaintext
rm /etc/apt/sources.list.d/docker.list
rm /etc/apt/keyrings/docker.asc
```

# Portainer

## Installation

```plaintext
apt install docker.io -y
```

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

Le système va maintenant extraire la dernière image [Portainer](https://www.portainer.io) et configurer le conteneur exécuté sur le port 9000. Vous pourrez accéder à Portainer en visitant http://ip:9000.

## Désinstallation

```plaintext
docker stop portainer && docker rm portainer && docker image prune -a
```

# BONUS Proxmox VE Helper-Scripts

-   Pour créer directement un nouveau container LXC Debian avec Docker sur [Proxmox VE](https://www.proxmox.com/en/products/proxmox-virtual-environment/overview), **exécutez la commande ci-dessous _dans_ le shell Proxmox VE** :

```plaintext
bash -c "$(wget -qLO - https://github.com/community-scripts/ProxmoxVE/raw/main/ct/docker.sh)"
```

Le script vous proposera différentes options d'installations, comme Portainer.

# Liens utiles

-   [Proxmox VE](https://www.proxmox.com/en/products/proxmox-virtual-environment/overview)
-   [Linux Containers](https://linuxcontainers.org)
-   [Docker](https://www.docker.com)
-   [Portainer](https://www.portainer.io)
-   [Proxmox VE Helper-Scripts](https://community-scripts.github.io/ProxmoxVE/)
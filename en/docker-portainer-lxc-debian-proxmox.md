---
title: Installing Docker and Portainer (Debian/LXC Environment)
description: >-
  This guide provides installation and uninstallation commands for Docker Engine
  and Portainer on a host or guest machine running a Debian-based distribution
  (such as an LXC Container).
published: true
date: '2026-11-22T19:42:48.262Z'
tags:
  - docker
  - lxc
  - proxmox
  - container
  - debian
editor: markdown
dateCreated: '2026-11-22T19:42:48.262Z'
---

> **IMPORTANT:** All Docker and Portainer commands below (except the Proxmox script) must be executed **inside** your [LXC container](https://en.wikipedia.org/wiki/LXC), via the container's [shell](https://en.wikipedia.org/wiki/Unix_shell) or via [SSH](https://en.wikipedia.org/wiki/Secure_Shell).

-----

## 1\. Installing Docker Engine

You can choose between installation via Docker's official APT repository (recommended for stability) or via the quick install script.

### Option A: Installation via the Official APT Repository (Recommended)

This method sets up Docker's official APT sources for standardized management and updates.

1.  **Refresh repositories and install dependencies:**
    ```plaintext
    apt update
    apt install ca-certificates curl -y
    ```
2.  **Add Docker's official GPG key:**
    ```plaintext
    install -m 0755 -d /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
    chmod a+r /etc/apt/keyrings/docker.asc
    ```
3.  **Add the repository to APT sources:**
    ```plaintext
    echo \
      "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
      $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
      tee /etc/apt/sources.list.d/docker.list > /dev/null
    apt update
    ```
4.  **Install Docker packages:**
    ```plaintext
    apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
    ```

### Option B: Installation via the Official Docker Script (Quick Method)

This method uses a script to handle system detection and installation.

1.  **Prepare and download the script:**
    ```plaintext
    apt update
    apt install curl -y
    curl -fsSL https://get.docker.com -o get-docker.sh
    ```
2.  **Test what the script will do (Optional):**
    ```plaintext
    sh get-docker.sh --dry-run
    ```
3.  **Install Docker:**
    ```plaintext
    sh get-docker.sh
    ```

-----

## 2\. Managing Docker

### Complete Docker Uninstallation

To uninstall Docker and remove all persistent data:

1.  **Uninstall packages:**
    ```plaintext
    apt purge docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-ce-rootless-extras -y
    ```
2.  **Remove images, containers, and persistent volumes/data:**
    ```plaintext
    rm -rf /var/lib/docker
    rm -rf /var/lib/containerd
    ```
3.  **Clean APT sources and keyrings:**
    ```plaintext
    rm /etc/apt/sources.list.d/docker.list
    rm /etc/apt/keyrings/docker.asc
    ```

-----

## 3\. Installing and Managing Portainer

[Portainer](https://www.portainer.io) is a graphical management interface for Docker, making container and stack deployment easier.

### Installing Portainer

1.  **Install the distribution's Docker package (if not already done):**

    ```plaintext
    apt install docker.io -y
    ```

2.  **Run the Portainer CE container:**

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

    *The container is configured to listen on port **9000** (web interface) and port **8000** (Edge Agent port, optional).*

3.  **Accessing the interface:** After a few seconds, access Portainer by visiting **`http://<LXC_IP>:9000`** for the initial configuration.

### Uninstalling Portainer

To stop and remove the Portainer container and its images:

```plaintext
docker stop portainer && docker rm portainer && docker image prune -a
```

-----

## 4\. BONUS: Proxmox VE Helper-Scripts

If you use **Proxmox VE** as your hypervisor, you can automate the creation of a Debian LXC container with Docker and Portainer pre-installed.

> **WARNING:** This command must be executed **in the Proxmox VE shell (the host)**, and not inside a container.

```plaintext
bash -c "$(wget -qLO - https://github.com/community-scripts/ProxmoxVE/raw/main/ct/docker.sh)"
```

The script will offer you various installation options, including Portainer.

-----

## Useful Links

  - [Proxmox VE](https://www.proxmox.com/en/products/proxmox-virtual-environment/overview)
  - [Linux Containers](https://linuxcontainers.org)
  - [Docker](https://www.docker.com)
  - [Portainer](https://www.portainer.io)
  - [Proxmox VE Helper-Scripts](https://community-scripts.github.io/ProxmoxVE/)

-----

> **Author**: this guide is proposed by **Amaury aka BlablaLinux**. Find all my services on [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).

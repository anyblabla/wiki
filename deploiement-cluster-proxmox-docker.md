---
title: Guide de déploiement d'un cluster Proxmox à trois nœuds avec Docker
description: Apprenez à déployer un cluster Proxmox VE complet avec trois nœuds via Docker Compose. Un guide idéal pour tester le quorum, la haute disponibilité et le réseau sans matériel supplémentaire.
published: true
date: 2026-01-03T21:56:45.212Z
tags: docker, proxmox, pve, cluster, virtualisation, ha, haute disponibilité, lab
editor: markdown
dateCreated: 2026-01-03T21:50:40.907Z
---

Ce guide explique comment faire fonctionner un environnement **Proxmox Virtual Environment (PVE)** complet à l'intérieur de conteneurs Docker. C'est la solution parfaite pour tester des scénarios de haute disponibilité (HA), de quorum ou de migration sans mobiliser plusieurs serveurs physiques.

> **Note :** Pour installer Docker avant de commencer, vous pouvez consulter cette page : [Installer Docker, Portainer et les agents LXC sur Debian/Proxmox](https://wiki.blablalinux.be/fr/docker-portainer-lxc-debian-proxmox).

## Prérequis du système

Avant de commencer, assurez-vous que votre système hôte remplit les conditions suivantes :

* **Noyau Linux :** Version 6.14 ou supérieure recommandée.
* **Virtualisation :** Le processeur doit supporter la virtualisation matérielle (VT-x ou AMD-V) activée dans le BIOS.
* **Modules :** Les modules `kvm` et `tun` doivent être chargés (`modprobe kvm`).
* **Note pour Docker Desktop :** Si vous êtes sur Windows/macOS, supprimez la ligne `/sys/kernel/security` du fichier YAML pour éviter un échec au démarrage.

## Configuration du fichier Docker Compose

Ce fichier définit trois nœuds indépendants communiquant via un réseau privé Docker avec support IPv6 pour une simulation réseau plus réaliste.

```yaml
services:
  pve-1:
    image: ghcr.io/longqt-sea/proxmox-ve
    container_name: pve-1
    hostname: pve-1
    privileged: true
    restart: unless-stopped
    cgroup: host
    ports:
      - "8006:8006" # Interface web pve-1
      - "2222:22"   # Accès SSH pve-1
      - "3128:3128" # Proxy SPICE
    networks:
      - pve_lab_net
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
      - /usr/lib/modules:/usr/lib/modules:ro
      - /sys/kernel/security:/sys/kernel/security
      - ./VM-Backup:/var/lib/vz/dump
      - ./ISOs:/var/lib/vz/template/iso
  pve-2:
    image: ghcr.io/longqt-sea/proxmox-ve
    container_name: pve-2
    hostname: pve-2
    privileged: true
    restart: unless-stopped
    cgroup: host
    ports:
      - "8007:8006" # Interface web pve-2
      - "2223:22"   # Accès SSH pve-2
      - "3129:3128" # Proxy SPICE
    networks:
      - pve_lab_net
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
      - /usr/lib/modules:/usr/lib/modules:ro
      - /sys/kernel/security:/sys/kernel/security
      - ./VM-Backup:/var/lib/vz/dump
      - ./ISOs:/var/lib/vz/template/iso
  pve-3:
    image: ghcr.io/longqt-sea/proxmox-ve
    container_name: pve-3
    hostname: pve-3
    privileged: true
    restart: unless-stopped
    cgroup: host
    ports:
      - "8008:8006" # Interface web pve-3
      - "2224:22"   # Accès SSH pve-3
      - "3130:3128" # Proxy SPICE
    networks:
      - pve_lab_net
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
      - /usr/lib/modules:/usr/lib/modules:ro
      - /sys/kernel/security:/sys/kernel/security
      - ./VM-Backup:/var/lib/vz/dump
      - ./ISOs:/var/lib/vz/template/iso
networks:
  pve_lab_net:
    enable_ipv6: true
    driver: bridge
    ipam:
      config:
        - subnet: fd00::/48

```

## Étapes d'initialisation du cluster

### 1. Configuration des accès root

Après avoir exécuté `docker compose up -d`, vous devez définir manuellement les mots de passe pour accéder aux interfaces web :

```bash
docker exec -it pve-1 passwd
docker exec -it pve-2 passwd
docker exec -it pve-3 passwd

```

**Important :** Redémarrez les conteneurs après cette étape : `docker restart pve-1 pve-2 pve-3`.

### 2. Création du cluster (Quorum)

* Connectez-vous sur **pve-1** : `https://localhost:8006`.
* Allez dans **Datacenter > Cluster** et créez un cluster nommé "Lab-Docker".
* Copiez le **Join Information**.
* Connectez-vous sur **pve-2** (port 8007) et **pve-3** (port 8008) pour rejoindre le cluster en utilisant les identifiants de pve-1.

## Configuration du réseau et du stockage

### Gestion du réseau interne

Les conteneurs Proxmox arrivent avec deux ponts réseau pré-configurés :

* **`vmbr1` (NAT) :** Connecté au réseau Docker, il permet aux machines virtuelles d'accéder immédiatement à Internet.
* **`vmbr2` (Vide) :** Un pont libre pour vos propres configurations (Macvlan, Veth, ou passthrough).

### Stockage partagé

Les volumes `./ISOs` et `./VM-Backup` sont montés sur les trois nœuds simultanément. Cela simule un **stockage partagé** (type NFS/NAS), permettant de tester la **migration à chaud (Live Migration)** entre les nœuds sans déplacer les fichiers de disques.

## Analyse des limitations (Lab vs Production)

| Caractéristique | Comportement dans Docker | Impact sur l'apprentissage |
| --- | --- | --- |
| **Performance** | Inférieure au bare-metal | Négligeable pour le test de fonctionnalités. |
| **Passthrough PCI** | Très limité et complexe | Ne convient pas pour le passthrough GPU. |
| **Réalisme HA** | Logique identique au physique | Excellent pour comprendre le quorum et le fencing. |
| **Réseau** | Abstraction Docker | Suffisant pour apprendre les bridges et VLANs. |
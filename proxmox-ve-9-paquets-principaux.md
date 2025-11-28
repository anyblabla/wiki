---
title: Paquets Principaux de Proxmox Virtual Environment (PVE) 9.1.x
description: Guide complet des paquets principaux de Proxmox VE 9.1.x, y compris KVM, LXC, clustering (Corosync), et outils CLI (qm, pct). Idéal pour l'administration système.
published: true
date: 2025-11-28T23:03:37.590Z
tags: qemu, lxc, proxmox, debian, pve, cluster, corosync, paquets, packages, virtualisation, dkms, ha, pvesh, qm, pct, sysadmin, hyperviseur, zfs
editor: markdown
dateCreated: 2025-11-28T22:51:32.025Z
---

Cette documentation s'applique spécifiquement aux installations **Proxmox Virtual Environment (PVE) version 9.1.x**, qui est basée sur la branche de développement **Debian 13 "Trixie"**.

**Proxmox VE** est une plateforme de virtualisation open-source complète combinant les technologies de conteneurisation (**LXC**) et de virtualisation complète (**KVM**). Son architecture est modulaire, reposant sur plusieurs paquets logiciels clés gérant l'interface, le clustering, le stockage, et les machines virtuelles/conteneurs.

***

## 📦 Paquets du noyau et du système d'exploitation

| Paquet | Description | Rôle Principal | Lien de documentation |
| :--- | :--- | :--- | :--- |
| **proxmox-ve** | Le **méta-paquet** principal. Assure l'installation et le maintien de toutes les dépendances essentielles. | Installation et dépendances du système PVE. | [Documentation Proxmox VE](https://pve.proxmox.com/pve-docs/) |
| **proxmox-kernel-6.17...** | Les paquets du **noyau Linux** spécifiques à Proxmox (version 6.17 dans PVE 9.1), optimisés pour la virtualisation. | Cœur du système d'exploitation et support des hyperviseurs. | [Documentation noyau](https://searxng.blablalinux.be/search?q=https://pve.proxmox.com/wiki/Kernel) |

***

## ☁️ Gestion de la virtualisation (KVM/QEMU et LXC)

Ces paquets sont le cœur des fonctions d'hyperviseur de Proxmox. 

| Paquet | Description | Rôle Principal | Lien de documentation |
| :--- | :--- | :--- | :--- |
| **pve-qemu-kvm** | La version de **QEMU/KVM** adaptée par Proxmox. **KVM** est l'hyperviseur pour l'exécution des **machines virtuelles (VM)**. | Exécution des machines virtuelles (VM). | [Documentation QEMU/KVM](https://searxng.blablalinux.be/search?q=https://pve.proxmox.com/wiki/KVM) |
| **qemu-server** | Ensemble d'outils et de scripts qui intègrent QEMU/KVM dans PVE. | Intégration et gestion des VM dans PVE. | [Documentation QEMU/KVM](https://searxng.blablalinux.be/search?q=https://pve.proxmox.com/wiki/KVM) |
| **lxc-pve** | La version de **Linux Containers (LXC)** adaptée. **LXC** permet d'exécuter des **conteneurs** isolés, légers et partageant le noyau de l'hôte. | Exécution des conteneurs. | [Documentation LXC](https://pve.proxmox.com/wiki/Linux_Container) |
| **pve-container** | Outils et scripts pour la gestion des conteneurs LXC via l'interface Proxmox. | Intégration et gestion des conteneurs dans PVE. | [Documentation LXC](https://pve.proxmox.com/wiki/Linux_Container) |

***

## 🛠️ Gestion et interface (Web, API)

| Paquet | Description | Rôle Principal | Lien de documentation |
| :--- | :--- | :--- | :--- |
| **pve-manager** | Le service de l'**API Web** qui fournit la gestion centrale et l'interface utilisateur graphique (GUI). | Interface utilisateur Web et gestion centrale du nœud. | [Documentation API](https://pve.proxmox.com/pve-docs/api-viewer/) |
| **libpve-access-control** | Gère les mécanismes d'**authentification** et d'**autorisation** (utilisateurs, groupes, rôles). | Sécurité et gestion des utilisateurs. | [Documentation utilisateurs/droits](https://pve.proxmox.com/wiki/User_Management) |
| **novnc-pve** | Permet l'accès aux consoles des machines virtuelles via **VNC** directement dans le navigateur. | Accès distant à la console des VM via Web. | Non spécifique |

***

## 🔗 Clustering et haute disponibilité (HA)

Ces paquets sont cruciaux pour créer un **cluster** et gérer la **Haute Disponibilité**. 

| Paquet | Description | Rôle Principal | Lien de documentation |
| :--- | :--- | :--- | :--- |
| **corosync** | Le moteur de **messagerie de cluster** qui assure la communication rapide et la synchronisation de l'état (*ring*). | Communication inter-nœuds et gestion du *quorum*. | [Documentation cluster](https://searxng.blablalinux.be/search?q=https://pve.proxmox.com/wiki/Cluster) |
| **pve-cluster** | L'intégration de Corosync et d'autres outils pour la gestion centralisée de la configuration du cluster (`/etc/pve`). | Gestion de la configuration et du *quorum* du cluster. | [Documentation cluster](https://searxng.blablalinux.be/search?q=https://pve.proxmox.com/wiki/Cluster) |
| **pve-ha-manager** | Le service qui gère la **Haute Disponibilité (HA)**, surveillant et orchestrant le redémarrage automatique des invités. | Gestion de la haute disponibilité des invités. | [Documentation HA](https://pve.proxmox.com/wiki/High_Availability) |

***

## 💾 Stockage, sauvegarde et réseau

| Paquet | Description | Rôle Principal | Lien de documentation |
| :--- | :--- | :--- | :--- |
| **libpve-storage-perl** | Bibliothèque qui gère la configuration et l'interaction avec les différents **backends de stockage**. | Gestion des ressources de stockage. | [Documentation stockage](https://pve.proxmox.com/wiki/Storage) |
| **proxmox-backup-client** | L'outil de ligne de commande pour la sauvegarde vers un **Proxmox Backup Server (PBS)** ou en local. | Client de sauvegarde/restauration. | [Documentation PBS client](https://searxng.blablalinux.be/search?q=https://pbs.proxmox.com/docs/client.html) |
| **pve-firewall** | Le service de **pare-feu** intégré, permettant de filtrer le trafic au niveau du nœud PVE et pour chaque invité. | Sécurité réseau (pare-feu). | [Documentation pare-feu](https://pve.proxmox.com/wiki/Firewall) |
| **zfsutils-linux** | Outils natifs pour gérer les *pools* et *datasets* **ZFS**. | Support et gestion de ZFS. | [Documentation ZFS](https://pve.proxmox.com/wiki/ZFS) |

***

## ⌨️ Outils d'administration en ligne de commande

| Commande | Paquet Associé | Description | Lien de documentation (Man pages) |
| :--- | :--- | :--- | :--- |
| **qm** | `qemu-server` | Gère les **Machines Virtuelles (VM)** KVM/QEMU. | [Man page qm](https://pve.proxmox.com/pve-docs/qm.1.html) |
| **pct** | `pve-container` | Gère les **Conteneurs (CT)** LXC. | [Man page pct](https://pve.proxmox.com/pve-docs/pct.1.html) |
| **pvesh** | `pve-manager` | Le **Shell de l'API Proxmox**. Permet d'interagir directement avec l'API Web. | [Man page pvesh](https://pve.proxmox.com/pve-docs/pvesh.1.html) |
| **pvecm** | `pve-cluster` | Gère le **Cluster Proxmox** (statut, ajout, retrait de nœuds). | [Man page pvecm](https://pve.proxmox.com/pve-docs/pvecm.1.html) |
| **ha-manager** | `pve-ha-manager` | Gère les ressources de **Haute Disponibilité (HA)**. | [Man page ha-manager](https://pve.proxmox.com/pve-docs/ha-manager.1.html) |
| **pve-firewall** | `pve-firewall` | Gère la configuration du **pare-feu** depuis la ligne de commande. | [Man page pve-firewall](https://searxng.blablalinux.be/search?q=https://pve.proxmox.com/pve-docs/pve-firewall.1.html) |
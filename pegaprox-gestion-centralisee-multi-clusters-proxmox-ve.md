---
title: PegaProx - Gestion centralisée multi-clusters Proxmox VE
description: Découvrez PegaProx, une solution open-source de gestion centralisée multi-clusters pour Proxmox VE. Idéal pour piloter tout votre parc de virtualisation depuis une interface unique.
published: true
date: 2026-05-23T20:26:54.197Z
tags: proxmox, cluster, virtualisation, hyperviseur, open-source, pegaprox
editor: markdown
dateCreated: 2026-05-23T20:24:18.332Z
---

[PegaProx](https://pegaprox.com) est une solution open-source et gratuite qui sert de console de gestion centralisée pour vos infrastructures de virtualisation. Elle est principalement conçue pour Proxmox VE (avec un support pour XCP-ng en cours de développement). 

On peut le comparer à ce que vCenter est pour VMware : une interface graphique unique et moderne pour piloter tout son parc.

---

## 📌 Fonctionnalités principales

### 🖥️ Gestion multi-clusters et tableau de bord unique
Plus besoin de sauter d'une interface Proxmox à une autre. PegaProx regroupe tous vos clusters et nœuds isolés au même endroit. Un tableau de bord global affiche en temps réel l'utilisation du CPU, de la RAM, du stockage et du réseau de l'ensemble de votre infrastructure.

### 🔄 Migrations avancées (cross-cluster et VMware)
* **Migration cross-cluster :** Déplacez vos machines virtuelles (VM) à chaud d'un cluster Proxmox à un autre, sans coupure de service.
* **Migration depuis VMware ESXi :** Intègre des outils pour importer et convertir facilement vos VM ESXi vers Proxmox VE avec un temps d'arrêt minimal.

### ⚖️ Répartition de charge (ProxLB) et haute disponibilité
* **Équilibrage automatique :** Le système surveille la santé des nœuds et peut répartir automatiquement la charge de travail entre vos différents serveurs.
* **Haute disponibilité (HA) :** Prise en charge du basculement automatique, y compris pour les petites infrastructures (comme les clusters à 2 nœuds).

### 💾 Stockage, sauvegarde et cycle de vie des VM
* Gestion complète des VM et conteneurs (création, snapshots, clones, alimentation).
* Pilotage des pools de stockage distribués Ceph.
* Intégration et gestion des serveurs de sauvegarde Proxmox Backup Server (PBS).

### 🔐 Sécurité, rôles et authentification d'entreprise
* **SSO et identité :** Connexion compatible avec les annuaires LDAP / Active Directory et les protocoles OIDC (Keycloak, Authentik, Google, Microsoft, etc.).
* **Droits fins (RBAC) :** Gestion précise des autorisations par utilisateur, par groupe ou par ressource.
* **Journal d'audit :** Historique complet des actions effectuées sur la plateforme pour une traçabilité totale.

---

## 🚀 Pourquoi l'adopter ?
* **Interface moderne :** Une interface utilisateur fluide, rapide et très soignée.
* **Centralisation :** Parfait dès que l'on gère plus d'un cluster ou plusieurs hyperviseurs indépendants.
* **Alternative libre :** Permet de retrouver le confort de gestion des solutions propriétaires (comme VMware vCenter) mais avec des outils 100% open-source.

---
> **Auteur** : ce guide est proposé par **Amaury aka BlablaLinux**. Retrouvez l'ensemble de mes services sur [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).

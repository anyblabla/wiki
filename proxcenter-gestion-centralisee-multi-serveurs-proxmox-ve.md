---
title: ProxCenter - Gestion centralisée multi-serveurs Proxmox VE
description: Découvrez ProxCenter, un panneau de contrôle open-source moderne pour centraliser la gestion de vos serveurs Proxmox VE et Proxmox Backup Server (PBS) depuis une seule interface web unifiée.
published: true
date: 2026-05-23T21:10:10.142Z
tags: proxmox, pbs, virtualisation, dashboard, open-source, proxcenter
editor: markdown
dateCreated: 2026-05-23T20:39:27.131Z
---

[ProxCenter](https://proxcenter.io) est un panneau de gestion (dashboard) web open-source et moderne conçu pour centraliser et simplifier la supervision de vos infrastructures de virtualisation. Il regroupe au même endroit le suivi de vos hyperviseurs Proxmox VE et de vos serveurs de sauvegarde Proxmox Backup Server (PBS).

L'outil est proposé en deux déclinaisons : une édition **Community** entièrement gratuite et auto-hébergée (incluant le dashboard, le monitoring, l'inventaire, la gestion PBS et les alertes basiques) et une édition **Enterprise** débloquant des fonctionnalités avancées (DRS, micro-segmentation, SSO, etc.).

Développé avec des technologies récentes comme Next.js et Tailwind CSS, il offre une interface fluide et légère pour garder un œil global sur vos ressources sans multiplier les onglets de connexion.

---

## 📌 Fonctionnalités principales

### 🖥️ Tableau de bord unifié et multi-serveurs
Regroupez l'ensemble de vos nœuds Proxmox VE indépendants et vos instances PBS sur une seule et unique interface graphique. Le tableau de bord offre une vue d'ensemble immédiate sur la charge processeur, l'utilisation de la mémoire, l'espace disque et l'état général de votre infrastructure.

### 💾 Intégration native de Proxmox Backup Server (PBS)
Contrairement aux outils de gestion classiques, ProxCenter intègre une surveillance étroite des sauvegardes. Vous pouvez suivre l'état de vos banques de données (datastores), vérifier la réussite ou l'échec des dernières tâches de sauvegarde et analyser les tendances d'utilisation du stockage.

### 🔄 Gestion du cycle de vie des VM et conteneurs
Pilotez directement vos machines virtuelles et conteneurs LXC depuis l'interface :
* Démarrage, arrêt, redémarrage et suspension.
* Accès à la console à distance.
* Graphiques de performance et métriques d'utilisation en temps réel pour chaque ressource.

### 🔑 Connexion sécurisée par jeton d'API
L'application se connecte à vos hyperviseurs en utilisant les clés d'API (API tokens) officielles de Proxmox VE et PBS. L'installation est rapide, 100% auto-hébergée sur votre infrastructure, non intrusive et ne nécessite aucune modification de la configuration de vos serveurs de production.

---

## 💎 Éditions : Community vs Enterprise

ProxCenter s'adapte à la taille de votre infrastructure grâce à ses deux versions, évolutives à tout moment sans perte de données :

* **Édition Community (Gratuite) :** Parfaite pour l'essentiel. Comprend le tableau de bord de supervision, le monitoring, l'inventaire complet, la gestion PBS et un système d'alertes basiques.
* **Édition Enterprise (Licence) :** Conçue pour les environnements de production exigeants. Elle ajoute l'équilibrage automatique de charge (DRS), la micro-segmentation réseau, le plan de reprise d'activité (Site Recovery), les scans de vulnérabilités (CVE), l'analyse par IA, la gestion avancée des droits (RBAC) avec support LDAP/AD/SSO, ainsi qu'un support technique prioritaire.

---

## 🚀 Cas d'utilisation

* **Supervision multi-serveurs :** Idéal pour les administrateurs et les passionnés d'autohébergement qui gèrent plusieurs serveurs Proxmox isolés et souhaitent centraliser leur suivi.
* **Gestion du stockage et des sauvegardes :** Parfait pour s'assurer en un coup d'œil que l'ensemble du parc de machines est correctement sauvegardé sur PBS.
* **Interface légère et moderne :** Offre une alternative d'administration simplifiée, rapide et esthétique pour les tâches courantes du quotidien.

---
> **Auteur** : ce guide est proposé par **Amaury aka BlablaLinux**. Retrouvez l'ensemble de mes services sur [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).
---
title: Stratégie de sauvegarde multi-niveaux sur Proxmox
description: Découvrez ma stratégie complète de sauvegarde sous Proxmox : snapshots automatisés, double redondance sur PBS avec rétention GFS, et protection granulaire des bases de données applicatives.
published: true
date: 2026-03-27T14:14:46.402Z
tags: proxmox, sauvegarde, snapshot, pbs, sqlite, databasus, devops
editor: markdown
dateCreated: 2026-03-27T13:43:45.239Z
---

> **Note importante** : Ce document décrit la stratégie de sauvegarde telle qu'elle est configurée actuellement. Cette architecture se rapproche au maximum de la mise en place réelle, mais elle est sujette à des évolutions fréquentes en fonction de certains critères techniques ou d'optimisation.

---

## 1. Snapshots automatisés (cv4pve-autosnap)
Les snapshots permettent un retour en arrière instantané (rollback) en cas d'erreur de configuration ou de mise à jour défaillante sur un service spécifique. Cette couche repose sur l'outil [cv4pve-autosnap](/fr/cv4pve-autosnap).

* **Services critiques (horaires)** : Gitea, Vaultwarden, 2FAuth, AdGuard, etc. Snapshot toutes les heures avec une rétention de 24h.
* **Snapshots généraux** : Toutes les VM et LXC sont snapshotés chaque jour à 00h10 (garde 7 jours) et chaque dimanche à 00h20 (garde 4 semaines). Pour plus de détails sur la mise en place technique, consultez mon guide sur l'[automatisation des snapshots Proxmox multi-nœuds](/fr/automatisation-snapshots-proxmox-multi-noeuds).
* **Maintenance** : Purge automatique des journaux de snapshot chaque jour à 05h00.

## 2. Sauvegardes Proxmox Backup Server (PBS) et redondance
C'est le cœur de la rétention à long terme avec un facteur de déduplication optimisé (actuellement supérieur à **17x**).

* **Flux de redondance locale** : Les données sont d'abord sauvegardées sur une première banque de données, puis répliquées immédiatement vers une deuxième banque de données sur le même serveur PBS.
* **Politique de rétention (GFS)** : 
    * **7** journaliers
    * **4** hebdomadaires
    * **3** mensuels
    * **1** annuel
* **Externalisation hors-site** : Une fois par mois, une synchronisation complète est effectuée vers un second serveur PBS distant pour une sécurité hors site totale.

## 3. Protection granulaire des bases de données
C'est un complément indispensable aux sauvegardes "bloc" pour garantir l'intégrité des données applicatives et faciliter les restaurations ciblées.

* **Databasus** : Sauvegarde et monitoring (healthcheck) des bases de données pour *Etherpad, FreshRSS, Gitea, Joplin, KitchenOwl*, etc. Fréquence : toutes les 12h (08h00 / 20h00). Retrouvez ma configuration sur [mon instance ByteStash](https://bytestash.blablalinux.be/s/e6bf4cd936e4e11db03f6348a6288eba).
* **Script SQLite "maison"** : Utilisation de mon script personnel pour l'export et la centralisation des bases SQLite multi-services vers un stockage dédié. Consultez le guide complet sur [la sauvegarde et restauration SQLite](/fr/sauvegarde-restauration-sqlite-multiservices-proxmox).

---
> **Auteur** : ce guide est proposé par **Amaury aka BlablaLinux**. Retrouvez l'ensemble de mes services sur [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).

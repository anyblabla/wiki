---
title: Migration d'un serveur Mumble d'une machine virtuelle Debian 12 vers un container LXC Debian 13
description: Guide complet pour migrer un service Mumble d'une VM Debian 12 vers un container LXC sous Debian 13 Trixie sur Proxmox VE. Optimisation des ressources et mise à jour des fichiers de configuration.
published: true
date: 2026-01-23T21:12:00.715Z
tags: proxmox, debian, migration, trixie, debian 13, mumble, debian 12, bookworm
editor: markdown
dateCreated: 2026-01-23T21:12:00.715Z
---

## Introduction

Ce guide documente la migration d'un serveur Mumble (Murmur) depuis une machine virtuelle **Debian 12 Bookworm** vers un container **LXC Debian 13 Trixie** hébergé sur un cluster **Proxmox VE**. Cette opération permet de réduire l'empreinte mémoire tout en utilisant la dernière version de Debian.

## Prérequis

* Un accès root sur l'ancienne machine virtuelle et le nouveau container.
* Une copie des fichiers essentiels : `mumble-server.ini` et `mumble-server.sqlite`.
* Un container LXC Debian 13 (Trixie) déjà créé.

## Procédure de migration

### 1. Installation du service sur le nouveau container

Commencez par mettre à jour les dépôts et installez le paquet du serveur :

```bash
apt update && apt install mumble-server -y
systemctl stop mumble-server

```

### 2. Restauration et adaptation de la configuration

Comme Debian 13 introduit de nouvelles options (audit des groupes, gestion des enregistreurs), il est préférable d'utiliser le fichier de configuration par défaut du nouveau système et d'y reporter vos réglages.

* **Configuration :** Reportez vos personnalisations (message de bienvenue, nom du serveur, mots de passe) dans le fichier `/etc/mumble-server.ini`.
* **Base de données :** Placez votre ancien fichier SQLite dans `/var/lib/mumble-server/mumble-server.sqlite`.

### 3. Gestion des permissions et sécurité

L'utilisateur `mumble-server` doit être propriétaire de ses fichiers pour fonctionner correctement. Pour une sécurité renforcée, nous restreignons l'accès aux fichiers sensibles.

```bash
# Attribution de la propriété à l'utilisateur système
chown mumble-server:mumble-server /etc/mumble-server.ini
chown -R mumble-server:mumble-server /var/lib/mumble-server/

# Restriction des droits d'accès (Lecture/Écriture pour le service uniquement)
chmod 640 /etc/mumble-server.ini
chmod 640 /var/lib/mumble-server/mumble-server.sqlite

```

> **Note de sécurité :** Ces commandes garantissent que seul l'utilisateur `mumble-server` peut lire la configuration et la base de données, protégeant ainsi vos jetons et mots de passe des autres utilisateurs du système.

### 4. Activation et démarrage automatique

Assurez-vous que le service démarre dès le lancement du container :

```bash
systemctl enable mumble-server
systemctl start mumble-server

```

## Vérification du bon fonctionnement

Utilisez la commande suivante pour confirmer que le serveur est actif :
`systemctl status mumble-server`

Vérifiez également dans les journaux que la base de données SQLite a bien été chargée sans erreur.

![mumble-bbl.png](/migration-mumble-debian-12-13-proxmox-lxc/mumble-bbl.png)
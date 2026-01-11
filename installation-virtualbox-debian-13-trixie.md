---
title: Installation de VirtualBox sur Debian 13 (Trixie)
description: Apprenez à installer VirtualBox 7.2 sur Debian 13 (Trixie) via le dépôt officiel. Guide pas à pas incluant l'importation de la clé GPG, l'ajout du dépôt et l'installation de l'Extension Pack.
published: true
date: 2026-01-11T22:06:34.030Z
tags: serveur, linux, virtualisation, virtualbox, debian13, trixie
editor: markdown
dateCreated: 2026-01-11T22:06:34.030Z
---

Cette page détaille la procédure manuelle pour installer la version officielle d'Oracle sur Debian 13. L'utilisation du dépôt tiers permet de bénéficier de la version 7.2+, souvent plus récente que celle des dépôts standards.

> **Note :** Pour éviter les erreurs de permissions lors de l'utilisation de redirections (pipes), nous effectuons la majeure partie de la procédure en mode super-utilisateur.

### Alternative rapide

Pour une installation automatisée en une seule commande (compatible Debian, Ubuntu et Mint), vous pouvez utiliser mon script disponible ici : [GitHub - anyblabla/virtualbox](https://github.com/anyblabla/virtualbox).

---

## 1. Passage en mode super-utilisateur

Pour garantir le bon fonctionnement des commandes complexes et des écritures de fichiers système, passez en root :

```bash
sudo -i

```

## 2. Installation des prérequis

Avant d'ajouter le dépôt, il est nécessaire d'installer les outils de gestion des clés et les en-têtes du noyau pour la compilation des modules :

```bash
apt update && apt upgrade -y
apt install -y wget gnupg bzip2 build-essential dkms linux-headers-$(uname -r)

```

## 3. Importation de la clé GPG d'Oracle

Cette clé permet au système de valider l'authenticité des paquets téléchargés :

```bash
wget -qO- https://www.virtualbox.org/download/oracle_vbox_2016.asc | gpg --dearmor --yes -o /usr/share/keyrings/oracle-virtualbox-2016.gpg

```

## 4. Ajout du dépôt officiel

Ajoutez la source de paquets Oracle spécifique à Debian 13 (Trixie) dans votre configuration :

```bash
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/oracle-virtualbox-2016.gpg] https://download.virtualbox.org/virtualbox/debian trixie contrib" > /etc/apt/sources.list.d/virtualbox.list

```

## 5. Installation de VirtualBox

Mettez à jour votre cache de paquets et installez la dernière version majeure :

```bash
apt update
apt install virtualbox-7.2

```

## 6. Configuration des groupes utilisateurs

Pour que votre utilisateur puisse gérer les machines virtuelles, accéder aux périphériques USB et aux disques physiques, ajoutez-le aux groupes nécessaires (remplacez `votre_nom` par votre identifiant réel) :

```bash
usermod -aG vboxusers votre_nom
usermod -aG disk votre_nom

```

*Note : Si vous ne connaissez pas votre nom d'utilisateur, tapez `exit` puis `whoami` avant de revenir en root.*

## 7. Installation de l'Extension Pack

L'Extension Pack ajoute le support de l'USB 3.0 et du RDP.

1. Téléchargez le fichier correspondant à votre version sur la page officielle : [VirtualBox Downloads](https://www.virtualbox.org/wiki/Downloads) (Lien : *All supported platforms*).
2. Installez-le avec la commande suivante (toujours en root) :

```bash
VBoxManage extpack install Oracle_VirtualBox_Extension_Pack-7.2.x.vbox-extpack

```

---

## 8. Finalisation

Une fois l'installation terminée, quittez le mode root :

```bash
exit

```

**Important :** Vous devez impérativement fermer votre session utilisateur et vous reconnecter (ou redémarrer l'ordinateur) pour que les droits sur l'USB et les disques soient effectifs.

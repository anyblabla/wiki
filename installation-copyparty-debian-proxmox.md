---
title: Installation de Copyparty sur Debian (LXC Proxmox)
description: Apprenez à installer CopyParty sur un conteneur Debian (Proxmox). Ce guide complet détaille la configuration du service, l'accès via reverse proxy et l'automatisation des mises à jour par cron.
published: true
date: 2026-03-05T11:53:37.077Z
tags: lxc, proxmox, debian, logiciel libre, copyparty, auto-update, serveur de fichiers
editor: markdown
dateCreated: 2026-03-05T11:53:37.077Z
---

### 🌟 Présentation

**[Copyparty](https://github.com/9001/copyparty)** est un serveur de fichiers (File Server) ultra-léger, rapide et polyvalent. Il permet de partager des fichiers, de diffuser du contenu multimédia et de gérer des téléversements (uploads) via une interface web moderne ou via des protocoles comme WebDAV, FTP et TFTP. Il est particulièrement apprécié pour sa faible consommation de ressources et sa facilité de déploiement.

Ce guide est un pas à pas complet pour installer et automatiser **Copyparty** dans un conteneur Debian sur Proxmox VE.

---

### 🛠 1. Dépendances et pré-requis

On commence par mettre à jour le système et installer les paquets nécessaires au fonctionnement du service et de l'indexation.

```bash
apt update && apt upgrade -y
apt install -y python3 python3-pip python3-jinja2 wget locate nano

```

### 🚀 2. Installation de l'exécutable

Nous utilisons la version "SFX" (Single File Executable) qui regroupe tout le nécessaire dans un seul fichier Python.

1. **Création des dossiers (données et logs) :**
```bash
mkdir -p /var/lib/copyparty /var/log/copyparty

```


2. **Téléchargement de l'exécutable :**
```bash
wget https://github.com/9001/copyparty/releases/latest/download/copyparty-sfx.py -O /usr/local/bin/copyparty-sfx.py

```


3. **Attribution des droits d'exécution :**
```bash
chmod +x /usr/local/bin/copyparty-sfx.py

```



### ⚙️ 3. Configuration du service

Il faut maintenant créer le fichier de configuration qui définit le comportement de Copyparty.

1. **Création du fichier avec nano :**
```bash
nano /etc/copyparty.conf

```


2. **Contenu à personnaliser :**
> **⚠️ IMPORTANT :** Modifiez les valeurs suivies d'un commentaire `# <--` pour correspondre à votre installation.



```ini
[global]
  i: 0.0.0.0
  p: 3923
  ansi
  e2dsa
  e2ts
  z, qr
  theme: 2
  grid
  no-robots
  
  # Configuration reverse proxy
  rproxy: 1
  xff-hdr: "X-Forwarded-For"
  xff-src: 192.168.1.50     # <--- REMPLACEZ par l'IP de votre reverse proxy (Nginx/Traefik)

[accounts]
  votre_utilisateur: votre_mot_de_passe   # <--- MODIFIEZ votre login et mot de passe ici

[/]
  /var/lib/copyparty
  accs:
    r: * # Lecture autorisée pour tous (public)
    rwmda: votre_utilisateur # <--- REMPLACEZ par le nom d'utilisateur choisi ci-dessus

```

*(Quittez nano avec `Ctrl+O`, `Entrée`, puis `Ctrl+X`).*

### 🔧 4. Création du service Systemd

Pour que Copyparty se lance automatiquement au démarrage du LXC.

1. **Création du fichier de service :**
```bash
nano /etc/systemd/system/copyparty.service

```


2. **Contenu du service :**

```ini
[Unit]
Description=CopyParty Service
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /usr/local/bin/copyparty-sfx.py -c /etc/copyparty.conf
Restart=always

[Install]
WantedBy=multi-user.target

```

3. **Activation et démarrage :**

```bash
systemctl daemon-reload
systemctl enable --now copyparty

```

---

## 🔄 Mise à jour automatique (script et cron)

### 1. Création du script de mise à jour

On place le script dans le dossier `/root/` pour un accès direct.

1. **Ouvrir nano pour créer le script :**
```bash
nano /root/update_copyparty.sh

```


2. **Contenu du script :**

```bash
#!/bin/bash
# Auteur : Amaury aka BlablaLinux (https://link.blablalinux.be)
# Description : Mise à jour automatique de Copyparty SFX

URL="https://github.com/9001/copyparty/releases/latest/download/copyparty-sfx.py"
DEST="/usr/local/bin/copyparty-sfx.py"
SERVICE_NAME="copyparty"
LOG="/var/log/copyparty/update.log"

# Téléchargement de la nouvelle version
wget -q $URL -O "$DEST.new"

# Comparaison avec l'ancienne version
if ! cmp -s "$DEST" "$DEST.new"; then
    mv "$DEST.new" "$DEST"
    chmod +x "$DEST"
    systemctl restart $SERVICE_NAME
    echo "$(date): Copyparty a été mis à jour et redémarré." >> $LOG
else
    rm "$DEST.new"
    echo "$(date): Copyparty est déjà à jour." >> $LOG
fi

```

3. **Rendre le script exécutable :**

```bash
chmod +x /root/update_copyparty.sh

```

### 2. Planification avec Cron

Pour vérifier les mises à jour tous les dimanches à 4h00 du matin.

1. **Ouvrir le crontab :**

```bash
crontab -e

```

2. **Ajouter la ligne à la fin :**

```cron
0 4 * * 0 /bin/bash /root/update_copyparty.sh > /dev/null 2>&1

```

---

### ✅ Commandes de vérification

* **Vérifier la version :** `python3 /usr/local/bin/copyparty-sfx.py --version`
* **Logs de mise à jour :** `cat /var/log/copyparty/update.log`
* **Statut du service :** `systemctl status copyparty`

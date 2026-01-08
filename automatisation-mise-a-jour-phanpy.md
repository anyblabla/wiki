---
title: Automatisation de la mise à jour de Phanpy
description: Découvrez comment automatiser la mise à jour de Phanpy sur un serveur Debian. Ce guide propose un script Bash pour vérifier les releases GitHub, gérer l'installation et configurer une tâche Cron.
published: true
date: 2026-01-08T22:58:27.325Z
tags: mastodon, lxc, debian, script, automatisation, phanpy
editor: markdown
dateCreated: 2026-01-08T22:58:27.325Z
---

Cette page documente la mise en place d'un script Bash permettant de maintenir à jour l'instance **Phanpy** (client web pour Mastodon) en surveillant les sorties officielles sur GitHub.

## Liens utiles

* **Dépôt officiel :** [cheeaun/phanpy](https://github.com/cheeaun/phanpy){:target="_blank"}
* **Releases GitHub :** [Phanpy Releases](https://github.com/cheeaun/phanpy/releases){:target="_blank"}
* **Instance de démonstration :** [Phanpy par Blabla Linux](https://phanpy.blablalinux.be){:target="_blank"}

---

## Prérequis

Le script utilise `curl` pour interroger l'API GitHub. Sur une installation minimale (template LXC Debian), cette dépendance peut être absente.

1. **Mettre à jour les dépôts et installer curl :**
```bash
apt update && apt install curl -y

```



> **Note :** Bien qu'il n'existe pas d'image Docker officielle pour Phanpy (le projet étant composé de fichiers statiques), vous pouvez retrouver mes guides sur Docker pour d'autres services ici : [Installation Docker, Portainer et LXC sur Debian et Proxmox](https://wiki.blablalinux.be/fr/docker-portainer-lxc-debian-proxmox).

---

## Préparation de l'environnement

Il est nécessaire de créer un dossier pour vos scripts et d'initialiser le fichier de version pour que le script puisse comparer l'installation locale avec GitHub.

1. **Créer le dossier des scripts :**
```bash
mkdir -p /root/scripts

```


2. **Initialiser le fichier de version :**
Identifiez votre version actuelle et enregistrez-la dans le fichier témoin pour permettre la première comparaison :
```bash
echo "2025.11.26.ac85274" > /var/www/html/.phanpy_version

```



---

## Création du script d'automatisation

1. **Créer le fichier du script :**
```bash
nano /root/scripts/update_phanpy.sh

```


2. **Coller le contenu suivant :**

```bash
#!/bin/bash
# Script de mise à jour automatique pour Phanpy
# Auteur : Amaury aka BlablaLinux

REPO="cheeaun/phanpy"
INSTALL_DIR="/var/www/html"
VERSION_FILE="$INSTALL_DIR/.phanpy_version"
FILENAME="phanpy-dist.tar.gz"

echo "--- Début de la vérification Phanpy ---"

# Récupération de la dernière version stable via l'API GitHub
LATEST_RELEASE=$(curl -s https://api.github.com/repos/$REPO/releases/latest | grep '"tag_name":' | sed -E 's/.*"([^"]+)".*/\1/')

if [ -z "$LATEST_RELEASE" ]; then
    echo "Erreur : Impossible de joindre l'API GitHub."
    exit 1
fi

# Récupération de la version locale
CURRENT_VERSION=$([ -f "$VERSION_FILE" ] && cat "$VERSION_FILE" || echo "aucune")

if [ "$LATEST_RELEASE" == "$CURRENT_VERSION" ]; then
    echo "Phanpy est déjà à jour ($CURRENT_VERSION)."
    exit 0
fi

echo "Nouvelle version détectée : $LATEST_RELEASE. Mise à jour en cours..."

TEMP_DIR=$(mktemp -d)
URL="https://github.com/$REPO/releases/download/$LATEST_RELEASE/$FILENAME"

if curl -L -s -o "$TEMP_DIR/$FILENAME" "$URL"; then
    # Nettoyage préventif des anciens fichiers pour éviter les conflits d'assets
    # On supprime tout dans /var/www/html sauf le fichier de version
    find "$INSTALL_DIR" -mindepth 1 -not -name '.phanpy_version' -delete
    
    # Extraction de la nouvelle version
    tar -xzf "$TEMP_DIR/$FILENAME" -C "$INSTALL_DIR"
    
    # Mise à jour du tag de version locale
    echo "$LATEST_RELEASE" > "$VERSION_FILE"
    echo "Mise à jour vers $LATEST_RELEASE terminée avec succès."
else
    echo "Erreur lors du téléchargement de l'archive."
    exit 1
fi

rm -rf "$TEMP_DIR"

```

3. **Rendre le script exécutable :**
```bash
chmod +x /root/scripts/update_phanpy.sh

```



---

## Validation manuelle et vérification

Lancez le script une première fois pour vérifier que le déploiement se déroule correctement :

```bash
/root/scripts/update_phanpy.sh

```

Une fois le script terminé, vérifiez que le fichier de version a bien été mis à jour avec le dernier tag GitHub :

```bash
cat /var/www/html/.phanpy_version

```

---

## Configuration de la tâche Cron

Pour automatiser la vérification chaque nuit à 04h15 et générer un log de suivi :

1. **Ouvrir la configuration cron de root :**
```bash
crontab -e

```


2. **Ajouter la ligne suivante à la fin du fichier :**
```cron
15 4 * * * /bin/bash /root/scripts/update_phanpy.sh > /var/log/phanpy_update.log 2>&1

```
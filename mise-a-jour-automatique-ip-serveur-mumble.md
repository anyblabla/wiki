---
title: Mise à jour automatique de l'IP publique pour un serveur Mumble
description: Tutoriel pour automatiser la mise à jour de la directive registerHostname d'un serveur Mumble sur LXC Debian en cas d'IP publique dynamique, avec redémarrage ciblé et gestion propre des logs.
published: true
date: 2026-05-24T19:10:30.833Z
tags: lxc, cron, debian, bash, mumble, auto-hebergement
editor: markdown
dateCreated: 2026-05-24T19:10:30.833Z
---

Ce guide explique comment automatiser la mise à jour de la directive `registerHostname` dans la configuration d'un serveur Mumble installé sur un conteneur LXC (Debian/Ubuntu), lorsque l'on dispose d'une adresse IP publique dynamique.

Le script vérifie l'IP toutes les heures et ne redémarre le serveur Mumble **que** si un changement est détecté, évitant ainsi les coupures inutiles. De plus, il reste totalement silencieux pour ne pas polluer les journaux système.

## Prérequis

Le script utilise l'outil `curl` pour récupérer l'IP publique. S'il n'est pas présent sur le conteneur, il faut l'installer :

```bash
apt update && apt install curl -y

```

## Création du script de vérification

Je centralise le script dans un dossier dédié aux scripts système :

```bash
mkdir -p /root/scripts
nano /root/scripts/update_mumble_ip.sh

```

Voici le code complet à coller à l'intérieur :

```bash
#!/bin/bash

# --- CONFIGURATION ---
INI_FILE="/etc/mumble/mumble-server.ini"
RESTART_CMD="systemctl restart mumble-server"
# ---------------------

CURRENT_IP=$(curl -s https://api.ipify.org)

if [ -z "$CURRENT_IP" ]; then
    echo "$(date '+%Y-%m-%d %H:%M:%S') - Erreur : Impossible de récupérer l'IP publique."
    exit 1
fi

CONFIGURED_IP=$(grep -E '^[[:space:]]*registerHostname\s*=' "$INI_FILE" | grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}')

if [ "$CURRENT_IP" != "$CONFIGURED_IP" ]; then
    echo "$(date '+%Y-%m-%d %H:%M:%S') - L'IP a changé ! Ancienne: $CONFIGURED_IP | Nouvelle: $CURRENT_IP"
    echo "$(date '+%Y-%m-%d %H:%M:%S') - Mise à jour de registerHostname..."
    
    sed -i -E "s/^[[:space:]]*(registerHostname\s*=.*)/registerHostname=$CURRENT_IP/" "$INI_FILE"
    
    echo "$(date '+%Y-%m-%d %H:%M:%S') - Redémarrage du serveur Mumble..."
    $RESTART_CMD
    echo "$(date '+%Y-%m-%d %H:%M:%S') - Mise à jour terminée avec succès."
fi

```

Ensuite, je rends le script exécutable :

```bash
chmod +x /root/scripts/update_mumble_ip.sh

```

## Automatisation avec cron

Pour lancer la vérification de manière transparente toutes les heures à la minute 0, j'ouvre le planificateur de tâches de l'utilisateur root :

```bash
crontab -e

```

Je remplace le contenu ou j'ajoute ces lignes à la fin du fichier :

```text
# Verification et mise a jour de l'IP publique du serveur Mumble toutes les heures (avec logs)
0 * * * * /root/scripts/update_mumble_ip.sh >> /var/log/mumble_ip_update.log 2>&1

```

## Suivi et logs

Le script est conçu pour rester muet si l'adresse IP n'a pas changé, évitant ainsi de remplir un fichier de log pour rien.

Si un changement d'IP survient, les actions et le redémarrage de Mumble seront consignés avec horodatage. Pour consulter l'historique, il me suffit de lire le fichier de log :

```bash
cat /var/log/mumble_ip_update.log

```

---

> **Auteur** : ce guide est proposé par **Amaury aka BlablaLinux**. Retrouvez l'ensemble de mes services sur [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).

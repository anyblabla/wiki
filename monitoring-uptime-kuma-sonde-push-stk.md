---
title: Monitoring - Utilisation des sondes push (exemple avec SuperTuxKart)
description: Apprenez à surveiller un service UDP ou un serveur derrière un NAT grâce à la sonde Push d'Uptime Kuma. Guide pratique basé sur l'exemple concret d'un serveur de jeu SuperTuxKart.
published: true
date: 2026-01-20T23:18:23.989Z
tags: monitoring, debian, bash, linux, logiciel libre, uptime kuma, stk, supertuxkart
editor: markdown
dateCreated: 2026-01-20T23:18:23.989Z
---

## Présentation

Contrairement aux sondes classiques où **Uptime Kuma** interroge un service (méthode dite "Pull"), la sonde **push** attend que le service lui envoie lui-même un signal de vie. C’est la solution idéale pour :

* Les serveurs derrière un NAT agressif (sans ouverture de port entrant).
* Les services utilisant le protocole **UDP** (comme les serveurs de jeux).
* Le suivi de tâches planifiées (vérifier qu'une sauvegarde s'est bien terminée).

## Concept de fonctionnement

Le serveur surveillé exécute un script localement. Si les conditions de santé sont remplies, il effectue une requête HTTP (via `curl`) vers l'instance Uptime Kuma. Si cette dernière ne reçoit rien pendant une période définie, elle considère que le service est hors ligne.

## Cas pratique : serveur SuperTuxKart

SuperTuxKart utilisant le protocole UDP, une sonde classique peine à vérifier si le service répond réellement. Nous allons donc utiliser une sonde push.

### 1. Configuration dans Uptime Kuma

1. Créez une nouvelle sonde de type **Push**.
2. **Nom de la sonde :** `Serveur STK`.
3. **Intervalle de battement :** `60 secondes`.
4. Copiez l'**URL push** générée (type : `https://kuma.blablalinux.be/api/push/ID_UNIQUE?status=up&msg=OK`).

### 2. Mise en place du script sur le serveur

Sur votre système (Debian, Ubuntu ou Linux Mint), créez un script pour valider que le serveur de jeu écoute bien sur son port (par défaut 2759).

**Fichier :** `/usr/local/bin/check_stk.sh`

```bash
#!/bin/bash

# Configuration
PORT=2759
URL_KUMA="VOTRE_URL_PUSH_ICI"

# Vérification du port UDP (nécessite le paquet iproute2)
if ss -unlp | grep -q ":$PORT"; then
    curl -s "$URL_KUMA" > /dev/null
fi

```

### 3. Automatisation de la vérification

Pour que la sonde soit mise à jour chaque minute, on utilise une tâche `cron`.

```bash
crontab -e
# Ajoutez la ligne suivante en bas du fichier :
* * * * * /bin/bash /usr/local/bin/check_stk.sh

```

![uk-stk.png](/monitoring-uptime-kuma-sonde-push-stk/uk-stk.png)

## Pourquoi utiliser cette méthode ?

* **Fiabilité :** On ne vérifie pas seulement si la machine répond au ping, mais si le processus du jeu est réellement actif.
* **Sécurité :** Aucun port supplémentaire n'est à ouvrir sur votre pare-feu pour le monitoring, seul le flux sortant vers Uptime Kuma est nécessaire.
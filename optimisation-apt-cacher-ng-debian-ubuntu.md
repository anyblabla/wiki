---
title: Optimisation d'APT-Cacher NG pour Debian et Ubuntu
description: Guide technique pour stabiliser APT-Cacher NG sur Debian Trixie et Ubuntu 24.04. Résolution des erreurs de signature et de requêtes HTTP pour une gestion fluide d'un parc de machines virtuelles.
published: true
date: 2025-12-25T19:25:27.734Z
tags: cache, debian, ubuntu, apt, linux, administration système
editor: markdown
dateCreated: 2025-12-25T19:25:27.734Z
---

## Présentation

Dans un environnement informatique comportant de nombreux invités (machines virtuelles ou conteneurs), **APT-Cacher NG** est l'outil idéal pour économiser la bande passante et accélérer les mises à jour. Cependant, sur des versions récentes comme **Debian Trixie** (Testing) ou **Ubuntu 24.04** (Noble), des erreurs de signatures ou de requêtes HTTP peuvent apparaître.

Ce wiki détaille comment configurer le serveur pour qu'il soit autonome et stable, sans nécessiter d'interventions manuelles régulières.

## 1. Configuration du serveur (`acng.conf`)

Le secret d'un cache stable réside dans la gestion des fichiers de signatures (`InRelease`, `Release`). En évitant de stocker ces fichiers qui changent très fréquemment, on élimine les risques de corruption de données.

Modifiez le fichier `/etc/apt-cacher-ng/acng.conf` avec ces paramètres optimisés :

```conf
# /etc/apt-cacher-ng/acng.conf - Configuration optimisée (Blabla Linux)

CacheDir: /var/cache/apt-cacher-ng
LogDir: /var/log/apt-cacher-ng
SupportDir: /usr/lib/apt-cacher-ng

# --- Optimisation et stabilité ---

# 1. Éviter les erreurs de signature (Message manipulated)
# On ne met pas en cache les fichiers d'index et de signatures. 
# Les clients les téléchargent en direct (quelques ko), garantissant une validité totale.
DontCache: .*/dists/.*(Release|InRelease|Packages).*

# 2. Fraîcheur des données
# On force le proxy à vérifier la source toutes les 30 secondes.
FreshIndexMaxAge: 30

# 3. Fiabilité des téléchargements
# On désactive les Range-Requests sur les fichiers volatiles pour éviter les hash incohérents.
VfileUseRangeOps: 0

# --- Maintenance automatique ---
# Le serveur nettoie lui-même les vieux paquets (.deb) après 4 jours.
ExThreshold: 4
FollowIndexFileRemoval: 1

```

> **Note technique :** Avec cette configuration, le serveur est autonome. Il n'est plus nécessaire de prévoir des tâches de nettoyage manuel car les fichiers problématiques ne sont plus stockés physiquement sur le disque.

## 2. Cas particulier d'Ubuntu 24.04

Si vous rencontrez une erreur `400 Bad Request` sur Ubuntu Noble, cela est souvent dû au dépôt `noble-proposed` qui est instable et parfois mal synchronisé sur les miroirs régionaux.

### Résolution

Éditez le fichier de sources sur la machine cliente :
`sudo nano /etc/apt/sources.list.d/ubuntu.sources`

Retirez le terme **`noble-proposed`** de la ligne `Suites`. Une fois la modification enregistrée, lancez :

```bash
sudo apt clean && sudo apt update

```

## 3. Maintenance recommandée

Si vous souhaitez tout de même forcer un nettoyage complet du cache de temps en temps, utilisez l'outil intégré. Cela permet de respecter la cohérence de la base de données interne d'APT-Cacher NG :

```bash
# Commande de maintenance officielle
/usr/lib/apt-cacher-ng/acngtool expire -c /etc/apt-cacher-ng

```
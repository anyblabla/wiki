---
title: Optimisation des logs et de Fail2Ban pour Nginx Proxy Manager
description: Apprenez à optimiser la rotation des logs et l'analyse Fail2Ban pour Nginx Proxy Manager. Un guide pratique pour gérer efficacement plus de 90 proxy hosts sans saturer votre système.
published: true
date: 2026-02-13T12:29:30.806Z
tags: docker, npm, fail2ban, sécurité, linux, sysadmin, nginx proxy manager, logrotate
editor: markdown
dateCreated: 2026-02-13T12:15:21.018Z
---

## Introduction

Lorsque l'on gère une infrastructure auto-hébergée avec de nombreux services (plus de 90 proxy hosts dans mon cas), les journaux d'accès (logs) peuvent rapidement devenir un enfer. Sans optimisation, les fichiers pèsent plusieurs dizaines de Mo, ralentissent les analyses de sécurité et saturent l'espace disque de votre conteneur ou VM.

**Ce guide vous apprend à :**

1. Automatiser une rotation quotidienne des logs.
2. Vérifier la prise en compte de vos alias personnalisés.
3. Utiliser une boîte à outils d'analyse rapide via des alias Bash.
4. Comprendre les codes erreurs pour affiner votre stratégie de bannissement.

## 1. Automatisation de la rotation (Logrotate)

Pour garder un système réactif, nous passons d'une rotation hebdomadaire à une rotation **quotidienne**. Cela permet à Fail2Ban de travailler sur des fichiers légers et récents.

### Configuration du fichier

Créez un fichier nommé `logrotate.custom` dans le dossier de votre projet **Nginx Proxy Manager** (NPM).

> **⚠️ Attention :** Le service Logrotate est très strict sur la syntaxe. Ne placez **aucun commentaire** (`#`) sur la même ligne que les instructions numériques (comme `rotate 14`), sinon le service plantera avec une erreur de type "bad rotation count".

```bash
# Configuration pour /etc/logrotate.d/nginx-proxy-manager

/data/logs/*_access.log /data/logs/*/access.log {
    su npm npm
    create 0644
    daily
    rotate 14
    missingok
    notifempty
    compress
    delaycompress
    sharedscripts
    postrotate
        kill -USR1 `cat /run/nginx/nginx.pid 2>/dev/null` 2>/dev/null || true
    endscript
}

/data/logs/*_error.log /data/logs/*/error.log {
    su npm npm
    create 0644
    daily
    rotate 20
    missingok
    notifempty
    compress
    sharedscripts
    postrotate
        kill -USR1 `cat /run/nginx/nginx.pid 2>/dev/null` 2>/dev/null || true
    endscript
}

```

### Intégration dans Docker

Ajoutez ce fichier comme volume dans votre fichier `docker-compose.yml` pour qu'il soit utilisé par le conteneur NPM :

```yaml
services:
  npm:
    image: jc21/nginx-proxy-manager:latest
    # ... reste de la config
    volumes:
      - ./data:/data
      - ./logrotate.custom:/etc/logrotate.d/nginx-proxy-manager

```

## 2. Activation des alias dans le système

Avant d'ajouter vos commandes personnalisées, vous devez vous assurer que votre shell (le terminal) les chargera correctement à chaque connexion.

1. Ouvrez votre fichier de configuration principal : `nano ~/.bashrc`
2. Vérifiez que le bloc suivant est présent et **n'est pas commenté** (pas de `#` devant ces lignes) :

```bash
if [ -f ~/.bash_aliases ]; then
    . ~/.bash_aliases
fi

```

3. Sauvegardez et forcez le rechargement : `source ~/.bashrc`

## 3. Boîte à outils du technicien (Alias Bash)

Les alias sont des raccourcis clavier. Copiez ces lignes dans votre fichier `~/.bash_aliases`.

### Gestion de Fail2Ban

* **`f2bs`** : État global des jails (services protégés).
* **`f2bnpm`** : Affiche les IP actuellement bannies par la jail NPM.
* **`f2b-ban <IP>`** / **`f2b-unban <IP>`** : Pour bannir ou gracier une IP manuellement.
* **`block-ip <IP>`** : Bloque l'IP directement via `iptables`. C'est plus radical car le trafic est coupé avant même d'arriver à Nginx.

### Analyse et statistiques

* **`f2b-analyze`** : Parcourt tous vos fichiers de logs (même si vous en avez 95) et génère un rapport complet dans `/root/f2b/analyse_fail2ban.txt`.
* **`f2b-stat-ip`** : Extrait du rapport les 10 adresses IP les plus actives (les attaquants potentiels).
* **`f2b-stat-dom`** : Identifie vos noms de domaines les plus visés.
* **`f2b-stat-code`** : Affiche le décompte des codes erreurs (403, 502, etc.).

```bash
# --- Gestion simplifiée ---
alias f2bs='docker exec -t fail2ban fail2ban-client status'
alias f2bnpm='docker exec -t fail2ban fail2ban-client status npm'
alias f2bbanned='docker exec -t fail2ban fail2ban-client banned'
alias f2breload='docker exec -t fail2ban fail2ban-client reload'

# --- Bannissement manuel ---
f2b-ban() { 
    docker exec -t fail2ban fail2ban-client set npm banip "$1"
}
f2b-unban() { 
    docker exec -t fail2ban fail2ban-client set npm unbanip "$1"
}

# --- Analyse profonde ---
f2b-analyze() {
    echo "Analyse des logs en cours... Patientez."
    echo "--- ANALYSE FAIL2BAN DU $(date) ---" > /root/f2b/analyse_fail2ban.txt
    for logfile in $(docker exec fail2ban sh -c "ls /var/log/proxy-host-*_access.log"); do
        echo "Traitement de $logfile..."
        echo -e "\n--- FICHIER: $logfile ---" >> /root/f2b/analyse_fail2ban.txt
        docker exec fail2ban fail2ban-regex "$logfile" /etc/fail2ban/filter.d/npm.conf --print-all-matched >> /root/f2b/analyse_fail2ban.txt
    done
    echo "Analyse terminée : /root/f2b/analyse_fail2ban.txt"
}

# --- Statistiques ---
alias f2b-stat-ip="grep -oP 'Client \K[0-9.]+' /root/f2b/analyse_fail2ban.txt | sort | uniq -c | sort -rn | head -n 10"
alias f2b-stat-dom="grep -oP 'https \K[^ ]+' /root/f2b/analyse_fail2ban.txt | sort | uniq -c | sort -rn | head -n 10"
alias f2b-stat-code="grep -oP ' - - \K[0-9]{3}' /root/f2b/analyse_fail2ban.txt | sort | uniq -c"

# --- Sécurité réseau ---
alias block-ip='iptables -I DOCKER-USER -j DROP -s'

```

## 4. Stratégie de sécurité : comprendre les codes d'état

Pour affiner vos filtres, voici comment interpréter les retours de NPM :

* **401 (Unauthorized) :** Échec d'authentification. Quelqu'un tente de forcer un accès. **Action : Bannissement prioritaire.**
* **403 (Forbidden) :** Accès à une ressource interdite (scans de vulnérabilités). **Action : Bannissement recommandé.**
* **502 (Bad Gateway) :** Le service de destination est injoignable. **Attention :** Cela arrive souvent lors de vos maintenances Docker. Utilisez un seuil élevé (ex: 15 tentatives) pour éviter les bannissements accidentels.
---
title: Script de maintenance Proxmox : mise à jour de Watchtower
description: Ce script Bash est conçu pour les administrateurs utilisant Proxmox Virtual Environment (VE) pour héberger des conteneurs LXC exécutant Docker.
published: true
date: 2025-10-27T22:49:47.343Z
tags: docker, lxc, script, bash, watchtower
editor: markdown
dateCreated: 2025-10-26T16:28:46.835Z
---

# 🚀 Script de Maintenance Proxmox : Mise à Jour de Watchtower

Ce guide fournit des scripts Bash robustes pour les utilisateurs de Proxmox VE. Ils automatisent la modification des fichiers `docker-compose.yml` afin d'ajuster la planification de l'outil de mise à jour automatique des images Docker, **Watchtower**.

## 1\. Informations Générales

### 1.1. 🎯 À Quoi Sert Ce Script ?

| Champ | Description |
| :--- | :--- |
| **Objectif Principal** | Remplacer l'intervalle de mise à jour de Watchtower (p. ex., toutes les 3 heures) par une **planification CRON spécifique** (p. ex., tous les mercredis à 10h00). |
| **Pour Qui ?** | Les utilisateurs de Proxmox qui gèrent des LXC avec des services Docker (`docker-compose`) et qui veulent **éviter les mises à jour aléatoires** qui pourraient perturber les services en production. |
| **Avantages Techniques** | Il gère les subtilités de l'environnement Proxmox, filtre uniquement les conteneurs actifs, et **recherche dynamiquement le fichier de configuration** même s'il est dans un sous-répertoire (`/root/mon_service/watchtower/`). |

### 1.2. 🔎 Mécanisme de Recherche Corrigé (Profondeur Limitée)

Pour garantir que le script trouve le fichier `docker-compose.yml` de Watchtower, qu'il soit dans `/root/watchtower/` ou dans un sous-répertoire comme `/root/mon_app/watchtower/`, il utilise la commande `find` avec une profondeur de recherche limitée :

| Élément | Description |
| :--- | :--- |
| **Point de départ** | La recherche s'effectue toujours à partir du répertoire de l'utilisateur root (`/root/`) à l'intérieur du conteneur LXC. |
| **Profondeur de recherche** | La recherche est limitée à **4 niveaux de sous-répertoires** (paramètre `-maxdepth 4`). Cela permet de trouver des chemins comme `/root/dossier1/dossier2/dossier3/docker-compose.yml`, tout en évitant de parcourir l'intégralité du système de fichiers, ce qui pourrait provoquer des lenteurs ou des blocages. |
| **Ciblage** | La commande **ne recherche que** les fichiers nommés `docker-compose.yml` dont le chemin se termine par `watchtower/docker-compose.yml$`. |

-----

## 2\. 💻 Script 1 : Conversion d'Intervalle vers CRON (Première Utilisation)

Ce script est destiné à être exécuté une seule fois pour remplacer l'ancienne variable d'intervalle par la nouvelle variable de planification CRON.

### 2.1. Installation et Préparation

1.  Connectez-vous à votre nœud Proxmox en SSH.
2.  Créez le fichier du script (par exemple, `script1_watchtower.sh`) :
    ```bash
    nano /root/script1_watchtower.sh
    ```
3.  Collez le code du **Script 1 Corrigé** ci-dessous.
4.  Rendez le script exécutable :
    ```bash
    chmod +x /root/script1_watchtower.sh
    ```

### 2.2. Le Script 1 (Version Corrigée)

```bash
#!/bin/bash

# Configuration
# ==============
# Remplacement: WATCHTOWER_POLL_INTERVAL=10800 sera remplacé par WATCHTOWER_SCHEDULE=0 0 10 ? * WED (Mercredi 10h00)
MODIFICATION_SED='s/- WATCHTOWER_POLL_INTERVAL=10800/- WATCHTOWER_SCHEDULE=0 0 10 ? * WED/'
TARGET_DESCRIPTION="watchtower/docker-compose.yml sous /root"


# Fonction pour redémarrer Docker Compose (MODIFIÉE pour accepter le répertoire)
restart_docker_compose() {
    local CTID=$1
    local COMPOSE_DIR=$2 
    local TIMEOUT_DURATION=30

    echo "   Tentative de redémarrage avec 'docker compose' dans $COMPOSE_DIR (Timeout: ${TIMEOUT_DURATION}s)..."

    local LOG_FILE="/tmp/pct_exec_log_$CTID.log"
    timeout "$TIMEOUT_DURATION" pct exec "$CTID" -- sh -c "cd $COMPOSE_DIR && docker compose down && docker compose up -d" > "$LOG_FILE" 2>&1
    local exit_code=$?
    
    if [ $exit_code -eq 0 ]; then
        grep -E 'Removed|Created|Started' "$LOG_FILE"
        echo "   ✅ Redémarrage réussi avec 'docker compose'."
        rm -f "$LOG_FILE"
        return 0
    elif [ $exit_code -eq 124 ]; then
        echo "   ⚠️ Redémarrage forcé par timeout. Service probablement relancé. Continuer."
        rm -f "$LOG_FILE"
        return 0
    else
        echo "   ❌ Échec du redémarrage (Code: $exit_code). Consultez le fichier $LOG_FILE pour les détails."
        return 1
    fi
}


# 1. Identification des conteneurs LXC à traiter
echo "Recherche des conteneurs LXC ACTIFS dont le nom contient 'docker' (via pct)..."

CONTAINER_IDS=$(pct list | awk '/running/ && /docker/ {print $1}')


# Traitement des Conteneurs (LOGIQUE DE RECHERCHE CORRIGÉE)
# =========================
if [ -z "$CONTAINER_IDS" ]; then
    echo "⚠️ Aucun conteneur LXC/CT en cours d'exécution avec 'docker' dans son nom n'a été trouvé. Aucune action n'a été effectuée."
    exit 0
fi

echo "Conteneurs trouvés (ID): $CONTAINER_IDS"
echo "-----------------------------------"

for CTID in $CONTAINER_IDS; do
    CONTAINER_NAME=$(pct config "$CTID" | grep 'hostname' | awk '{print $2}')
    echo "▶️ Traitement du conteneur LXC (ID $CTID) : $CONTAINER_NAME"
    
    # NOUVELLE LOGIQUE DE RECHERCHE: find avec une profondeur limitée (-maxdepth 4)
    TARGET_CONTAINER_PATH=$(pct exec "$CTID" -- sh -c "find /root -maxdepth 4 -name docker-compose.yml 2>/dev/null | grep 'watchtower/docker-compose.yml$' | head -n 1" 2>/dev/null)

    # Vérification du chemin trouvé
    if [ -z "$TARGET_CONTAINER_PATH" ]; then
        echo "   Fichier de configuration Watchtower non trouvé sous /root. Ce CT/LXC est ignoré."
        echo "-----------------------------------"
        continue
    fi
    
    # 2. Le chemin exact est maintenant stocké dans $TARGET_CONTAINER_PATH
    if [ -n "$TARGET_CONTAINER_PATH" ]; then
        echo "   Fichier de configuration Watchtower trouvé à: $TARGET_CONTAINER_PATH"

        # 3. Modification du fichier (pct pull/push)
        TEMP_FILE="/tmp/docker-compose-temp-$CTID.yml"
        
        pct pull "$CTID" "$TARGET_CONTAINER_PATH" "$TEMP_FILE" >/dev/null 2>&1
        if [ $? -ne 0 ]; then
            echo "❌ ERREUR: Impossible de copier le fichier depuis le conteneur $CTID. Ignoré."
            rm -f "$TEMP_FILE" 2>/dev/null
            continue
        fi

        TEMP_FILE_BAK="${TEMP_FILE}.bak"
        cp "$TEMP_FILE" "$TEMP_FILE_BAK"
        
        # Exécute la substitution SED
        sed -i "$MODIFICATION_SED" "$TEMP_FILE"
        echo "   ✅ Modification appliquée au fichier temporaire."
        
        pct exec "$CTID" -- cp "$TARGET_CONTAINER_PATH" "$TARGET_CONTAINER_PATH.bak" >/dev/null 2>&1
        pct push "$CTID" "$TEMP_FILE" "$TARGET_CONTAINER_PATH" >/dev/null 2>&1
        echo "   ✅ Fichier mis à jour dans le conteneur. Sauvegarde interne: ${TARGET_CONTAINER_PATH}.bak"
        
        rm -f "$TEMP_FILE" "$TEMP_FILE_BAK"
        
        # 4. Redémarrage du service Docker Compose
        COMPOSE_DIR=$(dirname "$TARGET_CONTAINER_PATH")
        
        restart_docker_compose "$CTID" "$COMPOSE_DIR" 
        
        if [ $? -eq 0 ]; then
             echo "✅ Opération complète terminée pour $CONTAINER_NAME (ID $CTID)."
        else
             echo "⚠️ Modification réussie, mais redémarrage échoué pour $CONTAINER_NAME (ID $CTID). **Veuillez redémarrer le service manuellement.**"
        fi
    else
        echo "   Fichier de configuration Watchtower non trouvé à $TARGET_DESCRIPTION. Ce CT/LXC est ignoré."
    fi
    
    echo "-----------------------------------"
done

echo "Toutes les opérations sont terminées."
```

### 2.3. Exécuter le Script

Pour lancer le **Script 1** :

```bash
/root/script1_watchtower.sh
```

-----

## 3\. 🔧 Adaptation du Script pour d'Autres Usages

Le script est conçu pour être facilement adaptable à toute autre modification de fichier dans vos conteneurs LXC.

### 3.1. Adapter la Cible (Où et Qui)

Pour modifier le comportement de ciblage et de recherche :

| Variable | Usage | Comment l'Adapter |
| :--- | :--- | :--- |
| **`CONTAINER_IDS`** | **Ciblage :** Liste les CT à traiter. | Modifiez l'`awk` pour cibler des noms différents (p. ex., `/nginx/` au lieu de `/docker/`) ou retirez `/docker/` pour cibler tous les LXC actifs. |
| **`TARGET_CONTAINER_PATH_TO_TEST`** | **Chemin du Fichier :** Chemin exact du fichier *dans le conteneur*. | Modifiez-le pour pointer vers tout autre fichier de configuration (p. ex., `/etc/nginx/nginx.conf`). |
| **`COMPOSE_DIR`** | **Répertoire de Travail :** Utilisé pour le `cd` avant le redémarrage. | Si vous modifiez un fichier qui nécessite un redémarrage avec `docker compose`, mettez ici le répertoire contenant le `docker-compose.yml`. |

### 3.2. Adapter la Modification (Quoi)

La variable **`MODIFICATION_SED`** contient l'expression de substitution qui effectue la modification. Elle est au format `sed`: `'s/CHAÎNE_RECHERCHÉE/CHAÎNE_DE_REMPLACEMENT/'`.

| Élément | Description | Exemple d'Adaptation |
| :--- | :--- | :--- |
| **Recherche** (`CHAÎNE_RECHERCHÉE`) | La ligne exacte ou le motif à trouver. | Pour changer un port `8080:8080`, utilisez : `8080:8080`. |
| **Remplacement** (`CHAÎNE_DE_REMPLACEMENT`) | La nouvelle valeur à insérer. | Pour changer le port en `8081:8080`, utilisez : `8081:8080`. |

**Exemple d'adaptation :** Si vous vouliez remplacer l'adresse d'un serveur dans un fichier de configuration :

```bash
# Remplacer 'http://old-server:8080' par 'http://new-server:9000'
MODIFICATION_SED='s/http:\/\/old-server:8080/http:\/\/new-server:9000/' 
```

### 3.3. Adapter l'Action Post-Modification

L'étape de redémarrage est gérée par la fonction **`restart_docker_compose`**.

  * **Si vous modifiez un service Docker :** Laissez la fonction telle quelle, car elle gère le redémarrage de `docker compose`.
  * **Si vous modifiez un service système (ex: Nginx) :** Modifiez le corps de la fonction pour utiliser `pct exec "$CTID" -- systemctl restart nginx` au lieu de `docker compose down/up`. Pensez à ajuster le `TIMEOUT_DURATION` si nécessaire.

-----

## 4\. 🛑 Annexe : Substitution Robuste (Script 2)

Le **Script 1** peut échouer si vous tentez de **modifier à nouveau** la planification CRON, car il recherche une chaîne de caractères très spécifique.

Pour garantir que vos changements de planification (ex: passer de Mercredi à Vendredi) fonctionnent **à chaque fois**, utilisez le **Script 2** ci-dessous.

### ⚠️ Avertissement

Utilisez le **Script 2** uniquement si vous avez déjà exécuté le Script 1 et que la variable `WATCHTOWER_SCHEDULE` est déjà présente dans vos fichiers `docker-compose.yml`.

### 4.1. 🚀 Script 2 : Modification d'une planification existante (Corrigé)

Ce script utilise la substitution robuste et intègre la nouvelle logique de recherche limitée.

```bash
#!/bin/bash

# Configuration
# ==============
# La substitution robuste: recherche la ligne commençant par WATCHTOWER_SCHEDULE= et remplace TOUTE la valeur.
MODIFICATION_SED='s/^- WATCHTOWER_SCHEDULE=.*/- WATCHTOWER_SCHEDULE=0 0 10 ? * FRI/'
TARGET_DESCRIPTION="watchtower/docker-compose.yml sous /root" 


# Fonction pour redémarrer Docker Compose (identique au Script 1)
restart_docker_compose() {
    local CTID=$1
    local COMPOSE_DIR=$2 
    local TIMEOUT_DURATION=30

    echo "   Tentative de redémarrage avec 'docker compose' dans $COMPOSE_DIR (Timeout: ${TIMEOUT_DURATION}s)..."

    local LOG_FILE="/tmp/pct_exec_log_$CTID.log"
    timeout "$TIMEOUT_DURATION" pct exec "$CTID" -- sh -c "cd $COMPOSE_DIR && docker compose down && docker compose up -d" > "$LOG_FILE" 2>&1
    local exit_code=$?
    
    if [ $exit_code -eq 0 ]; then
        grep -E 'Removed|Created|Started' "$LOG_FILE"
        echo "   ✅ Redémarrage réussi avec 'docker compose'."
        rm -f "$LOG_FILE"
        return 0
    elif [ $exit_code -eq 124 ]; then
        echo "   ⚠️ Redémarrage forcé par timeout. Service probablement relancé. Continuer."
        rm -f "$LOG_FILE"
        return 0
    else
        echo "   ❌ Échec du redémarrage (Code: $exit_code). Consultez le fichier $LOG_FILE pour les détails."
        return 1
    fi
}


# 1. Identification des conteneurs LXC à traiter (identique au Script 1)
echo "Recherche des conteneurs LXC ACTIFS dont le nom contient 'docker' (via pct)..."
CONTAINER_IDS=$(pct list | awk '/running/ && /docker/ {print $1}')


# Traitement des Conteneurs (LOGIQUE DE RECHERCHE CORRIGÉE)
# =========================
if [ -z "$CONTAINER_IDS" ]; then
    echo "⚠️ Aucun conteneur LXC/CT en cours d'exécution avec 'docker' dans son nom n'a été trouvé. Aucune action n'a été effectuée."
    exit 0
fi

echo "Conteneurs trouvés (ID): $CONTAINER_IDS"
echo "-----------------------------------"

for CTID in $CONTAINER_IDS; do
    CONTAINER_NAME=$(pct config "$CTID" | grep 'hostname' | awk '{print $2}')
    echo "▶️ Traitement du conteneur LXC (ID $CTID) : $CONTAINER_NAME"
    
    # NOUVELLE LOGIQUE DE RECHERCHE: find avec une profondeur limitée (-maxdepth 4)
    TARGET_CONTAINER_PATH=$(pct exec "$CTID" -- sh -c "find /root -maxdepth 4 -name docker-compose.yml 2>/dev/null | grep 'watchtower/docker-compose.yml$' | head -n 1" 2>/dev/null)

    # Vérification du chemin trouvé
    if [ -z "$TARGET_CONTAINER_PATH" ]; then
        echo "   Fichier de configuration Watchtower non trouvé sous /root. Ce CT/LXC est ignoré."
        echo "-----------------------------------"
        continue
    fi
    
    # 2. Le chemin exact est maintenant stocké dans $TARGET_CONTAINER_PATH
    if [ -n "$TARGET_CONTAINER_PATH" ]; then
        echo "   Fichier de configuration Watchtower trouvé à: $TARGET_CONTAINER_PATH"

        # 3. Modification du fichier
        TEMP_FILE="/tmp/docker-compose-temp-$CTID.yml"
        
        pct pull "$CTID" "$TARGET_CONTAINER_PATH" "$TEMP_FILE" >/dev/null 2>&1
        if [ $? -ne 0 ]; then
            echo "❌ ERREUR: Impossible de copier le fichier depuis le conteneur $CTID. Ignoré."
            rm -f "$TEMP_FILE" 2>/dev/null
            continue
        fi

        TEMP_FILE_BAK="${TEMP_FILE}.bak"
        cp "$TEMP_FILE" "$TEMP_FILE_BAK"
        
        # Application de la substitution robuste
        sed -i "$MODIFICATION_SED" "$TEMP_FILE"
        echo "   ✅ Modification appliquée au fichier temporaire."
        
        pct exec "$CTID" -- cp "$TARGET_CONTAINER_PATH" "$TARGET_CONTAINER_PATH.bak" >/dev/null 2>&1
        pct push "$CTID" "$TEMP_FILE" "$TARGET_CONTAINER_PATH" >/dev/null 2>&1
        echo "   ✅ Fichier mis à jour dans le conteneur. Sauvegarde interne: ${TARGET_CONTAINER_PATH}.bak"
        
        rm -f "$TEMP_FILE" "$TEMP_FILE_BAK"
        
        # 4. Redémarrage du service Docker Compose
        COMPOSE_DIR=$(dirname "$TARGET_CONTAINER_PATH")
        
        restart_docker_compose "$CTID" "$COMPOSE_DIR" 
        
        if [ $? -eq 0 ]; then
             echo "✅ Opération complète terminée pour $CONTAINER_NAME (ID $CTID)."
        else
             echo "⚠️ Modification réussie, mais redémarrage échoué pour $CONTAINER_NAME (ID $CTID). **Veuillez redémarrer le service manuellement.**"
        fi
    else
        echo "   Fichier de configuration Watchtower non trouvé à $TARGET_DESCRIPTION. Ce CT/LXC est ignoré."
    fi
    
    echo "-----------------------------------"
done

echo "Toutes les opérations sont terminées."
```
---
title: Modification des fichiers docker-compose.yml de Watchtower (LXC/Proxmox)
description: Ce guide fournit deux scripts Bash conçus pour être exécutés sur l'hôte Proxmox (PVE) afin d'automatiser la mise à jour des conteneurs Watchtower (installés via Docker Compose dans des CT/LXC).
published: true
date: 2025-12-15T11:04:45.603Z
tags: docker, lxc, script, bash, watchtower
editor: markdown
dateCreated: 2025-10-26T16:28:46.835Z
---

| Script | Fonction | Utilité |
| :--- | :--- | :--- |
| **Script 1 :** `convert_watchtower.sh` | Conversion de `POLL_INTERVAL` à `SCHEDULE` | **À exécuter une seule fois** pour migrer l'ancienne variable vers la nouvelle (standard Cron). |
| **Script 2 :** `update_watchtower.sh` | Modification de la planification `SCHEDULE` | **À utiliser pour toutes les modifications futures** de la fréquence de mise à jour. |

-----

## I. Prérequis et installation

1.  **Connexion SSH :** Connectez-vous à votre hôte Proxmox en tant que `root`.
2.  **Création du répertoire :**
    ```bash
    mkdir -p /root/scripts
    cd /root/scripts
    ```
3.  **Création des fichiers :** Créez les deux fichiers de script (par exemple `convert_watchtower.sh` et `update_watchtower.sh`).
4.  **Permissions :** Rendez les scripts exécutables :
    ```bash
    chmod +x convert_watchtower.sh update_watchtower.sh
    ```

-----

## II. Script 1 : conversion initiale (`POLL_INTERVAL` vers `SCHEDULE`)

Ce script est conçu pour remplacer la ligne obsolète `WATCHTOWER_POLL_INTERVAL=10800` par la nouvelle syntaxe `WATCHTOWER_SCHEDULE`.

**Fichier :** `/root/scripts/convert_watchtower.sh`

```bash
#!/bin/bash

# Configuration
# ==============
# Remplacement: WATCHTOWER_POLL_INTERVAL=10800 sera remplacé par WATCHTOWER_SCHEDULE=0 0 10 ? * WED (Mercredi 10h00)
# Indentation standarde de 6 espaces pour garantir la validité du YAML.
INDENTATION_YAMLLINE='      '
MODIFICATION_SED_CLEAN='s/^[[:space:]]*//'
MODIFICATION_SED_REPLACE="s/^- WATCHTOWER_POLL_INTERVAL=10800/${INDENTATION_YAMLLINE}- WATCHTOWER_SCHEDULE=0 0 10 ? * WED/"
TARGET_DESCRIPTION="watchtower/docker-compose.yml sous /root"


# Fonction pour redémarrer Docker Compose (Inclut l'affichage des 3 dernières lignes de log en cas d'échec)
restart_docker_compose() {
    local CTID=$1
    local COMPOSE_DIR=$2 
    local TIMEOUT_DURATION=30

    echo "  Tentative de redémarrage avec 'docker compose' dans $COMPOSE_DIR (Timeout: ${TIMEOUT_DURATION}s)..."

    local LOG_FILE="/tmp/pct_exec_log_$CTID.log"
    timeout "$TIMEOUT_DURATION" pct exec "$CTID" -- sh -c "cd $COMPOSE_DIR && docker compose down && docker compose up -d" > "$LOG_FILE" 2>&1
    local exit_code=$?
    
    if [ $exit_code -eq 0 ]; then
        grep -E 'Removed|Created|Started' "$LOG_FILE"
        echo "  ✅ Redémarrage réussi avec 'docker compose'."
        rm -f "$LOG_FILE"
        return 0
    elif [ $exit_code -eq 124 ]; then
        echo "  ⚠️ Redémarrage forcé par timeout. Service probablement relancé. Continuer."
        rm -f "$LOG_FILE"
        return 0
    else
        echo "  ❌ Échec du redémarrage (Code: $exit_code). Détails du log :"
        pct exec "$CTID" -- tail -n 3 "$LOG_FILE" 2>/dev/null
        return 1
    fi
}


# 1. Identification des conteneurs LXC à traiter
echo "Recherche des conteneurs LXC ACTIFS dont le nom contient 'docker' (via pct)..."
CONTAINER_IDS=$(pct list | awk '/running/ && /docker/ {print $1}')


# Traitement des conteneurs
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
    
    # LOGIQUE DE RECHERCHE ROBUSTE : Trouve le docker-compose.yml contenant 'watchtower' sous /root (max 4 niveaux)
    TARGET_CONTAINER_PATH=$(pct exec "$CTID" -- sh -c "find /root -maxdepth 4 -name 'docker-compose.yml' 2>/dev/null | grep 'watchtower' | head -n 1" 2>/dev/null)

    # Vérification du chemin trouvé
    if [ -z "$TARGET_CONTAINER_PATH" ]; then
        echo "  Fichier de configuration Watchtower non trouvé sous /root. Ce CT/LXC est ignoré."
        echo "-----------------------------------"
        continue
    fi
    
    if [ -n "$TARGET_CONTAINER_PATH" ]; then
        echo "  Fichier de configuration Watchtower trouvé à: $TARGET_CONTAINER_PATH"

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
        
        # === DOUBLE SUBSTITUTION POUR GARANTIR LA CORRECTION DE L'INDENTATION ===
        
        # 1. Nettoyage : Supprime tous les espaces/tabulations en début de ligne sur la ligne ciblée (POLL_INTERVAL).
        sed -i '/WATCHTOWER_POLL_INTERVAL=/ s/^[[:space:]]*//' "$TEMP_FILE"
        
        # 2. Remplacement : Convertit POLL_INTERVAL en SCHEDULE, ajoutant l'indentation fixe.
        sed -i "/WATCHTOWER_POLL_INTERVAL=/ s/^- WATCHTOWER_POLL_INTERVAL=.*|${MODIFICATION_SED_REPLACE}/" "$TEMP_FILE"

        # === FIN DE LA DOUBLE SUBSTITUTION ===
        
        # VÉRIFICATION DE LA MODIFICATION
        if cmp -s "$TEMP_FILE" "$TEMP_FILE_BAK"; then
            echo "  ⚠️ Aucune ligne correspondante (WATCHTOWER_POLL_INTERVAL) n'a été trouvée pour modification. Fichier non mis à jour."
            rm -f "$TEMP_FILE" "$TEMP_FILE_BAK"
            echo "-----------------------------------"
            continue
        fi

        echo "  ✅ Modification appliquée au fichier temporaire."
        
        # Sauvegarde l'original dans le CT, puis pousse le fichier modifié
        pct exec "$CTID" -- cp "$TARGET_CONTAINER_PATH" "$TARGET_CONTAINER_PATH.bak" >/dev/null 2>&1
        pct push "$CTID" "$TEMP_FILE" "$TARGET_CONTAINER_PATH" >/dev/null 2>&1
        echo "  ✅ Fichier mis à jour dans le conteneur. Sauvegarde interne: ${TARGET_CONTAINER_PATH}.bak"
        
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
        echo "  Fichier de configuration Watchtower non trouvé à $TARGET_DESCRIPTION. Ce CT/LXC est ignoré."
    fi
    
    echo "-----------------------------------"
done

echo "Toutes les opérations sont terminées."
```

-----

## III. Script 2 : modification de planification (`SCHEDULE`)

Ce script est le script de routine. Il met à jour la valeur CRON `WATCHTOWER_SCHEDULE` dans tous les fichiers `docker-compose.yml` pertinents.

**Fichier :** `/root/scripts/update_watchtower.sh`

```bash
#!/bin/bash

# Configuration
# ==============
# La nouvelle valeur CRON. (Exemple: 0 0 10 ? * FRI = Tous les Vendredis à 10:00)
NOUVELLE_VALEUR_CRON='0 0 10 ? * FRI'
# Indentation standarde de 6 espaces pour garantir la validité du YAML.
INDENTATION_YAMLLINE='      '

TARGET_DESCRIPTION="watchtower/docker-compose.yml sous /root" 


# Fonction pour redémarrer Docker Compose (inchangée)
restart_docker_compose() {
    local CTID=$1
    local COMPOSE_DIR=$2 
    local TIMEOUT_DURATION=30

    echo "  Tentative de redémarrage avec 'docker compose' dans $COMPOSE_DIR (Timeout: ${TIMEOUT_DURATION}s)..."

    local LOG_FILE="/tmp/pct_exec_log_$CTID.log"
    timeout "$TIMEOUT_DURATION" pct exec "$CTID" -- sh -c "cd $COMPOSE_DIR && docker compose down && docker compose up -d" > "$LOG_FILE" 2>&1
    local exit_code=$?
    
    if [ $exit_code -eq 0 ]; then
        grep -E 'Removed|Created|Started' "$LOG_FILE"
        echo "  ✅ Redémarrage réussi avec 'docker compose'."
        rm -f "$LOG_FILE"
        return 0
    elif [ $exit_code -eq 124 ]; then
        echo "  ⚠️ Redémarrage forcé par timeout. Service probablement relancé. Continuer."
        rm -f "$LOG_FILE"
        return 0
    else
        echo "  ❌ Échec du redémarrage (Code: $exit_code). Détails du log :"
        pct exec "$CTID" -- tail -n 3 "$LOG_FILE" 2>/dev/null
        return 1
    fi
}


# 1. Identification des conteneurs LXC à traiter
echo "Recherche des conteneurs LXC ACTIFS dont le nom contient 'docker' (via pct)..."
CONTAINER_IDS=$(pct list | awk '/running/ && /docker/ {print $1}')


# Traitement des conteneurs
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
    
    # LOGIQUE DE RECHERCHE ROBUSTE : Trouve le docker-compose.yml contenant 'watchtower' sous /root (max 4 niveaux)
    TARGET_CONTAINER_PATH=$(pct exec "$CTID" -- sh -c "find /root -maxdepth 4 -name 'docker-compose.yml' 2>/dev/null | grep 'watchtower' | head -n 1" 2>/dev/null)

    # Vérification du chemin trouvé
    if [ -z "$TARGET_CONTAINER_PATH" ]; then
        echo "  Fichier de configuration Watchtower non trouvé sous /root. Ce CT/LXC est ignoré."
        echo "-----------------------------------"
        continue
    fi
    
    if [ -n "$TARGET_CONTAINER_PATH" ]; then
        echo "  Fichier de configuration Watchtower trouvé à: $TARGET_CONTAINER_PATH"

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
        
        # === DOUBLE SUBSTITUTION POUR GARANTIR LA CORRECTION DE L'INDENTATION ===
        
        # 1. Nettoyage : Supprime tous les espaces/tabulations en début de ligne sur la ligne ciblée (SCHEDULE).
        sed -i '/WATCHTOWER_SCHEDULE=/ s/^[[:space:]]*//' "$TEMP_FILE"
        
        # 2. Remplacement : Ajoute l'indentation fixe (6 espaces) et la nouvelle valeur CRON.
        sed -i "/WATCHTOWER_SCHEDULE=/ s/^- WATCHTOWER_SCHEDULE=.*/${INDENTATION_YAMLLINE}- WATCHTOWER_SCHEDULE=${NOUVELLE_VALEUR_CRON}/" "$TEMP_FILE"

        # === FIN DE LA DOUBLE SUBSTITUTION ===
        
        # VÉRIFICATION DE LA MODIFICATION
        if cmp -s "$TEMP_FILE" "$TEMP_FILE_BAK"; then
            echo "  ⚠️ Aucune ligne correspondante (WATCHTOWER_SCHEDULE) n'a été trouvée pour modification. Fichier non mis à jour. (Probablement déjà en place)"
            rm -f "$TEMP_FILE" "$TEMP_FILE_BAK"
            echo "-----------------------------------"
            continue
        fi

        echo "  ✅ Modification appliquée au fichier temporaire."
        
        # Sauvegarde l'original dans le CT, puis pousse le fichier modifié
        pct exec "$CTID" -- cp "$TARGET_CONTAINER_PATH" "$TARGET_CONTAINER_PATH.bak" >/dev/null 2>&1
        pct push "$CTID" "$TEMP_FILE" "$TARGET_CONTAINER_PATH" >/dev/null 2>&1
        echo "  ✅ Fichier mis à jour dans le conteneur. Sauvegarde interne: ${TARGET_CONTAINER_PATH}.bak"
        
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
        echo "  Fichier de configuration Watchtower non trouvé à $TARGET_DESCRIPTION. Ce CT/LXC est ignoré."
    fi
    
    echo "-----------------------------------"
done

echo "Toutes les opérations sont terminées."
```
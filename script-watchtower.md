---
title: Script de Maintenance Proxmox : Mise à Jour de Watchtower
description: Ce script ajuste la planification de l'outil de mise à jour automatique des images Docker, Watchtower.
published: true
date: 2025-10-26T16:28:46.835Z
tags: docker, lxc, script, bash, watchtower
editor: markdown
dateCreated: 2025-10-26T16:28:46.835Z
---

## 🚀 Script de Maintenance Proxmox : Mise à Jour de Watchtower

Ce script Bash est conçu pour les administrateurs utilisant **Proxmox Virtual Environment (VE)** pour héberger des conteneurs LXC exécutant Docker. Il automatise la modification des fichiers `docker-compose.yml` pour ajuster la planification de l'outil de mise à jour automatique des images Docker, **Watchtower**.

### 🎯 À Quoi Sert Ce Script ?

| Champ | Description |
| :--- | :--- |
| **Objectif Principal** | Remplacer l'intervalle de mise à jour de Watchtower (p. ex., toutes les 3 heures) par une **planification CRON spécifique** (p. ex., tous les mercredis à 10h00). |
| **Pour Qui ?** | Les utilisateurs de Proxmox qui gèrent des LXC avec des services Docker (`docker-compose`) et qui veulent **éviter les mises à jour aléatoires** qui pourraient perturber les services en production. |
| **Avantages Techniques** | Il gère les subtilités de l'environnement Proxmox : utilise `pct exec` et `pct pull/push` au lieu de `lxc-attach`, filtre uniquement les conteneurs actifs et évite les blocages sur les conteneurs mal configurés. |

-----

## 💻 Installation et Utilisation

Ce script doit être exécuté en tant qu'utilisateur **root** sur votre nœud hôte Proxmox VE.

### Étape 1 : Créer et Rendre Exécutable le Fichier

1.  Connectez-vous à votre nœud Proxmox en SSH.
2.  Créez le fichier du script (par exemple, `update_watchtower.sh`) :
    ```bash
    nano /root/update_watchtower.sh
    ```
3.  Collez le code ci-dessous (version finale stable).
4.  Rendez le script exécutable :
    ```bash
    chmod +x /root/update_watchtower.sh
    ```

### Étape 2 : Le Script (Version Finale Stable)

```bash
#!/bin/bash

# Configuration
# ==============
# Remplacement: WATCHTOWER_POLL_INTERVAL=10800 sera remplacé par WATCHTOWER_SCHEDULE=0 0 10 ? * WED (Mercredi 10h00)
MODIFICATION_SED='s/- WATCHTOWER_POLL_INTERVAL=10800/- WATCHTOWER_SCHEDULE=0 0 10 ? * WED/'
TARGET_CONTAINER_PATH_TO_TEST="/root/watchtower/docker-compose.yml" # Emplacement du fichier Watchtower à l'intérieur du conteneur


# Fonction pour redémarrer Docker Compose
restart_docker_compose() {
    local CTID=$1
    local COMPOSE_DIR="/root/watchtower"
    local TIMEOUT_DURATION=30

    echo "   Tentative de redémarrage avec 'docker compose' dans $COMPOSE_DIR (Timeout: ${TIMEOUT_DURATION}s)..."

    local LOG_FILE="/tmp/pct_exec_log_$CTID.log"
    # La commande utilise 'timeout' pour éviter de bloquer le script et redirige la sortie.
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

# Filtre les conteneurs qui sont 'running' et dont le nom contient 'docker'.
CONTAINER_IDS=$(pct list | awk '/running/ && /docker/ {print $1}')


# Traitement des Conteneurs
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
    
    # 2. Vérification de l'existence du fichier (méthode non bloquante : test -f)
    pct exec "$CTID" -- test -f "$TARGET_CONTAINER_PATH_TO_TEST" >/dev/null 2>&1
    
    if [ $? -eq 0 ]; then
        echo "   Fichier de configuration Watchtower trouvé à: $TARGET_CONTAINER_PATH_TO_TEST"

        # 3. Modification du fichier (pct pull/push)
        TEMP_FILE="/tmp/docker-compose-temp-$CTID.yml"
        
        # Copie le fichier du CT vers l'hôte
        pct pull "$CTID" "$TARGET_CONTAINER_PATH_TO_TEST" "$TEMP_FILE" >/dev/null 2>&1
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
        
        # Sauvegarde l'original dans le CT, puis pousse le fichier modifié
        pct exec "$CTID" -- cp "$TARGET_CONTAINER_PATH_TO_TEST" "$TARGET_CONTAINER_PATH_TO_TEST.bak" >/dev/null 2>&1
        pct push "$CTID" "$TEMP_FILE" "$TARGET_CONTAINER_PATH_TO_TEST" >/dev/null 2>&1
        echo "   ✅ Fichier mis à jour dans le conteneur. Sauvegarde interne: ${TARGET_CONTAINER_PATH_TO_TEST}.bak"
        
        rm -f "$TEMP_FILE" "$TEMP_FILE_BAK"
        
        # 4. Redémarrage du service Docker Compose
        restart_docker_compose "$CTID"
        
        if [ $? -eq 0 ]; then
             echo "✅ Opération complète terminée pour $CONTAINER_NAME (ID $CTID)."
        else
             echo "⚠️ Modification réussie, mais redémarrage échoué pour $CONTAINER_NAME (ID $CTID). **Veuillez redémarrer le service manuellement.**"
        fi
    else
        echo "   Fichier de configuration Watchtower non trouvé à $TARGET_CONTAINER_PATH_TO_TEST. Ce CT/LXC est ignoré."
    fi
    
    echo "-----------------------------------"
done

echo "Toutes les opérations sont terminées."
```

### Étape 3 : Exécuter le Script

Pour lancer le script :

```bash
/root/update_watchtower.sh
```

-----

## 🔧 Adaptation du Script pour d'Autres Usages

Le script est conçu pour être facilement adaptable à toute autre modification de fichier dans vos conteneurs LXC.

### 1\. Adapter la Cible (Où et Qui)

Pour modifier le comportement de ciblage et de recherche :

| Variable | Usage | Comment l'Adapter |
| :--- | :--- | :--- |
| **`CONTAINER_IDS`** | **Ciblage :** Liste les CT à traiter. | Modifiez l'`awk` pour cibler des noms différents (p. ex., `/nginx/` au lieu de `/docker/`) ou retirez `/docker/` pour cibler tous les LXC actifs. |
| **`TARGET_CONTAINER_PATH_TO_TEST`** | **Chemin du Fichier :** Chemin exact du fichier *dans le conteneur*. | Modifiez-le pour pointer vers tout autre fichier de configuration (p. ex., `/etc/nginx/nginx.conf`). |
| **`COMPOSE_DIR`** | **Répertoire de Travail :** Utilisé pour le `cd` avant le redémarrage. | Si vous modifiez un fichier qui nécessite un redémarrage avec `docker compose`, mettez ici le répertoire contenant le `docker-compose.yml`. |

### 2\. Adapter la Modification (Quoi)

La variable **`MODIFICATION_SED`** contient l'expression de substitution qui effectue la modification. Elle est au format `sed`: `'s/CHAÎNE_RECHERCHÉE/CHAÎNE_DE_REMPLACEMENT/'`.

| Élément | Description | Exemple d'Adaptation |
| :--- | :--- | :--- |
| **Recherche** (`CHAÎNE_RECHERCHÉE`) | La ligne exacte ou le motif à trouver. | Pour changer un port `8080:8080`, utilisez : `8080:8080`. |
| **Remplacement** (`CHAÎNE_DE_REMPLACEMENT`) | La nouvelle valeur à insérer. | Pour changer le port en `8081:8080`, utilisez : `8081:8080`. |
| **Variable** | **Modification :** Remplacer une ligne par une autre. | `MODIFICATION_SED='s/old_value/new_value/'` |

**Exemple d'adaptation :** Si vous vouliez remplacer l'adresse d'un serveur dans un fichier de configuration :

```bash
# Remplacer 'http://old-server:8080' par 'http://new-server:9000'
MODIFICATION_SED='s/http:\/\/old-server:8080/http:\/\/new-server:9000/' 
```

*(Attention aux barres obliques `/` qui doivent être échappées par `\` dans les expressions `sed` si elles font partie de la chaîne de recherche/remplacement.)*

### 3\. Adapter l'Action Post-Modification

L'étape de redémarrage est gérée par la fonction **`restart_docker_compose`**.

  * **Si vous modifiez un service Docker :** Laissez la fonction telle quelle, car elle gère le redémarrage de `docker compose`.
  * **Si vous modifiez un service système (ex: Nginx) :** Modifiez le corps de la fonction pour utiliser `pct exec "$CTID" -- systemctl restart nginx` au lieu de `docker compose down/up`. Pensez à ajuster le `TIMEOUT_DURATION` si nécessaire.
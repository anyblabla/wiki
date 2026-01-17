---
title: Normalisation des tags Proxmox VE
description: Apprenez à normaliser les étiquettes sur un cluster Proxmox VE. Ce script Bash automatise le renommage massif du tag community-scripts en community-script sur l'ensemble de vos nœuds PVE.
published: true
date: 2026-01-17T12:54:56.909Z
tags: proxmox, bash, administration système, automatisation, tag, étiquette
editor: markdown
dateCreated: 2026-01-17T12:54:56.909Z
---

## 1. Introduction

Dans un environnement Proxmox VE multi-nœuds (cluster), la cohérence des tags est essentielle pour l'organisation et l'automatisation. Ce script permet de corriger massivement une erreur de nommage (remplacer `community-scripts` par `community-script`) sur l'ensemble des ressources du cluster.

## 2. Le script Bash

Voici le code complet à enregistrer sous `rename_tags.sh` :

```bash
#!/bin/bash

# Couleurs pour le monitoring
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m'

echo -e "${BLUE}=== Début de la mise à jour des tags sur le cluster ===${NC}"

# 1. Extraction : type, VMID et le nœud associé pour chaque ressource avec le tag cible
resources=$(pvesh get /cluster/resources --type vm --output-format json | \
            jq -r '.[] | select(.tags != null) | select(.tags | contains("community-scripts")) | "\(.type)/\(.vmid)/\(.node)"')

if [ -z "$resources" ]; then
    echo -e "${YELLOW}Aucun tag 'community-scripts' détecté.${NC}"
    exit 0
fi

for res in $resources; do
    TYPE=$(echo $res | cut -d'/' -f1)
    VMID=$(echo $res | cut -d'/' -f2)
    NODE=$(echo $res | cut -d'/' -f3)

    # Chemin API précis vers le nœud propriétaire de la ressource
    CONF_PATH="/nodes/${NODE}/${TYPE}/${VMID}/config"

    # 2. Récupération des tags actuels
    current_tags=$(pvesh get $CONF_PATH --output-format json | jq -r '.tags // ""')

    # 3. Double vérification et remplacement
    if [[ "$current_tags" == *"community-scripts"* ]]; then
        new_tags=$(echo "$current_tags" | sed 's/community-scripts/community-script/g')

        echo -e "Nœud [${BLUE}${NODE}${NC}] -> ${TYPE} ${GREEN}${VMID}${NC}"
        echo -e "   Ancien: ${current_tags}"
        echo -e "   Nouveau: ${new_tags}"

        # 4. Application de la modification via l'API Proxmox
        pvesh set $CONF_PATH --tags "$new_tags"
    fi
done

echo -e "${BLUE}=== Opération terminée ===${NC}"

```

## 3. Pourquoi ce script ?

L'utilisation de l'interface graphique pour renommer des tags sur un grand nombre de machines virtuelles et conteneurs (LXC) est chronophage.

* **Automatisation** : Le script parcourt l'ensemble du cluster (`PVE1`, `PVE2`, etc.) instantanément.
* **Intégrité** : Il préserve les autres tags associés (ex : `debian`, `docker`).
* **Flexibilité** : Il peut être exécuté depuis n'importe quel nœud membre du cluster.

## 4. Fonctionnement technique

Le script s'appuie sur deux outils :

1. **`pvesh`** : Utilitaire en ligne de commande qui interagit avec l'API locale de Proxmox.
2. **`jq`** : Processeur JSON utilisé pour filtrer les données retournées par l'API.

Il identifie dynamiquement sur quel nœud physique se trouve la ressource pour envoyer la commande de modification au bon endroit dans le système de fichiers distribué `/etc/pve`.

## 5. Installation et utilisation

Le script nécessite le paquet `jq`. Pour l'installer :

```bash
apt update && apt install jq -y

```

**Exécution :**

```bash
chmod +x rename_tags.sh
./rename_tags.sh

```

## 6. Adaptation

Pour renommer un autre tag, il suffit de modifier deux valeurs :

* Le filtre `contains("ancien-tag")` dans la requête `resources`.
* La règle de substitution `sed 's/ancien-tag/nouveau-tag/g'`.

Tu peux insérer ce bloc de code juste après le paragraphe **"5. Installation et utilisation"**, ou créer une nouvelle section dédiée aux résultats.

Je te conseille de l'appeler **"Exemple de sortie du script"** pour illustrer concrètement l'action du script sur ton cluster de 6 nœuds.

Voici comment l'intégrer dans ta page de wiki :

## 7. Exemple de sortie du script

Voici un aperçu du journal d'exécution montrant le traitement dynamique sur les différents nœuds (`pvenocl2` à `pvenocl6`) :

```text
root@pvenocl:~/scripts# ./rename_tags.sh 
=== Début de la mise à jour des tags sur le cluster ===
Nœud [pvenocl5] -> lxc 102
   Ancien: 192.168.2.218;adblock;adguard;community-scripts;debian;dns;linux;lxc;server
   Nouveau: 192.168.2.218;adblock;adguard;community-script;debian;dns;linux;lxc;server
Nœud [pvenocl3] -> lxc 103
   Ancien: 192.168.2.253;community-scripts;debian;docker;linux;lxc;rss;server;watchtower
   Nouveau: 192.168.2.253;community-script;debian;docker;linux;lxc;rss;server;watchtower
Nœud [pvenocl2] -> lxc 109
   Ancien: 192.168.2.104;community-scripts;debian;lan;linux;lxc;server;wake
   Nouveau: 192.168.2.104;community-script;debian;lan;linux;lxc;server;wake
Nœud [pvenocl4] -> lxc 116
   Ancien: 192.168.2.219;adblock;adguard;community-scripts;debian;dns;linux;lxc;server
   Nouveau: 192.168.2.219;adblock;adguard;community-script;debian;dns;linux;lxc;server
Nœud [pvenocl6] -> lxc 127
   Ancien: 192.168.2.194;community-scripts;debian;linux;lxc;mail;server
   Nouveau: 192.168.2.194;community-script;debian;linux;lxc;mail;server
...
=== Opération terminée ===

```
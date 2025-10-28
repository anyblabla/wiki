---
title: Automatisation de la mise à jour des VMs et LXC Proxmox
description: Ce guide fournit deux scripts Bash à exécuter via Cron sur votre hôte Proxmox VE pour automatiser la mise à jour des machines virtuelles (VMs) et des conteneurs (LXC) basés sur Debian/Ubuntu.
published: true
date: 2025-10-28T14:58:39.723Z
tags: lxc, proxmox, cron, crontab, script, vm
editor: markdown
dateCreated: 2025-10-26T16:38:37.191Z
---

## ⚠️ Avertissements cruciaux avant l'automatisation

### 1\. Clusters avec HA (Haute Disponibilité) 🚫

**Il est fortement déconseillé d'utiliser ces scripts sur un cluster Proxmox ayant la Haute Disponibilité (HA) activée.** Les actions de redémarrage (`qm shutdown/start` ou `pct reboot`) pourraient être interprétées comme des défaillances par le système HA, entraînant des conflits ou des migrations imprévues.

### 2\. Le redémarrage n'est pas garanti 🔄

Le script détecte la nécessité d'un redémarrage en vérifiant le marqueur `/var/run/reboot-required` à l'intérieur de l'invité. Cette vérification et la mise à jour nécessitent :

  * Le **QEMU Guest Agent** installé et fonctionnel (pour les VMs).
  * Un OS invité basé sur **Debian/Ubuntu**.

-----

## I. Script pour les machines virtuelles (VMs)

Ce script utilise l'agent invité (`qm guest exec`) pour mettre à jour les VMs et les redémarrer en toute sécurité si nécessaire.

### Étape 1 : création du script `update_vms.sh`

1.  Connectez-vous à votre hôte Proxmox en SSH (`root`).
2.  Créez le fichier de script :
    ```bash
    nano /usr/local/bin/update_vms.sh
    ```
3.  Collez le contenu suivant :

<!-- end list -->

```bash
#!/bin/bash
#
# SCRIPT : update_vms.sh
# OBJECTIF : Mettre à jour toutes les VMs Debian/Ubuntu en cours d'exécution
#
# ==============================================================================

LOGFILE="/var/log/update_vms_cron.log"
EXCLUDED_VMS="" # IDs de VMs à exclure (séparés par des espaces)

exec 1>>$LOGFILE 2>&1

echo "=================================================="
echo "Démarrage de la mise à jour des VMs le $(date)"
echo "=================================================="

# Commandes internes pour la VM (utilise apt-get pour la stabilité)
UPDATE_COMMAND="export DEBIAN_FRONTEND=noninteractive LC_ALL=C.UTF-8 && \
                apt-get update -y && \
                apt-get full-upgrade -y && \
                apt-get autoremove -y && \
                apt-get clean && \
                snap refresh 2>/dev/null || true" 

REBOOT_CHECK_COMMAND="[ -f /var/run/reboot-required ] && echo 'REBOOT_YES' || echo 'REBOOT_NO'"

# Boucle sur les VMs en cours d'exécution
for VMID in $(/usr/sbin/qm list | grep running | awk '{print $1}')
do
    if [[ " $EXCLUDED_VMS " =~ " $VMID " ]]; then
        echo "    [SKIP] VM $VMID exclue."
        continue 
    fi
    
    echo "--> Traitement de la VM VMID $VMID..."
    
    # 2. Exécution des mises à jour
    echo "    - Exécution des mises à jour..."
    /usr/sbin/qm guest exec $VMID --timeout 300 /bin/bash -- -c "$UPDATE_COMMAND"
    
    if [ $? -ne 0 ]; then
        echo "    [ERREUR CRITIQUE] La mise à jour de la VM $VMID a échoué. Poursuite vers la prochaine VM."
        continue
    fi
    
    # 3. Vérification du besoin de redémarrage
    echo "    - Vérification du besoin de redémarrage..."
    REBOOT_CHECK_OUTPUT=$(/usr/sbin/qm guest exec $VMID --timeout 60 /bin/bash -- -c "$REBOOT_CHECK_COMMAND")
    
    if [[ "$REBOOT_CHECK_OUTPUT" == *"REBOOT_YES"* ]]; then

        echo "    [ALERTE] Redémarrage nécessaire pour la VM $VMID. Redémarrage en cours..."

        # 4. Redémarrage sécurisé
        echo "    - Arrêt de la VM $VMID (shutdown)..."
        /usr/sbin/qm shutdown $VMID --timeout 120 

        if /usr/sbin/qm status $VMID | grep -q running; then
            echo "    - Arrêt gracieux échoué. Forçage de l'arrêt (stop)..."
            /usr/sbin/qm stop $VMID
            sleep 5
        fi

        echo "    - Démarrage de la VM $VMID..."
        /usr/sbin/qm start $VMID
        echo "    [OK] Redémarrage de la VM $VMID terminé."

    else
        echo "    [OK] VM $VMID mise à jour avec succès. Aucun redémarrage critique nécessaire."
    fi
    
done

echo "=================================================="
echo "Fin de la mise à jour des VMs le $(date)"
echo "=================================================="

exec 1>&- 2>&-
```

### Étape 2 : rendre le script exécutable

```bash
chmod +x /usr/local/bin/update_vms.sh
```

-----

## II. Script pour les conteneurs (LXC)

Ce script utilise l'outil `pct` pour mettre à jour les conteneurs et les redémarrer directement.

### Étape 1 : création du script `update_lxcs.sh`

1.  Créez le fichier de script sur l'hôte Proxmox :
    ```bash
    nano /usr/local/bin/update_lxcs.sh
    ```
2.  Collez le contenu suivant :

<!-- end list -->

```bash
#!/bin/bash
#
# SCRIPT : update_lxcs.sh
# OBJECTIF : Mettre à jour tous les conteneurs LXC en cours d'exécution
#
# ==============================================================================

LOGFILE="/var/log/update_lxcs_cron.log"
EXCLUDED_CTIDS="" # CTIDs à exclure (séparés par des espaces)

exec 1>>$LOGFILE 2>&1

echo "=================================================="
echo "Démarrage de la mise à jour des LXC le $(date)"
echo "=================================================="

# Commandes internes pour le conteneur
UPDATE_COMMAND="export DEBIAN_FRONTEND=noninteractive LC_ALL=C.UTF-8 && \
                apt-get update -y && \
                apt-get full-upgrade -y && \
                apt-get autoremove -y && \
                apt-get clean && \
                snap refresh 2>/dev/null || true" 

REBOOT_CHECK_COMMAND="[ -f /var/run/reboot-required ] && echo 'REBOOT_YES' || echo 'REBOOT_NO'"

# Boucle sur les conteneurs en cours d'exécution
for CTID in $(/usr/sbin/pct list | grep running | awk '{print $1}')
do
    if [[ " $EXCLUDED_CTIDS " =~ " $CTID " ]]; then
        echo "    [SKIP] Conteneur $CTID exclu."
        continue 
    fi
    
    echo "--> Traitement du conteneur CTID $CTID..."
    
    # 2. Exécution des mises à jour
    echo "    - Exécution des mises à jour..."
    /usr/sbin/pct exec $CTID -- bash -c "$UPDATE_COMMAND"
    
    if [ $? -ne 0 ]; then
        echo "    [ERREUR CRITIQUE] La mise à jour du conteneur $CTID a échoué. Poursuite vers le prochain LXC."
        continue
    fi
    
    # 3. Vérification du besoin de redémarrage
    echo "    - Vérification du besoin de redémarrage..."
    REBOOT_CHECK_OUTPUT=$(/usr/sbin/pct exec $CTID -- bash -c "$REBOOT_CHECK_COMMAND")
    
    if [[ "$REBOOT_CHECK_OUTPUT" == *"REBOOT_YES"* ]]; then

        echo "    [ALERTE] Redémarrage nécessaire pour le conteneur $CTID. Redémarrage en cours..."
        
        # 4. Redémarrage direct (pct reboot)
        /usr/sbin/pct reboot $CTID --timeout 120 

        echo "    [OK] Redémarrage du conteneur $CTID terminé."

    else
        echo "    [OK] Conteneur $CTID mis à jour avec succès. Aucun redémarrage critique nécessaire."
    fi
    
done

echo "=================================================="
echo "Fin de la mise à jour des LXC le $(date)"
echo "=================================================="

exec 1>&- 2>&-
```

### Étape 2 : rendre le script exécutable

```bash
chmod +x /usr/local/bin/update_lxcs.sh
```

-----

## III. Planification avec Cron 📅

Utilisez la **crontab de l'utilisateur `root`** pour garantir les privilèges nécessaires.

1.  Ouvrez la crontab :

    ```bash
    crontab -e
    ```

2.  Ajoutez les lignes de planification. Par exemple, pour les exécuter tous les **dimanches à 4h00 et 4h30 du matin** :

    ```cron
    # Mise à jour des VMs (tous les dimanches à 4h00)
    0 4 * * 0 /usr/local/bin/update_vms.sh

    # Mise à jour des LXC (tous les dimanches à 4h30)
    30 4 * * 0 /usr/local/bin/update_lxcs.sh
    ```

Voulez-vous que je vérifie un autre de vos documents pour des incohérences similaires ?
---
title: Script d'automatisation des mises à jour des VMs et LXC Proxmox avec notification Gotify
description: Scripts Cron pour VMs/LXC Proxmox : Automatisez la mise à jour (apt-get/snap) et le redémarrage sécurisé de vos conteneurs et machines virtuelles Debian/Ubuntu. Recevez une notification détaillée via Gotify.
published: true
date: 2025-10-30T22:12:10.436Z
tags: lxc, proxmox, cron, crontab, script, bash, vm, pve, gotify
editor: markdown
dateCreated: 2025-10-30T22:12:10.436Z
---

## ⚠️ Avertissements cruciaux avant l'automatisation

### 1\. Clusters avec HA (Haute Disponibilité) 🚫

**Il est fortement déconseillé d'utiliser ces scripts sur un cluster Proxmox ayant la Haute Disponibilité (HA) activée.** Les actions de redémarrage (`qm shutdown/start` ou `pct reboot`) pourraient être interprétées comme des défaillances par le système HA, entraînant des conflits ou des migrations imprévues.

### 2\. Le redémarrage n'est pas garanti 🔄

Le script détecte la nécessité d'un redémarrage en vérifiant le marqueur `/var/run/reboot-required` à l'intérieur de l'invité. Cette vérification et la mise à jour nécessitent :

  * Le **QEMU Guest Agent** installé et fonctionnel (pour les VMs).
  * Un OS invité basé sur **Debian/Ubuntu**.

-----

## I. Pré-requis : installation de `curl` et configuration Gotify 🔔

### 1\. Installation de `curl`

Si ce paquet n'est pas installé sur votre hôte Proxmox, installez-le :

```bash
apt-get install curl -y
```

### 2\. Informations Gotify

Vous aurez besoin de deux informations à insérer dans les deux scripts :

  * **URL Gotify** : L'URL complète de votre serveur (ex: `https://gotify.mondomaine.com`).
  * **Token Gotify** : Le jeton (Token) de l'application Gotify.

-----

## II. Script pour les machines virtuelles (VMs)

### Étape 1 : création du script `update_vms.sh`

1.  Connectez-vous à votre hôte Proxmox en SSH (`root`).
2.  Créez le fichier de script :
    ```bash
    nano /usr/local/bin/update_vms.sh
    ```
3.  Collez le contenu suivant. **⚠️ Remplacez `VOTRE_URL_GOTIFY` et `VOTRE_TOKEN_GOTIFY` par vos propres valeurs.**

<!-- end list -->

```bash
#!/bin/bash
#
# SCRIPT : update_vms.sh
# OBJECTIF : Mettre à jour toutes les VMs Debian/Ubuntu en cours d'exécution
#
# ==============================================================================

# --- PARAMÈTRES DE GOTIFY ---
GOTIFY_URL="VOTRE_URL_GOTIFY"
GOTIFY_TOKEN="VOTRE_TOKEN_GOTIFY"

# --- PARAMÈTRES DU SCRIPT ---
LOGFILE="/var/log/update_vms_cron.log"
EXCLUDED_VMS="" # IDs de VMs à exclure (séparés par des espaces)
SUCCESS_COUNT=0
FAILURE_COUNT=0
REBOOT_LIST=""

exec 1>>$LOGFILE 2>&1

# --- FONCTION DE NOTIFICATION GOTIFY (MÉTHODE FORM-DATA) ---
send_gotify_notification() {
    local title="$1"
    local message="$2"
    local priority="$3"
    curl -k -s -X POST "$GOTIFY_URL/message?token=$GOTIFY_TOKEN" \
        -F "title=$title" \
        -F "message=$message" \
        -F "priority=$priority" > /dev/null 2>&1
}

echo "=================================================="
echo "Démarrage de la mise à jour des VMs le $(date)"
echo "=================================================="

# Commandes internes pour la VM
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
        echo "    [SKIP] VM $VMID exclue."
        continue 
    fi
    
    echo "--> Traitement de la VM VMID $VMID..."
    
    # 2. Exécution des mises à jour
    echo "    - Exécution des mises à jour..."
    /usr/sbin/qm guest exec $VMID --timeout 300 /bin/bash -- -c "$UPDATE_COMMAND"
    
    if [ $? -ne 0 ]; then
        echo "    [ERREUR CRITIQUE] La mise à jour de la VM $VMID a échoué. Poursuite vers la prochaine VM."
        FAILURE_COUNT=$((FAILURE_COUNT + 1))
        continue
    fi
    
    # 3. Vérification du besoin de redémarrage
    echo "    - Vérification du besoin de redémarrage..."
    REBOOT_CHECK_OUTPUT=$(/usr/sbin/qm guest exec $VMID --timeout 60 /bin/bash -- -c "$REBOOT_CHECK_COMMAND")
    
    if [[ "$REBOOT_CHECK_OUTPUT" == *"REBOOT_YES"* ]]; then

        echo "    [ALERTE] Redémarrage nécessaire pour la VM $VMID. Redémarrage en cours..."
        REBOOT_LIST="$REBOOT_LIST VMID $VMID"

        # 4. Redémarrage sécurisé
        echo "    - Arrêt de la VM $VMID (shutdown)..."
        /usr/sbin/qm shutdown $VMID --timeout 120 

        if /usr/sbin/qm status $VMID | grep -q running; then
            echo "    - Arrêt gracieux échoué. Forçage de l'arrêt (stop)..."
            /usr/sbin/qm stop $VMID
            sleep 5
        fi

        echo "    - Démarrage de la VM $VMID..."
        /usr/sbin/qm start $VMID
        echo "    [OK] Redémarrage de la VM $VMID terminé."

    else
        echo "    [OK] VM $VMID mise à jour avec succès. Aucun redémarrage critique nécessaire."
    fi
    SUCCESS_COUNT=$((SUCCESS_COUNT + 1))
done

echo "=================================================="
echo "Fin de la mise à jour des VMs le $(date)"
echo "=================================================="

# --- ENVOI DE LA NOTIFICATION FINALE ---
TOTAL_VMS=$((SUCCESS_COUNT + FAILURE_COUNT))

if [ $FAILURE_COUNT -gt 0 ]; then
    TITLE="❌ VMs Update ÉCHEC(s) sur $HOSTNAME"
    MESSAGE="$FAILURE_COUNT VMs sur $TOTAL_VMS ont rencontré une ERREUR. $SUCCESS_COUNT VMs mises à jour. Redémarrées : $REBOOT_LIST"
    PRIORITY=8
elif [ -n "$REBOOT_LIST" ]; then
    TITLE="⚠️ VMs Update Succès & Redémarrage(s)"
    MESSAGE="$SUCCESS_COUNT VMs mises à jour. Redémarrage effectué sur : $REBOOT_LIST"
    PRIORITY=6
else
    TITLE="✅ VMs Update SUCCÈS sur $HOSTNAME"
    MESSAGE="$SUCCESS_COUNT VMs mises à jour. Aucune VM n'a nécessité de redémarrage."
    PRIORITY=4
fi

send_gotify_notification "$TITLE" "$MESSAGE" $PRIORITY

exec 1>&- 2>&-
exit 0
```

### Étape 2 : rendre le script exécutable

```bash
chmod +x /usr/local/bin/update_vms.sh
```

-----

## III. Script pour les conteneurs (LXC)

### Étape 1 : création du script `update_lxcs.sh`

1.  Créez le fichier de script sur l'hôte Proxmox :
    ```bash
    nano /usr/local/bin/update_lxcs.sh
    ```
2.  Collez le contenu suivant. **⚠️ Remplacez `VOTRE_URL_GOTIFY` et `VOTRE_TOKEN_GOTIFY` par vos propres valeurs.**

<!-- end list -->

```bash
#!/bin/bash
#
# SCRIPT : update_lxcs.sh
# OBJECTIF : Mettre à jour tous les conteneurs LXC Debian/Ubuntu en cours d'exécution
#
# ==============================================================================

# --- PARAMÈTRES DE GOTIFY ---
GOTIFY_URL="VOTRE_URL_GOTIFY"
GOTIFY_TOKEN="VOTRE_TOKEN_GOTIFY"

# --- PARAMÈTRES DU SCRIPT ---
LOGFILE="/var/log/update_lxcs_cron.log"
EXCLUDED_CTIDS="" # CTIDs à exclure (séparés par des espaces)
SUCCESS_COUNT=0
FAILURE_COUNT=0
REBOOT_LIST=""

exec 1>>$LOGFILE 2>&1

# --- FONCTION DE NOTIFICATION GOTIFY (MÉTHODE FORM-DATA) ---
send_gotify_notification() {
    local title="$1"
    local message="$2"
    local priority="$3"
    curl -k -s -X POST "$GOTIFY_URL/message?token=$GOTIFY_TOKEN" \
        -F "title=$title" \
        -F "message=$message" \
        -F "priority=$priority" > /dev/null 2>&1
}

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
        echo "    [SKIP] Conteneur $CTID exclu."
        continue 
    fi
    
    echo "--> Traitement du conteneur CTID $CTID..."
    
    # 2. Exécution des mises à jour
    echo "    - Exécution des mises à jour..."
    /usr/sbin/pct exec $CTID -- bash -c "$UPDATE_COMMAND"
    
    if [ $? -ne 0 ]; then
        echo "    [ERREUR CRITIQUE] La mise à jour du conteneur $CTID a échoué. Poursuite vers le prochain LXC."
        FAILURE_COUNT=$((FAILURE_COUNT + 1))
        continue
    fi
    
    # 3. Vérification du besoin de redémarrage
    echo "    - Vérification du besoin de redémarrage..."
    REBOOT_CHECK_OUTPUT=$(/usr/sbin/pct exec $CTID -- bash -c "$REBOOT_CHECK_COMMAND")
    
    if [[ "$REBOOT_CHECK_OUTPUT" == *"REBOOT_YES"* ]]; then

        echo "    [ALERTE] Redémarrage nécessaire pour le conteneur $CTID. Redémarrage en cours..."
        REBOOT_LIST="$REBOOT_LIST CTID $CTID"
        
        # 4. Redémarrage direct (pct reboot)
        /usr/sbin/pct reboot $CTID --timeout 120 

        echo "    [OK] Redémarrage du conteneur $CTID terminé."

    else
        echo "    [OK] Conteneur $CTID mis à jour avec succès. Aucun redémarrage critique nécessaire."
    fi
    SUCCESS_COUNT=$((SUCCESS_COUNT + 1))
done

echo "=================================================="
echo "Fin de la mise à jour des LXC le $(date)"
echo "=================================================="

# --- ENVOI DE LA NOTIFICATION FINALE ---
TOTAL_CT=$((SUCCESS_COUNT + FAILURE_COUNT))

if [ $FAILURE_COUNT -gt 0 ]; then
    TITLE="❌ LXC Update ÉCHEC(s) sur $HOSTNAME"
    MESSAGE="$FAILURE_COUNT LXC sur $TOTAL_CT ont rencontré une ERREUR. $SUCCESS_COUNT LXC mis à jour. Redémarrés : $REBOOT_LIST"
    PRIORITY=8
elif [ -n "$REBOOT_LIST" ]; then
    TITLE="⚠️ LXC Update Succès & Redémarrage(s)"
    MESSAGE="$SUCCESS_COUNT LXC mis à jour. Redémarrage effectué sur : $REBOOT_LIST"
    PRIORITY=6
else
    TITLE="✅ LXC Update SUCCÈS sur $HOSTNAME"
    MESSAGE="$SUCCESS_COUNT LXC mis à jour. Aucun LXC n'a nécessité de redémarrage."
    PRIORITY=4
fi

send_gotify_notification "$TITLE" "$MESSAGE" $PRIORITY

exec 1>&- 2>&-
exit 0
```

### Étape 2 : rendre le script exécutable

```bash
chmod +x /usr/local/bin/update_lxcs.sh
```

-----

## IV. Planification avec Cron 📅

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
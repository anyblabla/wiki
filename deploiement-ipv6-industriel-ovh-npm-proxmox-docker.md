---
title: Déploiement ipv6 industriel (OVH → NPM → Proxmox → LXC Docker)
description: Guide complet pour déployer l'IPv6 de bout en bout : de la zone DNS OVH aux conteneurs Docker sous Proxmox, en passant par Nginx Proxy Manager. Inclus : méthodes manuelles et scripts d'automatisation.
published: true
date: 2026-04-30T21:23:50.254Z
tags: docker, lxc, proxmox, npm, automatisation, ipv6, ovh, réseau
editor: markdown
dateCreated: 2026-04-30T21:23:50.254Z
---

Ce guide s’adresse aux administrateurs système et passionnés d’auto‑hébergement utilisant **Proxmox VE** avec des conteneurs **LXC (Debian)** hébergeant **Docker**. Bien que les principes soient universels, ce tutoriel est illustré pour les utilisateurs exploitant **Nginx Proxy Manager (NPM)** comme reverse proxy.

### Mise en situation
Vous possédez un nom de domaine chez **OVH**, un routeur domestique compatible IPv6, et vous centralisez vos flux via **Nginx Proxy Manager (NPM)**. L’objectif est d’acheminer proprement l’IPv6 depuis votre fournisseur jusqu’au cœur de vos applications conteneurisées (comme **SearXNG**). Cela modernise votre stack, améliore la réputation de votre IP et évite les blocages liés aux limitations de l’IPv4.

---

## 1. Récupérer et diffuser son IPv6
La première étape consiste à rendre votre infrastructure visible sur le réseau mondial IPv6.

### Trouver son IP publique
* **Routeur grand public (Box, TP‑Link, etc.) :** Connectez‑vous à l’interface d’administration. Cherchez la section « Statut » ou « Réseau IPv6 » et notez l’adresse **WAN**.
* **Système Linux (passerelle ou Proxmox) :** Exécutez :
    ```bash
    ip -6 addr show | grep "scope global"
    ```
    > **Note :** L’adresse doit commencer par un chiffre (ex. `2001:...`). Ignorez les adresses `fe80::` (locales uniquement).

### Configuration DNS (OVH)
Dans votre espace client **OVH**, créez un enregistrement **AAAA** pour vos sous‑domaines, pointant vers votre adresse IPv6 globale.

---

## 2. Configurer NPM et les pare‑feux
Pour que le trafic IPv6 atteigne vos services, ouvrez les accès à trois niveaux :

* **Pare‑feu du routeur :** Autorisez les ports **80** (HTTP) et **443** (HTTPS) vers l’IP de votre hôte Proxmox.
* **Pare‑feu Proxmox VE (hôte) :** Si le pare‑feu est activé au niveau du Datacenter ou du nœud, autorisez ces ports vers le conteneur hébergeant **Nginx Proxy Manager**.
* **Pare‑feu du conteneur NPM (LXC) :** Si le pare‑feu est activé, assurez‑vous que les ports 80/443 sont autorisés en entrée.

---

## 3. Configurer les invités Proxmox (LXC)
Chaque conteneur doit obtenir une adresse IPv6 via **SLAAC** (Stateless Address Autoconfiguration).

### Méthode manuelle (1 à 2 LXC)
1. Sélectionnez votre conteneur dans Proxmox.
2. Ouvrez l’onglet **Network**.
3. Sélectionnez l’interface (souvent `eth0`) puis **Edit**.
4. Dans **IPv6**, cochez **SLAAC**.
5. Redémarrez le conteneur.

### Méthode automatisée (parc important)
Le script suivant applique automatiquement SLAAC à tous vos LXC.

**Script :**
```bash
for vmid in $(pct list | awk '{if(NR>1) print $1}'); do
  echo "Passage en IPv6 SLAAC pour le LXC $vmid..."
  pct set $vmid -net0 name=eth0,ip6=slaac
done
```

---

## 4. Activer l’IPv6 dans Docker
Docker ne gère pas l’IPv6 par défaut. Il faut lui attribuer une plage interne.

### Méthode manuelle (dans chaque LXC)
1. Ouvrez un shell dans le conteneur.
2. Modifiez : `/etc/docker/daemon.json`
3. Ajoutez :
    ```json
    {
      "log-driver": "journald",
      "ipv6": true,
      "fixed-cidr-v6": "fd00:d0ca:1::/64"
    }
    ```
4. Redémarrez Docker :  
   `systemctl restart docker`

---

## 5. Maintenance massive : le script industriel
Ce script est destiné aux environnements avec de nombreux conteneurs Docker.

🛑 **Avertissement sur le nommage :** Le script cible uniquement les LXC dont le nom contient **« docker- »**.

⚠️ **Attention :** Le fichier `daemon.json` sera **entièrement écrasé**.

### Script industriel
```bash
#!/bin/bash

# Boucle sur les LXC nommés "docker-..." (modifier si nécessaire)
for vmid in $(pct list | grep "docker-" | awk '{print $1}'); do
    status=$(pct status $vmid | awk '{print $2}')
    echo "Traitement du LXC $vmid (Statut : $status)"

    # Allumage temporaire si le conteneur est stoppé
    if [ "$status" == "stopped" ]; then
        echo "Démarrage temporaire de $vmid..."
        pct start $vmid && sleep 5
    fi

    # Injection du fichier daemon.json (ÉCRASE LE FICHIER EXISTANT)
    pct exec $vmid -- bash -c "mkdir -p /etc/docker && cat <<EOF> /etc/docker/daemon.json
{
  \"log-driver\": \"journald\",
  \"ipv6\": true,
  \"fixed-cidr-v6\": \"fd00:d0ca:1::/64\"
}
EOF"

    # Remise dans l'état initial
    if [ "$status" == "stopped" ]; then
        echo "Arrêt de $vmid..."
        pct stop $vmid
    else
        echo "Redémarrage de Docker dans $vmid..."
        pct exec $vmid -- systemctl restart docker
    fi
done
```

💡 **Clusters Proxmox :** Exécutez ce script sur chaque nœud.

---

### Cas concret : pourquoi l’IPv6 ?
Exemple avec **SearXNG** : une connectivité IPv6 native réduit drastiquement les Captchas et blocages, car vos requêtes ne passent plus par des adresses IPv4 partagées.

---

> **Auteur :** Guide proposé par **Amaury aka BlablaLinux**.  
> Retrouvez mes services sur : [https://blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/)

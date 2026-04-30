---
title: Déploiement ipv6 industriel (OVH → NPM → Proxmox → LXC Docker)
description: Guide complet pour déployer l'IPv6 de bout en bout : de la zone DNS OVH aux conteneurs Docker sous Proxmox, en passant par Nginx Proxy Manager. Inclus : méthodes manuelles et scripts d'automatisation.
published: true
date: 2026-04-30T22:00:53.162Z
tags: docker, lxc, proxmox, npm, automatisation, ipv6, ovh, réseau
editor: markdown
dateCreated: 2026-04-30T21:23:50.254Z
---

## Pré‑requis
Avant de commencer, assurez‑vous de disposer de :

- un routeur compatible IPv6  
- un /64 ou /56 fourni par votre opérateur  
- un nom de domaine chez OVH  
- Proxmox VE installé  
- des conteneurs LXC Debian  
- Docker installé dans les LXC  
- Nginx Proxy Manager (NPM)  
- un réseau local où les Router Advertisements (RA) sont activés  

> **Note importante :** SLAAC **ne fonctionne pas** si votre routeur ne diffuse pas de RA (Router Advertisements). Vérifiez ce point si vos LXC ne reçoivent aucune IPv6.

---

## Schéma global

```text
OVH DNS AAAA
      ↓
Routeur IPv6 (RA activés)
      ↓
Proxmox VE
      ↓
LXC Debian
      ↓
Docker (IPv6 activé)
      ↓
Nginx Proxy Manager
      ↓
Internet (IPv6 natif)
```

---

## Mise en situation
Vous possédez un nom de domaine chez **OVH**, un routeur domestique compatible IPv6, et vous centralisez vos flux via **Nginx Proxy Manager (NPM)**. L'objectif est d'acheminer proprement l'IPv6 depuis votre fournisseur jusqu'au cœur de vos applications conteneurisées (comme **SearXNG**). Cela permet de moderniser votre stack, d'améliorer la réputation de votre IP et d'éviter les blocages de services limités en IPv4.

---

## 1. Récupérer et diffuser son ipv6
La première étape est de rendre votre infrastructure visible sur le réseau mondial IPv6.

### Trouver son ip publique
* **Sur un routeur grand public (Box, TP-Link, etc.) :** Connectez-vous à l'interface d'administration du routeur. Cherchez la section "Statut" ou "Réseau IPv6" et notez l'adresse **WAN**.
* **Sur un système Linux (Passerelle ou Proxmox) :** Tapez cette commande dans le shell :
    ```bash
    ip -6 addr show | grep "scope global"
    ```
    > **Note :** L'adresse doit commencer par un chiffre (ex: `2001:...`). Ignorez les adresses commençant par `fe80::` (local uniquement).

### Exemple de sortie
```text
inet6 2001:db8:abcd:42::1234/64 scope global dynamic
```

### Configuration DNS (OVH)
Dans votre espace client **OVH** (Zone DNS), créez un enregistrement de type **AAAA** pour vos sous-domaines, pointant vers cette adresse IPv6 globale.

---

## 2. Configuration de NPM et des pare-feux
Pour que le trafic IPv6 atteigne vos services, vous devez ouvrir les accès à trois niveaux distincts :

* **Pare-feu du routeur :** Autorisez les ports **80** (HTTP) et **443** (HTTPS) en IPv6 vers l'IP de votre hôte Proxmox.
* **Pare-feu de Proxmox VE (niveau hôte) :** Si le pare-feu est activé au niveau du Datacenter ou du Nœud, créez une règle autorisant ces ports vers le conteneur accueillant **Nginx Proxy Manager**.
* **Pare-feu de l'invité NPM (niveau LXC) :** Dans l'interface de Proxmox, sélectionnez votre **LXC NPM > Firewall**. **Si vous avez activé le pare-feu sur ce conteneur**, assurez-vous que les ports 80/443 sont explicitement autorisés en entrée.

---

## 3. Configuration des invités Proxmox (LXC)
Chaque conteneur doit obtenir une IP via le mode **SLAAC** (Stateless Address Autoconfiguration).

### Méthode manuelle (pour 1 ou 2 LXC)
1. Dans l'interface Web de Proxmox, sélectionnez votre conteneur.
2. Allez dans l'onglet **Network**.
3. Sélectionnez l'interface réseau (souvent `eth0`) et cliquez sur **Edit**.
4. Dans la section **IPv6**, cochez la case **SLAAC**.
5. Cliquez sur **OK** et redémarrez le conteneur pour appliquer les changements.

### Exemple de sortie
```text
net0: name=eth0,bridge=vmbr0,ip6=slaac
```

### Méthode automatisée (parc important)
Si vous gérez de nombreux conteneurs, nous allons utiliser un script Bash pour automatiser la tâche. Ce script récupère la liste de tous vos conteneurs présents sur le nœud et force le réglage IPv6 sur "SLAAC".

**Mise en place étape par étape :**

1. Connectez-vous en **SSH** sur votre nœud Proxmox.
2. Créez le fichier du script :
    ```bash
    nano set_slaac.sh
    ```
3. Collez le code suivant dans l'éditeur :
    ```bash
    for vmid in $(pct list | awk '{if(NR>1) print $1}'); do
      echo "Passage en IPv6 SLAAC pour le LXC $vmid..."
      pct set $vmid -net0 name=eth0,ip6=slaac
    done
    ```
4. Sauvegardez et quittez (Faites `Ctrl+O`, puis `Entrée`, puis `Ctrl+X`).
5. Rendez le fichier exécutable par le système :
    ```bash
    chmod +x set_slaac.sh
    ```
6. Lancez le script :
    ```bash
    ./set_slaac.sh
    ```

---

## 4. Activation de l'ipv6 dans le moteur Docker
Docker ne gère pas l'IPv6 par défaut. Il faut lui donner une plage d'adresses internes.

### Méthode manuelle (dans chaque LXC)
1. Entrez dans le shell de votre conteneur.
2. Créez ou modifiez le fichier de configuration :
   ```bash
   nano /etc/docker/daemon.json
   ```
3. Insérez ce contenu :
    ```json
    {
      "log-driver": "journald",
      "ipv6": true,
      "fixed-cidr-v6": "fd00:d0ca:1::/64"
    }
    ```
4. Quittez et sauvegardez, puis redémarrez Docker :
    ```bash
    systemctl restart docker
    ```

---

## 5. Maintenance massive : le script industriel
Ce script est indispensable si vous avez un grand parc de conteneurs Docker.

🛑 **Avertissement sur le nommage** : Ce script est configuré pour cibler uniquement les conteneurs dont le nom contient **"docker-"**. Si vos LXC ne suivent pas cette nomenclature, le script ne les trouvera pas. 

⚠️ **Attention à l'écrasement** : Le script utilise la commande `cat` pour injecter la configuration. Cela signifie que **le fichier `daemon.json` existant dans chaque LXC sera totalement écrasé**. Si vous avez des configurations Docker spécifiques, vous devez les intégrer au script avant de le lancer.

**Ce que fait ce script :**  
Ce script parcourt tous les conteneurs LXC dont le nom contient « docker- », démarre temporairement ceux qui sont arrêtés, injecte un fichier `daemon.json` configurant l’IPv6 pour Docker, redémarre Docker ou éteint le conteneur selon son état initial, puis passe au suivant.

**Mise en place :**
1. Connectez-vous en **SSH** sur votre nœud Proxmox.
2. Créez le fichier :
    ```bash
    nano update_docker_ipv6.sh
    ```
3. Collez le code suivant :
    ```bash
    #!/bin/bash

    # Boucle sur les LXC nommés "docker-..." (A modifier si votre préfixe est différent)
    for vmid in $(pct list | grep "docker-" | awk '{print $1}'); do
        status=$(pct status $vmid | awk '{print $2}')
        echo "Traitement du LXC $vmid (Statut : $status)"

        # Allumage temporaire si le conteneur est stoppé
        if [ "$status" == "stopped" ]; then
            echo "Démarrage temporaire de $vmid..."
            pct start $vmid && sleep 5
        fi

        # Injection du fichier daemon.json (ÉCRASE LE FICHIER EXISTANT)
        pct exec $vmid -- bash -c "cat << 'EOF' > /etc/docker/daemon.json
    {
      \"log-driver\": \"journald\",
      \"ipv6\": true,
      \"fixed-cidr-v6\": \"fd00:d0ca:1::/64\"
    }
    EOF"

        # Remise dans l'état initial (Stop ou Restart Docker)
        if [ "$status" == "stopped" ]; then
            echo "Arrêt de $vmid..."
            pct stop $vmid
        else
            echo "Redémarrage de Docker dans $vmid..."
            pct exec $vmid -- systemctl restart docker
        fi
    done
    ```
4. Rendez-le exécutable :
    ```bash
    chmod +x update_docker_ipv6.sh
    ```
5. Lancez-le :
    ```bash
    ./update_docker_ipv6.sh
    ```

💡 **Note importante pour les clusters :** Si vos conteneurs sont répartis sur plusieurs serveurs, vous devez **répéter cette mise en place et exécuter le script sur chaque nœud** de votre cluster Proxmox.

---

## Sécurité IPv6
L’IPv6 expose chaque machine directement sur Internet.  
Contrairement à l’IPv4 NAT, il n’y a **aucune translation d’adresse** pour “cacher” vos machines.

Recommandations :

- activez les pare‑feux Proxmox et LXC  
- n’exposez que les ports nécessaires  
- surveillez les logs (`journalctl -u docker`, `dmesg`, `pve-firewall status`)  
- évitez les conteneurs Docker en mode `--network host` en IPv6  

---

## Dépannage IPv6

### Vérifier la connectivité sortante
```bash
curl -6 ifconfig.co
```

### Tester un ping IPv6
```bash
ping6 google.com
```

### Vérifier les routes
```bash
ip -6 route
```

### Vérifier les Router Advertisements
```bash
tcpdump -i eth0 icmp6
```

### Tester Docker en IPv6
```bash
docker run --rm busybox ping6 google.com
```

---

## Conclusion
Vous disposez maintenant d’une **stack IPv6 complète**, depuis OVH jusqu’aux conteneurs Docker, en passant par Proxmox et Nginx Proxy Manager.  
Votre infrastructure est modernisée, plus fiable, et vos services comme **SearXNG** bénéficient d’une meilleure réputation réseau.

![2026-04-30_00.07.58_cryptcheck.fr_510e702f01e8-flou.png](/deploiement-ipv6-industriel-ovh-npm-proxmox-docker/2026-04-30_00.07.58_cryptcheck.fr_510e702f01e8-flou.png)

---

### Cas concret : pourquoi l'ipv6 ?
Prenez **SearXNG** : en disposant d'une connectivité IPv6 native, vos requêtes sortantes vers les moteurs de recherche ne sont plus pénalisées par les limitations des adresses IPv4 partagées. Cela réduit drastiquement les Captchas et les blocages, rendant votre service beaucoup plus fiable.

---

> **Auteur** : ce guide est proposé par **Amaury aka BlablaLinux**. Retrouvez l'ensemble de mes services sur [https://blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).

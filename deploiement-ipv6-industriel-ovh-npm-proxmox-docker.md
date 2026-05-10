---
title: Déploiement ipv6 industriel (OVH → NPM → Proxmox → LXC Docker)
description: Guide complet pour déployer l'IPv6 de bout en bout : de la zone DNS OVH aux conteneurs Docker sous Proxmox, en passant par Nginx Proxy Manager. Inclus : méthodes manuelles et scripts d'automatisation.
published: true
date: 2026-05-10T15:01:23.261Z
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

> **Important :** SLAAC **ne fonctionne pas** si votre routeur ne diffuse pas de RA.  
> Si vos LXC ne reçoivent aucune IPv6, vérifiez ce point en premier.

---

# Schéma global

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

# 1. Récupérer et diffuser son IPv6

## Trouver son IPv6 publique

### Sur un routeur grand public  
Consultez la section « Statut » ou « Réseau IPv6 » et notez l’adresse **WAN IPv6**.

### Sur un système Linux (Proxmox ou passerelle)

```bash
ip -6 addr show | grep "scope global"
```

Ignorez les adresses commençant par `fe80::` (locales uniquement).

### Exemple

```
inet6 2001:db8:abcd:42::1234/64 scope global dynamic
```

## Ajouter l’enregistrement DNS AAAA (OVH)

Dans **OVH → Zone DNS**, créez un enregistrement **AAAA** pointant vers votre IPv6 publique.

---

# 2. Configuration de NPM et des pare‑feux

L’IPv6 expose directement vos machines sur Internet.  
Il est donc essentiel de **configurer les règles AVANT d’activer les pare‑feux**.

L’ordre recommandé :

1. Pare‑feu du routeur  
2. Groupes de sécurité Proxmox  
3. Règles globales du Centre de données  
4. Options du pare‑feu du Centre de données  
5. Options du pare‑feu du nœud  
6. Règles du conteneur NPM  
7. Activation finale

---

## 2.1. Pare‑feu du routeur

Autorisez en IPv6 vers l’adresse du conteneur NPM :

- port **80/tcp** (HTTP)  
- port **443/tcp** (HTTPS)

---

## 2.2. Groupes de sécurité Proxmox

Chemin :

**Centre de données → Pare‑feu → Groupe de sécurité**

Créez les groupes suivants.

### Groupe « ipv6 »

Ajoutez :

- une règle autorisant `ipv6-icmp` (ping IPv6, RA, ND, etc.)  
- une règle autorisant le trafic `ipv6` général

### Groupe « ssh »

Ajoutez :

- une règle utilisant la macro **SSH**

### Groupe « web »

Ajoutez :

- une règle utilisant la macro **HTTPS**  
- une règle utilisant la macro **HTTP**

![ipv6-pare-feu-pve.png](/deploiement-ipv6-industriel-ovh-npm-proxmox-docker/ipv6-pare-feu-pve.png)

---

## 2.3. Règles globales du Centre de données

Chemin :

**Centre de données → Pare‑feu → Ajouter**

Ajoutez les règles suivantes :

- appliquer le groupe **ssh**  
- appliquer le groupe **ipv6**  
- autoriser ICMP (ping IPv4)  
- autoriser le port TCP 25  
- autoriser le port TCP 3128  
- autoriser les ports TCP 5900 à 5999  
- autoriser le port TCP 8008  
- autoriser le port TCP 8006 (interface Web Proxmox)  
- autoriser le port UDP 111  
- autoriser les ports UDP 5405 à 5412  
- autoriser les ports TCP 60000 à 60050

![ipv6-pare-feu-pve-02.png](/deploiement-ipv6-industriel-ovh-npm-proxmox-docker/ipv6-pare-feu-pve-02.png)

---

## 2.4. Options du pare‑feu du Centre de données

Chemin :

**Centre de données → Pare‑feu → Options**

Vérifiez :

- Pare‑feu : Oui  
- Journaux : Oui  
- Politique d’entrée : **DROP**  
- Politique de sortie : ACCEPT  
- Politique de transfert : ACCEPT  

---

## 2.5. Options du pare‑feu du nœud Proxmox

Chemin :

**Votre nœud → Pare‑feu → Options**

Vérifiez :

- Pare‑feu : Oui  
- Filtrer les paquets SMURF : Oui  
- Filtrer les flags TCP : Non  
- NDP : Oui  
- Filtre MAC : Oui  
- Filtre IP : Non  
- log_level_in : info  
- log_level_out : nolog  
- nftables (préversion) : Non  

---

## 2.6. Pare‑feu du conteneur NPM (LXC)

Chemin :

**Votre conteneur NPM → Pare‑feu → Ajouter**

Ajoutez :

- le groupe **ipv6**  
- les ports TCP utilisés par NPM (ex. 81, 7880 si vous les utilisez)  
- le groupe **web** (HTTP + HTTPS)  
- le groupe **ssh** (si vous souhaitez un accès SSH)

![ipv6-pare-feu-lxc.png](/deploiement-ipv6-industriel-ovh-npm-proxmox-docker/ipv6-pare-feu-lxc.png)

Chemin des options :

**Votre conteneur NPM → Pare‑feu → Options**

Vérifiez :

- Pare‑feu : Oui  
- DHCP : Oui  
- NDP : Oui  
- Annonce de routeur : Non  
- Filtre MAC : Oui  
- Politique d’entrée : **DROP**  
- Politique de sortie : ACCEPT  

---

# 3. Configuration IPv6 des conteneurs LXC

## Méthode manuelle

1. Sélectionnez le conteneur  
2. Ouvrez **Réseau**  
3. Éditez l’interface (souvent `eth0`)  
4. Choisissez **SLAAC** en IPv6  
5. Redémarrez le conteneur

Exemple :

```
net0: name=eth0,bridge=vmbr0,ip6=slaac
```

## Méthode automatisée (script)

Créer un script :

```bash
nano set_slaac.sh
```

Contenu :

```bash
for vmid in $(pct list | awk '{if(NR>1) print $1}'); do
  echo "Passage en IPv6 SLAAC pour le LXC $vmid..."
  pct set $vmid -net0 name=eth0,ip6=slaac
done
```

Exécuter :

```bash
chmod +x set_slaac.sh
./set_slaac.sh
```

---

# 4. Activer l’IPv6 dans Docker

Dans chaque LXC :

Créer ou modifier :

```bash
nano /etc/docker/daemon.json
```

Contenu :

```json
{
  "log-driver": "journald",
  "ipv6": true,
  "fixed-cidr-v6": "fd00:d0ca:1::/64",
  "ip6tables": true,
  "experimental": true
}
```

Redémarrer Docker :

```bash
systemctl restart docker
```

---

# 5. Script massif pour Docker

Créer :

```bash
nano update_docker_ipv6.sh
```

Contenu :

```bash
#!/bin/bash
for vmid in $(pct list | grep "docker-" | awk '{print $1}'); do
    status=$(pct status $vmid | awk '{print $2}')
    echo "Traitement du LXC $vmid (Statut : $status)"

    if [ "$status" == "stopped" ]; then
        pct start $vmid && sleep 5
    fi

    pct exec $vmid -- bash -c "cat << 'EOF' > /etc/docker/daemon.json
{
  \"log-driver\": \"journald\",
  \"ipv6\": true,
  \"fixed-cidr-v6\": \"fd00:d0ca:1::/64\",
  \"ip6tables\": true,
  \"experimental\": true
}
EOF"

    if [ "$status" == "stopped" ]; then
        pct stop $vmid
    else
        pct exec $vmid -- systemctl restart docker
    fi
done
```

Exécuter :

```bash
chmod +x update_docker_ipv6.sh
./update_docker_ipv6.sh
```

---

# Sécurité IPv6

- activez les pare‑feux Proxmox et LXC  
- n’exposez que les ports nécessaires  
- surveillez les logs (`journalctl`, `pve-firewall status`)  
- évitez Docker en `--network host` en IPv6  

---

# Dépannage IPv6

Tester la sortie :

```bash
curl -6 ifconfig.co
```

Ping IPv6 :

```bash
ping6 google.com
```

Routes :

```bash
ip -6 route
```

RA :

```bash
tcpdump -i eth0 icmp6
```

Docker :

```bash
docker run --rm busybox ping6 google.com
```

---

# Conclusion

Votre infrastructure dispose maintenant d’une **chaîne IPv6 complète**, depuis OVH jusqu’aux conteneurs Docker, en passant par Proxmox et Nginx Proxy Manager.

![2026-04-30_00.07.58_cryptcheck.fr_510e702f01e8-flou.png](/deploiement-ipv6-industriel-ovh-npm-proxmox-docker/2026-04-30_00.07.58_cryptcheck.fr_510e702f01e8-flou.png)
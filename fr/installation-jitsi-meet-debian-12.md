---
title: Installation de Jitsi Meet sur Debian 12 (Bookworm)
description: >-
  Installez Jitsi Meet sur Debian 12 (Bookworm). Guide complet : dépôts, FQDN,
  SSL (Let's Encrypt ou Reverse Proxy), NAT (IP LAN/WAN) et optimisations de
  performance pour des visioconférences sécurisées.
published: true
date: '2026-11-02T23:46:21.245Z'
tags:
  - serveur
  - nginx
  - proxy
  - debian
  - jitsi
  - meet
  - prosody
  - nat
  - videobridge
editor: markdown
dateCreated: '2025-11-19T17:54:18.165Z'
---

aCette page explique comment installer et configurer la plateforme de visioconférence **Jitsi Meet** sur un serveur Debian 12 (Bookworm) en utilisant les dépôts officiels.

-----

## 1\. Prérequis 🛠️

  * **Système d'exploitation :** Debian 12 (Bookworm).
  * **Accès root/sudo :** Un utilisateur avec des privilèges `sudo`.
  * **Nom de domaine :** Un nom de domaine entièrement qualifié (FQDN).
  * **Ressources :** Minimum **2 Go de RAM** (4 Go recommandés).
  * **Ports requis :**
      * **TCP 80** et **TCP 443** (Web/SSL).
      * **UDP 10000** (Transport des médias).
      * **TCP 5222** (XMPP/Signalisation).
      * **TCP 4443** (Fallback pour les médias).

-----

## 2\. Configuration du FQDN et des paquets de base 🌐

### a. Mettre à jour le système et installer les outils de base

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y curl wget apt-transport-https gnupg2 nano
```

### b. Configurer le nom d'hôte

Définissez le nom d'hôte de votre serveur. Remplacez `meet.mondomaine.fr` par votre FQDN.

```bash
sudo hostnamectl set-hostname meet.mondomaine.fr
```

-----

## 3\. Installation du JDK (Java development kit) ☕

Jitsi Videobridge nécessite Java 11 ou supérieur.

```bash
sudo apt install -y openjdk-17-jdk
java -version
```

-----

## 4\. Ajout des dépôts Jitsi 📦

### a. Importer la clé GPG du dépôt

```bash
curl https://download.jitsi.org/jitsi-key.gpg.key | gpg --dearmor | sudo tee /usr/share/keyrings/jitsi-keyring.gpg > /dev/null
```

### b. Ajouter le dépôt Jitsi

```bash
echo "deb [signed-by=/usr/share/keyrings/jitsi-keyring.gpg] https://download.jitsi.org stable/" | sudo tee /etc/apt/sources.list.d/jitsi-stable.list
```

### c. Mettre à jour la liste des paquets

```bash
sudo apt update
```

-----

## 5\. Installation de Jitsi Meet 🚀

Installez le méta-paquet `jitsi-meet`.

```bash
sudo apt install jitsi-meet -y
```

Lors de l'installation, entrez votre **FQDN** (ex: `meet.mondomaine.fr`).

-----

## 6\. Gestion du certificat SSL 🔒 (Choisir une seule option)

Choisissez l'option qui correspond à la manière dont votre serveur est exposé à Internet.

### Option A : Installation directe du certificat Let's Encrypt (Exposition directe)

Utilisez cette option si votre serveur Jitsi a une adresse IP publique et que les ports **TCP 80 et 443** sont directement ouverts.

Exécutez le script d'installation automatique :

```bash
sudo /usr/share/jitsi-meet/scripts/install-letsencrypt-cert.sh
```

Entrez votre **adresse e-mail**.

### Option B : Configuration via reverse proxy (NPM, Traefik, etc.)

Utilisez cette option si un **reverse proxy** gère déjà le certificat SSL (typiquement derrière un NAT).

1.  **Ignorer le script Let's Encrypt** (voir Option A).
2.  **Configuration du reverse proxy :** Configurez votre proxy pour rediriger le trafic **HTTPS** (port 443) vers l'adresse IP privée de votre serveur Jitsi sur le port **443**.
3.  Activez l'option **"WebSockets Support"** dans votre reverse proxy.

-----

## 7\. Configuration NAT avancée (serveur derrière un routeur) 📡

Si votre serveur Jitsi a une **adresse IP privée** et que le trafic est redirigé via un routeur (NAT), vous devez indiquer au JVB ses adresses IP LAN et WAN pour le routage média (ICE/STUN). Cette étape est **nécessaire** dans le cas de l'**Option B du point 6**.

### Option A : Méthode moderne (Fichier `jvb.conf`)

1.  **Éditer le fichier de configuration JVB** :

    ```bash
    sudo nano /etc/jitsi/videobridge/jvb.conf
    ```

2.  **Ajouter la section de mappage statique** :

    ```hocon
    ice4j {
      harvest {
        mapping {
          static-mappings = [
            {
              local-address = "<IP.LAN.Locale>"
              public-address = "<IP.WAN.Publique>"
            }
          ]
        }
      }
    }
    ```

### Option B : Méthode classique (Fichier `sip-communicator.properties`)

1.  **Éditer le fichier de propriétés** :

<!-- end list -->

```bash
sudo nano /etc/jitsi/videobridge/sip-communicator.properties
```

2.  **Ajouter les adresses** :

<!-- end list -->

```properties
org.ice4j.ice.harvest.NAT_HARVESTER_LOCAL_ADDRESS=<IP.LAN.Locale>
org.ice4j.ice.harvest.NAT_HARVESTER_PUBLIC_ADDRESS=<IP.WAN.Publique>
```

3.  **Redémarrer Jitsi Videobridge (pour les deux options)** :

<!-- end list -->

```bash
sudo systemctl restart jitsi-videobridge2
```

-----

## 8\. Sécurisation : authentification requise pour la création de salle 🛡️

Restreindre la création de salles aux utilisateurs enregistrés via Prosody (XMPP).

### a. Configuration de Jicofo

```bash
sudo nano /etc/jitsi/jicofo/sip-communicator.properties
# Ajouter : org.jitsi.jicofo.auth.URL=XMPP:meet.mondomaine.fr
```

### b. Configuration de Prosody (XMPP)

```bash
sudo nano /etc/prosody/conf.d/meet.mondomaine.fr.cfg.lua
# Modifier VirtualHost principal à "internal_plain"
# Ajouter VirtualHost "guest.meet.mondomaine.fr" à "anonymous"
```

### c. Redémarrer les services et créer un utilisateur

```bash
sudo systemctl restart prosody jicofo jitsi-videobridge2
sudo prosodyctl register utilisateur_test meet.mondomaine.fr mot_de_passe_secret
```

-----

## 9\. Configuration du pare-feu (UFW) 🧱

Sécurisez l'accès sur le serveur Jitsi.

### a. Installation et règles

```bash
sudo apt install ufw -y
# Autoriser SSH
sudo ufw allow ssh

# Ports Jitsi Meet requis pour le trafic
sudo ufw allow 443/tcp     # Communication web ou communication proxy
sudo ufw allow 10000/udp   # Média principal (NAT requis si derrière un routeur)
sudo ufw allow 5222/tcp    # Signalisation XMPP
sudo ufw allow 4443/tcp    # Média Fallback (NAT requis si derrière un routeur)

# Si Option A (Exposition directe) : Autoriser le port 80 pour Let's Encrypt
# sudo ufw allow 80/tcp
```

### b. Activer UFW

```bash
sudo ufw enable
```

-----

## 10\. Optimisations de performance (tuning) ⚡

### a. Optimisation des tampons du noyau

```bash
sudo nano /etc/sysctl.conf
# Ajouter les lignes net.core.rmem_max, wmem_max, somaxconn
sudo sysctl -p
```

### b. Ajustement du gestionnaire de processus Jitsi (Jitsi Videobridge)

```bash
sudo nano /etc/default/jitsi-videobridge2
# Modifier JAVA_OPTS (ex: -Xms256m -Xmx4096m -XX:+UseG1GC -XX:+HeapDumpOnOutOfMemoryError)
sudo systemctl restart jitsi-videobridge2
```

-----

## 11\. Vérification et accès ✅

### Vérifier l'état des services

```bash
sudo systemctl status jitsi-videobridge2 jicofo nginx
```

### Accéder à Jitsi Meet

Votre Jitsi Meet est accessible via `https://meet.mondomaine.fr`.

-----

## 12\. Dépannage courant 🩺

### Problèmes de connectivité vidéo/audio (pas d'image, écran noir)

La grande majorité des problèmes de Jitsi Meet sont liés à la gestion du trafic média, qui utilise le protocole **UDP 10000**.

  * **Vérifiez le pare-feu :** Assurez-vous que les ports **UDP 10000**, **TCP 4443** et **TCP 5222** sont ouverts sur le serveur Jitsi lui-même (`sudo ufw status verbose`).
  * **Vérifiez le NAT/Redirection de ports :** Si le serveur est derrière un routeur, il est **essentiel** que le trafic **UDP 10000** et **TCP 4443** soit correctement redirigé (port forwarding) vers l'adresse IP privée de votre serveur Jitsi.
  * **Vérifiez la configuration NAT (Section 7) :** Confirmez que vous avez renseigné les adresses IP **LAN et WAN** dans la configuration de JVB si le serveur est en environnement NAT.

### Problèmes de certificat SSL

Si le navigateur signale une erreur de sécurité ou un certificat non valide.

  * **Option A (Directe) :** Assurez-vous que le **port 80** était ouvert lors de l'exécution du script Let's Encrypt et que votre FQDN pointe correctement vers le serveur.
  * **Option B (Reverse Proxy) :** Le Reverse Proxy doit être configuré pour parler en **HTTPS** au serveur Jitsi (port 443) et le Reverse Proxy doit détenir un certificat valide et public.

### Problèmes d'authentification

Si le bouton "Start meeting" est remplacé par "Login" et que la connexion échoue.

  * **Vérifiez l'utilisateur :** Confirmez que vous avez créé l'utilisateur dans Prosody (Section **8.c**) et que vous utilisez les bonnes informations d'identification.
  * **Vérifiez les services :** Assurez-vous que les services `prosody` et `jicofo` sont actifs et ont été redémarrés après les modifications de configuration (Section **8.c**).

### Logs et journaux

Pour un diagnostic plus avancé, consultez les journaux des composants principaux :

  * **Jitsi Videobridge (Média) :** `sudo journalctl -u jitsi-videobridge2`
  * **Jicofo (Signalisation) :** `sudo journalctl -u jicofo`
  * **Prosody (Authentification/XMPP) :** `sudo journalctl -u prosody`

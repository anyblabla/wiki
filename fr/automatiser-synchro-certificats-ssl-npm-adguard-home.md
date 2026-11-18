---
title: Synchroniser les certificats SSL de NPM vers AdGuard Home
description: >-
  Guide étape par étape pour automatiser la copie et le renouvellement des
  certificats SSL Let's Encrypt depuis Nginx Proxy Manager (Docker) vers une ou
  plusieurs instances AdGuard Home via SSH et Cron.
published: true
date: '2026-11-18T18:36:51.022Z'
tags:
  - npm
  - nginx proxy manager
  - adguard
  - docker
  - ssl
  - tls
  - lets encrypt
  - certificat
  - linux
  - bash
  - cron
  - ssh
  - automatisation
editor: markdown
dateCreated: '2026-11-18T18:35:47.766Z'
---

Tous les trois mois, lors du renouvellement automatique des certificats Let's Encrypt par Nginx Proxy Manager (NPM), il faut normalement récupérer les nouveaux fichiers numérotés (`fullchain12.pem`, `privkey12.pem`, etc.) et mettre à jour la configuration d'AdGuard Home.

Ce guide explique comment automatiser complètement cette tâche : AdGuard Home pointera vers des chemins de fichiers fixes, et un script vérifiera chaque nuit si le certificat a changé pour le pousser en SSH et redémarrer le service.

---

## Prérequis et architecture

* **Serveur NPM :** Un hôte ou conteneur LXC faisant tourner Nginx Proxy Manager sous Docker, avec le dossier `letsencrypt` monté en local (exemple : `/root/npm/letsencrypt`).
* **Serveurs AdGuard Home :** Une ou plusieurs instances AdGuard Home (ici deux instances en haute disponibilité : `192.168.1.10` et `192.168.1.11`).
* **Accès réseau :** L'hôte NPM doit pouvoir joindre les serveurs AdGuard Home en SSH.

---

## Étape 1 : Repérer le dossier du certificat sur l'hôte NPM

Dans le dossier `live/` de Nginx Proxy Manager, Certbot maintient en permanence des liens symboliques sans numéro (`fullchain.pem` et `privkey.pem`) qui pointent toujours vers la dernière version valide du certificat.

Sur votre machine hébergeant NPM, listez le contenu du dossier `live` :

```bash
ls -la /root/npm/letsencrypt/live/
```

Repérez le nom du sous-dossier correspondant à votre certificat (par exemple `npm-1`). Vérifiez qu'il s'agit bien du bon certificat en affichant sa date d'expiration :

```bash
openssl x509 -in /root/npm/letsencrypt/live/npm-1/fullchain.pem -noout -enddate
```

> **À adapter :** Remplacez `/root/npm/letsencrypt` par le chemin réel de votre volume NPM, et `npm-1` par le nom du dossier affiché chez vous.

---

## Étape 2 : Configurer l'accès SSH sans mot de passe

Pour que le script puisse copier les fichiers sans intervention humaine, générez une paire de clés SSH dédiée sur l'hôte NPM et déployez-la sur chaque instance AdGuard Home.

1. Générez la clé SSH dédiée :

```bash
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_adguard_cert -C "sync-cert-adguard"
```

2. Copiez la clé publique vers vos serveurs AdGuard Home (adaptez les adresses IP) :

```bash
ssh-copy-id -i ~/.ssh/id_adguard_cert.pub root@192.168.1.10
ssh-copy-id -i ~/.ssh/id_adguard_cert.pub root@192.168.1.11
```

3. Testez la connexion directe pour vérifier qu'aucun mot de passe n'est demandé :

```bash
ssh -i ~/.ssh/id_adguard_cert root@192.168.1.10 "hostname" && ssh -i ~/.ssh/id_adguard_cert root@192.168.1.11 "hostname"
```

---

## Étape 3 : Créer le script de synchronisation

Ce script compare l'empreinte SHA256 du certificat actuel avec la dernière empreinte connue. Si le certificat n'a pas changé, le script s'arrête immédiatement. S'il a été renouvelé, il copie les fichiers vers chaque instance AdGuard Home et redémarre le service.

1. Créez le dossier des scripts (s'il n'existe pas) et le fichier `sync-cert-adguard.sh` :

```bash
mkdir -p /root/scripts
cat << 'EOF' > /root/scripts/sync-cert-adguard.sh
#!/bin/bash

# --- VARIABLES À PERSONNALISER ---
CERT_DIR="/root/npm/letsencrypt/live/npm-1"
ADGUARD_HOSTS=("192.168.1.10" "192.168.1.11")
# ---------------------------------

SSH_KEY="/root/.ssh/id_adguard_cert"
DEST_DIR="/root/ssl"
HASH_FILE="/var/tmp/adguard_cert.sha256"

# Calcul de l'empreinte du certificat actuel (suit automatiquement le lien symbolique)
CURRENT_HASH=$(sha256sum "$CERT_DIR/fullchain.pem" | awk '{print $1}')
LAST_HASH=$(cat "$HASH_FILE" 2>/dev/null)

# Si le certificat n'a pas changé ou est introuvable, on arrête là
if [ -z "$CURRENT_HASH" ] || [ "$CURRENT_HASH" == "$LAST_HASH" ]; then
    exit 0
fi

SUCCESS=true

for HOST in "${ADGUARD_HOSTS[@]}"; do
    ssh -i "$SSH_KEY" "root@$HOST" "mkdir -p $DEST_DIR"
    scp -i "$SSH_KEY" -q "$CERT_DIR/fullchain.pem" "$CERT_DIR/privkey.pem" "root@$HOST:$DEST_DIR/"
    
    if [ $? -eq 0 ]; then
        ssh -i "$SSH_KEY" "root@$HOST" "systemctl restart AdGuardHome"
    else
        SUCCESS=false
    fi
done

# Enregistrement de la nouvelle empreinte uniquement si toutes les copies ont réussi
if [ "$SUCCESS" = true ]; then
    echo "$CURRENT_HASH" > "$HASH_FILE"
fi
EOF
```

> **Ce qu'il faut modifier dans ce script :**
> * `CERT_DIR` : Le chemin complet vers le dossier `npm-X` identifié à l'étape 1.
> * `ADGUARD_HOSTS` : La liste des adresses IP (ou noms d'hôtes locaux) de vos serveurs AdGuard Home, séparées par un espace. Si vous n'avez qu'un seul serveur, mettez simplement `ADGUARD_HOSTS=("192.168.1.10")`.
> * *Note :* Si AdGuard Home tourne sous Docker sur le serveur distant, remplacez `systemctl restart AdGuardHome` par `docker restart adguardhome`.

2. Rendez le script exécutable et lancez-le une première fois pour initialiser les fichiers :

```bash
chmod +x /root/scripts/sync-cert-adguard.sh
/root/scripts/sync-cert-adguard.sh
```

3. Vérifiez que les fichiers `fullchain.pem` et `privkey.pem` sont bien présents dans `/root/ssl/` sur vos serveurs AdGuard Home :

```bash
ssh -i ~/.ssh/id_adguard_cert root@192.168.1.10 "ls -lh /root/ssl/"
```

---

## Étape 4 : Configurer les chemins fixes dans AdGuard Home

Il ne reste plus qu'à indiquer à AdGuard Home de lire ces nouveaux emplacements fixes.

1. Connectez-vous à l'interface web d'**AdGuard Home**.
2. Allez dans **Paramètres** > **Paramètres de chiffrement**.
3. Cochez **Définir un emplacement du fichier des certificats** et renseignez :
   ```text
   /root/ssl/fullchain.pem
   ```
4. Cochez **Définir le chemin d'accès au fichier de clé privée** et renseignez :
   ```text
   /root/ssl/privkey.pem
   ```
5. Cliquez sur **Sauvegarder la configuration**.

> **Important :** Si vous utilisez `adguardhome-sync` pour répliquer votre configuration entre deux instances, sachez que les paramètres de chiffrement (TLS) ne sont pas synchronisés. Pensez à appliquer cette modification manuellement sur chaque instance AdGuard Home.

### Nettoyage optionnel

Si vous aviez l'habitude de déposer vos anciens certificats numérotés dans `/root/` sur vos serveurs AdGuard Home, vous pouvez maintenant les supprimer proprement sans toucher au dossier `/root/ssl/` :

```bash
ssh -i ~/.ssh/id_adguard_cert root@192.168.1.10 "rm -f /root/cert*.pem /root/chain*.pem /root/fullchain*.pem /root/privkey*.pem"
ssh -i ~/.ssh/id_adguard_cert root@192.168.1.11 "rm -f /root/cert*.pem /root/chain*.pem /root/fullchain*.pem /root/privkey*.pem"
```

---

## Étape 5 : Planifier l'exécution automatique avec Cron

Sur l'hôte Nginx Proxy Manager, ajoutez une tâche planifiée pour exécuter le script tous les jours à 4h00 du matin :

```bash
(crontab -l 2>/dev/null; echo "0 4 * * * /root/scripts/sync-cert-adguard.sh") | crontab -
```

Vérifiez que la tâche est bien enregistrée :

```bash
crontab -l
```

Dès que Nginx Proxy Manager renouvellera votre certificat, il sera automatiquement poussé sur vos instances AdGuard Home la nuit suivante sans aucune intervention de votre part.

---

> **Auteur** : ce guide est proposé par **Amaury aka BlablaLinux**. Retrouvez tous mes services sur [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).

---
title: Notesnook - Installer son propre serveur (Docker, LXC, NPM)
description: Installez votre propre serveur Notesnook : Docker Compose dans un LXC Debian, MongoDB sécurisé, stockage S3, e-mails SMTP et reverse proxy Nginx Proxy Manager, avec des commandes à copier-coller.
published: true
date: 2026-10-16T22:55:08.314Z
tags: docker, lxc, proxmox, debian, auto-hébergement, tutoriel, s3, nginx-proxy-manager, docker-compose, notesnook, e2ee, mongodb
editor: markdown
dateCreated: 2026-10-16T22:17:53.709Z
---

[Notesnook](https://notesnook.com) est une application de notes chiffrées de bout en bout dont le serveur de synchronisation peut être auto-hébergé. Cette page décrit une installation complète, pas à pas : **Docker Compose** dans un **LXC Debian**, **MongoDB** protégé par authentification, **stockage S3** externe pour les pièces jointes, envoi d'e-mails par **SMTP** et publication via **Nginx Proxy Manager** (NPM).

> Le dépôt officiel [notesnook-sync-server](https://github.com/streetwriters/notesnook-sync-server) indique que l'auto-hébergement est en phase alpha, sans support officiel. Ce guide adapte son `docker-compose.yml` (authentification MongoDB, healthchecks, S3 externe). **Comparez-le avec la version officielle avant de l'appliquer** : les images et les variables peuvent évoluer.
{.is-warning}

## Vue d'ensemble

| Service | Rôle | Port | Domaine d'exemple |
|---|---|---|---|
| `notesnook-server` | Synchronisation des notes | 5264 | `notes-sync.example.com` |
| `identity-server` | Comptes et authentification | 8264 | `notes-auth.example.com` |
| `sse-server` | Événements en temps réel | 7264 | `notes-events.example.com` |
| `monograph-server` | Publication de notes publiques | 3000 | `notes-share.example.com` |
| `notesnook-db` | MongoDB (réseau Docker interne) | 27017 | aucun (non exposé) |
| S3 externe | Pièces jointes | 443 | `s3.example.com` |

Les quatre serveurs applicatifs ont chacun leur adresse publique, que l'on renseigne dans les applications Notesnook.

> Monograph n'est pas indispensable au fonctionnement des notes, mais à ma connaissance le client teste les quatre serveurs avant d'enregistrer la configuration. Le plus simple est donc de le conserver.
{.is-info}

## Ce qu'il faut adapter

| Élément | Valeur d'exemple dans ce guide | À remplacer par |
|---|---|---|
| Domaine | `example.com` | Votre domaine |
| Sous-domaines | `notes-sync`, `notes-auth`, `notes-events`, `notes-share` | Libre, mais restez cohérent partout (DNS, NPM, `.env`, applications) |
| IP du LXC | `IP_DU_LXC` | L'adresse IP du conteneur sur votre réseau |
| IP de NPM | `IP_NPM` | L'adresse IP de votre reverse proxy (pour le pare-feu) |
| Dossier d'installation | `/opt/notesnook` | Libre |
| Point d'accès S3 | `https://s3.example.com` | L'URL publique en HTTPS de votre S3 |
| Bucket S3 | `notesnook-storage` | Le nom de votre bucket |
| Région S3 | `us-east-1` | La région de votre fournisseur S3 |
| Serveur SMTP | `smtp.example.com`, port `587` | Votre serveur SMTP et son port |
| Nom de l'instance | `Mes notes` | Libre |
| Inscriptions | `DISABLE_SIGNUPS=false` | `true` si l'instance est privée |

## Prérequis

- Un LXC Debian (12 ou 13) sur Proxmox, ou n'importe quel hôte Debian où installer Docker.
- Nginx Proxy Manager opérationnel, capable d'émettre des certificats Let's Encrypt.
- Un nom de domaine dont vous gérez le DNS.
- Un stockage compatible S3, joignable publiquement en HTTPS avec un certificat valide. Ce guide a été validé avec MinIO et l'un de ses forks.
- Un compte SMTP pour l'envoi d'e-mails (si vous utilisez Gmail : validation en deux étapes activée et mot de passe d'application, saisi **sans espaces**).

## 1. Préparer le LXC et installer Docker

Docker dans un LXC Proxmox nécessite l'option *nesting* (et *keyctl* pour un conteneur non privilégié). **Sur l'hôte Proxmox**, remplacez `CTID` par l'identifiant du conteneur :

```bash
pct set CTID --features nesting=1,keyctl=1
pct reboot CTID
```

**Dans le LXC**, installez Docker depuis le dépôt officiel (passez cette étape si Docker est déjà installé) :

```bash
apt update && apt install -y ca-certificates curl openssl
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" > /etc/apt/sources.list.d/docker.list
apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
docker --version && docker compose version
```

## 2. Préparer le stockage S3

Notesnook envoie les pièces jointes directement dans un bucket S3, et les applications y accèdent par des liens signés : **votre S3 doit donc être joignable publiquement en HTTPS**.

Créez un bucket dédié (`notesnook-storage`) et une clé d'accès **limitée à ce bucket**. N'utilisez pas vos clés d'administration : elles seraient stockées dans un service exposé sur Internet.

Politique d'accès à utiliser :

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:*"],
    "Resource": [
      "arn:aws:s3:::notesnook-storage",
      "arn:aws:s3:::notesnook-storage/*"
    ]
  }]
}
```

Si vous changez le nom du bucket, adaptez les deux lignes `Resource`.

#### Créer le bucket et la clé {.tabset}

##### Console web

1. **Buckets → Create Bucket** : nom `notesnook-storage`, options par défaut.
2. **Identity → Policies → Create Policy** : nom `notesnook-policy`, collez le JSON ci-dessus.
3. **Identity → Users → Create User** : une clé d'accès (par exemple `notesnook-user`) et un secret long, avec `notesnook-policy` comme politique. Vous pouvez aussi créer une clé d'accès restreinte depuis votre utilisateur existant.
4. Notez la clé d'accès et le secret : ils serviront à l'étape 4.

> Selon la version de MinIO ou du fork utilisé, le menu *Identity* peut être absent de la console web. Utilisez alors l'onglet « Ligne de commande ».
{.is-info}

##### Ligne de commande

Avec le client `mc` (ou `mcli` selon l'implémentation). Remplacez l'URL et vos clés d'administration :

```bash
mc alias set s3admin https://s3.example.com CLE_ADMIN SECRET_ADMIN
mc mb s3admin/notesnook-storage

cat > notesnook-policy.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:*"],
    "Resource": [
      "arn:aws:s3:::notesnook-storage",
      "arn:aws:s3:::notesnook-storage/*"
    ]
  }]
}
EOF

S3_KEY_ID="notesnook-user"
S3_KEY_SECRET=$(openssl rand -hex 20)

mc admin policy create s3admin notesnook-policy notesnook-policy.json
mc admin user add s3admin "$S3_KEY_ID" "$S3_KEY_SECRET"
mc admin policy attach s3admin notesnook-policy --user "$S3_KEY_ID"

echo "Clé : $S3_KEY_ID   Secret : $S3_KEY_SECRET"
```

Notez la clé et le secret affichés : ils serviront à l'étape 4. Sur un `mc` ancien, remplacez `policy create` par `policy add`, et `policy attach … --user` par `policy set s3admin notesnook-policy user="$S3_KEY_ID"`.

---

Votre proxy S3 doit accepter les gros envois et transmettre l'en-tête `Host` tel quel (sinon les signatures des liens échouent). Dans NPM, pour le Proxy Host de votre S3, ajoutez dans la configuration personnalisée :

```nginx
client_max_body_size 0;
proxy_set_header Host $host;
```

## 3. Créer les enregistrements DNS

Créez quatre enregistrements (A, AAAA ou CNAME) pointant vers votre reverse proxy, comme pour vos autres services :

```
notes-sync.example.com
notes-auth.example.com
notes-events.example.com
notes-share.example.com
```

## 4. Préparer les fichiers de l'installation

> **Gardez le même terminal ouvert de l'étape 4.2 jusqu'à la fin de l'étape 6.** Les mots de passe générés sont stockés dans des variables de session : si vous fermez le terminal, ils sont perdus.
{.is-warning}

### 4.1 Dossier et fichier clé MongoDB

MongoDB exige un fichier clé pour activer l'authentification sur un replica set (même à un seul nœud). L'UID `999` est celui de l'utilisateur `mongodb` de l'image officielle.

```bash
mkdir -p /opt/notesnook/data && cd /opt/notesnook
openssl rand -base64 756 > data/mongo-keyfile
chown 999:999 data/mongo-keyfile
chmod 400 data/mongo-keyfile
```

### 4.2 Variables et secrets

**Adaptez ce bloc** (voir le tableau « Ce qu'il faut adapter »), puis collez-le :

```bash
cd /opt/notesnook

export INSTANCE_NAME="Mes notes"
export DOMAIN="example.com"
export S3_URL="https://s3.example.com"
export S3_BUCKET="notesnook-storage"
export S3_REGION="us-east-1"
export S3_KEY_ID="notesnook-user"
export SMTP_SERVER="smtp.example.com"
export SMTP_PORT_NUM="587"
export SMTP_USER="notes@example.com"
```

Les deux secrets suivants sont saisis **avec l'historique du shell désactivé**, pour qu'ils n'apparaissent pas dans `~/.bash_history`. Remplacez les valeurs :

```bash
set +o history
export S3_KEY_SECRET="SECRET_DE_LA_CLE_S3"
export SMTP_PASS="MOT_DE_PASSE_SMTP"
set -o history
```

Générez ensuite les secrets de l'application et les mots de passe MongoDB :

```bash
export API_SECRET=$(openssl rand -hex 32)
export MONGO_APP_PASSWORD=$(openssl rand -hex 24)
export MONGO_ADMIN_PASSWORD=$(openssl rand -hex 24)
export MONGO_BACKUP_PASSWORD=$(openssl rand -hex 24)

echo "mongoadmin : $MONGO_ADMIN_PASSWORD"
echo "backup     : $MONGO_BACKUP_PASSWORD"
```

**Enregistrez ces deux mots de passe dans votre gestionnaire de mots de passe** : ils ne figureront pas dans le `.env`. Le compte `notesnook` utilisé par les services est, lui, écrit dans le `.env`.

### 4.3 Fichier `.env`

```bash
cat > .env <<EOF
INSTANCE_NAME=$INSTANCE_NAME
NOTESNOOK_API_SECRET=$API_SECRET

# true = inscriptions fermées
DISABLE_SIGNUPS=false

# URLs publiques
AUTH_SERVER_PUBLIC_URL=https://notes-auth.$DOMAIN
MONOGRAPH_PUBLIC_URL=https://notes-share.$DOMAIN
NOTESNOOK_APP_PUBLIC_URL=https://app.notesnook.com

# Stockage S3 externe
S3_SERVICE_URL=$S3_URL
S3_BUCKET_NAME=$S3_BUCKET
S3_REGION=$S3_REGION
S3_ACCESS_KEY_ID=$S3_KEY_ID
S3_ACCESS_KEY=$S3_KEY_SECRET

# SMTP
SMTP_USERNAME=$SMTP_USER
SMTP_PASSWORD=$SMTP_PASS
SMTP_HOST=$SMTP_SERVER
SMTP_PORT=$SMTP_PORT_NUM

# MongoDB (compte applicatif)
MONGO_APP_USER=notesnook
MONGO_APP_PASSWORD=$MONGO_APP_PASSWORD

# Vide pour l'instant : l'authentification est activée à l'étape 6
MONGO_AUTH_ARGS=
EOF

chmod 600 .env
```

Vérifiez le contenu **sans afficher les secrets** :

```bash
sed -E 's/(PASSWORD|SECRET|ACCESS_KEY)=.*/\1=********/' .env
```

> `NOTESNOOK_APP_PUBLIC_URL` désigne le client web. Cette installation n'en héberge pas : on laisse l'adresse du client officiel. Les applications utilisent vos serveurs dès que vous leur indiquez vos adresses.
{.is-info}

### 4.4 Fichier `docker-compose.yml`

```bash
cat > docker-compose.yml <<'EOF'
x-server-discovery: &server-discovery
  NOTESNOOK_SERVER_PORT: 5264
  NOTESNOOK_SERVER_HOST: notesnook-server
  IDENTITY_SERVER_PORT: 8264
  IDENTITY_SERVER_HOST: identity-server
  SSE_SERVER_PORT: 7264
  SSE_SERVER_HOST: sse-server
  SELF_HOSTED: 1
  IDENTITY_SERVER_URL: ${AUTH_SERVER_PUBLIC_URL}
  NOTESNOOK_APP_HOST: ${NOTESNOOK_APP_PUBLIC_URL}

x-env-files: &env-files
  - .env

services:
  notesnook-db:
    image: mongo:7.0.12
    container_name: notesnook-db
    hostname: notesnook-db
    restart: always
    command: --replSet rs0 --bind_ip_all ${MONGO_AUTH_ARGS}
    # Décommentez uniquement si un outil de sauvegarde externe doit joindre MongoDB
    # (voir l'étape 10 et restreignez ce port par pare-feu) :
    # ports:
    #   - "27017:27017"
    volumes:
      - ./data/db:/data/db
      - ./data/dbconfig:/data/configdb
      - ./data/mongo-keyfile:/etc/mongo/keyfile:ro
    networks:
      - notesnook
    healthcheck:
      test: ["CMD-SHELL", "mongosh --quiet --eval 'quit(db.hello().isWritablePrimary ? 0 : 1)'"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 30s

  identity-server:
    image: streetwriters/identity:latest
    container_name: notesnook-identity
    restart: always
    ports:
      - "8264:8264"
    networks:
      - notesnook
    env_file: *env-files
    depends_on:
      notesnook-db:
        condition: service_healthy
    environment:
      <<: *server-discovery
      MONGODB_CONNECTION_STRING: "mongodb://${MONGO_APP_USER}:${MONGO_APP_PASSWORD}@notesnook-db:27017/identity?replSet=rs0&authSource=admin"
      MONGODB_DATABASE_NAME: identity
    healthcheck:
      test: ["CMD-SHELL", "wget --spider -q http://127.0.0.1:8264/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  notesnook-server:
    image: streetwriters/notesnook-sync:latest
    container_name: notesnook-server
    restart: always
    ports:
      - "5264:5264"
    networks:
      - notesnook
    env_file: *env-files
    depends_on:
      notesnook-db:
        condition: service_healthy
      identity-server:
        condition: service_healthy
    environment:
      <<: *server-discovery
      MONGODB_CONNECTION_STRING: "mongodb://${MONGO_APP_USER}:${MONGO_APP_PASSWORD}@notesnook-db:27017/?replSet=rs0&authSource=admin"
      MONGODB_DATABASE_NAME: notesnook
      S3_INTERNAL_SERVICE_URL: ${S3_SERVICE_URL}
      S3_INTERNAL_BUCKET_NAME: ${S3_BUCKET_NAME}
      S3_SERVICE_URL: ${S3_SERVICE_URL}
      S3_BUCKET_NAME: ${S3_BUCKET_NAME}
      S3_REGION: ${S3_REGION}
      S3_ACCESS_KEY_ID: ${S3_ACCESS_KEY_ID}
      S3_ACCESS_KEY: ${S3_ACCESS_KEY}
    healthcheck:
      test: ["CMD-SHELL", "wget --spider -q http://127.0.0.1:5264/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  sse-server:
    image: streetwriters/sse:latest
    container_name: notesnook-sse
    restart: always
    ports:
      - "7264:7264"
    networks:
      - notesnook
    env_file: *env-files
    depends_on:
      notesnook-server:
        condition: service_healthy
    environment:
      <<: *server-discovery
    healthcheck:
      test: ["CMD-SHELL", "wget --spider -q http://127.0.0.1:7264/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  monograph-server:
    image: streetwriters/monograph:latest
    container_name: notesnook-monograph
    restart: always
    ports:
      - "3000:3000"
    networks:
      - notesnook
    depends_on:
      notesnook-server:
        condition: service_healthy
    environment:
      API_HOST: http://notesnook-server:5264
      HOST: 0.0.0.0
      PUBLIC_URL: ${MONOGRAPH_PUBLIC_URL}
    healthcheck:
      test: ["CMD-SHELL", "bun -e \"fetch('http://127.0.0.1:3000/').then(r => process.exit(r.status < 500 ? 0 : 1)).catch(() => process.exit(1))\""]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 20s

networks:
  notesnook:
EOF

docker compose config -q && echo "Compose OK"
```

Quelques points de conception à connaître :

- `restart: always` et un healthcheck sur chaque service ; les `depends_on` attendent que le service précédent soit réellement sain.
- Le service Monograph n'a **pas** de `env_file` : il n'a besoin que de deux variables et ne doit pas recevoir vos secrets SMTP, S3 et API.
- `HOST: 0.0.0.0` est indispensable pour Monograph : sans lui, il n'écoute que sur le loopback du conteneur et reste injoignable depuis NPM.
- Le healthcheck de MongoDB (`db.hello()`) fonctionne sans authentification.

## 5. Démarrer MongoDB, initialiser le replica set et créer les comptes

MongoDB démarre d'abord **sans authentification** (le `.env` contient `MONGO_AUTH_ARGS=` vide), le temps de créer les comptes.

```bash
docker compose up -d notesnook-db

until docker exec notesnook-db mongosh --quiet --eval 'db.hello().ok' >/dev/null 2>&1; do echo "Attente de MongoDB..."; sleep 2; done

docker exec notesnook-db mongosh --quiet --eval 'rs.initiate({_id:"rs0",members:[{_id:0,host:"notesnook-db:27017"}]})'

until docker exec notesnook-db mongosh --quiet --eval 'quit(db.hello().isWritablePrimary ? 0 : 1)' >/dev/null 2>&1; do echo "Attente du nœud principal..."; sleep 2; done
echo "Replica set prêt"
```

Créez les trois comptes :

```bash
docker exec notesnook-db mongosh --quiet --eval "
const a = db.getSiblingDB('admin');
a.createUser({user:'mongoadmin', pwd:'$MONGO_ADMIN_PASSWORD', roles:[{role:'root', db:'admin'}]});
a.createUser({user:'notesnook', pwd:'$MONGO_APP_PASSWORD', roles:[{role:'readWrite', db:'notesnook'},{role:'readWrite', db:'identity'}]});
a.createUser({user:'backup', pwd:'$MONGO_BACKUP_PASSWORD', roles:[{role:'backup', db:'admin'},{role:'restore', db:'admin'}]});
printjson(a.getUsers().users.map(u => u.user));
"
```

Le résultat doit lister `backup`, `mongoadmin` et `notesnook`.

| Compte | Rôle | Usage |
|---|---|---|
| `notesnook` | Lecture/écriture sur les bases `notesnook` et `identity` | Les services Notesnook |
| `backup` | Rôles `backup` et `restore` | Un outil de sauvegarde externe (optionnel) |
| `mongoadmin` | `root` | Administration manuelle |

## 6. Activer l'authentification et démarrer toute la pile

```bash
sed -i 's|^MONGO_AUTH_ARGS=.*|MONGO_AUTH_ARGS="--auth --keyFile /etc/mongo/keyfile"|' .env
docker compose up -d
watch -n 5 docker compose ps
```

Attendez que tous les services affichent `healthy` (une à deux minutes), puis quittez `watch` avec `Ctrl+C`.

Vérifiez que l'authentification est active (la première commande doit échouer avec une erreur d'authentification, la seconde réussir) :

```bash
docker exec notesnook-db mongosh --quiet --eval 'db.adminCommand({listDatabases:1})'
docker exec notesnook-db mongosh --quiet -u mongoadmin -p "$MONGO_ADMIN_PASSWORD" --authenticationDatabase admin --eval 'db.adminCommand({listDatabases:1}).databases.map(d => d.name)'
```

Testez ensuite les services en local :

```bash
for p in 8264 5264 7264; do echo -n "port $p : "; curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:$p/health; done
curl -s -o /dev/null -w "monograph : %{http_code}\n" http://127.0.0.1:3000/
```

Les trois premiers doivent répondre `200`. Quand tout est bon, supprimez les mots de passe de la session :

```bash
unset S3_KEY_SECRET SMTP_PASS API_SECRET MONGO_APP_PASSWORD MONGO_ADMIN_PASSWORD MONGO_BACKUP_PASSWORD
```

## 7. Publier les services avec Nginx Proxy Manager

Créez quatre **Proxy Hosts**. Pour chacun :

1. Onglet **Détails** : nom de domaine, schéma `http`, **Forward IP** `IP_DU_LXC`, port selon le tableau ci-dessous. Cochez **Block Common Exploits** et **Websockets Support** ; laissez *Cache Assets* désactivé.
2. Onglet **SSL** : nouveau certificat Let's Encrypt, **Force SSL** et **HTTP/2**.
3. Icône d'engrenage de la fenêtre : collez le **bloc général** ci-dessous.
4. Onglet **Emplacement personnalisé** : ajoutez la localisation `/` avec le même schéma, la même IP et le même port, puis son engrenage pour le **bloc de la location**.

| Domaine | Port |
|---|---|
| `notes-sync.example.com` | 5264 |
| `notes-auth.example.com` | 8264 |
| `notes-events.example.com` | 7264 |
| `notes-share.example.com` | 3000 |

### Bloc général : sync, auth et share (partie commune)

```nginx
gzip on;
gzip_min_length 1000;
gzip_vary on;
gzip_proxied any;
gzip_comp_level 6;
gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
client_body_buffer_size 512k;
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
```

Ajoutez ensuite ces trois lignes **selon l'hôte** :

```nginx
# notes-sync
client_max_body_size 100m;
proxy_read_timeout 86400s;
proxy_send_timeout 86400s;
```

```nginx
# notes-auth et notes-share
client_max_body_size 10m;
proxy_read_timeout 300s;
proxy_send_timeout 300s;
```

### Bloc général : events (flux SSE)

Pas de compression ni de mise en tampon pour ce flux continu :

```nginx
gzip off;
client_body_buffer_size 64k;
client_max_body_size 1m;
proxy_read_timeout 86400s;
proxy_send_timeout 86400s;
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
```

### Bloc de la location `/`

Pour `notes-sync` et `notes-auth` :

```nginx
proxy_hide_header X-Powered-By;
add_header Referrer-Policy "no-referrer" always;
add_header X-Frame-Options SAMEORIGIN always;
add_header X-Content-Type-Options nosniff always;
add_header X-Xss-Protection "1; mode=block" always;
add_header X-Robots-Tag "noindex, noarchive, nofollow" always;
```

Pour `notes-events`, ajoutez en plus ces lignes (désactivation de la mise en tampon) :

```nginx
proxy_buffering off;
proxy_cache off;
chunked_transfer_encoding off;
add_header X-Accel-Buffering no always;
```

Pour `notes-share`, remplacez la ligne `Referrer-Policy` par `add_header Referrer-Policy "strict-origin-when-cross-origin" always;`. Ces pages sont publiques : supprimez aussi la ligne `X-Robots-Tag` si vous souhaitez que les notes publiées soient référencées par les moteurs de recherche.

### Vérification depuis l'extérieur

```bash
for s in sync auth events share; do echo -n "notes-$s : "; curl -s -o /dev/null -w "%{http_code}\n" https://notes-$s.example.com/health; done
```

Les trois premiers doivent répondre `200`. Pour `notes-share`, un code autre que `200` n'est pas forcément anormal : cette route de santé n'existe peut-être pas sur Monograph. Un `502`, un `504` ou une erreur de certificat est en revanche un vrai problème.

## 8. Créer le premier compte

Utilisez un client Notesnook (web, ordinateur ou Android) en configurant d'abord vos quatre adresses. La procédure complète, avec captures d'écran, est décrite dans la page [Utiliser l'instance BlablaLinux](/tutoriels/notesnook/utiliser-instance-blablalinux) : elle est rédigée pour mon instance, il suffit d'y remplacer mes adresses par les vôtres.

Vérifications à faire avec votre premier compte :

1. **Synchronisation** : créez une note sur un appareil et vérifiez qu'elle apparaît sur un second.
2. **Pièces jointes** : ajoutez une image à une note, puis vérifiez sa présence dans le bucket.
3. **E-mails** : déclenchez un envoi (« Mot de passe oublié », par exemple) et vérifiez la réception, spams compris.
4. **Abonnement** : dans **Paramètres → Détails de l'abonnement**, regardez le forfait affiché. Sur mon instance, il s'agit du forfait complet « Believer », avec une échéance qui se recalcule à chaque chargement. À vérifier sur votre version, car ce comportement peut changer d'une version à l'autre.

Si l'instance est **privée**, fermez les inscriptions une fois votre compte créé :

```bash
cd /opt/notesnook
sed -i 's/^DISABLE_SIGNUPS=false/DISABLE_SIGNUPS=true/' .env
docker compose up -d
```

## 9. Sécuriser l'installation

- **Pare-feu** : ne laissez joindre les ports 5264, 8264, 7264 et 3000 que depuis `IP_NPM`, avec le pare-feu Proxmox du LXC. Les règles `ufw` seules ne suffisent pas, car Docker les contourne.
- **MongoDB** : ne publiez jamais son port sur Internet. Il n'est exposé que si vous décommentez `ports` dans le compose pour un outil de sauvegarde (étape 10).
- **Fichier `.env`** : il contient tous vos secrets. Gardez les droits `600` et excluez-le de tout dépôt Git.
- **E-mails** : si vous ouvrez l'instance au public, utilisez une adresse d'envoi dédiée plutôt qu'un compte personnel. Les destinataires verront l'expéditeur, et les comptes grand public ont des limites d'envoi.
- **Abus** : sur une instance publique, ajoutez une limitation de débit sur `notes-auth` dans NPM, ainsi qu'une jail Fail2Ban sur ses journaux d'accès.
- **Versions des images** : `latest` peut changer à chaque `pull`. Une fois l'installation validée, épinglez les versions par empreinte :

```bash
for i in streetwriters/identity streetwriters/notesnook-sync streetwriters/sse streetwriters/monograph; do
  docker image inspect --format '{{index .RepoDigests 0}}' $i:latest
done
```

Remplacez ensuite chaque `image: nom:latest` du compose par la valeur affichée (`nom@sha256:...`).

## 10. Sauvegardes

Deux choses à sauvegarder : la base MongoDB (comptes et notes chiffrées) et le bucket S3 (pièces jointes). Pour MongoDB, une sauvegarde manuelle des deux bases (le mot de passe est demandé sans être affiché) :

```bash
read -rsp "Mot de passe mongoadmin : " MPW; echo
for db in notesnook identity; do
  docker exec notesnook-db mongodump --uri="mongodb://mongoadmin:${MPW}@localhost:27017/?authSource=admin&directConnection=true" --db="$db" --archive --gzip > "/root/backup-notesnook-${db}-$(date +%F).gz"
done
unset MPW
ls -lh /root/backup-notesnook-*.gz
```

**Testez la restauration** (`mongorestore`) sur une instance de test avant d'en avoir besoin : une sauvegarde jamais restaurée reste une hypothèse.

### Avec un outil de sauvegarde externe (optionnel)

Pour un outil comme Databasus, décommentez le bloc `ports` du service `notesnook-db`, **restreignez le port 27017 à l'IP de l'outil** par pare-feu, puis `docker compose up -d`. Réglages à utiliser dans l'outil :

| Champ | Valeur |
|---|---|
| Hôte / port | `IP_DU_LXC` / `27017` |
| Utilisateur | `backup` |
| Mot de passe | Celui noté à l'étape 4.2 |
| Base d'authentification | `admin` |
| Connexion directe | **Activée** |
| Base | `notesnook`, puis une seconde configuration pour `identity` |

Il faut sauvegarder les **deux** bases : `identity` contient les comptes utilisateurs, sans lesquels les notes sont inutilisables. Si l'outil signale que l'hôte `notesnook-db` est introuvable, activez la connexion directe, ou ajoutez `IP_DU_LXC notesnook-db` dans le fichier `/etc/hosts` de la machine qui exécute l'outil.

## 11. Mises à jour

Faites une sauvegarde (étape 10), puis :

```bash
cd /opt/notesnook
docker compose pull
docker compose up -d
docker image prune -f
docker compose ps
```

Lisez les notes de version du dépôt avant de mettre à jour : le serveur est en alpha et son comportement peut changer.

## Dépannage

| Symptôme | Cause probable | Que faire |
|---|---|---|
| Le client indique qu'il ne peut pas se connecter à un des serveurs et « Enregistrer » reste grisé | Un des quatre services ne répond pas | Testez chaque adresse avec `curl -i https://notes-xxx.example.com/health`, regardez le serveur mentionné dans le message d'erreur |
| `504` ou `502` dans NPM | Mauvaise IP, mauvais port ou mauvais schéma, ou pare-feu | Le schéma doit être `http`. Testez depuis la machine NPM : `curl -i --max-time 5 http://IP_DU_LXC:PORT/` |
| Monograph : connexion réinitialisée alors que le conteneur tourne | Il écoute sur le loopback du conteneur | Vérifiez `HOST: 0.0.0.0` dans son bloc `environment` |
| Un conteneur reste `unhealthy` | Healthcheck en échec | `docker inspect --format '{{json .State.Health.Log}}' notesnook-identity` (adaptez le nom) et `docker compose logs --tail=50 nom-du-service` |
| `not authorized` ou `Authentication failed` dans les logs | Identifiants différents entre `.env` et MongoDB, ou mauvaise base d'authentification | Vérifiez que `MONGO_APP_PASSWORD` correspond au compte `notesnook`, et que la chaîne contient `authSource=admin` |
| Aucun e-mail reçu | SMTP mal configuré, ou connexion bloquée | `docker compose logs --since 10m identity-server notesnook-server \| grep -i -E "smtp\|mail"` |
| Erreur `unable to get certificate CRL` à l'envoi d'e-mails | Le serveur ne peut pas télécharger les listes de révocation des certificats | Vérifiez votre filtre DNS (AdGuard Home, Pi-hole) et le pare-feu sortant, port 80 compris : autorisez le domaine bloqué |
| Pièces jointes qui échouent, erreur de signature | `Host` modifié par le proxy S3, ou envoi trop gros | Appliquez la configuration S3 de l'étape 2 (`client_max_body_size 0;` et `Host` inchangé) |
| Les avertissements `DataProtection-Keys` au démarrage | Clés internes .NET non persistées | Sans impact constaté sur cette installation |

## Liens utiles

- [Dépôt notesnook-sync-server](https://github.com/streetwriters/notesnook-sync-server)
- [Site officiel de Notesnook](https://notesnook.com)
- [Utiliser l'instance BlablaLinux (configuration des clients)](/fr/tutoriels-notesnook-utiliser-instance-blablalinux)
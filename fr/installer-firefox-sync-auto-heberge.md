---
title: Installer un serveur Firefox Sync auto-hébergé (Docker, PostgreSQL, NPM)
description: >-
  Tutoriel complet pour héberger votre propre serveur Firefox Sync : Docker
  Compose, PostgreSQL, Nginx Proxy Manager, gestion des inscriptions et
  sauvegardes avec Databasus.
published: true
date: '2026-11-11T22:41:24.451Z'
tags:
  - firefox
  - corosync
  - docker
  - postgresql
  - databasus
  - tutoriel
  - docker compose
  - auto hébergement
  - nginx proxy manager
editor: markdown
dateCreated: '2026-11-08T21:12:58.386Z'
---

Cette page décrit, pas à pas, comment j'ai installé mon serveur **Firefox Sync auto-hébergé**. Pour comprendre ce que fait ce service et comment configurer Firefox, consultez d'abord la page [Firefox Sync auto-hébergé](/fr/firefox-sync-auto-heberge).

> [!NOTE] Ce que vous allez obtenir
> Un serveur de synchronisation Firefox accessible en HTTPS, basé sur `syncstorage-rs` (le serveur officiel de Mozilla, écrit en Rust) avec une base PostgreSQL, le tout en Docker Compose derrière Nginx Proxy Manager.

## Architecture

| Élément | Rôle |
|---|---|
| `syncserver` | Serveur de stockage de synchronisation et Tokenserver (image `syncserver-postgres`) |
| `postgres` | Base de données unique, partagée par le stockage et le Tokenserver |
| Nginx Proxy Manager | Reverse proxy HTTPS (certificat, en-têtes, protections de base) |
| Databasus | Sauvegardes planifiées de la base PostgreSQL |

Les comptes utilisateurs ne sont **pas** gérés par ce serveur : l'authentification passe par les comptes Mozilla (FxA). Seul le **stockage** des données de synchronisation est déplacé chez vous.

## Prérequis

- Un LXC (ou une VM) **Debian** avec **Docker** et le plugin **Docker Compose** installés.
- Un reverse proxy **Nginx Proxy Manager** avec un certificat valide pour votre domaine.
- Un nom de domaine dédié, par exemple `syncstorage-rs.blablalinux.be`.
- Le port 8000 du LXC joignable depuis le reverse proxy.

Vérifiez Docker :

```bash
docker --version && docker compose version
```

## 1. Enregistrement DNS

Créez un enregistrement DNS pour le sous-domaine choisi (`syncstorage-rs.blablalinux.be`) qui pointe vers l'IP publique de votre reverse proxy.

## 2. Préparer le dossier et le `docker-compose.yml`

Je place tout dans `/root/syncstorage-rs` :

```bash
mkdir -p /root/syncstorage-rs && cd /root/syncstorage-rs
```

Créez le fichier `docker-compose.yml` :

```yaml
services:
  postgres:
    image: postgres:17-alpine
    container_name: syncserver-postgres
    ports:
      - "5432:5432"
    restart: always
    environment:
      POSTGRES_USER: sync
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: syncdb
    volumes:
      - ./postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U sync -d syncdb"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 20s

  syncserver:
    image: ghcr.io/mozilla-services/syncstorage-rs/syncserver-postgres:${SYNCSERVER_VERSION:-latest}
    platform: linux/amd64
    container_name: syncserver
    restart: always
    depends_on:
      postgres:
        condition: service_healthy
    ports:
      - "${SYNC_PORT:-8000}:${SYNC_PORT:-8000}"
    environment:
      SYNC_HOST: "0.0.0.0"
      SYNC_PORT: "${SYNC_PORT:-8000}"
      SYNC_MASTER_SECRET: "${SYNC_MASTER_SECRET}"
      SYNC_SYNCSTORAGE__DATABASE_URL: "postgres://sync:${POSTGRES_PASSWORD}@postgres/syncdb"
      SYNC_TOKENSERVER__DATABASE_URL: "postgres://sync:${POSTGRES_PASSWORD}@postgres/syncdb"
      SYNC_TOKENSERVER__ENABLED: "true"
      SYNC_TOKENSERVER__RUN_MIGRATIONS: "true"
      SYNC_TOKENSERVER__FXA_EMAIL_DOMAIN: "api.accounts.firefox.com"
      SYNC_TOKENSERVER__FXA_OAUTH_SERVER_URL: "https://oauth.accounts.firefox.com"
      SYNC_TOKENSERVER__INIT_NODE_URL: "${SYNC_TOKENSERVER__INIT_NODE_URL}"
      SYNC_TOKENSERVER__INIT_NODE_CAPACITY: "${SYNC_TOKENSERVER__INIT_NODE_CAPACITY:-50}"
      SYNC_TOKENSERVER__ALLOW_NEW_USERS: "${SYNC_TOKENSERVER__ALLOW_NEW_USERS:-true}"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:${SYNC_PORT:-8000}/__heartbeat__"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
```

Points à noter :

- **Une seule base** (`syncdb`) sert au stockage et au Tokenserver.
- **`restart: always`** et un **healthcheck** sont définis sur les deux services. Le serveur ne démarre que lorsque PostgreSQL est `healthy`.
- Les données PostgreSQL sont stockées dans `./postgres-data` (bind mount), ce qui simplifie les sauvegardes et la migration.

> [!WARNING] Le port 5432
> Le port PostgreSQL est exposé pour permettre les sauvegardes par un outil externe (Databasus). **Ne le redirigez jamais vers Internet.** Pour le limiter à l'IP du LXC, remplacez `"5432:5432"` par `"IP_DU_LXC:5432:5432"`.

## 3. Créer le fichier `.env`

Les secrets sont générés au hasard au moment de la création du fichier :

```bash
cat > .env <<EOF
POSTGRES_PASSWORD=$(openssl rand -hex 24)
SYNC_MASTER_SECRET=$(openssl rand -hex 32)
SYNC_TOKENSERVER__INIT_NODE_URL=https://syncstorage-rs.blablalinux.be
SYNC_TOKENSERVER__INIT_NODE_CAPACITY=50
SYNC_TOKENSERVER__ALLOW_NEW_USERS=true
SYNC_PORT=8000
SYNCSERVER_VERSION=latest
EOF

chmod 600 .env
```

Adaptez `SYNC_TOKENSERVER__INIT_NODE_URL` avec **votre** domaine, en HTTPS.

> [!IMPORTANT] L'URL du nœud ne se change pas à la volée
> `INIT_NODE_URL` et `INIT_NODE_CAPACITY` ne sont écrits dans la base qu'au **premier démarrage**. Si vous vous trompez, supprimez `./postgres-data` et relancez (voir la section Dépannage). Je vous conseille aussi de remplacer `latest` par une version précise dans `SYNCSERVER_VERSION` une fois le service validé.

## 4. Démarrer et vérifier

```bash
docker compose config -q && echo "compose OK"
docker compose up -d
docker compose ps
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8000/__heartbeat__
```

- Le `curl` doit afficher `200`.
- `docker compose ps` doit montrer les deux conteneurs en `healthy` après environ une minute.
- En cas de souci : `docker compose logs -f syncserver`.

## 5. Nginx Proxy Manager

Dans NPM, créez un **hôte proxy** (*Hôtes* → *Hôtes proxy* → *Ajouter Hôte proxy*). Voici ce que j'ai configuré, onglet par onglet.

### Onglet « Détails »

| Champ | Valeur |
|---|---|
| Noms de domaine | `syncstorage-rs.blablalinux.be` |
| Schéma | `http` |
| Nom d'hôte de redirection / IP | IP du LXC (dans mon cas `192.168.2.149`) |
| Port de redirection | `8000` |
| Liste d'accès | Accessible au public |
| Ressources du cache | Désactivé |
| Bloquer les exploits courants | Activé |
| Prise en charge de Websockets | Désactivé |

> [!NOTE] Pas de websockets
> Firefox Sync fonctionne avec de simples requêtes HTTPS (API REST). Les websockets ne sont pas nécessaires.

### Onglet « Emplacement personnalisé »

Ajoutez une localisation (*Ajouter localisation*) avec :

| Champ | Valeur |
|---|---|
| Location | `/` |
| Schéma | `http` |
| Nom d'hôte de redirection / IP | IP du LXC |
| Port de redirection | `8000` |
| Liste d'accès | Inherit from Proxy Host |

Dans le champ de configuration de cette localisation (icône engrenage), ajoutez ces en-têtes :

```nginx
proxy_hide_header X-Powered-By;
add_header Referrer-Policy "no-referrer" always;
add_header X-Frame-Options SAMEORIGIN always;
add_header X-Xss-Protection "1; mode=block" always;
add_header X-Robots-Tag "noindex, noarchive, nofollow" always;
```

Le dernier en-tête (`X-Robots-Tag`) demande aux moteurs de recherche de ne pas indexer l'URL du service. Ce n'est pas une protection, seulement une bonne pratique pour un service sensible.

### Onglet « SSL »

| Option | Valeur |
|---|---|
| Certificat SSL | Votre certificat (chez moi, le wildcard `*.blablalinux.be`) ou un nouveau certificat Let's Encrypt |
| Forcer SSL | Activé |
| Prise en charge de HTTP/2 | Activé |
| HSTS activé | Activé |
| Sous-domaines HSTS | Activé |

### Configuration Nginx personnalisée (icône engrenage de la fenêtre)

C'est l'icône en haut à droite de la fenêtre de l'hôte proxy, à côté des onglets :

```nginx
gzip on;
gzip_min_length 1000;
gzip_disable "msie6";
gzip_vary on;
gzip_proxied any;
gzip_comp_level 6;
gzip_buffers 16 8k;
gzip_http_version 1.1;
gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
client_body_buffer_size 512k;
proxy_read_timeout 60s;
client_max_body_size 10m;
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
```

Deux valeurs sont volontairement **restreintes** par rapport à ce que je mets sur d'autres services :

- `client_max_body_size 10m;` : les requêtes Sync sont petites. Une limite évite que n'importe qui envoie de gros corps vers un service public. Si une synchronisation échoue avec une erreur 413, augmentez à `20m`.
- `proxy_read_timeout 60s;` : Sync fait des requêtes courtes, inutile de garder des connexions ouvertes 24 h.

Cliquez sur **Enregistrer**.

### Test depuis l'extérieur

```bash
curl https://syncstorage-rs.blablalinux.be/__heartbeat__
```

Vous devez obtenir une réponse JSON de ce type (la version varie selon l'image) :

```json
{"status":"Ok","database":"Ok","quota":{"enabled":false,"size":2097152000},"version":"0.24.1"}
```

> [!NOTE] Quotas désactivés
> Avec PostgreSQL, les quotas par utilisateur sont désactivés (`"enabled":false`). Chaque utilisateur peut donc stocker sans limite, ce qui est une raison de plus de contrôler les inscriptions (section suivante).

Si vous obtenez une erreur 403, désactivez temporairement « Bloquer les exploits courants » pour vérifier que l'option n'est pas en cause.

## 6. Configurer Firefox et inscrire les utilisateurs

La procédure côté navigateur (nouvelle installation, Firefox déjà synchronisé, Android) est détaillée sur la page [Firefox Sync auto-hébergé](/fr/firefox-sync-auto-heberge). L'essentiel : régler `identity.sync.tokenserver.uri` sur `https://VOTRE-DOMAINE/1.0/sync/1.5` dans `about:config`, **avant** de se connecter.

Pour vérifier que votre premier compte est bien enregistré :

```bash
cd /root/syncstorage-rs
docker compose exec postgres psql -U sync -d syncdb -c "SELECT uid, created_at FROM users;"
docker compose exec postgres psql -U sync -d syncdb -c "SELECT node, capacity, current_load FROM nodes;"
```

Vous devez voir une ligne dans `users`, et `current_load` à 1 sur votre nœud, avec la capacité définie dans le `.env`.

## 7. Contrôler les inscriptions

Le serveur n'authentifie pas lui-même les utilisateurs : toute personne disposant d'un compte Mozilla peut s'y inscrire si elle règle son navigateur sur votre URL. Deux réglages permettent de limiter cela.

**Fermer les inscriptions** avec `SYNC_TOKENSERVER__ALLOW_NEW_USERS` : quand il vaut `false`, seuls les utilisateurs déjà enregistrés peuvent utiliser le service. Il n'existe pas de liste blanche par adresse e-mail : l'option est tout-ou-rien.

**Plafonner le nombre d'utilisateurs** avec `SYNC_TOKENSERVER__INIT_NODE_CAPACITY` : une fois la limite atteinte, les nouveaux utilisateurs reçoivent une erreur, tandis que ceux déjà inscrits ne sont pas touchés.

La méthode que j'utilise pour une liste fermée :

1. Laisser `ALLOW_NEW_USERS=true` au départ.
2. Faire connecter chaque personne autorisée.
3. Fermer ensuite les inscriptions :

```bash
cd /root/syncstorage-rs
sed -i 's/^SYNC_TOKENSERVER__ALLOW_NEW_USERS=.*/SYNC_TOKENSERVER__ALLOW_NEW_USERS=false/' .env
docker compose up -d
```

4. Vérifier avec un compte jamais utilisé : la synchronisation doit être refusée, et les journaux doivent mentionner que l'inscription est désactivée :

```bash
docker compose logs syncserver | grep -i registration
```

> [!WARNING] À tester sur votre version
> Contrôlez que le blocage fonctionne bien avec **votre** version de l'image. Si un nouveau compte parvient quand même à se synchroniser, votre version ne connaît peut-être pas encore cette option : utilisez alors une version plus récente de l'image.

## 8. Sauvegardes avec Databasus

Toutes les données de synchronisation sont dans la base `syncdb`. Une seule sauvegarde couvre donc le stockage et le Tokenserver.

Dans Databasus :

1. Créez un **workspace**, puis ajoutez une **destination de stockage** (idéalement hors du LXC).
2. Cliquez sur **Add database**, choisissez PostgreSQL et saisissez :

| Champ | Valeur |
|---|---|
| Hôte | IP du LXC |
| Port | `5432` |
| Base | `syncdb` |
| Utilisateur | `sync` (ou un utilisateur en lecture seule, voir ci-dessous) |
| Mot de passe | Valeur de `POSTGRES_PASSWORD` dans le `.env` |

3. Choisissez la planification, la rétention et gardez le **chiffrement activé**.
4. Lancez une première sauvegarde manuelle pour valider.

Pour créer vous-même un **utilisateur en lecture seule** :

```bash
cd /root/syncstorage-rs
docker compose exec postgres psql -U sync -d syncdb <<'EOF'
CREATE USER databasus WITH PASSWORD 'CHANGE_MOI';
GRANT CONNECT ON DATABASE syncdb TO databasus;
GRANT USAGE ON SCHEMA public TO databasus;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO databasus;
GRANT SELECT ON ALL SEQUENCES IN SCHEMA public TO databasus;
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT SELECT ON TABLES TO databasus;
EOF
```

Remplacez `CHANGE_MOI` par un mot de passe généré, par exemple avec `openssl rand -hex 16`.

> [!IMPORTANT] Sauvegardez la clé de chiffrement de Databasus
> Si vous la perdez, vos sauvegardes deviennent illisibles. Pensez aussi à tester une restauration sur une base vide : une sauvegarde jamais restaurée n'est pas validée.

## 9. Mise à jour

Avec une version pinnée dans `SYNCSERVER_VERSION` :

```bash
cd /root/syncstorage-rs
# modifiez SYNCSERVER_VERSION dans .env, puis :
docker compose pull
docker compose up -d
docker compose ps
```

Faites une sauvegarde Databasus juste avant. Les migrations de base sont exécutées au démarrage (`RUN_MIGRATIONS`).

## Dépannage

| Symptôme | Piste |
|---|---|
| Conteneur `unhealthy` ou qui redémarre | `docker compose logs -f syncserver` et `docker compose logs postgres` |
| `curl` local sans réponse | Vérifier le port (`SYNC_PORT`) et que les deux conteneurs sont `healthy` |
| Erreur 502 depuis le domaine | IP ou port incorrect dans NPM, ou conteneur arrêté |
| Erreur 403 depuis le domaine | Désactiver « Bloquer les exploits courants » pour tester |
| Erreur 413 pendant la synchro | Augmenter `client_max_body_size` (par exemple `20m`) |
| Mauvaise URL de nœud au premier démarrage | Voir la remise à zéro ci-dessous |
| Message « No Authorization header » dans les journaux | Requête reçue sans jeton (par exemple un test avec un navigateur ou `curl`). Sans conséquence |

**Remise à zéro** (supprime toutes les données de synchronisation) :

```bash
cd /root/syncstorage-rs
docker compose down && rm -rf ./postgres-data && docker compose up -d
```

## Sources

- Documentation officielle : [Run your own Sync-1.5 server with Docker](https://mozilla-services.github.io/syncstorage-rs/how-to/how-to-run-with-docker.html)
- Code source : [github.com/mozilla-services/syncstorage-rs](https://github.com/mozilla-services/syncstorage-rs)

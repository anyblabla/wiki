---
title: Stockage objet S3 MinIO pour PeerTube
description: Guide complet pour configurer le stockage objet S3 MinIO avec PeerTube, incluant l'optimisation Nginx Proxy Manager et la sécurisation par politique d'accès JSON personnalisée.
published: true
date: 2026-04-20T15:43:29.876Z
tags: docker, proxmox, sécurité, auto-hébergement, peertube, nginx proxy manager, minio, s3
editor: markdown
dateCreated: 2026-04-20T15:43:29.876Z
---

> **Environnement de référence**
>
>   - PeerTube dans un conteneur LXC Debian + Docker sur Proxmox VE
>   - MinIO autohébergé, accessible via un sous-domaine dédié (`minio.example.be`)
>   - Nginx Proxy Manager (NPM) comme reverse proxy
>   - Un proxy host NPM par service, chacun sur son propre sous-domaine

-----

## 1\. Prérequis

Avant de commencer, s'assurer que :

  - MinIO est opérationnel et accessible depuis le conteneur PeerTube
  - Un bucket dédié est créé (ex : `peertube`)
  - Les credentials MinIO (Access Key / Secret Key) sont disponibles
  - NPM est en place devant MinIO avec un sous-domaine HTTPS valide

> **Important :** La documentation officielle PeerTube précise explicitement que le bucket doit être **public** et disposer de règles CORS autorisant le trafic depuis n'importe quelle origine.

-----

## 2\. Configurer le bucket MinIO

### Accès public sécurisé (Custom Policy)

Dans l'interface MinIO, au lieu d'un accès "Public" générique, appliquez une politique **Custom** sur le bucket `peertube`. Cela permet au navigateur de lire les vidéos sans autoriser n'importe qui à lister tous vos fichiers.

Allez dans **Buckets** \> **peertube** \> **Access Policy**, choisissez **Custom** et collez ce code :

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": { "AWS": ["*"] },
            "Action": ["s3:GetObject"],
            "Resource": ["arn:aws:s3:::peertube/*"]
        }
    ]
}
```

### Règles CORS

Les règles CORS sont gérées par NPM dans cette architecture (voir [section 4](https://www.google.com/search?q=%234-configurer-nginx-proxy-manager)). Il n'est donc pas nécessaire de les configurer directement dans MinIO.

-----

## 3\. Configurer local-production.json

### Structure du bloc `object_storage`

Ajouter ou remplacer le bloc `object_storage` dans `/path/to/config/local-production.json` :

```json
{
  "object_storage": {
    "enabled": true,
    "endpoint": "minio.example.be",
    "region": "us-east-1",
    "force_path_style": true,
    "credentials": {
      "access_key_id": "VOTRE_ACCESS_KEY",
      "secret_access_key": "VOTRE_SECRET_KEY"
    },
    "streaming_playlists": {
      "bucket_name": "peertube",
      "prefix": "hls/",
      "store_live_streams": false
    },
    "web_videos": {
      "bucket_name": "peertube",
      "prefix": "web-videos/"
    },
    "user_exports": {
      "bucket_name": "peertube",
      "prefix": "user-exports/"
    },
    "original_video_files": {
      "bucket_name": "peertube",
      "prefix": "original-video-files/"
    },
    "captions": {
      "bucket_name": "peertube",
      "prefix": "captions/"
    }
  }
}
```

### Points importants

**`force_path_style: true`** est obligatoire avec MinIO. Sans cette option, le client S3 génère des URLs en style `bucket.endpoint` qui ne fonctionnent pas avec MinIO.

**`region`** : MinIO n'a pas de région réelle, mais le client AWS S3 en attend une. La valeur `us-east-1` convient parfaitement.

**`streaming_playlists`** (avec un `s` final) est le nom exact de la clé attendu par PeerTube. Une faute de frappe ici empêcherait silencieusement le stockage HLS de fonctionner.

**`base_url`** est omis intentionnellement. Cette clé n'est utile que si un CDN ou un serveur de cache est placé devant le bucket. Sans CDN, la laisser vide ou absente.

**`max_upload_part`** n'est pas défini ici. La valeur par défaut de 2 GB est suffisante. Cette option n'est à ajuster qu'en cas d'échecs constatés sur les uploads de fichiers très lourds.

### Un seul bucket, des préfixes distincts

Tous les types de contenu partagent ici le même bucket `peertube`, organisés par préfixes (sous-dossiers) :

| Type de contenu | Préfixe dans le bucket |
|---|---|
| Playlists HLS | `hls/` |
| Vidéos web | `web-videos/` |
| Exports utilisateurs | `user-exports/` |
| Fichiers vidéo originaux | `original-video-files/` |
| Sous-titres | `captions/` |

Il est également possible d'utiliser des buckets séparés par type de contenu, en adaptant les `bucket_name` en conséquence.

### Appliquer la configuration

Redémarrer PeerTube après toute modification du fichier :

```bash
docker compose restart peertube
```

-----

## 4\. Configurer Nginx Proxy Manager

Ces blocs sont à renseigner dans l'onglet **Advanced** du proxy host MinIO dans NPM.

### Bloc Location (onglet Advanced du proxy host MinIO)

```nginx
proxy_hide_header X-Powered-By;
add_header Referrer-Policy "no-referrer" always;
add_header X-Frame-Options SAMEORIGIN always;
add_header X-Xss-Protection "1; mode=block" always;
add_header X-Robots-Tag "noindex, noarchive, nofollow" always;
# --- Configuration CORS pour PeerTube ---
add_header 'Access-Control-Allow-Methods' 'GET, HEAD, OPTIONS' always;
add_header 'Access-Control-Allow-Headers' 'DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range,Authorization' always;
add_header 'Access-Control-Expose-Headers' 'Content-Length,Content-Range' always;
if ($request_method = 'OPTIONS') {
    add_header 'Access-Control-Max-Age' 1728000;
    add_header 'Content-Type' 'text/plain; charset=utf-8';
    add_header 'Content-Length' 0;
    return 204;
}
```

> **Attention :** Ne pas ajouter `add_header 'Access-Control-Allow-Origin' '*'` ici. MinIO renvoie déjà ce header lui-même. En le dupliquant, le navigateur reçoit deux valeurs pour ce header et refuse la requête avec une erreur CORS (`header contains multiple values`).

> **Note Nginx :** Les `add_header` définis dans un bloc `if` écrasent ceux définis en dehors dans le même contexte. C'est pourquoi `Access-Control-Allow-Methods` et `Access-Control-Allow-Headers` sont définis avec `always` en dehors du `if`, et le bloc `if` ne contient que les headers spécifiques aux réponses OPTIONS (`Max-Age`, `Content-Type`, `Content-Length`).

### Bloc Global (paramètres globaux NPM)

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
proxy_read_timeout 86400s;
client_max_body_size 0;
# Optimisation pour l'API MinIO / S3
proxy_set_header Host $http_host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_connect_timeout 300;
# Désactiver la mise en cache tampon pour éviter les lenteurs sur les gros flux
proxy_buffering off;
proxy_request_buffering off;
```

`proxy_buffering off` et `proxy_request_buffering off` sont essentiels pour MinIO : sans eux, Nginx tenterait de mettre en tampon les flux entrants et sortants, ce qui provoque des lenteurs et des timeouts sur les transferts de fichiers volumineux.

`client_max_body_size 0` supprime la limite de taille des uploads côté Nginx. PeerTube gère ses propres limites en amont.

> **Isolation des règles CORS :** Ces blocs ne s'appliquent qu'au proxy host MinIO. Chaque service ayant son propre proxy host NPM avec son propre sous-domaine, les règles CORS de MinIO n'ont aucun impact sur les autres services.

-----

## 5\. Migrer les vidéos existantes vers S3

Une fois le stockage S3 opérationnel et validé, il est recommandé de migrer l'intégralité des vidéos existantes vers MinIO plutôt que de conserver un stockage mixte (local + S3).

### Pourquoi migrer tout le contenu

  - **Cohérence** : tout le contenu au même endroit, pas de logique mixte à maintenir
  - **Libérer de l'espace** sur le volume du conteneur PeerTube pour les transcodages temporaires et les logs
  - **Scalabilité** : l'espace MinIO est plus facile à étendre qu'un volume LXC

### Commande de migration

```bash
docker compose exec -u peertube peertube npm run create-move-video-storage-job -- --to-object-storage --all-videos
```

### Suivi de la migration

Surveiller les logs en temps réel pendant la migration :

```bash
docker compose logs -f peertube
```

La migration est **non destructive** et **reprise possible** : si elle est interrompue, relancer la même commande pour traiter les vidéos non encore migrées.

### Migration inverse (S3 vers stockage local)

Si un retour en arrière est nécessaire :

```bash
docker compose exec -u peertube peertube npm run create-move-video-storage-job -- --to-filesystem --all-videos
```

-----

## 6\. Vérification et dépannage

### Vérifier que le stockage S3 est actif

Uploader une nouvelle vidéo et vérifier dans l'interface MinIO que les fichiers apparaissent bien dans le bucket `peertube`, sous les préfixes configurés.

### Erreur CORS : header avec valeurs multiples

**Symptôme** dans la console du navigateur :

```
The 'Access-Control-Allow-Origin' header contains multiple values
'https://peertube.example.be, *', but only one is allowed.
```

**Cause :** Le header \`Access-Control-Allow-Origin' est envoyé deux fois — une fois par MinIO, une fois par NPM.

**Solution :** Retirer `add_header 'Access-Control-Allow-Origin' '*'` du bloc location NPM. MinIO s'en charge seul.

### Erreur HLS : manifestLoadError

Si les vidéos ne se lisent pas et que la console affiche `HLS.js error: networkError - fatal: true - manifestLoadError`, vérifier en priorité les règles CORS (voir ci-dessus) et l'accessibilité publique du bucket MinIO.

### Valider la syntaxe de local-production.json

Avant de redémarrer PeerTube, valider la syntaxe JSON :

```bash
python3 -m json.tool local-production.json > /dev/null && echo "JSON valide"
```

### Problèmes d'upload vers S3

Si des uploads échouent sur des fichiers très volumineux, essayer de réduire la taille des parties multipart (valeur par défaut : 2 GB) :

```json
"object_storage": {
  "max_upload_part": "500MB"
}
```

Ne modifier cette valeur qu'en cas de problème avéré.

-----

*Documentation rédigée à partir d'une mise en production réelle sur une instance PeerTube BlablaLinux autohébergée sur Proxmox VE — Avril 2026.*

> **Auteur** : ce guide est proposé par **Amaury aka BlablaLinux**. Retrouvez l'ensemble de mes services sur [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).
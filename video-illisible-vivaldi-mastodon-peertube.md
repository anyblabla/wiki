---
title: Vidéo illisible sur Mastodon/PeerTube dans Vivaldi (Flatpak) : diagnostic pas à pas et vraie cause
description: Une vidéo Mastodon/PeerTube ne se lance plus, uniquement dans Vivaldi. Post-mortem complet : fausses pistes (NPM, CORS, CSP) et vraie cause (codecs propriétaires Flatpak corrompus).
published: true
date: 2026-09-13T22:34:02.771Z
tags: mastodon, peertube, cors, minio, homelab, nginx-proxy-manager, self-hosting, vivaldi, flatpak, codecs, h264, debug
editor: markdown
dateCreated: 2026-09-13T22:34:02.771Z
---

## Le symptôme

Une vidéo publiée depuis un moment sur mon instance Mastodon (fichier `.mp4` local dans `public/system/media_attachments/`) refusait subitement de se lancer. Le lecteur remontait une erreur générique :

```
NotSupportedError: Failed to load because no supported source was found
```

En creusant, j'ai constaté que le problème touchait **aussi PeerTube** — même symptôme sur des vidéos HLS servies depuis mon instance S3/MinIO. Les deux services n'ont a priori rien en commun côté application, ce qui a orienté l'enquête vers l'infrastructure partagée : mon reverse-proxy central, **Nginx Proxy Manager (NPM)**.

Cet article retrace le cheminement complet de l'investigation — y compris les fausses pistes, qui ont quand même permis de corriger un vrai bug au passage — jusqu'à la cause réelle, complètement inattendue.

## Fausse piste n°1 : le service de fichiers statiques de Mastodon

Ma première hypothèse : le conteneur `mastodon-web` (Puma/Rails) ne serait pas censé servir directement les fichiers statiques de `public/system/`, et NPM, qui proxifie tout vers ce conteneur sans bloc `location` dédié, laisserait Rails appliquer sa Content Security Policy stricte (`SecureHeaders`) même aux fichiers médias :

```
content-security-policy: default-src 'none'; form-action 'none'
```

C'est un vrai axe d'amélioration (dans un déploiement Mastodon standard, c'est le nginx frontal qui devrait intercepter `/system/` avant Puma), mais comme la suite l'a montré, **ce n'était pas la cause du bug** : cette CSP est renvoyée à tous les navigateurs sans distinction, or le problème ne touchait qu'un seul navigateur.

## Fausse piste n°2 (mais vrai bug trouvé au passage !) : en-têtes CORS dupliqués côté NPM

Sur PeerTube, l'inspection des en-têtes de réponse via les DevTools a révélé quelque chose de concret :

```
access-control-expose-headers: Date, Etag, Server, ... X-Amz*, X-Amz*, *
access-control-expose-headers: Content-Length,Content-Range

strict-transport-security: max-age=31536000; includeSubDomains
strict-transport-security: max-age=63072000;includeSubDomains; preload
```

Deux en-têtes envoyés **en double** : une fois par MinIO (le vrai backend S3), une fois rajoutés par NPM via `add_header`, sans `proxy_hide_header` préalable pour retirer la version du backend. En nginx, `add_header` n'écrase jamais un en-tête déjà présent dans la réponse amont — il vient s'empiler. Résultat : un en-tête HTTP dupliqué, que certains navigateurs tolèrent et que d'autres rejettent plus strictement.

**Correctif appliqué** sur le Proxy Host NPM `minio.blablalinux.be` (bloc `location /`, Emplacement personnalisé) :

```nginx
proxy_hide_header X-Powered-By;
proxy_hide_header Access-Control-Expose-Headers;
proxy_hide_header Strict-Transport-Security;
proxy_hide_header X-Xss-Protection;

add_header Referrer-Policy "no-referrer" always;
add_header X-Frame-Options SAMEORIGIN always;
add_header X-Xss-Protection "1; mode=block" always;
add_header X-Robots-Tag "noindex, noarchive, nofollow" always;

# --- Configuration CORS (on laisse MinIO gérer l'Origin) ---
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

La règle à retenir : **toujours faire précéder un `add_header` d'un `proxy_hide_header` sur le même nom d'en-tête**, dès qu'on sait que le backend peut déjà l'envoyer lui-même.

Ce correctif a réellement résolu le problème vidéo... **sur PeerTube**. Sur Mastodon, la vidéo continuait à échouer malgré une configuration NPM propre. Signal que la vraie cause était ailleurs.

## Le vrai indice : ça ne touchait qu'un seul navigateur

En reprenant l'enquête depuis le début, un test croisé simple a tout changé :

| Navigateur | Résultat |
|---|---|
| Firefox (desktop) | ✅ Fonctionne |
| Chrome (desktop) | ✅ Fonctionne |
| Vivaldi (desktop) | ❌ Échoue |
| Vivaldi, navigation privée, sans extension | ❌ Échoue toujours |
| Navigateur mobile (Android) | ✅ Fonctionne |

Chrome et Vivaldi partagent pourtant le même moteur Chromium. Si le problème venait du serveur (CSP, CORS, en-têtes), il toucherait les deux de façon identique. Le fait que seul Vivaldi échoue, même en navigation privée sans extensions, élimine complètement les pistes réseau/serveur pour ce cas précis : **le problème est local au binaire Vivaldi**.

## Le diagnostic définitif : `vivaldi://media-internals`

Chromium expose une page de diagnostic interne très détaillée pour le pipeline vidéo : `vivaldi://media-internals` (accessible aussi via `chrome://media-internals` sur les autres navigateurs Chromium). Ouverte dans un onglet pendant la lecture de la vidéo en échec, elle a livré l'erreur exacte :

```
Warning, FFmpegDemuxer failed to create a valid/supported audio decoder
configuration from muxed stream, config: codec: aac, profile: unknown...

Cannot select VaapiVideoDecoder for video decoding. status=kUnsupportedConfig
Cannot select DecryptingVideoDecoder for video decoding. status=kUnsupportedEncryptionMode
Cannot select VpxVideoDecoder for video decoding. status=kUnsupportedConfig
Cannot select Dav1dVideoDecoder for video decoding. status=kUnsupportedCodec
Cannot select FFmpegVideoDecoder for video decoding. status=kUnsupportedConfig

error: video decoder initialization failed with DecoderStatus::Codes::kUnsupportedConfig
```

Le fichier vidéo est encodé en **H.264 High Profile** avec de l'audio **AAC** — deux codecs sous licence propriétaire. Fait notable : même le décodeur logiciel **FFmpeg**, censé être le filet de sécurité universel, échoue. Ce n'est donc pas juste un problème d'accélération matérielle GPU : Vivaldi n'a tout simplement plus accès aux codecs propriétaires du tout.

## La cause racine : le paquet de codecs Flatpak corrompu

Chromium open source (dont dérive Vivaldi) ne peut pas légalement embarquer nativement les codecs H.264/AAC dans son build — contrairement au Chrome officiel de Google, qui les inclut sous licence propre. La version **Flatpak** de Vivaldi contourne cette limitation en téléchargeant un paquet séparé, stocké hors du sandbox applicatif :

```
~/.var/app/com.vivaldi.Vivaldi/data/vivaldi-extra-libs/
```

Après une mise à jour récente de Vivaldi, ce paquet s'est visiblement retrouvé corrompu ou désynchronisé, expliquant l'apparition soudaine du problème (la vidéo fonctionnait très bien avant).

### Vérification et correctif

Confirmer la version installée :

```bash
flatpak list --app | grep -i vivaldi
```

Forcer le renouvellement du paquet de codecs en le supprimant (Vivaldi le retélécharge automatiquement) :

```bash
flatpak kill com.vivaldi.Vivaldi
mv ~/.var/app/com.vivaldi.Vivaldi/data/vivaldi-extra-libs \
   ~/.var/app/com.vivaldi.Vivaldi/data/vivaldi-extra-libs.bak
flatpak run com.vivaldi.Vivaldi
```

Au premier lancement, Vivaldi confirme le diagnostic dans les logs console :

```
'Proprietary media' support is not installed. Attempting to fix this for the next restart.
```

**Important :** laisser Vivaldi tourner normalement (ne pas l'interrompre avec `Ctrl+C`) le temps qu'il retélécharge le paquet, puis le redémarrer une seconde fois pour que le nouveau paquet soit bien pris en compte. Vérification finale :

```bash
ls ~/.var/app/com.vivaldi.Vivaldi/data/vivaldi-extra-libs/
```

La présence d'un nouveau dossier `media-codecs-8.2` (en plus de l'ancien, horodaté) confirme le téléchargement réussi. La lecture vidéo est repassée au vert immédiatement après.

## En résumé

| Piste explorée | Verdict |
|---|---|
| Service de fichiers statiques Mastodon / CSP Puma | Non responsable de ce bug précis (amélioration possible, non urgente) |
| En-têtes CORS/HSTS dupliqués sur NPM (MinIO) | **Vrai bug, corrigé** — mais distinct du problème vidéo Mastodon |
| Extensions / bloqueur de trackers Vivaldi | Écarté (reproductible en navigation privée) |
| Codecs propriétaires Flatpak corrompus | **Cause racine confirmée et corrigée** |

**Leçon principale :** face à un bug "ça marche ailleurs mais pas ici", tester systématiquement plusieurs navigateurs — y compris ceux qui partagent le même moteur — permet d'isoler très vite si la cause est côté serveur ou côté client. `vivaldi://media-internals` (ou l'équivalent `chrome://media-internals`) est l'outil de diagnostic à connaître pour tout problème de lecture vidéo dans un navigateur basé sur Chromium.

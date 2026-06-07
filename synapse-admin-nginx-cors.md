---
title: Configuration Nginx pour Synapse Admin (CORS)
description: Comment exposer l'API d'administration de Matrix Synapse via NGINX avec les bons en-têtes CORS pour permettre l'utilisation de Synapse Admin depuis un domaine externe.
published: false
date: 2026-06-07T12:59:04.040Z
tags: nginx, npm, administration, cors, matrix, synapse, self-hosting, reverse-proxy
editor: markdown
dateCreated: 2026-06-07T12:53:47.398Z
---

## Pourquoi cette configuration est nécessaire

[Synapse Admin](https://synapse-admin.blablalinux.be) est une interface web qui s'exécute **entièrement dans votre navigateur**. Lorsque vous vous connectez, votre navigateur envoie des requêtes directement vers **votre propre serveur Matrix Synapse** — pas vers l'instance qui héberge l'interface.

Si vous obtenez une erreur de connexion ou une erreur **CORS**, le problème ne vient pas de Synapse Admin. Il vient de la configuration de votre reverse proxy.

Par défaut, Synapse expose son API d'administration sur le port `8008` en local uniquement. Quand un navigateur tente d'y accéder depuis un domaine externe (celui de Synapse Admin), il est bloqué par la politique **Same-Origin** du navigateur. Il faut donc configurer NGINX pour :

1. Proxy-passer les requêtes `/_synapse/admin/` vers votre instance Synapse
2. Ajouter les en-têtes **CORS** appropriés pour autoriser les requêtes cross-origin

> **Note :** Cette configuration est à appliquer sur le proxy host de **votre domaine Matrix** (ex. `matrix.votre-domaine.tld`), pas sur celui de Synapse Admin.

---

## Symptôme — L'erreur que vous voyez

Si votre reverse proxy n'est pas correctement configuré, Synapse Admin affiche le message d'erreur suivant en bas de l'interface :

```
M_INVALID (undefined): Failed to fetch
```

Cette erreur signifie que le navigateur n'a pas pu joindre l'API `/_synapse/admin/` de votre serveur Synapse. Les causes les plus courantes sont :

- Le bloc `location /_synapse/admin/` est absent de la configuration NGINX de votre domaine Matrix
- Les en-têtes CORS ne sont pas définis, et le navigateur bloque la requête cross-origin
- L'IP ou le port de `proxy_pass` ne pointe pas vers votre instance Synapse

---

## Informations de référence

| Élément | Valeur |
|---|---|
| Endpoint à exposer | `/_synapse/admin/` |
| Port Synapse par défaut | `8008` |
| IP dans l'exemple | `192.168.2.109` *(à remplacer)* |
| Compatibilité | NGINX natif et Nginx Proxy Manager |

---

## Bloc de configuration nginx

Ajoutez ce bloc dans la configuration de votre domaine Matrix.

**Avec Nginx Proxy Manager**, collez-le dans l'onglet *"Advanced" → Custom Nginx Configuration* de votre proxy host Matrix.

> ⚠️ Remplacez `192.168.2.109:8008` par l'adresse IP et le port de votre propre instance Synapse. Si Synapse tourne sur la même machine que NGINX, utilisez `127.0.0.1:8008`.

```nginx
# Routage et CORS pour l'API d'administration Synapse (Synapse Admin)
location /_synapse/admin/ {

    # Supprime l'en-tête CORS natif de Synapse pour éviter les doublons
    proxy_hide_header 'Access-Control-Allow-Origin';

    # Gestion du préflight CORS (requête OPTIONS envoyée par le navigateur)
    if ($request_method = 'OPTIONS') {
        add_header 'Access-Control-Allow-Origin'  '$http_origin' always;
        add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS' always;
        add_header 'Access-Control-Allow-Headers' 'Origin, X-Requested-With, Content-Type, Accept, Authorization' always;
        add_header 'Access-Control-Max-Age'    1728000;
        add_header 'Content-Type'              'text/plain; charset=utf-8';
        add_header 'Content-Length'            0;
        return 204;
    }

    # En-têtes CORS pour toutes les autres requêtes
    add_header 'Access-Control-Allow-Origin'  '$http_origin' always;
    add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE, OPTIONS' always;
    add_header 'Access-Control-Allow-Headers' 'Origin, X-Requested-With, Content-Type, Accept, Authorization' always;

    # ⚠️ Remplacez l'IP et le port par ceux de votre Synapse
    proxy_pass         http://192.168.2.109:8008;
    proxy_set_header   Host              $host;
    proxy_set_header   X-Real-IP         $remote_addr;
    proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header   X-Forwarded-Proto $scheme;
}
```

---

## Explication des directives clés

### proxy_hide_header

```nginx
proxy_hide_header 'Access-Control-Allow-Origin';
```

Synapse envoie déjà ses propres en-têtes CORS. Sans cette directive, NGINX en ajouterait un second exemplaire, ce qui provoquerait une erreur dans le navigateur — les doublons d'en-têtes CORS ne sont pas tolérés.

### Bloc if OPTIONS (préflight)

```nginx
if ($request_method = 'OPTIONS') { ... return 204; }
```

Avant toute requête cross-origin, le navigateur envoie d'abord une requête `OPTIONS` pour vérifier les permissions. Ce bloc y répond immédiatement avec un `204 No Content` et les en-têtes d'autorisation, sans solliciter Synapse.

### Access-Control-Allow-Origin avec $http_origin

```nginx
add_header 'Access-Control-Allow-Origin' '$http_origin' always;
```

On autorise l'origine exacte de la requête plutôt qu'un wildcard `*`. C'est obligatoire car les requêtes contenant un en-tête `Authorization` exigent une origine explicite — un wildcard est refusé par le navigateur dans ce contexte.

---

## Vérification

Après avoir appliqué la configuration, rechargez NGINX :

```bash
# NGINX natif
nginx -s reload

# Avec Docker / Nginx Proxy Manager
docker restart npm
```

Rendez-vous ensuite sur [synapse-admin.blablalinux.be](https://synapse-admin.blablalinux.be), renseignez l'URL de votre serveur Matrix ainsi que vos identifiants administrateur (nom d'utilisateur + mot de passe, ou nom d'utilisateur + token). La connexion doit s'établir sans erreur CORS.

---

**Auteur :** ce guide est proposé par Amaury aka BlablaLinux. Retrouvez l'ensemble de mes services sur [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).
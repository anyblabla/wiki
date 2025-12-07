---
title: Les meilleurs Headers pour NPM - Sécurité, Gzip et gestion du Proxy NGINX
description: Ce guide essentiel détaille les configurations NGINX avancées pour NPM. Il couvre l'amélioration de la sécurité via les entêtes HTTP, l'optimisation des performances avec Gzip et la gestion des connexions longues pour les applications modernes.
published: true
date: 2025-12-07T01:30:53.673Z
tags: docker, lxc, nginx, proxy, npm, gzip, performance
editor: markdown
dateCreated: 2025-12-07T01:26:52.363Z
---

En tant que technicien et administrateur système, documenter mes configurations est essentiel. Cette page présente les blocs de code NGINX personnalisés que j'utilise avec **Nginx Proxy Manager (NPM)** pour garantir une **sécurité maximale** et une **performance optimale** sur mes environnements **Linux**.

Cette configuration vise à renforcer la sécurité, optimiser les performances et assurer la bonne transmission des requêtes pour les hôtes gérés par NPM.

-----

## 1\. Bloc `Custom Locations (Advanced)` : Sécurité et Entêtes HTTP

Ce bloc de code doit être inséré dans la section **Custom Locations (Advanced)** de votre hôte NPM. Il permet de modifier les entêtes HTTP échangés avec le client, principalement pour des raisons de sécurité.

### Le Bloc de Code Complet

```nginx
proxy_hide_header X-Powered-By;
add_header Referrer-Policy "no-referrer" always;
add_header X-Frame-Options SAMEORIGIN always;
add_header X-Xss-Protection "1; mode=block" always;
add_header X-Robots-Tag "noindex, noarchive, nofollow" always;
```

![headers-gzip-npm.png](/meilleurs-headers-npm-nginx-securite-gzip/headers-gzip-npm.png)

### Explication Directive par Directive

| Directive | Explication | Objectif |
| :--- | :--- | :--- |
| `proxy_hide_header X-Powered-By;` | **Cache** l'entête `X-Powered-By` généralement ajoutée par le serveur backend (ex: PHP, Node.js). | **Sécurité :** Empêche les attaquants de connaître la technologie et la version du serveur d'application. |
| `add_header Referrer-Policy "no-referrer" always;` | Définit la politique de l'entête `Referrer-Policy` à **`no-referrer`**. | **Sécurité/Vie Privée :** Assure qu'aucune information sur la page d'origine (`referrer`) n'est envoyée lors de la navigation vers d'autres sites. |
| `add_header X-Frame-Options SAMEORIGIN always;` | Définit l'entête `X-Frame-Options` à **`SAMEORIGIN`**. | **Sécurité :** Protège contre les attaques de type **Clickjacking** en empêchant le contenu d'être intégré dans une `<iframe>` d'une autre origine. |
| `add_header X-Xss-Protection "1; mode=block" always;` | Active le filtre anti-XSS (Cross-Site Scripting) du navigateur avec l'option **`mode=block`**. | **Sécurité :** Bloque le rendu des pages si le navigateur détecte une attaque XSS. |
| `add_header X-Robots-Tag "noindex, noarchive, nofollow" always;` | Ajoute l'entête `X-Robots-Tag` demandant aux robots d'indexation de **ne pas indexer** la page. | **Vie Privée/Performance :** Utile pour les services internes. À retirer pour un site public. |

-----

## 2\. Bloc `Settings (Custom Nginx Configuration)` : Optimisation Globale et Proxification

Ce bloc doit être inséré dans la section **Settings** de NPM, dans la zone **Custom Nginx Configuration**. Il définit des paramètres globaux pour l'instance NGINX (compression, buffers) et les entêtes pour la **communication avec le serveur backend**.

### Le Bloc de Code Complet

```nginx
gzip on;
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
proxy_set_header Host $host;
proxy_set_header X-Scheme $scheme;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-Host $http_host;
proxy_set_header Connection $http_connection;
proxy_set_header X-Forwarded-Protocol $scheme;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

![headers-gzip-npm-02.png](/meilleurs-headers-npm-nginx-securite-gzip/headers-gzip-npm-02.png)

### A. Compression GZIP (Performance)

| Directive | Explication | Objectif |
| :--- | :--- | :--- |
| `gzip on;` | **Active la compression Gzip** des réponses. | **Performance :** Réduit la taille des données transférées. |
| `gzip_disable "msie6";` | **Désactive Gzip** spécifiquement pour Internet Explorer 6. | **Compatibilité.** |
| `gzip_vary on;` | Ajoute l'entête `Vary: Accept-Encoding`. | **Cache :** Assure que le cache gère les versions compressées et non compressées. |
| `gzip_proxied any;` | Permet la compression pour toutes les requêtes, même celles qui passent par un proxy. | **Performance :** Assure que Gzip fonctionne en environnement proxy. |
| `gzip_comp_level 6;` | Définit le niveau de compression (de 1 à 9, **6 étant un bon compromis**). | **Performance/CPU.** |
| `gzip_buffers 16 8k;` | Définit le nombre (16) et la taille (8k) des buffers pour la compression. | **Performance :** Gestion optimale de la mémoire. |
| `gzip_http_version 1.1;` | Spécifie la version HTTP minimale pour la compression. | **Compatibilité.** |
| `gzip_types text/plain ... application/xml+rss text/javascript;` | Liste des **types MIME** à compresser. | **Performance.** |

### B. Gestion des Connexions et des En-têtes (Stabilité)

| Directive | Explication | Objectif |
| :--- | :--- | :--- |
| `client_body_buffer_size 512k;` | Définit la taille du buffer pour le corps des requêtes client. | **Stabilité :** Augmente la capacité à gérer de grandes requêtes sans écrire sur le disque. |
| `proxy_read_timeout 86400s;` | Définit le timeout de lecture d'une réponse du serveur backend (**24 heures**). | **Stabilité :** Crucial pour les connexions longues (WebSockets, gros transferts). |
| `client_max_body_size 0;` | Définit la taille maximale autorisée pour le corps de la requête client à **illimitée** (0). | **Fonctionnalité :** Permet l'upload de fichiers de très grande taille. |
| `proxy_set_header Host $host;` | Transmet le nom d'hôte de la requête originale au backend. | **Fonctionnalité :** Indispensable pour que le backend sache quel service répondre. |
| `proxy_set_header X-Real-IP $remote_addr;` | Transmet l'adresse IP réelle du client. | **Fonctionnalité :** Permet aux logs du backend d'afficher l'IP réelle. |
| `proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;` | Ajoute l'IP du client à l'entête `X-Forwarded-For`. | **Fonctionnalité :** Standard pour suivre le chemin d'une requête. |

-----

## 3\. ⚠️ Problèmes courants et solutions

Voici les problèmes que ces configurations peuvent engendrer et les lignes à retirer ou modifier pour les résoudre :

| Problème Rencontré | Cause Potentielle | Solution |
| :--- | :--- | :--- |
| Le service **ne fonctionne plus** après avoir été indexé par Google. | Ligne : `add_header X-Robots-Tag "noindex, noarchive, nofollow" always;` | **Retirer** cette ligne. Elle est destinée aux services privés. Si le service doit être public, supprimez-la. |
| Les **WebSockets** (ex: console web, chat) se déconnectent après une minute. | Ligne : `proxy_read_timeout 86400s;` | Bien que long, si la coupure persiste, la configuration des headers `Connection` et `Upgrade` doit être vérifiée (gérée par NPM, mais peut être écrasée). |
| Les **uploads de fichiers volumineux** sont rejetés. | Ligne : `client_max_body_size 0;` | Cette ligne autorise l'upload. Si le problème persiste, la limite vient du serveur backend (ex: `upload_max_filesize` en PHP). |
| Le backend génère des URLs en HTTP (au lieu de HTTPS). | Lignes : `proxy_set_header X-Scheme $scheme;` ou `proxy_set_header X-Forwarded-Protocol $scheme;` | Ces lignes devraient résoudre le problème. Si non, vérifier si le backend utilise un autre header (ex: `X-Forwarded-Proto`). |

-----

## 💡 Note sur l'Optimisation et le Logiciel Libre

Ces configurations NGINX avancées ne sont pas seulement pour la performance : elles sont essentielles dans l'esprit du **logiciel libre** et du **reconditionnement** que je soutiens. En optimisant la compression Gzip et la gestion des ressources, on s'assure que même le matériel reconditionné fonctionne avec une efficacité maximale. Chaque cycle CPU gagné, chaque paquet de données réduit, contribue à prolonger la vie du matériel et à garantir une expérience utilisateur rapide, même sur des machines modestes, un principe clé que je partage avec le collectif **Emmabuntüs**.

Adopter ces pratiques est une manière de rendre l'informatique reconditionnée à la fois performante et sécurisée.
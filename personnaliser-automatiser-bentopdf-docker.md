---
title: Personnalisation et automatisation de BentoPDF sur Docker
description: Guide pas à pas pour déployer BentoPDF avec Docker. Apprenez à personnaliser le branding, configurer Nginx pour le support WebAssembly et automatiser la maintenance du serveur.
published: true
date: 2026-02-15T02:03:56.024Z
tags: docker, linux, automatisation, bentopdf, wasm, branding
editor: markdown
dateCreated: 2026-02-15T02:03:56.024Z
---

Ce tutoriel détaille la mise en place d'une instance **BentoPDF** personnalisée. Nous allons configurer l'identité visuelle, optimiser les performances et automatiser les mises à jour.

## Étape 1 : Préparation de l'environnement

Commencez par créer le dossier qui accueillera votre projet et placez votre logo à l'intérieur.

```bash
# Création du dossier de travail
mkdir -p ~/bentopdf && cd ~/bentopdf

# Note : Copiez votre fichier logo dans ce dossier avant de continuer
# Exemple : cp /chemin/votre/logo.png ~/bentopdf/mon-logo.png

```

## Étape 2 : Configuration du branding

C'est ici que vous définissez l'identité de votre instance. Dans les fichiers suivants, soyez attentif aux valeurs suivantes :

| Variable | Description |
| --- | --- |
| **VITE_BRAND_NAME** | Le nom qui apparaîtra en haut à gauche de l'interface. |
| **VITE_BRAND_LOGO** | Le nom exact du fichier image de votre logo. |
| **VITE_FOOTER_TEXT** | Le texte de copyright affiché tout en bas de la page. |

> ⚠️ **Règle d'or :** N'utilisez **pas de guillemets** pour les valeurs de ces variables dans les fichiers ci-dessous. Cela garantit qu'elles sont correctement lues par le moteur de build.

## Étape 3 : Création du fichier de configuration Docker

Exécutez cette commande pour générer le fichier `docker-compose.yml`. **Pensez à adapter les lignes marquées d'un commentaire.**

```bash
cat <<EOF > docker-compose.yml
services:
  bentopdf:
    build:
      context: .
      args:
        - VITE_DEFAULT_LANGUAGE=fr
        - VITE_BRAND_NAME=Mon instance BentoPDF # <-- A PERSONNALISER
        - VITE_BRAND_LOGO=mon-logo.png           # <-- NOM DE VOTRE FICHIER LOGO
        - VITE_FOOTER_TEXT=© 2026 Votre Nom      # <-- TEXTE DU PIED DE PAGE
    container_name: bentopdf
    restart: always
    volumes:
      - ./mon-logo.png:/usr/share/nginx/html/mon-logo.png:ro # <-- ADAPTER LE NOM DU LOGO ICI AUSSI
    ports:
      - 8080:8080
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080"]
      interval: 30s
      timeout: 10s
      retries: 3
EOF

```

## Étape 4 : Création du Dockerfile personnalisé

Ce fichier va compiler BentoPDF en y injectant vos réglages. Veillez à ce que les valeurs correspondent à celles du fichier précédent.

```bash
cat <<EOF > Dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
RUN apk add --no-cache git && \\
    git clone https://github.com/alam00000/bentopdf.git . && \\
    npm ci

# Les valeurs ici doivent correspondre à votre compose
ARG VITE_DEFAULT_LANGUAGE=fr
ARG VITE_BRAND_NAME=Mon instance BentoPDF
ARG VITE_BRAND_LOGO=mon-logo.png
ARG VITE_FOOTER_TEXT=© 2026 Votre Nom

ENV VITE_DEFAULT_LANGUAGE=\$VITE_DEFAULT_LANGUAGE
ENV VITE_BRAND_NAME=\$VITE_BRAND_NAME
ENV VITE_BRAND_LOGO=\$VITE_BRAND_LOGO
ENV VITE_FOOTER_TEXT=\$VITE_FOOTER_TEXT

RUN npm run build

FROM nginxinc/nginx-unprivileged:alpine
USER root
RUN mkdir -p /etc/nginx/tmp && chown -R 101:101 /etc/nginx/tmp
USER 101

COPY --from=builder /app/dist /usr/share/nginx/html
COPY --from=builder /app/nginx.conf /etc/nginx/nginx.conf

EXPOSE 8080
CMD ["nginx", "-g", "daemon off;"]
EOF

```

## Étape 5 : Automatisation des mises à jour

Ce script gère la maintenance. Il utilise `git stash` pour protéger vos fichiers `Dockerfile` et `docker-compose.yml` pendant les mises à jour du code source officiel.

```bash
cat <<EOF > ~/update_bentopdf.sh
#!/bin/bash
PROJECT_DIR="\$HOME/bentopdf"

echo "--- Mise à jour de BentoPDF ---"
cd "\$PROJECT_DIR" || exit

git stash
git pull origin main
git stash pop || echo "Aucun changement local à réappliquer."

docker compose up -d --build

# Nettoyage profond pour économiser l'espace disque (SSD/RAM)
docker system prune -f
docker builder prune -f
docker image prune -f

echo "--- Mise à jour et nettoyage terminés ! ---"
EOF

# Activation obligatoire du script
chmod +x ~/update_bentopdf.sh

```

## Étape 6 : Configuration du proxy inverse Nginx

Pour que les outils de traitement PDF (WASM) fonctionnent, ajoutez ces paramètres dans votre configuration Nginx :

```nginx
# En-têtes obligatoires pour WebAssembly
proxy_hide_header Cross-Origin-Embedder-Policy;
proxy_hide_header Cross-Origin-Opener-Policy;
proxy_hide_header Content-Security-Policy;

add_header Cross-Origin-Embedder-Policy "require-corp" always;
add_header Cross-Origin-Opener-Policy "same-origin" always;

# Politique de sécurité et support MIME
client_max_body_size 0;
proxy_read_timeout 86400s;

types {
    application/wasm wasm;
}
gzip on;
gzip_types text/plain text/css application/javascript application/wasm image/svg+xml;

```
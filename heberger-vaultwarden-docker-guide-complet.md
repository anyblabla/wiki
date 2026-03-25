---
title: Héberger Vaultwarden avec Docker (step-by-step)
description: Apprenez à déployer Vaultwarden avec Docker. Ce guide Blabla Linux vous accompagne pour sécuriser votre coffre-fort avec Argon2 et optimiser Nginx Proxy Manager avec des headers de sécurité avancés.
published: true
date: 2026-03-25T15:33:09.488Z
tags: docker, sécurité, auto-hébergement, nginx proxy manager, vaultwarden
editor: markdown
dateCreated: 2026-03-25T15:32:52.898Z
---

## 📖 Introduction
**[Vaultwarden](https://github.com/dani-garcia/vaultwarden)** est une version légère du serveur Bitwarden écrite en Rust. Idéal pour l'auto-hébergement, il consomme peu de ressources et offre toutes les fonctions premium gratuitement (2FA, rapports, organisations). Pour approfondir vos connaissances, vous pouvez consulter le **[GitHub du projet](https://github.com/dani-garcia/vaultwarden)** ainsi que leur **[Wiki officiel](https://github.com/dani-garcia/vaultwarden/wiki)**.

> **Ce que vous allez accomplir :**
> En suivant ce guide, je vais vous accompagner dans le déploiement d'une instance Vaultwarden professionnelle et "durcie". Je ne vais pas me contenter de vous faire lancer un conteneur : je vais vous montrer comment isoler la configuration dans un fichier `.env`, sécuriser l'accès administratif avec un hash Argon2, et configurer un proxy inverse (NPM) avec des en-têtes de sécurité avancés. À la fin, vous disposerez d'un coffre-fort personnel chiffré, accessible en HTTPS, et protégé par une double authentification.

---

## 🛠️ Étape 1 : préparation de l'environnement
Je commence par préparer les dossiers de travail directement dans le répertoire root.

```bash
cd /root
mkdir -p vaultwarden/vw-data
cd vaultwarden
```

---

## 🔑 Étape 2 : sécurisation de l'accès admin (méthode forte)
Pour une sécurité maximale, je ne vais pas utiliser un simple mot de passe. Je vais générer une clé aléatoire complexe que je vais ensuite transformer en empreinte (hash) Argon2.

### 1. Générer la clé secrète
Je génère d'abord une chaîne aléatoire de 48 caractères :
```bash
openssl rand -base64 48
```
> **⚠️ ATTENTION :** Copiez précieusement le résultat (la clé en clair). C'est ce code qui vous sera demandé pour vous connecter à la page `/admin`.

### 2. Créer le hash Argon2
Je passe maintenant cette clé dans l'outil de hashage de Vaultwarden :
```bash
docker run --rm -it vaultwarden/server /vaultwarden hash
```
> **Action :** Collez la clé générée précédemment à l'invite "Password". Copiez ensuite le résultat qui commence par `$$argon2id$$`.

---

## 📝 Étape 3 : création du fichier .env
J'y stocke la configuration complète pour garder un environnement propre.

```bash
nano .env
```

**Contenu à adapter (anonymisé) :**

```text
DOMAIN=https://vw.votre-domaine.be
SIGNUPS_ALLOWED=false
ICONS_SERVICE=internal
# Note : Doublez les symboles '$' pour Docker Compose (ex: $$argon2id$$)
ADMIN_TOKEN=$$argon2id$$v=19$$m=65540,t=3,p=4$$...votre_hash_ici...
```

---

## 🐋 Étape 4 : le fichier docker-compose.yml
```bash
nano docker-compose.yml
```

```yaml
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: always
    env_file: .env
    volumes:
      - ./vw-data:/data
    ports:
      - 8080:80
```

---

## 🚀 Étape 5 : lancement
Je déploie maintenant la solution.

```bash
docker compose up -d
```

---

## 🌐 Étape 6 : Nginx Proxy Manager (NPM)

### 🔹 Onglet Détails et SSL
* **Détails** : Pointez vers l'IP de votre serveur, port `8080`. Activez **Bloquer les exploits courants** et surtout **Prise en charge de Websockets**.
* **SSL** : Force SSL, HTTP/2, HSTS activé, Sous-domaines HSTS.

### 🔹 Onglet Emplacement personnalisé (Custom location)
Ajoutez une localisation `/`, cliquez sur l'engrenage et collez ce bloc :

```nginx
proxy_hide_header X-Powered-By;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "no-referrer" always;
add_header X-Frame-Options SAMEORIGIN always;
add_header X-Xss-Protection "1; mode=block" always;
add_header X-Robots-Tag "noindex, noarchive, nofollow" always;
```

### 🔹 Onglet Avancé (Custom Nginx configuration)
Collez ce second bloc pour les performances et les headers standards :

```nginx
gzip on;
gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
client_max_body_size 0;
proxy_read_timeout 86400s;

proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
```

---

## ⚙️ Étape 7 : configuration post-installation (interface admin)
Accédez à `https://vw.votre-domaine.be/admin`. 

> **💡 Note importante :** Pour vous connecter, vous devez utiliser votre **clé de 48 caractères générée à l'étape 2** (et non le hash Argon2 du fichier .env).

### 🔹 General settings
* **Allow new signups** : Je vous conseille de le **décocher** pour fermer l'accès.
* **Require email verification on signups** : Cochez cette case.
* **Email domain whitelist** : Limitez les inscriptions à vos domaines de confiance.

### 🔹 SMTP email settings
* **Host** : `smtp.gmail.com` | **Port** : `587` | **Security** : `StartTLS`.
* Saisissez votre **Username** et votre **Password** (mot de passe d'application).
* Testez la configuration avec le bouton **"Test SMTP settings"**.

### 🔹 Email 2FA settings
* **Enabled** : Cochez cette case pour activer la 2FA serveur.
* **Setup email 2FA at signup** : Cochez cette case pour forcer la protection dès le départ.

---

## 🔐 Étape 8 : distinguer l'admin de l'utilisateur
Il est impératif de comprendre la différence entre ces deux accès, même si vous utilisez la même adresse email pour les deux :

1.  **L'accès admin (/admin) :** C'est votre centre de commande technique. Il ne possède pas de coffre-fort. Vous y accédez grâce au `ADMIN_TOKEN`.
2.  **Le compte utilisateur :** C'est votre coffre-fort personnel. Pour stocker vos mots de passe, vous devez créer un compte sur la page d'accueil de votre instance.
    * **Pourquoi utiliser le même email ?** Vous pouvez le faire sans conflit car les deux systèmes d'authentification sont totalement distincts. L'email n'est qu'un identifiant ; ce qui vous donne le pouvoir d'administrer, c'est la connaissance du token (clé de 48 caractères).
    * Une fois le compte créé, validez votre adresse mail via le lien reçu et activez immédiatement la **2FA** (TOTP/Email) dans vos paramètres personnels.

---

> **Auteur** : ce guide est proposé par **Amaury aka BlablaLinux**. Retrouvez l'ensemble de mes services sur [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).
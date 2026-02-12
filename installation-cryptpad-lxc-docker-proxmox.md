---
title: Installation de Cryptpad en LXC Docker (méthode BlablaLinux)
description: Apprenez à installer CryptPad sur un conteneur LXC Docker sous Proxmox. Ce guide détaillé optimise les performances des WebSockets et la sécurité pour une instance collaborative robuste et privée.
published: true
date: 2026-02-12T23:05:45.696Z
tags: docker, lxc, proxmox, nginx, npm, compose, cryptpad, office, onlyoffice
editor: markdown
dateCreated: 2026-02-12T22:52:05.131Z
---

Ce guide détaille la préparation spécifique d'un conteneur **LXC Debian** sous **Proxmox** pour accueillir Cryptpad, en mettant l'accent sur la levée des restrictions système nécessaires au bon fonctionnement des WebSockets.

### 🔗 Ressources officielles

* **Site web :** [cryptpad.org](https://cryptpad.org)
* **Projet GitHub :** [cryptpad/cryptpad](https://github.com/cryptpad/cryptpad)
* **Documentation :** [docs.cryptpad.org](https://docs.cryptpad.org/en/)

---

## 1️⃣ Préparation de l'hôte (Proxmox)

Par défaut, un conteneur LXC limite le nombre de fichiers ouverts. Il faut augmenter cette limite au niveau de l'hyperviseur pour éviter les déconnexions et erreurs de WebSockets.

1. Sur l'hôte Proxmox, éditez le fichier de configuration du conteneur :
`nano /etc/pve/lxc/ID_DU_LXC.conf` (remplacez ID par le numéro de votre LXC).
2. Ajoutez les lignes suivantes en bas du fichier :
```text
features: keyctl=1,nesting=1
lxc.prlimit.nofile: 65536

```


3. **Redémarrez complètement le conteneur** depuis l'interface Proxmox (Stop puis Start).

---

## 2️⃣ Optimisation du système (LXC Debian)

Forcez le système Debian à utiliser les limites autorisées par l'hôte en modifiant la configuration PAM.

1. Éditez le fichier des limites : `nano /etc/security/limits.conf`
2. Ajoutez ces lignes avant la fin du fichier (`# End of file`) :
```text
* soft    nofile    65536
* hard    nofile    65536
root soft    nofile    65536
root hard    nofile    65536

```


3. Déconnectez-vous et reconnectez-vous, puis vérifiez avec : `ulimit -n`.
*Le résultat doit être **65536**.*

---

## 3️⃣ Déploiement et arborescence

### A. Création des répertoires

```bash
# Création du répertoire principal
mkdir ~/cryptpad && cd ~/cryptpad

# Création des sous-répertoires pour les volumes
mkdir -p data/blob data/block data/data data/files data/logs customize config onlyoffice-dist onlyoffice-conf

```

### B. Permissions (UID 4001)

Cryptpad utilise un utilisateur interne spécifique (UID 4001). Il faut lui donner la propriété de **tous** les dossiers de travail pour que le conteneur puisse écrire dedans :

```bash
chown -R 4001:4001 config data customize onlyoffice-dist onlyoffice-conf

```

### C. Docker Compose

Créez le fichier `nano docker-compose.yml` :

```yaml
services:
  cryptpad:
    image: "cryptpad/cryptpad:latest"
    container_name: cryptpad
    restart: always

    environment:
      - CPAD_MAIN_DOMAIN=https://cryptpad.exemple.be
      - CPAD_SANDBOX_DOMAIN=https://scryptpad.exemple.be
      - CPAD_CONF=/cryptpad/config/config.js
      - CPAD_INSTALL_ONLYOFFICE=yes

    volumes:
      - ./data/blob:/cryptpad/blob
      - ./data/block:/cryptpad/block
      - ./customize:/cryptpad/customize
      - ./data/data:/cryptpad/data
      - ./data/files:/cryptpad/datastore
      - ./onlyoffice-dist:/cryptpad/www/common/onlyoffice/dist
      - ./onlyoffice-conf:/cryptpad/onlyoffice-conf
      - ./config/config.js:/cryptpad/config/config.js

    ports:
      - "3000:3000"
      - "3001:3001"
      - "3003:3003"

```

---

## 4️⃣ Configuration serveur (`config.js`)

Créez le fichier `nano config/config.js`.
*Lien de référence : [config.example.js (GitHub)](https://github.com/cryptpad/cryptpad/blob/main/config/config.example.js)*

```javascript
// SPDX-FileCopyrightText: 2023 XWiki CryptPad Team <contact@cryptpad.org> and contributors
// SPDX-License-Identifier: AGPL-3.0-or-later

module.exports = {
    /* Origin principale (URL utilisateur) */
    httpUnsafeOrigin: 'https://cryptpad.exemple.be',

    /* Origin sandbox (isolation UI) */
    httpSafeOrigin: "https://scryptpad.exemple.be",

    /* Adresse d'écoute pour Docker */
    httpAddress: '0.0.0.0',

    /* Ports (Laissés commentés si par défaut) */
    //httpPort: 3000,
    //httpSafePort: 3001,
    // websocketPort: 3003,

    /* Sessions */
    otpSessionExpiration: 7*24, // hours

    /* Confidentialité */
    logIP: true,

    /* Clés admin - Ajoutez votre clé ici après création du compte (voir étape 5) */
    adminKeys: [
        // "[nom@cryptpad.exemple.be/VOTRE_CLE_PUBLIQUE=]",
    ],

    /* Stockage et rétention */
    inactiveTime: 90, // jours
    archiveRetentionTime: 15,
    accountRetentionTime: 365,
    disableIntegratedEviction: true,

    /* Taille maximale d'upload (50 Mo) */
    maxUploadSize: 50 * 1024 * 1024,

    /* Volumes de base de données (chemins internes) */
    filePath: './datastore/',
    archivePath: './data/archive',
    pinPath: './data/pins',
    taskPath: './data/tasks',
    blockPath: './block',
    blobPath: './blob',
    blobStagingPath: './data/blobstage',
    decreePath: './data/decrees',
    logPath: './data/logs',

    /* Débogage */
    logToStdout: false,
    logLevel: 'info',
    logFeedback: false,
    verbose: false,
    installMethod: 'docker-blablalinux',
};

```

---

## 5️⃣ Activation de l'administration

### A. Récupérer la clé publique (clé de signature)

1. Lancez l'instance : `docker compose up -d`.
2. Rendez-vous sur votre instance via le navigateur.
3. **Créez un compte utilisateur** et connectez-vous.
4. Allez dans **Paramètres** (icône en haut à droite) > onglet **Compte**.
5. Cherchez la section **Clé de signature publique** et copiez la chaîne complète.

### B. Inscrire l'administrateur

Éditez votre fichier `config/config.js` et insérez votre clé dans le tableau `adminKeys` :

```javascript
    /* =====================
     * Administration
     * ===================== */
    adminKeys: [
        "[votre-nom@cryptpad.exemple.be/ABCDEF123456789...=]",
    ],

```

### C. Appliquer la modification

Redémarrez le conteneur pour valider vos droits :

```bash
docker compose restart cryptpad

```

---

## 6️⃣ Personnalisation de l'interface (`application_config.js`)

Pour modifier l'interface, créez le fichier `nano customize/application_config.js`.
*Lien de référence interne : [application_config_internal.js (GitHub)](https://github.com/cryptpad/cryptpad/blob/main/www/common/application_config_internal.js)*

```javascript
(() => {
    const factory = (AppConfig) => {
        // Activer les applications en "accès anticipé"
        AppConfig.enableEarlyAccess = true;

        // Masquer les applications non utilisées
        AppConfig.hiddenTypes = ['drive', 'teams', 'contacts', 'todo', 'file', 'accounts', 'calendar', 'poll', 'convert'];

        // Langues disponibles
        AppConfig.availableLanguages = ['en', 'fr'];

        // Liens personnalisés (Anonymisez-les avec vos propres URLs)
        // Lien vers votre politique de confidentialité
        AppConfig.privacy = 'https://exemple.be/politique-de-confidentialite/';
        // Lien vers vos conditions d'utilisation
        AppConfig.terms = 'https://exemple.be/conditions-d-utilisation/';
        // Lien vers votre page de statut (Uptime)
        AppConfig.status = 'https://status.exemple.be';

        // Code source (obligatoire AGPL)
        AppConfig.source = true;

        // Langues supportées par l'admin
        AppConfig.supportLanguages = [ 'fr', 'en' ];

        return AppConfig;
    };
    if (typeof(module) !== 'undefined' && module.exports) {
        module.exports = factory(require('../www/common/application_config_internal.js'));
    } else if ((typeof(define) !== 'undefined' && define !== null) && (define.amd !== null)) {
        define(['/common/application_config_internal.js'], factory);
    }
})();

```

---

## 7️⃣ Configuration de Nginx Proxy Manager (NPM)

### A. Détails et SSL

* **Détails** : IP du LXC, Port `3000`, **Prise en charge des Websockets : ON**.
* **SSL** : Force SSL : **ON**, HSTS activé : **ON**.

### B. Emplacement personnalisé (`/`)

* **Emplacement** : `/` | **Schéma** : `http` | **Port** : `3000`
* **Configuration** :
```nginx
proxy_hide_header X-Powered-By;
add_header Referrer-Policy "no-referrer" always;

```



### C. Onglet avancé

```nginx
# Optimisations Gzip
gzip on;
gzip_min_length 1000;
gzip_disable "msie6";
gzip_vary on;
gzip_proxied any;
gzip_comp_level 6;
gzip_buffers 16 8k;
gzip_http_version 1.1;
gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

# Paramètres de transfert et délais d'attente
client_body_buffer_size 512k;
proxy_read_timeout 86400s;
client_max_body_size 0;

# En-têtes standards
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header Connection upgrade;
proxy_set_header Upgrade $http_upgrade;

# Routage spécifique des Websockets vers le port 3003
location ^~ /cryptpad_websocket {
    proxy_pass http://192.168.2.121:3003;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection upgrade;
}

```
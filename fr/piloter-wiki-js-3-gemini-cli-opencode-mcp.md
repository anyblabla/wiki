---
title: >-
  Piloter Wiki.js 3 en langage naturel avec Gemini CLI, OpenCode et le protocole
  MCP
description: >-
  Guide complet pour connecter Gemini CLI et OpenCode à l'API MCP de Wiki.js 3.
  Installation, configuration Nginx Proxy Manager, jetons PAT et résolution des
  erreurs de quota.
published: false
date: '2026-11-11T21:20:33.975Z'
tags:
  - gemini-cli
  - opencode
  - mcp
  - nginx-proxy-manager
  - wiki-js
  - debian
  - auto-hebergement
editor: markdown
dateCreated: '2026-11-11T21:19:17.944Z'
---

Le protocole **MCP (Model Context Protocol)** permet de connecter un assistant IA directement à vos applications et outils auto-hébergés. En activant l'API MCP sur votre instance **Wiki.js 3**, vous pouvez effectuer des recherches, lire des pages ou explorer la structure de votre documentation directement en langage naturel depuis votre terminal Linux ou votre environnement de travail, que ce soit via **Gemini CLI** ou **OpenCode**.

Ce guide détaille l'installation complète des outils, la génération des jetons d'accès, les ajustements réseau nécessaires sur Nginx Proxy Manager pour gérer le *streaming* HTTP, ainsi que la configuration fine pour éviter les blocages de quota d'API.

---

## 1. Prérequis

* Un système Linux (Debian, Ubuntu ou dérivé) avec **Node.js** (version 18 ou supérieure) et **npm** installés.
* Un serveur **Wiki.js v3** accessible en HTTPS (ex. `https://wiki.exemple.com`).
* Un reverse proxy **Nginx Proxy Manager (NPM)** configuré devant votre Wiki.js.
* Un compte Google avec une clé d'API issue de **Google AI Studio** (obtenue sur [aistudio.google.com/api-keys](https://aistudio.google.com/api-keys)).
* *(Recommandé)* Un abonnement **Gemini AI Plus** ou l'activation du mode *Pay-as-you-go* sur votre projet Google AI Studio pour bénéficier de limites d'utilisation confortables et éviter les blocages quotidiens du mode gratuit.

---

## 2. Étape 1 : Obtenir son jeton d'accès (PAT) sur Wiki.js 3

1. Connectez-vous à votre instance Wiki.js 3 avec votre compte administrateur ou utilisateur.
2. Accédez à votre profil utilisateur, puis dans la section **Jetons d'accès personnels** (*Personal Access Tokens* / PAT).
3. Générez un nouveau jeton en cochant la portée (*scope*) **`mcp`**.
4. Copiez la clé JWT générée et conservez-la précieusement.

---

## 3. Étape 2 : Configurer le reverse proxy (Nginx Proxy Manager)

Le protocole MCP s'appuie sur le *streaming* HTTP. Par défaut, la mise en mémoire tampon (*proxy buffering*) de Nginx bloque le flux de données en continu, ce qui provoque un blocage infini des CLI IA sur l'état `Thinking...`.

Pour corriger ce comportement dans **Nginx Proxy Manager** :

1. Éditez l'hôte proxy correspondant à votre instance Wiki.js.
2. Dans l'onglet **Details**, cochez la case **Websockets Support**.
3. Allez dans l'onglet **Custom Locations** (*Emplacements personnalisés*) et ajoutez une route dédiée :
   * **Location :** `/_mcp`
   * **Scheme / IP / Port :** L'adresse réseau de votre conteneur Wiki.js (ex. `http://192.168.1.50:80`).
4. Dans le bloc de configuration Nginx personnalisée de cet emplacement, collez les directives suivantes :

    proxy_buffering off;
    proxy_cache off;
    gzip off;
    chunked_transfer_encoding on;

    proxy_read_timeout 600s;
    proxy_send_timeout 600s;

5. Sauvegardez la configuration.

---

## 4. Étape 3 : Installation et configuration de Gemini CLI

### A. Installation de Gemini CLI
Si Gemini CLI n'est pas encore présent sur votre système, installez-le globalement via `npm` :

    sudo npm install -g @google/gemini-cli

Vérifiez également la présence de `ripgrep` (utilitaire optionnel pour la recherche locale de fichiers) :

    sudo apt update && sudo apt install -y ripgrep

### B. Obtenir et configurer la clé API
1. Rendez-vous directement sur la page de gestion des clés : [https://aistudio.google.com/api-keys](https://aistudio.google.com/api-keys).
2. Générez une nouvelle clé d'API (pensez à l'associer à un projet disposant des accès requis ou d'un abonnement Gemini AI Plus / compte facturable pour débloquer les limites d'utilisation).
3. Définissez votre clé d'API dans votre environnement Shell :

    export GEMINI_API_KEY="VOTRE_CLE_API_GOOGLE_AI_STUDIO"

*(Pour rendre cette variable permanente, ajoutez cette ligne dans votre fichier `~/.bashrc` ou `~/.profile`).*

### C. Déclaration du serveur MCP
Le fichier de configuration de Gemini CLI réside dans `~/.gemini/settings.json`. Éditez-le :

    nano ~/.gemini/settings.json

Ajoutez ou adaptez la structure JSON suivante :

    {
      "mcpServers": {
        "wiki": {
          "url": "https://wiki.exemple.com/_mcp",
          "type": "http",
          "headers": {
            "Authorization": "Bearer VOTRE_JETON_PAT_ICI"
          }
        }
      },
      "security": {
        "auth": {
          "selectedType": "gemini-api-key"
        }
      },
      "general": {
        "plan": {
          "modelRouting": false
        }
      },
      "model": {
        "name": "gemini-3.6-flash"
      }
    }

> **Remarque importante sur l'option `modelRouting` :**  
> L'option `"modelRouting": false` empêche le CLI de basculer automatiquement sur des modèles Pro lors des étapes de planification. Sur les comptes API gratuits sans abonnement étendu, les modèles Pro dépassent rapidement le quota quotidien (erreur 429). L'utilisation d'un modèle Flash tel que `gemini-3.6-flash` garantit un fonctionnement fluide.

### D. Validation dans Gemini CLI
Lancez Gemini CLI :

    gemini

Lors de la première sollicitation d'une recherche, validez l'exécution des outils du serveur `wiki` en sélectionnant l'option **`3. Allow all server tools for this session`**.

---

## 5. Étape 4 : Installation et configuration d'OpenCode

**OpenCode** est un environnement d'agent IA interactif disponible sous forme d'application Desktop Linux ou d'outil en ligne de commande (CLI Terminal).

### A. Installation d'OpenCode
Selon vos préférences et votre distribution, plusieurs méthodes d'installation sont disponibles depuis la plateforme officielle [opencode.ai/product](https://opencode.ai/product)[cite: 13] :

#### Option 1 : Application Desktop Linux
Téléchargez directement le paquet ou l'exécutable Linux depuis le bouton **Download for Linux** sur la page produit[cite: 13].

#### Option 2 : Terminal CLI (Node.js / npm)
Si vous préférez utiliser l'interface terminal d'OpenCode[cite: 13], installez le paquet globalement avec `npm`[cite: 13] :

    npm install -g @opencode/cli

*(Note : Vous pouvez également l'installer via le script shell automatisé `curl -fsSL https://opencode.ai/v2/install | bash` ou via d'autres gestionnaires de paquets tels que Bun, Brew ou Paru sur Arch Linux[cite: 13]).*

---

### B. Configuration globale d'OpenCode avec le MCP Wiki.js
Éditez le fichier de configuration d'OpenCode dans votre dossier personnel :

    nano ~/.config/opencode/opencode.json

Déclarez le serveur MCP sous la clé `mcpServers` :

    {
      "mcpServers": {
        "wiki": {
          "type": "http",
          "url": "https://wiki.exemple.com/_mcp",
          "headers": {
            "Authorization": "Bearer VOTRE_JETON_PAT_ICI"
          }
        }
      }
    }

*(Note : Si vous souhaitez limiter cette intégration à un projet spécifique, vous pouvez également déposer ce même fichier `opencode.json` à la racine de votre dossier de travail).*

---

### C. Validation dans OpenCode
1. Démarrez OpenCode dans votre terminal :

    opencode

2. Dans la session interactive, vérifiez la détection du serveur MCP en tapant la commande :

    /mcp list

3. OpenCode doit afficher la confirmation : `Configured MCP Servers: - wiki (HTTP): https://wiki.exemple.com/_mcp`[cite: 11].

---

## 6. Exemples de requêtes en langage naturel

Une fois les deux outils configurés, vous pouvez poser vos questions directement sans syntaxe particulière :

* *"Quelles sont les pages récentes de mon wiki concernant Nginx ?"*
* *"Affiche-moi le contenu de la page d'installation de Nginx Proxy Manager."*
* *"Fais un résumé des étapes de sécurisation avec Fail2Ban d'après mon wiki."*

---

## 7. Guide de dépannage

### 1. Gemini CLI reste bloqué sur `Thinking...`
* **Cause :** Nginx met en tampon (*buffer*) la réponse HTTP en streaming.
* **Solution :** Assurez-vous d'avoir ajouté `proxy_buffering off;` et `gzip off;` sur l'emplacement `/_mcp` dans NPM.

### 2. Erreur `HTTP 429 / TerminalQuotaError` (Quota dépassé)
* **Cause :** Dépassement du nombre maximal de requêtes quotidiennes attribuées au modèle principal sur le Free Tier.
* **Solutions :**
  1. Passez sur un abonnement Gemini AI Plus ou activez la facturation Pay-as-you-go sur Google AI Studio.
  2. Vérifiez que `Plan Model Routing` est réglé sur `false` (dans `/settings` du CLI ou dans `settings.json`).
  3. Passez sur le modèle `gemini-3.6-flash` ou `gemini-3.5-flash-lite`.
  4. Créez un projet secondaire sur [aistudio.google.com/api-keys](https://aistudio.google.com/api-keys) pour générer une nouvelle clé d'API si le quota d'un projet est temporairement atteint.

### 3. OpenCode indique `No MCP tools or servers connected`
* **Cause :** Le fichier `opencode.json` n'est pas localisé au bon endroit ou comporte une erreur de syntaxe JSON.
* **Solution :** Vérifiez le chemin `~/.config/opencode/opencode.json` et validez la structure JSON à l'aide d'un linter.

---

> **Author**: this guide is proposed by **Amaury aka BlablaLinux**. Find all my services on [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).

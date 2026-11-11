---
title: Piloter Wiki.js 3 avec Gemini CLI, OpenCode et le protocole MCP
description: >-
  Guide complet pour connecter Gemini CLI et OpenCode à l'API MCP de Wiki.js 3.
  Installation, configuration Nginx Proxy Manager, jetons PAT et résolution des
  erreurs de quota.
published: true
date: '2026-11-11T23:29:50.375Z'
tags:
  - opencode
  - mcp
  - debian
  - gemini cli
  - nginx proxy manager
  - wikijs
  - auto hébergement
editor: markdown
dateCreated: '2026-11-11T21:19:17.944Z'
---

Le protocole **MCP (Model Context Protocol)** permet de connecter un assistant IA directement à vos applications et outils auto-hébergés. En activant le serveur MCP intégré de **Wiki.js 3**, vous pouvez effectuer des recherches, lire des pages ou explorer la structure de votre documentation en langage naturel depuis votre terminal Linux, que ce soit via **Gemini CLI** ou **OpenCode**.

Ce guide détaille l'activation du MCP dans Wiki.js, la génération du jeton d'accès, les ajustements nécessaires sur Nginx Proxy Manager pour gérer le *streaming* HTTP, l'installation des deux outils, ainsi que la configuration pour éviter les blocages de quota d'API.

---

## 1. Prérequis

* Un système Linux (Debian, Ubuntu ou dérivé) avec **Node.js** (version 18 ou supérieure) et **npm** installés.
* Un serveur **Wiki.js v3** accessible en HTTPS (ex. `https://wiki.exemple.com`).
* Un reverse proxy **Nginx Proxy Manager (NPM)** configuré devant votre Wiki.js.
* Un compte Google avec une clé d'API issue de **Google AI Studio** (à créer sur [aistudio.google.com/api-keys](https://aistudio.google.com/api-keys)).
* *(Recommandé)* Activer la facturation (*Pay-as-you-go*) sur le projet Google associé à votre clé, afin d'obtenir des limites confortables et d'éviter les blocages quotidiens du niveau gratuit.

---

## 2. Étape 1 : Activer le MCP et obtenir son jeton d'accès (PAT) sur Wiki.js 3

Le serveur MCP de Wiki.js 3 est exposé sur le chemin `/_mcp`, mais il est **désactivé par défaut**.

1. Connectez-vous à votre instance Wiki.js 3 avec un compte administrateur.
2. Dans la **zone d'administration**, ouvrez la section **MCP** et activez le serveur MCP. Si vous souhaitez que l'IA puisse modifier des pages, accordez l'accès en écriture ; sinon, laissez l'accès en lecture seule.
3. Accédez à votre profil utilisateur, puis à la section **Jetons d'accès personnels** (*Personal Access Tokens* / PAT).
4. Générez un nouveau jeton en cochant la portée (*scope*) **`mcp`**.
5. Copiez le jeton généré (il n'est affiché qu'une seule fois) et conservez-le précieusement.

Pour ne pas écrire le jeton en clair dans les fichiers de configuration, exportez-le dans votre environnement Shell (à ajouter à `~/.bashrc` pour le rendre permanent) :

```bash
export WIKI_MCP_TOKEN="VOTRE_JETON_PAT_ICI"
```

---

## 3. Étape 2 : Configurer le reverse proxy (Nginx Proxy Manager)

Le protocole MCP s'appuie sur le *streaming* HTTP. Par défaut, la mise en mémoire tampon (*proxy buffering*) de Nginx bloque le flux de données, ce qui provoque un blocage infini des CLI IA sur l'état `Thinking...`.

Pour corriger ce comportement dans **Nginx Proxy Manager** :

1. Éditez l'hôte proxy correspondant à votre instance Wiki.js.
2. Allez dans l'onglet **Custom Locations** (*Emplacements personnalisés*) et ajoutez une route dédiée :
   * **Location :** `/_mcp`
   * **Scheme / IP / Port :** l'adresse réseau de votre conteneur Wiki.js (ex. `http://192.168.1.50:80`).
3. Cliquez sur l'icône d'engrenage de cet emplacement pour ouvrir son bloc de configuration Nginx personnalisée, et collez les directives suivantes :

```nginx
proxy_buffering off;
proxy_cache off;
gzip off;
chunked_transfer_encoding on;

proxy_read_timeout 600s;
proxy_send_timeout 600s;
```

4. Sauvegardez la configuration.

> **Note :** cochez également **Websockets Support** dans l'onglet **Details** de l'hôte proxy si votre Wiki.js l'utilise déjà pour l'édition collaborative ; ce n'est toutefois pas ce réglage qui corrige le blocage sur `Thinking...`, c'est la désactivation du buffering sur `/_mcp`.

---

## 4. Étape 3 : Installation et configuration de Gemini CLI

### A. Installation de Gemini CLI
Installez-le globalement via `npm` :

```bash
sudo npm install -g @google/gemini-cli
```

Vérifiez également la présence de `ripgrep` (utilitaire optionnel pour la recherche locale de fichiers) :

```bash
sudo apt update && sudo apt install -y ripgrep
```

### B. Configurer la clé API
1. Rendez-vous sur la page de gestion des clés : [https://aistudio.google.com/api-keys](https://aistudio.google.com/api-keys).
2. Générez une nouvelle clé d'API, en l'associant à un projet dont la facturation est activée si vous voulez dépasser les limites du niveau gratuit.
3. Définissez-la dans votre environnement Shell :

```bash
export GEMINI_API_KEY="VOTRE_CLE_API_GOOGLE_AI_STUDIO"
```

*(Pour rendre ces variables permanentes, ajoutez-les dans votre fichier `~/.bashrc` ou `~/.profile`.)*

### C. Déclaration du serveur MCP
Le fichier de configuration de Gemini CLI se trouve dans `~/.gemini/settings.json`. Éditez-le :

```bash
nano ~/.gemini/settings.json
```

Ajoutez ou adaptez la structure JSON suivante :

```json
{
  "mcpServers": {
    "wiki": {
      "httpUrl": "https://wiki.exemple.com/_mcp",
      "headers": {
        "Authorization": "Bearer $WIKI_MCP_TOKEN"
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
```

> **Remarques :**
> * `httpUrl` désigne un serveur MCP en *Streamable HTTP* (`url` seul correspond à l'ancien transport SSE). La forme `"url": "...", "type": "http"` générée par `gemini mcp add --transport http` fonctionne aussi.
> * Les variables `$NOM` sont développées par Gemini CLI dans `settings.json`, y compris dans `headers` : le jeton reste ainsi dans votre environnement et non dans le fichier.
> * L'option `"modelRouting": false` empêche le CLI de basculer automatiquement sur des modèles Pro lors des étapes de planification. Sur les comptes API gratuits, les modèles Pro dépassent rapidement le quota quotidien (erreur 429).
> * `gemini-3.6-flash` et `gemini-3.5-flash-lite` sont toujours disponibles, mais ce ne sont plus les modèles Flash les plus récents : consultez la [liste officielle des modèles](https://ai.google.dev/gemini-api/docs/models) et adaptez le nom si vous souhaitez un modèle plus récent.

Vous pouvez aussi ajouter le serveur en une commande :

```bash
gemini mcp add --transport http --scope user wiki https://wiki.exemple.com/_mcp
```

*(puis ajouter l'en-tête `Authorization` dans `settings.json`, ou via l'option `--header` selon la version).*

### D. Validation dans Gemini CLI
Lancez Gemini CLI :

```bash
gemini
```

Tapez `/mcp` pour vérifier que le serveur `wiki` est bien connecté. Lors de la première recherche, validez l'exécution des outils du serveur `wiki` en sélectionnant l'option **`3. Allow all server tools for this session`**.

---

## 5. Étape 4 : Installation et configuration d'OpenCode

**OpenCode** est un agent IA open source disponible sous forme d'application Desktop Linux ou d'outil en ligne de commande (CLI).

### A. Installation d'OpenCode
Plusieurs méthodes sont disponibles (voir [opencode.ai](https://opencode.ai)) :

#### Option 1 : Script d'installation (recommandé)

```bash
curl -fsSL https://opencode.ai/install | bash
```

Le binaire est installé dans `~/.opencode/bin` par défaut (ou dans `~/bin` / `$XDG_BIN_DIR` s'ils existent) ; le script ajoute le chemin à votre `PATH`. Ouvrez un nouveau terminal pour le prendre en compte.

#### Option 2 : npm
Attention : le paquet npm s'appelle **`opencode-ai`** (et non `opencode`).

```bash
npm install -g opencode-ai@latest
```

#### Option 3 : Application Desktop
Téléchargez le paquet `.deb`, `.rpm` ou l'AppImage depuis [opencode.ai/download](https://opencode.ai/download).

*(D'autres gestionnaires sont disponibles : Homebrew, `paru -S opencode-bin` sur Arch Linux, mise, etc.)*

---

### B. Connecter OpenCode à un modèle
OpenCode a besoin d'un fournisseur de modèle. Lancez `opencode`, tapez `/connect`, choisissez **Google** puis collez votre clé d'API Google AI Studio, et sélectionnez un modèle avec `/models`.

---

### C. Configuration globale d'OpenCode avec le MCP Wiki.js
Éditez le fichier de configuration d'OpenCode :

```bash
mkdir -p ~/.config/opencode
nano ~/.config/opencode/opencode.json
```

Dans OpenCode, les serveurs MCP se déclarent sous la clé **`mcp`** (et non `mcpServers`), avec le type `remote` :

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "wiki": {
      "type": "remote",
      "url": "https://wiki.exemple.com/_mcp",
      "enabled": true,
      "oauth": false,
      "headers": {
        "Authorization": "Bearer {env:WIKI_MCP_TOKEN}"
      }
    }
  }
}
```

> **Notes :**
> * `"oauth": false` désactive la détection OAuth automatique, inutile ici puisque l'authentification se fait par jeton.
> * `{env:WIKI_MCP_TOKEN}` lit la variable d'environnement définie à l'étape 1 (la syntaxe `$VAR` n'est pas développée dans le JSON d'OpenCode).
> * Pour limiter l'intégration à un projet précis, placez ce même fichier `opencode.json` à la racine du dossier de travail.
> * Sur la branche **OpenCode V2**, les serveurs se déclarent sous `mcp.servers` au lieu de directement sous `mcp` ; reportez-vous à la documentation de votre version si la configuration ci-dessus n'est pas reconnue.

---

### D. Validation dans OpenCode
1. Vérifiez la déclaration du serveur depuis le terminal :

```bash
opencode mcp list
```

2. Démarrez OpenCode (`opencode`) et posez une question portant sur votre wiki : si le serveur est bien connecté, l'agent utilisera les outils du serveur `wiki`.

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
* **Solution :** assurez-vous d'avoir ajouté `proxy_buffering off;` et `gzip off;` sur l'emplacement `/_mcp` dans NPM.

### 2. Erreur `HTTP 429 / TerminalQuotaError` (quota dépassé)
* **Cause :** dépassement du nombre maximal de requêtes quotidiennes attribuées au modèle principal sur le niveau gratuit.
* **Solutions :**
  1. Activez la facturation Pay-as-you-go sur le projet Google associé à votre clé.
  2. Vérifiez que `Plan Model Routing` est réglé sur `false` (dans `/settings` du CLI ou dans `settings.json`).
  3. Passez sur un modèle Flash, par exemple `gemini-3.6-flash` ou `gemini-3.5-flash-lite`.
  4. Créez un projet secondaire sur [aistudio.google.com/api-keys](https://aistudio.google.com/api-keys) pour générer une nouvelle clé si le quota d'un projet est temporairement atteint.

### 3. OpenCode indique `No MCP tools or servers connected`
* **Cause :** le fichier `opencode.json` n'est pas au bon endroit, comporte une erreur de syntaxe JSON, ou déclare le serveur sous `mcpServers` au lieu de `mcp`.
* **Solution :** vérifiez le chemin `~/.config/opencode/opencode.json`, la clé `mcp`, et validez le JSON à l'aide d'un linter (`python3 -m json.tool ~/.config/opencode/opencode.json`).

### 4. Erreur `401 Unauthorized` ou `403 Forbidden` sur `/_mcp`
* **Cause :** jeton absent, expiré ou sans la portée `mcp`, ou variable `WIKI_MCP_TOKEN` non définie dans le terminal où l'outil est lancé.
* **Solution :** vérifiez `echo "$WIKI_MCP_TOKEN"`, régénérez un jeton avec la portée `mcp` si besoin.

### 5. Erreur `404 Not Found` sur `/_mcp`
* **Cause :** le serveur MCP n'est pas activé dans la zone d'administration de Wiki.js 3 (désactivé par défaut), ou la *Custom Location* de NPM pointe vers la mauvaise adresse.
* **Solution :** activez-le dans **Administration > MCP** et contrôlez l'IP/port de l'emplacement `/_mcp`.

---

> **Auteur** : ce guide est proposé par **Amaury alias BlablaLinux**. Retrouvez tous mes services sur [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).

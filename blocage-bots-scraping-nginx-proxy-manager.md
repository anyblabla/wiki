---
title: Blocage des bots de scraping avec Nginx Proxy Manager
description: Protégez vos services web contre le scraping et le spam. Ce guide explique comment filtrer les requêtes par mots-clés et User-Agent dans Nginx Proxy Manager, tout en épargnant le réseau local.
published: true
date: 2026-05-29T20:52:21.847Z
tags: nginx, npm, sécurité, sysadmin, anti-bot, scraping
editor: markdown
dateCreated: 2026-05-29T20:52:21.847Z
---

Face au bruit de fond du web, aux outils d'IA et aux scrapers industriels, utiliser un simple pare-feu ne suffit plus. Beaucoup de bots passent par des VPN ou des proxys résidentiels pour contourner les blocages géographiques.

La solution ? Filtrer le trafic à la volée directement dans Nginx en analysant **ce que les bots demandent** (les mots-clés) et **qui ils prétendent être** (leur User-Agent), tout en protégeant les accès légitimes de notre réseau local.

---

## Le bloc de configuration personnalisé

Voici le fichier complet à créer (par exemple sous le nom `searxng-custom-block.conf`). C'est ce script qui va analyser chaque requête :

```nginx
# --- ANTI-BOTS PAR MOTS-CLÉS (LinkedIn, Telegram, Spam Russe, Scrapers B2B, Requêtes Chinoises & Footprint Roc Villa) ---
# Ajout des termes commerciaux, blocs Unicode chinois et signatures de spam pour bloquer le scraping via VPN
if ($request_uri ~* "(linkedin\.com|t\.me|2025.*Present|избавиться|порчи|маету|pet\+dry\+food|private\+label|wholesale|distributor|importer|Roc\+Villa|mariage\+en\+Italie|solas|tedarik%C3%A7isi|%E[4-9]%[0-9A-F]{2}%[0-9A-F]{2})") {
    set $block 1;
}

# --- ANTI-BOTS PAR USER-AGENT (Scripts de scraping - Sauf searx.space, watchtower et shoutrrr) ---
if ($http_user_agent ~* "^(?!.*(searx|watchtower|shoutrrr)).*(python|go-http-client|axios|curl|wget|scrapy|headless|feedly|llamaindex|langchain|http\.rb|faraday)") {
    set $block 1;
}

# --- EXCEPTION POUR LE RÉSEAU LOCAL ET DOCKER ---
# Si l'IP est locale (LAN ou Docker), on force $block à 0, peu importe le User-Agent ou la requête !
if ($allowed_ip = "yes") {
    set $block 0;
}

```

---

## Décorticage des règles de filtrage

Le fichier de configuration repose sur une logique simple : on initialise une variable (par exemple `$block`), on passe au tamis les requêtes suspectes pour lever un drapeau d'alerte (`set $block 1`), et on applique un coupe-circuit pour le réseau local.

### 1. Le filtrage par mots-clés (`$request_uri`)

Cette première règle analyse l'URL demandée par le client. Si elle contient l'un des termes ciblés, la requête est immédiatement marquée pour blocage.

* **Les plateformes et réseaux sociaux (`linkedin\.com|t\.me`) :** Bloque les tentatives de partage automatique ou de scraping sauvage provenant de bots qui scannent les liens.
* **Le spam et la voyance russe (`избавиться|порчи|маету`) :** Nettoie les requêtes contenant des caractères cyrilliques typiques des spams russes (désenvoûtement, rituels, etc.).
* **Le scraping commercial et B2B (`wholesale|distributor|importer|private\+label|tedarik%C3%A7isi`) :** Cible les bots marchands qui siphonnent les données à des fins d'indexation commerciale ou de prospection sauvage.
* **Les "footprints" et spams de référencement (`2025.*Present|Roc\+Villa|mariage\+en\+Italie|solas`) :** Bloque des signatures de campagnes de spam très spécifiques qui cherchent à polluer les statistiques ou à exploiter des failles de formulaires.
* **Le blocage Unicode des requêtes chinoises (`%E[4-9]%[0-9A-F]{2}%[0-9A-F]{2}`) :** Une expression régulière chirurgicale qui intercepte les encodages de caractères chinois dans l'URL, souvent liés à des scans ou du spam automatisé.

### 2. Le filtrage par signature de code (`$http_user_agent`)

Ici, on cible l'identité de l'attaquant. Les scrapers utilisent majoritairement des scripts automatisés. On bloque donc les bibliothèques de programmation les plus courantes (`python`, `go-http-client`, `axios`, `curl`, `wget`, `scrapy`, `llamaindex`, `langchain`, etc.).

> 💡 **La subtilité de la règle :** L'utilisation d'une "assertion négative" `^(?!.*(searx|watchtower|shoutrrr))` au tout début permet de créer une **liste blanche**. Si le User-Agent contient l'un de nos outils légitimes (comme nos services de monitoring ou de notification), la règle l'ignore et ne le bloque pas, même s'il utilise `curl` ou `python` en arrière-plan.

### 3. Le coupe-circuit local (`$allowed_ip`)

C'est la règle de sécurité indispensable pour éviter de se bloquer soi-même (les faux positifs).
Si l'IP d'origine appartient au réseau local (LAN) ou aux sous-réseaux Docker (ce qui définit la variable `$allowed_ip` à `"yes"`), on force `$block` à `0`. L'accès nous est garanti, peu importe le contenu de la requête ou l'outil utilisé.

---

## Comment et où inclure ce fichier ?

Pour que Nginx applique ces restrictions, le fichier personnalisé doit être injecté au bon endroit dans l'architecture de Nginx Proxy Manager.

### Étape 1 : emplacement du fichier

Le script de filtrage doit être déposé sur le serveur dans le dossier des configurations personnalisées de NPM :
`/chemin_vers_votre_docker/npm/data/nginx/custom/searxng-custom-block.conf`

### Étape 2 : l'intégration dans le pipeline de sécurité

Pour que le filtrage soit automatique sur l'ensemble de vos hôtes de proxy, il faut l'appeler dans le fichier de configuration principal qui gère les proxys (généralement `server_proxy.conf` situé dans le même dossier).

Il suffit d'y ajouter cette ligne, idéalement au milieu de vos règles de vérification :

```nginx
# Inclusion des règles personnalisées anti-scraping
include /data/nginx/custom/searxng-custom-block.conf;

```

### Étape 3 : application de la sentence

Juste après l'inclusion de vos listes de blocage dans ce fichier principal, Nginx doit valider le verdict. Si la variable `$block` est passée à `1` suite à l'analyse, on coupe la connexion en renvoyant une erreur **HTTP 403 Forbidden** :

```nginx
# Application du verdict
if ($block) {
    return 403;
}

```

Une fois les fichiers enregistrés, un simple rechargement de la configuration de Nginx Proxy Manager active cette barrière invisible contre les scrapers.

---

> **Auteur** : ce guide est proposé par **Amaury aka BlablaLinux**. Retrouvez l'ensemble de mes services sur [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).

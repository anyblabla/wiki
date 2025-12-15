---
title: Bloquer les robots d’IA directement via NGINX Proxy Manager
description: Apprenez à centraliser le blocage des principaux crawlers d’IA dans NGINX Proxy Manager (NPM) en utilisant un fichier de configuration custom. Une alternative efficace aux modifications des fichiers robots.txt individuels.
published: true
date: 2025-12-15T23:32:58.582Z
tags: nginx, proxy, npm, blocage-crawlers, robots, ia, ai, crawlers
editor: markdown
dateCreated: 2025-12-15T23:32:58.582Z
---

## Introduction
Plutôt que de modifier les fichiers `robots.txt` de chaque site individuellement (ce qui demande de maintenir des configurations séparées) :

* Exemple Gitea : [https://wiki.blablalinux.be/fr/robots-txt-gitea-blocage-ia](https://wiki.blablalinux.be/fr/robots-txt-gitea-blocage-ia)
* Exemple WordPress : [https://wiki.blablalinux.be/fr/robots-txt-wordpress-anti-ia](https://wiki.blablalinux.be/fr/robots-txt-wordpress-anti-ia)

... nous allons agir de manière beaucoup plus **efficace et centralisée** !

Nous allons implémenter le blocage directement au niveau du **proxy inverse**, ici via **NGINX Proxy Manager (NPM)**. Cette méthode permet de rejeter la connexion pour les User-Agents ciblés *avant* qu'ils n'atteignent l'application cible, économisant ainsi des ressources et simplifiant l'administration.

## 1. Comprendre l’architecture des configurations personnalisées de NPM
NGINX Proxy Manager offre des **points d'insertion (`include**`) bien définis dans la configuration NGINX générée. Ces points vous permettent d'ajouter des directives personnalisées sans modifier les fichiers de configuration de base.

Voici la liste des points d'insertion disponibles dans le répertoire `/data/nginx/custom/` (selon la documentation NPM) :

| Fichier de Configuration Personnalisé | Emplacement d'Inclusion | Portée |
| --- | --- | --- |
| `/data/nginx/custom/root_top.conf` | En haut de `nginx.conf` | Globale |
| `/data/nginx/custom/http_top.conf` | En haut du bloc `http` principal | Globale (HTTP/HTTPS) |
| **`/data/nginx/custom/server_proxy.conf`** | **Fin de chaque bloc `server` de proxy** | **Par hôte (recommandé)** |
| `/data/nginx/custom/http.conf` | Fin du bloc `http` principal | Globale (HTTP/HTTPS) |
| `/data/nginx/custom/root.conf` | À la toute fin de `nginx.conf` | Globale |
| *Autres...* | *Voir la documentation pour `events`, `stream`, etc.* | *Spécifique* |

> **⚠️ Attention : création des fichiers**
> **Ces fichiers de configuration personnalisés n'existent pas par défaut !** Ils sont facultatifs. Vous devez impérativement créer le fichier souhaité (dans notre cas, `server_proxy.conf`) dans le répertoire `/data/nginx/custom/` avant d'y placer votre code.

Pour plus de détails sur les points d'insertion, consultez la documentation officielle :
➡️ **[NGINX Proxy Manager - Documentation Configuration Avancée](https://nginxproxymanager.com/advanced-config/)**

## 2. Implémentation du blocage anti-IA
Nous allons utiliser le fichier `/data/nginx/custom/server_proxy.conf` pour cibler uniquement les configurations de proxy (les plus courantes pour les services exposés).

### Étape A : créer et éditer le fichier
Via votre console Linux sur votre hôte (Proxmox/Docker) :

```bash
# Assurez-vous d'être dans le bon volume de données de NPM
# Exemple de création (si le volume est monté quelque part)
mkdir -p /chemin/vers/data/nginx/custom/
touch /chemin/vers/data/nginx/custom/server_proxy.conf
nano /chemin/vers/data/nginx/custom/server_proxy.conf

```

### Étape B : insérer le code de blocage
Ajoutez le bloc de code suivant à l'intérieur de `server_proxy.conf` :

```nginx
# --- BLOCAGE CENTRALISÉ DES ROBOTS D'IA ---

# 1. Initialisation de la variable de blocage
set $block 0;

# 2. Vérification des User-Agents des IA/Crawlers
# Cette expression régulière vérifie l'en-tête http_user_agent
if ($http_user_agent ~* "(AddSearchBot|AI2Bot|AI2Bot\-DeepResearchEval|Ai2Bot\-Dolma|aiHitBot|amazon\-kendra|Amazonbot|AmazonBuyForMe|Andibot|Anomura|anthropic\-ai|Applebot|Applebot\-Extended|atlassian\-bot|Awario|bedrockbot|bigsur\.ai|Bravebot|Brightbot\ 1\.0|BuddyBot|Bytespider|CCBot|Channel3Bot|ChatGLM\-Spider|ChatGPT\ Agent|ChatGPT\-User|Claude\-SearchBot|Claude\-User|Claude\-Web|ClaudeBot|Cloudflare\-AutoRAG|CloudVertexBot|cohere\-ai|cohere\-training\-data\-crawler|Cotoyogi|Crawl4AI|Crawlspace|Datenbank\ Crawler|DeepSeekBot|Devin|Diffbot|DuckAssistBot|Echobot\ Bot|EchoboxBot|FacebookBot|facebookexternalhit|Factset_spyderbot|FirecrawlAgent|FriendlyCrawler|Gemini\-Deep\-Research|Google\-CloudVertexBot|Google\-Extended|Google\-Firebase|Google\-NotebookLM|GoogleAgent\-Mariner|GoogleOther|GoogleOther\-Image|GoogleOther\-Video|GPTBot|iAskBot|iaskspider|iaskspider/2\.0|IbouBot|ICC\-Crawler|ImagesiftBot|imageSpider|img2dataset|ISSCyberRiskCrawler|Kangaroo\ Bot|KlaviyoAIBot|KunatoCrawler|laion\-huggingface\-processor|LAIONDownloader|LCC|LinerBot|Linguee\ Bot|LinkupBot|Manus\-User|meta\-externalagent|Meta\-ExternalAgent|meta\-externalfetcher|Meta\-ExternalFetcher|meta\-webindexer|MistralAI\-User|MistralAI\-User/1\.0|MyCentralAIScraperBot|netEstate\ Imprint\ Crawler|NotebookLM|NovaAct|OAI\-SearchBot|omgili|omgilibot|OpenAI|Operator|PanguBot|Panscient|panscient\.com|Perplexity\-User|PerplexityBot|PetalBot|PhindBot|Poggio\-Citations|Poseidon\ Research\ Crawler|QualifiedBot|QuillBot|quillbot\.com|SBIntuitionsBot|Scrapy|SemrushBot\-OCOB|SemrushBot\-SWA|ShapBot|Sidetrade\ indexer\ bot|Spider|TerraCotta|Thinkbot|TikTokSpider|Timpibot|TwinAgent|VelenPublicWebCrawler|WARDBot|Webzio\-Extended|webzio\-extended|wpbot|WRTNBot|YaK|YandexAdditional|YandexAdditionalBot|YouBot|ZanistaBot)") {
    set $block 1;
}

# 3. EXCEPTION : Autoriser l'accès à robots.txt
if ($request_uri = "/robots.txt") {
    set $block 0;
}

# 4. Exécution du blocage (retour 403 Forbidden)
if ($block) {
    return 403;
}
# --- FIN BLOCAGE ANTI-IA ---

```

> **📌 Note sur la maintenance**
> Les User-Agents des robots d'IA sont en constante évolution et de nouveaux crawlers apparaissent régulièrement. Il est crucial de maintenir la liste ci-dessus à jour. La liste complète des bots couramment bloqués est inspirée de configurations communautaires telles que :
> ➡️ **[AI Robots NGINX Block List sur GitHub](https://github.com/ai-robots-txt/ai.robots.txt/blob/main/nginx-block-ai-bots.conf)**

## 3. Application de la configuration
Après avoir enregistré votre fichier `server_proxy.conf`, vous devez **redémarrer le conteneur NGINX Proxy Manager** pour qu'il inclue ce nouveau fichier dans la configuration principale et recharge le service NGINX :

```bash
docker restart <nom_du_conteneur_npm>

```

*(Remplacez `<nom_du_conteneur_npm>` par le nom de votre conteneur NPM, par exemple `nginx-proxy-manager`).*

## 4. Test de vérification du blocage (depuis votre terminal Linux)
Utilisez l'outil **`curl`** avec l'option `-A` (pour spécifier le User-Agent) afin de simuler l'accès d'un robot bloqué, puis d'un utilisateur normal.

### A. Tester avec un robot d'IA (attendu : code 403)
Exécutez la commande suivante en remplaçant `votre-site.com` par l'une de vos URL gérées par NPM :

```bash
# Simuler GPTBot tentant d'accéder à votre page d'accueil
curl -A "GPTBot" -I https://votre-site.com/

```

**Résultat attendu :** Vous devriez voir un code de réponse HTTP **`403 Forbidden`**.

### B. Tester l'exclusion `robots.txt` (attendu : code 200/301/302)
Ciblez `/robots.txt` avec le même User-Agent bloqué :

```bash
# Simuler GPTBot tentant d'accéder à robots.txt
curl -A "GPTBot" -I https://votre-site.com/robots.txt

```

**Résultat attendu :** Vous devriez obtenir une réponse de succès (`200 OK`) ou une redirection, mais **pas de `403**`.

### C. Tester avec un utilisateur standard (attendu : code 200 ou 301/302)
Vérifiez l'accès normal sans spécifier de User-Agent bloqué :

```bash
# Simuler un navigateur standard
curl -I https://votre-site.com/

```

**Résultat attendu :** Vous devriez recevoir une réponse de succès ou la redirection normale de votre application.

Si ces trois tests renvoient les codes attendus, votre configuration est fonctionnelle !
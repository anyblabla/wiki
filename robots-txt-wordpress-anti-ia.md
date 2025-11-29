---
title: Configuration avancée du robots.txt pour WordPress (Anti-IA et SEO)
description: Ce guide explique comment optimiser votre fichier robots.txt WordPress pour bloquer les robots d'IA spécifiques (GPTBot, ClaudeBot, etc.) tout en assurant une parfaite indexation SEO par les moteurs de recherche standards.
published: true
date: 2025-11-29T14:11:55.484Z
tags: wordpress, seo, indexation, robots.txt, anti-ia, blocage-crawlers, optimisation-wordpress
editor: markdown
dateCreated: 2025-11-29T14:11:55.484Z
---

Ce guide détaille la création et l'optimisation d'un fichier `robots.txt` sur WordPress. L'objectif principal est de **bloquer les robots d'Intelligence Artificielle (IA)** tout en assurant une **indexation optimale par les moteurs de recherche standards** (Google, Bing, etc.) et en sécurisant les zones sensibles de WordPress.

## 1\. Structure et objectif du fichier

Le protocole `robots.txt` fonctionne par ordre de spécificité : les règles les plus spécifiques (ciblant un robot nommé) priment sur les règles génériques (`User-agent: *`).

Notre fichier est divisé en deux sections principales :

1.  **Blocage spécifique :** Liste des agents d'IA à bloquer (`Disallow: /`).
2.  **Règles générales :** Règles pour les robots standards (`User-agent: *`) qui ne sont pas bloqués par la première section.

## 2\. Contenu du `robots.txt`

Voici le code complet à utiliser. **Remplacez `votre-site.com` par votre propre nom de domaine.**

```text
# SITEMAPS
Sitemap: https://votre-site.com/sitemap.xml
Sitemap: https://votre-site.com/news-sitemap.xml

# 1. RULES FOR ARTIFICIAL INTELLIGENCE ROBOTS (BLOCKED)
# These specific agents are explicitly disallowed from crawling the entire site.
User-agent: AddSearchBot
Disallow: /
User-agent: AI2Bot
Disallow: /
User-agent: Ai2Bot-Dolma
Disallow: /
User-agent: aiHitBot
Disallow: /
User-agent: AmazonBuyForMe
Disallow: /
User-agent: atlassian-bot
Disallow: /
User-agent: amazon-kendra-
Disallow: /
User-agent: Amazonbot
Disallow: /
User-agent: Andibot
Disallow: /
User-agent: Anomura
Disallow: /
User-agent: anthropic-ai
Disallow: /
User-agent: Applebot
Disallow: /
User-agent: Applebot-Extended
Disallow: /
User-agent: Awario
Disallow: /
User-agent: bedrockbot
Disallow: /
User-agent: bigsur.ai
Disallow: /
User-agent: Bravebot
Disallow: /
User-agent: Brightbot 1.0
Disallow: /
User-agent: BuddyBot
Disallow: /
User-agent: Bytespider
Disallow: /
User-agent: CCBot
Disallow: /
User-agent: ChatGPT Agent
Disallow: /
User-agent: ChatGPT-User
Disallow: /
User-agent: Claude-SearchBot
Disallow: /
User-agent: Claude-User
Disallow: /
User-agent: Claude-Web
Disallow: /
User-agent: ClaudeBot
Disallow: /
User-agent: Cloudflare-AutoRAG
Disallow: /
User-agent: CloudVertexBot
Disallow: /
User-agent: cohere-ai
Disallow: /
User-agent: cohere-training-data-crawler
Disallow: /
User-agent: Cotoyogi
Disallow: /
User-agent: Crawlspace
Disallow: /
User-agent: Datenbank Crawler
Disallow: /
User-agent: DeepSeekBot
Disallow: /
User-agent: Devin
Disallow: /
User-agent: Diffbot
Disallow: /
User-agent: DuckAssistBot
Disallow: /
User-agent: Echobot Bot
Disallow: /
User-agent: EchoboxBot
Disallow: /
User-agent: FacebookBot
Disallow: /
User-agent: facebookexternalhit
Disallow: /
User-agent: Factset_spyderbot
Disallow: /
User-agent: FirecrawlAgent
Disallow: /
User-agent: FriendlyCrawler
Disallow: /
User-agent: Gemini-Deep-Research
Disallow: /
User-agent: Google-CloudVertexBot
Disallow: /
User-agent: Google-Extended
Disallow: /
User-agent: Google-Firebase
Disallow: /
User-agent: Google-NotebookLM
Disallow: /
User-agent: GoogleAgent-Mariner
Disallow: /
User-agent: GoogleOther
Disallow: /
User-agent: GoogleOther-Image
Disallow: /
User-agent: GoogleOther-Video
Disallow: /
User-agent: GPTBot
Disallow: /
User-agent: iaskspider/2.0
Disallow: /
User-agent: IbouBot
Disallow: /
User-agent: ICC-Crawler
Disallow: /
User-agent: ImagesiftBot
Disallow: /
User-agent: img2dataset
Disallow: /
User-agent: ISSCyberRiskCrawler
Disallow: /
User-agent: Kangaroo Bot
Disallow: /
User-agent: LinerBot
Disallow: /
User-agent: Linguee Bot
Disallow: /
User-agent: meta-externalagent
Disallow: /
User-agent: Meta-ExternalAgent
Disallow: /
User-agent: meta-externalfetcher
Disallow: /
User-agent: Meta-ExternalFetcher
Disallow: /
User-agent: meta-webindexer
Disallow: /
User-agent: MistralAI-User
Disallow: /
User-agent: MistralAI-User/1.0
Disallow: /
User-agent: MyCentralAIScraperBot
Disallow: /
User-agent: netEstate Imprint Crawler
Disallow: /
User-agent: NovaAct
Disallow: /
User-agent: OAI-SearchBot
Disallow: /
User-agent: omgili
Disallow: /
User-agent: omgilibot
Disallow: /
User-agent: OpenAI
Disallow: /
User-agent: Operator
Disallow: /
User-agent: PanguBot
Disallow: /
User-agent: Panscient
Disallow: /
User-agent: panscient.com
Disallow: /
User-agent: Perplexity-User
Disallow: /
User-agent: PerplexityBot
Disallow: /
User-agent: PetalBot
Disallow: /
User-agent: PhindBot
Disallow: /
User-agent: Poseidon Research Crawler
Disallow: /
User-agent: QualifiedBot
Disallow: /
User-agent: QuillBot
Disallow: /
User-agent: quillbot.com
Disallow: /
User-agent: SBIntuitionsBot
Disallow: /
User-agent: Scrapy
Disallow: /
User-agent: SemrushBot-OCOB
Disallow: /
User-agent: SemrushBot-SWA
Disallow: /
User-agent: ShapBot
Disallow: /
User-agent: Sidetrade indexer bot
Disallow: /
User-agent: TerraCotta
Disallow: /
User-agent: Thinkbot
Disallow: /
User-agent: TikTokSpider
Disallow: /
User-agent: Timpibot
Disallow: /
User-agent: VelenPublicWebCrawler
Disallow: /
User-agent: WARDBot
Disallow: /
User-agent: Webzio-Extended
Disallow: /
User-agent: wpbot
Disallow: /
User-agent: YaK
Disallow: /
User-agent: YandexAdditional
Disallow: /
User-agent: YandexAdditionalBot
Disallow: /
User-agent: YouBot
Disallow: /

# 2. RULES FOR STANDARD SEARCH ROBOTS (ALLOWED)
# These rules apply to ALL other robots not specifically listed above (e.g., Googlebot, Bingbot).
User-agent: *
Disallow: /wp-admin/
Allow: /wp-admin/admin-ajax.php
Disallow: /wp-content/plugins/
Disallow: /wp-content/cache/
Disallow: /wp-content/themes/*.zip
Allow: /wp-content/uploads/
```

## 3\. Détail des règles

### A. Blocage des robots IA (secteur 1)

  * **Syntaxe :** `User-agent: [Nom du bot]` suivi de `Disallow: /`.
  * **Fonctionnement :** Chaque robot d'IA spécifiquement nommé (comme `GPTBot`, `ClaudeBot`, etc.) trouve sa règle d'exclusion personnelle (`Disallow: /`) et s'abstient de crawler l'intégralité du site.

### B. Moteurs de recherche standards (secteur 2)

  * **Syntaxe :** `User-agent: *` (s'applique à tous les robots non mentionnés précédemment, comme Googlebot ou Bingbot).
  * **Fonctionnement :** Ces robots, ne correspondant pas aux noms bloqués ci-dessus, suivent la règle fourre-tout `User-agent: *`.

#### Règles WordPress de sécurité et d'optimisation :

| Directive | Description |
| :--- | :--- |
| `Disallow: /wp-admin/` | **Bloque** l'accès au tableau de bord d'administration (sauf le fichier AJAX essentiel). |
| `Allow: /wp-admin/admin-ajax.php` | **Autorise** l'accès au fichier AJAX, souvent nécessaire pour le bon fonctionnement de nombreux plugins et du thème. |
| `Disallow: /wp-content/plugins/` | **Bloque** le crawling des dossiers de plugins. |
| `Disallow: /wp-content/cache/` | **Bloque** les fichiers temporaires de cache (inutiles pour l'indexation). |
| `Disallow: /wp-content/themes/*.zip` | **Bloque** les fichiers .zip des thèmes. |
| `Allow: /wp-content/uploads/` | **Autorise** l'indexation de tous vos médias (images, documents) qui sont cruciaux pour le SEO. |

## 4\. Mise en place sur WordPress

Pour mettre en place ce fichier, utilisez l'une des méthodes suivantes :

1.  **Via un plugin SEO (recommandé) :** La plupart des plugins SEO (Yoast, Rank Math, SEOPress) offrent une interface pour modifier directement le contenu du `robots.txt` virtuel de WordPress.
2.  **Manuellement (si possible) :** Téléchargez le fichier, collez le contenu ci-dessus, et téléversez-le à la racine de votre installation WordPress (dossier principal).

> **💡 Exemple concret :** Pour voir à quoi ressemble ce fichier en production, vous pouvez consulter : **[https://blablalinux.be/robots.txt](https://blablalinux.be/robots.txt)**
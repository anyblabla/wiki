---
title: Analyse du fichier robots.txt de Gitea (blablalinux.be)
description: Analyse du fichier robots.txt de l'instance Gitea blablalinux.be, qui applique une stratégie stricte de blocage des robots d’IA, tout en autorisant l'indexation sélective par les moteurs de recherche traditionnels.
published: true
date: 2025-11-30T17:34:55.173Z
tags: seo, indexation, robots.txt, anti-ia, gitea
editor: markdown
dateCreated: 2025-11-30T17:02:16.215Z
---

Le fichier **`robots.txt`** de cette instance Gitea est un excellent exemple d'une **stratégie à double niveau** : il assure la **protection des données** face à la collecte massive des agents d'intelligence artificielle (IA), tout en maintenant une **accessibilité contrôlée** pour le référencement traditionnel.

**Note de réutilisation :** Ce fichier `robots.txt` peut être **utilisé tel quel** pour n'importe quelle autre instance Gitea. Les chemins spécifiques bloqués sont des URLs standard de Gitea, et la liste des agents d'IA bloqués est universelle.

-----

## I. Décortication du fichier `robots.txt`

Le fichier est structuré en deux grandes sections distinctes.

### 1\. Règles pour les robots d'intelligence artificielle (bloqués)

Cette section est la plus volumineuse et la plus spécifique.

| Directive | Contenu | Explication |
| :--- | :--- | :--- |
| **`User-agent: [Nom du Bot]`** | Liste exhaustive d'agents utilisateurs liés à l'IA, à l'analyse de données (*scraping*), et à l'exploration étendue (ex. : ChatGPT, Claude, Gemini, GPTBot, Scrapy). | Le propriétaire du site cible explicitement ces **collecteurs de données** pour empêcher l'utilisation du code source ou du contenu hébergé pour l'entraînement de modèles d'IA (LLMs). |
| **`Disallow: /`** | Associée à chaque agent utilisateur de la liste. | Cette directive **bloque totalement** l'accès au site pour l'agent ciblé. **Aucune page** n'est autorisée à être explorée par ces bots. |

### 2\. Règles pour les robots standards (autorisés et sécurisés)

Cette section concerne les agents utilisateurs généralistes (moteurs de recherche traditionnels comme Googlebot, Bingbot, etc.).

| Directive | Contenu | Explication |
| :--- | :--- | :--- |
| **`User-agent: *`** | Le caractère générique `*` désigne **tous les autres robots** qui ne sont pas spécifiquement bloqués à la section 1. | Ceci permet de définir des règles générales pour l'indexation traditionnelle. |
| **`Disallow: /user/settings`**, **`/notifications`**, **`/login`**, **`/install`**, etc. | Liste ciblée de chemins bloqués. | Ces chemins correspondent à des **pages privées, sensibles ou utilitaires** de Gitea. Les bloquer garantit qu'elles n'apparaissent pas dans les résultats de recherche. |
| **`Allow: /`** | Placé après les directives `Disallow` spécifiques. | Ceci autorise l'indexation des **dépôts publics**, des pages d'exploration de code et des *issues* pour les moteurs de recherche. Les règles `Disallow` priment toujours sur le `Allow` général. |

-----

## II. Implications en référencement (SEO) et stratégie de visibilité

Cette configuration privilégie la **protection des données** tout en maintenant un **SEO ciblé** sur le contenu public du code.

### 1\. Avantages pour le SEO traditionnel

  * **Indexation du contenu clé :** L'autorisation d'accès via `Allow: /` permet aux moteurs de recherche d'indexer les dépôts et les projets publics, assurant que le code source soit découvrable.
  * **Optimisation du budget de crawl :** Le blocage des chemins sensibles (`/login`, `/settings`) concentre l'effort d'exploration des robots sur le contenu à haute valeur ajoutée, améliorant l'efficacité du référencement.

### 2\. Mesure défensive anti-IA

L'exclusion massive (Section 1) est une mesure défensive pour :

  * **Protéger la propriété intellectuelle (PI) :** Empêcher les entreprises d'IA d'utiliser le code source et le contenu du site pour l'entraînement de leurs modèles d'apprentissage.
  
![web-scraping.jpeg](/robots-txt-gitea-blocage-ia/web-scraping.jpeg)
  
  * **Réduire la charge serveur :** Diminuer le trafic généré par des robots souvent très gourmands en ressources, assurant une meilleure performance pour les utilisateurs humains.

-----

## III. Fichier `robots.txt` complet

Voici le contenu intégral du fichier, prêt à être déployé sur toute instance Gitea :

```robots.txt
# 1. RÈGLES POUR LES ROBOTS D'INTELLIGENCE ARTIFICIELLE (BLOQUÉS)
# Ces agents spécifiques sont explicitement non autorisés à crawler l'intégralité du site.
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

# 2. RÈGLES POUR LES ROBOTS STANDARDS (AUTORISÉS & SÉCURISÉS)
User-agent: *
# Bloque les chemins sensibles ou privés de Gitea
Disallow: /user/settings
Disallow: /notifications
Disallow: /explore/users
Disallow: /repo/migrate
Disallow: /repo/create
Disallow: /login
Disallow: /install

# Autorise l'indexation de tous les référentiels publics, issues, etc.
Allow: /
```

-----

## IV. Suivi et mises à jour du fichier `robots.txt`

Le fichier `robots.txt` étant versionné sur Gitea, il est possible de s'abonner aux changements. Cela permet aux administrateurs ou aux utilisateurs intéressés d'être tenus informés de toute modification ou ajout (notamment pour bloquer de nouveaux robots d'IA).

Vous pouvez vous **abonner au flux RSS** des mises à jour du fichier `robots.txt` sur Gitea Blablalinux en utilisant le lien suivant dans votre lecteur RSS favori :

➡️ **Lien du flux RSS :** **[https://gitea.blablalinux.be/Stars/ai.robots.txt/rss/branch/main/robots.txt](https://gitea.blablalinux.be/Stars/ai.robots.txt/rss/branch/main/robots.txt)**

-----

## V. Instructions d'installation : où placer le fichier `robots.txt` dans Gitea

Le fichier doit être placé dans le sous-répertoire `public` au sein du répertoire de personnalisation de Gitea (`custom`).

> **Chemin d'accès :**
> `<CHEMIN_INSTALLATION_GITEA>/custom/public/robots.txt`

### 1\. Procédure d'installation par ligne de commande (Bash)

**a. Préparation (Définir le chemin et créer le répertoire)**

```bash
GITEA_BASE_DIR="/var/lib/gitea"  # À adapter au chemin réel
mkdir -p "$GITEA_BASE_DIR/custom/public"
```

**b. Création du fichier `robots.txt`**

```bash
cat << 'EOF' > "$GITEA_BASE_DIR/custom/public/robots.txt"
# [Le contenu complet du robots.txt listé ci-dessus est inséré ici]
User-agent: AddSearchBot
Disallow: /
# ... (le reste de la liste longue)
User-agent: YouBot
Disallow: /

# 2. RÈGLES POUR LES ROBOTS STANDARDS (AUTORISÉS & SÉCURISÉS)
User-agent: *
# Bloque les chemins sensibles ou privés de Gitea
Disallow: /user/settings
Disallow: /notifications
Disallow: /explore/users
Disallow: /repo/migrate
Disallow: /repo/create
Disallow: /login
Disallow: /install

# Autorise l'indexation de tous les référentiels publics, issues, etc.
Allow: /
EOF
```

**c. Ajuster les permissions (Crucial)**

```bash
chown -R git:git "$GITEA_BASE_DIR/custom" # Ajustez l'utilisateur/groupe si nécessaire
chmod 644 "$GITEA_BASE_DIR/custom/public/robots.txt"
```
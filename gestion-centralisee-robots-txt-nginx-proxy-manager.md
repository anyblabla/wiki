---
title: Gestion centralisée des fichiers robots.txt avec Nginx Proxy Manager
description: Découvrez comment centraliser et configurer les fichiers robots.txt pour 15 services auto-hébergés via NPM afin de sécuriser vos accès et uniformiser le blocage des robots d'IA.
published: true
date: 2026-01-04T12:02:59.461Z
tags: nginx, npm, sécurité, sysadmin, seo, robots.txt, auto-hébergement, ia
editor: markdown
dateCreated: 2025-12-23T22:33:31.704Z
---

## Introduction

Dans un écosystème auto-hébergé composé de nombreux services (Docker, instances Web, outils tiers), la gestion du fichier `robots.txt` est souvent un casse-tête. Par défaut, soit le service ne propose rien (erreur 404), soit il propose un fichier générique qui ne protège pas les zones sensibles (comme l'administration ou les API), soit il est écrasé à chaque mise à jour du conteneur.

L'objectif de cette documentation est de montrer comment **déléguer la gestion du robots.txt à Nginx Proxy Manager (NPM)**. Au lieu de laisser l'application gérer ce fichier, c'est le reverse proxy qui intercepte la requête et renvoie une réponse personnalisée. Cela permet d'harmoniser ses règles, de sécuriser ses services et de bloquer les robots d'intelligence artificielle de manière uniforme sur toute son infrastructure.

### Les avantages de cette méthode

* **Persistance totale :** Vos règles ne dépendent plus des fichiers internes de l'application.
* **Centralisation :** Tout est géré dans l'interface de NPM (onglet Advanced).
* **Sécurité :** On bloque l'accès aux dossiers sensibles (`/admin`, `/settings`, `/api`) avant même que l'application ne soit sollicitée.
* **Uniformité :** On peut insérer une signature ou un lien vers sa propre politique de confidentialité sur tous ses services.

---

## Mise en œuvre technique

Pour chaque service exposé via NPM, la procédure est identique :

1. Éditez votre **Proxy Host** dans NPM.
2. Allez dans l'onglet **Advanced**.
3. Collez le bloc correspondant au service dans le champ **Custom Nginx Configuration**.
4. Le serveur Nginx interceptera alors l'URL `/robots.txt` et servira le contenu défini.

> ⚠️ Certaines configurations contiennent un lien vers le Sitemap Blabla Linux ! Remplacer ce lien par le vôtre, ou supprimer le ✔️

---

## Bibliothèque de configurations par service

### Web et CMS

#### 1. WordPress

```nginx
location = /robots.txt {
    add_header Content-Type text/plain;
    return 200 "Sitemap: https://blablalinux.be/sitemap.xml\nSitemap: https://blablalinux.be/news-sitemap.xml\n\nUser-agent: *\nDisallow: /wp-admin/\nAllow: /wp-admin/admin-ajax.php\nDisallow: /wp-content/plugins/\nDisallow: /wp-content/cache/\nDisallow: /wp-content/themes/*.zip\nAllow: /wp-content/uploads/\nAllow: /\n";
}

```

#### 2. Wiki.js

```nginx
location = /robots.txt {
    add_header Content-Type text/plain;
    return 200 "Sitemap: https://wiki.blablalinux.be/sitemap.xml\n\nUser-agent: *\nDisallow: /a/dashboard\nDisallow: /a/settings\nAllow: /\n";
}

```

### Médias et social

#### 3. PeerTube

```nginx
location = /robots.txt {
    add_header Content-Type text/plain;
    return 200 "User-agent: *\nDisallow: /admin\nDisallow: /api/\nDisallow: /search\nAllow: /w/\nAllow: /c/\nAllow: /videos\nAllow: /\n";
}

```

#### 4. Mastodon

```nginx
location = /robots.txt {
    add_header Content-Type text/plain;
    return 200 "User-agent: *\nDisallow: /media_proxy/\nDisallow: /interact/\nDisallow: /api/v1/instance/domain_blocks\nDisallow: /admin\nAllow: /\n";
}

```

### Outils et utilitaires

#### 5. YOURLS

```nginx
location = /robots.txt {
    add_header Content-Type text/plain;
    return 200 "User-agent: *\nDisallow: /admin/\nDisallow: /includes/\nDisallow: /\n";
}

```

#### 6. Stirling PDF

```nginx
location = /robots.txt {
    add_header Content-Type text/plain;
    return 200 "User-agent: *\nAllow: /\n";
}

```

#### 7. BentoPDF

```nginx
location = /robots.txt {
    add_header Content-Type text/plain;
    return 200 "User-agent: *\nAllow: /\n";
}

```

#### 8. IT-Tools

```nginx
location = /robots.txt {
    add_header Content-Type text/plain;
    return 200 "User-agent: *\nAllow: /\n";
}

```

#### 9. OmniTools

```nginx
location = /robots.txt {
    add_header Content-Type text/plain;
    return 200 "User-agent: *\nAllow: /\n";
}

```

#### 10. PairDrop

```nginx
location = /robots.txt {
    add_header Content-Type text/plain;
    return 200 "User-agent: *\nDisallow: /\n";
}

```

#### 11. Vert SH

```nginx
location = /robots.txt {
    add_header Content-Type text/plain;
    return 200 "User-agent: *\nAllow: /\n";
}

```

### Développement et administration

#### 12. Gitea

```nginx
location = /robots.txt {
    add_header Content-Type text/plain;
    return 200 "User-agent: *\nDisallow: /user/login\nDisallow: /user/sign_up\nDisallow: /repo/migrate\nAllow: /\n";
}

```

#### 13. ByteStash

```nginx
location = /robots.txt {
    add_header Content-Type text/plain;
    return 200 "User-agent: *\nDisallow: /settings\nDisallow: /admin\nDisallow: /login\nAllow: /\n";
}

```

#### 14. Listmonk

```nginx
location = /robots.txt {
    add_header Content-Type text/plain;
    return 200 "User-agent: *\nDisallow: /admin\nDisallow: /api/\nAllow: /subscription/form\nAllow: /archive\nAllow: /\n";
}

```

#### 15. LittleLink

```nginx
location = /robots.txt {
    add_header Content-Type text/plain;
    return 200 "User-agent: *\nDisallow: /admin\nAllow: /\n";
}

```

#### 16. Apache

```nginx
location = /robots.txt {
    add_header Content-Type text/plain;
    return 200 "Sitemap: https://fichiers.blablalinux.be/sitemap.xml\nUser-agent: *\nDisallow: /files/\nDisallow: /*.png\nDisallow: /*.ico\nDisallow: /*.old\nAllow: /\n";
}

```

---

## Signature et blocage des IA (Centralisé)

Pour parfaire cette installation, j'intègre au début de chaque fichier `robots.txt` une signature qui renvoie vers une page dédiée de mon Wiki.

### Pourquoi cette page ?

Elle sert de référence publique pour expliquer aux administrateurs de robots (et notamment ceux des intelligences artificielles comme OpenAI, Claude ou Perplexity) que le blocage est géré de manière proactive directement au niveau du serveur. Cela évite de répéter de longues listes de blocage dans chaque fichier et centralise l'explication de ma démarche éthique et technique.

**Lien de documentation utilisé :** [https://wiki.blablalinux.be/fr/blocage-robots-ia-nginx-proxy-manager](https://wiki.blablalinux.be/fr/blocage-robots-ia-nginx-proxy-manager)
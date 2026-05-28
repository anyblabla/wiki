---
title: Anatomie d'une recherche - Humain vs Robot
description: Découvrez comment différencier une recherche humaine d'une attaque de robot dans vos logs SearXNG. Une analyse concrète étape par étape pour sécuriser et comprendre le trafic de votre serveur.
published: false
date: 2026-05-28T16:24:59.916Z
tags: nginx, fail2ban, sécurité, searxng, autohébergement, logs
editor: markdown
dateCreated: 2026-05-28T16:24:59.916Z
---

Sur un moteur de recherche comme SearXNG, l'analyse des journaux d'accès (logs) permet de distinguer très facilement le comportement d'un véritable utilisateur humain de celui d'un script automatisé (bot).

Voici le décryptage de deux situations réelles capturées en direct sur les logs de mon serveur.

---

## 1. La recherche humaine : une trace chronologique et interactive

Un être humain interagit en temps réel avec l'interface. Sa navigation génère une suite logique de requêtes qui racontent une histoire. Voici la trace complète d'un utilisateur (dont l'IP a été anonymisée) sur une minute environ.

### Les logs bruts complets

```text
[28/May/2026:18:14:34 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "POST /autocompleter?q=app HTTP/1.1" 499
[28/May/2026:18:14:34 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "POST /autocompleter?q=appl HTTP/1.1" 200
[28/May/2026:18:14:35 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "POST /autocompleter?q=appli HTTP/1.1" 200
[28/May/2026:18:14:35 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "POST /autocompleter?q=applic HTTP/1.1" 499
[28/May/2026:18:14:35 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "POST /autocompleter?q=applica HTTP/1.1" 499
[28/May/2026:18:14:36 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "POST /autocompleter?q=application HTTP/1.1" 499
[28/May/2026:18:14:36 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "POST /autocompleter?q=application HTTP/1.1" 200
[28/May/2026:18:14:36 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "POST /autocompleter?q=application+g HTTP/1.1" 499
[28/May/2026:18:14:37 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "POST /autocompleter?q=application+ges HTTP/1.1" 499
[28/May/2026:18:14:37 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "POST /autocompleter?q=application+gestion HTTP/1.1" 499
[28/May/2026:18:14:38 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "POST /autocompleter?q=application+gestion+s HTTP/1.1" 499
[28/May/2026:18:14:38 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "POST /autocompleter?q=application+gestion+st HTTP/1.1" 499
[28/May/2026:18:14:38 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "POST /autocompleter?q=application+gestion+sto HTTP/1.1" 200
[28/May/2026:18:14:38 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "POST /autocompleter?q=application+gestion+stoc HTTP/1.1" 200
[28/May/2026:18:14:40 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "POST /search HTTP/1.1" 200
[28/May/2026:18:14:40 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "GET /clienta3jiao5xm0fmh64y.css HTTP/1.1" 200
[28/May/2026:18:14:40 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "GET /static/themes/simple/sxng-ltr.min.css HTTP/1.1" 304
[28/May/2026:18:14:40 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "GET /static/themes/simple/sxng-core.min.js HTTP/1.1" 304
[28/May/2026:18:14:40 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "GET /static/themes/simple/chunk/chlzpS6K.min.js HTTP/1.1" 304
[28/May/2026:18:14:40 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "GET /static/themes/simple/chunk/DyePpW7L.min.js HTTP/1.1" 304
[28/May/2026:18:14:40 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "GET /static/themes/simple/chunk/aUw47Wy0.min.js HTTP/1.1" 304
[28/May/2026:18:14:40 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "GET /static/themes/simple/chunk/CQ8vfMdp.min.js HTTP/1.1" 304
[28/May/2026:18:14:40 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "GET /static/themes/simple/chunk/DH1EQbEY.min.js HTTP/1.1" 304
[28/May/2026:18:14:41 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "GET /static/themes/simple/chunk/DGJ63wI6.min.js HTTP/1.1" 304
[28/May/2026:18:14:41 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "GET /static/themes/simple/chunk/BnP4vIuG.min.js HTTP/1.1" 304
[28/May/2026:18:14:41 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "GET /favicon_proxy?authority=www.odoo.com&h=10b48... HTTP/1.1" 200
[28/May/2026:18:14:41 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "GET /favicon_proxy?authority=www.cartelis.com&h=cec3cd... HTTP/1.1" 200
[28/May/2026:18:14:41 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "GET /favicon_proxy?authority=odysseedigitale.fr&h=120c2b... HTTP/1.1" 200
[28/May/2026:18:14:41 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "GET /favicon_proxy?authority=www.gestisoft.com&h=6ab16b... HTTP/1.1" 200
[28/May/2026:18:14:41 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "GET /favicon_proxy?authority=www.capterra.fr&h=893e17... HTTP/1.1" 200
[28/May/2026:18:14:41 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "GET /favicon_proxy?authority=www.appvizer.fr&h=0b8878... HTTP/1.1" 200
[28/May/2026:18:14:41 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "GET /favicon_proxy?authority=www.stockpit.app&h=68cb0a... HTTP/1.1" 200
[28/May/2026:18:14:53 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "POST /autocompleter HTTP/1.1" 200
[28/May/2026:18:14:53 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "POST /search HTTP/1.1" 200
[28/May/2026:18:14:54 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "GET /clienta3jiao5xm0fmh64y.css HTTP/1.1" 200
[28/May/2026:18:14:54 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "GET /favicon_proxy?authority=www.getapp.fr&h=2d3424... HTTP/1.1" 200
[28/May/2026:18:14:54 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "GET /favicon_proxy?authority=pdf.wondershare.fr&h=c27584... HTTP/1.1" 200
[28/May/2026:18:14:54 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "GET /favicon_proxy?authority=www.apogea.fr&h=67952f... HTTP/1.1" 200
[28/May/2026:18:14:54 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "GET /favicon_proxy?authority=www.rollsrapides.com&h=1b42cc... HTTP/1.1" 200
[28/May/2026:18:14:55 +0200] [Client 176.182.xxx.xxx] [yes FR] [searxng.blablalinux.be] "GET /favicon_proxy?authority=www.wiilog.fr&h=749c8f... HTTP/1.1" 200

```

### L'analyse du comportement

Quand on regarde ces logs, on y lit l'histoire de ce que l'utilisateur a fait, étape par étape :

**Étape 1 : la première recherche (18:14:34 à 18:14:41)**

* **La frappe au clavier :** il commence à taper dans la barre de recherche. L'auto-complétion s'emballe au rythme de ses doigts : `app`, `appl`, `appli`... puis il ajoute un espace et continue jusqu'à taper `application gestion stoc`.
* **Les annulations (code 499) :** l'utilisateur tape souvent plus vite que le temps de réponse du serveur. Le navigateur annule donc la requête précédente (d'où le code HTTP 499) pour envoyer la nouvelle lettre.
* **La validation (18:14:40) :** il s'arrête sur une suggestion et valide. C'est la ligne `"POST /search HTTP/1.1" 200`. La page s'affiche.
* **Le rendu de la page :** son navigateur télécharge les fichiers nécessaires à l'affichage de la page web (`/static/themes/simple...`). Les codes `304` indiquent que les fichiers étaient déjà dans le cache de l'utilisateur, ce qui est normal.
* **Le chargement des résultats :** mon instance récupère et affiche les icônes des sites trouvés (`/favicon_proxy`), comme Odoo, Capterra ou Stockpit. Il cherchait clairement un logiciel de gestion.

**Étape 2 : la deuxième recherche (18:14:53 à 18:14:55)**

* **Le changement de mot-clé (18:14:53) :** 12 secondes plus tard, il décide d'affiner ou de modifier sa recherche. Il clique à nouveau dans la barre et valide.
* **L'affichage des nouveaux résultats :** la page s'actualise avec un code HTTP 200 et affiche de toutes nouvelles icônes de sites (GetApp, Wondershare, Apogea...).

---

## 2. La recherche par un robot : une exécution brute et ciblée

Un robot (bot de scraping, crawler) n'utilise pas d'interface graphique. Il envoie des requêtes directes, massives et standardisées. Il masque souvent son origine géographique réelle derrière un proxy ou un VPN.

### Les logs bruts

```text
[28/May/2026:18:14:26 +0200] [Client 104.22.20.71] [yes US] [searxng.blablalinux.be] "GET /search?q=%E5%85%83%E7%A5%9E6.7%E7%89%88%E6%9C%AC%E4%BB%80%E4%B9%88%E6%97%B6%E5%80%99%E4%B8%8A%E7%BA%BF&engines=bing,duckduckgo HTTP/1.1" 403

```

### L'analyse du comportement

* **Pas d'interaction préalable :** le robot interroge directement l'URL de recherche sans jamais solliciter l'auto-complétion au clavier.
* **Requête encodée :** la chaîne de caractères correspond à du texte chinois encodé en URL (ici, une recherche automatisée).
* **Ciblage de moteurs spécifiques :** l'URL force l'utilisation exclusive de certains moteurs (`&engines=bing,duckduckgo`), ce qui est une signature typique des scripts de scraping.
* **Contournement GeoIP :** même si la requête cible du contenu asiatique, l'adresse IP vue par le serveur est américaine (`[yes US]`). Le robot utilise un VPN ou un proxy pour cacher sa vraie source.
* **Le blocage applicatif (code 403) :** grâce à une règle personnalisée sur mon proxy Nginx, le serveur détecte les caractères chinois interdits dans la requête et la bloque immédiatement (code HTTP 403). Le robot ne peut pas accéder aux ressources.

---

> **Bilan :** alors qu'un humain consomme des ressources de manière fluide et logique pour afficher une page web, le robot cherche uniquement à extraire des données brutes en un temps record. Une bonne analyse des logs permet de filtrer ces requêtes inutiles et de protéger nos services auto-hébergés.

> **Auteur** : ce guide est proposé par **Amaury aka BlablaLinux**. Retrouvez l'ensemble de mes services sur [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).

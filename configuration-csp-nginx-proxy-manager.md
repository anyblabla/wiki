---
title: Guide de configuration CSP
description: Apprenez à configurer le Content Security Policy (CSP) dans Nginx Proxy Manager pour sécuriser vos services comme Immich, Wiki.js ou Vert.sh sans bloquer les fonctionnalités vitales de vos outils.
published: true
date: 2026-02-10T23:50:34.619Z
tags: nginx, npm, sécurité, csp, auto-hébergement
editor: markdown
dateCreated: 2026-02-10T23:48:25.977Z
---

Le **Content Security Policy (CSP)** est une couche de sécurité supplémentaire qui permet de détecter et d'atténuer certains types d'attaques, notamment les injections de données (XSS) et les détournements de clics.

Dans **Nginx Proxy Manager**, un CSP mal configuré peut bloquer l'affichage d'images, de vidéos ou même casser des fonctionnalités vitales comme les WebSockets ou les conversions vidéo.

---

## Structure de base pour la majorité des services

Voici le bloc de base recommandé. Il autorise le chargement de ressources depuis votre propre domaine, ainsi que les images en `data:` et `blob:`, indispensables pour les miniatures dans la plupart des applications modernes.

```nginx
# CSP standard : sécurisé mais flexible
add_header Content-Security-Policy "default-src 'self' 'unsafe-inline' 'unsafe-eval'; img-src 'self' https: data: blob:; style-src 'self' 'unsafe-inline';";

```

---

## Cas particuliers par service

Certains services exigent des directives spécifiques pour fonctionner correctement. Voici les réglages validés par **Blabla Linux**.

### Vert.sh et Picsur (Gestion des médias)

Ces outils utilisent des **WebWorkers** (FFmpeg en navigateur) et des **blobs** pour les aperçus.

* **Problème :** Les vidéos ne se convertissent pas ou les miniatures restent blanches.
* **Solution :** Ajouter `worker-src` et `connect-src *`.

```nginx
add_header Content-Security-Policy "default-src 'self' 'unsafe-inline' 'unsafe-eval'; img-src 'self' https: data: blob:; worker-src 'self' blob:; connect-src *;";

```

### Immich et Memos (Temps réel)

Ces applications utilisent des **WebSockets** pour la synchronisation en direct.

* **Problème :** Message "Serveur hors ligne" ou interface figée.
* **Solution :** Autoriser les protocoles `wss:` et `https:` dans `connect-src`.

```nginx
add_header Content-Security-Policy "default-src 'self' 'unsafe-inline' 'unsafe-eval'; connect-src 'self' wss: https:; img-src * data: blob:;";

```

### Wiki.js (Intégration PeerTube et GitHub)

* **Problème :** Impossible d'intégrer des vidéos PeerTube ou de synchroniser avec GitHub.
* **Solution :** Ajouter `frame-src` pour les iframes et `connect-src` pour l'API GitHub.

```nginx
add_header Content-Security-Policy "default-src 'self' 'unsafe-inline' 'unsafe-eval'; frame-src 'self' https://peertube.votredomaine.be; connect-src 'self' https://github.com https://api.github.com;";

```

---

## Le piège de l'isolation et les logos personnalisés

Si vous injectez un logo personnalisé sur vos sous-domaines (ex: sur `picsur.blablalinux.be`), vous devez être vigilant avec les headers d'isolation mémoire (COOP/COEP).

**Le problème :**
Lorsque vous activez des politiques de sécurité strictes pour FFmpeg (WASM), le navigateur refuse de charger toute image provenant d'un domaine différent de celui sur lequel vous vous trouvez. Si votre logo est hébergé sur `blablalinux.be` mais affiché sur un sous-domaine, il sera bloqué.

**Les deux solutions Blabla Linux :**

1. **La méthode permissive (recommandée) :**
Retirer les headers `Cross-Origin-Embedder-Policy` et `Cross-Origin-Opener-Policy` dans NPM pour laisser le logo s'afficher normalement.
2. **La méthode stricte (si l'isolation est requise) :**
Ajouter l'attribut `crossorigin="anonymous"` dans votre balise d'image injectée via `sub_filter` :
```html
<img src="https://blablalinux.be/logo.png" crossorigin="anonymous">

```



---

## Comment déboguer un CSP

Si un élément ne s'affiche pas :

1. Ouvrez la console de votre navigateur (**F12**).
2. Cherchez les messages en rouge commençant par `Refused to load...`.
3. La console vous indiquera la directive manquante (ex: `worker-src` ou `frame-src`).

> **Note d'Amaury aka BlablaLinux :** Toujours tester vos modifications en navigation privée pour forcer le navigateur à lire la nouvelle politique de sécurité.
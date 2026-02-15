---
title: Page d'erreur 404 centralisée avec redirection SSL (méthode statique)
description: Apprenez à créer une page d'erreur 404 sécurisée par SSL (certificat Wildcard) sur Nginx Proxy Manager, utilisant un fichier HTML statique pour un rendu professionnel, dynamique et responsive.
published: true
date: 2026-02-15T20:55:11.459Z
tags: npm, 404, auto-hébergement, nginx proxy manager, matrix, ssl
editor: markdown
dateCreated: 2026-02-15T20:48:51.990Z
---

Ce guide détaille comment transformer une erreur de domaine inconnu sur **Nginx Proxy Manager (NPM)** en une page personnalisée, sécurisée par SSL, et hébergée directement via un fichier statique sur votre serveur.

## Pourquoi cette nouvelle méthode ?

Si vous avez utilisé ma précédente documentation sur l'injection HTML directe, vous avez remarqué deux limites : l'absence de certificat SSL valide sur les domaines inconnus et la difficulté de maintenir un code complexe dans une petite zone de texte de l'interface.

> ![erreur-404-no-ssl.png](/page-erreur-404-statique-ssl-npm/erreur-404-no-ssl.png)
> *Légende : L'ancienne méthode avec le code HTML injecté directement dans les paramètres globaux.*

### Ce qui change aujourd'hui

| Caractéristique | Ancienne méthode | Nouvelle méthode (celle-ci) |
| --- | --- | --- |
| **Sécurité SSL** | Souvent absente (alerte navigateur) | **SSL Wildcard actif** (Cadenas vert) |
| **Hébergement** | Code stocké en base de données NPM | **Fichier .html** dans votre stockage serveur |
| **Performance** | Basique | **HSTS et HTTP/2** activés pour la vitesse |
| **Maintenance** | Copier-coller fastidieux | Édition directe du fichier en SSH/SFTP |

---

## Étape 1 : préparation du stockage et du fichier HTML

Le but est de rendre votre code accessible au conteneur NPM via un volume.

1. Connectez-vous en SSH à votre instance et créez le dossier :
`mkdir -p ~/npm/data/404/`
2. Créez votre fichier `index.html` :
`nano ~/npm/data/404/index.html`
3. Collez le code complet (disponible en fin de tutoriel) en veillant à personnaliser vos liens.

---

## Étape 2 : configuration de l'hôte "fallback"

Nous créons un hôte proxy dédié qui servira de point d'ancrage pour l'erreur.

1. Dans NPM, cliquez sur **Ajouter un hôte proxy**.
2. **Onglet Détails :** Configurez votre domaine de secours (ex: `fallback.blablalinux.be`) pointant vers `127.0.0.1` sur le port `80`.
> ![erreur-404-ssl-matrix-03.png](/page-erreur-404-statique-ssl-npm/erreur-404-ssl-matrix-03.png)


3. **Onglet SSL :** Sélectionnez votre certificat **Wildcard** et activez toutes les options de sécurité (Forcer SSL, HTTP/2, HSTS).
> ![erreur-404-ssl-matrix-04.png](/page-erreur-404-statique-ssl-npm/erreur-404-ssl-matrix-04.png)


4. **Configuration Nginx personnalisée :** Cliquez sur le petit engrenage en haut à droite (paramètres globaux de l'hôte) et insérez le bloc suivant :
> ![erreur-404-ssl-matrix-05.png](/page-erreur-404-statique-ssl-npm/erreur-404-ssl-matrix-05.png)



```nginx
location / {
    root /data/404;
    index index.html;
    add_header Content-Type text/html;
    try_files $uri $uri/ /index.html =404;
}

```

---

## Étape 3 : activation de la redirection par défaut

Dernière étape pour rediriger les visiteurs "perdus" vers cet hôte sécurisé.

1. Allez dans **Paramètres** (Settings) > **Site par défaut**.
2. Cliquez sur **Modifier** et choisissez **Redirection**.
3. Saisissez l'URL complète : `https://fallback.blablalinux.be`.
4. Effacez tout code restant dans la section "HTML personnalisé" pour éviter les conflits.

---

## Résultat final

Une fois configuré, toute URL inconnue affichera votre page Matrix avec un certificat SSL valide.

> ![erreur-404-ssl-matrix.png](/page-erreur-404-statique-ssl-npm/erreur-404-ssl-matrix.png)
> *Le rendu final sur navigateur bureau avec le cadenas SSL bien visible.*

> ![erreur-404-ssl-matrix-02.png](/page-erreur-404-statique-ssl-npm/erreur-404-ssl-matrix-02.png)
> *Le rendu parfaitement centré et responsive sur mobile.*
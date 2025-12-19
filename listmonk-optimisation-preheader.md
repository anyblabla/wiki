---
title: Optimisation des campagnes Listmonk - Le preheader invisible
description: Guide technique pour ajouter un texte d'aperçu (preheader) dans Listmonk via HTML et configurer des données d'abonné fictives pour tester la personnalisation des messages.
published: true
date: 2025-12-19T15:25:29.468Z
tags: listmonk, mail, header, preheader, newsletter, email, mailing
editor: markdown
dateCreated: 2025-12-19T15:19:31.897Z
---

L'outil de diffusion **Listmonk** ne propose pas nativement de champ pour définir le texte d'aperçu (preheader) d'un email. Ce texte est pourtant crucial pour augmenter le taux d'ouverture. Ce guide explique comment l'intégrer proprement via le code HTML et comment tester le rendu avec des données fictives.

---

## Pourquoi utiliser un preheader ?

Le preheader est la première ligne de texte qui s'affiche à la suite de l'objet dans un client de messagerie. Sans intervention, les clients mail affichent les premiers éléments textuels rencontrés, ce qui nuit au professionnalisme du message.

## Mise en œuvre technique

Le code suivant doit être inséré tout au début de votre message, avant même la bannière d'en-tête.

### Structure du code HTML

```html
<div style="display: none; max-height: 0px; overflow: hidden;">
  VOTRE_TEXTE_D_APERÇU_ICI
</div>
<div style="display: none; max-height: 0px; overflow: hidden;">
  &nbsp;&zwnj;&nbsp;&zwnj;&nbsp;&zwnj;&nbsp;&zwnj;&nbsp;&zwnj;&nbsp;&zwnj;&nbsp;&zwnj;&nbsp;&zwnj;&nbsp;&zwnj;&nbsp;&zwnj;&nbsp;&zwnj;&nbsp;&zwnj;&nbsp;&zwnj;&nbsp;&zwnj;
</div>

```

---

## Configuration des tests et métadonnées

Pour valider le rendu du preheader et de la personnalisation (comme le nom de l'abonné), il est indispensable d'utiliser les outils de prévisualisation de Listmonk.

### Métadonnées de la campagne

Dans l'onglet **Campagnes**, assurez-vous de bien définir :

* **Sujet :** L'objet visible qui doit être complété par votre preheader.
* **Nom :** Le nom interne pour votre organisation.
* **Type de corps :** HTML (obligatoire pour utiliser le preheader invisible).

### Données d'abonné fictives

Pour tester vos balises de personnalisation, allez dans l'onglet **Aperçu** de votre campagne. Listmonk permet de définir un **JSON d'abonné fictif**. Cela simule un profil réel sans envoyer de mail à votre base.

Exemple de JSON à utiliser pour tester le message :

```json
{
  "email": "test@exemple.be",
  "name": "Amaury",
  "attributes": {
    "ville": "Bruxelles",
    "interet": "Linux"
  }
}

```

### Repérage dans l'interface de Listmonk

```text
+-------------------------------------------------------------+
|  CAMPAGNE : [ Nouvelle mise à jour Wiki ]                   |
+-------------------------------------------------------------+
|  Corps | [ Aperçu ] | Paramètres | Envoi                    |
+-------------------------------------------------------------+
|                                                             |
|  [ Aperçu du rendu HTML ]                                   |
|  +-------------------------------------------------------+  |
|  | (Ici s'affiche votre mail avec la bannière)           |  |
|  +-------------------------------------------------------+  |
|                                                             |
|  [ Métadonnées de l'abonné (JSON pour le test) ]            |
|  +-------------------------------------------------------+  |
|  | {                                                     |  |
|  |   "name": "Amaury",                                   |  |
|  |   "email": "test@blablalinux.be"                      |  |
|  | }                                                     |  |
|  +-------------------------------------------------------+  |
|                                                             |
|  [ Bouton : Envoyer un test ] [ Bouton : Régénérer ]        |
+-------------------------------------------------------------+

```

> **Note :** Le texte du preheader n'apparaîtra **pas** dans l'aperçu visuel de Listmonk (car il est masqué par le CSS). Pour le valider réellement, vous devez cliquer sur **"Envoyer un test"** vers votre propre adresse mail.

## Bonnes pratiques

* **Longueur :** Visez entre 40 et 100 caractères pour le texte d'aperçu.
* **Complémentarité :** Le preheader doit compléter l'objet du mail, pas le répéter.
* **Nettoyage :** N'oubliez pas la chaîne `&nbsp;&zwnj;` pour éviter que le début du corps du mail ne vienne "polluer" l'aperçu dans la boîte de réception.

## Campagne

Exemple campagne de difusion Listmonk BlablaLinux, liste "Wiki" 👇

![listmonk-optimisation-preheader.png](/listmonk-optimisation-preheader/listmonk-optimisation-preheader.png)
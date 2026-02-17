---
title: Suppression des tags @TrackLink dans les archives Listmonk
description: Guide technique expliquant comment supprimer les tags @TrackLink des campagnes Listmonk via SQL. Comprenez pourquoi les archives affichent des erreurs et pourquoi l'interface UI ne suffit pas.
published: true
date: 2026-02-17T18:42:31.640Z
tags: listmonk, maintenance, postgresql, sql, tracklink, archive
editor: markdown
dateCreated: 2026-02-17T18:42:31.640Z
---

Ce guide explique pourquoi les liens dans les archives publiques de **Listmonk** peuvent devenir inopérants et pourquoi l'intervention en base de données est la seule solution technique possible.

## 1. Comprendre le problème : pourquoi l'erreur 404 ?

Lors de la consultation des archives, on peut constater que les liens contiennent le tag `@TrackLink` et renvoient une erreur (souvent une **404**).

### L'explication technique du suivi

C'est un comportement normal lié à la conception de Listmonk :

* **Le rôle du tag :** dans Listmonk, `@TrackLink` est un "placeholder". Lors de l'envoi d'un e-mail, ce tag est remplacé dynamiquement par un lien de suivi unique propre à chaque abonné.
* **Le problème des archives :** les archives sont publiques et consultées de manière anonyme. Comme il n'y a pas d'identifiant d'abonné à associer, Listmonk ne peut pas remplacer le tag. Le lien reste donc brut et invalide.

### Pourquoi ne pas passer par l'interface utilisateur ?

On pourrait être tenté d'éditer la campagne directement dans l'interface d'administration pour retirer le tag manuellement, mais c'est impossible pour deux raisons :

* **Contenu figé :** une fois qu'une campagne est publiée et envoyée, son contenu est verrouillé en lecture seule dans l'interface. Cela garantit l'intégrité de ce qui a été réellement expédié.
* **Modification forcée :** pour corriger l'affichage dans les archives sans pouvoir utiliser l'interface, il faut impérativement intervenir "sous le capot", au niveau de la base de données PostgreSQL.

## 2. Nettoyage d'une campagne spécifique

Si vous souhaitez tester l'opération sur une seule campagne (en utilisant l'identifiant récupéré dans l'URL de l'administration) :

```bash
# Remplacer [ID_CAMPAGNE] par le numéro correspondant
docker exec -i listmonk_db psql -U listmonk -d listmonk -c "UPDATE campaigns SET body = REPLACE(body, '@TrackLink', '') WHERE id = [ID_CAMPAGNE];"

```

## 3. Nettoyage global de toutes les campagnes

Pour traiter l'intégralité de l'historique (cas de la migration de BlablaLinux avec **42 campagnes**) :

```bash
docker exec -i listmonk_db psql -U listmonk -d listmonk -c "UPDATE campaigns SET body = REPLACE(body, '@TrackLink', '') WHERE body LIKE '%@TrackLink%';"

```

## 4. Résultat attendu

Le tag est supprimé directement dans la table `campaigns`. Listmonk n'essaie plus d'injecter de suivi sur ces anciens messages et les liens deviennent des liens standards, parfaitement fonctionnels pour n'importe quel visiteur.
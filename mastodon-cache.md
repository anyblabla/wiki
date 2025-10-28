---
title: Maintenance Mastodon : Nettoyage et optimisation des caches avec tootctl
description: Mastodon accumule divers types de données et de caches au fil du temps (images, médias, comptes distants, etc.). L'utilisation régulière des commandes tootctl est essentielle pour libérer de l'espace disque et maintenir la performance de votre instance.
published: true
date: 2025-10-28T12:46:40.372Z
tags: mastodon, cache, delete
editor: markdown
dateCreated: 2024-05-06T22:29:10.684Z
---

J'ai compris. Je vais améliorer la structure et la clarté de votre guide sur le nettoyage de Mastodon, en conservant **intégralement votre code** et vos liens (y compris l'image interne) à leur emplacement d'origine.

-----

# 🧹 Maintenance Mastodon : Nettoyage et optimisation des caches avec `tootctl`

Mastodon accumule divers types de données et de caches au fil du temps (images, médias, comptes distants, etc.). L'utilisation régulière des commandes `tootctl` est essentielle pour libérer de l'espace disque et maintenir la performance de votre instance.

-----

## I. Les principaux niveaux de nettoyage

Voici les commandes de base pour vider les différents niveaux de cache et supprimer les données obsolètes :

| Nettoyage | Commande `tootctl` | Description | Poids / Impact |
| :--- | :--- | :--- | :--- |
| **Vignettes en cache** | `tootctl cache clear` | Supprime les petites vignettes mises en cache (provenant d'autres serveurs). | Léger |
| **Vieux médias** | `tootctl media remove` | Supprime les médias (images, vidéos) plus anciens que la période de rétention configurée. | Lourd |
| **Médias non référencés** | `tootctl media remove-orphans` | Supprime les fichiers médias qui ne sont plus liés à aucun statut ou profil. | Variable |
| **Comptes distants obsolètes** | `tootctl accounts cull` | Supprime les comptes distants qui n'existent plus ou qui n'ont pas été actifs/utilisés depuis longtemps. | Lourd |
| **Statuts obsolètes** | `tootctl statuses remove` | Supprime les statuts (toots) qui ont expiré ou qui n'existent plus sur la fédération. | Très lourd |

  - Les principaux niveaux de nettoyage :

<!-- end list -->

1.  Vignettes en cache : **tootctl cache clear**
2.  Vieux média : **tootctl media remove**
3.  Médias non référencés : **tootctl media remove-orphans**
4.  Comptes distants qui n’existent plus (lourd) : **tootctl accounts cull**
5.  Statuts qui n’existent plus (très lourd) : **tootctl statuses remove**

-----

## II. Exécution des commandes de nettoyage

Toutes les commandes de nettoyage `tootctl` doivent être exécutées en tant qu'utilisateur `mastodon` depuis le répertoire `live` de votre instance.

  - Tous les caches peuvent être nettoyés par grâce à **tootctl**, utilisable comme suit :

<!-- end list -->

1.  **Passer en mode super-utilisateur (root) :**

<!-- end list -->

```plaintext
su -
```

2.  **Se déplacer dans le répertoire de l'instance Mastodon :**

<!-- end list -->

```plaintext
cd /var/www/mastodon/live
```

3.  **Exécuter chaque commande de nettoyage :**

Pour chaque commande, on utilise `sudo -u mastodon` (pour s'assurer que l'utilisateur `mastodon` exécute la tâche), on définit la variable d'environnement Rails sur `production`, et on indique le chemin d'accès au binaire `tootctl`.

```plaintext
sudo -u mastodon RAILS_ENV=production PATH=/opt/rbenv/versions/mastodon/bin bin/tootctl cache clear
```

```plaintext
sudo -u mastodon RAILS_ENV=production PATH=/opt/rbenv/versions/mastodon/bin bin/tootctl media remove
```

```plaintext
sudo -u mastodon RAILS_ENV=production PATH=/opt/rbenv/versions/mastodon/bin bin/tootctl media remove-orphans
```

```plaintext
sudo -u mastodon RAILS_ENV=production PATH=/opt/rbenv/versions/mastodon/bin bin/tootctl accounts cull 
```

```plaintext
sudo -u mastodon RAILS_ENV=production PATH=/opt/rbenv/versions/mastodon/bin bin/tootctl statuses remove
```

-----

## III. Notes importantes

**Calendrier de maintenance :** Les commandes `media remove` et `accounts cull` sont recommandées pour une exécution régulière (hebdomadaire ou mensuelle). Les commandes `statuses remove` et `accounts cull` peuvent prendre beaucoup de temps et sont gourmandes en ressources.

**Ressource :** Pour en savoir plus sur les pratiques de l'administrateur de l'instance :

![](/mastodon-cache/mastodon.blablalinux.png)

[https://mastodon.blablalinux.be/@blablalinux](https://mastodon.blablalinux.be/@blablalinux)
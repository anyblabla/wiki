---
title: Delete cache Mastodon
description: Faire un peu le ménage sur une instance Mastodon en supprimant certains éléments mis en cache.
published: true
date: 2025-04-19T23:32:12.030Z
tags: mastodon, cache, delete
editor: markdown
dateCreated: 2024-05-06T22:29:10.684Z
---

Mastodon a plusieurs nettoyages possibles pour vider les différents niveaux de cache.

-   Les principaux niveaux de nettoyage :

1.  Vignettes en cache : **tootctl cache clear**
2.  Vieux média : **tootctl media remove**
3.  Médias non référencés : **tootctl media remove-orphans**
4.  Comptes distants qui n’existent plus (lourd) : **tootctl accounts cull**
5.  Statuts qui n’existent plus (très lourd) : **tootctl statuses remove**

-   Tous les caches peuvent être nettoyés par grâce à **tootctl**, utilisable comme suit :

```plaintext
su -
```

```plaintext
cd /var/www/mastodon/live
```

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

![](/mastodon-cache/mastodon.blablalinux.png)

[https://mastodon.blablalinux.be/@blablalinux](https://mastodon.blablalinux.be/@blablalinux)
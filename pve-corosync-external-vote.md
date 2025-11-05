---
title: Proxmox - Support de vote externe Corosync (QDevice)
description: Déploiement du QDevice Net (votant externe Corosync) pour augmenter la haute disponibilité (HA) des clusters Proxmox VE, notamment ceux à 2 nœuds, en garantissant le quorum après une défaillance de nœud.
published: true
date: 2025-11-05T18:12:37.439Z
tags: proxmox, pve, corosync, qdevice, sync, quorum
editor: markdown
dateCreated: 2025-11-05T18:12:37.439Z
---

Cette page décrit comment déployer un **votant externe** (External Voter), connu sous le nom de **Corosync Quorum Device (QDevice)**, dans un cluster Proxmox VE. Lorsqu'il est correctement configuré, le cluster peut **résister à davantage de défaillances de nœuds** sans compromettre les propriétés de sécurité de la communication du cluster, augmentant ainsi significativement la **haute disponibilité (HA)**, même dans les petites configurations (par exemple, 2 nœuds avec 1 QDevice, soit 2+1).

## 1\. Principes de fonctionnement

Le support de vote externe repose sur l'interaction entre deux services :

  * Le démon **QDevice** : Il s'exécute sur **chaque nœud** Proxmox VE du cluster.
  * Le démon de **vote externe** (**QDevice Net**) : Il s'exécute sur un **serveur indépendant** qui agit comme arbitre.

### 1.1 Vue d'ensemble technique du QDevice

Le QDevice est un démon qui fournit un nombre configuré de votes au sous-système de quorum du cluster, en se basant sur la décision d'un arbitre tiers externe. Son rôle principal est de permettre au cluster de **maintenir le quorum** après la défaillance d'un nœud.

Le mécanisme garantit la sécurité en deux étapes :

1.  Le dispositif externe voit tous les nœuds et choisit de donner son vote à **une seule partition du cluster**.
2.  Il ne le fera que si cette partition est en mesure de **retrouver le quorum** après avoir reçu le vote additionnel.

Actuellement, seul **QDevice Net** est pris en charge comme arbitre tiers.

### 1.2 Exigences de QDevice Net

**QDevice Net** est le démon de vote externe. Il est conçu pour supporter plusieurs clusters et est particulièrement facile à déployer :

  * **Hôte externe indépendant** : Le démon doit s'exécuter sur un serveur **totalement distinct** des nœuds du cluster.
  * **Connexion TCP/IP** : Contrairement à Corosync lui-même, qui utilise l'UDP et est sensible à la faible latence, le QDevice se connecte au cluster via **TCP/IP**. Il peut donc fonctionner **en dehors du LAN** du cluster et n'est pas limité par les exigences de latence stricte de Corosync.
  * **Paquet requis** : L'hôte externe doit simplement avoir un **accès réseau** au cluster et disposer du paquet `corosync-qnetd` (disponible pour les hôtes basés sur Debian et autres distributions Linux).
  * **Configuration minimale** : Il est presque sans configuration et sans état. **Aucun fichier de configuration n'est nécessaire** sur l'hôte exécutant QDevice Net.

-----

## 2\. Configurations de cluster prises en charge

L'utilisation du QDevice est fortement dépendante du nombre de nœuds du cluster en raison de la manière dont les votes sont attribués.

| Nombre de Nœuds | Recommandation | Votes du QDevice | Conséquence principale |
| :--- | :--- | :--- | :--- |
| **Nombre pair (ex. 2, 4, 6)** | **Recommandé.** | **1 vote** additionnel. | Augmente la disponibilité. En cas de panne du QDevice, le cluster se retrouve dans la même position que sans QDevice (avec la nécessité d'une majorité). |
| **Cluster de 2 nœuds** | **Fortement recommandé** pour la HA (configuration 2+1). | **1 vote** additionnel. | Permet au nœud restant de conserver le quorum (1 nœud + 1 QDevice = 2 voix sur 3). |
| **Nombre impair (ex. 3, 5, 7)** | **Déconseillé** (sauf si les inconvénients sont compris). | $\text{N-1}$ votes (où $N$ est le nombre de nœuds). | Permet de survivre à $\text{N-2}$ pannes de nœuds (ex. 1 nœud restant dans un cluster de 3). |

### Mise en garde pour les clusters impairs

Pour les clusters de taille impaire, le QDevice fournit $(\mathbf{N-1})$ votes pour éviter un scénario de *split-brain*. Bien que cela permette théoriquement de tolérer $\mathbf{N-2}$ pannes de nœuds, deux inconvénients majeurs existent :

1.  **Le QDevice comme point de défaillance unique (SPOF)** : Si le démon QNet lui-même tombe en panne, le cluster perd immédiatement son droit à ces $(\mathbf{N-1})$ votes. Si, par exemple, un nœud du cluster tombe ensuite en panne, le cluster **perd immédiatement le quorum**.
2.  **Surcharge du nœud restant** : Le fait que presque tous les nœuds puissent tomber en panne peut entraîner une **récupération massive des services HA** sur le seul nœud restant, ce qui pourrait le surcharger. De plus, un cluster Ceph cesse de fournir des services si seuls $\frac{N-1}{2}$ nœuds ou moins restent en ligne, indépendamment du QDevice.

Vous devez comprendre ces implications et inconvénients avant de décider d'utiliser cette technologie dans une configuration de cluster de taille impaire.

-----

## 3\. Configuration de QDevice Net

Il est recommandé d'exécuter tout démon fournissant des votes à `corosync-qdevice` en tant qu'utilisateur non privilégié. Les paquets fournis par Proxmox VE et Debian sont déjà configurés pour cela. Le trafic entre le démon et le cluster **doit être chiffré** pour garantir une intégration sécurisée.

### Étape 1 : Installation des paquets

Installez le paquet `corosync-qnetd` sur votre serveur externe :

```bash
external# apt install corosync-qnetd
```

Installez le paquet `corosync-qdevice` sur **tous les nœuds** du cluster Proxmox VE :

```bash
pve# apt install corosync-qdevice
```

Assurez-vous que tous les nœuds du cluster sont **en ligne** à cette étape.

### Étape 2 : Configuration du QDevice

Exécutez la commande suivante sur **un seul** des nœuds Proxmox VE pour configurer le QDevice :

```bash
pve# pvecm qdevice setup <IP-DU-QDEVICE>
```

> L'adresse IP doit être celle de votre serveur externe où `corosync-qnetd` est installé.
>
> **Note :** La clé SSH du cluster sera automatiquement copiée vers le QDevice. Assurez-vous d'avoir configuré l'accès par clé pour l'utilisateur *root* sur votre serveur externe, ou autorisez temporairement la connexion *root* par mot de passe pendant cette phase de configuration. Si vous rencontrez une erreur telle que `Host key verification failed.`, l'exécution de `pvecm updatecerts` sur le nœud pourrait résoudre le problème.

### Étape 3 : Vérification

Une fois l'installation et la configuration terminées avec succès ("Done"), vous pouvez vérifier le statut du QDevice avec la commande :

```bash
pve# pvecm status
```

Le statut devrait afficher le QDevice ainsi que son vote, comme illustré dans cet exemple de sortie :

```
...

Votequorum information
~~~~~~~~~~~~~~~~~~~~~
Expected votes:   3
Highest expected: 3
Total votes:      3
Quorum:           2
Flags:            Quorate Qdevice

Membership information
~~~~~~~~~~~~~~~~~~~~~~
    Nodeid      Votes    Qdevice Name
    0x00000001      1    A,V,NMW 192.168.22.180 (local)
    0x00000002      1    A,V,NMW 192.168.22.181
    0x00000000      1            Qdevice 
```

Dans la section `Votequorum information`, la ligne `Flags: Quorate Qdevice` confirme que le QDevice est actif et participe au quorum. De plus, le QDevice apparaît avec un `Nodeid` de `0x00000000` et un vote dans la section `Membership information`.
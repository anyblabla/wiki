---
title: Gestion du démarrage et de la haute disponibilité sous Proxmox
description: Apprenez à configurer l'ordre de démarrage et les délais dans Proxmox pour stabiliser votre infrastructure, tout en maîtrisant les impacts critiques de la haute disponibilité sur ces réglages.
published: true
date: 2026-04-26T21:41:57.042Z
tags: proxmox, virtualisation, sysadmin, tutoriel, haute disponibilité
editor: markdown
dateCreated: 2026-04-26T21:40:09.063Z
---

Dans un environnement de serveur, laisser toutes les machines virtuelles (VM) et conteneurs (LXC) démarrer simultanément est une erreur courante. Cela peut saturer le processeur, bloquer les accès disques ou empêcher des services de fonctionner car leurs dépendances ne sont pas encore prêtes.

## Principes de l'ordre de démarrage et des délais

Proxmox permet de définir une hiérarchie de lancement pour que vos services s'activent de manière logique et progressive.

### L'ordre de priorité
Il s'agit de la position dans la file d'attente. Proxmox traite les chiffres du plus petit au plus grand.
* **Priorité basse (ex : 1, 2) :** pour les services d'infrastructure vitaux (DNS, pare-feu, reverse proxy).
* **Priorité haute (ex : 10, 20) :** pour les applications finales.

### Le délai de démarrage
Ce paramètre indique à Proxmox combien de secondes il doit attendre **après** avoir lancé une machine avant de passer à la suivante. Cela permet de lisser la consommation de ressources et de s'assurer qu'un service est pleinement opérationnel avant de lancer celui qui en dépend.

---

## Stratégie pour les stockages externes et matériels lents

Certains services dépendent de stockages externes (disques USB, NAS, montages réseau) qui mettent plus de temps à s'initialiser que les disques internes.

### La technique du tampon de sécurité
Si vous utilisez des périphériques lents, il est crucial d'insérer une **cassure temporelle** dans votre liste de démarrage.

* **Le principe :** appliquez un délai de **120 secondes** sur la dernière machine démarrant sur le stockage local rapide.
* **L'objectif :** ce temps de pause garantit que le système a fini de monter les disques externes avant que Proxmox ne tente de lancer les machines qui s'y trouvent. Sans ce tampon, les services sur stockage lent risquent de passer en erreur au démarrage.

---

## Interaction entre la haute disponibilité et le démarrage

La haute disponibilité (HA) est un outil puissant, mais elle modifie fondamentalement les règles de démarrage du cluster.

### L'annulation des réglages manuels
C'est le point de vigilance majeur : **dès qu'une machine est intégrée à la HA, Proxmox ignore les réglages d'ordre et de délai définis dans les options de la VM.**
C'est le gestionnaire de ressources HA (*LRM*) qui prend le contrôle. Il cherchera à démarrer la machine le plus rapidement possible sur un nœud disponible, sans respecter vos pauses chronométrées.

### Comment gérer l'ordre avec la HA ?
Puisque les délais en secondes ne sont plus pris en compte, vous devez utiliser la **priorité de groupe** dans la configuration HA :
1.  Créez des groupes HA distincts (ex : Infrastructure, Applications).
2.  Attribuez des niveaux de priorité aux ressources. Les priorités hautes seront traitées en premier.
3.  **Attention :** la HA ne sait pas "attendre". Elle lance les priorités par blocs. Si un service HA dépend d'un service non HA qui a un délai de 30 secondes, le service HA risque de démarrer trop tôt et d'échouer.

---

## Recommandations pour une infrastructure résiliente

1.  **Démarrer l'infrastructure en premier :** le DNS et le proxy doivent toujours être en tête de liste.
2.  **Éviter le mélange des genres :** essayez de ne pas faire dépendre une machine en HA d'une machine en démarrage manuel.
3.  **Vérifier les stockages :** une machine en HA doit avoir accès à ses données sur tous les nœuds du cluster. Un disque USB physiquement branché sur un seul serveur rend la HA inutile, voire dangereuse.
4.  **Prévoir les services lourds :** les bases de données et les outils complexes doivent disposer de délais plus longs (45-60s) pour stabiliser leurs processus avant la suite du boot.

---

> **Auteur** : ce guide est proposé par **Amaury aka BlablaLinux**. Retrouvez l'ensemble de mes services sur [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).
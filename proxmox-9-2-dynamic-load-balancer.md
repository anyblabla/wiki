---
title: Proxmox VE 9.2 - Configurer le Dynamic Load Balancer (Équilibreur de charge)
description: Découvrez le Dynamic Load Balancer de Proxmox VE 9.2 et apprenez à configurer ses options de rebalancement.
published: true
date: 2026-05-21T19:52:09.916Z
tags: proxmox, cluster, ha, tutoriel, pve-9.2, load-balancer, crs
editor: markdown
dateCreated: 2026-05-21T18:48:27.725Z
---

Le CRS (Cluster Resource Scheduler) de Proxmox VE s'enrichit d'une fonctionnalité majeure : un **Dynamic Load Balancer (Équilibreur de charge dynamique)** intégré directement au cœur du cluster. Ce guide détaille son fonctionnement et la signification des paramètres de configuration.

## 1. Présentation du Dynamic Load Balancer

Auparavant, le CRS de Proxmox se basait principalement sur des règles statiques ou des configurations de groupes HA (High Availability) figées. Avec cette évolution, le scheduler intègre désormais un **mode dynamique** complet.

Le système utilise les métriques d'utilisation des ressources **en temps réel** des nœuds physiques et des invités (VM/CT) pour prendre ses décisions de placement ou de déplacement.

### Migration automatique des VM/CT sous HA

Le gros point fort mentionné dans le guide d'administration est sa capacité à collaborer avec la pile HA (Haute Disponibilité) :

* L'équilibreur de charge analyse en continu le déséquilibre (*imbalance*) global des ressources entre les différents nœuds du cluster.
* Si un nœud commence à saturer (CPU/RAM) alors qu'un autre a de la marge, le Load Balancer peut déclencher **automatiquement des migrations à chaud** des invités managés par la HA pour rééquilibrer la charge.
* **Sécurité importante :** Le Load Balancer respecte strictement toutes les règles de HA définies par l'utilisateur (comme les restrictions de groupes ou de priorités). Pas de risque qu'une VM migre là où elle n'a pas le droit d'aller.

### Un contrôle granulaire pour l'administrateur

Pour éviter que les machines virtuelles ne passent leur temps à migrer au moindre pic de charge (ce qui surchargerait le réseau du cluster), Proxmox a prévu des paramètres de réglage précis :

* Les options de configuration sont accessibles directement depuis l'interface Web (dans les options du Datacenter ou le panneau HA).
* On peut configurer le comportement et surtout la **sensibilité (l'agressivité)** de l'équilibreur de charge via différents paramètres ajustables pour s'adapter à la taille et aux spécificités de l'infrastructure.

---

## 2. Comprendre les Ratios et Pourcentages de Configuration

C'est justement là que se trouve toute l'intelligence mathématique du nouveau système. La documentation technique de Proxmox (via les directives du fichier datacenter.cfg) détaille précisément comment ces pourcentages et ces coefficients numériques déterminent si le cluster est « équilibré » ou s'il doit déclencher des migrations.

Voici comment le système calcule et utilise ces valeurs pour éviter de faire n'importe quoi :

### Le calcul de l'asymétrie globale : ha-auto-rebalance-threshold

Le système évalue en continu un score d'asymétrie (l'*imbalance* ou le déséquilibre du cluster). C'est un coefficient qui va généralement de 0 (parfaitement équilibré) à plus de 1 (totalement asymétrique).

* Par défaut, le seuil d'activation (ha-auto-rebalance-threshold) est fixé à **0.3 (soit 30%)**.
* **Ce que ça signifie :** Si l'écart de charge (calculé sur le CPU et la mémoire RAM en mode dynamique) entre le nœud le plus chargé et le nœud le moins chargé dépasse ce ratio d'asymétrie de 30%, le mécanisme de rebalancement considère que le cluster est en situation d'anomalie et se prépare à agir.

### Le gain minimal obligatoire : ha-auto-rebalance-margin

Migrer une machine virtuelle à chaud consomme de la bande passante réseau et un peu de ressources CPU sur les hôtes. Proxmox ne veut pas déplacer une VM pour un gain dérisoire. C'est là qu'intervient la marge d'amélioration relative (ha-auto-rebalance-margin).

* Par défaut, elle est réglée à **0.1 (soit 10%)**.
* **Ce que ça signifie :** Avant de valider et d'exécuter la migration d'une VM, l'algorithme simule le déplacement. Il s'assure que cette migration va réduire le déséquilibre global du cluster d'**au moins 10%** par rapport à la situation actuelle. Si le gain estimé est inférieur à 10%, la migration est annulée car jugée non rentable face au coût réseau du déplacement.

### La temporisation anti-panique : ha-auto-rebalance-hold-duration

Pour éviter qu'un simple pic de CPU temporaire (comme le lancement d'une mise à jour ou un backup lourd sur une VM) ne provoque une vague de migrations inutiles, Proxmox utilise une temporisation basée sur les cycles (*rounds*) de la pile HA.

* Par défaut, la valeur ha-auto-rebalance-hold-duration est fixée à **3 rounds**.
* **Ce que ça signifie :** Le seuil d'asymétrie de 30% mentionné plus haut doit être dépassé et **rester critique pendant au moins 3 cycles HA consécutifs** avant que l'équilibreur ne s'autorise à planifier la première migration. Si la charge redescend d'elle-même au bout de deux cycles, rien ne bouge.

---

## 3. Synthèse des options (datacenter.cfg)

Voici le récapitulatif des directives introduites à vérifier ou ajuster dans votre configuration :

| Option | Valeur par défaut | Description |
| --- | --- | --- |
| ha-auto-rebalance | 1 | 1 pour activé, 0 pour désactivé. |
| ha-auto-rebalance-threshold | 0.3 | Déclenchement dès 30% d'asymétrie globale entre les nœuds. |
| ha-auto-rebalance-margin | 0.1 | Exige un gain d'au moins 10% sur l'asymétrie pour valider la migration. |
| ha-auto-rebalance-hold-duration | 3 | Attendre 3 cycles (rounds) de confirmation HA avant d'agir. |

L'avantage, c'est que si le comportement par défaut s'avère trop sensible pour l'infrastructure ou, au contraire, pas assez réactif, on peut affiner ces pourcentages et ces ratios directement pour ajuster le curseur selon ses besoins.

---

## 4. Liens utiles et documentation officielle

Pour approfondir le sujet ou vérifier les options de syntaxe exactes, vous pouvez consulter la documentation officielle de Proxmox :

* [Proxmox VE Administration Guide](https://pve.proxmox.com/pve-docs/pve-admin-guide.html) : Le manuel complet officiel pour la gestion globale de votre infrastructure et de vos datacenters.
* [Proxmox VE Administration Guide - CRS Load Balancer](https://pve.proxmox.com/pve-docs/pve-admin-guide.html#_crs_load_balancer) : La section du manuel décrivant en détail le fonctionnement général et l'intégration du répartiteur de charge au Cluster Resource Scheduler.
* [Proxmox VE Datacenter Configuration Manual](https://pve.proxmox.com/pve-docs/datacenter.cfg.5.html) : La page de manuel technique de référence détaillant toutes les variables et directives applicables au fichier de configuration /etc/pve/datacenter.cfg.

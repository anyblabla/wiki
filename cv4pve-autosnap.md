---
title: Automatiser les instantanés (Snapshots) Proxmox VE avec CV4PVE-AUTOSNAP
description: Détails de l'installation et de la configuration de l'outil CV4PVE-AUTOSNAP (Corsinvest) dans un LXC pour automatiser les instantanés (Snapshots) de vos VM et CT sur Proxmox VE.
published: true
date: 2025-10-28T12:04:11.353Z
tags: proxmox, snapshot
editor: markdown
dateCreated: 2024-12-28T18:49:39.139Z
---

## 💡 Contexte et choix de l'outil

Pour la prise d'instantanés des invités sur [Proxmox VE](https://www.proxmox.com/en/proxmox-virtual-environment/overview), plusieurs méthodes existent :

  * L'interface web ([GUI](https://w.wiki/CZ6F)) de Proxmox.
  * La ligne de commande ([CLI](https://w.wiki/6gSf)) avec l'outil natif [PVESH](https://pve.proxmox.com/pve-docs/chapter-pvesh.html).
  * Des outils tiers, tels que [CV4PVE-ADMIN](https://corsinvest.it/cv4pve-admin-proxmox/) (GUI multiclusters) qui utilise, en coulisses, **CV4PVE-AUTOSNAP**.

> J'utilise CV4PVE-ADMIN, qui est un gestionnaire multiclusters. Mais je n'utilise que la fonction de prise automatique d'instantanés. C'est sortir la grosse artillerie pour pas grand-chose.
>
> Je vais donc vous montrer comment utiliser CV4PVE-AUTOSNAP. Non pas sur l'hôte Proxmox lui-même, mais en l'isolant sur un conteneur ([LXC](https://w.wiki/CXkQ)).
>
> Personnellement, j'ai déployé un conteneur [Debian 12 Bookworm](https://www.debian.org/releases/bookworm/).

-----

## I. Installation et préparation dans le conteneur LXC

Toutes les commandes suivantes doivent être exécutées à l'intérieur de votre conteneur LXC (Debian/Ubuntu) dédié.

### 1\. Téléchargement du binaire

Récupérez la dernière version de CV4PVE-AUTOSNAP disponible à [cette adresse](https://github.com/Corsinvest/cv4pve-autosnap/releases). *(Version 1.15.0 au moment où j'écris ces lignes)* :

```plaintext
wget https://github.com/Corsinvest/cv4pve-autosnap/releases/download/v1.15.0/cv4pve-autosnap-linux-x64.zip
```

### 2\. Décompression et installation

Installez l'utilitaire `unzip`, décompressez l'archive, et rendez le binaire exécutable.

```plaintext
apt install unzip
```

```plaintext
unzip cv4pve-autosnap-linux-x64.zip
```

```plaintext
chmod +x cv4pve-autosnap
```

### 3\. Finalisation

Déplacez le fichier binaire dans le répertoire `/bin/` pour pouvoir l'exécuter de n'importe où dans le système :

```plaintext
mv cv4pve-autosnap /bin/
```

Créez l'emplacement pour les fichiers log qui seront générés par les tâches Cron :

```plaintext
mkdir /root/autosnap-logs
```

### 4\. Vérification

Pour obtenir de l'aide sur les options disponibles :

```plaintext
cv4pve-autosnap --help
```

-----

## II. Création du script d'instantanés (Snapshots)

Nous allons maintenant créer un premier script pour la prise d'instantanés à des heures précises (9h et 21h dans cet exemple).

### 1\. Structure de la commande de base

La commande principale de `cv4pve-autosnap` utilise la structure suivante, où toutes les informations de connexion et de rétention sont fournies :

`cv4pve-autosnap --host=XXX.XXX.X.XXX --username=root@pam --password=XXXXXXXX --vmid=XXX snap --label=XXXXX --keep=X`

> **À partir d'ici, toutes les commandes sont à adapter \!**

### 2\. Création du script `autosnap-921.sh`

Nous créons notre script, que l'on va nommer ici `autosnap-921.sh` :

```plaintext
nano autosnap-921.sh 
```

Collez le contenu suivant :

```plaintext
#!/bin/bash
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=""
############# SNAPSHOT SCRIPT ITSELF ######################
echo " " >> /root/autosnap-logs/921-$(date "+%Y-%m-%d").log
echo "###NEW JOB STARTS HERE###" >> /root/autosnap-logs/921-$(date "+%Y-%m-%d").log
cv4pve-autosnap --host=XXX.XXX.X.XXX --username=root@pam --password=XXXXXXXX --vmid=all snap --label=-921- --keep=2 >> /root/autosnap-logs/921-$(date "+%Y-%m-%d").log
echo "###NEW JOB ENDS HERE###" >> /root/autosnap-logs/921-$(date "+%Y-%m-%d").log
```

### 3\. Personnalisation des arguments

Vous devez adapter les arguments de la commande pour qu'elle corresponde à votre environnement Proxmox VE :

| Argument | Description | Valeurs possibles |
| :--- | :--- | :--- |
| `--host=` | Adresse IP du nœud Proxmox. | `XXX.XXX.X.XXX` |
| `--host=` | **Pour un cluster**, l'adresse IP du nœud maître, ou l'ensemble des adresses IP des nœuds. | `--host='XXX.XXX.X.XXX,XXX.XXX.X.XXX,XXX.XXX.X.XXX'` |
| `--username=` | Identifiant de connexion à Proxmox. | `--username=root@pam` **OU** `--username=XXXX@pve` |
| `--password=` | Mot de passe du compte spécifié. | `--password=XXXXXXXX` |
| `--vmid=` | ID de l'invité. Utilisez `all` pour tous les invités. | `--vmid=all` |
| `--vmid="all,-XXX"` | **Pour ignorer un invité**, utilisez cette syntaxe pour exclure l'ID spécifié. | `--vmid="all,-XXX"` |
| `--label=` | Nom de l'instantané (utilisé pour la rétention et le nettoyage). | `--label='-921-'` |
| `--keep=` | Spécifie le nombre d'instantanés avec ce label à conserver (rétention). | `--keep=X` |

> **Personnellement, je précise l'ensemble des adresses IP des nœuds \! Si une machine est inaccessible, les instantanés des invités sur les machines accessibles seront effectués.**

### 4\. Rendre le script exécutable

Sauvegardez le fichier (avec **CTRL + X**, puis **O**, puis **ENTER** sous `nano`), puis rendez le script exécutable :

```plaintext
chmod +x autosnap-921.sh
```

-----

## III. Planification avec Cron

Nous allons créer une tâche Cron pour appeler ce script à des intervalles réguliers.

### 1\. Ouvrir le crontab

Ouvrez le fichier de planification des tâches Cron (utilisez `crontab -e` pour l'utilisateur actuel, qui est idéalement `root` dans le conteneur) :

```plaintext
crontab -e
```

### 2\. Ajouter la planification

Collez la ligne suivante pour exécuter le script tous les jours à 9 h 00 et 21 h 00 :

```plaintext
0 9,21 * * * /root/autosnap-921.sh
```

| Champ | Valeur | Explication |
| :--- | :--- | :--- |
| **Minute** | `0` | À la 0ème minute (début de l'heure) |
| **Heure** | `9,21` | À 9h et à 21h |
| **Jour du mois** | `*` | Tous les jours du mois |
| **Mois** | `*` | Tous les mois |
| **Jour de la semaine** | `*` | Tous les jours de la semaine |

-----

## IV. Exemples de maintenance et de gestion

L'outil permet de réaliser d'autres tâches utiles en plus de la création d'instantanés.

### 1\. Suppression des instantanés (Nettoyage)

Pour supprimer l'ensemble des instantanés qui possèdent le label `-921-` sur l'ensemble des invités (en définissant le seuil de rétention à 0), utilisez cette commande :

```bash
cv4pve-autosnap --host=XXX.XXX.X.XXX --username=root@pam --password=XXXXXXXX --vmid=all clean --label='-921-' --keep=0
```

> **Attention :** Cette commande supprime tous les instantanés du label spécifié. **À adapter \!**

### 2\. Affichage du statut

Pour obtenir des informations sur les instantanés existants effectués par l'outil, utilisez cette commande :

```bash
cv4pve-autosnap --host=XXX.XXX.X.XXX --username=root@pam --password=XXXXXXXX --vmid=all status
```

> **À noter :** Vous pouvez encapsuler ces deux commandes au travers d'un fichier `.sh` (script) pour plus de facilité d'exécution ou de planification.

-----

## V. Aller plus loin : stratégies de rétention avancées

Vous pouvez répliquer le modèle de rétention horaire/quotidien/hebdomadaire/mensuel/annuel que propose l'outil de sauvegarde natif de Proxmox VE.

### 1\. Création de scripts pour chaque intervalle

Créez des fichiers `.sh` dédiés pour chaque intervalle de sauvegarde, en adaptant le label et la rétention (`--keep=X`).

| Nom du Script | Fréquence | Label Ex. | Rétention (`--keep=`) |
| :--- | :--- | :--- | :--- |
| `autosnap-hourly.sh` | Chaque heure | `-hourly-` | `--keep=3` |
| `autosnap-daily.sh` | Chaque jour | `-daily-` | `--keep=7` |
| `autosnap-weekly.sh` | Chaque semaine | `-weekly-` | `--keep=4` |
| `autosnap-monthly.sh` | Chaque mois | `-monthly-` | `--keep=2` |
| `autosnap-yearly.sh` | Chaque année | `-yearly-` | `--keep=1` |

> **Contenu typique d'un script (ex. : `autosnap-hourly.sh`) :**
>
> ```bash
> #!/bin/bash
> #... (PATH et MAILTO)
> cv4pve-autosnap --host=XXX.XXX.X.XXX --username=root@pam --password=XXXXXXXX --vmid=all snap --label=-hourly- --keep=3 >> /root/autosnap-logs/hourly-$(date "+%Y-%m-%d").log
> ```

### 2\. Planification des tâches Cron

Vous créez une tâche Cron pour chaque fichier `.sh` dans votre `crontab -e` :

```cron
# Tâches planifiées
0 * * * * /root/autosnap-hourly.sh
10 0 * * * /root/autosnap-daily.sh
20 0 * * 0 /root/autosnap-weekly.sh
30 0 1 * * /root/autosnap-monthly.sh
40 0 1 1 * /root/autosnap-yearly.sh
```

Ceci reproduit fidèlement la stratégie de rétention que l'on peut obtenir avec l'outil de créations de sauvegardes de Proxmox VE (onglet “Rétention”) :

> **Explication des tâches Cron :**
>
>   * **Hourly :** Toutes les heures (à `0` minute).
>   * **Daily :** Tous les jours à `0` heure `10` minute.
>   * **Weekly :** Tous les dimanches (jour `0`) à `0` heure `20` minute.
>   * **Monthly :** Le premier jour (`1`) de chaque mois, à `0` heure `30` minute.
>   * **Yearly :** Le premier jour (`1`) du premier mois (`1`, donc le 1er janvier) à `0` heure `40` minute.

### 3\. Exemples de fichiers (référence)

> Chez moi, toutes les tâches sont créées, mais pas actives \! Il suffit pour ça de décommenter dans crontab avec :
>
> `crontab -e`

> Voici l'ensemble de mes fichiers .sh :
>

> Voici en captures le contenu de l'ensemble des fichiers .sh que j'ai créés :

| Fichier | Image | Fichier | Image |
| :--- | :--- | :--- | :--- |
| `autosnap-921.sh` |  | `autosnap-cleanall-921.sh` |  |
| `autosnap-hourly.sh` |  | `autosnap-cleanall-hourly.sh` |  |
| `autosnap-daily.sh` |  | `autosnap-cleanall-daily.sh` |  |
| `autosnap-weekly.sh` |  | `autosnap-cleanall-weekly.sh` |  |
| `autosnap-monthly.sh` |  | `autosnap-cleanall-monthly.sh` |  |
| `autosnap-yearly.sh` |  | `autosnap-cleanall-yearly.sh` |  |
| `autosnap-status.sh` |  | | |

-----

## VI. Exemple de fichier log

Le fichier log est essentiel pour valider que les opérations se sont déroulées correctement.

```plaintext
###NEW JOB STARTS HERE###
ACTION Snap
PVE Version:     8.3.2
VMs:             all
Label:           -921-
Keep:            2
State:           False
Only running:    False
Timeout:         30 sec.
Timestamp format: yyMMddHHmmss
Max % Storage :   95%
                 Storage      Type     Valid   Used %    Disk Size  Disk Usage
        pvenocl/directory        dir       Ok      14.1  457.38 GB     64.35 GB
            pvenocl/local        dir       Ok      20.8   47.58 GB       9.9 GB
              pvenocl/zfs    zfspool       Ok      11.5  224.81 GB     25.92 GB
          pvenocl/zfs-usb    zfspool       Ok      10.9  449.63 GB     49.12 GB
       pvenocl2/directory        dir       Ok         0    219 GB         44 KB
            pvenocl2/local        dir       Ok       8.6  108.2 GB       9.31 GB
              pvenocl2/zfs    zfspool       Ok      19.4  449.63 GB     87.42 GB
          pvenocl2/zfs-usb    zfspool       Ok       1.4  449.63 GB       6.23 GB
            pvenocl3/local        dir       Ok       3.5  210.64 GB       7.31 GB
              pvenocl3/zfs    zfspool       Ok        10  449.63 GB     45.05 GB
          pvenocl3/zfs-usb    zfspool       Ok       2.9  899.25 GB     26.03 GB
----- VM 100 qemu running -----
Create snapshot: auto-921-241229210002
VM execution 00:00:01.2017896
----- VM 103 qemu stopped -----
Skip VM is template
----- VM 104 lxc stopped -----
Skip VM is template
----- VM 105 qemu running -----
Create snapshot: auto-921-241229210004
VM execution 00:00:00.5710229
----- VM 106 qemu running -----
Create snapshot: auto-921-241229210004
VM execution 00:00:00.5707401
----- VM 107 qemu running -----
Create snapshot: auto-921-241229210005
VM execution 00:00:00.5674122
----- VM 108 qemu stopped -----
Skip VM is template
----- VM 109 qemu stopped -----
Skip VM is template
----- VM 110 qemu stopped -----
Skip VM is template
----- VM 111 qemu stopped -----
Skip VM is template
----- VM 112 qemu stopped -----
Skip VM is template
----- VM 113 qemu stopped -----
Skip VM is template
----- VM 114 qemu stopped -----
Skip VM is template
----- VM 115 qemu stopped -----
Skip VM is template
----- VM 116 qemu stopped -----
Skip VM is template
----- VM 117 lxc stopped -----
Skip VM is template
----- VM 118 qemu stopped -----
Skip VM is template
----- VM 119 qemu stopped -----
Skip VM is template
----- VM 120 qemu stopped -----
Skip VM is template
----- VM 124 qemu running -----
Create snapshot: auto-921-241229210005
VM execution 00:00:01.6069741
----- VM 126 lxc stopped -----
Skip VM is template
----- VM 128 lxc stopped -----
Skip VM is template
----- VM 132 lxc running -----
Create snapshot: auto-921-241229210007
VM execution 00:00:02.1245899
----- VM 135 qemu stopped -----
Skip VM is template
----- VM 136 lxc running -----
Create snapshot: auto-921-241229210009
VM execution 00:00:03.6640259
----- VM 137 qemu running -----
Create snapshot: auto-921-241229210013
VM execution 00:00:03.1440733
----- VM 142 qemu running -----
Create snapshot: auto-921-241229210016
VM execution 00:00:00.6068154
----- VM 144 lxc running -----
Create snapshot: auto-921-241229210016
VM execution 00:00:00.6045220
----- VM 146 lxc running -----
Create snapshot: auto-921-241229210017
VM execution 00:00:01.6246597
----- VM 152 lxc stopped -----
Create snapshot: auto-921-241229210019
VM execution 00:00:00.5677819
----- VM 157 lxc stopped -----
Skip VM is template
----- VM 158 lxc stopped -----
Skip VM is template
----- VM 159 lxc stopped -----
Skip VM is template
----- VM 161 lxc stopped -----
Skip VM is template
----- VM 163 lxc stopped -----
Skip VM is template
----- VM 165 lxc stopped -----
Skip VM is template
----- VM 175 qemu running -----
Create snapshot: auto-921-241229210019
VM execution 00:00:00.5823049
----- VM 177 lxc running -----
Create snapshot: auto-921-241229210020
VM execution 00:00:01.0895075
----- VM 101 lxc running -----
Create snapshot: auto-921-241229210021
VM execution 00:00:00.6403238
----- VM 102 lxc running -----
Create snapshot: auto-921-241229210022
VM execution 00:00:00.6529256
----- VM 121 lxc running -----
Create snapshot: auto-921-241229210022
VM execution 00:00:00.6391276
----- VM 122 lxc stopped -----
Create snapshot: auto-921-241229210023
VM execution 00:00:02.2441079
----- VM 123 lxc running -----
Create snapshot: auto-921-241229210025
VM execution 00:00:00.8560169
----- VM 131 lxc running -----
Create snapshot: auto-921-241229210026
VM execution 00:00:02.2622291
----- VM 133 lxc stopped -----
Create snapshot: auto-921-241229210028
VM execution 00:00:00.6562011
----- VM 134 lxc running -----
Create snapshot: auto-921-241229210029
VM execution 00:00:02.2574055
----- VM 138 lxc running -----
Create snapshot: auto-921-241229210031
VM execution 00:00:00.6279803
----- VM 141 lxc running -----
Create snapshot: auto-921-241229210032
VM execution 00:00:00.6631722
----- VM 143 lxc stopped -----
Create snapshot: auto-921-241229210033
VM execution 00:00:00.7071216
----- VM 145 lxc stopped -----
Create snapshot: auto-921-241229210033
Error in task: snapshot feature is not available
----- VM 147 lxc running -----
Create snapshot: auto-921-241229210034
VM execution 00:00:04.4050547
----- VM 150 lxc running -----
Create snapshot: auto-921-241229210039
VM execution 00:00:00.6856690
----- VM 151 lxc running -----
Create snapshot: auto-921-241229210039
VM execution 00:00:00.6992125
----- VM 155 lxc running -----
Create snapshot: auto-921-241229210040
VM execution 00:00:00.6955720
----- VM 156 lxc running -----
Create snapshot: auto-921-241229210041
VM execution 00:00:00.6566546
----- VM 160 lxc running -----
Create snapshot: auto-921-241229210041
VM execution 00:00:00.6361933
----- VM 162 lxc running -----
Create snapshot: auto-921-241229210042
VM execution 00:00:05.4351452
----- VM 164 lxc running -----
Create snapshot: auto-921-241229210048
VM execution 00:00:01.7351612
----- VM 176 lxc running -----
Create snapshot: auto-921-241229210049
VM execution 00:00:01.7243946
----- VM 186 lxc stopped -----
Create snapshot: auto-921-241229210051
VM execution 00:00:00.6461545
----- VM 125 lxc running -----
Create snapshot: auto-921-241229210052
VM execution 00:00:00.6596117
----- VM 127 lxc running -----
Create snapshot: auto-921-241229210052
VM execution 00:00:00.6743573
----- VM 129 lxc running -----
Create snapshot: auto-921-241229210053
VM execution 00:00:05.4295270
----- VM 130 lxc running -----
Create snapshot: auto-921-241229210059
VM execution 00:00:00.6699872
----- VM 139 lxc running -----
Create snapshot: auto-921-241229210059
VM execution 00:00:00.6590028
----- VM 140 lxc running -----
Create snapshot: auto-921-241229210100
VM execution 00:00:00.6689948
----- VM 148 lxc running -----
Create snapshot: auto-921-241229210101
VM execution 00:00:00.6738950
----- VM 149 lxc running -----
Create snapshot: auto-921-241229210101
VM execution 00:00:01.7348000
----- VM 153 lxc running -----
Create snapshot: auto-921-241229210103
VM execution 00:00:00.6645257
----- VM 154 lxc running -----
Create snapshot: auto-921-241229210104
VM execution 00:00:02.2298501
----- VM 168 lxc running -----
Create snapshot: auto-921-241229210106
VM execution 00:00:00.6576744
----- VM 172 lxc running -----
Create snapshot: auto-921-241229210107
VM execution 00:00:00.6656634
----- VM 173 lxc running -----
Create snapshot: auto-921-241229210107
VM execution 00:00:00.6735121
----- VM 174 lxc running -----
Create snapshot: auto-921-241229210108
VM execution 00:00:00.6571412
----- VM 178 lxc running -----
Create snapshot: auto-921-241229210109
VM execution 00:00:00.6541993
----- VM 187 lxc running -----
Create snapshot: auto-921-241229210109
VM execution 00:00:05.4500119
----- VM 189 lxc running -----
Create snapshot: auto-921-241229210115
VM execution 00:00:05.4209600
Total execution 00:01:18.3608764
###NEW JOB ENDS HERE###
```

-----

## VII. Ressources et générateurs Cron

Pour faciliter la création de vos planifications :

  * [https://crontab.guru](https://crontab.guru)
  * [https://crontab.cronhub.io](https://crontab.cronhub.io)
  * [https://www.freeformatter.com/cron-expression-generator-quartz.html](https://www.freeformatter.com/cron-expression-generator-quartz.html)
  * [https://crontab-generator.org](https://crontab-generator.org)
  * [https://www.uptimia.com/cron-expression-generator](https://www.uptimia.com/cron-expression-generator)
  * [https://hyperping.com/cron-expression-generator](https://hyperping.com/cron-expression-generator)
  * [https://quickref.me/cron.html](https://quickref.me/cron.html)
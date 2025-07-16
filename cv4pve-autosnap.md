---
title: CV4PVE AUTOSNAP
description: Outil d'instantané automatique pour Proxmox VE.
published: true
date: 2025-07-16T22:15:34.971Z
tags: proxmox, snapshot
editor: markdown
dateCreated: 2024-12-28T18:49:39.139Z
---

# Explications

Pour la prise d'instantanés des machines virtuelles (VM) et des conteneurs (CT) sur [Proxmox VE](https://www.proxmox.com/en/proxmox-virtual-environment/overview), il existe plusieurs solutions.

Passer par…

-   l'interface web ([GUI](https://w.wiki/CZ6F))
-   la ligne de commande ([CLI](https://w.wiki/6gSf)) avec [PVESH](https://pve.proxmox.com/pve-docs/chapter-pvesh.html)
-   des outils tiers

En parlant d'outils tiers, il existe [CV4PVE-ADMIN](https://corsinvest.it/cv4pve-admin-proxmox/) (GUI) qui utilise, pour la prise des instantanés automatiques, [CV4PVE-AUTOSNAP](https://github.com/Corsinvest/cv4pve-autosnap). Ce dernier est disponible seul pour être utilisé en ligne de commande.

[J'utilise CV4PVE-ADMIN](https://x.com/BlablaLinux/status/1786173587134550115), qui est un gestionnaire multiclusters. Mais je n'utilise que la fonction de prise automatique d'instantanés. C'est sortir la grosse artillerie pour pas grand-chose.

Je vais donc vous montrer comment utiliser CV4PVE-AUTOSNAP. Non pas sur l'hôte Proxmox lui-même, mais en l'isolant sur un conteneur ([LXC](https://w.wiki/CXkQ)).

Personnellement, j'ai déployé un conteneur [Debian 12 Bookworm](https://www.debian.org/releases/bookworm/).

# Pas à pas

-   Récupération de la dernière version de CV4PVE-AUSNAP disponible à [cette adresse](https://github.com/Corsinvest/cv4pve-autosnap/releases). Version 1.15.0 au moment où j'écris ces lignes :

```plaintext
wget https://github.com/Corsinvest/cv4pve-autosnap/releases/download/v1.15.0/cv4pve-autosnap-linux-x64.zip
```

-   Installation de l'utilitaire unzip :

```plaintext
apt install unzip
```

-   Décompression de l'archive téléchargée :

```plaintext
unzip cv4pve-autosnap-linux-x64.zip
```

-   Nous rendons le binaire obtenu (cv4pve-autosnap) exécutable :

```plaintext
chmod +x cv4pve-autosnap
```

-   Nous déplaçons le fichier binaire dans le répertoire /bin/ afin de pouvoir l'utiliser partout :

```plaintext
mv cv4pve-autosnap /bin/
```

-   On crée l'emplacement pour les fichiers log :

```plaintext
mkdir autosnap-logs
```

-   Pour obtenir de l'aide, on utilise l'option help :

```plaintext
cv4pve-autosnap --help
```

-   La structure de la commande qui nous allons utiliser est la suivante :

`cv4pve-autosnap --host=XXX.XXX.X.XXX --username=root@pam --password=XXXXXXXX --vmid=XXX snap --label=XXXXX --keep=X`

**À partir d'ici, toutes les commandes sont à adapter !**

-   Nous pouvons créer notre script, que l'on va nommer ici autosnap-921.sh pour une prise d'instantané à 9 h et 21 h :

```plaintext
nano autosnap-921.sh 
```

-   Nous collons ce qui suit :

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

-   Préciser l'adresse IP de votre nœud Proxmox :

`--host=XXX.XXX.X.XXX`

-   Pour un cluster, préciser l'adresse IP du nœud maître uniquement, ou l'ensemble des adresses IP des nœuds :

`--host='XXX.XXX.X.XXX,XXX.XXX.X.XXX,XXX.XXX.X.XXX'`

**Personnellement, je précise l'ensemble des adresses IP des nœuds ! Si une machine est inaccessible, les instantanés des invités sur les machines accessibles seront effectués.**

-   Préciser l'identifiant :

`--username=root@pam`

**OU**

`--username=XXXX@pve`

-   Préciser le mot de passe :

`--password=XXXXXXXX`

-   Préciser l'invité (--vmid=XXX), ou comme ici, l'ensemble des invités :

`--vmid=all`

-   Pour ignorer un invité :

`--vmid="all,-XXX"`

-   Préciser le nom :

`--label='-921-'`

-   Spécifier le nombre d'instantanés à conserver (rétention) :

`--keep=X`

-   Nous pouvons sauvegarder avec **CTRL + X** suivi de **O** suivi de **ENTER**.
-   Nous rendons le script exécutable :

```plaintext
chmod +x autosnap-921.sh
```

-   Nous pouvons maintenant créer une tâche cron qui appellera notre script à 9 h et 21 h :

```plaintext
crontab -e
```

-   Ont choisi l'éditeur par défaut, qui sera pour ma part /bin/nano (1) :

![](/cv4pve-autosnap/crontab-editor.png)

-   Nous pouvons coller ce qui suit :

```plaintext
0 9,21 * * * /root/autosnap-921.sh
```

# Exemples

-   Pour supprimer l'ensemble des instantanés avec le label -921- sur l'ensemble des invités, utiliser cette commande (**à adapter**) :

`cv4pve-autosnap --host=XXX.XXX.X.XXX --username=root@pam --password=XXXXXXXX --vmid=all clean --label='-921-' --keep=0`

-   Pour obtenir des informations sur les instantanés effectués, utiliser cette commande (**à adapter**) :

`cv4pve-autosnap --host=XXX.XXX.X.XXX --username=root@pam --password=XXXXXXXX --vmid=all status`

Vous pouvez appeler ces deux commandes au travers d'un fichier .sh (script) pour plus de facilité.

# Aller plus loin

-   Vous pouvez créer plusieurs fichiers .sh :

`autosnap-hourly.sh`

`autosnap-daily.sh`

`autosnap-weekly.sh`

`autosnap-monthly.sh`

`autosnap-yearly.sh`

-   Dans ces fichiers, vous adaptez l'option de rétention :

`--keep=3`

`--keep=7`

`--keep=4`

`--keep=2`

`--keep=1`

-   Vous créez une tâche cron pour chaque fichier .sh :

`0 * * * * /root/autosnap-hourly.sh`

`10 0 * * * /root/autosnap-daily.sh`

`20 0 * * 0 /root/autosnap-weekly.sh`

`30 0 1 * * /root/autosnap-monthly.sh`

`40 0 1 1 * /root/autosnap-yearly.sh`

-   En procédant de la sorte, on se calque sur ce que l'on peut obtenir avec l'outil de créations de sauvegardes de Proxmox VE (onglet “Rétention”) :

![](/cv4pve-autosnap/backup-pve.png)

-   Chez moi, toutes les tâches sont créées, mais pas actives ! Il suffit pour ça de décommenter dans crontab avec :

`crontab -e`

![](/cv4pve-autosnap/crontab-final.png)

-   Voici l'ensemble de mes fichiers .sh :

![](/cv4pve-autosnap/full-files.png)

-   Voici en captures le contenu de l'ensemble des fichiers .sh que j'ai créés :

![](/cv4pve-autosnap/autosnap-921.png)

autosnap-921.sh

![](/cv4pve-autosnap/autosnap-cleanall-921.png)

autosnap-cleanall-921.sh

![](/cv4pve-autosnap/autosnap-hourly.png)

autosnap-hourly.sh

![](/cv4pve-autosnap/autosnap-cleanall-hourly.png)

autosnap-cleanall-hourly.sh

![](/cv4pve-autosnap/autosnap-daily.png)

autosnap-daily.sh

![](/cv4pve-autosnap/autosnap-cleanall-daily.png)

autosnap-cleanall-daily.sh

![](/cv4pve-autosnap/autosnap-weekly.png)

autosnap-weekly.sh

![](/cv4pve-autosnap/autosnap-cleanall-weekly.png)

autosnap-cleanall-weekly.sh

![](/cv4pve-autosnap/autosnap-monthly.png)

autosnap-monthly.sh

![](/cv4pve-autosnap/autosnap-cleanall-monthly.png)

autosnap-cleanall-monthly.sh

![](/cv4pve-autosnap/autosnap-yearly.png)

autosnap-yearly.sh

![](/cv4pve-autosnap/autosnap-cleanall-yearly.png)

autosnap-cleanall.yearly.sh

![](/cv4pve-autosnap/autosnap-status-02.png)

autosnap-status.sh

-   Voici un exemple de fichier log :

```plaintext
###NEW JOB STARTS HERE###
ACTION Snap
PVE Version:      8.3.2
VMs:              all
Label:            -921-
Keep:             2
State:            False
Only running:     False
Timeout:          30 sec.
Timestamp format: yyMMddHHmmss
Max % Storage :   95%
                  Storage      Type     Valid   Used %    Disk Size  Disk Usage
        pvenocl/directory       dir        Ok      14.1   457.38 GB    64.35 GB
            pvenocl/local       dir        Ok      20.8    47.58 GB      9.9 GB
              pvenocl/zfs   zfspool        Ok      11.5   224.81 GB    25.92 GB
          pvenocl/zfs-usb   zfspool        Ok      10.9   449.63 GB    49.12 GB
       pvenocl2/directory       dir        Ok         0      219 GB       44 KB
           pvenocl2/local       dir        Ok       8.6    108.2 GB     9.31 GB
             pvenocl2/zfs   zfspool        Ok      19.4   449.63 GB    87.42 GB
         pvenocl2/zfs-usb   zfspool        Ok       1.4   449.63 GB     6.23 GB
           pvenocl3/local       dir        Ok       3.5   210.64 GB     7.31 GB
             pvenocl3/zfs   zfspool        Ok        10   449.63 GB    45.05 GB
         pvenocl3/zfs-usb   zfspool        Ok       2.9   899.25 GB    26.03 GB
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

# Générateurs cron (liens utiles)

-   [https://crontab.guru](https://crontab.guru)
-   [https://crontab.cronhub.io](https://crontab.cronhub.io)
-   [https://www.freeformatter.com/cron-expression-generator-quartz.html](https://www.freeformatter.com/cron-expression-generator-quartz.html)
-   [https://crontab-generator.org](https://crontab-generator.org)
-   [https://www.uptimia.com/cron-expression-generator](https://www.uptimia.com/cron-expression-generator)
-   [https://hyperping.com/cron-expression-generator](https://hyperping.com/cron-expression-generator)
-   [https://quickref.me/cron.html](https://quickref.me/cron.html)
---
title: Mettre à jour automatiquement ses enregistrements DNS OVHcloud (IPv4 et IPv6 dynamiques)
description: Guide pour mettre à jour automatiquement vos DNS OVHcloud (IPv4 et IPv6 dynamiques) sur votre serveur auto-hébergé avec un script Bash.
published: true
date: 2026-10-11T10:58:57.260Z
tags: administration, dns, ipv6, ovh, ipv4, scripts, reseau
editor: markdown
dateCreated: 2026-10-10T22:51:45.882Z
---

Ce guide explique comment garder à jour, avec un script Bash et l'API OVHcloud, les enregistrements DNS d'une infrastructure auto-hébergée dont l'adresse IPv4 publique change parfois et dont le préfixe IPv6 est attribué dynamiquement par le fournisseur d'accès.

Le problème est simple : un outil comme `ddclient` ou DynHost s'occupe très bien de l'IPv4, mais l'IPv6 est souvent oubliée. Quand le fournisseur change le préfixe, l'enregistrement `AAAA` pointe vers une adresse qui n'existe plus. Les clients IPv6 n'arrivent plus, et comme les navigateurs retombent automatiquement sur l'IPv4, on ne s'en aperçoit presque pas, jusqu'au jour où l'on constate qu'il n'y a plus aucun trafic IPv6 dans les journaux.

Le script :

- met à jour l'enregistrement `AAAA` du domaine principal ;
- met à jour les enregistrements `A` et `AAAA` d'un service hébergé sur une autre machine (l'exemple utilise le sous-domaine `mumble`) ;
- peut aussi gérer l'enregistrement `A` du domaine principal, mais laisse ce rôle à `ddclient` par défaut ;
- ne fait rien, et n'écrit rien dans le journal, tant que l'IPv4 et l'IPv6 ne changent pas ;
- ne crée et ne supprime jamais d'enregistrement.

> **Important :** ce script modifie les enregistrements d'une zone DNS existante. Il ne change ni la délégation du domaine, ni les serveurs DNS, ni le routage IPv6, ni le pare-feu de ton réseau.

## 1. Principe de fonctionnement

### 1.1. Les enregistrements gérés

| Enregistrement | Valeur publiée |
| --- | --- |
| `AAAA` du domaine principal (`@`) | L'IPv6 publique du serveur qui exécute le script (typiquement le reverse proxy). |
| `AAAA` du service (`mumble`) | Le préfixe `/64` de ce serveur, suivi de l'identifiant d'interface du service. |
| `A` du service (`mumble`) | L'IPv4 publique. |
| `A` du domaine principal (`@`) | Optionnel. Laissé à `ddclient` ou DynHost par défaut. |

Les autres sous-domaines n'ont rien à faire : si ta zone contient un enregistrement joker `* CNAME votredomaine.tld.`, ils suivent automatiquement le domaine principal.

### 1.2. Comment l'IPv6 du service est construite

Le script ne peut pas lire l'adresse d'une autre machine. Mais deux machines d'un même réseau local partagent le même préfixe `/64` et ne diffèrent que par leur **identifiant d'interface** (les 64 derniers bits), qui reste stable lorsque le préfixe change.

Le script prend donc le préfixe de l'adresse de son propre serveur et y ajoute l'identifiant du service, que tu lui indiques une fois pour toutes dans `SERVICE_IID`.

Cette méthode n'est valable que si le service utilise réellement cette adresse. L'identifiant est souvent dérivé de l'adresse MAC (c'est le cas des conteneurs LXC sous Proxmox), mais pas toujours : relève-le toujours sur la machine elle-même (voir le paragraphe 7.2).

### 1.3. Un fonctionnement silencieux

Le script mémorise la dernière IPv4 et la dernière IPv6 qu'il a publiées. À chaque exécution, il ne fait qu'un appel à un service de détection d'IPv4 et une lecture locale. Il ne contacte l'API OVHcloud que si quelque chose a changé.

En cas d'échec (API indisponible, droits insuffisants, etc.), l'état n'est pas mémorisé : l'exécution suivante réessaie toute seule.

### 1.4. Des garde-fous volontaires

- Il **ne crée pas** d'enregistrement manquant : il signale l'erreur.
- Il **refuse de modifier** un nom qui possède plusieurs enregistrements du même type, car il ne peut pas savoir lequel est le bon.
- Il ignore les adresses IPv6 locales (`fd00::/8`), temporaires ou dépréciées.
- Il s'arrête sur une configuration invalide plutôt que de publier une adresse fausse.

## 2. Prérequis

- Un domaine dont la zone DNS est gérée par OVHcloud (compte OVHcloud Europe : le script utilise `eu.api.ovh.com`).
- Un serveur Linux Debian ou Ubuntu avec Bash, `curl`, `jq` et un accès Internet.
- Sur ce serveur, une interface réseau qui porte une adresse IPv6 publique.
- Si tu gères un service sur une autre machine : cette machine doit être dans le même réseau `/64`.
- Un routage et un pare-feu IPv6 déjà configurés. **Un enregistrement DNS ne rend pas un service joignable : il ne fait qu'indiquer où le chercher.**

## 3. Préparer la zone DNS

Dans l'espace client OVHcloud, ouvre la zone DNS du domaine et crée à l'avance les enregistrements que le script va gérer. Leur valeur initiale n'a pas d'importance, le script la corrigera.

| Type | Sous-domaine | Valeur initiale |
| --- | --- | --- |
| `AAAA` | domaine principal (`@`) | l'IPv6 actuelle du serveur |
| `A` | `mumble` | l'IPv4 publique actuelle |
| `AAAA` | `mumble` | l'IPv6 actuelle du service |

Quelques règles à respecter :

- **Un seul enregistrement par type et par nom** pour ce que le script gère.
- Si `ddclient` ou DynHost gère déjà l'`A` du domaine principal, ne touche pas à cette gestion : le script la laisse tranquille par défaut.
- Les TTL de ces enregistrements sont ramenés à 300 secondes au premier passage. Un TTL court limite la durée de l'indisponibilité IPv6 lors d'un changement de préfixe.

## 4. Créer les identifiants API OVHcloud

Ouvre la page officielle de création d'identifiants : <https://eu.api.ovh.com/createToken/>

Renseigne un nom et une description, choisis la durée de validité qui te convient, et accorde **uniquement** ces droits, en remplaçant `votredomaine.tld` par le nom de ta zone :

```
GET  /domain/zone/votredomaine.tld/record
GET  /domain/zone/votredomaine.tld/record/*
PUT  /domain/zone/votredomaine.tld/record/*
POST /domain/zone/votredomaine.tld/refresh
```

Ces droits suffisent pour lire les enregistrements, modifier ceux qui existent et rafraîchir la zone. Le script n'a besoin ni de `POST` sur les enregistrements ni de `DELETE`, puisqu'il ne crée ni ne supprime rien.

Tu obtiens trois valeurs : l'**Application Key** (AK), l'**Application Secret** (AS) et la **Consumer Key** (CK). Elles sont sensibles : ne les publie jamais (dépôt Git, forum, capture d'écran, message de dépannage).

## 5. Installer les dépendances

```bash
apt update
apt install -y curl jq iproute2 util-linux ca-certificates
```

## 6. Créer le fichier d'identifiants

```bash
nano /root/.ovh_dns.env
```

```bash
OVH_AK="ton_application_key"
OVH_AS="ton_application_secret"
OVH_CK="ton_consumer_key"
```

Puis restreins l'accès au seul utilisateur `root` :

```bash
chown root:root /root/.ovh_dns.env
chmod 600 /root/.ovh_dns.env
```

## 7. Créer le script

```bash
nano /usr/local/bin/update_ovh_dns.sh
```

```bash
#!/bin/bash
# update_ovh_dns.sh
# Synchronise chez OVHcloud : l'AAAA du domaine principal et, en option, les
# enregistrements A et AAAA d'un service hébergé sur une autre machine.
# Sans effet (et sans bruit) tant que l'IPv4 publique et l'IPv6 ne changent pas.
#
# Usage :  update_ovh_dns.sh              (cron)
#          DRY_RUN=1 update_ovh_dns.sh    (simulation : aucune modification)
set -euo pipefail
umask 077

# -------------------------------------------------------------------
# CONFIGURATION — À ADAPTER
# -------------------------------------------------------------------
ZONE="votredomaine.tld"                 # nom de la zone DNS OVH
OVH_ENDPOINT="https://eu.api.ovh.com/1.0"
OVH_ENV_FILE="/root/.ovh_dns.env"       # contient OVH_AK, OVH_AS, OVH_CK (chmod 600)
IPV6_IFACE="eth0"                       # interface portant l'IPv6 publique de ce serveur
TTL=300                                 # TTL appliqué aux enregistrements gérés

# 1 = gérer aussi l'enregistrement A du domaine principal (0 si ddclient/DynHost s'en occupe)
MANAGE_APEX_A="${MANAGE_APEX_A:-0}"

# Service hébergé sur une autre machine (ex. un serveur vocal) ayant ses propres A et AAAA.
# 0 = ne gérer que l'AAAA du domaine principal.
MANAGE_SERVICE=1
SERVICE_SUB="mumble"                    # sous-domaine du service
SERVICE_IID=""                          # 4 derniers groupes de SON adresse IPv6, ex. 0000:0000:0000:0042

DRY_RUN="${DRY_RUN:-0}"
STATE_FILE="/var/run/ovh_dns_pushed.txt"
LOG_FILE="/var/log/ovh_dns_update.log"
LOCKFILE="/run/lock/ovh-dns-update.lock"

CHANGED=0
GROUPS4_RE='^[0-9a-f]{1,4}(:[0-9a-f]{1,4}){3}$'

# -------------------------------------------------------------------
# JOURNAL ET ERREURS
# -------------------------------------------------------------------
log() {
    local msg
    msg="$(date '+%Y-%m-%d %H:%M:%S') - $*"
    echo "$msg" >> "$LOG_FILE"
    if [ -t 1 ]; then echo "$msg"; fi
    return 0
}

# Les erreurs vont aussi sur stderr hors terminal : cron peut alors prévenir par e-mail.
err() {
    log "ERREUR : $*"
    if [ ! -t 1 ]; then echo "ERREUR : $*" >&2; fi
    return 0
}

die() { err "$*"; exit 1; }

# -------------------------------------------------------------------
# API OVHCLOUD — ovh_api METHODE /chemin [corps-json]
# -------------------------------------------------------------------
ovh_api() {
    local method="$1" path="$2" body="${3:-}" url ts sig
    url="${OVH_ENDPOINT}${path}"
    ts=$(curl -s --max-time 10 "${OVH_ENDPOINT}/auth/time") || return 1
    sig='$1$'$(printf '%s+%s+%s+%s+%s+%s' "$OVH_AS" "$OVH_CK" "$method" "$url" "$body" "$ts" | sha1sum | cut -d' ' -f1)
    curl -s --max-time 15 --fail-with-body -X "$method" "$url" \
        -H "Content-Type: application/json" \
        -H "X-Ovh-Application: ${OVH_AK}" \
        -H "X-Ovh-Consumer: ${OVH_CK}" \
        -H "X-Ovh-Timestamp: ${ts}" \
        -H "X-Ovh-Signature: ${sig}" \
        ${body:+--data "$body"}
}

# -------------------------------------------------------------------
# DÉTECTION DES ADRESSES
# -------------------------------------------------------------------
detect_ipv4() {
    local ip
    ip=$(curl -4 -s --max-time 5 https://api.ipify.org || true)
    [[ "$ip" =~ ^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+$ ]] || return 1
    echo "$ip"
}

# Première IPv6 publique (2000::/3) : ni ULA (fd00::/8), ni temporaire, ni dépréciée.
detect_ipv6() {
    local addr
    addr=$(ip -6 addr show dev "$IPV6_IFACE" scope global 2>/dev/null \
        | awk '/inet6/ && $2 ~ /^[23]/ && !/temporary/ && !/deprecated/ && !/tentative/ && !/dadfailed/ {print $2; exit}')
    addr="${addr%/*}"
    [ -n "$addr" ] || return 1
    echo "$addr" | tr 'A-F' 'a-f'
}

# -------------------------------------------------------------------
# ensure_record TYPE SOUS-DOMAINE (vide = domaine principal) CIBLE
# -------------------------------------------------------------------
ensure_record() {
    local type="$1" sub="$2" target="$3" label ids id rec cur curttl out
    local -a match_ids=() match_cur=() match_ttl=()
    label="${type} ${sub:-@}"

    ids=$(ovh_api GET "/domain/zone/${ZONE}/record?fieldType=${type}") \
        || { err "API (liste ${type}) : ${ids:-pas de réponse}"; return 1; }
    echo "$ids" | jq -e 'type == "array"' >/dev/null 2>&1 \
        || { err "API (liste ${type}) : ${ids}"; return 1; }

    # Première passe : repérer les enregistrements correspondants, sans rien modifier.
    for id in $(echo "$ids" | jq -r '.[]'); do
        rec=$(ovh_api GET "/domain/zone/${ZONE}/record/${id}") \
            || { err "API (lecture ${id}) : ${rec:-pas de réponse}"; return 1; }
        [ "$(echo "$rec" | jq -r '.subDomain')" = "$sub" ] || continue
        match_ids+=("$id")
        match_cur+=("$(echo "$rec" | jq -r '.target' | tr 'A-F' 'a-f')")
        match_ttl+=("$(echo "$rec" | jq -r '.ttl')")
    done

    case "${#match_ids[@]}" in
        0) err "enregistrement ${label} introuvable dans ${ZONE} (création automatique désactivée)"; return 1 ;;
        1) ;;
        *) err "plusieurs enregistrements ${label} : intervention manuelle requise, aucune modification effectuée"; return 1 ;;
    esac

    id="${match_ids[0]}"; cur="${match_cur[0]}"; curttl="${match_ttl[0]}"

    if [ "$cur" = "$target" ] && [ "$curttl" = "$TTL" ]; then
        log "OK : ${label} déjà à jour (${target})"
    elif [ "$DRY_RUN" = "1" ]; then
        log "SIMULATION : ${label} ${cur} (TTL ${curttl}) -> ${target} (TTL ${TTL})"
        CHANGED=1
    else
        out=$(ovh_api PUT "/domain/zone/${ZONE}/record/${id}" "{\"target\":\"${target}\",\"ttl\":${TTL}}") \
            || { err "API (mise à jour ${label}) : ${out:-pas de réponse}"; return 1; }
        log "MAJ : ${label} ${cur} -> ${target} (TTL ${TTL})"
        CHANGED=1
    fi
    return 0
}

# -------------------------------------------------------------------
# PROGRAMME PRINCIPAL
# -------------------------------------------------------------------
main() {
    local NEW_IPV4 NEW_IPV6 PREFIX SERVICE_V6="" STATE out rc=0

    [ "$ZONE" != "votredomaine.tld" ] || die "remplace ZONE par le nom réel de ta zone DNS"
    [[ "$TTL" =~ ^[0-9]+$ ]] || die "TTL invalide"
    command -v jq >/dev/null || die "jq est requis (apt install jq)"
    [ -r "$OVH_ENV_FILE" ] || die "fichier d'identifiants introuvable : ${OVH_ENV_FILE}"
    # shellcheck disable=SC1090
    . "$OVH_ENV_FILE"
    [ -n "${OVH_AK:-}" ] && [ -n "${OVH_AS:-}" ] && [ -n "${OVH_CK:-}" ] \
        || die "identifiants API incomplets dans ${OVH_ENV_FILE}"

    exec 200>"$LOCKFILE"
    flock -n 200 || exit 0

    NEW_IPV4=$(detect_ipv4) || die "IPv4 publique introuvable"
    NEW_IPV6=$(detect_ipv6) || die "aucune IPv6 publique stable sur ${IPV6_IFACE}"
    PREFIX=$(echo "$NEW_IPV6" | awk -F: '{print $1":"$2":"$3":"$4}')
    [[ "$PREFIX" =~ $GROUPS4_RE ]] \
        || die "préfixe /64 non lisible dans ${NEW_IPV6} (adresse compressée dans les 4 premiers groupes)"

    if [ "$MANAGE_SERVICE" = "1" ]; then
        SERVICE_IID="${SERVICE_IID,,}"
        [[ "$SERVICE_IID" =~ $GROUPS4_RE ]] \
            || die "SERVICE_IID doit contenir 4 groupes hexadécimaux séparés par ':' (ou passe MANAGE_SERVICE à 0)"
        SERVICE_V6="${PREFIX}:${SERVICE_IID}"
    fi

    STATE="${NEW_IPV4}|${NEW_IPV6}|${SERVICE_V6}"
    if [ "$DRY_RUN" != "1" ] && [ -f "$STATE_FILE" ] && [ "$(cat "$STATE_FILE")" = "$STATE" ]; then
        exit 0
    fi
    log "Synchronisation DNS OVH — IPv4 : ${NEW_IPV4} | IPv6 de ce serveur : ${NEW_IPV6} | IPv6 du service : ${SERVICE_V6:-non géré}"

    ensure_record AAAA "" "$NEW_IPV6" || rc=1
    if [ "$MANAGE_SERVICE" = "1" ]; then
        ensure_record AAAA "$SERVICE_SUB" "$SERVICE_V6" || rc=1
        ensure_record A "$SERVICE_SUB" "$NEW_IPV4" || rc=1
    fi
    if [ "$MANAGE_APEX_A" = "1" ]; then
        ensure_record A "" "$NEW_IPV4" || rc=1
    fi

    if [ "$CHANGED" -eq 1 ] && [ "$DRY_RUN" != "1" ]; then
        if out=$(ovh_api POST "/domain/zone/${ZONE}/refresh"); then
            log "Zone ${ZONE} rafraîchie chez OVH"
        else
            err "API (rafraîchissement de la zone) : ${out:-pas de réponse}"
            rc=1
        fi
    fi

    # L'état n'est mémorisé qu'en cas de succès complet : sinon, le cron réessaie.
    if [ "$rc" -eq 0 ] && [ "$DRY_RUN" != "1" ]; then
        echo "$STATE" > "$STATE_FILE"
    fi
    exit "$rc"
}

# Lance main() seulement si le script est exécuté directement (permet de le "sourcer" pour des tests).
if [ "${BASH_SOURCE[0]}" = "$0" ]; then
    main "$@"
fi
```

Puis rends-le exécutable par `root` uniquement :

```bash
chown root:root /usr/local/bin/update_ovh_dns.sh
chmod 700 /usr/local/bin/update_ovh_dns.sh
```

### 7.1. Les paramètres à adapter

| Paramètre | Rôle |
| --- | --- |
| `ZONE` | Nom de la zone DNS OVH. Le script refuse de démarrer tant que la valeur par défaut est présente. |
| `IPV6_IFACE` | Interface portant l'IPv6 publique du serveur (`ip -br addr` pour la trouver). |
| `TTL` | Durée de vie, en secondes, appliquée aux enregistrements gérés. |
| `MANAGE_APEX_A` | `0` : ne pas toucher à l'`A` du domaine principal. `1` : le gérer aussi. |
| `MANAGE_SERVICE` | `1` : gérer les `A` et `AAAA` d'un service distinct. `0` : ne gérer que l'`AAAA` du domaine principal. |
| `SERVICE_SUB` | Sous-domaine du service (`mumble` dans l'exemple). |
| `SERVICE_IID` | Les quatre derniers groupes de l'adresse IPv6 du service. Obligatoire si `MANAGE_SERVICE=1`. |

### 7.2. Trouver `SERVICE_IID`

Sur la machine qui héberge le service, affiche ses adresses IPv6 :

```bash
ip -6 addr show scope global
```

Repère l'adresse **stable** : celle qui n'est pas marquée `temporary` et qui a le même préfixe que ton serveur. Exemple avec un préfixe de documentation :

```
inet6 2001:db8:1234:5678:be24:11ff:fe8b:9a6f/64 scope global dynamic mngtmpaddr
```

L'identifiant est constitué des quatre derniers groupes : `be24:11ff:fe8b:9a6f`.

Vérifie ensuite que ton choix est cohérent avec ce qui est déjà publié : l'enregistrement `AAAA` actuel du service doit être égal au préfixe suivi de cet identifiant.

```bash
dig +short AAAA mumble.votredomaine.tld
```

Si tu recrées la machine ou son interface réseau (nouvelle adresse MAC, par exemple), l'identifiant change : mets alors `SERVICE_IID` à jour.

## 8. Tester

### 8.1. Simulation

Avant toute modification réelle, lance le script en mode simulation :

```bash
DRY_RUN=1 /usr/local/bin/update_ovh_dns.sh
```

Il lit les enregistrements et affiche ce qu'il ferait, sans rien envoyer à OVH. Si tout est déjà à jour, tu vois uniquement des lignes `OK`. Sinon, tu vois des lignes `SIMULATION` avec l'ancienne et la nouvelle valeur.

La simulation ne vérifie pas que les droits de modification sont valides : seule une exécution réelle le prouve.

### 8.2. Exécution réelle

```bash
/usr/local/bin/update_ovh_dns.sh
tail -n 20 /var/log/ovh_dns_update.log
```

### 8.3. Vérifier ce qui est réellement publié

Interroge d'abord un résolveur public, puis directement les serveurs d'OVH pour contourner les caches :

```bash
dig +short AAAA votredomaine.tld @1.1.1.1
dig +short AAAA mumble.votredomaine.tld @1.1.1.1
dig +short A mumble.votredomaine.tld @1.1.1.1

dig +short AAAA votredomaine.tld @$(dig +short NS votredomaine.tld | head -1)
```

Si OVH renvoie la nouvelle valeur mais pas le résolveur public, il suffit d'attendre l'expiration du TTL.

### 8.4. Vérifier que l'IPv6 entre vraiment

Un DNS correct ne prouve pas que le service est joignable. Depuis une machine extérieure ayant de l'IPv6 (un téléphone en 4G ou 5G, wifi coupé, convient) :

```bash
curl -6 -sI https://votredomaine.tld
```

Si cette commande échoue alors que le DNS est correct, regarde le pare-feu IPv6 de ton routeur.

## 9. Automatiser avec cron

Crée la tâche :

```bash
echo '*/5 * * * * root /usr/local/bin/update_ovh_dns.sh' > /etc/cron.d/update_ovh_dns
chmod 644 /etc/cron.d/update_ovh_dns
```

Le script est silencieux quand rien ne change. En cas d'erreur, il écrit aussi le message sur la sortie d'erreur : si un serveur de messagerie est installé sur la machine, cron peut donc t'avertir par e-mail.

Le journal reste petit (seules les synchronisations y sont inscrites), mais une rotation évite qu'il grossisse indéfiniment. Crée `/etc/logrotate.d/ovh_dns_update` :

```
/var/log/ovh_dns_update.log {
    monthly
    rotate 6
    compress
    missingok
    notifempty
}
```

## 10. Adapter le script à son infrastructure

### 10.1. `ddclient` gère déjà l'IPv4 du domaine principal

Garde `MANAGE_APEX_A=0`. Vérifie en plus que `ddclient` ne met pas aussi à jour un `AAAA`, sinon les deux outils se disputeraient l'enregistrement :

```bash
grep -vE '^\s*#|^\s*$' /etc/ddclient.conf
```

Tu ne dois y trouver ni `usev6=` ni un second nom d'hôte pointant vers l'`AAAA`.

### 10.2. Remplacer `ddclient`

Passe `MANAGE_APEX_A=1` et désactive `ddclient`. Assure-toi qu'un seul outil modifie l'enregistrement : deux mécanismes concurrents peuvent s'écraser mutuellement. Si l'enregistrement est de type DynHost chez OVH, vérifie qu'il peut aussi être modifié par l'API de zone.

### 10.3. Pas de service distinct

Passe `MANAGE_SERVICE=0`. Le script ne gère alors que l'`AAAA` du domaine principal, et `SERVICE_IID` peut rester vide.

### 10.4. Plusieurs services

Duplique dans `main()` les lignes `ensure_record` du service, avec un autre sous-domaine et son propre identifiant :

```bash
ensure_record AAAA "autre" "${PREFIX}:xxxx:xxxx:xxxx:xxxx" || rc=1
ensure_record A "autre" "$NEW_IPV4" || rc=1
```

Après toute modification des paramètres du script, supprime le fichier d'état pour forcer une synchronisation :

```bash
rm -f /var/run/ovh_dns_pushed.txt
```

## 11. Dépannage

| Message dans le journal | Cause probable |
| --- | --- |
| `remplace ZONE par le nom réel de ta zone DNS` | `ZONE` contient encore la valeur par défaut. |
| `identifiants API incomplets` | Une des trois valeurs manque dans `/root/.ovh_dns.env`. |
| `API (liste AAAA) : ...` suivi d'un message OVH | Droits insuffisants sur le jeton, mauvais nom de zone, ou identifiants invalides. Le message d'OVH est recopié dans le journal. |
| `enregistrement ... introuvable` | L'enregistrement n'existe pas dans la zone. Crée-le à la main : le script n'en crée jamais. |
| `plusieurs enregistrements ...` | Le même nom a plusieurs enregistrements du même type. Supprime les doublons inutiles. |
| `aucune IPv6 publique stable sur ...` | Mauvaise interface, pas d'IPv6 publique, ou adresse uniquement temporaire. Vérifie avec `ip -6 addr show scope global`. |
| `préfixe /64 non lisible ...` | L'adresse du serveur est écrite avec `::` dans ses quatre premiers groupes (cas rare, par exemple une délégation en `/48`). Le script refuse plutôt que de deviner. |
| `SERVICE_IID doit contenir 4 groupes ...` | `SERVICE_IID` est vide ou mal formé. Corrige-le ou passe `MANAGE_SERVICE=0`. |

Si le script fonctionne à la main mais pas depuis cron, vérifie les chemins absolus, les droits du fichier d'identifiants et le contenu de `/var/log/ovh_dns_update.log`.

## 12. Limites

- **Le pare-feu du routeur.** Si tes règles IPv6 visent des adresses explicites, elles changent elles aussi à chaque rotation de préfixe. Le script ne peut pas les mettre à jour : utilise, si ton équipement le permet, des règles qui ne dépendent pas du préfixe.
- **Une courte coupure IPv6 à chaque rotation**, d'une durée maximale égale au TTL (300 secondes), le temps que les caches expirent.
- **L'identifiant du service est une valeur à maintenir.** Il change si la machine ou son interface est recréée.
- **La détection de l'IPv4 dépend d'un service externe** (`api.ipify.org`). Si celui-ci est indisponible, le script s'arrête sans rien modifier.
- **La construction du préfixe suppose un `/64` lisible** dans les quatre premiers groupes de l'adresse du serveur.

## Conclusion

Ce script légèrement outillé suffit pour garder à jour un domaine OVHcloud malgré un préfixe IPv6 qui change, sans plateforme d'automatisation supplémentaire. Il demande une préparation rigoureuse (enregistrements créés à l'avance, jeton limité à la zone, identifiant du service relevé sur la machine), mais une fois en place, il se fait oublier.

---

**Auteur :** Amaury, alias BlablaLinux.

Retrouvez l'ensemble de mes services publics sur [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).
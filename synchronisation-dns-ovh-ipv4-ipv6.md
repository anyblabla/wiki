---
title: Synchronisation automatique des enregistrements DNS OVH en IPv4 et IPv6
description: Guide de configuration d'un script Bash pour synchroniser automatiquement les enregistrements DNS OVH en IPv4 et IPv6, même en cas de changement d'adresse IP ou de préfixe IPv6.
published: false
date: 2026-10-10T22:51:45.882Z
tags: administration, dns, ipv6, ovh, ipv4, scripts, reseau
editor: markdown
dateCreated: 2026-10-10T22:51:45.882Z
---

Ce guide explique comment automatiser la mise à jour des enregistrements DNS d'un domaine hébergé chez **OVHcloud**, à l'aide d'un script Bash exécuté sur un serveur Linux.

Cette solution est particulièrement utile pour les infrastructures auto-hébergées dont l'adresse IPv4 publique ou le préfixe IPv6 peut changer, notamment lorsque le fournisseur d'accès à Internet attribue un préfixe IPv6 dynamique.

Le script permet de :

- Mettre à jour l'enregistrement AAAA du domaine principal.
- Mettre à jour les enregistrements A et AAAA de sous-domaines spécifiques.
- Gérer éventuellement l'enregistrement A du domaine principal, en remplacement d'un outil comme `ddclient`.
- Détecter les modifications nécessaires avant de modifier les enregistrements DNS.
- Éviter les modifications inutiles lorsque les enregistrements sont déjà corrects.
- Journaliser les opérations et signaler les erreurs.
- Automatiser les vérifications avec Cron.

> **Important :** ce script met à jour les enregistrements d'une zone DNS existante. Il ne modifie ni la délégation du domaine, ni les serveurs DNS faisant autorité, ni le routage IPv6 du réseau.

## 1. Principe de fonctionnement

### 1.1. Gestion des adresses IP

Le script récupère l'IPv4 publique à l'aide d'un service externe de détection d'adresse IP. Il détecte également une adresse IPv6 globale sur une interface réseau définie dans sa configuration.

L'adresse IPv6 sélectionnée doit correspondre à une adresse stable, non temporaire et non dépréciée, dont le préfixe est effectivement celui que l'on souhaite publier dans le DNS.

### 1.2. Gestion des enregistrements DNS

Le fonctionnement dépend des besoins de chaque infrastructure.

| Enregistrement           | Fonction                                                                               |
| ------------------------ | -------------------------------------------------------------------------------------- |
| `A` de l'apex (`@`)      | Adresse IPv4 du domaine principal, gérée facultativement par le script.                |
| `AAAA` de l'apex (`@`)   | Adresse IPv6 du serveur principal.                                                     |
| `A` d'un sous-domaine    | Adresse IPv4 publique du serveur.                                                      |
| `AAAA` d'un sous-domaine | Adresse IPv6 construite à partir du préfixe et de l'identifiant d'interface configuré. |

Dans cet exemple, le sous-domaine `mumble` représente un service hébergé dans un conteneur ou sur une machine distincte. Il peut être remplacé par un autre nom.

**Attention :** un enregistrement DNS ne configure pas le réseau. Pour qu'un service soit joignable en IPv6, le routage, le pare-feu et la configuration du conteneur doivent également être adaptés.

### 1.3. Fonctionnement idempotent

À chaque exécution, le script compare les valeurs attendues aux enregistrements présents dans la zone OVH.

Si les valeurs sont identiques, aucune modification n'est envoyée. Si une valeur diffère, le script met à jour l'enregistrement correspondant.

Cette méthode permet de détecter également une modification manuelle de la zone ou un enregistrement qui ne correspondrait plus à l'adresse IP attendue.

Le script ne crée ni ne supprime automatiquement des enregistrements DNS. Les entrées nécessaires doivent être préparées à l'avance.

## 2. Prérequis

Avant de commencer, il faut disposer :

- D'un domaine dont la zone DNS est gérée par OVHcloud.
- D'un serveur Linux disposant de Bash et d'un accès réseau à l'API OVHcloud.
- Des droits administrateur sur ce serveur.
- D'une interface réseau portant une adresse IPv6 globale adaptée à l'usage prévu, si la gestion IPv6 est activée.
- Des enregistrements DNS préalablement créés dans la zone OVH.
- Des identifiants API OVHcloud disposant uniquement des autorisations nécessaires.

Le script est prévu pour une distribution Debian ou Ubuntu. Les commandes d'installation peuvent être adaptées à d'autres distributions.

## 3. Préparer les enregistrements DNS

Dans l'espace client OVHcloud, ouvre la zone DNS du domaine concerné.

Crée les enregistrements qui seront gérés par le script. Pour l'exemple utilisé dans ce guide :

| Type   | Sous-domaine            | Cible initiale                   |
| ------ | ----------------------- | -------------------------------- |
| `AAAA` | Domaine principal (`@`) | IPv6 globale actuelle du serveur |
| `A`    | `mumble`                | IPv4 publique actuelle           |
| `AAAA` | `mumble`                | IPv6 actuelle du service         |

Si le script doit également gérer l'IPv4 du domaine principal, il faudra aussi disposer de l'enregistrement `A` de l'apex.

Les cibles initiales servent uniquement à créer les entrées DNS. Elles seront actualisées lors de l'exécution du script si nécessaire.

**Ne crée pas plusieurs enregistrements du même type pour un nom que ce script doit gérer.** Le script s'arrête lorsqu'il détecte plusieurs entrées correspondantes, afin d'éviter de modifier involontairement une configuration DNS particulière.

Si l'IPv4 de l'apex est déjà gérée par `ddclient` ou un autre mécanisme, conserve cette gestion indépendante et laisse l'option correspondante désactivée dans le script.

## 4. Créer les identifiants API OVHcloud

### 4.1. Générer les clés

Rends-toi sur la page officielle de création des identifiants API :

[https://eu.api.ovh.com/createToken/](https://eu.api.ovh.com/createToken/)

Crée des identifiants associés à ton compte OVHcloud et limite leurs droits à la zone DNS concernée.

Pour ce script, les autorisations nécessaires sont les suivantes, en remplaçant `votredomaine.tld` par le nom de la zone :

```
GET  /domain/zone/votredomaine.tld/record
GET  /domain/zone/votredomaine.tld/record/*
PUT  /domain/zone/votredomaine.tld/record/*
POST /domain/zone/votredomaine.tld/refresh
```

Ces droits permettent de lire les enregistrements, de modifier ceux qui existent et de demander le rafraîchissement de la zone.

Le script ne crée ni ne supprime d'enregistrements. Il n'a donc pas besoin des autorisations `POST /record` ou `DELETE /record/*`.

### 4.2. Conserver les identifiants

La création des identifiants fournit trois valeurs :

- `Application Key` (AK)
- `Application Secret` (AS)
- `Consumer Key` (CK)

Ces valeurs sont sensibles. Ne les publie jamais dans un dépôt Git, un forum ou un fichier de configuration accessible aux utilisateurs non autorisés.

## 5. Installer les dépendances

Sur Debian ou Ubuntu, exécute :

```
apt update
apt install -y bash curl jq python3 iproute2 coreutils util-linux ca-certificates
```

Les paquets installés fournissent les outils nécessaires à la détection des adresses, au traitement des réponses JSON, à la manipulation des adresses IPv6 et à la gestion du verrou d'exécution.

## 6. Créer le fichier de configuration API

Crée le fichier qui contiendra les identifiants :

```
nano /etc/ovh_dns.env
```

Ajoute les trois clés obtenues auprès d'OVHcloud :

```
OVH_AK="ton_application_key"
OVH_AS="ton_application_secret"
OVH_CK="ton_consumer_key"
```

Enregistre le fichier, puis restreins son accès :

```
chown root:root /etc/ovh_dns.env
chmod 600 /etc/ovh_dns.env
```

Seul l'utilisateur `root` pourra ainsi lire ou modifier ce fichier.

## 7. Créer le script de synchronisation

Crée le fichier suivant :

```
nano /usr/local/sbin/update_ovh_dns.sh
```

Colle le script ci-dessous, puis adapte les paramètres de configuration avant de l'exécuter.

```
#!/bin/bash
# update_ovh_dns.sh
# Synchronisation des enregistrements DNS OVH en IPv4 et IPv6.

set -Eeuo pipefail
umask 077

# -------------------------------------------------------------------
# CONFIGURATION
# -------------------------------------------------------------------

ZONE="votredomaine.tld"
OVH_ENDPOINT="https://eu.api.ovh.com/1.0"
OVH_ENV_FILE="/etc/ovh_dns.env"

# Interface réseau portant l'IPv6 globale de référence.
IPV6_IFACE="eth0"

# Identifiant de 64 bits du service isolé.
# Exemple : les quatre derniers groupes de l'adresse IPv6 du service.
SERVICE_IID="be24:11ff:fe8b:9a6f"

# TTL appliqué aux enregistrements gérés.
TTL=300

# 0 : ne pas gérer l'enregistrement A de l'apex.
# 1 : gérer également l'enregistrement A de l'apex.
MANAGE_APEX_A=0

# 0 : exécution normale.
# 1 : simulation sans modification de la zone.
DRY_RUN="${DRY_RUN:-0}"

LOG_FILE="/var/log/ovh_dns_update.log"
LOCKFILE="/run/lock/ovh-dns-update.lock"

CHANGED=0

# -------------------------------------------------------------------
# JOURNALISATION
# -------------------------------------------------------------------

log() {
    local msg
    msg="$(date '+%Y-%m-%d %H:%M:%S') - $*"
    printf '%s\n' "$msg" >> "$LOG_FILE"
    printf '%s\n' "$msg"
}

die() {
    log "ERREUR : $*"
    exit 1
}

# -------------------------------------------------------------------
# API OVHCLOUD
# -------------------------------------------------------------------

ovh_api() {
    local method="$1"
    local path="$2"
    local body="${3:-}"
    local url timestamp signature

    url="${OVH_ENDPOINT}${path}"

    timestamp="$(
        curl -fsS --max-time 10 \
            "${OVH_ENDPOINT}/auth/time"
    )" || return 1

    signature='$1$'$(printf '%s+%s+%s+%s+%s+%s' \
        "$OVH_AS" "$OVH_CK" "$method" "$url" "$body" "$timestamp" |
        sha1sum | cut -d' ' -f1)

    local -a args
    args=(
        -fsS
        --max-time 15
        -X "$method"
        "$url"
        -H "Content-Type: application/json"
        -H "X-Ovh-Application: ${OVH_AK}"
        -H "X-Ovh-Consumer: ${OVH_CK}"
        -H "X-Ovh-Timestamp: ${timestamp}"
        -H "X-Ovh-Signature: ${signature}"
    )

    if [[ -n "$body" ]]; then
        args+=(--data "$body")
    fi

    curl "${args[@]}"
}

# -------------------------------------------------------------------
# DÉTECTION DES ADRESSES IP
# -------------------------------------------------------------------

detect_ipv4() {
    local ip

    ip="$(curl -4 -fsS --max-time 10 https://api.ipify.org)" ||
        return 1

    python3 -c '
import ipaddress, sys
try:
    ip = ipaddress.ip_address(sys.argv[1])
    if ip.version != 4 or not ip.is_global:
        raise ValueError()
    print(ip)
except ValueError:
    sys.exit(1)
' "$ip"
}

detect_ipv6() {
    local addr

    addr="$(
        ip -6 addr show dev "$IPV6_IFACE" scope global |
        awk '
            /inet6/ &&
            !/temporary/ &&
            !/deprecated/ &&
            !/tentative/ &&
            !/dadfailed/ {
                print $2
                exit
            }
        '
    )" || return 1

    addr="${addr%%/*}"
    [[ -n "$addr" ]] || return 1

    python3 -c '
import ipaddress, sys
try:
    ip = ipaddress.IPv6Address(sys.argv[1])
    if not ip.is_global:
        raise ValueError()
    print(ip)
except ValueError:
    sys.exit(1)
' "$addr"
}

# Construit une IPv6 de service en combinant le préfixe /64
# de l'adresse de référence avec un identifiant d'interface.
build_service_ipv6() {
    python3 -c '
import ipaddress, sys

try:
    host = ipaddress.IPv6Address(sys.argv[1])
    iid = sys.argv[2].split(":")

    if len(iid) != 4:
        raise ValueError("SERVICE_IID doit contenir quatre groupes")

    if any(not x or len(x) > 4 for x in iid):
        raise ValueError("Groupe IPv6 invalide")

    if any(any(c not in "0123456789abcdefABCDEF" for c in x)
           for x in iid):
        raise ValueError("Caractère IPv6 invalide")

    suffix = int("".join(iid), 16)
    prefix = int(host) & (((1 << 64) - 1) << 64)

    print(ipaddress.IPv6Address(prefix | suffix))
except ValueError as e:
    print(str(e), file=sys.stderr)
    sys.exit(1)
' "$1" "$SERVICE_IID"
}

# -------------------------------------------------------------------
# NORMALISATION DES CIBLES
# -------------------------------------------------------------------

normalize_target() {
    local type="$1"
    local target="$2"

    case "$type" in
        A)
            python3 -c '
import ipaddress, sys
try:
    ip = ipaddress.ip_address(sys.argv[1])
    if ip.version != 4 or not ip.is_global:
        raise ValueError()
    print(ip)
except ValueError:
    sys.exit(1)
' "$target"
            ;;
        AAAA)
            python3 -c '
import ipaddress, sys
try:
    ip = ipaddress.ip_address(sys.argv[1])
    if ip.version != 6 or not ip.is_global:
        raise ValueError()
    print(ip)
except ValueError:
    sys.exit(1)
' "$target"
            ;;
        *)
            return 1
            ;;
    esac
}

# -------------------------------------------------------------------
# VÉRIFICATION ET MISE À JOUR DES ENREGISTREMENTS
# -------------------------------------------------------------------

ensure_record() {
    local type="$1"
    local sub="$2"
    local target="$3"
    local ids id rec cur curttl found=0
    local label normalized_cur normalized_target
    local body

    label="${type} ${sub:-@}"

    normalized_target="$(normalize_target "$type" "$target")" ||
        { log "Cible ${label} invalide : ${target}"; return 1; }

    ids="$(ovh_api GET \
        "/domain/zone/${ZONE}/record?fieldType=${type}")" ||
        { log "Lecture de la liste ${type} impossible"; return 1; }

    jq -e 'type == "array"' <<< "$ids" >/dev/null ||
        { log "Réponse API inattendue pour ${label}"; return 1; }

    while IFS= read -r id; do
        [[ -n "$id" ]] || continue

        rec="$(ovh_api GET \
            "/domain/zone/${ZONE}/record/${id}")" ||
            { log "Lecture de l'enregistrement ${id} impossible"; return 1; }

        [[ "$(jq -r '.subDomain // ""' <<< "$rec")" == "$sub" ]] ||
            continue

        found=$((found + 1))

        if (( found > 1 )); then
            log "Plusieurs enregistrements ${label} : intervention requise"
            return 1
        fi

        cur="$(jq -r '.target // ""' <<< "$rec")"
        curttl="$(jq -r '.ttl // 0' <<< "$rec")"

        normalized_cur="$(normalize_target "$type" "$cur" 2>/dev/null || true)"

        if [[ "$normalized_cur" == "$normalized_target" &&
              "$curttl" == "$TTL" ]]; then
            log "OK : ${label} déjà à jour (${normalized_target})"
            continue
        fi

        if [[ "$DRY_RUN" == "1" ]]; then
            log "SIMULATION : ${label} ${cur} (TTL ${curttl}) -> ${normalized_target} (TTL ${TTL})"
            CHANGED=1
            continue
        fi

        body="$(jq -cn \
            --arg target "$normalized_target" \
            --argjson ttl "$TTL" \
            '{target:$target,ttl:$ttl}')"

        ovh_api PUT \
            "/domain/zone/${ZONE}/record/${id}" "$body" >/dev/null ||
            { log "Mise à jour de ${label} impossible"; return 1; }

        log "MAJ : ${label} ${cur} -> ${normalized_target}"
        CHANGED=1

    done < <(jq -r '.[]' <<< "$ids")

    if (( found == 0 )); then
        log "Enregistrement ${label} introuvable : création automatique désactivée"
        return 1
    fi

    return 0
}

# -------------------------------------------------------------------
# EXÉCUTION PRINCIPALE
# -------------------------------------------------------------------

main() {
    local NEW_IPV4 NEW_IPV6 SERVICE_V6 rc=0

    [[ "$ZONE" != "votredomaine.tld" ]] ||
        die "Remplace ZONE par le nom réel de ta zone DNS"

    [[ "$TTL" =~ ^[0-9]+$ ]] ||
        die "TTL invalide"

    [[ "$MANAGE_APEX_A" == "0" || "$MANAGE_APEX_A" == "1" ]] ||
        die "MANAGE_APEX_A doit valoir 0 ou 1"

    [[ "$DRY_RUN" == "0" || "$DRY_RUN" == "1" ]] ||
        die "DRY_RUN doit valoir 0 ou 1"

    command -v jq >/dev/null || die "jq est requis"
    command -v curl >/dev/null || die "curl est requis"
    command -v python3 >/dev/null || die "python3 est requis"
    command -v flock >/dev/null || die "flock est requis"

    [[ -r "$OVH_ENV_FILE" ]] ||
        die "Fichier API inaccessible : ${OVH_ENV_FILE}"

    # shellcheck disable=SC1090
    . "$OVH_ENV_FILE"

    [[ -n "${OVH_AK:-}" && -n "${OVH_AS:-}" &&
       -n "${OVH_CK:-}" ]] ||
        die "Identifiants API incomplets"

    exec 200>"$LOCKFILE"
    flock -n 200 || exit 0

    NEW_IPV4="$(detect_ipv4)" ||
        die "Détection de l'IPv4 publique impossible"

    NEW_IPV6="$(detect_ipv6)" ||
        die "Aucune IPv6 globale adaptée détectée sur ${IPV6_IFACE}"

    SERVICE_V6="$(build_service_ipv6 "$NEW_IPV6")" ||
        die "Impossible de construire l'IPv6 du service"

    log "Début de synchronisation pour ${ZONE}"
    log "IPv4 publique : ${NEW_IPV4}"
    log "IPv6 hôte : ${NEW_IPV6}"
    log "IPv6 service : ${SERVICE_V6}"

    ensure_record AAAA "" "$NEW_IPV6" || rc=1

    # Exemple : sous-domaine d'un service isolé.
    # Adapter le nom ou commenter ces lignes si inutile.
    ensure_record AAAA "mumble" "$SERVICE_V6" || rc=1
    ensure_record A "mumble" "$NEW_IPV4" || rc=1

    if [[ "$MANAGE_APEX_A" == "1" ]]; then
        ensure_record A "" "$NEW_IPV4" || rc=1
    fi

    if (( CHANGED == 1 )) && [[ "$DRY_RUN" == "0" ]]; then
        if ovh_api POST "/domain/zone/${ZONE}/refresh" >/dev/null; then
            log "Rafraîchissement de la zone demandé à OVHcloud"
        else
            log "ERREUR : rafraîchissement de la zone impossible"
            rc=1
        fi
    fi

    if (( rc == 0 )); then
        log "Synchronisation terminée sans erreur"
    else
        log "Synchronisation terminée avec une ou plusieurs erreurs"
    fi

    return "$rc"
}

main "$@"
```

### 7.1. Adapter la configuration

Les paramètres suivants doivent être vérifiés avant le premier lancement :

- `ZONE` : nom de la zone DNS OVH, par exemple `exemple.tld`.
- `IPV6_IFACE` : interface portant l'adresse IPv6 de référence, par exemple `eth0`, `ens18` ou `br0`.
- `SERVICE_IID` : les 64 bits de l'identifiant d'interface du service, exprimés sous la forme de quatre groupes hexadécimaux.
- `MANAGE_APEX_A` : `0` si un autre outil gère l'IPv4 de l'apex, `1` si le script doit également la gérer.

Pour déterminer le nom de l'interface réseau, tu peux utiliser :

```
ip -br addr
```

Pour examiner ses adresses IPv6 :

```
ip -6 addr show dev eth0 scope global
```

Remplace `eth0` par le nom réel de l'interface.

### 7.2. Comprendre `SERVICE_IID`

Le script ne récupère pas l'adresse IPv6 du conteneur lui-même. Il construit une adresse à partir du préfixe IPv6 `/64` de l'hôte et d'un identifiant de 64 bits fourni dans la configuration.

Par exemple, avec un préfixe fictif `2001:db8:1234:5678::/64` et l'identifiant `0000:0000:0000:0042`, l'adresse construite sera :

```
2001:db8:1234:5678::42
```

Cet exemple utilise un préfixe réservé à la documentation. Il illustre uniquement le calcul.

**Attention :** cette méthode n'est appropriée que si le service utilise effectivement cette adresse dans le même `/64` routé. L'identifiant doit correspondre à l'adresse réellement configurée sur le service ; il ne faut pas le déduire aveuglément d'une adresse MAC. Une adresse construite mais absente du réseau ne rendra pas le service joignable.

### 7.3. Permissions du script

Après avoir enregistré le fichier, applique les permissions suivantes :

```
chown root:root /usr/local/sbin/update_ovh_dns.sh
chmod 700 /usr/local/sbin/update_ovh_dns.sh
```

Le script est ainsi exécutable uniquement par `root`.

## 8. Tester le script

### 8.1. Première exécution en simulation

Avant toute modification réelle, lance :

```
DRY_RUN=1 /usr/local/sbin/update_ovh_dns.sh
```

Le mode simulation effectue les détections et les lectures nécessaires, puis indique les modifications envisagées sans envoyer de requête de modification DNS.

Il ne remplace pas un test réel : il ne garantit pas que les droits de modification sont valides ni que le rafraîchissement de la zone fonctionnera.

### 8.2. Consulter le journal

Pour afficher les dernières opérations :

```
tail -n 50 /var/log/ovh_dns_update.log
```

Pour suivre le journal en direct :

```
tail -f /var/log/ovh_dns_update.log
```

### 8.3. Appliquer les changements

Lorsque la simulation ne présente aucune anomalie, lance le script normalement :

```
/usr/local/sbin/update_ovh_dns.sh
```

Vérifie les messages retournés et le journal. Si le script signale un enregistrement introuvable, ne crée pas de nouvel enregistrement au hasard : vérifie d'abord le nom, le type et la zone DNS concernés.

### 8.4. Vérifier les réponses DNS

Après la mise à jour, interroge le DNS :

```
dig +short A exemple.tld
dig +short AAAA exemple.tld
dig +short A mumble.exemple.tld
dig +short AAAA mumble.exemple.tld
```

Remplace `exemple.tld` par le domaine utilisé.

Les changements ne sont pas nécessairement visibles immédiatement depuis tous les résolveurs, en raison des caches DNS et du TTL.

## 9. Automatiser avec Cron

Une fois le fonctionnement validé, crée un fichier de tâche système :

```
nano /etc/cron.d/update-ovh-dns
```

Ajoute la ligne suivante :

```
*/5 * * * * root /usr/local/sbin/update_ovh_dns.sh
```

Enregistre le fichier et applique des permissions adaptées :

```
chown root:root /etc/cron.d/update-ovh-dns
chmod 644 /etc/cron.d/update-ovh-dns
```

Cette configuration lance le script toutes les cinq minutes.

Comme le script vérifie les enregistrements existants, il ne modifie la zone que lorsqu'une correction est nécessaire. Il effectue néanmoins des lectures API à chaque exécution : il convient donc de tenir compte des limites de requêtes de l'API OVHcloud.

## 10. Adapter le script à son infrastructure

### 10.1. Ne pas gérer l'IPv4 de l'apex

Si `ddclient` ou un autre outil gère déjà l'enregistrement A du domaine principal, conserve :

```
MANAGE_APEX_A=0
```

Le script continue de gérer l'AAAA de l'apex et les enregistrements spécifiques configurés.

### 10.2. Remplacer `ddclient`

Si tu souhaites que le script gère aussi l'IPv4 de l'apex, configure :

```
MANAGE_APEX_A=1
```

Assure-toi au préalable qu'aucun autre outil ne modifie simultanément le même enregistrement. Deux mécanismes concurrents peuvent se remplacer mutuellement et rendre le résultat imprévisible.

### 10.3. Ne pas gérer de sous-domaine spécifique

Si tu n'as pas de service isolé à gérer, supprime ou commente les deux lignes suivantes dans la fonction `main()` :

```
ensure_record AAAA "mumble" "$SERVICE_V6" || rc=1
ensure_record A "mumble" "$NEW_IPV4" || rc=1
```

Tu peux également remplacer `mumble` par le nom d'un autre service, à condition de préparer ses enregistrements DNS et de vérifier la configuration IPv6 correspondante.

Si plusieurs services doivent être gérés, chaque service peut disposer de son propre identifiant d'interface. Dans ce cas, il est préférable d'adapter explicitement le script plutôt que de réutiliser un identifiant unique pour tous les services.

## 11. Dépannage

### Erreur d'authentification API

Vérifie :

- Les trois identifiants dans `/etc/ovh_dns.env`.
- Les autorisations associées au token.
- Le nom exact de la zone DNS.
- L'endpoint API utilisé.

Ne copie jamais les secrets dans un message de dépannage public.

### Aucune IPv6 adaptée n'est détectée

Vérifie l'interface configurée et les adresses disponibles :

```
ip -6 addr show dev eth0 scope global
```

Une adresse absente, temporaire, dépréciée ou non globale ne convient pas à ce mécanisme de détection. Si plusieurs adresses sont présentes, il faut vérifier que celle sélectionnée est bien celle dont le préfixe doit être publié.

### Enregistrement introuvable

Vérifie le type et le nom de l'enregistrement dans la zone OVH. Le script ne crée pas automatiquement les entrées manquantes.

### Plusieurs enregistrements correspondent au même nom

Le script s'arrête pour cet enregistrement. Examine la zone avant de modifier quoi que ce soit : plusieurs adresses peuvent être intentionnelles dans certaines configurations DNS.

### Le DNS ne semble pas actualisé

Vérifie le journal, puis interroge directement le DNS. Distingue une erreur de mise à jour de l'API d'un délai de propagation ou de cache DNS.

### Le script fonctionne manuellement, mais pas avec Cron

Contrôle les permissions, le chemin absolu du script, le fichier de configuration, l'interface réseau et le journal. Cron n'utilise pas nécessairement le même environnement que ton terminal interactif.

## 12. Sécurité et limites

- Les clés API doivent être protégées et limitées aux droits strictement nécessaires.
- Le script doit être exécuté avec les privilèges permettant de lire les adresses réseau et d'écrire le journal.
- Les enregistrements gérés doivent être préexistants et clairement identifiés.
- Le calcul de l'adresse d'un service suppose un `/64` adapté et un identifiant d'interface correct.
- La détection de l'IPv4 dépend d'un service externe : si celui-ci est indisponible, le script s'arrête sans appliquer les modifications.
- Le script ne configure ni les règles de pare-feu, ni le routage, ni les adresses des conteneurs.
- Le mode simulation permet d'anticiper les modifications, mais ne garantit pas la réussite de toutes les opérations en production.

## Conclusion

Cette méthode fournit une solution légère pour synchroniser des enregistrements DNS OVH à partir des adresses d'un serveur Linux, sans dépendre d'une plateforme d'automatisation supplémentaire.

Elle convient notamment aux infrastructures auto-hébergées disposant d'une IPv4 publique et d'un préfixe IPv6 susceptible de changer. Elle demande toutefois une configuration initiale rigoureuse, en particulier pour les services isolés qui possèdent leur propre adresse IPv6.

---

**Auteur :** Amaury, alias BlablaLinux.

Retrouvez l'ensemble de mes services publics sur [blablalinux.be/mes-services-publics/](https://blablalinux.be/mes-services-publics/).

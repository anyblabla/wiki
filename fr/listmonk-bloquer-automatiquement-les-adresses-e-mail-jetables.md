---
title: ListMonk - Bloquer automatiquement les adresses e-mail jetables
description: >-
  Guide pas à pas pour bloquer les adresses e-mail jetables dans ListMonk :
  utilisateur API restreint, script Bash d'import de la liste communautaire
  disposable-email-domains et mise à jour hebdomadaire automatique par cron.
published: true
date: '2026-11-13T15:06:14.315Z'
tags:
  - listmonk
  - newsletter
  - anti spam
  - email
  - api
  - bash
  - cron
  - docker
  - auto hébergement
  - logiciel libre
editor: markdown
dateCreated: '2026-11-13T15:06:14.315Z'
---

Ce guide explique comment **bloquer automatiquement les adresses e-mail jetables** (Yopmail, Mailinator, Guerrilla Mail, etc.) dans une instance **ListMonk** auto-hébergée, en synchronisant chaque semaine la liste de domaines bloqués avec une liste communautaire d'environ 10 000 domaines.

> Les commandes sont données pour une instance ListMonk déployée avec Docker Compose, sur Debian ou Ubuntu, avec l'API accessible en local sur le port `9000`. Adapte les chemins et l'URL à ton installation.
{.is-info}

## Pourquoi faire ça ?

- Une liste de domaines saisie à la main est vite **incomplète** : de nouveaux services d'adresses jetables apparaissent en permanence.
- La liste communautaire [disposable-email-domains](https://github.com/disposable-email-domains/disposable-email-domains) est **vérifiée par des mainteneurs** (chaque ajout doit citer un service permettant de générer une adresse), ce qui limite les faux positifs.
- Le blocage de domaine ne remplace pas le **double opt-in** ni la **gestion des rebonds**, qui restent indispensables pour garder une bonne réputation d'expéditeur.

## Principe de fonctionnement

1. Un script télécharge la liste communautaire.
2. Il la fusionne avec ta propre liste de domaines (fichier texte), nettoie et dédoublonne le tout.
3. Il compare le résultat avec la liste actuellement enregistrée dans ListMonk.
4. **Seulement si elle a changé**, il sauvegarde les paramètres actuels puis envoie la nouvelle liste via l'API de ListMonk.
5. Une tâche cron lance le script chaque semaine.

Le script s'authentifie avec un **utilisateur API dédié**, dont le rôle est limité à la lecture et à la modification des paramètres.

## Prérequis

- Une instance ListMonk fonctionnelle, avec un accès administrateur à l'interface web.
- Un accès `root` (ou `sudo`) sur le serveur qui héberge ListMonk.
- Les paquets `curl` et `jq`.

## Étape 1 : installer les dépendances

```bash
apt update && apt install -y curl jq
```

## Étape 2 : créer un rôle restreint

Dans l'interface de ListMonk, va dans **Utilisateurs → Rôles utilisateur**, puis clique sur **Ajouter**.

1. **Nom** : `blocklist-api`
2. Dans **Autorisations**, plusieurs cases sont cochées par défaut. **Décoche-les toutes** (abonnés, campagnes, rebonds, médias, modèles, y compris `tx:send`). Vérifie visuellement qu'il ne reste rien de coché.
3. Descends tout en bas, dans le groupe **Paramètres**, et coche **uniquement** :
   - `settings:get`
   - `settings:manage`
4. Laisse `settings:maintain` et toutes les autres cases décochées.
5. Clique sur **Enregistrer**.

> Ce rôle ne peut que lire et modifier les paramètres. Si le jeton fuitait un jour, il ne donnerait accès ni à tes abonnés, ni à tes campagnes.
{.is-success}

## Étape 3 : créer l'utilisateur API

Va dans **Utilisateurs → Utilisateurs**, puis clique sur **Ajouter** et renseigne :

| Champ | Valeur |
|---|---|
| Type | **API** |
| Statut | Activé |
| Nom d'utilisateur | `blocklist-bot` |
| Nom | `Mise à jour blocklist` (libre) |
| Rôle utilisateur | `blocklist-api` |
| Rôle de liste | Aucun |

Clique sur **Enregistrer**.

> Le nom d'utilisateur doit correspondre **exactement** à la variable `API_USER` du script.
{.is-warning}

## Étape 4 : enregistrer le jeton

Juste après l'enregistrement, ListMonk affiche le **jeton API une seule fois**. Copie-le immédiatement et stocke-le dans un fichier protégé :

```bash
echo 'LE_JETON_AFFICHE' > /root/listmonk/.api_token
chmod 600 /root/listmonk/.api_token
```

> Si tu fermes la fenêtre sans avoir copié le jeton, il faut supprimer l'utilisateur API et en recréer un. Ne publie ni ne partage jamais ce jeton.
{.is-danger}

## Étape 5 : tester le jeton

```bash
curl -s -o /dev/null -w '%{http_code}\n' \
  -H "Authorization: token blocklist-bot:$(cat /root/listmonk/.api_token)" \
  http://127.0.0.1:9000/api/settings
```

Le résultat attendu est **`200`**.

- `401` : le nom d'utilisateur ou le jeton est incorrect.
- `403` : il manque une permission sur le rôle (vérifie `settings:get` et `settings:manage`).

## Étape 6 : créer ta liste personnalisée

Ce fichier contient **tes propres domaines à bloquer**, en plus de la liste communautaire (un domaine par ligne). Il sera fusionné avec elle à chaque exécution, sans doublon.

```bash
touch /root/listmonk/blocklist-custom.txt
```

Pour y ajouter un domaine :

```bash
echo 'exemple.com' >> /root/listmonk/blocklist-custom.txt
```

> Sans ce fichier, la synchronisation hebdomadaire écraserait les domaines que tu aurais ajoutés à la main dans l'interface.
{.is-info}

## Étape 7 : créer le script

```bash
cat > /root/update_listmonk_blocklist.sh << 'EOF'
#!/usr/bin/env bash
# Synchronise la liste de domaines bloqués de Listmonk avec disposable-email-domains
set -euo pipefail
export LC_ALL=C

LISTMONK_URL="http://127.0.0.1:9000"
API_USER="blocklist-bot"
TOKEN_FILE="/root/listmonk/.api_token"
CUSTOM_FILE="/root/listmonk/blocklist-custom.txt"
BACKUP_DIR="/root/listmonk/settings-backups"
UPSTREAM_URL="https://raw.githubusercontent.com/disposable-email-domains/disposable-email-domains/main/disposable_email_blocklist.conf"
MIN_DOMAINS=1000
KEY="privacy.domain_blocklist"

TMP=$(mktemp -d)
trap 'rm -rf "$TMP"' EXIT
log() { echo "[$(date '+%F %T')] $*"; }

[ -r "$TOKEN_FILE" ] || { log "ERREUR : jeton API introuvable ($TOKEN_FILE)"; exit 1; }
AUTH="Authorization: token ${API_USER}:$(cat "$TOKEN_FILE")"

# 1. Téléchargement de la liste communautaire
curl -fsSL --max-time 60 "$UPSTREAM_URL" -o "$TMP/upstream.txt" \
  || { log "ERREUR : téléchargement impossible"; exit 1; }

# 2. Fusion avec la liste personnalisée, nettoyage, dédoublonnage
cat "$TMP/upstream.txt" "$CUSTOM_FILE" 2>/dev/null \
  | tr -d '\r' | tr '[:upper:]' '[:lower:]' \
  | sed 's/^[[:space:]]*//;s/[[:space:]]*$//' \
  | grep -E '^[a-z0-9]([a-z0-9.-]*[a-z0-9])?\.[a-z0-9-]{2,}$' \
  | sort -u > "$TMP/new.txt"

COUNT=$(wc -l < "$TMP/new.txt")
if [ "$COUNT" -lt "$MIN_DOMAINS" ]; then
  log "ERREUR : seulement $COUNT domaines, abandon (minimum $MIN_DOMAINS)"
  exit 1
fi

# 3. Lecture des paramètres actuels
curl -fsS -H "$AUTH" "$LISTMONK_URL/api/settings" -o "$TMP/settings.json" \
  || { log "ERREUR : lecture des paramètres impossible (jeton ou URL ?)"; exit 1; }
jq -e '.data' "$TMP/settings.json" > /dev/null \
  || { log "ERREUR : réponse inattendue de l'API"; exit 1; }

jq -r --arg k "$KEY" '.data[$k][]?' "$TMP/settings.json" | sort -u > "$TMP/current.txt"

if cmp -s "$TMP/current.txt" "$TMP/new.txt"; then
  log "Aucun changement ($COUNT domaines)"
  exit 0
fi

# 4. Sauvegarde des paramètres actuels (10 dernières conservées)
mkdir -p "$BACKUP_DIR"
cp "$TMP/settings.json" "$BACKUP_DIR/settings-$(date +%F-%H%M).json"
ls -1t "$BACKUP_DIR"/settings-*.json | tail -n +11 | xargs -r rm --

# 5. Construction du nouveau JSON et envoi
jq --arg k "$KEY" --rawfile list "$TMP/new.txt" \
  '.data | .[$k] = ($list | split("\n") | map(select(length > 0)))' \
  "$TMP/settings.json" > "$TMP/payload.json"

HTTP=$(curl -sS -o "$TMP/resp.json" -w '%{http_code}' -X PUT \
  -H "$AUTH" -H "Content-Type: application/json" \
  --data @"$TMP/payload.json" "$LISTMONK_URL/api/settings")

if [ "$HTTP" = "200" ]; then
  log "OK : liste mise à jour ($(wc -l < "$TMP/current.txt") -> $COUNT domaines)"
else
  log "ERREUR : l'API a répondu HTTP $HTTP : $(cat "$TMP/resp.json")"
  exit 1
fi
EOF
chmod 700 /root/update_listmonk_blocklist.sh
```

Le script intègre plusieurs garde-fous :

- il **refuse d'écraser** la liste si le téléchargement renvoie moins de 1 000 domaines ;
- il **ne fait rien** si la liste n'a pas changé (donc aucun redémarrage inutile de ListMonk) ;
- il **sauvegarde** les paramètres avant chaque modification et conserve les 10 dernières sauvegardes.

## Étape 8 : premier lancement

> Lance-le à un moment où **aucune campagne n'est en cours d'envoi** : ListMonk redémarre automatiquement après une modification des paramètres, ce qui interromprait son traitement.
{.is-warning}

```bash
/root/update_listmonk_blocklist.sh
```

Le résultat attendu ressemble à :

```
[2026-11-13 15:53:24] OK : liste mise à jour (0 -> 10182 domaines)
```

Dans l'interface, va dans **Paramètres → Paramètres**, onglet **Confidentialité** : l'onglet **Domaine bloqué** doit afficher environ 10 000 domaines.

## Étape 9 : vérifier que le script est idempotent

Relance le script une seconde fois :

```bash
/root/update_listmonk_blocklist.sh
```

Le résultat attendu est :

```
[2026-11-13 15:55:48] Aucun changement (10182 domaines)
```

ListMonk n'est alors pas redémarré, car rien n'a été modifié.

## Étape 10 : planifier avec cron

Crée une tâche hebdomadaire (ici, le dimanche à 4 h du matin) avec un fichier de log :

```bash
cat > /etc/cron.d/listmonk-blocklist << 'EOF'
0 4 * * 0 root /root/update_listmonk_blocklist.sh >> /var/log/listmonk-blocklist.log 2>&1
EOF
chmod 644 /etc/cron.d/listmonk-blocklist
```

Vérifie que le service cron est actif et que le fichier est en place :

```bash
systemctl status cron --no-pager | head -5
cat /etc/cron.d/listmonk-blocklist
```

> `crontab -l` répondra « no crontab for root » : c'est normal, car la tâche est dans `/etc/cron.d/` et non dans la crontab personnelle de l'utilisateur. Cron lit ce dossier automatiquement.
{.is-info}

## Maintenance

**Consulter les logs** :

```bash
cat /var/log/listmonk-blocklist.log
```

**Ajouter un domaine personnel** : ajoute-le à `blocklist-custom.txt` (voir l'étape 6). Il sera pris en compte à la prochaine exécution, ou immédiatement si tu lances le script à la main.

**Revenir en arrière** : les sauvegardes des paramètres sont dans `/root/listmonk/settings-backups/`. Chaque fichier contient l'ensemble des paramètres de ListMonk tels qu'ils étaient avant la modification.

## Dépannage

| Symptôme | Cause probable | Solution |
|---|---|---|
| `lecture des paramètres impossible` | Jeton ou nom d'utilisateur incorrect, ou ListMonk injoignable | Refais le test de l'étape 5 et vérifie `LISTMONK_URL` |
| HTTP `403` | Permissions manquantes sur le rôle | Vérifie `settings:get` et `settings:manage` |
| `téléchargement impossible` | Pas d'accès sortant vers `raw.githubusercontent.com` | Vérifie le pare-feu et le DNS du serveur |
| `seulement N domaines, abandon` | Téléchargement incomplet ou liste vide | Relance plus tard ; le garde-fou a protégé ta liste actuelle |
| `jq: command not found` | `jq` n'est pas installé | `apt install -y jq` |

## Liens utiles

- [Dépôt disposable-email-domains](https://github.com/disposable-email-domains/disposable-email-domains)
- [Documentation de ListMonk](https://listmonk.app/docs/)

---

**Auteur :** ce guide est proposé par Amaury alias BlablaLinux. Retrouvez tous mes services sur https://blablalinux.be/mes-services-publics/

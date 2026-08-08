# 🖥️ INFRA_NOTES — Zellia / Workflow WhatsApp Cabinet Médical

> **Usage :** Journal des décisions et manipulations d'**infrastructure** (VPS Hostinger, base de données n8n, `.env`, réseau, migrations). Distinct du CHANGELOG workflow (`CHANGELOG_v7-v28_Workflow1.md`), qui ne trace que l'évolution des nœuds. À consulter pour retracer *pourquoi* une décision d'infra a été prise ou *comment* le serveur est configuré.
>
> Pour l'état courant synthétique de l'infra, voir le bloc **Infra** du README principal. Ce fichier contient le détail et l'historique.

---

## Configuration serveur (état courant)

| Élément | Valeur |
|---|---|
| **Hébergement** | Hostinger VPS **KVM 1** (upgrade KVM 2 envisagé) |
| **OS conteneur n8n** | Alpine 3.24 — **Docker Hardened Image** (pas d'`apk`, rien d'installable dans le conteneur) |
| **Version n8n** | v2.7.4 self-hosted |
| **Reverse proxy** | Traefik (SSL Let's Encrypt, `mytlschallenge`) |
| **Conteneurs** | `root-n8n-1`, `root-traefik-1` (compose `/root/docker-compose.yml`) |
| **URL** | `https://n8n.srv1019773.hstgr.cloud` |
| **Base de données** | SQLite (`/home/node/.n8n/database.sqlite`, volume `n8n_data`) — mode WAL actif |
| **Fichier `.env`** | `/root/.env` (sauvegarde : `/root/.env.bak`) |

---

## Journal des décisions

### 24/07/2026 — Pruning des exécutions + alignement fuseau

**Contexte déclencheur :** `database.sqlite` avait atteint **1,2 Go** pour un seul cabinet en test, sans purge configurée depuis l'installation. Présence d'un `crash.journal` (vide) indiquant qu'au moins un crash avait déjà eu lieu — cohérent avec une base qui gonfle sans limite jusqu'à saturer le disque.

**1. Pruning activé** (dans `/root/.env`, service n8n) :
```
EXECUTIONS_DATA_PRUNE=true
EXECUTIONS_DATA_MAX_AGE=336            # 14 jours
EXECUTIONS_DATA_PRUNE_MAX_COUNT=10000
EXECUTIONS_DATA_SAVE_ON_SUCCESS=all
EXECUTIONS_DATA_SAVE_ON_ERROR=all
```
- **14 jours retenus** : au-delà, le taux de consultation réel de l'historique tombe à ~0 (un incident de 3 semaines n'est plus déboguable utilement). Le plafond `MAX_COUNT` protège contre un pic de messages sur une courte période. Les deux limites agissent ensemble (purge dès que l'une est atteinte).
- **`SAVE_ON_SUCCESS=all` conservé** (et non `none`) : la traçabilité des exécutions réussies sera nécessaire pour le **Workflow anti no-show** (prouver « le rappel J-1 est bien parti »). Choix assumé de garder une base un peu plus lourde contre cette capacité de preuve.
- Appliqué via `docker compose -f /root/docker-compose.yml up -d` (recrée le conteneur, volume + base intacts). Vérifié dans le conteneur avec `docker exec root-n8n-1 env | grep EXECUTIONS`.

**2. Fuseau serveur corrigé** — `GENERIC_TIMEZONE` : `Europe/Berlin` → `Africa/Casablanca` dans `/root/.env`.
- **Situation trouvée :** double réglage contradictoire. L'**UI Settings n8n** était correctement sur `Africa/Casablanca` (confirmé visuellement + `settings` en base), mais le `.env` était resté sur `Europe/Berlin` (valeur par défaut du template Hostinger).
- **Sans effet fonctionnel jusqu'ici** : dans n8n l'UI Settings l'emporte sur `GENERIC_TIMEZONE`, et le Workflow 1 force `Africa/Casablanca` en dur dans toutes ses expressions `$now.setZone(...)`. Le Berlin dans le `.env` était une **bombe à retardement inerte** — sans impact tant que l'UI est réglée, mais qui aurait resurgi en cas de reset/réinstall/migration effaçant le réglage UI.
- **Correction = ceinture + bretelles** : les deux sources disent désormais Casablanca, plus aucun conflit possible.
- ⚠️ Ne couvre **pas** les **14 offsets `+01:00` en dur** répartis sur `15-PRD` (×3), `REPORT — Calculer nouveaux créneaux` (×4), `7-LA — Préparer événement` (×4), `7-ACR-CAL` (×2), `7-ACR2-CAL` (×1) — voir dettes actives du README (échéance GMT+0 du 20/09/2026).

**3. VACUUM SQLite — écarté volontairement.**
- Le pruning supprime les *lignes* mais **SQLite ne rend pas l'espace disque** au système : le fichier reste à sa taille max atteinte (1,2 Go). Un `VACUUM` serait nécessaire pour le récupérer.
- **Impossible proprement sur cette installation** : (a) `sqlite3` non installable — image durcie sans `apk` ; (b) la CLI n8n n'expose **aucune** commande de VACUUM/compactage (vérifié sur la doc CLI officielle). La seule voie restante (arrêter n8n, compacter le fichier depuis l'hôte, remettre) est disproportionnée pour du simple confort d'espace.
- **Décision : ne pas faire.** Le fichier ne regrossira plus (le pruning tourne ; SQLite **réutilise** en interne l'espace libéré pour les nouvelles écritures → taille stabilisée). 1,2 Go sur 50 Go de disque ne gêne en rien. Manipuler à la main la base d'une prod pour récupérer un espace inutile = risque réel contre bénéfice nul.

**Filet de sécurité de la session :** snapshot VPS créé via hPanel avant toute manipulation (Backups & Monitoring → Create snapshot ; ⚠️ expire sous 24 h, 1 seul stocké). Backups hebdomadaires automatiques Hostinger = sauvegarde de fond (jusqu'à 4 conservés).

---

### 28/07/2026 — Observation (audit workflow v28) : CRON de nettoyage trop fréquent

**Constat** (relevé à l'audit du fichier `v28Juillet27.json`, pas encore corrigé) : le nœud `CRON — Nettoyage sessions (toutes les 5 min)` a une règle `interval: [{ field: 'minutes' }]` **sans `minutesInterval`** → n8n l'exécute **chaque minute**, soit ≈**1 440 exécutions/jour** qui ne font qu'une lecture Google Sheets + un filtre, le plus souvent pour rien.

**Impact infra :**
- **Volume d'exécutions loggées** : avec `SAVE_ON_SUCCESS=all`, chacune de ces 1 440 exéc/j est écrite en base → le pruning (14 j) travaille en continu pour évacuer du bruit. C'est le principal contributeur au gonflement de `database.sqlite` observé le 24/07.
- **Quota Google Sheets** : 1 lecture/min sur l'onglet `sessions` = consommation constante du quota API, sans valeur métier (les sessions n'expirent qu'à 30 min).

**Action recommandée (côté workflow, pas `.env`)** : ouvrir le nœud → Trigger Interval → **Minutes = 5** (ou 15, l'expiration étant à 30 min). Après correction, la charge de logs du CRON est divisée par ≥5. À traiter **avant** de considérer la migration Postgres — une bonne partie de la pression sur SQLite vient de là, pas de la charge patient réelle (un seul cabinet en test).

> ℹ️ Distinction : cette dette est **dans le workflow** (dette 🔴 du README §14), pas dans la config serveur. Notée ici uniquement parce qu'elle explique en partie le gonflement de base traité le 24/07 et conditionne l'urgence réelle du passage à Postgres.

---

### 05/08/2026 — ✅ CRON de nettoyage corrigé (5 min) + re-audit workflow

**Correction appliquée** (confirmée dans `WhatsApp_Appointment_Automation.json`) : le `Schedule Trigger` de la branche de nettoyage de sessions est passé d'un intervalle mal configuré (déclenchement chaque minute) à une **expression cron `*/5 * * * *`** → exécution **toutes les 5 minutes**.

**Effet infra mesuré/attendu :**
- **Volume d'exécutions loggées divisé par ~5** : de ≈**1 440** à ≈**288 exéc/jour** pour la seule branche CRON. Avec `SAVE_ON_SUCCESS=all`, c'est autant d'écritures en moins en base → le pruning (14 j) ne travaille plus à évacuer ce bruit en continu. **Le principal contributeur au gonflement de `database.sqlite` identifié le 24/07 est neutralisé.**
- **Quota Google Sheets** : lecture de l'onglet `sessions` ramenée à 1/5 min au lieu de 1/min — consommation API résiduelle négligeable.

**Conséquence sur la feuille de route infra :** l'**urgence** du passage à PostgreSQL retombe. La pression sur SQLite venait en grande partie de ce CRON, pas de la charge patient réelle (un seul cabinet en test). La migration Postgres reste recommandée **avant le 3ᵉ cabinet** (pour l'écriture concurrente multi-tenant), mais n'est plus motivée par un risque de saturation disque à court terme.

> **Autres constats du re-audit 05/08 (côté workflow, pour mémoire — détail dans README §14 / CHANGELOG) :** intégrité intacte (0 réf cassée / 0 orphelin / 0 cible inexistante) ; 2 autres dettes d'audit v28 également fermées (`nb_annulations || 0` ; `.item` retiré de `7-LA — Écrire patients_rdv`) ; `name` interne du JSON renommé (`WhatsApp_Appointment_Automation`) ; workflow `active`. Une dette mineure ouverte sans impact infra : `7-LA — Lire liste_attente` sans `alwaysOutputData`.

---

## Décisions futures / à surveiller

- **Migration SQLite → PostgreSQL** *(recommandée avant le 3ᵉ cabinet)* : SQLite verrouille en écriture (`SQLITE_BUSY` sous charge concurrente multi-client) et ne rend pas l'espace disque. PostgreSQL gère l'écriture concurrente nativement et permet le VACUUM. **Moyen concret** : la CLI n8n `export:entities` gère le transfert d'un type de base à l'autre :
  ```
  docker exec -u node root-n8n-1 n8n export:entities --outputDir=./outputs --includeExecutionHistoryDataTables=true
  ```
  puis réimport dans une base Postgres vide (`import:entities --truncateTables true`).
- **Upgrade KVM 1 → KVM 2** *(envisagé)* : redimensionnement à chaud de la même VM (disque/base/volumes conservés). Impose **un reboot** → faire hors heures d'ouverture du cabinet. Vérifier après reboot : conteneurs remontés (`restart: always` OK), IP publique inchangée (sinon reconfigurer l'URL webhook côté Meta), disque étendu (`df -h`, sinon `resize2fs`). N'augmente pas la conso RAM de n8n sans passage en queue mode + workers (utile seulement à partir du 3ᵉ-4ᵉ cabinet).
- **Multi-tenant** *(à partir du 2ᵉ client signé)* : rendre `00-CONFIG` lisible depuis un onglet Google Sheets `clients` indexé par `phone_number_id` (clé de tenant naturelle en Model B). Voir discussion architecture (options A/B/C).
- **Backup quotidien automatique des workflows** *(à mettre en place au passage en production live)* : workflow de sauvegarde déclenché par CRON quotidien (~3h) exportant les workflows en JSON vers un stockage externe (Google Drive), conservation 30 j rolling, notification si échec. **Objectif** : récupération en cas de fausse manipulation sur le canvas — le snapshot VPS couvre la VM entière, ce backup cible spécifiquement les définitions de workflow, plus granulaire et plus fréquent.
  - **Deux architectures possibles :**

    | Approche | Mécanisme | Remarque |
    |---|---|---|
    | **A — Workflow n8n pur** | Schedule → HTTP Request sur l'API n8n → nœud Google Drive | Reste dans n8n (cohérent avec le workflow manuel sur canvas) ; dépend de l'API interne + credential Drive |
    | **B — CRON système + CLI** | `crontab` VPS → `docker exec … n8n export:workflow --backup` → `rclone`/script vers Drive | Plus robuste, pas de token API ; sort de n8n vers le `crontab` Linux |

  - ⚠️ **Contrainte image durcie** : un workflow n8n qui tenterait de lancer une commande shell (`zip`, `docker exec`) se heurte à l'absence d'outils système (même mur que le VACUUM). L'approche A doit rester 100 % nœuds n8n ; l'approche B s'exécute sur l'hôte, hors conteneur.
  - **Note** : la CLI `n8n export:workflow --backup` (voir section Sauvegarde ci-dessous) est le moyen le plus simple et robuste — pas de token d'API à maintenir. Privilégier B si l'aisance avec le `crontab` le permet, sinon A.
  - **Séquencement** : déclencheur = passage au cabinet pilote **en live** (pas avant). En phase de validation actuelle, snapshot VPS + export manuel occasionnel suffisent (un seul cabinet, pas encore de données patients réelles).

---

## Sauvegarde workflows/credentials (CLI n8n)

Complément au snapshot VPS, pour versionner workflows + credentials en JSON directement depuis le conteneur :
```
docker exec -u node root-n8n-1 n8n export:workflow --backup --output=backups/latest/
docker exec -u node root-n8n-1 n8n export:credentials --backup --output=backups/latest/
```
> ⚠️ `export:credentials --decrypted` expose les secrets en clair — à éviter sauf migration entre installations à clés différentes.

---

*INFRA_NOTES — créé le 24/07/2026, dernière entrée le **05/08/2026**. Historique : (24/07) pruning des exécutions (14 j) + alignement fuseau `.env` sur `Africa/Casablanca` + décisions VACUUM (écarté) / PostgreSQL (à venir) ; (28/07) observation CRON trop fréquent ; (**05/08**) **CRON corrigé en `*/5 * * * *`** → volume d'exécutions ÷5, pression SQLite neutralisée, urgence Postgres retombée. Décisions futures : migration Postgres (avant 3ᵉ cabinet), upgrade KVM2, multi-tenant, backup quotidien automatique des workflows. Distinct du CHANGELOG workflow.*

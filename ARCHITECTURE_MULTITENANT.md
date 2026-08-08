# 🏢 ARCHITECTURE MULTI-TENANT — Servir N cabinets avec un workflow

> **Usage :** Décision d'architecture pour passer de 1 cabinet (état actuel) à N cabinets. Extrait d'une discussion projet pour ne pas la perdre au nettoyage des conversations. Référencé depuis `INFRA_NOTES.md` (« rendre `00-CONFIG` lisible depuis un onglet `clients` indexé par `phone_number_id` — options A/B/C »).
>
> **Statut : décision prise → Option B.** Déclencheur d'implémentation = **2ᵉ client signé**. Tant que mono-cabinet, le `00-CONFIG` Set actuel suffit ; migrer avant serait prématuré.

---

## 1. LE PROBLÈME

En Model B (une WABA par cabinet — voir `BUSINESS_LEGAL_CONTEXTE.md` §7), plusieurs cabinets envoient leurs messages au **même workflow n8n**. Or `00-CONFIG — Variables cabinet` est aujourd'hui un nœud **Set en dur** : nom du médecin, IDs Calendar/Sheet, numéro secrétaire y sont écrits dans le JSON pour **un seul** cabinet.

Question : **comment le workflow charge-t-il la bonne config selon le cabinet destinataire du message ?**

### La clé de tenant : `phone_number_id`

Chaque message entrant Meta porte le `phone_number_id` de **la ligne WhatsApp qui l'a reçu** (= le cabinet destinataire). Unique par cabinet en Model B → clé de tenant naturelle.

> ⚠️ **Point d'implémentation critique (relevé à l'audit du JSON v28) :** le nœud `02 — Extraire données message` extrait aujourd'hui `telephone` (expéditeur), `message_brut`, `type_message`, `timestamp` — **mais PAS le `phone_number_id` du destinataire.** L'option B exige de l'extraire d'abord (voir §4, étape 1). Il se trouve dans le payload Meta à `entry[0].changes[0].value.metadata.phone_number_id`, ou selon le shape post-trigger utilisé, à `$json.metadata?.phone_number_id`.

---

## 2. LES TROIS OPTIONS

```
                 1 WORKFLOW, N CABINETS — où vit la config, comment on la route ?
```

| | **A — Duplication** | **B — Config en Sheet** ✅ | **C — Sous-workflow config** |
|---|---|---|---|
| **Principe** | 1 copie complète du workflow par cabinet | 1 workflow ; `00-CONFIG` lit un onglet `clients` filtré sur `phone_number_id` | 1 workflow ; un sous-workflow résout la config et la renvoie via Execute Workflow |
| **Config vit où** | En dur dans chaque JSON (état actuel × N) | Google Sheets, onglet `clients` (1 ligne = 1 cabinet) | Source centralisée (Sheet/base) derrière un sous-workflow |
| **Ajout d'un cabinet** | Dupliquer JSON + Find & Replace + réimporter | **Ajouter une ligne** dans la Sheet | Ajouter une ligne dans la source |
| **Correction de bug** | ⛔ à répliquer sur **chaque** copie | ✅ **une seule fois** | ✅ une seule fois |
| **Complexité initiale** | Nulle (déjà comme ça) | Faible-moyenne | Moyenne-élevée |
| **Isolation d'exécution** | Totale (workflows séparés) | Partagée (1 workflow) | Partagée (1 workflow) |
| **Coût par message** | Aucun surcoût | +1 lookup Sheets `clients` | +1 appel sous-workflow |

---

## 3. ARBITRAGE

**Option A — écartée** au-delà de 2-3 cabinets. La dette de maintenance explose : chaque bug corrigé (et il en reste — offsets `+01:00`, résidus `.item`, `alwaysOutputData`) devrait être réappliqué manuellement à N copies. Piège classique de la duplication. À ne garder que si une contrainte réglementaire imposait une isolation totale des workflows.

**Option B — RETENUE.** Sweet spot pilote → petite échelle (2-5 cabinets) :
- Reste 100 % dans n8n, cohérent avec la stack Google Sheets.
- Debuggable à la main (on lit l'onglet `clients` à l'œil).
- Ajouter un cabinet = ajouter une ligne, zéro manipulation de JSON.
- Coût = 1 lookup Sheets/message, négligeable au volume actuel. La contrainte de quota Sheets (60 req/min/utilisateur) est déjà adressée par la migration **Service Account** prévue.
- Résout au passage la non-dette « `__rl` (resource locator) casse l'autocomplétion » : les IDs Calendar/Sheet aujourd'hui déployés par Find & Replace deviennent des **colonnes** injectées dynamiquement, plus rien en dur.

**Option C — sur-ingénierie pour aujourd'hui.** Ne se justifie qu'à plus grande échelle, quand la résolution de config devient elle-même complexe (héritage de defaults, cache, fallbacks). À reconsidérer bien plus tard, pas avant que B montre ses limites.

---

## 4. IMPLÉMENTATION DE L'OPTION B (plan de recâblage)

> À exécuter **au 2ᵉ client signé**, dans une session dédiée. Récapitulé ici pour figer la cible.

### Étape 1 — Extraire le `phone_number_id` dans `02 — Extraire données message`
Ajouter un 5ᵉ champ au nœud Set `02` (actuellement : `telephone`, `message_brut`, `type_message`, `timestamp`) :

| Champ | Mode | Valeur (à adapter au shape réel du payload post-trigger) |
|---|---|---|
| `phone_number_id_destinataire` | Expression | `={{ $json.metadata?.phone_number_id || $json.entry?.[0]?.changes?.[0]?.value?.metadata?.phone_number_id || '' }}` |

> ⚠️ Vérifier le chemin exact en inspectant une exécution réelle du WhatsApp Trigger — le shape varie selon la version du nœud. Ne pas coder à l'aveugle.

### Étape 2 — Transformer `00-CONFIG — Variables cabinet`
Aujourd'hui : nœud **Set** avec 9 variables en dur (`includeOtherFields: true`).
Cible : **remplacer les valeurs en dur par une lecture Sheets**.

Deux sous-approches possibles :
- **B1 (simple)** : insérer un nœud **Google Sheets (Get Row)** `CONFIG — Lire client` juste avant `00-CONFIG`, filtre `phone_number_id = {{ $('02 …').first().json.phone_number_id_destinataire }}`, `Return First Match` + `Always Output Data`. Puis `00-CONFIG` (Set) recopie chaque colonne lue en variable `PROD_*` : `={{ $('CONFIG — Lire client').first().json.PROD_NOM_MEDECIN }}`, etc. On garde `00-CONFIG` comme point d'accès unique — tout l'aval continue de lire `$('00-CONFIG …').first().json.PROD_XXX` **sans modification**. ✅ Recommandé : rétrocompatible, un seul nœud ajouté.
- **B2 (direct)** : faire lire directement le nœud Sheets par l'aval. ⛔ Non recommandé : oblige à recâbler tous les nœuds aval qui référencent `00-CONFIG`.

> **Choix : B1.** `00-CONFIG` reste le contrat d'interface ; seule sa source change (Sheet au lieu de littéraux). Aucun nœud aval à toucher.

### Étape 3 — Créer l'onglet `clients` dans le Google Sheet
Une ligne par cabinet. Colonnes = les 9 variables actuelles de `00-CONFIG`, indexées par la clé de tenant :

| Colonne | Exemple (cabinet pilote) | Note |
|---|---|---|
| `phone_number_id` | `998215733371244` | 🔑 clé de tenant (match sur l'entrant) |
| `PROD_NOM_MEDECIN` | `Ch. BADROUR` | |
| `PROD_NOM_MEDECIN_FR` | `Dr. Ch. BADROUR` | |
| `PROD_NOM_MEDECIN_AR` | `د. شيماء بدرور` | |
| `PROD_NUMERO_SECRETAIRE` | `212604152155` | |
| `PROD_CALENDAR_ID` | `24c90…@group.calendar.google.com` | résout la non-dette `__rl` |
| `PROD_SHEET_ID` | `1PXikP…FNs9uwno` | ⚠️ voir note ci-dessous |
| `PROD_NOM_CABINET` | *(à renseigner)* | placeholder aujourd'hui |
| `FMT_DATE_PATIENT` | *(helper date)* | ⚠️ voir note ci-dessous |

> **Notes de vigilance :**
> - `PROD_SHEET_ID` en colonne suppose que chaque cabinet a **son propre Google Sheet** de données (`patients_rdv`/`liste_attente`/`sessions`). À décider : Sheet par cabinet (isolation forte, recommandé) ou onglets préfixés dans un Sheet partagé. **Si l'onglet `clients` vit dans le Sheet central mais que les données patient sont par-cabinet**, attention à ne pas confondre le Sheet de config et le Sheet de données.
> - `FMT_DATE_PATIENT` est une **fonction stockée en string** — la mettre en colonne fonctionne mais reste identique pour tous les cabinets ; envisager de la garder en dur dans `00-CONFIG` (constante) plutôt qu'en colonne dupliquée. Seules les vraies variables par-cabinet ont leur place dans la Sheet.

### Étape 4 — Point d'accès inchangé
Comme `00-CONFIG` s'exécute tôt sur le chemin patient (juste après `02`), il reste accessible via `$('00-CONFIG — Variables cabinet').first().json.PROD_XXX` depuis tout l'aval — **pattern `.first()` déjà en place, robuste à la cardinalité.** L'aval ne voit aucune différence.

---

## 5. RÈGLES À GARDER EN TÊTE

- **Lecture config très tôt** : le lookup `clients` doit précéder `00-CONFIG`, lui-même en tête de flux, pour que tout l'aval hérite de la config via `.first()`.
- **`Always Output Data` sur le lookup `clients`** : un `phone_number_id` non trouvé (cabinet non provisionné) doit produire un item géré, pas tuer la branche en silence — même principe que les autres lectures Sheets critiques.
- **Fallback cabinet inconnu** : prévoir un garde-fou si le `phone_number_id` entrant ne matche aucune ligne (message d'erreur secrétaire, ou log). Un message arrivant sur une ligne non provisionnée ne doit pas planter.
- **La branche CRON ne passe pas par `00-CONFIG`** (trigger séparé) : si un jour un nœud CRON multi-tenant doit connaître le cabinet, il faudra résoudre le tenant autrement (le CRON balaie *toutes* les sessions, tous cabinets confondus — repenser à ce moment-là).
- **Isolation des sessions** : avec un Sheet de données par cabinet, l'onglet `sessions` est naturellement isolé. Avec un Sheet partagé, la clé de session devrait inclure le `phone_number_id` pour éviter les collisions entre cabinets.

---

*ARCHITECTURE_MULTITENANT — créé le 08/08/2026. Extrait de la discussion projet référencée dans INFRA_NOTES (« options A/B/C »). Décision : **Option B (config en Sheet indexée par `phone_number_id`)**, implémentation au 2ᵉ client signé, sous-approche B1 (00-CONFIG reste le point d'accès, source = lookup Sheets). Valeurs réelles du cabinet pilote extraites du JSON v28. Point d'attention majeur : `02 — Extraire données message` n'extrait pas encore le `phone_number_id` destinataire — prérequis n°1.*

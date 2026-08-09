# 📋 README PROJET — IA & Automatisation Cabinet Médical Maroc

> **Usage :** Ressource de projet à lire en début de conversation pour restituer le contexte sans réexpliquer. Historique détaillé des versions et dettes résolues → voir `CHANGELOG_v7-v28_Workflow1.md` (sorti d'ici pour alléger).

---

## 📍 ÉTAT COURANT — post-v28, 8 Août 2026

**Phase :** finalisation du **Workflow 1** (Prise de RDV / Annulation / Report / **Inscription liste d'attente** / cascade refus). Développement en parallèle du **Workflow 2 (anti no-show / rappels)** et consolidation de la branche liste d'attente. Le contexte projet est **Zellia** (agence) / produit **Mimosa** (nom interne, jamais client-facing) — cabinet pilote **Dr Chaimaa Badrour, dentiste à Tanger** (`drbadrour.ma`).

**Fichier workflow :** `WhatsApp_Appointment_Automation.json` — **136 nœuds** (dont 2 `NoOp`, 0 sticky note). Intégrité re-vérifiée à l'audit du **08/08/2026** : **0 référence `$('…')` cassée, 0 connexion vers nœud inexistant, 0 orphelin, 0 cible de connexion inexistante.** Workflow `active: true`.

> 🆕 **Depuis le dernier export (vérifié 08/08/2026) :** **BUG-CONV clôturé** (`11c-WA` et `13-PRD` utilisent désormais le pattern cascade `SET → 7-MENU → 7-RDVEX → session 05`, plus de réponse français par défaut sur le sous-flux « choix RDV existant ») et **`alwaysOutputData` ajouté sur `7-LA — Lire liste_attente`**. Résidus `.item` réduits de 34 → **28**. Détail → CHANGELOG §28 + PARTIE A. **Dettes actives restantes** : `phoneNumberId` en dur sur `CRON — Notifier patient session expirée`, credential divergent sur `13-PRD`, préfixe `=` parasite sur `PROD_PHONE_NUMBER_ID`, placeholder `PROD_NOM_CABINET` (voir « Reste à faire »).

> ✅ **`name` interne du JSON désormais renommé en `WhatsApp_Appointment_Automation`** (l'ancien avertissement sur `…v13Juillet27…_Change Fussha` est levé — le fichier a été proprement ré-exporté). Nom de fichier et `name` interne sont maintenant cohérents.

**🆕 Ce qui a changé depuis v17 (récapitulatif — détail complet dans le CHANGELOG) :**
1. **Cabinet pilote = Dr Ch. BADROUR** : `00-CONFIG` mis à jour (`PROD_NOM_MEDECIN=Ch. BADROUR`) + **nouvelles variables bilingues `PROD_NOM_MEDECIN_FR` (`Dr. Ch. BADROUR`) et `PROD_NOM_MEDECIN_AR` (`د. شيماء بدرور`)**.
2. **Architecture nom médecin bilingue** : fonction canonique `projeter()` (mode → `fr`/`ar`) qui sélectionne `PROD_NOM_MEDECIN_FR` ou `_AR` selon `mode_ecriture_final`, avec dédoublonnage du préfixe `Dr` et détection de script arabe. Câblée dans les nœuds de confirmation patient.
3. **Branche ESCAPE** (token `00`) : un patient qui envoie `00` réinitialise sa session (`libre`) et reçoit une notif de reset — sortie de secours si bloqué en session.
4. **Sous-flux « choix RDV existant »** : le blocage anti-doublon (`11c`) ne s'arrête plus sèchement — il propose *reporter (1)* / *annuler (2)* via l'état `attente_choix_rdv_existant` + le nœud `7-RDVEX`.
5. **`7-ACA-IF4 — RDV déjà passé ?`** : une annulation d'un RDV déjà expiré ne déclenche plus la cascade liste d'attente (créneau inutilisable) → notif secrétaire directe.
6. **`7-ACR-LIRE — Relire nb_annulations`** ajouté dans la prise de RDV (relit le compteur avant écriture — tentative de correction de la dette T4.9, **fallback `|| 0` à ajouter, voir dettes**).
7. **Refus de confirmation d'annulation** → propose désormais un report (`7-ACA-IF — Reporter ?`) au lieu de renvoyer sèchement au menu.

**Décisions d'architecture liste d'attente (verrouillées) :**
- **Déclenchement inscription** : uniquement quand **0 créneau libre sur 14 jours** (`FENETRE_RECHERCHE_JOURS`). Simplicité pilote.
- **Préférence stockée** : **aucune** (FIFO pur, tri par `date_demande` format triable `yyyy-MM-dd HH:mm`).
- **Isolation d'état** : `attente_confirmation_inscription_LA` (inscription) **distinct** de `attente_confirmation_liste_attente` (cascade) — accepter *d'entrer* dans la file ≠ accepter *un créneau proposé*.

**✅ Corrigé depuis le dernier export (vérifié à l'audit 08/08/2026) :**
1. ✅ **`name` interne du JSON renommé** (`WhatsApp_Appointment_Automation`) — cohérence version rétablie.
2. ✅ **Les 3 dettes d'audit v28 fermées** (CRON `*/5 * * * *`, `nb_annulations || 0`, `.item` retiré de `7-LA — Écrire patients_rdv`).
3. ✅ **BUG-CONV clôturé (nouveau)** : `11c-WA — Bloquer doublon RDV` et `13-PRD — Demander prénom` lisaient `SET — Profil linguistique final` en direct alors qu'ils sont atteignables via `7-MENU`/`7-RDVEX` (sous-flux « choix RDV existant ») où ce nœud n'est jamais exécuté → réponse français par défaut. **Corrigé** : pattern cascade `SET → 7-MENU → 7-RDVEX → session 05` en try/catch (défaut `inconnu`), vérifié dans le `textBody` des deux nœuds.
4. ✅ **`alwaysOutputData` ajouté sur `7-LA — Lire liste_attente` (nouveau)** : le lookup `telephone` + `statut=en_attente` ne tue plus la branche en silence quand le patient qui annule n'était pas lui-même en attente.

**Reste à faire :**
1. **Valider le chemin NON de l'inscription LA** + les sous-flux (ESCAPE, choix RDV existant, RDV expiré) en conditions réelles.
2. 🟠 **Câbler `phoneNumberId` sur `CRON — Notifier patient session expirée`** — actuellement en dur `998215733371244` (28 autres nœuds Send utilisent `PROD_PHONE_NUMBER_ID`). La branche CRON part d'un trigger séparé et **ne traverse pas `00-CONFIG`** → ajouter un mini-`Set` `CRON-CONFIG` en tête de branche CRON qui redéfinit `PROD_PHONE_NUMBER_ID`. **Bloquant multi-cabinet.**
3. 🟠 **Aligner le credential de `13-PRD — Demander prénom`** sur `UlsIZcI7TK8xx8vd` (utilise aujourd'hui `gd4yOWdWN1iwdMHX`). Corriger au passage son `.item` résiduel (`$('02 — Extraire données message').item` → `.first()`).
4. 🟠 **Nettoyer `PROD_PHONE_NUMBER_ID` dans `00-CONFIG`** : la valeur est `=998215733371244` (préfixe `=` parasite → champ Fixed évalué comme expression). Remettre `998215733371244` **sans** `=`.
5. 🟠 **Renseigner `PROD_NOM_CABINET`** : encore le placeholder `Centre dentaire Dr. X`. Mettre le vrai libellé cabinet Badrour **avant toute démo/mise en prod client-facing.**
6. 🟡 **Ajouter `alwaysOutputData` à `7-ACA-SHT0 — Lire RDV patient`** (lecture critique sans `aod` → silent branch death si 0 RDV).
7. 🟡 **Homogénéiser les 28 résidus `.item`** restants. Aucun n'est bloquant en mono-patient, mais à durcir en `.first()`/`.last()` par hygiène avant toute introduction de batch.

**Contexte v16 (rappel — campagne de test antérieure) :** Workflow 1 fonctionnellement clos, campagne T1–T6 passée (50/50), bug T2.8 (annulation darija → réponse français au lieu de fusha) résolu. Détail → CHANGELOG.

**🔧 Corrections d'infrastructure appliquées en v17 (issues du debug de la branche LA) :**
- **Timezone du workflow réglé sur `Africa/Casablanca`** (Settings n8n). Corrige tous les `$now` qui héritaient du fuseau serveur VPS (GMT+2 en été européen). ⚠️ Le Maroc repasse à **GMT+0 le 20/09/2026** — l'usage du nom de zone IANA fait suivre ce basculement automatiquement, contrairement aux offsets `+01:00` en dur (voir dette).
- **`14-PRD` `timeMax` porté de 7 à 15 jours** — alignement sur la fenêtre de scan de `15-PRD` (14 j). Sans ça, zone aveugle jours 8-14 : les créneaux occupés y étaient vus comme libres (bug rencontré et résolu en test).

---

## 1. CONTEXTE ENTREPRENEUR

### Profil
- Développeur familiarisé avec **n8n** et les outils IA
- Marocain, basé au **Maroc**
- Première aventure entrepreneuriale
- Budget de départ : **15 000 MAD**

### Vision business
Lancer une activité de **conseil et d'automatisation IA** pour les PME marocaines. Le modèle repose sur :
- Des projets ponctuels (automatisation clé en main)
- Des **abonnements mensuels de maintenance** (revenu récurrent)
- De la formation interne aux outils IA

### Marché cible
Le Maroc est en pleine transformation digitale (programme Maroc Digital 2030). La majorité des PME marocaines n'ont aucune automatisation en place — fenêtre d'opportunité réelle, concurrence locale quasi-nulle sur ce segment.

---

## 2. SECTEUR CHOISI POUR DÉMARRER

### Cabinets médicaux privés & petites cliniques

**Pourquoi ce secteur en priorité :**
- La douleur est immédiatement chiffrable : un médecin avec 5 no-shows/jour × 300 MAD = **45 000 MAD/mois perdus**
- ROI démontrable en 2 semaines
- La secrétaire adopte les outils (WhatsApp + Google) sans formation longue
- Le médecin ne touche à rien — il voit juste son agenda toujours propre
- Concurrence locale quasi-nulle sur ce segment
- Revenu récurrent naturel (maintenance mensuelle 500–2 000 MAD/mois)

**Secteurs en attente (phase 2) :**
- Agences immobilières (leads perdus, CRM WhatsApp)
- Fiduciaires / cabinets comptables (collecte docs, relances)
- Centres de formation professionnelle

---

## 3. STACK TECHNIQUE RETENU

### Outils d'automatisation
| Outil | Rôle | Pourquoi |
|---|---|---|
| **n8n** (Cloud ou VPS) | Moteur de workflow | Maîtrisé par l'entrepreneur, flexible |
| **WhatsApp Business API** (Meta) | Canal de communication | Canal n°1 au Maroc |
| **OpenAI GPT-4o** | Intelligence artificielle | Comprend darija, arabe, français, mixte |
| **Google Sheets** | Base de données légère + sessions | Visible, debuggable à la main |
| **Google Calendar** | Gestion des RDV | Familier pour les cabinets |

### Principe de conception
Les outils visibles par le client (ce qu'il utilise au quotidien) sont uniquement :
- **WhatsApp** pour les patients
- **Google Calendar** pour l'agenda
- **Google Sheets** pour les données (consulté par la secrétaire)

n8n et OpenAI sont **invisibles** pour le client — ils tournent en coulisse.

---
## 4. WORKFLOW CONSTRUIT — GESTION RDV CABINET MÉDICAL

### Fichiers livrés
- `WhatsApp_Appointment_Automation.json` — workflow n8n **post-v28** complet à importer *(136 nœuds dont 2 NoOp, 0 sticky). Template multi-cabinet ; cabinet pilote Dr Ch. BADROUR ; nom médecin bilingue (`projeter()` + `PROD_NOM_MEDECIN_FR`/`_AR`) ; branches ESCAPE / choix RDV existant / RDV expiré ; timezone `Africa/Casablanca` ; sélecteurs d'onglet Sheets en mode « By name » ; Phone Number ID Meta externalisé (`PROD_PHONE_NUMBER_ID`). ✅ `name` interne cohérent, workflow `active`. Dettes fermées : les 3 d'audit v28 (CRON 5 min, `nb_annulations || 0`, `.item` retiré de `7-LA — Écrire`) + **BUG-CONV** (cascade `11c-WA`/`13-PRD`) + **`aod` sur `7-LA — Lire liste_attente`**. Dettes actives résiduelles : `phoneNumberId` CRON en dur, credential `13-PRD`, `=` parasite sur `PROD_PHONE_NUMBER_ID`, placeholder `PROD_NOM_CABINET` (voir §14).*
- `README_PROJET_CONTEXTE_V17-17Juillet26.md` — ce document

### Architecture générale (v10)

```
Patient WhatsApp
      ↓
[WhatsApp Trigger]
      ↓
[02 — Extraire données]
      ↓
[00-CONFIG — Variables cabinet]   ← NOUVEAU v10 (Set, includeOtherFields=true)
      ↓                              définit PROD_NUMERO_SECRETAIRE, PROD_NOM_MEDECIN,
      ↓                              PROD_CALENDAR_ID, PROD_NOM_CABINET, PROD_SHEET_ID
[02b — Vrai message patient ?] → 03b — Téléphone existe ? → 03 type = texte ? → NON → [04 — Vocal non supporté]
      ↓ OUI
[05 — Lire session Google Sheets]
      ↓
[06 — Session active ?]
   OUI → [07 — Router selon ÉTAT] → branche selon l'état en cours
   NON → [08 — Interpréteur GPT-4o] → [09 — Parser JSON]
                ↓
         [SWITCH — Mode d'écriture] (7 sorties)
                ↓
    [SET — Style XXX] × 7 styles → [SET — Profil linguistique final]
                ↓
         [10 — Router selon INTENTION]
              ↓             ↓           ↓          ↓
         prise_rdv    annulation     report      autre
                                                   ↓
                                      [Réponse générique — Menu principal]
                                      [SESSION → attente_menu_principal]
```

> **Nœud `00-CONFIG — Variables cabinet` (v10)** : inséré tout en tête de flux, juste après l'extraction du message. C'est un nœud **Set** avec `includeOtherFields: true` (il ajoute les variables `PROD_*` sans écraser les champs amont). Toutes les variables spécifiques au cabinet sont désormais centralisées ici au lieu d'être éparpillées en dur dans les nœuds. Comme il est exécuté en tête du chemin principal, il reste accessible via `$('00-CONFIG — Variables cabinet').first().json.PROD_XXX` depuis tous les nœuds aval situés sur le chemin patient (pattern `.first()` — robuste aux changements de cardinalité).
>
> ⚠️ **Couverture partielle** : `PROD_NUMERO_SECRETAIRE`, `PROD_NOM_MEDECIN` et `PROD_PHONE_NUMBER_ID` sont câblées ; `PROD_CALENDAR_ID`, `PROD_SHEET_ID` et `PROD_NOM_CABINET` sont définies mais déployées par Find & Replace (voir §5 et dettes actives).

---

### Gestion de sessions

Chaque patient a une **session indépendante** stockée dans Google Sheets (onglet `sessions`). Cela permet à plusieurs patients de dialoguer simultanément sans collision ni perte de contexte.

**États de session possibles :**
| État | Signification |
|---|---|
| `libre` | Aucune conversation en cours |
| `attente_prenom` | Le bot a demandé le prénom |
| `attente_choix_creneau` | Le bot a proposé 3 créneaux, attend 1/2/3 |
| `attente_confirmation_rdv` | Le bot attend OUI/NON |
| `attente_confirmation_annul` | Le bot attend Iyeh/La pour confirmer l'annulation |
| `attente_choix_report` | Le bot a proposé de nouveaux créneaux pour un report |
| `attente_menu_principal` | Le bot a envoyé le menu général (1/2/3/4), attend le choix |
| `attente_confirmation_liste_attente` | Un créneau libéré a été proposé à un patient en liste d'attente, attend Oui/Non |
| `attente_confirmation_inscription_LA` | **(v17)** Le bot a proposé l'*inscription* en liste d'attente (0 créneau dispo), attend OUI/NON — **distinct** de `attente_confirmation_liste_attente` : ici le patient n'a encore aucune place, il consent à *entrer* dans la file |
| `attente_choix_rdv_existant` | **(v28)** Le patient a déjà un RDV `confirmé` et a redemandé un RDV → le bot lui propose *reporter (1)* ou *annuler (2)* le RDV existant. Lu par `7-RDVEX — Lire choix RDV existant` (mapping **1=report / 2=annulation**, ⚠️ différent du menu principal où 1=prise_rdv) |

Un **CRON de nettoyage** remet à `libre` les sessions inactives depuis plus de 30 minutes (`EXPIRATION_MS = 30 min`).

> ✅ **Fréquence CRON corrigée (vérifié 05/08/2026)** : le `Schedule Trigger` est passé en expression cron `*/5 * * * *` → exécution toutes les 5 minutes (≈288 lectures Sheets/jour). L'ancienne incohérence (`interval: [{ field: 'minutes' }]` sans `minutesInterval`, qui déclenchait chaque minute) est résolue.

---

### Système de profil linguistique (7 modes — mis à jour v4)

**Principe :** GPT-4o retourne un champ `mode_ecriture` (en plus de `intention`, `urgence`, `infos`). Ce champ alimente un pipeline de profil linguistique avant le routage par intention.

**Flux après le nœud 09 — Parser réponse IA :**
```
[09 — Parser]
      ↓
[SWITCH — Mode d'écriture]
      ↓ (7 sorties)
SET darija_latin | SET darija_arabe | SET arabe_fussha | SET francais_approximatif
SET francais_correct | SET mixte | SET inconnu
      ↓ (convergent tous vers)
[SET — Profil linguistique final]
      ↓
[10 — Router selon intention]
```

**Valeurs de `mode_ecriture` (7 — `arabe_fussha` ajouté en v4) :**
| Valeur | Exemple patient | Style de réponse |
|---|---|---|
| `darija_latin` | `bghit rdv` | Darija marocaine en lettres latines |
| `darija_arabe` | `بغيت رنديفو` | Darija marocaine en alphabet arabe |
| `arabe_fussha` | `أريد حجز موعد` | Arabe standard moderne (fus'ha), registre soutenu |
| `francais_approximatif` | `je veu un rdv` | Français simple et court |
| `francais_correct` | `Bonjour, je souhaite prendre RDV` | Français poli et professionnel |
| `mixte` | `salam je veux rdv` | Mélange darija latin + français |
| `inconnu` | Message incompréhensible | Fallback français |

**Champ `mode_ecriture_final`** : propagé dans le contexte session (JSON stringifié) pour être récupéré lors des échanges suivants, même sans repasser par GPT-4o.

> ### Projection des 7 modes d'entrée vers 2 modes de sortie
>
> GPT-4o **détecte toujours les 7 modes** en entrée (SWITCH + 7 nœuds `SET — Style XXX` inchangés), mais la **réponse** ne se fait que dans **2 langues**. Projection centralisée dans le champ `mode_ecriture_final` de **`SET — Profil linguistique final`** :
>
> | `mode_ecriture` détecté (entrée) | → `mode_ecriture_final` (sortie) |
> |---|---|
> | `francais_correct`, `francais_approximatif` | **`francais_correct`** |
> | `arabe_fussha`, `darija_latin`, `darija_arabe`, `mixte` | **`arabe_fussha`** |
> | `inconnu` / vide / inattendu | **`francais_correct`** (fallback) |
>
> **Conséquence** : darija ou mixte → réponse en fusha ; français (même approximatif) → français correct. Plus aucune réponse en darija/mixte émise. La valeur projetée est propagée dans le contexte session → tous les tours suivants en héritent.
>
> **Couverture** : **17 nœuds patient** rendent dynamiquement (prise RDV, annulation, report, LA, menu). Les 7 nœuds en français fixe sont légitimes (6 notifs secrétaire + `04 — Vocal non supporté` + `CRON — Notifier patient`). *Historique de la convergence (v13 puis correctif branche Report v14 MODIF1–3) → CHANGELOG.*
>
> ⚠️ **Seul angle mort restant** : `liste_attente.mode_ecriture_final` (saisie manuelle) n'est PAS projetée à la lecture — une valeur `darija_arabe` en sheet déclenche la branche darija au lieu de fusha. À traiter avant déploiement LA. Voir dettes actives.
>
> *Principe : projeter à la source propage seul via le contexte session, mais chaque textBody aval doit savoir rendre le mode cible — d'où l'audit branche par branche (la branche Report avait été manquée une version entière).*

**Confirmations multilingues** : les nœuds `7-ACR-WA — Confirmer au patient` (RDV ordinaire) et `7-LA-WA — Confirmer au patient` (liste d'attente) génèrent un message différent selon `mode_ecriture_final` avec une **structure identique** (emojis + date formatée "lundi 1 juin à 10:00" + médecin + signature).

---

### Menu principal et intention inconnue

Quand l'intention détectée est `autre` ou non reconnue, le workflow :
1. Envoie un **menu numéroté** au patient (4 options)
2. Sauvegarde la session en état `attente_menu_principal`
3. À la réponse suivante → nœud `7-MENU — Lire choix menu principal` interprète le chiffre **sans passer par GPT-4o**

```javascript
// Logique de lecture du menu (sans GPT, ultra-rapide)
['1', '١', 'rdv', 'rendez-vous']  → prise_rdv
['2', '٢', 'annuler', 'annulation'] → annulation
['3', '٣', 'reporter', 'changer']  → report
['4', '٤', 'question', 'autre']    → question
```

---

### Structure du JSON GPT-4o (v4 — mise à jour)

Le System Prompt du nœud 08 retourne `mode_ecriture` avec désormais **7 valeurs** (ajout de `arabe_fussha`) :

```json
{
  "intention": "prise_rdv | annulation | report | confirmation | question | autre",
  "mode_ecriture": "darija_latin | darija_arabe | arabe_fussha | francais_approximatif | francais_correct | mixte | inconnu",
  "urgence": true | false,
  "infos": {
    "prenom": null,
    "jour_souhaite": null,
    "heure_souhaitee": null,
    "nouvelle_date": null,
    "nouvelle_heure": null,
    "raison": null,
    "tiers": null
  }
}
```

> ⚠️ Le champ est `mode_ecriture` et non `langue`. Toutes les références à `$json.langue` ont été remplacées.

---

### Branches du workflow

**✅ Prise de RDV (VÉRIFIÉE en test, raffinée en v6, anti-doublon ajouté en v8) :**
1. Cherche si patient connu dans `patients_rdv` via `11-PRD — Chercher patient existant` (filtre par téléphone, retourne toutes les lignes — y compris RDV passés ou annulés)
1bis. **Bloc anti-doublon RDV (NOUVEAU v8)** :
   - `11a-PRD — Vérifier RDV actif` : relit `patients_rdv` avec **double filtre** `telephone` + `statut=confirmé` → ne retourne que les RDV actifs (les `annulé` sont ignorés).
   - `11b-PRD — A déjà un RDV ?` : IF `event_id exists`.
     - **OUI (RDV actif existant)** → `11c-WA — Bloquer doublon RDV` : message multilingue (lit `mode_ecriture_final`) rappelant la date/heure du RDV existant et proposant un **menu numéroté** *1 = reporter / 2 = annuler*. **🆕 (v28) La session bascule en `attente_choix_rdv_existant`** (`SESSION — Écrire attente_choix_rdv_existant`) — le patient n'est plus laissé dans le vide. Voir la sous-branche « Choix RDV existant » ci-dessous.
     - **NON** → continue vers `12-PRD — Patient connu ?`
2. Si inconnu → demande le prénom (`13-PRD`) → session `attente_prenom`
3. Si connu → récupère les créneaux libres via Google Calendar (`14-PRD`)
4. `15-PRD — Calculer créneaux disponibles` :
   - Hydrate `prenom` selon la cascade : `Edit Fields` (nouveau patient) > **`11-PRD` (sheet patients_rdv — source de vérité)** *(v6)* > `ctx.prenom` (session mémorisée) > `''`
   - Calcule les 3 prochains créneaux disponibles, normalisés UTC+1 Maroc
5. Propose les 3 créneaux → session `attente_choix_creneau` (le `prenom` est stocké dans le contexte JSON pour être relu à la prochaine réponse du patient)
6. Patient répond 1/2/3 → `7-ACR — Valider choix créneau` → IF choix valide
   - Valide → crée l'événement Calendar (`7-ACR-CAL`) → **🆕 (v28) `7-ACR-LIRE — Relire nb_annulations`** (Get Row `patients_rdv`, filtre `telephone`, relit le compteur dans l'exécution courante) → **`7-ACR-SET — Préparer données RDV`** : nœud Set qui hydrate toutes les colonnes du RDV (telephone, prenom, medecin, date_rdv, heure_rdv, statut=`confirmé`, event_id=`$json.id`, `nb_annulations` recopié depuis `7-ACR-LIRE`, mode_ecriture_final relu depuis le contexte session via JSON.parse) → `7-ACR-SHT — Enregistrer RDV` (Sheets `appendOrUpdate`, lit `$json.*` local) → confirme patient (multilingue via `projeter()`) + notifie secrétaire
   - Invalide → renvoie le menu créneaux
7. Libère la session (`libre`)

> ✅ **(résolu post-v28) `7-ACR-SET` / `nb_annulations`** : le champ vaut désormais `={{ $json.nb_annulations || 0 }}` — un nouveau patient (item vide de `7-ACR-LIRE` en aod) écrit `0` au lieu de `undefined`. Maillon final de T4.9 posé. Voir §14.

**🆕 (v28) Sous-branche « Choix RDV existant » (patient qui a déjà un RDV et redemande) :**

Déclenchée par le bloc anti-doublon quand `11b-PRD — A déjà un RDV ?` = OUI. Se déroule en **2 exécutions** (le patient tape `1` ou `2` dans un 2ᵉ message).

```
[11c-WA — Bloquer doublon RDV]  (menu multilingue : 1=reporter, 2=annuler)
   → [SESSION — Écrire attente_choix_rdv_existant]
     (contexte = {mode_ecriture_final} ; terminal de 1ʳᵉ exécution)

[Réponse 1/2 du patient]  (2ᵉ exécution)
   → [07 — Router]  (sortie 7 : état attente_choix_rdv_existant)
   → [7-RDVEX — Lire choix RDV existant]  (Code : mappe 1→report, 2→annulation)
     ⚠️ mapping INVERSE du menu principal (où 1=prise_rdv) — ne pas confondre
   → [10 — Router selon intention]  (réutilise le routeur d'intention standard)
        report      → branche Report
        annulation  → branche Annulation
        autre       → menu principal (fallback)
```

> **Point clé** : `7-RDVEX` réinjecte dans `10 — Router selon intention` en produisant un champ `intention` (`report`/`annulation`) — pas de branche dédiée, on recycle le routage existant. Le `mode_ecriture_final` est propagé depuis le contexte session pour rester bilingue.

**🆕 (v28) Branche ESCAPE (token de secours `00`) :**

Insérée **entre `05 — Lire session patient` et `06 — Session active ?`**, elle intercepte le message brut `00` avant tout traitement.

```
[05 — Lire session patient]
   → [ESCAPE — Token réservé 00 ?]  (IF : message_brut.trim() === '00')
        OUI → [SESSION — Libre (ESC)]  (force etat=libre, contexte={})
            → [ESCAPE — Notifier reset]  (WhatsApp « session réinitialisée »)
        NON → [06 — Session active ?]  (flux normal)
```

> **Usage** : filet de sécurité si un patient reste coincé dans un état (bug, abandon en cours de dialogue). Il tape `00` et repart de zéro sans intervention de la secrétaire. Le token `00` est réservé — à mentionner dans le message d'onboarding.

**✅ Annulation (VÉRIFIÉE en test, raffinée en v6, refondue en v8 ; branche « RDV expiré » ajoutée v28) :**
1. `7-ACA-SHT0 — Lire RDV patient` puis (via `ANNUL — Chercher RDV existant` côté entrée de branche) → IF RDV trouvé
2. Si aucun RDV → `ANNUL — Aucun RDV trouvé`
3. Affiche le RDV et demande confirmation (Iyeh/La) → session `attente_confirmation_annul`. **Message multilingue selon `mode_ecriture_final`** *(v6)*.
4. `7-ACA — Lire réponse confirmation annulation` → `7-ACA-IF — Confirme annulation ?`
   - NON → `Réponse générique — Menu principal` *(v8 — un refus de confirmation renvoie au menu au lieu de libérer sèchement)*
5. Si confirmé → `7-ACA-SHT0 — Lire RDV patient` → `7-ACA-CAL` supprime l'événement Calendar → `7-ACA-SHT` met à jour la ligne dans Sheets :
   - statut → `annulé`
   - `nb_annulations` → incrémenté de 1 *(v6)*
   - **toutes les autres colonnes (prenom, medecin, date_rdv, heure_rdv, event_id) préservées explicitement** *(v6)*
6. `7-ACR-HIST — Logger annulation` (journalisation)
7. **`7-ACA-FLAG — Classer annulation` (NOUVEAU v8 — remplace l'ancien `7-ACA-IF2`)** : nœud **Code** qui calcule `annulation_tardive` (RDV à moins de 2h), **🆕 (v28) `rdv_expire`** (RDV déjà passé, `diffMs < 0`) et `type_annulation` (`tardive`/`normale`), puis propage le contexte enrichi.
7bis. **🆕 (v28) `7-ACA-IF4 — RDV déjà passé ?`** (IF sur `rdv_expire`) : décide si le créneau libéré vaut la peine d'être reproposé.
   - **OUI (RDV expiré)** → `7-ACA-NOTIF — Résumé annulation secrétaire` **directement** : un créneau dans le passé est inutilisable, inutile de déclencher la cascade liste d'attente.
   - **NON (RDV futur)** → continue vers la consultation liste d'attente (étape 8).
   > **Raison** : sans ce garde-fou, annuler un no-show déjà passé notifiait un patient en attente pour un créneau… déjà écoulé. Le flag `rdv_expire` évite cette fausse notification.
8. **Liste d'attente consultée AVANT la notification secrétaire (réordonnancement v8)** : `7-ACA-SHT2 — Chercher liste attente` (filtre `statut=en_attente` + `returnFirstMatch`) → `7-ACA-IF3 — Quelqu'un en attente ?`
   - OUI → `7-ACA-WA — Notifier patient en attente` → `7-ACA-CTX — Préparer session attente` → `SESSION — Écrire attente_liste_attente` → puis converge vers la notification secrétaire
   - NON → directement vers la notification secrétaire
9. **`7-ACA-NOTIF — Résumé annulation secrétaire` (NOUVEAU v8 — notification UNIQUE consolidée)** : remplace les deux anciens nœuds séparés (`Alerte TARDIVE` / `Notification normale`). Un seul message à la secrétaire qui agrège : infos du RDV annulé, drapeau tardif/normal (lu depuis `7-ACA-FLAG`), et l'état de la liste d'attente (patient en attente notifié ou créneau orphelin — lu depuis `7-ACA-SHT2` via cascade `try/catch`).
10. `7-ACA-WA — Confirmer annulation au patient` : message multilingue **sobre, registre médical**, sans CTA "Tapez OUI" *(v6)*.
11. Libère la session → `SESSION — Libre (ANNUL)`

**✅ Liste d'attente avec cascade refus (VÉRIFIÉE en test — v7) :**

Déclenchée après une annulation qui libère un créneau. Branche complétée et stabilisée le 29/05/2026, étendue avec cascade refus le 05/06/2026.

**Décision architecturale clé (v7) :** abandon du `delete` Google Sheets au profit d'un **statut** dans la sheet `liste_attente`. Raison : Google Sheets refuse de supprimer la dernière ligne non figée d'une feuille (erreur `INVALID_ARGUMENT`), et garder un historique est de toute façon précieux pour le cabinet. Une colonne `statut` (`en_attente` / `traité`) a donc été ajoutée à la sheet `liste_attente`.

**Hypothèse métier v7 :** cabinets **mono-médecin** uniquement. Conséquence : la clé d'unicité d'un patient en file d'attente est son `telephone` seul (un patient ne peut avoir qu'une ligne `en_attente` à un instant T). Pour cabinets multi-médecins, il faudra introduire une clé composite `telephone + medecin` ou un `id_la` unique.

```
[ANNUL] → 7-ACA-SHT2 — Chercher liste attente
          (filtre statut=en_attente + Return only First Matching Row → FIFO)
       → 7-ACA-IF3 — Quelqu'un en attente ?
            OUI → 7-ACA-WA — Notifier patient en attente
                → 7-ACA-CTX — Préparer session attente
                  (état = attente_confirmation_liste_attente
                   + contexte = {date_rdv, heure_rdv, medecin, event_id, prenom, mode_ecriture_final})
                → SESSION — Écrire attente_liste_attente
                  (propage tel quel $json.contexte — NE PAS reconstruire)

[Réponse du patient en attente]
       → 07 — Router (état attente_confirmation_liste_attente)
       → 7-LA — Lire réponse Oui/Non (parser tolérant : 20+ variantes oui/non FR/AR/darija)
                   → produit le champ $json.reponse_liste_attente = 'oui' | 'non' | 'inconnu'
       → 7-LA — IF réponse = OUI  (teste reponse_liste_attente == 'oui')
            OUI  → 7-LA — Préparer événement (lit ctx à la RACINE + mode_ecriture_final)
                 → 7-LA — Re-vérifier dispo (Calendar Get Many)
                 → 7-LA — IF créneau encore libre ?
                      LIBRE   → 7-LA-CAL — Créer RDV (summary "RDV - prenom - tel")
                              → 7-LA — Lire liste_attente (filtre statut=en_attente)
                              → 7-LA — Écrire patients_rdv (event_id = node Calendar, nb_annulations = 0)
                              → 7-LA — Marquer LA traitée
                                (Update Row, match sur telephone, set statut=traité — v7)
                              → 7-LA-WA — Confirmer au patient (multilingue, format aligné RDV ordinaire)
                              → SESSION — Libre (LA)
                      REPRIS  → 7-LA-WA — Désolé, créneau repris (multilingue, recipient avec String())
                              → SESSION — Libre (LA repris) (timestamp ISO avec $now.toISO())
            FAUX → 7-LA-IF — Réponse = NON ?  ← NOUVEAU v10 (teste reponse_liste_attente == 'non')
                      NON (vrai refus) → 7-LA-WA — Patient refuse (multilingue)
                            → 7-LA — Marquer LA traitée (refus)        ← NOUVEAU v7 (cloné)
                              (P1 sort de la liste, statut=traité)
                            → 7-LA — Chercher suivant LA               ← NOUVEAU v7
                              (filtre statut=en_attente
                               + medecin_souhaite = medecin du créneau libéré (extrait du contexte session)
                               + Return only First Matching Row
                               + Always Output Data)
                            → 7-LA-IF — Suivant trouvé ?               ← NOUVEAU v7
                              (Value: String($json.telephone || '') — Operator: "is not empty"
                               — Convert types where required ✅ activé
                               PAS "exists" qui matche aussi undefined)
                      TRUE  → 7-LA — Préparer notif suivant ← NOUVEAU v7
                                (fusionne profil P2 + créneau lu depuis contexte session
                                 via JSON.parse de $('7-LA — Lire réponse Oui/Non').item.json.contexte
                                 — PAS $('7-LA — Préparer événement') qui est dans la branche OUI)
                            → 7-LA-WA — Notifier suivant    ← NOUVEAU v7
                              (multilingue, lit mode_ecriture_final du suivant)
                            → 7-LA — Préparer session P2    ← NOUVEAU v7
                              (état = attente_confirmation_liste_attente + contexte complet)
                            → SESSION — Écrire attente_LA (cascade refus)  ← NOUVEAU v7 (cloné)
                              (Append or Update, match sur telephone, mode expression "=" activé)
                            → SESSION — Libre (LA refus)
                      FALSE → 7-LA — Notif secrétaire (créneau orphelin)  ← NOUVEAU v7
                            → SESSION — Libre (LA refus)
                      INCONNU (réponse non comprise) → 7-LA-WA — Redemander Oui/Non  ← NOUVEAU v10
                            (message multilingue « je n'ai pas compris, répondez Oui ou Non »
                             — NE TOUCHE PAS à la session : le patient reste en
                             attente_confirmation_liste_attente et peut re-répondre)
```

> **Points clés de la branche LA (v7, complétée v10) :**
> - **Réponse Oui/Non/Incompris — logique 3-way** *(v10)* : le parser `7-LA — Lire réponse Oui/Non` retourne `reponse_liste_attente ∈ {oui, non, inconnu}`. Deux IF en cascade routent les trois cas :
>   - `7-LA — IF réponse = OUI` (`reponse_liste_attente == 'oui'`) → branche acceptation.
>   - Sortie FALSE → `7-LA-IF — Réponse = NON ?` (`reponse_liste_attente == 'non'`) → vrai refus → cascade de réattribution.
>   - Sortie FALSE de ce 2ᵉ IF (donc `inconnu`) → `7-LA-WA — Redemander Oui/Non` : message multilingue de re-sollicitation **qui ne modifie pas la session**. Le patient reste en `attente_confirmation_liste_attente` et son prochain message repasse par le même parser. **Résout la dette « parser rate sur réponses courtes »** : auparavant `inconnu` était silencieusement traité comme un refus (le patient perdait le créneau sans comprendre pourquoi). Désormais une réponse ambiguë déclenche une clarification, pas une perte de créneau.
> - **Statut au lieu de delete** : `7-LA — Marquer LA traitée` fait un `Update Row` (clé `telephone`, valeur `statut=traité`). Le `delete` Sheets a été abandonné car Google refuse de supprimer la dernière ligne non figée.
> - **Filtre `statut=en_attente`** présent sur `7-ACA-SHT2 — Chercher liste attente`, `7-LA — Lire liste_attente`, et `7-LA — Chercher suivant LA` — garantit qu'on ne re-notifie jamais un patient déjà traité.
> - **Filtre `medecin_souhaite` sur `7-LA — Chercher suivant LA`** *(v7)* : compare la colonne `medecin_souhaite` de `liste_attente` à la valeur `medecin` du contexte session (créneau libéré). Anticipation multi-médecin sans coût en mono-médecin. Requiert que les chaînes soient strictement identiques entre `patients_rdv.medecin` et `liste_attente.medecin_souhaite` (mêmes espaces, casse, ponctuation).
> - **`Return only First Matching Row`** sur `7-ACA-SHT2` et `7-LA — Chercher suivant LA` → garantit FIFO (le patient inscrit en premier est servi en premier) + évite les multi-notifications concurrentes (bug latent v6 résolu).
> - **IF `Suivant trouvé ?` robuste aux types** *(v7)* : `Value 1 = ={{ String($json.telephone || '') }}`, `Operator = is not empty`, `Convert types where required` ACTIVÉ. Indispensable car Google Sheets renvoie `telephone` en type `number` → un IF strict en mode String plante avec `Wrong type: 'XXX' is a number but was expecting a string`.
> - **Cascade récursive naturellement** : si P2 refuse aussi, la même branche se redéclenche → P3 reçoit la notif → etc. Aucun code spécifique nécessaire.
> - **Création event** : `event_id` dans `7-LA — Écrire patients_rdv` référence explicitement `$('7-LA-CAL — Créer RDV').item.json.id` — pas `$json.id`.
> - **Contexte session** : `SESSION — Écrire attente_liste_attente` recopie tel quel `$json.contexte` (déjà préparé par `7-ACA-CTX`).
> - **Téléphone WhatsApp** : tous les nodes WA de la branche LA enveloppent le téléphone dans `String(...)`.
> - **Format de date confirmation** : `7-LA-WA — Confirmer au patient` formate la date avec `toLocaleDateString('fr-FR', {weekday:'long', day:'numeric', month:'long'})` et déduplique le préfixe "Dr" via regex `/^dr\.?\s/i`.
> - **Référence cross-branche INTERDITE** : dans la branche NON, on ne peut pas remonter à `7-LA — Préparer événement` (qui est dans la branche OUI). Les Code nodes de la cascade refus lisent le contexte créneau via `JSON.parse($('7-LA — Lire réponse Oui/Non').item.json.contexte)`.
> - **Secrétaire alertée si liste épuisée** : `7-LA — Notif secrétaire (créneau orphelin)` envoie un WhatsApp à la secrétaire avec date/heure/médecin du créneau orphelin — à recaser manuellement.

**✅ Inscription liste d'attente — ENTRÉE (NOUVEAU v17, chemin OUI testé end-to-end) :**

Déclenchée en **fin de branche Prise de RDV**, quand `15-PRD — Calculer créneaux disponibles` ne retourne **aucun créneau** sur la fenêtre de recherche (14 jours). Complète le cycle LA : cette branche *remplit* `liste_attente` (la cascade refus v7 la *vide*). Se déroule en **2 exécutions n8n séparées** (le patient répond OUI/NON dans un 2ᵉ message qui redéclenche le Trigger).

```
[15-PRD — Calculer créneaux] (produit creneaux[])
       ↓
[LA-IN-IF-Créneaux dispo?]  (IF : ($json.creneaux || []).length > 0)
    true (out 0)  → [16-PRD — Proposer créneaux]  (flux RDV normal, INCHANGÉ)
    false (out 1) → [LA-IN-WA Proposer inscription]  (WhatsApp multilingue :
                    « aucun créneau, voulez-vous être inscrit ? OUI/NON »)
                  → [SESSION — Écrire attente_inscription_LA]
                    (état = attente_confirmation_inscription_LA
                     contexte = {prenom, medecin_souhaite, mode_ecriture_final}
                     ⚠️ SOURCE = $('15-PRD …').first() — PAS $json :
                     après un nœud WhatsApp Send, $json = réponse Meta, pas les données patient)
                    [nœud TERMINAL de 1ʳᵉ exécution — pas de sortie]

[Réponse OUI/NON du patient]  (2ᵉ exécution, redéclenche le Trigger)
       ↓
[07 — Router]  (sortie 6 : état attente_confirmation_inscription_LA)
       ↓
[LA-IN — Lire réponse Oui/Non]  (Code : lit le texte via
                                 $('02 — Extraire données message').first().json.message_brut ;
                                 réhydrate ctx depuis la session ;
                                 produit $json.reponse = 'OUI' | 'NON')
       ↓
[LA-IN-IF2 — Réponse = OUI ?]  (IF : $json.reponse == 'OUI')
    OUI (out 0) → [LA-IN — Inscrire (append)]  (Google Sheets APPEND dans liste_attente :
                                                statut=en_attente,
                                                date_demande = $now.setZone('Africa/Casablanca').toFormat('yyyy-MM-dd HH:mm'),
                                                mode_ecriture_final rempli)
                → [LA-IN-WA — Confirmer inscription]  (WhatsApp « inscrit ✅ » multilingue)
                → [SESSION — Libre (LA-IN)]
    NON (out 1) → [LA-IN-WA — Refus poli]  (WhatsApp « pas de souci » multilingue)
                → [SESSION — Libre (LA-IN)]
```

> **Points clés de la branche Inscription LA (v17) — tous rencontrés et résolus en test :**
> - **Couplage fenêtre `14-PRD` ↔ `15-PRD`** : `15-PRD` scanne les créneaux sur `FENETRE_RECHERCHE_JOURS = 14` jours, mais `14-PRD` (Get Calendar) ne récupérait les événements que sur 7 jours (`timeMax`). **Zone aveugle jours 8-14** → un créneau occupé dans cette zone était vu comme libre → la LA ne se déclenchait jamais pour un agenda plein « à moyen terme ». **Fix : `timeMax` de `14-PRD` porté à 15 jours** (marge d'1 jour sur les 14 du scan). **Principe : `timeMax` de `14-PRD` ≥ `FENETRE_RECHERCHE_JOURS` de `15-PRD`, toujours.**
> - **La fenêtre de recherche EST le seuil de bascule LA** : le même `FENETRE_RECHERCHE_JOURS` sert à (1) chercher des créneaux à proposer ET (2) décider quand basculer en liste d'attente. Élargir ce nombre réduit les inscriptions LA ; le réduire les augmente. Fixé à 14 (quinzaine) — arbitrage « on propose la quinzaine, au-delà on prévient dès qu'une place se libère ». À ajuster avec le cabinet selon sa gestion des reports/urgences.
> - **`$json` après un WhatsApp Send = réponse Meta** : `SESSION — Écrire attente_inscription_LA` est placé APRÈS le WhatsApp de proposition → `$json.telephone`/`$json.prenom` y sont `undefined` (la sortie du nœud WhatsApp est la réponse de l'API Meta : `messaging_product`, `contacts`, `messages[]`). **Fix : référencer explicitement `$('15-PRD …').first().json.*`** (dernier nœud Code portant les données patient). *Même piège que `SESSION — Libre (LA)` de la cascade.*
> - **Texte patient dans `02 — Extraire données message`** : le nœud `LA-IN — Lire réponse Oui/Non` lit le message via `$('02 — Extraire données message').first().json.message_brut` (même source que `7-LA — Lire réponse Oui/Non`). Un premier jet lisant `$json.message`/`.text`/`.body` (champs inexistants ici) rendait le texte vide → détection ni OUI ni NON → fallback NON systématique (un « Oui » était traité comme refus).
> - **`SESSION — Libre (LA-IN)` DÉDIÉ — ne pas réutiliser `SESSION — Libre (LA)`** : le nœud de la cascade lit le téléphone depuis `$('7-LA — Lire réponse Oui/Non')`, nœud **hors du chemin d'exécution** de cette branche → référence cassée / téléphone vide → mauvaise session (ou aucune) remise à `libre`. Le nœud LA-IN lit depuis `$('LA-IN — Lire réponse Oui/Non').first()`, seul point commun aux chemins OUI et NON.
> - **`date_demande` triable pour un FIFO robuste** : format `yyyy-MM-dd HH:mm` (et non `$now.toISO()` brut), en heure Maroc forcée via `.setZone('Africa/Casablanca')`. Se trie alphabétiquement = chronologiquement, cohérent avec la lecture FIFO de la cascade.
> - **`medecin_souhaite` vide au pilote** : `15-PRD` ne produit pas ce champ (un patient en flux prise de RDV n'a pas précisé de praticien) → la colonne reste vide à l'inscription. **Voulu** (FIFO pur). Le format du contexte le conserve pour stabilité, mais il ne se remplit pas tant qu'on est en mono-médecin / pilote.
> - **Câblage `LA-IN — Inscrire (append)` via `$json`** : ce nœud lit `$json.telephone`/`.prenom`/`.medecin_souhaite` sans référence explicite — ça marche **parce qu'il est placé juste après `LA-IN-IF2` (un IF, qui propage `$json` intact depuis le nœud Code)**. Dépendance de position à connaître (contrairement à `mode_ecriture_final`, référencé explicitement).

**✅ Report / Rescheduling (VALIDÉE en test end-to-end — v9) :**

Déclenchée par l'intention `report` (sortie 2 du nœud `10 — Router selon intention`).

```
[10 — Router] (report)
   → REPORT — Chercher RDV existant (filtre telephone)
   → REPORT — RDV trouvé ?
        NON → ANNUL — Aucun RDV trouvé (réutilise le node "aucun RDV" de la branche annulation)
        OUI → REPORT — Récupérer créneaux libres (Calendar Get Many)
            → REPORT — Calculer nouveaux créneaux (Code)
              (EXCLUT l'ancien créneau via rdv.event_id pour ne pas le proposer comme "libre")
            → REPORT — Proposer nouveaux créneaux (WhatsApp, 3 créneaux)
            → SESSION — Écrire attente_choix_report (état = attente_choix_report)

[Réponse du patient : 1/2/3]
   → 07 — Router (état attente_choix_report)
   → 7-ACR2 — Valider choix report (Code : parse le choix + relit ctx.creneaux depuis la session)
   → 7-ACR2-IF — Choix valide ? (teste $json.valide, booléen calculé par le Code node)
        INVALIDE → 7-ACR — Choix invalide, renvoyer menu (node partagé avec la prise de RDV)
        VALIDE → 7-ACR2-CAL — Modifier RDV Calendar (operation=update, MÊME event_id)
            → 7-ACR2-LIRE — Relire RDV (nb_annulations)   ← NOUVEAU v9
              (Get Row patients_rdv, filtre telephone, returnFirstMatch
               — relit nb_annulations dans l'exécution courante, source de vérité)
            → 7-ACR2-SHT — Mettre à jour RDV (Sheets update, match telephone, 8 colonnes mappées)
            → 7-ACR-HIST — Logger report
            → 7-ACR2-WA — Confirmer report au patient (multilingue)
            → 7-ACR2-NOTIF — Notifier secrétaire report
            → SESSION — Libre (REPORT)
```

> **Points clés de la branche Report (v9) :**
> - **Modification, pas recréation** : `7-ACR2-CAL` fait un `update` Calendar sur le **même `event_id`** que le RDV d'origine — l'historique de l'événement est préservé, pas de doublon agenda.
> - **Exclusion de l'ancien créneau** : `REPORT — Calculer nouveaux créneaux` lit `rdv.event_id` et l'exclut des conflits, sinon le créneau actuel du patient apparaîtrait comme "libre" et serait reproposé.
> - **`nb_annulations` préservé via relecture sheet** *(v9)* : la branche Report se déroule en **deux exécutions séparées** (le patient écrit "reporter", puis répond "1/2/3" — chaque message redéclenche le Trigger). Le contexte session passé d'une exécution à l'autre ne contient PAS `nb_annulations`, et `$('REPORT — Chercher RDV existant')` est **inaccessible** dans la 2ᵉ exécution (cross-exécution, pas seulement cross-branche). Solution retenue : ajouter `7-ACR2-LIRE` qui relit `patients_rdv` dans l'exécution courante juste avant l'`update`, et recopie `nb_annulations` **sans l'incrémenter** (un report n'est pas une annulation). Mapping `7-ACR2-SHT` désormais complet : 8/8 colonnes.
> - **Node `7-ACR — Choix invalide` mutualisé** avec la branche prise de RDV (les deux branches IF "Choix valide ?" pointent dessus en sortie `false`).
> - **`7-ACR2-IF` en `typeValidation: none`** (et non `strict`) : acceptable ici car il teste `$json.valide`, un booléen produit par le Code node `7-ACR2`, et non une valeur brute issue de Google Sheets.

---

### Structure Google Sheets (3 onglets obligatoires)

**Onglet `patients_rdv` :**
```
telephone | prenom | medecin | date_rdv | heure_rdv | statut | event_id | nb_annulations
```

> 📝 La colonne s'appelle **`medecin`** (médecin assigné — fait accompli sur un RDV confirmé).

**Onglet `liste_attente` :**
```
telephone | prenom | medecin_souhaite | disponibilites | date_demande | statut | mode_ecriture_final | date_traitement
```

> 📝 La colonne s'appelle **`medecin_souhaite`** (médecin demandé par le patient — préférence, pas encore satisfaite). **Ne pas confondre avec `medecin`** dans `patients_rdv`. Le code des Code nodes doit utiliser le nom exact selon la sheet lue.

> ⚠️ La colonne `mode_ecriture_final` doit être présente dans `liste_attente`. **Depuis v17**, elle est **remplie automatiquement** à l'inscription (le patient a écrit → sa langue est connue → `LA-IN — Inscrire (append)` écrit `mode_ecriture_final` depuis `$('LA-IN — Lire réponse Oui/Non').first()`). Une saisie manuelle reste possible pour les lignes de test. Valeurs valides : `darija_latin`, `darija_arabe`, `arabe_fussha`, `francais_correct`, `francais_approximatif`, `mixte`, `inconnu`. Si vide, le workflow tombe sur le fallback `inconnu` (français propre formaté). *Note : la valeur écrite par l'inscription v17 est déjà projetée (`francais_correct`/`arabe_fussha`) — voir l'angle mort résiduel sur les saisies manuelles en dettes actives.*
>
> ⚠️ **Colonne `statut` (nouvelle v7)** : valeurs `en_attente` (par défaut à l'inscription) / `traité` (mis à jour quand le patient a accepté ou refusé un créneau libéré). Le workflow filtre `statut=en_attente` partout pour ne pas re-notifier un patient déjà traité. Toute ligne existante non renseignée doit être initialisée manuellement à `en_attente`.

**Onglet `sessions` :**
```
telephone | etat | contexte | derniere_activite
```

> ⚠️ Les numéros de téléphone sont stockés avec l'indicatif pays sans le 0 initial (ex: `212661234567`).
> Le champ `contexte` contient un JSON stringifié incluant `mode_ecriture_final` (profil linguistique) et, pour la liste d'attente, les champs du créneau libéré (`date_rdv`, `heure_rdv`, `medecin`, `event_id`, `prenom`) — **à la racine** du contexte.

---

## 5. REMPLACEMENTS À FAIRE DANS LE JSON

### Configuration centralisée (v10) — nœud `00-CONFIG — Variables cabinet`

Depuis v10, les variables spécifiques au cabinet sont centralisées dans **un seul nœud Set** en tête de flux. Pour un nouveau cabinet, ouvrir `00-CONFIG — Variables cabinet` et modifier les valeurs :

| Variable CONFIG | Valeur actuelle (pilote Badrour) | Câblée dans le workflow ? |
|---|---|---|
| `PROD_NUMERO_SECRETAIRE` | `212604152155` | ✅ câblée — notifs secrétaire |
| `PROD_NOM_MEDECIN` | `Ch. BADROUR` | ✅ câblée — fallback + nœuds sans variante bilingue (25 occ.) |
| `PROD_NOM_MEDECIN_FR` | `Dr. Ch. BADROUR` | ✅ **(v28)** câblée — nœuds patient FR via `projeter()` (6 occ.) |
| `PROD_NOM_MEDECIN_AR` | `د. شيماء بدرور` | ✅ **(v28)** câblée — nœuds patient AR via `projeter()` (6 occ.) |
| `PROD_PHONE_NUMBER_ID` | `998215733371244` | ✅ câblée — ~29 nœuds WhatsApp Send |
| `PROD_CALENDAR_ID` | `24c90eaf…@group.calendar.google.com` | ⏳ définie, déployée par Find & Replace (×8) |
| `PROD_SHEET_ID` | `1PXikPWXPhrH7QyYMZ6n988kItBLuuOcK3owFNs9uwno` | ⏳ définie, déployée par Find & Replace (×44) |
| `PROD_NOM_CABINET` | `Centre dentaire Dr. X` | ⏳ définie, pas câblée *(placeholder — à renseigner)* |
| `FMT_DATE_PATIENT` | *(helper de formatage de date)* | ⏳ définie dans `00-CONFIG`, usage à généraliser |

> **🆕 (v28) Nom du médecin — 3 variables + `projeter()`** : le nom s'affiche désormais dans la langue du patient. Les nœuds de confirmation patient appellent `projeter(mode_ecriture_final)` (retourne `'ar'` pour `arabe_fussha`/`darija_arabe`, sinon `'fr'`) puis choisissent `PROD_NOM_MEDECIN_AR` ou `PROD_NOM_MEDECIN_FR` (fallback `PROD_NOM_MEDECIN`). Le code ajoute `Dr ` si absent **et** si le nom n'est pas déjà en script arabe (`/[\u0600-\u06FF]/`). Câblé dans `7-ACR-WA`, `7-ACA-WA — Notifier patient en attente`, `7-ACR2-WA`, `7-LA-WA — Notifier suivant` (+ `ANNUL — Demander confirmation`). **Pour un nouveau cabinet : renseigner les 3 variables** (le `_FR` sans préfixe `Dr` géré par le code, le `_AR` avec le nom arabe complet).

> **Déploiement multi-cabinet — 2 leviers** : (1) valeurs câblées → changer dans `00-CONFIG` uniquement ; (2) IDs Sheet/Calendar → **Find & Replace sur le JSON exporté** (les champs `__rl` cassent l'autocomplétion en expression — câblage volontairement évité). Les onglets Sheets sont en mode **« By name »** (35 nœuds : `sessions`/`patients_rdv`/`liste_attente`) → aucune re-sélection manuelle à l'import. Le `WhatsApp Trigger` n'a pas de Phone Number ID (dépend de la credential). Détail des externalisations passées : voir CHANGELOG.

### Anciens placeholders (rappel — désormais couverts par CONFIG ou en dur)

| Placeholder historique | Couverture v10 |
|---|---|
| `REMPLACER_NUMERO_SECRETAIRE` | → `PROD_NUMERO_SECRETAIRE` (CONFIG, câblé) |
| `REMPLACER_NOM_MEDECIN` | → `PROD_NOM_MEDECIN` (CONFIG, câblé) |
| `REMPLACER_CALENDAR_ID` | → `PROD_CALENDAR_ID` (CONFIG, **à câbler**) |
| `REMPLACER_GOOGLE_SHEET_ID` | → `PROD_SHEET_ID` (CONFIG, **à câbler**) |
| `REMPLACER_PHONE_NUMBER_ID` | Phone Number ID Meta — toujours en dur dans les nœuds WhatsApp Send (`998215733371244`) |

> ℹ️ Le workflow actuel contient des credentials et IDs configurés pour le **cabinet pilote (Dr Ch. BADROUR, Tanger)**. Les credentials Google/WhatsApp et les IDs Sheet/Calendar sont à remplacer pour chaque nouveau déploiement client.

---

## 6. CREDENTIALS À CONFIGURER DANS N8N

| Credential | Type n8n | Nœuds concernés |
|---|---|---|
| OpenAI API Key | OpenAI API | `08 — Interpréteur intention (GPT-4o)` |
| Google Sheets | Google Sheets OAuth2 | Tous les nœuds Sheets |
| Google Calendar | Google Calendar OAuth2 | Tous les nœuds Calendar |
| WhatsApp Business (Send) | WhatsApp Business Cloud | Nœuds WhatsApp Send |
| WhatsApp Business (Trigger) | WhatsApp Business Cloud | `WhatsApp Trigger` |

---

## 7. PARAMÈTRES MÉTIER À ADAPTER

Dans les nœuds `15-PRD — Calculer créneaux disponibles` et `REPORT — Calculer nouveaux créneaux` :

```javascript
// Horaires de travail (à adapter au cabinet)
const horairesTravail = [
  '09:00','09:30','10:00','10:30','11:00','11:30',
  '14:00','14:30','15:00','15:30','16:00','16:30'
];

// Jours ouvrés (1=Lun, 2=Mar, 3=Mer, 4=Jeu, 5=Ven, 6=Sam)
const joursOuvres = [1, 2, 3, 4, 5];

// Durée consultation : modifier d.setMinutes(d.getMinutes() + 30)

// (v17) Fenêtre de recherche des créneaux = seuil de bascule en liste d'attente
const FENETRE_RECHERCHE_JOURS = 14; // 0 créneau dans cette fenêtre → inscription LA
```

> ⚠️ **(v17) Couplage à respecter** : si on change `FENETRE_RECHERCHE_JOURS` dans `15-PRD`, il faut ajuster `timeMax` du nœud `14-PRD — Récupérer créneaux Google Calendar` en conséquence (`timeMax` ≥ `FENETRE_RECHERCHE_JOURS`, actuellement 15 j pour 14 j de scan). Sinon zone aveugle → créneaux occupés vus comme libres.

Dans le nœud `CRON — Filtrer sessions expirées` :
```javascript
const EXPIRATION_MS = 30 * 60 * 1000; // 30 minutes
```

> ⚠️ **(v17) Timezone du workflow** : réglé sur `Africa/Casablanca` dans les Settings n8n (⋯ → Settings → Timezone). Force tous les `$now`/`new Date()` sans offset sur l'heure Maroc, indépendamment du fuseau serveur VPS. **Échéance dure** : le Maroc repasse à GMT+0 le 20/09/2026 — le nom de zone IANA suit automatiquement, mais les offsets `+01:00` **en dur** dans `15-PRD` (×12) devront être convertis avant cette date (voir dettes actives).

---
## 8. DÉCISIONS D'ARCHITECTURE ET LEURS RAISONS

| Décision | Raison |
|---|---|
| **WhatsApp Trigger** au lieu de Webhook | Plus simple, authentification automatique |
| **WhatsApp Send Message natif** au lieu de HTTP Request | Plus lisible, maintenu par n8n, credentials unifiés |
| **Google Sheets comme gestionnaire de sessions** | Visible en temps réel pour le debug, modifiable manuellement |
| **GPT-4o temperature 0.1** | Réponses stables et prévisibles — JSON fiable |
| **JSON stringifié dans le champ contexte** | Permet de stocker n'importe quelle structure dans une colonne Sheets |
| **appendOrUpdate sur les sessions** | Gère création et mise à jour en un seul nœud, évite les doublons |
| **event_id sauvegardé dans le contexte session** | Permet modifier/supprimer le bon événement Calendar sans re-cherche |
| **Répondre `libre` plutôt que supprimer la ligne** | Plus rapide, évite les doublons lors du prochain contact |
| **SWITCH mode_ecriture avant le router intention** | Profil linguistique établi une seule fois, propagé via contexte session pour toute la conversation |
| **`mode_ecriture` dans `contexte` JSON session** | Permet de répondre dans la langue du patient même dans les branches sans GPT (menu, créneaux) |
| **Menu numéroté sans GPT pour `attente_menu_principal`** | Zéro coût API, réponse instantanée pour les intentions ambiguës |
| **Nœud JS `7-MENU — Lire choix menu`** | Interprétation déterministe (1/2/3/4) — plus fiable que GPT pour des chiffres simples |
| **Mode `arabe_fussha` ajouté** *(v4)* | Couvrir les patients qui écrivent en arabe standard et non en darija — registre soutenu attendu |
| **Re-vérification Calendar avant d'attribuer un créneau libéré** *(v4)* | Évite la double-attribution si le créneau a été repris entre la notification et la réponse du patient en attente |
| **Garde-fou `throw` si contexte incomplet (7-LA)** *(v4)* | Échec propre plutôt que création d'un RDV à une date invalide silencieuse |
| **`SESSION — Écrire attente_liste_attente` recopie `$json.contexte` tel quel** *(v5)* | Évite de perdre `mode_ecriture_final` lors de la propagation (le node `7-ACA-CTX` est la source unique de vérité du contexte) |
| **Parser oui/non `7-LA — Lire réponse Oui/Non` avec 20+ variantes + découpage en mots** *(v5)* | Reconnaître `yes`, `d'accord`, `parfait`, `impossible`, etc. — un patient répondant naturellement ne doit pas tomber en `inconnu` (qui est traité comme refus) |
| **`String(...)` autour de `telephone` dans tous les nodes WA de la branche LA** *(v5)* | Google Sheets renvoie parfois `telephone` en type `number` → erreur `phoneNumber.replace is not a function` côté node WhatsApp |
| **`mode_ecriture_final` dans la sheet `liste_attente`** *(v5)* | Permet d'envoyer la confirmation dans la langue du patient en attente même si lui n'a pas encore interagi avec le bot (saisie manuelle pour l'instant, automatique plus tard) |
| **Format de confirmation LA aligné sur le RDV ordinaire** *(v5)* | Expérience patient cohérente — même structure visuelle (emojis, date formatée, médecin), peu importe l'origine du RDV |
| **Notifications multilingues partout (annulation, liste d'attente, demande de prénom)** *(v6)* | Le `mode_ecriture_final` doit être lu et appliqué dans **tous** les nodes WhatsApp Send — pas de hardcode darija ou français |
| **Mapping exhaustif des colonnes en `update` Google Sheets (7-ACA-SHT)** *(v6)* | Le node `update` n'effectue PAS de patch partiel : les colonnes non mappées peuvent être vidées. Toujours mapper TOUTES les colonnes, en relisant les valeurs inchangées depuis le node de lecture amont (`ANNUL — Chercher RDV existant`) |
| **`prenom` lu depuis `11-PRD` dans `15-PRD`** *(v6)* | Pour un patient déjà connu (notamment avec RDV annulé) qui repasse une demande, `Edit Fields` n'a pas tourné et `ctx.prenom` est vide — la sheet `patients_rdv` est la seule source de vérité fiable. Si on ne le récupère pas là, `prenom` arrive vide jusqu'à l'écriture et écrase la cellule |
| **`nb_annulations` incrémenté à chaque annulation confirmée** *(v6)* | Permet le ciblage anti-no-show futur : identifier les patients à risque récidiviste, ajuster le ton des rappels |
| **Message d'annulation sobre, sans CTA "Tapez OUI"** *(v6)* | Le patient annule un RDV médical — éviter le ton commercial. Garder une porte ouverte ("Nous restons à votre disposition") sans demander d'action numérique qui n'est pas câblée dans le flux. Souhait de santé adapté au mode d'écriture |
| **Bloc anti-doublon RDV (`11a`/`11b`/`11c`)** *(v8)* | Empêche un patient ayant déjà un RDV `confirmé` d'en créer un second par inadvertance. Le double filtre `telephone + statut=confirmé` garantit qu'un RDV annulé n'empêche PAS une nouvelle prise. Message de blocage multilingue orientant vers *reporter*/*annuler* — meilleure UX qu'un agenda pollué de doublons |
| **`7-ACA-FLAG` (Code) remplace `7-ACA-IF2` (branche IF)** *(v8)* | La classification tardive/normale devient une donnée (`type_annulation`) qui voyage dans `$json`, lisible par tous les nœuds aval sans re-calcul ni référence cross-branche. Plus robuste qu'un IF qui dédoublait la suite du flux en deux chemins quasi-identiques |
| **Notification annulation UNIQUE (`7-ACA-NOTIF — Résumé`)** *(v8)* | Fusionne les anciens `Alerte TARDIVE` / `Notification normale` en un seul message à la secrétaire. Agrège RDV + drapeau tardif + état liste d'attente via cascades `try/catch`. Moins de nœuds, message secrétaire plus complet, un seul point de maintenance |
| **Liste d'attente consultée AVANT la notif secrétaire** *(v8)* | Permet à la notification consolidée d'indiquer dans le même message si un patient en attente a été notifié ou si le créneau est orphelin. La secrétaire reçoit une vue d'ensemble en une fois |
| **Report = `update` Calendar sur le même `event_id`** *(v8)* | Préserve l'historique de l'événement agenda et évite un doublon (création + suppression). Exclusion explicite de l'ancien créneau dans le calcul pour ne pas le reproposer comme libre |

---
## 9. WORKFLOWS RESTANTS À CONSTRUIRE

### ✅ Workflow d'inscription automatique en liste d'attente — FAIT (v17)
Anciennement priorité 0. **Implémenté et testé (chemin OUI)** : quand `15-PRD — Calculer créneaux disponibles` ne retourne aucun créneau sur 14 jours, le patient se voit proposer l'inscription (OUI/NON) et, s'il accepte, une ligne est écrite dans `liste_attente` avec `statut=en_attente` + `mode_ecriture_final` renseignés automatiquement. La dette « sheet remplie manuellement » est **résolue**. Détail branche → §4. Détail debug → CHANGELOG v17. *Reste : valider le chemin NON.*

### Workflow anti no-show (priorité 1)
Se déclenche automatiquement à partir des RDV dans Google Sheets.
- **J-1 (24h avant)** : message WhatsApp au patient → il répond OUI/NON
- Si NON → annulation automatique + libération du créneau + liste d'attente
- Si pas de réponse → **J-0 (2h avant)** : deuxième rappel
- Si toujours pas de réponse → alerte secrétaire

### Workflow post-consultation (priorité 2)
- **2h après** la consultation : message de suivi personnalisé
- Proposition de prochain RDV si nécessaire
- **48h après** : enquête de satisfaction (étoiles 1 à 5)
- Résultat enregistré dans Google Sheets

---

## 10. MODÈLE COMMERCIAL

### Pricing suggéré pour les cabinets médicaux

| Offre | Contenu | Prix |
|---|---|---|
| **Projet pilote** | Installation + configuration + formation secrétaire | 5 000 – 8 000 MAD (prix réduit contre témoignage) |
| **Projet standard** | Workflow complet RDV + anti no-show + post-consultation | 8 000 – 15 000 MAD |
| **Maintenance mensuelle** | Monitoring + corrections + évolutions mineures | 500 – 1 500 MAD/mois |

### Argument de vente principal
> *"Combien de RDV non honorés avez-vous par semaine ?"*
Le médecin calcule lui-même sa perte. La conversation change instantanément.

### Argument différenciant (v7)
> *"On vous fournit aussi le dossier CNDP F112 pré-rempli et le DPA conforme à la Loi 09-08."*
La conformité réglementaire est positionnée comme un **service add-on à valeur ajoutée**, pas comme une obligation pénible. Très peu de concurrents la proposent — différenciateur fort sur le segment santé.

---

## 11. CONFORMITÉ LÉGALE MAROC

### Structure juridique entrepreneur
- **Auto-Entrepreneur** *(recommandé pour démarrer)*
  - Plafond annuel services : **200 000 MAD**
  - Impôt : **1% du chiffre d'affaires**
  - Inscription rapide, simple, peu coûteuse
  - Permet l'émission de factures conformes
- **SARL** *(à viser quand)*
  - Le revenu approche 150-180 000 MAD/an
  - Ou un client majeur exige une forme sociale
- ⚠️ **Retenue à la source** : seuil de **80 000 MAD/an/client** déclenche une obligation côté client → à anticiper dans le pricing
- ❌ Opération informelle **fortement déconseillée** : les cabinets médicaux ont besoin de factures pour passer les dépenses en comptabilité

### Loi 09-08 / CNDP — Protection des données personnelles

**Contexte** : application effective depuis début 2025, sanctions jusqu'à **300 000 MAD**. Les données patient sont des **données de santé** → régime renforcé.

**Architecture juridique** :
- Le **cabinet médical** = **responsable de traitement** (data controller)
- **L'entrepreneur (Khalid)** = **sous-traitant** (data processor)
- Les déclarations CNDP se font **par type de traitement**, pas par projet
- Un cabinet a donc **un seul** dossier CNDP pour son traitement RDV — quelle que soit la solution technique utilisée

**Formulaires CNDP à déposer (par le cabinet, accompagné de l'entrepreneur)** :
- **F112 — Autorisation préalable** : *requis* (et pas F211 = simple déclaration), car données de santé
  - Période d'instruction : **2 mois**
  - 14 sections à remplir : guide pré-rempli disponible
  - Pièces justificatives : 10 items checklist disponible
- **F118 — Déclaration de transfert international** : *requis*, car les données transitent par Meta (États-Unis/Irlande), Google (États-Unis/Irlande), OpenAI (États-Unis)

**DPA (Data Processing Agreement)** : produit, 14 articles + 2 annexes
- Couvre : obligations parties, sous-traitants ultérieurs autorisés (Meta/Google/OpenAI), mesures de sécurité, notification des incidents, droits des personnes concernées, transfert international, plafond de responsabilité, droit d'audit
- À signer entre le cabinet et l'entrepreneur **au moment de la signature commerciale**

**Notice d'information patient** (article 5 Loi 09-08) : produite, 2 formats
- Affiche A4 pour salle d'attente
- Message WhatsApp d'onboarding au premier contact patient

### Implication opérationnelle critique
> **Le dossier CNDP doit être déposé par le cabinet avant la mise en production réelle**. Le délai d'instruction (2 mois) est à anticiper. Stratégie recommandée : pendant les 2 mois, le système peut tourner en **mode test interne** au cabinet (patients-employés, secrétaire).

---

## 12. PLAN DE DÉPLOIEMENT PRODUCTION

### Tâches à compléter AVANT signature client (préparation proactive)

**Infrastructure entrepreneur :**
- [ ] Sécurisation du VPS Hostinger (firewall, fail2ban, certificat SSL valide)
- [ ] Custom domain pour n8n (`n8n.tondomaine.ma`) — meilleure crédibilité que `srv1019773.hstgr.cloud`
- [ ] Google Cloud Project dédié activité (séparation des tests perso)
- [ ] Templates n8n de workflows réutilisables, paramétrés (placeholders client clairement balisés)

**Meta WhatsApp Business :**
- [ ] **Meta Business Verification** — ⚠️ CRITIQUE — délai 3-7 jours ouvrés
  - Inscription Meta Business Suite avec entité légale (Auto-Entrepreneur OK)
  - Documents : registre commerce, ICE, RIB, justificatif domicile
  - **Sans verification = mode sandbox** = limitations sur numéros destinataires
- [ ] Création de l'App Meta dédiée (WhatsApp Business API)
- [ ] Soumission des **Message Templates** types (confirmation, rappel, slot libéré)
  - Délai validation Meta : 24-48h
  - À soumettre AVANT signature pour ne pas faire attendre le client

**Légal / Conformité :**
- [ ] Statut Auto-Entrepreneur enregistré
- [ ] DPA type prêt à signer (préparé)
- [ ] CGV type prêtes (à produire)
- [ ] Guide CNDP F112 / F118 prêt à transmettre au cabinet (préparé)

### Tâches APRÈS signature client (déploiement par cabinet)

- [ ] Achat carte SIM dédiée au cabinet (numéro de service)
- [ ] **Warm-up du numéro WhatsApp** (envoi progressif de messages humains pendant ~7-14 jours pour éviter le ban)
- [ ] Soumission du **Display Name** WhatsApp (nom affiché côté patient) à Meta — délai 1-3 jours
- [ ] Duplication du template n8n et personnalisation cabinet (nom médecin, horaires, IDs Sheets/Calendar)
- [ ] Création des credentials Google (Sheets + Calendar) dédiées au cabinet
- [ ] Configuration des Phone Number IDs et webhooks
- [ ] Formation de la secrétaire (présentation PPTX d'onboarding prête, 18 slides)
- [ ] Lancement progressif : tests internes 1 semaine → exposition graduelle aux vrais patients
- [ ] Dépôt du dossier CNDP F112 par le cabinet (l'entrepreneur fournit le DPA + données techniques)

### Critère bloquant — Business Verification
> **Ne JAMAIS signer un premier client sans avoir complété la Meta Business Verification au préalable.** Le délai 3-7 jours créerait un trou opérationnel après signature, dommageable pour la confiance. Anticiper.

---
---

## 13. PRINCIPES n8n & DEBUG (référence dédoublonnée)

> Fusionne les anciennes sections « Guide de debug » et « Bonnes pratiques ». Chaque principe apparaît une seule fois.

### Données & références de nœuds
- **Référencer les nœuds par nom** : `$('Nom du nœud').first().json.x`, pas `$json.x`, dès qu'on dépend d'un nœud précis — robuste aux insertions futures et aux changements de cardinalité.
- **`.first()` et non `.item`** après tout nœud qui change le nombre/l'ordre d'items (Sheets multi-lignes, branches IF). Indispensable pour les écritures de session.
- **Source de vérité par donnée** : `prenom` → sheet `patients_rdv` ; `mode_ecriture_final` → contexte session (ou sheet `liste_attente` pour un patient en attente) ; `event_id` → retour du nœud Google Calendar (champ `id`, jamais `event_id`). Ne jamais lire `$json.x` si le nœud amont ne charge pas cette donnée.
- **Cross-branche INTERDIT** : `$('Nœud')` ne marche que si le nœud est sur le chemin d'exécution courant. Pour partager entre branches d'un IF, passer par le contexte session. Erreur typique : `createNoConnectionError`.
- **Cross-exécution ≠ cross-branche** : une branche en deux temps (patient écrit → répond plus tard) perd l'accès aux nœuds de la 1ʳᵉ exécution. Seuls le contexte session OU une relecture sheet dans l'exécution courante sont fiables (cf. `7-ACR2-LIRE` en branche Report).
- **Cascade « safe » sur nœuds convergents** : tout nœud atteignable par plusieurs chemins amont doit lire ses sources via `try/catch` en cascade avec fallback final. Check : *« ce nœud peut-il être atteint sans que le nœud référencé soit exécuté ? »* → si oui, cascade obligatoire.

### Google Sheets
- **`update` ne fait PAS de patch partiel** : mapper TOUTES les colonnes (relire les inchangées depuis le nœud de lecture amont) sinon les colonnes non mappées sont vidées. Préférer `Append or Update` quand possible.
- **Sheet `sessions` : toujours `Append or Update` (match `telephone`)**, jamais `Append` seul (doublons). `patients_rdv`/`liste_attente` : `Append` OK (historique).
- **`Always Output Data`** activé sur les nœuds Sheets/Calendar qui peuvent retourner 0 item, sinon le workflow bloque silencieusement et la branche « cas vide » n'est jamais empruntée.
- **`Return only First Matching Row`** sur les sheets lues en file d'attente → FIFO + pas de multi-notifications concurrentes.
- **`String(telephone)`** systématique côté nœuds WhatsApp : Sheets renvoie parfois `telephone` en `number` → `phoneNumber.replace is not a function`.
- **Noms de colonnes** : `patients_rdv.medecin` (assigné) ≠ `liste_attente.medecin_souhaite` (demandé). Ouvrir la sheet avant d'écrire un Code node pour confirmer le nom exact.
- **Chaînes médecin strictement identiques** entre `patients_rdv.medecin` et `liste_attente.medecin_souhaite` (espaces, casse, ponctuation, caractères invisibles) pour que le filtre cascade LA matche.
- **IDs Sheet/Calendar** : le `documentId` est un resource locator (`__rl`), pas une string plate. Externalisation par **Find & Replace sur le JSON exporté** (pas d'expression — casse l'autocomplétion UI).

### IF & types
- **`Convert types where required`** activé sur tout IF testant une valeur issue de Sheets (souvent `number`), sinon `Wrong type: 'X' is a number but was expecting a string`. Complément défensif : `String($json.x || '')` (gère aussi `undefined`).
- **Opérateur `is not empty`, pas `exists`** quand on veut exclure `undefined`/vide (`exists` matche `undefined`).

### Expressions & Code nodes
- **Mode Expression `=` obligatoire** sur les champs dynamiques : sinon n8n écrit le texte brut `{{ $json.x }}` en sheet — bug silencieux révélé au routage suivant. (Cause du bug T2.8 `textBody` remis en Fixed.)
- **Commentaires `//` INTERDITS dans les expressions inline** (textBody) : l'aplatissement copier-coller les met sur une ligne et commente tout le code suivant. Utiliser `/* ... */` ou rien. Les `//` ne sont sûrs que dans les Code nodes.
- **`extendSyntax` casse sur les IIFE imbriquées en argument** : extraire tous les `$('Nœud').first()` en `let` en tête d'une IIFE racine unique (reads au top avec try/catch, logique en dessous).

### Calendar
- **event ID** dans le champ `id` (jamais `event_id`). `$json.event_id` hérite silencieusement de valeurs résiduelles.
- **Delete Calendar** retourne `success: true` même pour un ID inexistant — un mismatch d'ID est invisible sans comparaison explicite.
- **Timezone** : `start`/`end` doivent inclure l'offset `+01:00` (Maroc) sinon événement décalé d'une heure. ⚠️ Ces offsets en dur cassent au passage GMT+0 du 20/09/2026 — voir dettes actives.
- **(v17) Fenêtre de récupération `timeMax` ≥ fenêtre de scan `15-PRD`** : `14-PRD — Récupérer créneaux` (Get Many) ne remonte que les événements dans `[timeMin, timeMax]`. Si `timeMax` (récup) < `FENETRE_RECHERCHE_JOURS` (scan de `15-PRD`), les créneaux occupés dans l'écart sont vus comme **libres** (bug : LA ne se déclenche jamais pour un agenda plein « à moyen terme »). Réglé à 15 j de récup pour 14 j de scan. Ne pas modifier l'un sans l'autre.
- **(v17) Après un WhatsApp Send, `$json` = réponse Meta** (`messaging_product`/`contacts`/`messages[]`), PAS les données patient. Tout nœud placé après un Send doit référencer explicitement `$('NomDuNœudCode').first().json.x` pour retrouver `telephone`/`prenom`/`mode_ecriture_final`. (Cause de deux bugs v17 : `telephone` undefined dans `SESSION — Écrire attente_inscription_LA`.)

### Projection linguistique (7→2)
- 7 modes détectés en entrée → 2 modes de sortie (`francais_correct` / `arabe_fussha`). Projection centralisée dans `SET — Profil linguistique final`, propagée via le contexte session.
- **Source canonique du mode** pour un nœud WhatsApp : `SET — Profil linguistique final` en priorité, fallback contexte session, fallback parser. (Lire *uniquement* le contexte session échoue en entrée de branche où la session n'est pas encore active — c'est la cause du bug T2.8.)
- Tout nœud WhatsApp patient doit avoir les branches `francais_correct` ET `arabe_fussha`.

### Debug rapide — situations fréquentes
- **Patient bloqué en session** → onglet `sessions` → passer `etat` à `libre`.
- **Workflow ne reçoit rien** → workflow activé ? webhook Meta OK ? credentials WhatsApp OK ?
- **Mauvaise intention/mode détecté** → exécution → nœud `09 — Parser réponse IA` → lire `intention`/`mode_ecriture` ; affiner le prompt du nœud `08`.
- **Confirmation en mauvaise langue** → `mode_ecriture_final` présent dans le contexte session ? nœud WhatsApp lit-il la source canonique (pas seulement le contexte) ?
- **Patient en attente non notifié** → `7-ACA-IF3` (liste lue ?), `7-ACA-CTX` (écrit `attente_confirmation_liste_attente` + contexte ?), `07 — Router` (sortie vers `7-LA — Lire réponse Oui/Non` ?).
- **Cascade refus LA : P2 non notifié** → `7-LA — Chercher suivant LA` a bien `statut=en_attente` + `Return only First Matching Row` + `Always Output Data` ? IF `Suivant trouvé ?` en `is not empty` + `Convert types` ?
- **`nb_annulations` remis à zéro / plafonné à 1** ⚠️ *partiellement traité* : le fallback `|| 0` de `7-ACR-SET` (posé et vérifié 05/08) règle le cas **nouveau patient** (plus de `undefined`). Le nœud de relecture `7-ACR-LIRE` est en place. **Reste à re-valider en conditions réelles** que le compteur *s'incrémente* bien à chaque annulation successive et ne repart pas de zéro à la reprise (symptôme original T4.9) — à confirmer avant le Workflow 2 anti-no-show, qui consommera ce compteur.

---

## 14. DETTES TECHNIQUES ACTIVES

> Uniquement les dettes **ouvertes**. Les dettes résolues (v9→v14) sont archivées dans le CHANGELOG.

### ✅ Dettes fermées et vérifiées présentes dans le JSON courant (08/08/2026)

> Corrigées et confirmées dans `WhatsApp_Appointment_Automation.json`. Trace courte ; détail dans le CHANGELOG (PARTIE A + §28).

- ✅ **CRON chaque minute → 5 min** : `Schedule Trigger` en `rule.interval = [{ field: 'cronExpression', expression: '*/5 * * * *' }]`.
- ✅ **`7-ACR-SET` / `nb_annulations` sans fallback** : champ = `={{ $json.nb_annulations || 0 }}`. Maillon final de T4.9 posé.
- ✅ **`.item` résiduel dans `7-LA — Écrire patients_rdv`** : plus aucun `.item` dans le nœud.
- ✅ **BUG-CONV (`11c-WA` / `13-PRD`)** : lecture nue de `SET — Profil linguistique final` sur un chemin où il n'est pas exécuté (`7-MENU`/`7-RDVEX`) → réponse français par défaut. **Corrigé** par pattern cascade `SET → 7-MENU → 7-RDVEX → session 05` (try/catch, défaut `inconnu`), vérifié dans le `textBody` des deux nœuds.
- ✅ **`7-LA — Lire liste_attente` sans `alwaysOutputData`** : `aod = true` activé — plus de branche morte silencieuse sur le lookup `telephone`+`statut=en_attente` quand 0 résultat est légitime.

### 🟠 Dettes actives détectées à l'audit du 08/08/2026

- 🟠 **`CRON — Notifier patient session expirée` — `phoneNumberId` en dur** : le nœud envoie `998215733371244` littéral, alors que les **28 autres nœuds WhatsApp Send** utilisent `={{ $('00-CONFIG …').first().json.PROD_PHONE_NUMBER_ID }}`. La branche CRON part d'un **`Schedule Trigger` séparé qui ne traverse pas `00-CONFIG`** → `.first()` sur ce nœud n'est pas fiable dans le run CRON isolé. **Fix recommandé : insérer un mini-`Set` `CRON-CONFIG`** en tête de branche CRON redéfinissant `PROD_PHONE_NUMBER_ID` (et `PROD_NUMERO_SECRETAIRE` si un jour la notif secrétaire y arrive), puis référencer ce Set. **Bloquant multi-cabinet** (Phone Number ID propre à chaque cabinet).
- 🟠 **`13-PRD — Demander prénom` — credential divergent** : utilise `gd4yOWdWN1iwdMHX` (« …account-Send Message ») au lieu de `UlsIZcI7TK8xx8vd` (le reste du workflow, 28 nœuds). Marche tant que les deux tokens pointent la même WABA ; casse en multi-tenant ou si l'un expire. **Aligner sur `UlsIZcI7TK8xx8vd`.** Corriger au passage son unique `.item` résiduel (`$('02 — Extraire données message').item.json.telephone` → `.first()`).
- 🟠 **`PROD_PHONE_NUMBER_ID` — valeur `=998215733371244`** : le préfixe `=` parasite fait interpréter le champ **Fixed** comme une expression. « Marche par accident » (l'expression retourne le nombre) mais à nettoyer : valeur Fixed = `998215733371244` **sans** `=`.
- 🟡 **`7-ACA-SHT0 — Lire RDV patient` sans `alwaysOutputData`** : lecture critique sans `aod` → si le patient n'a aucun RDV, la branche peut mourir en silence. Activer `Always Output Data` par cohérence avec les autres lookups.

### 🟡 Résidus `.item` restants (hygiène, non bloquant)

- 🟡 **28 occurrences `.item` subsistent** dans le workflow (réduites de 34 → 28 depuis le dernier export). **Aucune n'est un bug dans le flux nominal** (1 patient = 1 item). Les cas à durcir en priorité restent les sources qui changent le compte/l'ordre :
  - `7-ACA-SHT — Marquer RDV annulé` → `$('7-ACA-IF — Confirme annulation ?').item` (×7 colonnes). IF → pairing préservé. **Sûr en pratique.**
  - `7-LA — Marquer LA traitée` → `$('7-LA — Lire liste_attente').item`. Source = lookup Sheets, ambigu si >1 ligne matche → mérite un `.first()` par précaution.
  - `13-PRD — Demander prénom` → `$('02 — Extraire données message').item` (voir dette credential ci-dessus).
  - *À durcir en `.first()`/`.last()` sur un nœud nommé avant toute introduction de traitement par lot. Priorité basse.*

### Dettes reportées des versions précédentes

- 🟡 **Angle mort `liste_attente.mode_ecriture_final` — RÉDUIT en v17** : la projection 7→2 n'est PAS appliquée à la *lecture* de cette sheet. **Mais l'inscription automatique écrit déjà une valeur projetée** (`francais_correct`/`arabe_fussha`, issue de `LA-IN — Lire réponse Oui/Non`) → les lignes créées par le workflow sont saines. Le risque ne subsiste que sur les **saisies manuelles** (une valeur `darija_arabe` tapée à la main déclencherait la branche darija au lieu de fusha). À traiter proprement : projeter à la lecture, ou restreindre la consigne de saisie manuelle à `francais_correct`/`arabe_fussha`. Testé par T6.3/T6.4.
- 🟠 **Offsets `+01:00` en dur — échéance 20/09/2026** *(v17)* : le Maroc quitte le GMT+1 permanent pour GMT+0 le 20/09/2026 (décret 2.26.530). **Recompté à l'audit v28 : 14 offsets `+01:00` en dur** répartis sur `15-PRD` (3×), `REPORT — Calculer nouveaux créneaux` (4×), `7-LA — Préparer événement` (4×), `7-ACR-CAL` (2×), `7-ACR2-CAL` (1×). Ces instants de créneaux seront alors décalés d'1h. À convertir en calculs basés sur `Africa/Casablanca` (nom de zone IANA → suit le basculement) AVANT cette date. Le timezone du workflow couvre les `$now`, PAS ces offsets inline. Non bloquant pour le pilote de juillet-août.
- 🟡 **Regex Oui/Non de `LA-IN — Lire réponse Oui/Non` moins complète que `7-LA`** *(v17)* : détection maison (`\b(oui|o|wah|wa|...)\b`) vs la liste de 20+ variantes éprouvée de `7-LA` (`ouiVariants`/`nonVariants`, découpage en mots). Risques mineurs : `\bwa\b` (mot darija courant) et `o` seul → faux positifs. À homogénéiser. Non bloquant (fallback = NON, sûr).
- 🟡 **Hypothèse cabinet mono-médecin** : clé d'unicité d'un patient en file = `telephone` seul. Pour du multi-médecin (phase 2), clé composite `telephone + medecin_souhaite`. *Note : le filtre `medecin_souhaite` est déjà en place sur `7-LA — Chercher suivant LA` — la cascade refus est déjà compatible multi-médecin.*
- 🟡 **Amélioration message T3.7** : un patient envoyant une image seule reçoit « messages vocaux non supportés » (mélange image/vocal). Message à préciser.
- ⚪ **Code mort darija/mixte** : branches `darija_*`/`mixte` inatteignables dans ~14 textBody (la projection ne produit que `francais_correct`/`arabe_fussha`). Inoffensif, hors chemin critique. Nettoyage prévu post-clôture.
- ⚪ **Formatage de date dupliqué** (`ANNUL — Demander confirmation`, `11c-WA`, `7-ACA-WA — Notifier patient en attente`, `7-LA-WA — Confirmer`, `7-LA-WA — Notifier suivant`) : recalcul inline. Sortie correcte — dette d'architecture. La variable `FMT_DATE_PATIENT` a été ajoutée à `00-CONFIG` (v28) mais **n'est pas encore généralisée** ; à câbler pour centraliser. *`eval` du helper bloqué par la sandbox n8n dans les expressions — passer par un Code node si centralisation.*
> *(`PROD_NOM_CABINET` placeholder `Centre dentaire Dr. X` — désormais listé en dette active 🟠 client-facing plus haut, à renseigner avant démo.)*

### ✅ Dettes clôturées depuis v17

- ✅ **Bug T2.8 — CLÔTURÉ (vérifié v28)** : `ANNUL — Demander confirmation annulation` a bien ses deux branches `francais_correct`/`arabe_fussha` et lit le mode depuis la source canonique. Plus de réponse français pour une annulation en darija.

---

## 15. RÉFÉRENCES RAPIDES

**Ressources & tests :**
- Fichier workflow : `WhatsApp_Appointment_Automation.json` (`name` interne cohérent, workflow `active`)
- Plan de test : `Plan_Test_Workflow1_v14.xlsx` (50 cas, 6 séries + Dashboard auto-agrégé)
- Historique versions + dettes résolues : `CHANGELOG_v7-v28_Workflow1.md`

**Infra :** n8n self-hosted v2.7.4 (Hostinger VPS KVM 1) · WhatsApp Business Cloud API (Phone Number ID `998215733371244`) · Google Sheets (doc `1PXikPWXPhrH7QyYMZ6n988kItBLuuOcK3owFNs9uwno` : `patients_rdv`/`liste_attente`/`sessions`) · Google Calendar (`DRV_Patients`) · GPT-4o temp 0.1 (parsers) / 0.7 (messages patients). Détail infra → `INFRA_NOTES_28Juillet26.md`.

**Maintenance base n8n (`.env`) :** Pruning des exécutions actif — `EXECUTIONS_DATA_PRUNE=true`, `MAX_AGE=336` (14 j), `PRUNE_MAX_COUNT=10000`, succès + erreurs conservés (`SAVE_ON_SUCCESS`/`SAVE_ON_ERROR=all`). Fuseau `GENERIC_TIMEZONE=Africa/Casablanca` aligné dans le `.env`. *Note migration : SQLite conserve l'espace disque même après pruning ; PostgreSQL recommandé avant le 3ᵉ cabinet.* ✅ **Le CRON tourne désormais toutes les 5 min (`*/5 * * * *`)** — le volume d'exécutions loggées est revenu à un niveau normal (≈288 exéc/j au lieu de ≈1 440).

**Config centrale :** `00-CONFIG — Variables cabinet` — `PROD_NUMERO_SECRETAIRE`, **`PROD_NOM_MEDECIN`=`Ch. BADROUR`, `PROD_NOM_MEDECIN_FR`=`Dr. Ch. BADROUR`, `PROD_NOM_MEDECIN_AR`=nom arabe complet**, `PROD_PHONE_NUMBER_ID` (câblés) ; `PROD_CALENDAR_ID`, `PROD_SHEET_ID`, `PROD_NOM_CABINET`, `FMT_DATE_PATIENT` (définis ; `FMT_DATE_PATIENT` non généralisé) ; IDs Sheet/Calendar déployés par Find & Replace.

---

*README post-v28 — dernière synchronisation le **08/08/2026** après re-audit complet du fichier `WhatsApp_Appointment_Automation.json` (136 nœuds ; intégrité : 0 réf cassée / 0 orphelin / 0 cible inexistante ; `active: true`). Changements de fond vs v17 (inchangés) : cabinet pilote **Dr Ch. BADROUR** (Tanger), **nom médecin bilingue** (`projeter()` + `PROD_NOM_MEDECIN_FR`/`_AR`), **branche ESCAPE** (token `00`), **sous-flux « choix RDV existant »** (`11c`→`7-RDVEX`, 1=report/2=annulation), **`7-ACA-IF4 — RDV déjà passé ?`** (skip cascade LA si créneau expiré), **`7-ACR-LIRE`** en prise de RDV, **bug T2.8 clôturé**. **✅ Fermées et vérifiées dans le fichier** : 3 dettes d'audit v28 (CRON `*/5 * * * *`, `nb_annulations || 0`, `.item` de `7-LA — Écrire`) + **BUG-CONV** (cascade `11c-WA`/`13-PRD`) + **`aod` sur `7-LA — Lire liste_attente`** + `name` interne renommé. 🟠 **Dettes actives restantes** : `phoneNumberId` en dur sur `CRON — Notifier patient session expirée`, credential divergent sur `13-PRD`, préfixe `=` parasite sur `PROD_PHONE_NUMBER_ID`, placeholder `PROD_NOM_CABINET`, `aod` manquant sur `7-ACA-SHT0` (§14). Historique complet → `CHANGELOG_v7-v28_Workflow1.md`. Détail infra → `INFRA_NOTES.md`.*

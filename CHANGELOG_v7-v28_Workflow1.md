# 📜 CHANGELOG ARCHIVÉ — Workflow 1 (Cabinet Médical RDV WhatsApp)

> **Usage :** Historique détaillé des changements de version (v7 → v17) et des dettes techniques déjà résolues. **Sorti du README principal pour l'alléger.** À consulter uniquement pour retracer *pourquoi* une décision a été prise ou *quand* un bug a été corrigé. Pour l'état courant du projet, voir le README principal.
>
> Fichier workflow courant : `WhatsApp_Appointment_Automation.json` — 136 nœuds (dont 2 `NoOp`, 0 sticky note). `name` interne cohérent, workflow `active`. Dernier re-audit : 05/08/2026.

---

## PARTIE A — DETTES TECHNIQUES RÉSOLUES (archive)

> Ces dettes ont toutes été fermées. Conservées ici pour mémoire ; retirées de la liste des dettes actives du README.

- ✅ **CRON exécuté chaque minute — RÉSOLU (post-v28, vérifié 05/08/2026)** : le `Schedule Trigger` de la branche de nettoyage de sessions déclenchait chaque minute (`interval: [{ field: 'minutes' }]` sans `minutesInterval`, ≈1 440 exéc/j à vide). Fix : bascule en expression cron `*/5 * * * *` → toutes les 5 min. *Non bloquant fonctionnellement (expiration 30 min correcte), mais gaspillait quota Sheets + gonflait les logs face au pruning 14 j.*
- ✅ **`7-ACR-SET` / `nb_annulations` sans fallback — RÉSOLU (post-v28, vérifié 05/08/2026)** : `={{ $json.nb_annulations }}` écrivait `undefined` pour un nouveau patient (item vide renvoyé par `7-ACR-LIRE` en `alwaysOutputData`). Fix : `={{ $json.nb_annulations || 0 }}`. *Maillon final de la correction T4.9 : le nœud de relecture `7-ACR-LIRE` avait été ajouté en v28, mais le fallback à l'écriture manquait. Reste à re-valider en réel que le compteur s'incrémente bien à chaque annulation successive (symptôme original T4.9).*
- ✅ **`.item` résiduel dans `7-LA — Écrire patients_rdv` — RÉSOLU (post-v28, vérifié 05/08/2026)** : le nœud lisait `$('7-LA-IF — Créneau libre ?').item.json.creneau.*` (+ `7-LA-CAL … .item`) — dette de longue date. Le nœud ne contient désormais plus aucun `.item`. *Rappel du principe : jamais `.item` après un nœud qui change le compte/l'ordre d'items (lookup Sheets, IF, Calendar) — préférer `.first()`/`.last()` sur un nœud nommé.*
- ✅ **BUG-CONV — lecture nue de `SET — Profil linguistique final` sur `11c-WA` / `13-PRD` — RÉSOLU (post-v28, vérifié 08/08/2026)** : les deux nœuds étaient atteignables via le chemin `7-MENU`/`7-RDVEX` (sous-flux « choix RDV existant ») où `SET — Profil linguistique final` **n'est jamais exécuté** → `$('SET — Profil linguistique final').first()` levait, et le patient tombait sur la branche français par défaut. **Fix : pattern cascade** implémenté dans le `textBody` des deux nœuds — chemin 1 `SET — Profil linguistique final` (message direct/GPT) → chemin 2 `7-MENU — Lire choix menu principal` → chemin 3 `7-RDVEX — Lire choix RDV existant` → filet de sécurité session `05 — Lire session patient` (`JSON.parse(contexte).mode_ecriture_final`), le tout en `try/catch` avec `mode = 'inconnu'` par défaut. *Même famille que T2.8 : tout nœud sur une convergence multi-chemins qui lit `SET — Profil linguistique final` en direct doit cascader vers les routeurs amont présents sur CHAQUE chemin, puis vers la session.*
- ✅ **`7-LA — Lire liste_attente` sans `alwaysOutputData` — RÉSOLU (post-v28, vérifié 08/08/2026)** : lookup `telephone` + `statut=en_attente` atteint sur la branche « le patient qui annule était-il lui-même en attente ? », où 0 résultat est légitime → branche morte silencieuse. Fix : `alwaysOutputData = true` activé sur le nœud (vérifié dans le JSON courant). *Les 8 lectures Sheets « 0-résultat légitime » ont désormais toutes `aod`.*
- ✅ **Sheet `liste_attente` remplie manuellement — RÉSOLU (v17)** : la LA n'avait qu'une *sortie* (cascade refus v7) ; l'*entrée* (inscription automatique) manquait — aucun nœud ne créait de ligne, la sheet était éditée à la main. **Nouvelle branche d'inscription (9 nœuds)** : détection 0 créneau sur 14 j (`LA-IN-IF-Créneaux dispo?`) → proposition WhatsApp multilingue (`LA-IN-WA Proposer inscription`) → état isolé `attente_confirmation_inscription_LA` → OUI/NON (`LA-IN — Lire réponse Oui/Non` + `LA-IN-IF2`) → APPEND FIFO (`LA-IN — Inscrire (append)`). Colonnes `statut=en_attente` et `mode_ecriture_final` désormais renseignées automatiquement. *Chemin OUI validé end-to-end en conditions réelles ; chemin NON à confirmer.* Détail des 5 bugs rencontrés et corrigés → synthèse v16→v17 ci-dessous.
- ✅ **Parser Oui/Non incomplet pour réponses courtes — RÉSOLU (v10)** : la cause racine n'était pas la liste de variantes mais le **traitement de `inconnu` comme refus implicite**. Fix : logique 3-way `oui / non / inconnu` (2ᵉ IF `7-LA-IF — Réponse = NON ?` + nœud `7-LA-WA — Redemander Oui/Non`). Réponse non comprise → re-sollicitation multilingue **sans modifier la session**, au lieu de perte silencieuse du créneau.
- ✅ **Numéro WhatsApp secrétaire en dur (×5) — RÉSOLU (v10)** : tous les nœuds de notification secrétaire lisent `PROD_NUMERO_SECRETAIRE` depuis `00-CONFIG — Variables cabinet`.
- ✅ **`7-ACR2-SHT` / `nb_annulations` écrasé — RÉSOLU (v9)** : mapping des 8 colonnes ; `nb_annulations` relu dans l'exécution courante via `7-ACR2-LIRE` (Get Row, returnFirstMatch) et recopié sans incrémentation. *Apprentissage : cross-exécution ≠ cross-branche — `REPORT — Chercher RDV existant` inaccessible dans la 2ᵉ exécution ; seule la relecture sheet est fiable.*
- ✅ **`7-ACR-SET` nom médecin en dur — RÉSOLU (v12)** : lit `PROD_NOM_MEDECIN` (`FZ Radi`) au lieu de `Dr. F. Z. Radi` en dur. Chaîne `patients_rdv.medecin` = `PROD_NOM_MEDECIN` = filtre `liste_attente.medecin_souhaite` → mismatch cascade LA éliminé. `PROD_NOM_MEDECIN` câblé dans 7 nœuds.
- ✅ **Phone Number ID Meta en dur (~22 nœuds) — RÉSOLU (v12)** : `998215733371244` externalisé dans `00-CONFIG` (`PROD_PHONE_NUMBER_ID`).
- ✅ **Sélecteurs d'onglet Sheets « From list » (gid caché) — RÉSOLU (v12)** : 35 sélecteurs passés en mode « By name » (`sessions`/`patients_rdv`/`liste_attente`). *Apprentissage : un `gid` numérique mis en cache par « From list » devient invalide d'un Google Sheet à l'autre ; « By name » résout par nom, stable en multi-tenant.*
- ✅ **Bugs node 08/09 — RÉSOLU (vérifié v13)** : (1) `arabe_fussha` présent dans le prompt du nœud 08 ; (2) le `catch` du nœud 09 écrit `{ intention:'autre', mode_ecriture:'inconnu', urgence:false, infos:{} }` — plus de `langue:'fr'`. JSON GPT malformé → item avec champ de routage valide.
- ✅ **Projection 7→2 modes de sortie — IMPLÉMENTÉE (v13)** : table de projection dans le champ `mode_ecriture_final` de `SET — Profil linguistique final`. `fr_correct`/`fr_approx` → `francais_correct` ; `arabe_fussha`/`darija_latin`/`darija_arabe`/`mixte` → `arabe_fussha` ; reste → `francais_correct`. + 4 branches `arabe_fussha` ajoutées aux textBody qui s'arrêtaient à `darija_arabe`.
- ✅ **Projection linguistique branche Report — RÉSOLUE (v14, MODIF1–3)** : angle mort de la convergence v13. (1) `REPORT — Calculer nouveaux créneaux` projette+propage `mode_ecriture_final` ; (2) `7-ACR2 — Valider choix report` lit+propage le mode ; (3) `REPORT — Proposer nouveaux créneaux` + `7-ACR2-WA` switchent sur `mode_ecriture_final` ; (4) `ANNUL — Aucun RDV trouvé` lit le mode. 17 nœuds patient dynamiques au total. *Apprentissage : « les N nœuds sont couverts » doit énumérer les branches une à une — la branche Report (2 exécutions n8n) avait été comptée couverte à tort.*
- ✅ **Bug `heure_local` dans `7-ACA-WA — Notifier patient en attente` — RÉSOLU (vérifié v14)** : `heure_local` (variable inexistante) → `ReferenceError` avalée par `catch {}` vide → date brute ISO. Corrigé. *Apprentissage : un `catch {}` vide masque les erreurs de formatage.*

### Non-dettes (choix assumés, pas à résorber)
- ℹ️ **IDs Calendar/Sheets non câblés en expression (v12)** : `PROD_CALENDAR_ID` (×7) et `PROD_SHEET_ID` (doc ID ×35) définis dans `00-CONFIG` mais **non câblés — volontairement**. Les champs `__rl` (resource locator) cassent l'autocomplétion UI en expression. Déploiement multi-cabinet via **Find & Replace sur le JSON exporté** + mode « By name » des onglets.
- ⚠️ **Dépendance d'accès à `00-CONFIG` via `.first()` (v10)** : accès fiable sur le chemin patient (`00-CONFIG` exécuté après `02`). **La branche CRON part d'un trigger séparé et ne passe PAS par `00-CONFIG`** — à re-câbler uniquement si un jour un nœud CRON doit afficher le médecin/notifier la secrétaire (actuellement CRON ne fait que remettre à `libre` + notifier le patient).

---

## PARTIE B — SYNTHÈSES DE VERSION (v7 → v15)

## 17. SYNTHÈSE DES CHANGEMENTS v7 → v8

| Domaine | v7 | v8 |
|---|---|---|
| **Nombre de nœuds** | ~100 | **119** |
| **Branche Report** | 🔄 squelette, non testée | ✅ **complète** (Calendar update même event_id, exclusion ancien créneau, Sheets, confirmation multilingue, notif secrétaire) — en cours de test |
| **Anti-doublon RDV** | ❌ absent | ✅ bloc `11a`/`11b`/`11c` (un patient avec RDV `confirmé` ne peut plus créer de doublon ; les RDV `annulé` n'empêchent pas une reprise) |
| **Classification annulation** | `7-ACA-IF2` (branche IF dédoublant le flux) | `7-ACA-FLAG` (Code → champ `type_annulation` dans `$json`) |
| **Notif secrétaire annulation** | 2 nœuds (`Alerte TARDIVE` / `Normale`) | 1 nœud unique `7-ACA-NOTIF — Résumé` (agrège RDV + tardif + liste d'attente) |
| **Ordre liste d'attente / notif** | notif puis LA | **LA consultée AVANT** la notif (la notif inclut l'état LA) |
| **Refus confirmation annulation** | libération sèche | renvoi vers `Réponse générique — Menu principal` |
| **Dettes nouvelles** | — | `7-ACR2-SHT` sans `nb_annulations` ; n° secrétaire en dur ×5 ; double `=` sur 2 nodes SESSION LA |
| **Corrections appliquées (depuis mémoire 8 juin)** | bug cascade refus P3 | ✅ les 3 nodes `SESSION — Libre (LA*)` utilisent `.first()` sur `7-LA — Lire réponse Oui/Non` ; `SESSION — Libre (LA refus)` s'exécute bien immédiatement après `7-LA-WA — Patient refuse`, avant la recherche cascade |

---

## 18. SYNTHÈSE DES CHANGEMENTS v8 → v9

| Domaine | v8 | v9 |
|---|---|---|
| **Nombre de nœuds** | 119 | **120** |
| **Branche Report** | ✅ complète — en cours de test | ✅ **validée end-to-end** (choix valide/invalide, Calendar même event_id, Sheets 8 colonnes, confirmation multilingue, notif secrétaire) |
| **Bloc anti-doublon RDV** | ✅ construit, non testé | ✅ **validé** (RDV confirmé → bloqué ; RDV annulé → reprise possible) |
| **Notif annulation consolidée** | ✅ construite, non testée | ✅ **validée** (drapeau tardif/normal + état LA dans un seul message) |
| **`7-ACR2-SHT` / `nb_annulations`** | ❌ dette : mapping 7/8, risque d'écrasement | ✅ **résolu** : nouveau nœud `7-ACR2-LIRE` relit la sheet dans l'exécution courante, recopie sans incrémenter, mapping 8/8 |
| **Double `=` sur SESSION LA** | ⚠️ à vérifier sur 2 nodes | ✅ **corrigé** (`={{ ... }}` propre + normalisation téléphone) |
| **Numéro secrétaire en dur ×5** | dette identifiée | ⏳ **toujours présent** — non bloquant, à externaliser en phase déploiement |
| **Apprentissage clé** | — | **Cross-exécution ≠ cross-branche** : une branche en deux temps (message → réponse) perd l'accès aux nœuds de la 1ʳᵉ exécution. Seules les sources lisibles dans l'exécution courante (contexte session OU relecture sheet) sont fiables |

---

## 19. SYNTHÈSE DES CHANGEMENTS v9 → v10

| Domaine | v9 | v10 |
|---|---|---|
| **Nombre de nœuds** | 120 | **117 fonctionnels + 6 sticky** (≈ +3 nœuds fonctionnels nets) |
| **Configuration cabinet** | éparpillée en dur dans les nœuds | ✅ **centralisée** dans `00-CONFIG — Variables cabinet` (Set en tête de flux, `includeOtherFields=true`) |
| **Numéro secrétaire ×5** | ⏳ en dur (`212604152155`) | ✅ **externalisé** → `PROD_NUMERO_SECRETAIRE` lu via `.first()` dans les 5 nœuds |
| **Nom médecin** | en dur dans les messages | ✅ **externalisé** → `PROD_NOM_MEDECIN` (6 nœuds WhatsApp) |
| **Calendar ID / Sheet ID / Nom cabinet** | en dur | ⏳ **variables CONFIG créées mais pas encore câblées** (nœuds Calendar/Sheets gardent l'ID en dur) |
| **Parser Oui/Non (réponses courtes)** | ⏳ dette : `inconnu` traité comme refus → perte de créneau | ✅ **résolu** : logique 3-way `oui/non/inconnu` (`7-LA-IF — Réponse = NON ?` + `7-LA-WA — Redemander Oui/Non`). Réponse ambiguë → re-sollicitation sans toucher à la session |
| **Apprentissage clé** | — | **Centraliser tôt, câbler progressivement** : un nœud Set de config en tête de flux, accessible via `$('00-CONFIG').first()`, suffit à externaliser les valeurs cabinet sans refonte. Le câblage peut être incrémental (numéro + médecin d'abord, IDs ensuite). **Et** : distinguer un refus explicite d'une non-compréhension — répondre `inconnu` ≠ refuser ; redemander vaut mieux que pénaliser le patient |

---

## 20. SYNTHÈSE DES CHANGEMENTS v10 → v11

| Domaine | v10 | v11 |
|---|---|---|
| **Nombre de nœuds** | 117 fonctionnels + 6 sticky | **117 fonctionnels + 6 sticky** (ajout `7-ACR-SET`, compensé par une fusion/retrait ailleurs — net stable) |
| **Écriture RDV (branche prise_rdv)** | `7-ACR-SHT` mappait les colonnes via références cross-nœud | ✅ **nœud Set d'hydratation `7-ACR-SET — Préparer données RDV`** intercalé entre `7-ACR-CAL` et `7-ACR-SHT` : toutes les colonnes (telephone, prenom, medecin, date, heure, statut, event_id, mode_ecriture_final) sont préparées dans un Set ; `7-ACR-SHT` lit `$json.*` **local** → plus robuste (pas de dépendance cross-nœud fragile) |
| **Valeurs `00-CONFIG`** | cabinet fictif (`Dr Alaoui`, calendar vide) | ✅ **valeurs réelles du cabinet pilote** : `PROD_NOM_MEDECIN = FZ Radi`, `PROD_CALENDAR_ID` renseigné |
| **Nom médecin dans `7-ACR-SET`** | n/a (nœud inexistant) | ⚠️ **NOUVELLE DETTE** : écrit `Dr. F. Z. Radi` **en dur** (mode Fixed), ≠ `PROD_NOM_MEDECIN` → risque mismatch sur la cascade liste d'attente. À câbler sur CONFIG |
| **Câblage CONFIG (Calendar/Sheets/nom cabinet)** | ⏳ non câblé | ⏳ **toujours non câblé** (7 nœuds Calendar en dur, doc ID Sheets en dur, `PROD_NOM_CABINET` placeholder, `PROD_SHEET_ID` vide) |
| **Bugs node 08/09** (langue vs mode_ecriture, fallback, arabe_fussha manquant) | identifiés | ⏳ **toujours présents** — non corrigés dans cette version |
| **Décision `mode_reponse` (fr/ar)** | actée (19 juin), non implémentée | ⏳ **non implémentée** — projection table absente du pipeline |
| **Apprentissage clé** | — | **Un Set d'hydratation dédié avant un nœud Sheets/Calendar** fiabilise l'écriture : préparer toutes les colonnes en `$json` local plutôt que de disperser des `$('Node').item.json.x` dans le mapping du Sheets. **Contrepartie à surveiller** : tout Set d'hydratation est un endroit où des valeurs cabinet peuvent re-rentrer en dur — toujours y câbler `00-CONFIG` |

---

## 21. SYNTHÈSE DES CHANGEMENTS v11 → v12

> **Fil rouge v12 : industrialisation du template multi-cabinet.** Aucune nouvelle fonctionnalité métier — la version transforme le workflow d'un projet mono-cabinet en un **template réutilisable** dont le déploiement chez un nouveau client ne demande plus que (1) un Find & Replace de 2 IDs et (2) une reconnexion des credentials. Les onglets et le nom du médecin ne sont plus des pièges à re-régler manuellement.

| Domaine | v11 | v12 |
|---|---|---|
| **Nombre de nœuds** | 117 fonctionnels + 6 sticky | **117 fonctionnels + 6 sticky** (net stable — aucune addition/suppression de nœud, uniquement des modifications de paramètres) |
| **Nom médecin dans `7-ACR-SET`** | ⚠️ dette : `Dr. F. Z. Radi` en dur, ≠ `PROD_NOM_MEDECIN` → risque mismatch cascade LA | ✅ **RÉSOLU** : câblé `={{ $('00-CONFIG').first().json.PROD_NOM_MEDECIN }}`. `patients_rdv.medecin` = `liste_attente.medecin_souhaite` = `FZ Radi` (chaîne unique). `PROD_NOM_MEDECIN` désormais sur **7 nœuds** |
| **Sélecteurs d'onglet Sheets** | « From list » (gid numérique mis en cache, invalide après import dans un autre doc) | ✅ **« By name »** sur les 35 nœuds (`sessions`/`patients_rdv`/`liste_attente`) → plus de re-sélection manuelle par cabinet |
| **Phone Number ID Meta** | en dur (`998215733371244`) dans chaque nœud WhatsApp Send | ✅ **externalisé** → `PROD_PHONE_NUMBER_ID` (CONFIG), lu via expression dans ~22 nœuds |
| **Prompt node 08 — `arabe_fussha`** | ⏳ absent des valeurs énumérées | ✅ **présent** (7 modes complets dans le prompt) |
| **Fallback node 09 (catch JSON invalide)** | ⏳ écrit `langue: 'fr'` | ⏳ **toujours `langue: 'fr'`** au lieu de `mode_ecriture: 'inconnu'` → item sans champ de routage pour le SWITCH. Bug isolé au JSON GPT malformé (rare). **À corriger** |
| **Câblage CONFIG Calendar/Sheets IDs** | ⏳ classé « dette à câbler » | ℹ️ **reclassé NON-DETTE** : choix assumé Find & Replace sur JSON exporté (champs `__rl` cassent l'autocomplétion en expression). Combiné au « By name », le déploiement ne touche plus aux onglets |
| **Projection `mode_reponse` (fr/ar)** | ⏳ non implémentée | ⏳ **toujours non implémentée** (`raw.count == 0`) — réponses switchent encore sur les 7 modes |
| **Apprentissage clé** | — | **Un template multi-tenant se juge à son coût de déploiement, pas à son élégance interne.** Deux pièges silencieux éliminés : le `gid` d'onglet mis en cache par « From list » (invalide d'un Google Sheet à l'autre) et le Phone Number ID dupliqué. Choix pragmatique inverse pour les IDs doc/calendar : ne PAS câbler en expression (l'UI `__rl` le pénalise) et assumer le Find & Replace. La bonne abstraction dépend de l'outil — ici, « By name » pour les onglets mais Find & Replace pour les IDs |

---

## 22. SYNTHÈSE DES CHANGEMENTS v12 → v13

> **Fil rouge v13 : convergence linguistique de sortie.** GPT-4o continue de détecter finement les 7 modes d'écriture du patient, mais la **réponse** ne se fait plus que dans 2 langues — français correct ou arabe fusha. Objectif : un comportement de sortie prévisible et tenable (2 registres à maintenir au lieu de 7), tout en gardant la finesse de détection en entrée. Aucune nouvelle fonctionnalité métier, aucun nœud ajouté/supprimé.

| Domaine | v12 | v13 |
|---|---|---|
| **Nombre de nœuds** | 117 fonctionnels + 6 sticky (123) | **123** (net stable — uniquement des modifications de contenu de nœuds) |
| **Modes de réponse au patient** | 7 (`mode_ecriture_final` portait les 7 valeurs ; chaque textBody switchait dessus) | **2** (`francais_correct` / `arabe_fussha`) — projection centralisée dans `SET — Profil linguistique final` |
| **Projection `mode_reponse` (fr/ar)** | ⏳ actée mais non implémentée | ✅ **IMPLÉMENTÉE** : IIFE de projection dans le champ `mode_ecriture_final`. Règle : fr/fr-approx → fr ; fussha/darija(latin+arabe)/mixte → fussha ; reste → fr |
| **Branches `arabe_fussha` dans les textBody** | 10 nœuds patient sur 14 en avaient une ; 4 s'arrêtaient à `darija_arabe` | ✅ **4 branches fusha ajoutées** : `7-ACR-WA — Confirmer au patient`, `7-ACR — Choix invalide`, `ANNUL — Demander confirmation annulation`, `Réponse générique — Menu principal`. Les 14 nœuds patient peuvent désormais répondre en fusha |
| **Bug node 09 (fallback `catch`)** | ⏳ listé « ouvert » (écrit `langue:'fr'`) | ✅ **vérifié déjà résolu** dans le JSON : écrit `mode_ecriture:'inconnu'`. README corrigé pour refléter l'état réel |
| **Risque multi-notification `7-ACA-SHT2`** | listé comme risque latent (en mémoire) | ✅ **vérifié résolu** : `returnFirstMatch:true` + filtre `statut=en_attente` bien présents (depuis v7) |
| **Code mort darija/mixte** | n/a (modes vivants) | ℹ️ branches `darija_*`/`mixte` désormais inatteignables dans les textBody — **conservées** (inoffensif), nettoyage prévu post-clôture |
| **Chemin liste d'attente (sheet manuelle)** | n/a | ⚠️ **non couvert** : `liste_attente.mode_ecriture_final` (saisie manuelle) peut contenir les 7 anciennes valeurs ; la projection n'est pas appliquée à la lecture de cette sheet. À traiter avant déploiement réel de la LA |
| **Apprentissage clé** | — | **Projeter à la source ≠ projeter partout.** Centraliser la projection dans `SET — Profil linguistique final` propage la valeur réduite via le contexte session sans toucher au routage — élégant. Mais une projection amont ne produit l'effet voulu que si chaque textBody aval sait *rendre* le mode cible : d'où l'audit préalable des 14 nœuds (4 manquaient de la branche fusha). Et une projection ne couvre que les chemins qui passent par elle : les entrées manuelles en sheet (liste d'attente) restent un angle mort à traiter séparément |

---

---

## 23. SYNTHÈSE DES CHANGEMENTS v13 → v14

> **Fil rouge v14 : clôture de la convergence linguistique.** v13 avait projeté les 7 modes d'entrée vers 2 modes de sortie (fr/fussha) et *déclaré* les 14 nœuds patient couverts — mais la **branche Report avait été manquée** dans la passe. v14 corrige cet angle mort (MODIF1–3 + 4ᵉ nœud), ce qui rend la projection 2-modes **réellement complète sur tout le chemin GPT**. Aucune nouvelle fonctionnalité métier, aucun nœud ajouté/supprimé — uniquement des corrections de contenu.

| Domaine | v13 | v14 |
|---|---|---|
| **Nombre de nœuds** | 123 | **123** (net stable — uniquement modifications de contenu) |
| **Projection linguistique branche Report** | ⚠️ **angle mort non détecté** : README v13 la croyait couverte, mais les 2 Code nodes ne propageaient pas `mode_ecriture_final` et 2 WhatsApp Send portaient du darija en dur | ✅ **RÉSOLUE (MODIF1–3)** : `REPORT — Calculer` projette+propage `mode_ecriture_final` ; `7-ACR2 — Valider` lit+propage le mode ; `REPORT — Proposer créneaux` + `7-ACR2-WA` switchent sur `mode_ecriture_final` (fr/fussha) |
| **4ᵉ nœud `ANNUL — Aucun RDV trouvé`** | ⏳ flaggé pour suivi post-MODIF | ✅ **résolu** : lit `mode_ecriture_final` (canonique → session → parser) |
| **Couverture projection 2-modes (chemin GPT)** | annoncée « 14 nœuds » mais Report exclue de fait | ✅ **17 nœuds patient dynamiques** — projection complète sur prise RDV / annulation / report / LA / menu. 7 nœuds en fr fixe restants = légitimes (6 notifs secrétaire + vocal + CRON patient) |
| **Bug `heure_local` (`7-ACA-WA`)** | présent (date au format brut ISO) | ✅ **vérifié résolu** dans le JSON (`heure_local` absent) |
| **Nœud `00-CONFIG — Variables cabinet`** | valeurs partielles / placeholder | ✅ **peuplé valeurs réelles cabinet pilote** : `PROD_NOM_MEDECIN=FZ Radi`, `PROD_CALENDAR_ID` réel, `PROD_NUMERO_SECRETAIRE`, `PROD_PHONE_NUMBER_ID`, `PROD_SHEET_ID`. `PROD_NOM_CABINET` reste placeholder `Centre dentaire Dr. X` |
| **Chemin liste d'attente (sheet manuelle)** | ⚠️ angle mort signalé | ⚠️ **toujours ouvert** — `liste_attente.mode_ecriture_final` (saisie manuelle) non projetée à la lecture. Désormais **seul** angle mort linguistique restant. À traiter avant déploiement réel de la LA |
| **Code mort darija/mixte** | branches inatteignables conservées dans ~14 textBody | ℹ️ **inchangé** — désormais aussi inatteignables dans les 2 textBody Report. Conservées (inoffensif), nettoyage post-clôture |
| **Apprentissage clé** | — | **« Couvert » se vérifie branche par branche, pas en agrégat.** v13 a conclu « 14 nœuds patient couverts » sans isoler la branche Report (qui s'exécute en 2 exécutions n8n séparées) → elle est restée en darija pendant une version entière, invisible tant qu'on ne testait pas un report en arabe. Une convergence ne se déclare close qu'après audit nœud par nœud sur le JSON, jamais sur un comptage global |

---

## 24. SYNTHÈSE DES CHANGEMENTS v14 → v15

> **Fil rouge v15 : stabilisation par le test réel, pas de nouvelle fonctionnalité.** Le Workflow 1 est fonctionnellement clos depuis v14 ; cette version corrige un bug et une erreur de manipulation trouvés pendant l'exécution de la campagne de test T1–T6 en conditions WhatsApp réelles. Aucun changement d'architecture.

| Domaine | v14 | v15 |
|---|---|---|
| **Bug T3.3** (réponse ambiguë en confirmation d'annulation) | Une réponse ni "oui" ni "non" faisait basculer la session vers `attente_menu_principal` (le patient sortait du flux d'annulation sans le comprendre) | ✅ **CORRIGÉ (Option A)** : la sortie FALSE de `7-ACA-IF — Reporter ?` route désormais vers le nœud `7-ACA-WA — Redemander Oui/Non` (message de re-sollicitation multilingue) au lieu de `Réponse générique — Menu principal`. Ce nœud est un terminal : **aucune écriture de session**, le patient reste en `attente_confirmation_annul` et son prochain message repasse par le même parser (même logique 3-way que la liste d'attente v10, cf. `7-LA-WA — Redemander Oui/Non`) |
| **Erreur de copier-coller `7-LA-WA — Redemander Oui/Non`** | Le `textBody` d'annulation (contexte `7-ACA`) avait été collé par erreur dans ce nœud, censé re-solliciter sur une offre de créneau liste d'attente | ✅ **CORRIGÉ** : `textBody` original restauré — lit le contexte depuis `7-LA — Lire réponse Oui/Non`, référence l'offre de créneau (pas l'annulation). Vérifié dans le JSON : `JSON.parse($('7-LA — Lire réponse Oui/Non').item.json.contexte \|\| '{}')` |
| **Mode du champ `textBody`** | Basculé par erreur en mode Fixed sur au moins un nœud (texte affiché tel quel, sans évaluation d'expression) | ✅ **CORRIGÉ** : repassé en mode Expression (fx) — confirmé par le préfixe `=` sur la valeur `textBody` dans le JSON |
| **Nombre de nœuds** | 123 annoncé (117 fonctionnels + 6 sticky — **chiffre non revérifié depuis v10**) | **130** vérifiés par script (**124 fonctionnels** + 6 sticky). Un seul nœud net ajouté cette session : `7-ACA-WA — Redemander Oui/Non`. L'écart avec le chiffre historique (117 → 124 hors ajout du jour) est un **passif de documentation**, pas un changement fonctionnel de cette session — voir note en tête de document |
| **Campagne de test T1–T6** | Non démarrée formellement | 🔄 **en cours** — bug T3.3 trouvé et corrigé pendant l'exécution ; clôture du Workflow 1 dépendante du passage complet des 6 séries (`Plan_Test_Workflow1_v14.xlsx`, **50 cas** — le Dashboard interne du fichier fait foi, pas le chiffre de 57 annoncé en Section 16/mémoire) |
| **Apprentissage clé** | — | **Le format « Redemander sans écrire de session »**, déjà validé sur la liste d'attente (v10), est maintenant le patron standard pour toute réponse ambiguë dans une confirmation binaire oui/non : ne jamais faire porter au routeur de session la responsabilité de deviner une intention absente — router vers une re-sollicitation neutre et laisser le parser retenter au message suivant. Et : une erreur de copier-coller entre deux nœuds `textBody` du même type (WhatsApp Send) est silencieuse à l'exécution — seul un test réel en conditions de langue/contexte différentes la révèle, d'où l'intérêt de la campagne T1–T6 avant clôture |

---

## 25. SYNTHÈSE DES CHANGEMENTS v15 → v16

> **Fil rouge v16 : clôture de la campagne de test T1–T6.** Le Workflow 1, fonctionnellement clos depuis v14, passe l'intégralité de la campagne en conditions WhatsApp réelles. Correction du dernier bug linguistique (T2.8) trouvé pendant l'exécution. Aucune nouvelle fonctionnalité.

| Domaine | v15 | v16 |
|---|---|---|
| **Bug T2.8** (annulation darija → réponse français au lieu de fusha) | ⏳ corrigé côté n8n, en attente de vérif JSON | ✅ **RÉSOLU et vérifié** : `SESSION — Écrire attente_confirmation_annul` lisait `mode_ecriture_final` depuis `05 — Lire session patient` (session *pré-calcul*, stale) → `inconnu` écrit → fallback français. Fix : lire depuis `SET — Profil linguistique final` (calcul de l'exécution courante). **Principe architectural** : pour ÉCRIRE l'état de session, toujours la valeur calculée dans l'exécution courante (nœud `SET`), jamais `05 — Lire session` (qui reflète l'exécution *précédente*). `05` = lecture seule de ce qu'une exécution antérieure a stocké |
| **Campagne T1–T6** | 🔄 en cours (43/50) | ✅ **close 50/50** — les 6 séries passées en conditions réelles |
| **Apprentissage clé** | — | **Écrire un état de session ≠ le relire.** Un `05 — Lire session` reflète l'exécution *précédente* ; l'utiliser comme source d'une *écriture* propage une valeur périmée. Toute écriture de session doit sourcer la valeur du nœud `SET` calculé dans l'exécution courante |

---

## 26. SYNTHÈSE DES CHANGEMENTS v16 → v17

> **Fil rouge v17 : fermeture du cycle liste d'attente par l'ajout de l'*entrée*.** La LA n'avait qu'une sortie (cascade refus, v7) ; l'inscription automatique manquait, la sheet était remplie à la main. v17 ajoute une branche de 9 nœuds qui *remplit* la file quand aucun créneau n'est libre. Première **nouvelle fonctionnalité métier** depuis plusieurs versions. Chemin OUI validé end-to-end ; 5 bugs rencontrés et corrigés en test (tous documentés ci-dessous).

| Domaine | v16 | v17 |
|---|---|---|
| **Nombre de nœuds** | 124 fonctionnels + 6 sticky (130) | **133 fonctionnels + 0 sticky** (+9 branche inscription LA ; les 6 sticky supprimés en v17) |
| **Cycle liste d'attente** | sortie seule (cascade refus) ; entrée **manuelle** | ✅ **cycle complet** : entrée (inscription auto) + sortie (cascade) |
| **Branche inscription LA** | ❌ absente | ✅ **9 nœuds** — 0 créneau/14 j → proposition OUI/NON → APPEND FIFO (chemin OUI testé) |
| **État de session** | `attente_confirmation_liste_attente` (cascade) | + `attente_confirmation_inscription_LA` (inscription, **isolé** — évite la collision de routage : accepter *d'entrer* dans la file ≠ accepter *un créneau proposé*) |
| **Fenêtre recherche créneaux** | `15-PRD` scan 60 j ; `14-PRD` récup 7 j (désalignés, mais LA inexistante donc invisible) | **alignés** : `15-PRD` = 14 j (`FENETRE_RECHERCHE_JOURS`), `14-PRD` `timeMax` = 15 j (marge 1 j) |
| **Timezone dates** | `$now.toISO()` → hérite du fuseau serveur VPS (GMT+2 en été) | Settings workflow = `Africa/Casablanca` ; `date_demande` via `$now.setZone('Africa/Casablanca').toFormat('yyyy-MM-dd HH:mm')` (Maroc, triable FIFO) |
| **Dette à échéance dure** | — | 🟠 offsets `+01:00` **en dur** dans `15-PRD` (×12) — à convertir avant le **20/09/2026** (retour Maroc à GMT+0, décret 2.26.530). Le timezone workflow couvre les `$now`, pas ces offsets inline |
| **Angle mort `liste_attente.mode_ecriture_final`** | 🟠 ouvert (saisie manuelle non projetée) | 🟡 **réduit** — l'inscription auto v17 écrit déjà une valeur projetée ; le risque ne subsiste que sur les saisies manuelles |
| **Apprentissages clés** | — | **(1) Après un nœud WhatsApp Send, `$json` = réponse Meta**, pas les données patient → tout nœud aval doit référencer explicitement le dernier nœud Code (`$('15-PRD …').first()`). **(2) Couplage `14-PRD`↔`15-PRD`** : la fenêtre de récupération Calendar doit couvrir ≥ la fenêtre de scan créneaux, sinon zone aveugle (créneaux occupés vus comme libres). **(3) Ne pas réutiliser un nœud `SESSION — Libre` d'une autre branche** : il référence un nœud hors du chemin d'exécution courant → nœud dédié par branche. **(4) La fenêtre de recherche EST le seuil de bascule LA** — un seul paramètre pilote deux comportements (proposer des créneaux / basculer en attente) ; à arbitrer avec le cabinet |

**Détail des 9 nœuds v17 (branche inscription LA) :**

| Nœud | Type | Rôle |
|---|---|---|
| `LA-IN-IF-Créneaux dispo?` | IF | Aiguille `($json.creneaux \|\| []).length > 0` : true→`16-PRD`, false→inscription |
| `LA-IN-WA Proposer inscription` | WhatsApp | Message multilingue « 0 créneau, OUI/NON ? » |
| `SESSION — Écrire attente_inscription_LA` | Sheets (AoU) | État isolé + contexte `{prenom, medecin_souhaite, mode_ecriture_final}` depuis `$('15-PRD …').first()` — terminal de 1ʳᵉ exécution |
| `LA-IN — Lire réponse Oui/Non` | Code | Parse OUI/NON (texte via `02 … message_brut`), réhydrate ctx, produit `$json.reponse` |
| `LA-IN-IF2 — Réponse = OUI ?` | IF | `$json.reponse == 'OUI'` |
| `LA-IN — Inscrire (append)` | Sheets (**Append**) | Ligne FIFO : `statut=en_attente`, `date_demande` (Maroc, triable), `mode_ecriture_final` |
| `LA-IN-WA — Confirmer inscription` | WhatsApp | « inscrit ✅ » multilingue |
| `LA-IN-WA — Refus poli` | WhatsApp | « pas de souci » multilingue |
| `SESSION — Libre (LA-IN)` | Sheets (AoU) | Retour `libre` — lit le téléphone depuis `$('LA-IN — Lire réponse Oui/Non').first()` (seul point commun aux chemins OUI/NON) |

**Câblage structurel (2 MOD sur code testé 50/50 — vérifiées dans le JSON) :**
- **MOD1** : `15-PRD` → `LA-IN-IF-Créneaux dispo?` (au lieu de `15-PRD` → `16-PRD` direct) ; sortie true (0) → `16-PRD`, sortie false (1) → `LA-IN-WA Proposer inscription`.
- **MOD2** : `07 — Router selon état session` — nouvelle règle (sortie 6) `etat == attente_confirmation_inscription_LA` → `LA-IN — Lire réponse Oui/Non` ; le fallback « Réponse générique — Menu principal » a été recâblé sur la sortie 7.

**Les 5 bugs rencontrés et corrigés pendant le développement de la branche (chronologie de test) :**
1. **Créneaux proposés malgré agenda plein à J+10** → `14-PRD timeMax`=7 j < scan 14 j (zone aveugle). Fix : `timeMax`=15 j.
2. **`telephone` undefined dans `SESSION — Écrire attente_inscription_LA`** → nœud placé après un WhatsApp Send (`$json`=réponse Meta). Fix : référencer `$('15-PRD …').first()` pour `telephone` ET `contexte`.
3. **« Oui » traité comme refus** → `LA-IN — Lire réponse Oui/Non` lisait le texte via `$json.message`/`.text`/`.body` (inexistants). Fix : `$('02 — Extraire données message').first().json.message_brut`.
4. **`date_demande` en GMT+2 + ISO brut** → `$now.toISO()` sur fuseau serveur. Fix : timezone workflow `Africa/Casablanca` + `$now.setZone(...).toFormat('yyyy-MM-dd HH:mm')`.
5. **`mode_ecriture_final` vide dans `liste_attente`** → colonne non mappée dans `LA-IN — Inscrire`. Fix : mapper `$('LA-IN — Lire réponse Oui/Non').first().json.mode_ecriture_final`. *(La donnée était correcte à la source — vérifié à la sortie du nœud D — seul le mapping du nœud d'écriture manquait.)*

---

## 27. SYNTHÈSE DES CHANGEMENTS v17 → v28

> Regroupe l'évolution du fichier entre `…v17Juillet26…` et `Cabinet_Medical_RDV_WhatsApp_v28Juillet27.json`. Établie à partir de l'audit JSON du 28/07/2026 (le `name` interne du fichier était encore `…v13Juillet27…_Change Fussha`, non renommé).

| Domaine | v17 | v28 |
|---|---|---|
| **Nombre de nœuds** | 133 fonctionnels | **136** (dont 2 `NoOp`) |
| **Cabinet pilote** | Mimosa / Dr F. Z. Radi (test) | **Dr Ch. BADROUR** (Tanger, `drbadrour.ma`) |
| **Nom médecin** | `PROD_NOM_MEDECIN` unique | **+ `PROD_NOM_MEDECIN_FR` + `PROD_NOM_MEDECIN_AR`** ; sélection via `projeter()` selon `mode_ecriture_final` |
| **Confirmation patient** | nom médecin fixe (FR) | **nom médecin dans la langue du patient** (fr/ar) + dédoublonnage `Dr` + détection script arabe `/[\u0600-\u06FF]/` |
| **Sortie de secours** | ❌ absente | ✅ **branche ESCAPE** : token `00` → session `libre` + notif reset (insérée entre `05` et `06`) |
| **Anti-doublon RDV** | `11c` → arrêt sec, session inchangée | ✅ **sous-flux « choix RDV existant »** : `11c` propose 1=reporter/2=annuler → état `attente_choix_rdv_existant` → `7-RDVEX` → réinjection dans `10 — Router` |
| **Annulation d'un RDV expiré** | déclenche la cascade LA (fausse notif) | ✅ **`7-ACA-IF4 — RDV déjà passé ?`** (`rdv_expire`) : si expiré → notif secrétaire directe, pas de cascade |
| **`nb_annulations` en prise de RDV** | écrit depuis contexte/`$json` | **`7-ACR-LIRE`** relit la sheet avant écriture (tentative de fix T4.9) — ⚠️ **fallback `|| 0` manquant dans `7-ACR-SET`** |
| **Refus confirmation annulation** | → menu principal | → **`7-ACA-IF — Reporter ?`** propose un report |
| **Helper date** | inline dans 5 nœuds | **`FMT_DATE_PATIENT`** ajouté à `00-CONFIG` (non encore généralisé) |
| **Bug T2.8** | en cours de clôture | ✅ **clôturé** (branches fr/ar présentes, source canonique lue) |

**Nouveaux nœuds v28 (par rapport à v17) :**

| Nœud | Type | Rôle |
|---|---|---|
| `ESCAPE — Token réservé 00 ?` | IF | Intercepte `message_brut === '00'` |
| `SESSION — Libre (ESC)` | Sheets (AoU) | Force `etat=libre` sur token `00` |
| `ESCAPE — Notifier reset` | WhatsApp | « session réinitialisée » |
| `SESSION — Écrire attente_choix_rdv_existant` | Sheets (AoU) | État isolé après blocage doublon |
| `7-RDVEX — Lire choix RDV existant` | Code | Mappe 1→report / 2→annulation (⚠️ inverse du menu principal) |
| `7-ACA-IF4 — RDV déjà passé ?` | IF | Aiguille sur `rdv_expire` |
| `7-ACR-LIRE — Relire nb_annulations` | Sheets (read, aod) | Relit le compteur avant écriture RDV |

**Dettes détectées à l'audit v28 (28/07) — ✅ TOUTES RÉSOLUES depuis (vérifié 05/08/2026) :**
1. ✅ **CRON chaque minute → 5 min** — `Schedule Trigger` désormais en `rule.interval = [{ field: 'cronExpression', expression: '*/5 * * * *' }]`. Passage de ≈1 440 à ≈288 exéc/j.
2. ✅ **`7-ACR-SET` / `nb_annulations` sans fallback** — champ corrigé en `={{ $json.nb_annulations || 0 }}`. Nouveau patient (item vide de `7-ACR-LIRE` en aod) écrit `0` au lieu de `undefined`. Maillon final de T4.9 posé.
3. ✅ **`.item` résiduel dans `7-LA — Écrire patients_rdv`** — le nœud ne contient **plus aucun `.item`** (ni sur `7-LA-IF`, ni sur `7-LA-CAL`). Corrigé.

**Points d'intégrité re-vérifiés (audit 05/08/2026) :** 0 référence `$('…')` cassée (33 réfs distinctes, toutes valides) · 0 connexion vers nœud inexistant · 0 orphelin · 0 cible de connexion inexistante · routage `07` complet · projection linguistique (SWITCH 7 sorties → convergence 2 modes en aval) intacte · bug T2.8 confirmé clôturé (cascade `modeFrais`/`modeSession`/`inconnu` présente dans `SESSION — Écrire attente_confirmation_annul`) · timezone workflow `Africa/Casablanca` OK · `name` interne renommé.

---

## 28. SYNTHÈSE DES CHANGEMENTS post-v28 (audit 08/08/2026)

> Nouvel export du JSON re-audité le **08/08/2026**. Toujours **136 nœuds** (dont 2 `NoOp`, 0 sticky), `active: true`, `name` interne `WhatsApp_Appointment_Automation`. Intégrité re-confirmée : **0 référence `$('…')` cassée, 0 connexion vers nœud inexistant, 0 orphelin, 0 cible manquante.** Deux dettes fermées + compteur `.item` réduit.

| Domaine | v28 (05/08) | post-v28 (08/08) |
|---|---|---|
| **BUG-CONV** (`11c-WA` / `13-PRD` lisant `SET — Profil linguistique final` en direct) | 🔴 dette bloquante ouverte | ✅ **RÉSOLU** — pattern cascade `SET → 7-MENU → 7-RDVEX → session 05`, try/catch, défaut `inconnu` (voir PARTIE A) |
| **`7-LA — Lire liste_attente` / `alwaysOutputData`** | 🟡 dette mineure ouverte | ✅ **RÉSOLU** — `aod = true` (voir PARTIE A) |
| **Résidus `.item`** | 34 (hors `7-LA — Écrire`) | **28** au total dans le fichier — homogénéisation partielle effectuée |
| **`CRON — Notifier patient session expirée` / `phoneNumberId`** | non relevé | 🟠 **DETTE ACTIVE** — `phoneNumberId` en dur `998215733371244` au lieu de `PROD_PHONE_NUMBER_ID` (28 autres nœuds Send utilisent la variable). CRON ne passe pas par `00-CONFIG` → à câbler autrement (voir note) |
| **`13-PRD — Demander prénom` / credential** | non relevé | 🟠 **DETTE ACTIVE** — credential `gd4yOWdWN1iwdMHX` (« …account-Send Message ») ≠ `UlsIZcI7TK8xx8vd` du reste du workflow (28 nœuds). + 1 `.item` résiduel : `$('02 — Extraire données message').item.json.telephone` |
| **`PROD_PHONE_NUMBER_ID` (valeur dans `00-CONFIG`)** | supposé propre | 🟠 **DETTE ACTIVE** — valeur = `=998215733371244` : le préfixe `=` parasite fait interpréter le champ Fixed comme expression. À nettoyer en `998215733371244` (sans `=`) |
| **`PROD_NOM_CABINET`** | placeholder connu | 🟠 **DETTE ACTIVE** — toujours `Centre dentaire Dr. X`. À renseigner avec le vrai libellé cabinet Badrour avant client-facing |
| **`7-ACA-SHT0 — Lire RDV patient` / `alwaysOutputData`** | non relevé | 🟡 **DETTE MINEURE** — lecture critique sans `aod` ; si le patient n'a aucun RDV, la branche peut mourir en silence. À activer par cohérence |

**Détail des dettes actives (état 08/08/2026) — par priorité :**

1. 🟠 **`CRON — Notifier patient session expirée` — `phoneNumberId` en dur.** Le nœud envoie `998215733371244` littéral. La branche CRON part d'un `Schedule Trigger` séparé et **ne traverse pas `00-CONFIG`**, donc `$('00-CONFIG …').first()` n'y est pas fiable par le chemin classique. Deux options : (a) référencer `00-CONFIG` malgré tout (n8n autorise l'accès à un nœud non-ancêtre s'il a été exécuté au moins une fois dans le run — **fragile ici**, run CRON isolé) ; (b) recommandé : ajouter un mini-`Set` `CRON-CONFIG` en tête de branche CRON qui redéfinit `PROD_PHONE_NUMBER_ID` (et `PROD_NUMERO_SECRETAIRE` si un jour la notif secrétaire y arrive). Bloquant au déploiement multi-cabinet : chaque cabinet a son propre Phone Number ID.
2. 🟠 **`13-PRD — Demander prénom` — credential divergent.** Fonctionne tant que les deux credentials pointent la même WABA, mais casse en multi-tenant / si l'un des deux tokens expire. Aligner sur `UlsIZcI7TK8xx8vd`. Corriger au passage le `.item` résiduel (→ `.first()` sur `02 — Extraire données message`).
3. 🟠 **`PROD_PHONE_NUMBER_ID` = `=998215733371244`.** Préfixe `=` en trop dans un champ Fixed → n8n l'évalue comme expression. Aujourd'hui « marche par accident » (l'expression `=998215733371244` retourne le nombre), mais à nettoyer : valeur Fixed = `998215733371244` **sans** `=`.
4. 🟠 **`PROD_NOM_CABINET` = `Centre dentaire Dr. X`.** Placeholder résiduel. À remplacer par le libellé réel du cabinet Badrour avant toute démo/mise en prod client-facing.
5. 🟡 **`7-ACA-SHT0 — Lire RDV patient` sans `alwaysOutputData`.** Lecture critique ; activer `aod` par cohérence avec les autres lookups (silent branch death si 0 RDV).
6. 🟡 **28 résidus `.item`** répartis dans le fichier (dont celui de `13-PRD` ci-dessus). Aucun n'est un bug en mono-patient. À durcir en `.first()`/`.last()` par hygiène avant toute introduction de batch. Priorité basse.

**Points d'intégrité re-vérifiés (audit 08/08/2026) :** 0 référence `$('…')` cassée · 0 connexion vers nœud inexistant · 0 orphelin · 0 cible manquante · CRON `Schedule Trigger` toujours en `*/5 * * * *` · `nb_annulations || 0` présent dans `7-ACR-SET` · T2.8 toujours clôturé · timezone `Africa/Casablanca` · cascade BUG-CONV vérifiée dans le `textBody` de `11c-WA` et `13-PRD`.

---

*Archive mise à jour le **08/08/2026**. **Nouveau au 08/08 : BUG-CONV (`11c-WA`/`13-PRD`) et `alwaysOutputData` sur `7-LA — Lire liste_attente` sont vérifiés RÉSOLUS dans `WhatsApp_Appointment_Automation.json`** et archivés en PARTIE A ; `.item` global réduit de 34 à 28. **Dettes actives résiduelles :** `phoneNumberId` en dur sur `CRON — Notifier patient session expirée`, credential divergent sur `13-PRD`, préfixe `=` parasite sur `PROD_PHONE_NUMBER_ID`, placeholder `PROD_NOM_CABINET`, `aod` manquant sur `7-ACA-SHT0`. Rappel v17 → v28 : cabinet Badrour, nom médecin bilingue (`projeter()`), branches ESCAPE / choix RDV existant / RDV expiré, `7-ACR-LIRE`, T2.8 clôturé. Note test : `Plan_Test_Workflow1_v14.xlsx` couvre encore la matrice v14–v17 ; les sous-flux v28 (ESCAPE, choix RDV existant, RDV expiré) + le chemin NON de l'inscription LA restent à ajouter au plan de test.*

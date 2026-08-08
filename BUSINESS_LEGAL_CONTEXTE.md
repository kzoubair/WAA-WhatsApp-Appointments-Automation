# 💼 BUSINESS & LÉGAL — Zellia / Automatisation Cabinets Médicaux Maroc

> **Usage :** Contexte économique, juridique et commercial du projet. Distinct des fichiers techniques (`README_PROJET_CONTEXTE.md` = état workflow, `CHANGELOG_v7-v28_Workflow1.md` = versions, `INFRA_NOTES.md` = serveur). À consulter pour toute décision de pricing, contractualisation, conformité CNDP, ou préparation de vente/démo. **Ne contient aucun détail technique de workflow** — pour ça, voir les 3 autres fichiers.
>
> ⚠️ **Statut du contenu :** ce fichier a été reconstitué à partir du contexte de projet accumulé. Les valeurs marquées `À CONFIRMER` doivent être vérifiées/figées avant usage réel (facturation, signature, dépôt CNDP).

---

## 1. IDENTITÉ COMMERCIALE

| Élément | Valeur |
|---|---|
| **Agence (marque client-facing)** | **Zellia** — `zellia.ma` |
| **Nom de produit interne** | **Mimosa** — ⛔ **JAMAIS client-facing.** N'apparaît dans aucun document, message, ou support vu par le client |
| **Landing page** | `zellia.ma` — live sur Netlify (HTML statique) |
| **Chaîne DNS** | HostoWeb (registrar `.ma`) → Cloudflare (intermédiaire DNS) → Netlify. ⚠️ Cloudflare en **DNS-only (nuage gris)** pour éviter les conflits SSL avec Netlify |
| **Prise de RDV commerciale** | Widget popup Calendly — `calendly.com/kzoubair/30min` |
| **Builder** | Khalid (solo). Un collaborateur technique en cours d'onboarding pour la maintenance infra |

---

## 2. CABINET PILOTE

| Élément | Valeur |
|---|---|
| **Praticien** | **Dr Chaimaa Badrour**, dentiste |
| **Spécialités** | Orthodontie, implantologie, esthétique |
| **Ville** | Tanger |
| **Site** | `drbadrour.ma` |
| **Utilisateur quotidien principal** | La **secrétaire** du cabinet — c'est le champion interne clé, cible n°1 du discours de vente |

**Statut relation :** phase de validation technique. Répétition de démo + prise de contact avec le cabinet à faire.

---

## 3. MODÈLE ÉCONOMIQUE

### Structure de revenu

Deux composantes par cabinet :

| Composante | Nature |
|---|---|
| **Frais d'installation** | Paiement unique au démarrage (setup, configuration WABA, calibration) |
| **Abonnement mensuel (retainer)** | Revenu récurrent — forfait fixe en **MAD** par cabinet |

> **Principe de pricing :** séparer explicitement les deux lignes. Les frais d'installation ne sont **pas** dilués dans l'abonnement. Le retainer mensuel est un **forfait plat en MAD** — les coûts WhatsApp (templates Utility hors fenêtre de service) sont **absorbés dans ce forfait**, pas refacturés à l'usage.

### Fourchette de référence (README §2)
- Maintenance mensuelle : **500 – 2 000 MAD/mois** selon le cabinet.
- ⚠️ Montants exacts installation + abonnement pour Dr Badrour : **À CONFIRMER / figer avant devis.**

### Stratégie commerciale

**Ancrage de la douleur avant le prix (règle d'or) :**
- Toujours chiffrer le **coût des no-shows** avant d'énoncer un tarif.
- Argument type : 5 no-shows/jour × 300 MAD = **45 000 MAD/mois perdus**. ROI démontrable en ~2 semaines.
- Le prix énoncé après cet ancrage paraît négligeable face à la perte évitée.

**Levier de rareté :** utiliser la **phase pilote** comme argument de rareté (place limitée, tarif préférentiel early adopter).

**Ordre d'adresse dans la vente :**
1. **La secrétaire d'abord** — c'est elle qui utilise l'outil au quotidien ; en faire l'alliée interne.
2. Le médecin ensuite — argument = « agenda toujours propre, tu ne touches à rien ».

---

## 4. STRATÉGIE DE DÉMO

**Format : scénario en trois actes uniquement.**

```
Acte 1 : Prise de RDV        (patient WhatsApp → créneau → confirmation)
Acte 2 : Annulation          (patient annule → créneau libéré)
Acte 3 : Cascade liste d'attente  (le créneau libéré est proposé au suivant)
```

**Règles de démo :**
- Montrer **seulement** ces trois actes de bout en bout.
- **Mentionner brièvement** les autres fonctionnalités (report, menu, multilingue, anti-doublon, ESCAPE) sans les dérouler.
- **Ne détailler une fonctionnalité supplémentaire que si le prospect le demande.** Ne pas noyer la démo.

**Support technique de démo :** `scrcpy` (miroir d'écran Android pour Google Meet).

---

## 5. RÔLES DE DONNÉES & CONFORMITÉ (CNDP)

### Répartition des responsabilités

| Rôle | Qui | Responsabilité |
|---|---|---|
| **Responsable de traitement** (data controller) | **Le cabinet** (Dr Badrour) | Détermine les finalités ; responsable des dépôts CNDP |
| **Sous-traitant** (data processor) | **Zellia** | Traite les données pour le compte du cabinet |

> **Conséquence contractuelle :** un **DPA (Data Processing Agreement)** doit être **signé avec chaque client**. C'est Zellia qui est sous-traitant, le cabinet qui porte la responsabilité de traitement.

### Dépôts CNDP requis (portés par le cabinet)

| Formulaire | Objet | Régime |
|---|---|---|
| **F112** | Traitement de données de RDV médical (**santé-adjacent**) | **Autorisation** requise (pas simple déclaration — donnée sensible) |
| **F118** | **Transferts internationaux** vers Meta (WhatsApp), Google, OpenAI | Requis du fait des sous-traitants hors Maroc |

⚠️ Le régime **autorisation** (F112) est plus lourd qu'une simple déclaration — délai à anticiper avant mise en production live avec données patients réelles.

---

## 6. STATUT JURIDIQUE (ENTREPRENEUR)

| Élément | Statut |
|---|---|
| **Forme visée** | **Auto-Entrepreneur** |
| **Séquencement** | Enregistrement peut se faire **en parallèle** pendant l'essai pilote |
| **Contrainte** | **Obligatoire avant la première facture** — pas de facturation légale sans le statut |
| **Budget de départ** | 15 000 MAD |

---

## 7. ARCHITECTURE WhatsApp / META (côté business)

**Modèle retenu : Model B — une WABA par cabinet, hébergée par Zellia.**

| Aspect | Décision |
|---|---|
| **Structure** | 1 WhatsApp Business Account (WABA) **par cabinet client** |
| **Raison** | **Isolation de la quality rating** — un cabinet ne peut pas dégrader la réputation d'un autre |
| **Facturation Meta** | Carte de Zellia, facturée en **USD/EUR** (le forfait client reste en MAD → Zellia absorbe le change) |
| **Vérification OTP** | SIM physique utilisée une fois pour l'OTP de vérification, puis **stockée étiquetée par client** |
| **Meta Business Verification** | **Dépendance externe 3–7 jours** — bloquante avant le Workflow 2 (rappels) et avant de scaler. À compléter tôt |

### Rate card WhatsApp — échéance
- Le Maroc passe à un **rate card Utility autonome en octobre 2026** → **révision de pricing nécessaire à cette date** (impact sur le coût absorbé dans le forfait).

---

## 8. FEUILLE DE ROUTE PRODUIT (business milestones)

Ordre indicatif, dépendances externes en gras :

1. **Workflow 1** (RDV / annulation / report / liste d'attente) — en validation finale. *(détail technique → README)*
2. **Workflow 2** (anti no-show, rappels J-1 et J-2h) — **nécessite Meta Business Verification + templates approuvés** avant build.
   - 4 templates rédigés, catégorie **Utility** : `rappel_rdv_fr`, `rappel_rdv_ar`, `rappel_rdv_2h_fr`, `rappel_rdv_2h_ar`.
   - ⚠️ Contenu exact des templates : **À CONSIGNER** (ne figure pas encore dans un fichier — voir §10).
3. **Meta Business Verification** — dépendance externe 3–7 j, à lancer au plus tôt.
4. **Auto-Entrepreneur** — en parallèle du pilote, avant 1ère facture.
5. **Dépôts CNDP** F112 (autorisation) + F118 (transferts) — portés par le cabinet.
6. **Migration SQLite → PostgreSQL** — recommandée avant le **3ᵉ client**. *(détail → INFRA_NOTES)*
7. **Interface secrétaire AppSheet** — post-pilote. Cahier des charges produit : `CDC_App_Cabinet_v1.md`. ⚠️ **À uploader au projet** (voir §10).

---

## 9. PRINCIPES & APPRENTISSAGES BUSINESS

- **Le client ne voit que 3 outils** : WhatsApp (patients), Google Calendar (agenda), Google Sheets (secrétaire). n8n + OpenAI sont invisibles.
- **Zéro formation longue** : la secrétaire adopte l'outil parce qu'elle connaît déjà WhatsApp + Google.
- **Le médecin ne touche à rien** — argument de vente central.
- **Forfait plat, pas d'usage** : simplicité de facturation > optimisation à l'usage. Le risque de coût WhatsApp est porté par Zellia, pas répercuté.
- **Isolation par WABA** (Model B) : protège la réputation de chaque cabinet indépendamment — choix structurant pour le multi-client.
- **Secrétaire = champion interne** : la vente se gagne d'abord auprès d'elle.

---

## 10. TROUS À COMBLER (documents à rapatrier)

> Éléments mentionnés dans le contexte projet mais **pas encore matérialisés dans un fichier de référence**. À traiter avant tout nettoyage de conversations qui les contiendraient.

- [ ] **Contenu exact des 4 templates Workflow 2** (`rappel_rdv_fr/ar`, `rappel_rdv_2h_fr/ar`) — texte FR + AR, variables, catégorie Utility. → à consigner (ici ou dans un futur `README` Workflow 2).
- [ ] **`CDC_App_Cabinet_v1.md`** (cahier des charges interface secrétaire AppSheet) — existe en tant que livrable, **à uploader au projet**.
- [ ] **Discussion architecture multi-tenant « options A/B/C »** — référencée dans INFRA_NOTES mais non détaillée ; le raisonnement complet est à extraire si utile.
- [ ] **Montants figés** installation + abonnement pour Dr Badrour (actuellement fourchette 500–2 000 MAD).

---

*BUSINESS_LEGAL_CONTEXTE — créé le 08/08/2026. Reconstitué à partir du contexte projet accumulé pour combler l'absence d'un fichier business/légal dédié (les 3 fichiers existants ne couvrant que la technique). À faire vivre en parallèle des fichiers techniques. Valeurs `À CONFIRMER` / cases `§10` à traiter avant facturation, signature DPA, ou dépôt CNDP.*

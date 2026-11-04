# beat-and-teach-it : Specs parallèles (amendements validés au spec officiel)

> **Rôle** : registre des **règles métier validées** qui amendent, précisent ou étendent le spec officiel (immuable).
>
> **Primauté** : une entrée `SP-*` **en vigueur** prime sur la page officielle correspondante.
>
> **Périmètre** : règles de **comportement / flux / calculs**. Définitions d'entités → `.agents/domain-glossary.md`. Méthodes de code → `.agents/reference-utilitaires.md`.
>
> **Écriture** : **Build uniquement**, après validation explicite de l'utilisateur, à la **recette nuances de l'étape 6**, ou à la **clôture d'un chantier PO** (`/cloturer-chantier`, skill `cadrage-po`). Le raffineur et `@review` détectent et proposent, ils n'écrivent jamais.
>
> **Lecture** : jamais le fichier entier. Index puis entrées ciblées (grep).
>
> **Cycle de vie** : une règle qui change = nouvelle entrée + ancienne marquée caduque. Jamais de réécriture de l'histoire.
>
> Dernière mise à jour : 2026-11-04

## Index

| ID          | Statut                         | Type    | Ancre officielle                     | Titre                                                            |
| ----------- | ------------------------------ | ------- | ------------------------------------ | ---------------------------------------------------------------- |
| SP-THEME-01 | en vigueur (validé 2026-11-04) | AMENDE  | §7 (Palette de couleurs et thème)    | Trois thèmes d'accent, défaut bleu et persistance locale         |
| SP-NOTE-01  | en vigueur (validé 2026-11-04) | ETEND   | §2.3 (Phrase)                        | Notation `@fréquence` dans les tokens                            |
| SP-NOTE-02  | en vigueur (validé 2026-11-04) | PRECISE | §2.1 (Instrument) et §6 (flux audio) | Lecture pitchée : `reference_frequency`, playbackRate et molette |

## Domaines (codes)

- `THEME` — apparence : thèmes de couleurs, accents, persistance.
- `NOTE` — hauteur : notation `@fréquence`, lecture pitchée.

Liste fermée : extension sur validation utilisateur.

## Statuts & types

- **Statuts** : `en vigueur (validé YYYY-MM-DD)` · `hypothèse (retenue provisoire, en attente de confirmation)` · `caduc (→ SP-…)`
- **Types** : `AMENDE` (remplace la page) · `PRECISE` (complète une règle) · `ETEND` (cas non couvert)

## Contrôle d'intégrité

> Exécuté par `@review` à l'étape 5 et par le Build après chaque écriture : `node workflow-kit/bin/workflow-kit.mjs check-meta --spec` (mécanique, ne se fait pas à la main). 4 vérifications, tout écart = signalement.

1. **Index ↔ entrées** : autant de lignes `| SP-…` que d'entrées `### SP-…`, mêmes IDs.
2. **Statuts cohérents** : statut de l'index = ligne `> **Statut**` de l'entrée.
3. **Supersessions valides** : chaque cible de `caduc (→ SP-…)` / `**Remplace**` existe comme entrée.
4. **Mots-clés présents** : chaque entrée porte `> **Mots-clés** : …`.

---

## Entrées

### SP-THEME-01 : Trois thèmes d'accent, défaut bleu et persistance locale

> **Statut** : en vigueur (validé 2026-11-04) · **Type** : AMENDE
> **Ancre officielle** : §7 (Palette de couleurs et thème)
> **Source** : feature `.agents/feature/DONE-themes-couleurs.md`
> **Mots-clés** : thème, palette, accent, bleu, vert, rose, data-theme, localStorage, défaut
> **Gardé par** : `src/renderer/features/theme/tests/themes.test.ts::themes`, `src/renderer/features/theme/tests/theme-storage.test.ts::resolveInitialTheme`

**Règle.** Trois thèmes d'accent sont proposés : `blue` (défaut), `green` et `pink`. Le thème ne modifie que les accents — `primary`, `text.accent`, `button.edit` et le logo ; les gris (fonds, surfaces, bordures, champs, textes principal et secondaire, boutons neutres et destructifs) restent identiques dans les trois thèmes.

Le thème est appliqué sur `<html>` par l'attribut `data-theme` (`green` ou `pink`) ; le thème `blue` ne pose aucun attribut et laisse les valeurs par défaut de `:root`. Le choix est persisté dans `localStorage` sous la clé `beat-and-teach:theme` et restauré au démarrage, y compris avant le premier rendu (script anti-FOUC de `index.ejs`). Une clé absente, vide, inconnue ou malformée, comme une lecture ou une écriture `localStorage` en erreur, laisse l'application sur `blue` sans message d'erreur.

Le changement de thème est immédiat ; la transition de couleur reste limitée aux éléments concernés (150–200 ms), jamais une transition universelle sur `*`. Restent hors thème : les cinq couleurs de highlight de la grille (rouge, bleu, vert, jaune, orange), le rouge d'erreur des validations et le fond des modales.

**Cas limites.**

- `localStorage` indisponible (mode privé, quota) : lecture et écriture échouent silencieusement ; l'application reste utilisable sur `blue`.
- Une valeur `data-theme` inconnue posée dans le DOM retombe sur les variables de `:root` (bleu), sans style cassé.
- Le script anti-FOUC ne pose que `green` ou `pink` ; toute autre valeur est ignorée.
- Les rendus canvas (waveform de l'enregistreur) ne consomment pas les variables CSS : leurs couleurs restent des paramètres explicites, hors thème.

### SP-NOTE-01 : Notation `@fréquence` dans les tokens

> **Statut** : en vigueur (validé 2026-11-04) · **Type** : ETEND
> **Ancre officielle** : §2.3 (Phrase)
> **Source** : feature `.agents/feature/DONE-gerer-notes.md`
> **Mots-clés** : fréquence, hauteur, pitch, notation, arobase, token, phrase, source unique, token aplati
> **Gardé par** : `src/renderer/utils/token-parser.test.ts::parseToken`, `src/renderer/utils/update-token-frequency.test.ts::updateTokenFrequency`, `src/renderer/features/pattern/tests/pattern-parser.test.ts::parseSteps`

**Règle.** Un token de phrase peut porter un suffixe de hauteur : `symbole@fréquence` (`P@220`, `Hum@440.5`, `Hum@A3`, `Ts@Bb3`). La fréquence est acceptée en hertz (nombre strictement positif) ou en nom de note (`Nom[#b]octave`, casse insensible), résolue en hertz par tempérament égal (`A4 = 440`).

Le texte de la phrase reste la source unique de vérité : la fréquence vit dans le token, aucune colonne `frequencies` ni aucun tableau miroir n'est stocké. Un suffixe vide, non numérique, nul, négatif ou un nom de note invalide est ignoré : le token garde son symbole, sans fréquence et sans erreur. Un token sans suffixe n'a pas de fréquence, donc pas de playbackRate calculé.

`@` est interdit dans le symbole d'un instrument : la validation de création le refuse pour lever l'ambiguïté avec le suffixe. Dans un groupe `(…)`, chaque token peut porter sa propre fréquence. Pour poser ou retirer une fréquence, l'index de token est l'index **aplati** : un groupe de N tokens compte N, pas un step visuel.

**Cas limites.**

- `(A@220 B)` : seule A est pitchée.
- `P@0`, `P@-10`, `P@`, `P@abc` : suffixe ignoré.
- `A@B@440` : le premier `@` sépare le symbole ; le reste `B@440` n'est ni un nombre ni une note, donc aucune fréquence n'est retenue.
- `P@A#4` (dièse), `P@Bb3` (bémol), `P@a4` (casse) : reconnus.
- Un groupe `(A B C)` compte trois tokens aplatis : la molette sur B utilise l'index du token B, pas celui du groupe.

### SP-NOTE-02 : Lecture pitchée — `reference_frequency`, playbackRate et molette

> **Statut** : en vigueur (validé 2026-11-04) · **Type** : PRECISE
> **Ancre officielle** : §2.1 (Instrument) et §6 (flux audio)
> **Source** : feature `.agents/feature/DONE-gerer-notes.md`
> **Mots-clés** : playbackRate, reference_frequency, pitch, demi-ton, molette, clamp, beatpack, manifest
> **Gardé par** : `src/renderer/features/sequence/tests/sequence-service.test.ts::preparePattern`, `src/main/library/tests/export-library-service.test.ts`, `src/main/library/tests/import-library-service.test.ts`

**Règle.** Chaque instrument peut porter une fréquence de référence (`reference_frequency`, en hertz, nullable), détectée par autocorrélation ou saisie manuellement. À la lecture d'un token portant une fréquence cible : `playbackRate = cible / référence`, borné à `[0.25, 4]` ; le décalage en demi-tons est `round(12 * log2(playbackRate))`.

Si la référence est absente ou inférieure ou égale à zéro : `playbackRate = 1` et décalage `0`, sans erreur. La molette sur un step modifie la fréquence de ±1 demi-ton (`fréquence * 2^(±1/12)`) et réécrit le texte ; un double-clic retire la fréquence. Il n'y a pas de time-stretch en V1 : la durée du sample varie avec la hauteur.

L'export `.beatpack` écrit un manifest `version: 2` avec `referenceFrequency` par instrument ; l'import accepte `version: 1` et `version: 2`, et une valeur absente devient `null`.

**Cas limites.**

- Deux pistes, même instrument, même step, rates différents : chaque lecture crée sa propre source audio ; le `playbackRate` n'est pas partagé.
- Une référence `0` ou négative est traitée comme absente.
- Un rate hors bornes est clampé, pas rejeté.
- Un manifest v1 (sans `referenceFrequency`) importe des instruments à `null`.
- La molette ne fait rien si l'instrument n'a pas de fréquence de référence : le step n'est pas pitchable.

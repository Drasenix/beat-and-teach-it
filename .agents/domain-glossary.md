# Glossaire métier — Beat & Teach

> **Lire ce fichier avant de toucher aux modèles métier.**
>
> **Source de vérité des flux/règles** : `.agents/specs/spec.txt` (spec officiel) + `.agents/specs/spec-parallele.md` (`SP-*` qui priment).
> Définitions d'entités → ce fichier.

---

## Entités principales

### Instrument

- **Table** : `instruments`
- **Colonnes clés** : `id`, `slug`, `symbol`, `name`, `filepath`, `reference_frequency`
- **Relations** : —
- **Rôle métier** : unité sonore de base ; associe un **symbole** de notation à un **fichier audio** et un **nom lisible**.
- **Règles** : `symbol` et `slug` uniques. L'instrument `symbol: "."` (silence, `name: null`, `filepath: null`) est filtré de tous les affichages UI. Un instrument sans fichier audio peut exister mais est silencieux. `reference_frequency` (Hz, nullable) est la hauteur naturelle du sample ; elle sert de base au calcul du playbackRate et peut être détectée par autocorrélation ou saisie manuellement. `@` est interdit dans `symbol` (ambiguïté avec le suffixe de fréquence).

### Pattern

- **Table** : `patterns`
- **Colonnes clés** : `id`, `slug`, `name`, `sentences` (JSON), `highlights` (JSON)
- **Relations** : —
- **Rôle métier** : composition musicale ; contient un nom, une ou plusieurs **phrases** (pistes) et des **couleurs de surbrillance** par token.
- **Règles** : `slug` auto-généré depuis `name`, unique. Toutes les phrases sont normalisées sur la longueur de la **première** (padding `.`). Les phrases après la première sont bloquées si elles dépassent la longueur de la première. Les `highlights` sont stockés par **position de token** (un groupe `(…)` = 1 step visuel mais N tokens). Les fréquences de hauteur vivent dans le texte des phrases : aucune colonne `frequencies` ni tableau miroir.

### Phrase (Sentence)

- **Rôle métier** : ligne de notation textuelle, ex. `P Ts (K . P) Ts K .`.
- **Règles** : tokens séparés par des espaces ; chaque token = symbole d'instrument, groupe `(…)`, ou symbole suivi d'un suffixe de hauteur `@fréquence` (`P@220`, `Hum@A3`). `.` = silence ; une phrase peut être vide. Le texte est la source unique de vérité : la fréquence est portée par le token, jamais par un état séparé.

### Groupe `(…)`

- **Rôle métier** : succession rapide d'instruments dans une même pulsation (subdivision du temps).
- **Règles** : `P (Lo P)` → kick sur le temps, puis Lo puis kick en subdivision. Chaque token du groupe peut porter sa propre fréquence. Représentation interne : `PatternStep { id, symbol, valid, isGroup, steps?, frequency? }`. Dans la grille : bordure `border-primary` pointillée (`step-cell-group`). Un groupe compte pour **1 step visuel** mais **N tokens aplatis** pour les highlights, le mute et l'édition de fréquence.

---

## Termes transverses

### Thème

- **Codes** : `blue` (défaut), `green`, `pink`.
- **Rôle métier** : palette d'accent choisie par l'utilisateur ; ne change que les accents (`primary`, `text.accent`, `button.edit`, logo), jamais les gris, les highlights ni les erreurs.
- **Persistance** : `localStorage`, clé `beat-and-teach:theme`.
- **Règle** : `SP-THEME-01`.

### Fréquence de référence

- **Colonne** : `instruments.reference_frequency` (Hz, nullable).
- **Rôle métier** : hauteur naturelle du sample ; base du calcul du playbackRate.
- **Règle** : `SP-NOTE-02`.

### `.beatpack`

- **Rôle métier** : archive ZIP d'export/import de patterns et d'instruments (`manifest.json` + `audio/`).
- **Versions** : `1` (sans fréquence de référence) · `2` (avec `referenceFrequency`).
- **Règle** : `SP-NOTE-02`.

### Mute

- **Rôle métier** : mise en silence temporaire d'un step pour la lecture, sans modifier la phrase stockée.
- **Portée** : état de session, non persisté ; clé `"sentenceIndex-tokenIndex"` sur l'index **aplati**.

---

## Statuts et codes canoniques

| Entité                    | Codes canoniques                           | Notes                                   |
| ------------------------- | ------------------------------------------ | --------------------------------------- |
| `ConflictAction` (import) | `overwrite`, `skip`, `rename`              | Résolution de conflit `.beatpack`       |
| `HighlightColor` (grille) | `red`, `blue`, `green`, `yellow`, `orange` | Sémantique pédagogique fixe, hors thème |
| `ThemeId`                 | `blue`, `green`, `pink`                    | `blue` = défaut ; voir `SP-THEME-01`    |
| `LibraryManifestVersion`  | `1`, `2`                                   | v2 ajoute `referenceFrequency`          |

---

## Questions encore ouvertes

- Aucune pour l'instant.

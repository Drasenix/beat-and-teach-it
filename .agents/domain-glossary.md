# Glossaire métier — Beat & Teach

> **Lire ce fichier avant de toucher aux modèles métier.**
>
> **Source de vérité des flux/règles** : `.agents/specs/spec.txt` (spec officiel) + `.agents/specs/spec-parallele.md` (`SP-*` qui priment).
> Définitions d'entités → ce fichier.

---

## Entités principales

### Instrument
- **Table** : `instruments`
- **Colonnes clés** : `id`, `slug`, `symbol`, `name`, `filepath`
- **Relations** : —
- **Rôle métier** : unité sonore de base ; associe un **symbole** de notation à un **fichier audio** et un **nom lisible**.
- **Règles** : `symbol` et `slug` uniques. L'instrument `symbol: "."` (silence, `name: null`, `filepath: null`) est filtré de tous les affichages UI. Un instrument sans fichier audio peut exister mais est silencieux.

### Pattern
- **Table** : `patterns`
- **Colonnes clés** : `id`, `slug`, `name`, `sentences` (JSON), `highlights` (JSON)
- **Relations** : —
- **Rôle métier** : composition musicale ; contient un nom, une ou plusieurs **phrases** (pistes) et des **couleurs de surbrillance** par token.
- **Règles** : `slug` auto-généré depuis `name`, unique. Toutes les phrases sont normalisées sur la longueur de la **première** (padding `.`). Les phrases après la première sont bloquées si elles dépassent la longueur de la première. Les `highlights` sont stockés par **position de token** (un groupe `(…)` = 1 step visuel mais N tokens).

### Phrase (Sentence)
- **Rôle métier** : ligne de notation textuelle, ex. `P Ts (K . P) Ts K .`.
- **Règles** : tokens séparés par des espaces ; chaque token = symbole d'instrument ou groupe `(…)` ; `.` = silence ; une phrase peut être vide.

### Groupe `(…)`
- **Rôle métier** : succession rapide d'instruments dans une même pulsation (subdivision du temps).
- **Règles** : `P (Lo P)` → kick sur le temps, puis Lo puis kick en subdivision. Représentation interne : `PatternStep { id, symbol, valid, isGroup, steps? }`. Dans la grille : bordure `border-primary` pointillée (`step-cell-group`).

---

## Statuts et codes canoniques

| Entité | Codes canoniques | Notes |
|---|---|---|
| `ConflictAction` (import) | `overwrite`, `skip`, `rename` | Résolution de conflit `.beatpack` |

---

## Questions encore ouvertes

- Aucune pour l'instant.

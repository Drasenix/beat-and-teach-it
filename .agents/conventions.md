# Conventions — Beat & Teach

> Spécialisation projet du pack `electron-react-ts`. Conventions **techno** applicables à tout le code.

## Identité

- **Projet** : Beat & Teach — composition rythmique symbolique.
- **Stack** : Electron + React + TypeScript + Tone.js + SQLite (better-sqlite3) + Tailwind CSS 3.
- **Architecture** : feature-sliced (domain-driven), main/renderer séparé, IPC typé.

## Règles de développement strictes

- Pas de `for...of` (`.forEach()` obligatoire).
- Pas d'implicit `any` — toujours un type explicite (`const x: Type = ...`).
- Pas d'index comme `key` React — utiliser des ID stables uniques.
- Pas d'`eslint-disable` ni de `// eslint-disable-next-line`.
- `error: unknown` dans les `catch`, puis `instanceof Error` pour le narrowing.
- **Immutabilité** : toujours créer une nouvelle référence pour tableaux/objets.
- Pas de `useEffect` pour les dérivations calculables — utiliser `useMemo`.
- **Pas de commentaires dans le code** — le code doit être auto-documenté.

## Organisation de la logique

- **Feature-sliced** : chaque domaine suit `models/types/adapters/services/facade/engine/hooks/contexts/components/utils`.
- **Nommage strict** : `*-dto.ts`, `*-facade.ts`, `*-adapter.ts`, `*-engine.ts`, `*-service.ts`, `*-model.ts`.
- **Facade** = API publique du domaine — les composants n'appellent que les facades, jamais les services directement.
- **Adapter** = mapping DTO (main) ↔ Modèle (renderer).
- **Singleton pour la logique métier** : `AudioEngine`, `InstrumentEngine`.
- **Validation des données** : toujours valider côté main **et** côté renderer.
- **État global réservé aux données partagées** (listes patterns/instruments, session audio). Tout le reste en état local (`useState` dans le composant).
- **⚠️ Feature-sliced, pas hexagonal** : les `engine/` contiennent la logique métier pure, les `facade/` servent d'API publique, mais UI et métier cohabitent dans le même slice. Pas d'interface entre facade et service — l'appel IPC est direct. Pas de ports, pas d'inversion de dépendance formelle.

## Conventions par domaine

### Pattern

- `usePattern` = état local d'un pattern en cours d'édition.
- `usePatternSession` = wrapper avec gestion du mute (`Set<string>` de clés `"sentenceIndex-tokenIndex"`).
- `usePatternForm` = gestion de formulaire avec synchronisation des highlights.
- Parser = fonctions pures dans `pattern-parser.ts`.
- Normalisation automatique des phrases non-racines sur la longueur de la première.

### Instruments

- `InstrumentEngine` (singleton) pour la résolution symbole → nom/chemin.
- L'instrument `symbol: "."` est toujours filtré de l'UI.
- Slug auto-généré via `toSnakeCase()` — valide `[a-z0-9-]`.
- Les chemins audio relatifs sont résolus par rapport à `assets/audio/` ou `resources/assets/audio/`.

### Audio

- `AudioEngine` (singleton) encapsule Tone.js.
- `createSequence(tracks)` = un `Tone.Sequence` par piste, subdivision `'8n'`.
- Groupes `(…)` → `SequenceNote[]` (tableau) — Tone.js dispatche chaque élément à la même position temporelle.
- Step callback = `Tone.Loop` à `'8n'` → `activeStep` pour l'UI.
- BPM clampé 1-300 (UI slider 40-240).

### Sequence

- `preparePattern(sentence)` → `SequenceNotes[]` = `(string | null | string[])[]`.
- `prepareFilePaths(sentence)` → `InstrumentFilePath[]` (symboles uniques dédupliqués).
- Regex de parsing : `/\(([^)]*)\)|(\S+)/g`.

### Library

- `.beatpack` = ZIP avec `manifest.json` + `audio/*`.
- Export : archiver (store level), Import : adm-zip.
- `ConflictAction` : `'overwrite' | 'skip' | 'rename'`.
- Fichiers audio importés copiés dans `userData/imported-audio/`.

## Tests

- **TDD pour le code métier** : parser, validator, mute, adapters, engine, service, **utils** (fonctions pures) — écrire le test avant l'implémentation.
- **Pas de TDD pour le frontend** : composants React, hooks UI, contexts — tests après ou pas de tests.
- **Refactoring systématique** : extraire les fonctions pures, réduire la duplication, simplifier.
- **Respecter le style existant** : conventions de nommage, patterns, structure.
- **Force de proposition** : suggérer activement des améliorations d'architecture et de pratiques de dev, avec justification.

## Style

- **Formatter** : Prettier (`singleQuote: true`).
- **Linter** : ESLint (`eslint-config-erb`).

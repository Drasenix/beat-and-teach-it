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

### Responsabilités et abstractions

- **Une raison de changer par fichier** : un engine porte les règles métier d'un domaine ; un service, un accès (canal IPC, données) ; un adapter, un mapping DTO ↔ modèle. Une règle d'un autre domaine va dans ce domaine, pas dans un engine ou un service existant.
- **Où vit une règle** : règle métier → engine ; calcul pur sans règle métier → utils ; conversion DTO ↔ modèle → adapter ; accès IPC ou données → service.
- **KISS** : pas d'interface, de classe abstraite, de paramètre ou de garde qu'aucun besoin présent ne justifie. En test, `jest.mock()` remplace le module : pas d'interface pour la seule testabilité.
- **Sous-classe** : elle accepte tout ce que le parent accepte et rend ce qu'il promet.
- Critères communs (code mort, duplication, surenchère) : `workflow-kit/core/qualite-code.md`.

## Conventions par domaine

### Pattern

- `usePattern` = état local d'un pattern en cours d'édition.
- `usePatternSession` = wrapper avec gestion du mute (`Set<string>` de clés `"sentenceIndex-tokenIndex"`).
- `usePatternForm` = gestion de formulaire avec synchronisation des highlights.
- Parser = fonctions pures dans `pattern-parser.ts`.
- Hauteur : un token peut porter `@fréquence` (Hz ou nom de note). Le texte de la phrase est la source unique ; aucun tableau `frequencies` séparé.
- Index : highlights, mute et édition de fréquence utilisent l'index **aplati** (un groupe de N tokens compte N) ; `countSentenceSteps` ne sert qu'à l'affichage visuel.
- Normalisation automatique des phrases non-racines sur la longueur de la première.

### Instruments

- `InstrumentEngine` (singleton) pour la résolution symbole → nom/chemin.
- L'instrument `symbol: "."` est toujours filtré de l'UI.
- Slug auto-généré via `toSnakeCase()` — valide `[a-z0-9-]`.
- Les chemins audio relatifs sont résolus par rapport à `assets/audio/` ou `resources/assets/audio/`.
- `referenceFrequency` (Hz, nullable) = hauteur naturelle du sample ; détectée par autocorrélation (`detectPitch`) ou saisie manuellement, stockée dans `instruments.reference_frequency`.
- `@` est interdit dans le `symbol` (ambiguïté avec le suffixe de fréquence).

### Audio

- `AudioEngine` (singleton) encapsule Tone.js.
- `createSequence(tracks)` = un `Tone.Sequence` par piste, subdivision `'8n'`.
- Groupes `(…)` → `SequenceNote[]` (tableau) — Tone.js dispatche chaque élément à la même position temporelle.
- Step callback = `Tone.Loop` à `'8n'` → `activeStep` pour l'UI.
- BPM clampé 1-300 (UI slider 40-240).
- Pitch : `SequenceNote` porte `playbackRate` et `semitoneOffset` ; `playbackRate = cible / reference_frequency`, clampé `[0.25, 4]`, et la référence absente ou ≤ 0 donne `1`. Pas de time-stretch en V1.

### Sequence

- `preparePattern(sentence)` → `SequenceNotes[]` = `(SequenceNote | SequenceNote[])[]` (un groupe = tableau de notes ; `null` = silence).
- `prepareFilePaths(sentence)` → `InstrumentFilePath[]` (symboles uniques dédupliqués).
- `@fréquence` → `playbackRate = cible / reference_frequency`, clampé `[0.25, 4]`, `semitoneOffset = round(12 * log2(rate))`.
- Regex de parsing : `/\(([^)]*)\)|(\S+)/g`.

### Library

- `.beatpack` = ZIP avec `manifest.json` + `audio/*`.
- Export : archiver (store level), manifest `version: 2` avec `referenceFrequency` par instrument. Import : adm-zip, versions `1` et `2` acceptées.
- `ConflictAction` : `'overwrite' | 'skip' | 'rename'`.
- Fichiers audio importés copiés dans `userData/imported-audio/`.

### Thème

- Slice `features/theme/` : registre pur `themes.ts`, persistance `theme-storage.ts`, `ThemeContext` qui applique `data-theme` sur `<html>`.
- 3 thèmes : `blue` (défaut), `green`, `pink`. Seuls les accents changent ; gris, highlights, rouge d'erreur et backdrop restent fixes.
- Persistance `localStorage` (`beat-and-teach:theme`) ; valeur absente/invalide ou erreur de stockage → `blue`, sans erreur.
- Pas de couleur hex en dur dans le JSX : tokens Tailwind adossés à `--bt-*` (voir `.agents/ui-rules.md`).

### Recorder

- Slice `features/recorder/` : `useRecorder` (`getUserMedia` + `ScriptProcessorNode`), `wav-encoder.ts` (PCM 16 bits mono), `waveform-renderer.ts` (canvas).
- Sauvegarde via IPC `open-audio-save-dialog` puis `save-recorded-audio`.
- Le canvas de waveform ne consomme pas les variables CSS du thème : ses couleurs sont des paramètres explicites.

## Tests

- **TDD pour le code métier** : parser, validator, mute, adapters, engine, service, **utils** (fonctions pures) — écrire le test avant l'implémentation.
- **Pas de TDD pour le frontend** : composants React, hooks UI, contexts — tests après ou pas de tests.
- **Refactoring systématique** : extraire les fonctions pures, réduire la duplication, simplifier.
- **Respecter le style existant** : conventions de nommage, patterns, structure.
- **Force de proposition** : suggérer activement des améliorations d'architecture et de pratiques de dev, avec justification.

## Style

- **Formatter** : Prettier (`singleQuote: true`).
- **Linter** : ESLint (`eslint-config-erb`).

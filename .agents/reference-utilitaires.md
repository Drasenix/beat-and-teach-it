# Référence : Utilitaires et méthodes réutilisables

> Méthodes réutilisables du projet : avant d'écrire une nouvelle méthode, vérifier ici si l'équivalent existe déjà.
> Tenu par le Build, sur accord explicite de l'utilisateur (cf. `workflow-kit/core/agent-build.md` § Fichiers meta).
>
> **Garde-fou** : aucun agent ne modifie ce fichier sans l'accord explicite de l'utilisateur.

---

## Utilitaires généraux (renderer)

### util.ts

- **Fichier** : `src/renderer/utils/util.ts`
- **Type** : Utils (fonctions pures)
- **Méthodes** :
  - `removeDuplicates<T>(array: T[]): T[]` : dédoublonne par `Set` (ordre d'apparition conservé).
  - `removeParenthesis(string: string): string` : retire les `()` et normalise les espaces.
  - `toSnakeCase(name: string): string` : slug `[a-z0-9-]` — **produit des `-`, pas des `_`** (nom trompeur).
  - `extractIpcError(error: any, fallback?: string): string` : nettoie le préfixe Electron `Error invoking remote method '…': Error: ` avant affichage.
- **Utilisé par** : facades renderer, composants (affichage d'erreur IPC), génération de slugs d'instruments.

### normalize-spaces.ts

- **Fichier** : `src/renderer/utils/normalize-spaces.ts`
- **Type** : Utils (fonction pure)
- **Méthodes** :
  - `normalizeSpaces(text: string): string` (export par défaut) : collapse les espaces multiples **et** retire les espaces juste après `(` / juste avant `)`.
- **Utilisé par** : édition de phrases (composer).

---

## Parsing de phrases (notation symbolique)

### token-parser.ts

- **Fichier** : `src/renderer/utils/token-parser.ts`
- **Type** : Utils (fonctions pures) + type `ParsedToken = { symbol: string; frequency?: number }`
- **Méthodes** :
  - `parseToken(raw: string): ParsedToken` : sépare `symbole@fréquence`. Le suffixe est accepté en **Hz numérique** (`P@220`) ou en **nom de note** (`P@A3`) ; sinon `frequency` est absent. **Pièges** : un `@` en tête n'est pas un suffixe (`@440` reste un symbole) ; seul le premier `@` sépare, le reste doit être un nombre ou une note (`A@B@440` → symbole `A`, sans fréquence).
- **Utilisé par** : `sentence-tokenizer`, `extractSymbols`.

### sentence-tokenizer.ts

- **Fichier** : `src/renderer/utils/sentence-tokenizer.ts`
- **Type** : Utils (fonctions pures) + interface `TokenMatch = { group?: string; symbol?: string; fullMatch: string }`
- **Méthodes** :
  - `tokenizeSentence(sentence: string): TokenMatch[]` : regex `/\(([^)]*)\)|(\S+)/g` — un groupe `(…)` remonte dans `group` (contenu brut), sinon le symbole dans `symbol`.
  - `extractSymbols(sentence: string): string[]` : symboles **à plat** (les groupes sont éclatés en leurs tokens `parseToken`).
  - `extractUniqueSymbols(sentences: string[]): string[]` : union dédoublonnée sur plusieurs phrases.
- **Utilisé par** : sequence service/adapter, chargement des buffers audio.

---

## Pattern (grille et édition)

### pattern-parser.ts

- **Fichier** : `src/renderer/features/pattern/utils/pattern-parser.ts`
- **Type** : Utils (fonctions pures)
- **Méthodes** :
  - `createStep(raw, validSymbols, id): PatternStep` : parse un token (`@fréquence` compris) et pose `valid` selon le catalogue.
  - `createGroup(raw, symbols, startId): PatternStep` : éclate le contenu d'un groupe en sous-steps ; le `symbol` affiché perd les `@fréquence`.
  - `parseSteps(sentence, validSymbols): PatternStep[]` : tokenise puis crée steps et groupes.
  - `flatTokenCount(sentence): number` : nombre de tokens **aplatis** (retire les parenthèses) — un groupe de N tokens compte N.
  - `buildFlatTokenIndices(track): number[]` : pour chaque step, l'index aplati de son **premier** token ; un groupe avance de `steps.length`.
  - `countSentenceSteps(sentence): number` : nombre de steps **visuels** (un groupe = 1).
- **Utilisé par** : grille (`PatternSteps`), `usePattern`, édition de fréquence.
- **Pièges** : `countSentenceSteps` (visuel) et `flatTokenCount` (aplati) ne répondent pas à la même question ; utiliser le bon index pour les highlights, le mute et `updateTokenFrequency`.

### pattern-mute.ts

- **Fichier** : `src/renderer/features/pattern/utils/pattern-mute.ts`
- **Type** : Utils (fonctions pures)
- **Méthodes** :
  - `transformSentencesWithMute(sentences, mutedSteps): string[]` : remplace les tokens mutés par `.` pour la lecture, **sans** modifier les phrases stockées.
  - `createMuteKey(sentenceIndex, tokenIndex): string` / `isStepMuted(…)` / `toggleMute(…)` : clé `"sentenceIndex-tokenIndex"` sur l'index **aplati**, Set immuable.
- **Utilisé par** : `usePatternSession` (`sentencesForPlayback`).
- **Piège** : le mute vit en session et n'est pas persisté ; la clé d'un groupe suit les tokens aplatis, pas le step visuel.

### pattern-validator.ts

- **Fichier** : `src/renderer/features/pattern/utils/pattern-validator.ts`
- **Type** : Validator (fonctions pures)
- **Méthodes** :
  - `validatePattern(fields): string[]` (export par défaut) : nom requis, au moins une phrase, chaque phrase non vide.
  - `areAllSymbolsValid(sentences, validSymbols): boolean` : ignore les `@fréquence` via `parseToken`, tolère `.` et les groupes.
- **Utilisé par** : formulaire pattern, lecture.

---

## Fréquences / notes

### note-name-to-frequency.ts

- **Fichier** : `src/renderer/utils/note-name-to-frequency.ts`
- **Type** : Utils (fonction pure, export par défaut)
- **Méthodes** :
  - `noteNameToFrequency(noteName: string): number | null` : `Nom[#b]octave` → Hz (tempérament égal, `A4 = 440`). `null` si non reconnu.
- **Utilisé par** : `token-parser`.

### frequency-to-note.ts

- **Fichier** : `src/renderer/utils/frequency-to-note.ts`
- **Type** : Utils (fonction pure, export par défaut)
- **Méthodes** :
  - `frequencyToNoteName(frequency: number): string` : Hz → nom de note le plus proche (dièses), ex. `A4`.
- **Utilisé par** : `update-token-frequency`.

### update-token-frequency.ts

- **Fichier** : `src/renderer/utils/update-token-frequency.ts`
- **Type** : Utils (fonction pure, export par défaut)
- **Méthodes** :
  - `updateTokenFrequency(sentence: string, flatTokenIndex: number, frequency: number | null): string` : pose/retire un `@note` sur le token d'index **aplati**. **Piège** : un groupe `(…)` compte pour **N tokens** (son contenu éclaté), pas pour 1 step visuel. `frequency === null` retire le suffixe.
- **Utilisé par** : édition de fréquence par token.

### detect-pitch.ts

- **Fichier** : `src/renderer/utils/detect-pitch.ts`
- **Type** : Utils (fonction pure, export par défaut)
- **Méthodes** :
  - `detectPitch(samples: Float32Array, sampleRate: number): number | null` : autocomplétion par autocorrélation. `null` si échantillon trop court (`< 0.02 s`), énergie nulle, ou **confiance < 0.5** (constante interne `MIN_CONFIDENCE`).
- **Utilisé par** : feature recorder.

---

## Sequence (pattern → notes audio)

### sequence-service.ts

- **Fichier** : `src/renderer/features/sequence/service/sequence-service.ts`
- **Type** : Service (fonctions pures async)
- **Méthodes** :
  - `prepareFilePaths(sentence): Promise<InstrumentFilePath[]>` : symboles uniques (parenthèses retirées) → chemins audio ; lève si un symbole est inconnu.
  - `preparePattern(sentence): Promise<SequenceNotes[]>` : tokenise et produit les notes ; un groupe devient un tableau de notes.
- **Utilisé par** : `AudioContext` / `audio-facade`.
- **Pièges** : `playbackRate = fréquence cible / reference_frequency`, clampé `[0.25, 4]` ; référence absente ou ≤ 0 → rate `1` et offset `0` ; `semitoneOffset = round(12 * log2(rate))`.

---

## Audio (moteur Tone.js)

### AudioEngine (singleton)

- **Fichier** : `src/renderer/features/audio/engine/audio-engine.ts`
- **Type** : Engine (singleton)
- **Méthodes** :
  - `getInstance(): AudioEngine`
  - `setTempo(bpm: number): void` : BPM du Transport + recalcule `stepDuration` (`Tone.Time('8n').toSeconds()`).
  - `createPlayers(audioBuffers: AudioFileBuffer): Promise<void>` : décode et charge les buffers.
  - `addToPlayers(audioBuffers: AudioFileBuffer): Promise<void>` : ajoute des buffers sans reset.
  - `createSequence(tracks: SequenceNotes[][]): void` : reconstruit les `Tone.Loop` par piste (une boucle par piste, si `onStep` est posé).
  - `updateSequences(tracks: SequenceNotes[][]): void` : met à jour `trackNotes` ; reconstruit les boucles seulement si le nombre de pistes change.
  - `play(): Promise<void>` / `stop(): void`.
  - `playInstrument(name: string): Promise<void>` : pré-écoute d'un instrument.
  - `setStepCallback(cb: StepCallback): void` / `clearStepCallback(): void` : callback `(trackIndex, stepIndex)` pour l'UI.
  - `hasSymbol(symbol: string): boolean` / `registerSymbol(symbol: string): void`.
- **Utilisé par** : `AudioContext` / `audio-facade`.
- **Pièges** :
  - `Tone.Player.playbackRate` est un setter qui réécrit la `StateTimeline` : jamais d'assignation pendant la lecture ; le rate est fixé à la création du Player.
  - Le pool est indexé par `trackIndex`, instrument et `semitoneOffset` : deux fréquences du même demi-ton partagent le Player du premier usage.
  - `localStep` est une closure par boucle : un compteur de pas partagé entre pistes les décale.
  - `loadedSymbols` n'est pas purgé quand `createPlayers` reconstruit les `Tone.Players` : un symbole peut être cru chargé sans buffer.

---

## Instruments

### InstrumentEngine (singleton)

- **Fichier** : `src/renderer/features/instruments/engine/instrument-engine.ts`
- **Type** : Engine (singleton)
- **Méthodes** :
  - `getInstance(): InstrumentEngine`
  - `get isInitialized(): boolean`
  - `loadInstruments(instruments: Instrument[]): void` : remplace le catalogue en mémoire.
  - `getAllSymbols(): string[]`
  - `getInstrumentFilePathsFromSymbol(symbol: string): InstrumentFilePath[]` : **lève une erreur** si le symbole est inconnu.
  - `getInstrumentNameFromSymbol(symbol: string): InstrumentName` : **lève une erreur** si le symbole est inconnu.
  - `getInstrumentReferenceFrequency(symbol: string): number | null` : **lève une erreur** si le symbole est inconnu.
- **Utilisé par** : feature instruments (résolution symbole → nom/chemin).

### instrument-validator.ts

- **Fichier** : `src/renderer/features/instruments/utils/instrument-validator.ts`
- **Type** : Validator (fonction pure, export par défaut)
- **Méthodes** :
  - `validateInstrument(instrument): string[]` : symbole requis, **`@` interdit dans le symbole**, nom requis, fichier audio requis.
- **Utilisé par** : formulaire instrument.

---

## Thème

### themes.ts

- **Fichier** : `src/renderer/features/theme/utils/themes.ts`
- **Type** : Utils (registre pur)
- **Méthodes** :
  - `themes` : liste en lecture seule (Proxy) des 3 thèmes (`blue`, `green`, `pink`) avec `label` et `accentHex`.
  - `getThemeById(id: unknown): ThemeEntry | null` : `null` si inconnu, casse sensible.
  - `isValidThemeId(id: unknown): boolean` / `DEFAULT_THEME` (bleu).
- **Utilisé par** : `ThemeContext`, `ThemeSwitcher`.

### theme-storage.ts

- **Fichier** : `src/renderer/features/theme/utils/theme-storage.ts`
- **Type** : Utils (persistance localStorage)
- **Méthodes** :
  - `getStoredTheme(): ThemeId | null` / `saveTheme(theme: string): void` / `resolveInitialTheme(): ThemeId` (défaut `blue`).
- **Utilisé par** : `ThemeContext`.
- **Pièges** : clé `beat-and-teach:theme` ; toute lecture/écriture en erreur est avalée (retour `null` / no-op) ; `saveTheme` ignore un id invalide ; le script anti-FOUC de `index.ejs` ne pose que `green`/`pink`.

---

## Enregistreur (recorder)

### wav-encoder.ts

- **Fichier** : `src/renderer/features/recorder/utils/wav-encoder.ts`
- **Type** : Utils (fonction pure, export par défaut)
- **Méthodes** :
  - `encodeWav(samples: Float32Array, sampleRate: number): ArrayBuffer` : PCM 16 bits mono, en-tête RIFF/WAVE ; clippe les échantillons à `[-1, 1]`.
- **Utilisé par** : `useRecorder` / sauvegarde d'un enregistrement.

### waveform-renderer.ts

- **Fichier** : `src/renderer/features/recorder/utils/waveform-renderer.ts`
- **Type** : Utils (rendu canvas)
- **Méthodes** :
  - `renderWaveform(options): void` : dessine la forme d'onde avec `trimStart`/`trimEnd`, position de lecture et couleurs explicites.
  - `trimSamples(samples, trimStart, trimEnd): Float32Array` : extrait la portion conservée.
- **Utilisé par** : écran Recorder.
- **Piège** : le canvas ne consomme pas les variables CSS du thème ; les couleurs sont des paramètres (défauts hex dans le fichier).

---

> Note : documenter explicitement les entrées à sémantique piégeuse (nom trompeur, comportement contre-intuitif, dualité de statuts) pour éviter les régressions lors d'une réutilisation.

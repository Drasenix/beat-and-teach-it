# Référence : Utilitaires et méthodes réutilisables

> Fichier maintenu à chaque feature (étapes 3 et 5), sur validation explicite de l'utilisateur.
>
> Objectif : éviter la duplication. Avant d'écrire une nouvelle méthode, vérifier ici si l'équivalent existe déjà.
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
  - `parseToken(raw: string): ParsedToken` : sépare `symbole@fréquence`. Le suffixe est accepté en **Hz numérique** (`P@220`) ou en **nom de note** (`P@A3`) ; sinon `frequency` est absent.
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

## Audio (moteur Tone.js)

### AudioEngine (singleton)
- **Fichier** : `src/renderer/features/audio/engine/audio-engine.ts`
- **Type** : Engine (singleton)
- **Méthodes** :
  - `getInstance(): AudioEngine`
  - `setTempo(bpm: number): void` : BPM du Transport (`8n`).
  - `createPlayers(audioBuffers: AudioFileBuffer): Promise<void>` : décode et charge les buffers.
  - `addToPlayers(audioBuffers: AudioFileBuffer): Promise<void>` : ajoute des buffers sans reset.
  - `createSequence(tracks: SequenceNotes[][]): void` : reconstruit les `Tone.Loop` par piste.
  - `updateSequences(tracks: SequenceNotes[][]): void` : met à jour les notes ; reconstruit les loops si le nombre de pistes change.
  - `play(): Promise<void>` / `stop(): void`.
  - `playInstrument(name: string): Promise<void>` : pré-écoute d'un instrument.
  - `setStepCallback(cb: StepCallback): void` / `clearStepCallback(): void` : callback `(trackIndex, stepIndex)` pour l'UI.
  - `hasSymbol(symbol: string): boolean` / `registerSymbol(symbol: string): void`.
- **Utilisé par** : `AudioContext` / `audio-facade`.

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

---

> Note : documenter explicitement les entrées à sémantique piégeuse (nom trompeur, comportement contre-intuitif, dualité de statuts) pour éviter les régressions lors d'une réutilisation.

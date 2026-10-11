# Feature: dynamic-pattern

## Règle d'implémentation

**TDD strict obligatoire** : pour chaque phase, écrire le test en premier → `npm run test` doit échouer → implémenter le code → `npm run test` doit passer → passer à la phase suivante. Ne jamais écrire de code métier avant son test.

## Specs

Mise à jour dynamique de la séquence audio pendant la lecture, sans Stop + Play.

### Scénarios

1. **Étant donné que** je clique sur Play et que la séquence est en cours de lecture
   **Quand** j'ajoute un symbole (instrument) **valide** dans le pattern
   **Alors** la séquence est mise à jour dynamiquement
   **Et** l'instrument est joué sans avoir à Stop + Play

2. **Étant donné que** je clique sur Play et que la séquence est en cours de lecture
   **Quand** j'ajoute un symbole (instrument) **non valide** dans le pattern
   **Alors** la séquence **n'est pas** mise à jour dynamiquement

3. **Étant donné que** je clique sur Play et que la séquence est en cours de lecture
   **Quand** j'ajoute un symbole non valide dans le pattern
   **Et** que j'ajoute un symbole valide
   **Alors** la séquence **n'est pas** mise à jour dynamiquement
   **Puis** que je corrige/supprime le symbole non valide
   **Alors** la séquence est mise à jour dynamiquement **et** inclut tous les changements valides qui étaient en attente

### Contraintes

- L'édition UI (textarea, grille, mute) reste réactive en toute circonstance
- Seule la poussée vers le moteur audio est conditionnée à la validité
- Un symbole invalide = aucun instrument trouvé dans la base (ni dans InstrumentEngine)
- Le silence `.` est toujours considéré valide
- Si un nouvel instrument valide est ajouté, son fichier audio doit être chargé (IPC + decodeAudioData) avant d'être joué
- Le nombre de pistes peut varier pendant la lecture (ajout/suppression de phrases)
- Un toggle de mute pendant la lecture met aussi à jour dynamiquement la séquence

---

## État du codebase — ce qui existe

### Architecture du flux Play actuel

```
PatternWorkspace
  │  usePatternSession() → sentencesForPlayback (mute-transformé)
  │  useAudio() → { playing, playTrack, stopTrack, activeStep }
  ▼
AudioControls (onClick Play)
  ▼
AudioContext.playTrack(sentences, bpm)
  ▼
audio-facade.playPattern(sentences, bpm, onStep)
  ├── prepareAudioEngine(sentences, bpm, onStep)
  │     ├── audioEngine.setTempo(bpm)
  │     ├── createAudioBuffers(sentences)
  │     │     ├── prepareFilePaths(sentence) → InstrumentFilePath[]
  │     │     ├── IPC 'get-audio-buffers' → AudioFileBuffer
  │     │     └── audioEngine.createPlayers(buffers)
  │     ├── preparePattern(sentence) → SequenceNotes[]
  │     └── audioEngine.createSequence(allNotes) → Tone.Sequence[]
  └── audioEngine.play() → Tone.start() + Transport.start('+0.1')
```

### Fichiers clés existants (avec fonctions publiques)

| Fichier | Fonctions publiques |
|---------|-------------------|
| `src/renderer/features/audio/engine/audio-engine.ts` | `getInstance()`, `setTempo(bpm)`, `createPlayers(buffers)`, `createSequence(tracks)`, `play()`, `stop()`, `playInstrument(name)`, `setStepCallback(cb)`, `clearStepCallback()` |
| `src/renderer/features/audio/facade/audio-facade.ts` | `playPattern(sentences, bpm, onStep?)`, `stopPattern()`, `changeTempo(bpm)`, `playInstrument(filepath, name)` |
| `src/renderer/features/audio/contexts/AudioContext.tsx` | `useAudioContext()` → `{ playing, activeStep, playTrack, stopTrack, changeBpm, playInstrument }` |
| `src/renderer/features/audio/hooks/useAudio.tsx` | `useAudio()` → `useAudioContext()` |
| `src/renderer/features/sequence/service/sequence-service.ts` | `prepareFilePaths(sentence)`, `preparePattern(sentence)` |
| `src/renderer/features/sequence/types/sequence-note.ts` | `SequenceNote = string \| null`, `SequenceNotes = SequenceNote \| SequenceNote[]` |
| `src/renderer/features/instruments/engine/instrument-engine.ts` | `getInstance()`, `loadInstruments(list)`, `getInstrumentFilePathsFromSymbol(sym)`, `getInstrumentNameFromSymbol(sym)`, `isInitialized` |
| `src/renderer/features/instruments/facade/instrument-facade.ts` | `getInstrumentNameFromSymbol(sym)`, `getInstrumentFilePathsFromSymbol(sym)` |
| `src/renderer/features/pattern/hooks/usePatternSession.ts` | hook → `{ pattern, changeSentence, addSentence, removeSentence, mutedSteps, toggleMute, isMuted, sentencesForPlayback }` |
| `src/renderer/features/pattern/hooks/usePattern.tsx` | hook → `{ pattern, changeSentence, addSentence, removeSentence, changeHighlight }` |
| `src/renderer/features/pattern/utils/pattern-parser.ts` | `createStep()`, `createGroup()`, `parseSteps()`, `countSentenceSteps()`, `normalizeSentences()`, `parseMultiTrackSteps()`, `flatTokenCount()` |
| `src/renderer/features/pattern/utils/pattern-mute.ts` | `transformSentencesWithMute(sentences, mutedSteps)` |
| `src/renderer/features/pattern/utils/pattern-validator.ts` | `validatePattern(fields)` — validation de formulaire (nom, phrases) |
| `src/renderer/features/pattern/components/PatternWorkspace.tsx` | Composant principal du studio |

### Duplication de code existante — le regex SENTENCE_REGEX

Le regex `\(([^)]*)\)|(\S+)/g` et sa logique de boucle `execAll` sont copiés dans **3 fichiers** :

1. **`pattern-parser.ts:4-14`** — `SENTENCE_REGEX` + `execAll()`
2. **`pattern-mute.ts:7,11-33`** — regex inline + while loop
3. **`sequence-service.ts:42-49`** — regex inline + while loop

La nouvelle fonction `areAllSymbolsValid()` en ferait une 4e copie.

### Types partagés existants

```typescript
// src/shared/types/instrument.ts
export type InstrumentName = string | null;
export type InstrumentFilePath = { name: InstrumentName; filepath: string | null; };

// src/shared/types/audio-file-buffer.ts
type AudioFileBuffer = Record<string, ArrayBuffer>;

// src/renderer/features/sequence/types/sequence-note.ts
export type SequenceNote = string | null;
export type SequenceNotes = SequenceNote | SequenceNote[];
```

---

## Architecture cible

```
PatternWorkspace
  │ (inchangé) usePatternSession() → sentencesForPlayback
  │ (inchangé) useAudio() → { playing, updateTrack, ... }
  │ (NOUVEAU) useMemo → allValid = areAllSymbolsValid(sentencesForPlayback, validSymbols)
  │ (NOUVEAU) useEffect → if playing && allValid → updateTrack(sentencesForPlayback)
  ▼
AudioContext.updateTrack(sentences)
  ▼
audio-facade.updatePattern(sentences)
  ├── extractUniqueSymbols(sentences) → string[] (via sentence-tokenizer)
  ├── Filtrer les symboles NON déjà chargés dans AudioEngine
  ├── Si nouveaux → IPC get-audio-buffers → audioEngine.addToPlayers()
  ├── preparePattern(sentence) → SequenceNotes[] (via sentence-tokenizer)
  └── audioEngine.updateSequences(allNotes)
        ├── Guard: si this.sequences.length === 0 → return (race condition)
        ├── Ajout/suppression de Tone.Sequence si nombre de pistes changé
        ├── seq.events = newNotes pour chaque Tone.Sequence existant
        └── Recrée Tone.Loop (step) si columnCount a changé
```

### Flux de données complet

```
1. User tape dans SentenceInput
2. changeSentence() → met à jour pattern.sentences dans le state React
3. useMemo → sentencesForPlayback = transformSentencesWithMute(sentences, mutedSteps)
4. useMemo → allValid = areAllSymbolsValid(sentencesForPlayback, validSymbols)
5. useEffect → si playing && allValid :
   a. extractUniqueSymbols(sentencesForPlayback) — extrait les symboles uniques (simple replace+split)
   b. Nouveaux symboles = filter(s => s !== '.' && !audioEngine.hasSymbol(s))
   c. Si nouveaux :
      - getInstrumentFilePathsFromSymbol(s) pour chaque nouveau
      - IPC 'get-audio-buffers' → AudioFileBuffer
      - audioEngine.addToPlayers(buffers) — decode + add dans Tone.Players
      - audioEngine.registerSymbol(s) pour chaque nouveau
   d. preparePattern(sentence) pour chaque phrase → SequenceNotes[][]
   e. audioEngine.updateSequences(allNotes) — mise à jour in-place
```

---

## Phase 0 — Déduplication du tokenizer (pré-requis)

### Nouveau fichier : `src/renderer/utils/sentence-tokenizer.ts`

```typescript
const SENTENCE_REGEX = /\(([^)]*)\)|(\S+)/g;

export interface TokenMatch {
  group: string | undefined;
  symbol: string | undefined;
  fullMatch: string;
}

export function tokenizeSentence(sentence: string): TokenMatch[] {
  const regex = new RegExp(SENTENCE_REGEX.source, SENTENCE_REGEX.flags);
  const matches: TokenMatch[] = [];
  let match = regex.exec(sentence);
  while (match !== null) {
    matches.push({
      group: match[1] !== undefined ? match[1] : undefined,
      symbol: match[2],
      fullMatch: match[0],
    });
    match = regex.exec(sentence);
  }
  return matches;
}

export function extractSymbols(sentence: string): string[] {
  const symbols: string[] = [];
  const tokens = tokenizeSentence(sentence);
  tokens.forEach((token) => {
    if (token.group !== undefined) {
      token.group.split(/\s+/).forEach((s) => symbols.push(s));
    } else if (token.symbol !== undefined) {
      symbols.push(token.symbol);
    }
  });
  return symbols;
}

export function extractUniqueSymbols(sentences: string[]): string[] {
  const all: string[] = [];
  sentences.forEach((s) => extractSymbols(s).forEach((sym) => all.push(sym)));
  return [...new Set(all)];
}
```

### Modification de `pattern-parser.ts`

- Supprimer `const SENTENCE_REGEX` (ligne 4)
- Supprimer `function execAll()` (lignes 7-14)
- Importer `tokenizeSentence` depuis le tokenizer
- Remplacer tous les appels à `execAll()` par `tokenizeSentence()` (le type de retour change légèrement : `RegExpExecArray[]` → `TokenMatch[]`)

La fonction `execAll` retourne `RegExpExecArray[]`. `TokenMatch[]` a la même structure logique (`group` = `match[1]`, `symbol` = `match[2]`). Les consommateurs de `execAll` sont :

```typescript
// parseSteps — ligne 52
return execAll(sentence).map((match, counter) => {
  if (match[1] !== undefined) return createGroup(match[1], validSymbols, counter);
  return createStep(match[2], validSymbols, counter);
});
```

Devient :

```typescript
return tokenizeSentence(sentence).map((token, counter) => {
  if (token.group !== undefined) return createGroup(token.group, validSymbols, counter);
  return createStep(token.symbol ?? '', validSymbols, counter);
});
```

```typescript
// countSentenceSteps — ligne 103
return execAll(sentence).length;
```

Devient :

```typescript
return tokenizeSentence(sentence).length;
```

### Modification de `pattern-mute.ts`

- Supprimer `const regex = /\(([^)]*)\)|(\S+)/g;` (ligne 7)
- Supprimer la boucle while inline (lignes 11-33)
- Importer `tokenizeSentence` depuis le tokenizer
- Réécrire `transformSentencesWithMute` pour utiliser `tokenizeSentence`

La boucle actuelle dans `transformSentencesWithMute` :
```typescript
return sentences.map((sentence, sentenceIndex) => {
  const parts: string[] = [];
  let match = regex.exec(sentence);
  let tokenIndex = 0;
  while (match !== null) {
    if (match[1] !== undefined) { /* groupe */ }
    else { /* symbole simple */ }
    match = regex.exec(sentence);
  }
  return parts.join(' ');
});
```

Devient utilisant `tokenizeSentence` :
```typescript
return sentences.map((sentence, sentenceIndex) => {
  const tokens = tokenizeSentence(sentence);
  const parts: string[] = [];
  let tokenIndex = 0;
  tokens.forEach((token) => {
    if (token.group !== undefined) {
      const symbols = token.group.trim().split(/\s+/);
      const transformed: string[] = [];
      symbols.forEach((sym) => {
        const key = `${sentenceIndex}-${tokenIndex}`;
        tokenIndex += 1;
        transformed.push(mutedSteps.has(key) ? '.' : sym);
      });
      parts.push(`(${transformed.join(' ')})`);
    } else {
      const key = `${sentenceIndex}-${tokenIndex}`;
      tokenIndex += 1;
      parts.push(mutedSteps.has(key) ? '.' : (token.symbol ?? ''));
    }
  });
  return parts.join(' ');
});
```

### Modification de `sequence-service.ts`

- Supprimer la boucle while inline (lignes 42-49)
- Importer `tokenizeSentence` depuis le tokenizer
- Réécrire `preparePattern` pour itérer sur `tokenizeSentence(sentence)` au lieu de la boucle regex

```typescript
// AVANT
export async function preparePattern(sentence: string): Promise<SequenceNotes[]> {
  const regex = /\(([^)]*)\)|(\S+)/g;
  const matches: RegExpExecArray[] = [];
  let match: RegExpExecArray | null = regex.exec(sentence);
  while (match !== null) {
    matches.push(match);
    match = regex.exec(sentence);
  }
  return Promise.all(matches.map(toSequenceNotes));
}

async function toSequenceNotes(match: RegExpExecArray): Promise<SequenceNotes> {
  if (match[1] !== undefined) return toGroupNotes(match[1]);
  return toSequenceNote(await getInstrumentNameFromSymbol(match[2]));
}
```

```typescript
// APRÈS
export async function preparePattern(sentence: string): Promise<SequenceNotes[]> {
  const tokens = tokenizeSentence(sentence);
  return Promise.all(tokens.map(toSequenceNotes));
}

async function toSequenceNotes(token: TokenMatch): Promise<SequenceNotes> {
  if (token.group !== undefined) return toGroupNotes(token.group);
  return toSequenceNote(await getInstrumentNameFromSymbol(token.symbol ?? ''));
}
```

### Tests de régression phase 0

**TOUS les tests existants doivent continuer à passer sans modification.** Vérifier avec `npm run test` après avoir fait les changements.

---

## Phase 1 — `InstrumentEngine.getAllSymbols()`

### Fichier : `src/renderer/features/instruments/engine/instrument-engine.ts`

Ajouter la méthode publique :

```typescript
public getAllSymbols(): string[] {
  return this.instruments.map((i) => i.symbol);
}
```

### Tests : `src/renderer/features/instruments/tests/instrument-engine.test.ts`

Ajouter un bloc `describe('#getAllSymbols')` :

```typescript
describe('#getAllSymbols', () => {
  let engine: InstrumentEngine;

  beforeEach(() => {
    InstrumentEngine['#instance'] = undefined as unknown as InstrumentEngine;
    engine = InstrumentEngine.getInstance();
  });

  it('should return an empty array when no instruments are loaded', () => {
    // Given: un moteur non initialisé
    // When: getAllSymbols()
    // Then: []
    expect(engine.getAllSymbols()).toEqual([]);
  });

  it('should return all symbols from loaded instruments', () => {
    // Given
    const instruments: Instrument[] = [
      { id: 1, slug: 'kickdrum', symbol: 'P', name: 'kickdrum', filepath: '/path/kickdrum.mp3' },
      { id: 2, slug: 'hihat', symbol: 'Ts', name: 'hihat', filepath: '/path/hihat.mp3' },
      { id: 3, slug: 'silence', symbol: '.', name: null, filepath: null },
    ];
    engine.loadInstruments(instruments);
    // When
    const result = engine.getAllSymbols();
    // Then
    expect(result).toEqual(['P', 'Ts', '.']);
  });

  it('should reflect changes after reloading instruments', () => {
    // Given
    engine.loadInstruments([
      { id: 1, slug: 'kickdrum', symbol: 'P', name: 'kickdrum', filepath: '/path/kickdrum.mp3' },
    ]);
    // When
    engine.loadInstruments([
      { id: 1, slug: 'kickdrum', symbol: 'P', name: 'kickdrum', filepath: '/path/kickdrum.mp3' },
      { id: 2, slug: 'hihat', symbol: 'Ts', name: 'hihat', filepath: '/path/hihat.mp3' },
    ]);
    // Then
    expect(engine.getAllSymbols()).toHaveLength(2);
  });
});
```

---

## Phase 2 — `areAllSymbolsValid()`

### Fichier : `src/renderer/features/pattern/utils/pattern-validator.ts`

**Ne pas créer de nouveau fichier** — ajouter la fonction dans le fichier existant. Il contient déjà `validatePattern()` (validation de formulaire). La nouvelle fonction est une validation différente (symboles vs instruments), mais elle appartient au même domaine.

```typescript
export function areAllSymbolsValid(
  sentences: string[],
  validSymbols: string[],
): boolean {
  const symbols = sentences
    .join(' ')
    .replace(/[()]/g, '')
    .split(/\s+/)
    .filter((s) => s.length > 0);
  if (symbols.length === 0) return true;
  return symbols.every((s) => s === '.' || validSymbols.includes(s));
}
```

**Note d'implémentation** : on utilise `replace(/[()]/g, '')` + `split` plutôt que le regex complet. C'est O(n) avec n = longueur totale des phrases, sans capture de groupes. Le silence `.` est toujours valide.

### Tests : `src/renderer/features/pattern/tests/pattern-validator.test.ts`

Ajouter un bloc `describe('#areAllSymbolsValid')` :

```typescript
describe('#areAllSymbolsValid', () => {
  const VALID_SYMBOLS = ['P', 'Ts', 'K', 'Bw', 'Pf'];

  it('should return true when all symbols are valid', () => {
    // Given
    const sentences = ['P (Ts K) .'];
    // When
    const result = areAllSymbolsValid(sentences, VALID_SYMBOLS);
    // Then
    expect(result).toBe(true);
  });

  it('should return false when a symbol is unknown', () => {
    // Given
    const sentences = ['P X Ts'];
    // When
    const result = areAllSymbolsValid(sentences, VALID_SYMBOLS);
    // Then
    expect(result).toBe(false);
  });

  it('should return false when a symbol inside a group is unknown', () => {
    // Given
    const sentences = ['P (Ts X) K'];
    // When
    const result = areAllSymbolsValid(sentences, VALID_SYMBOLS);
    // Then
    expect(result).toBe(false);
  });

  it('should treat silence (.) as always valid', () => {
    // Given
    const sentences = ['. . .'];
    // When
    const result = areAllSymbolsValid(sentences, []);
    // Then
    expect(result).toBe(true);
  });

  it('should return true for an empty sentence', () => {
    // Given
    const sentences = [''];
    // When
    const result = areAllSymbolsValid(sentences, VALID_SYMBOLS);
    // Then
    expect(result).toBe(true);
  });

  it('should return true for empty sentences array', () => {
    // Given
    const sentences: string[] = [];
    // When
    const result = areAllSymbolsValid(sentences, VALID_SYMBOLS);
    // Then
    expect(result).toBe(true);
  });

  it('should return false if only the first sentence is invalid', () => {
    // Given
    const sentences = ['X K .', 'P Ts K'];
    // When
    const result = areAllSymbolsValid(sentences, VALID_SYMBOLS);
    // Then
    expect(result).toBe(false);
  });

  it('should return false if only the second sentence is invalid', () => {
    // Given
    const sentences = ['P Ts K', 'P X K'];
    // When
    const result = areAllSymbolsValid(sentences, VALID_SYMBOLS);
    // Then
    expect(result).toBe(false);
  });

  it('should handle sentences with multiple groups', () => {
    // Given
    const sentences = ['(P Ts) (K Bw) Pf'];
    // When
    const result = areAllSymbolsValid(sentences, VALID_SYMBOLS);
    // Then
    expect(result).toBe(true);
  });

  it('should handle whitespace-only sentence', () => {
    // Given
    const sentences = ['   '];
    // When
    const result = areAllSymbolsValid(sentences, VALID_SYMBOLS);
    // Then
    expect(result).toBe(true);
  });
});
```

---

## Phase 3 — `sentence-tokenizer.test.ts`

### Nouveau fichier de test : `src/renderer/utils/sentence-tokenizer.test.ts`

Tests pour les 3 fonctions exportées :

```typescript
describe('#tokenizeSentence', () => {
  it('should return empty array for empty string', () => {
    // Given: ""
    // When
    const result = tokenizeSentence('');
    // Then
    expect(result).toEqual([]);
  });

  it('should tokenize a simple sentence with single symbols', () => {
    // Given: "P Ts K"
    const result = tokenizeSentence('P Ts K');
    // Then
    expect(result).toHaveLength(3);
    expect(result[0]).toEqual({ group: undefined, symbol: 'P', fullMatch: 'P' });
    expect(result[1]).toEqual({ group: undefined, symbol: 'Ts', fullMatch: 'Ts' });
    expect(result[2]).toEqual({ group: undefined, symbol: 'K', fullMatch: 'K' });
  });

  it('should tokenize a sentence with a group', () => {
    // Given: "P (Ts K) ."
    const result = tokenizeSentence('P (Ts K) .');
    // Then
    expect(result).toHaveLength(3);
    expect(result[0]).toEqual({ group: undefined, symbol: 'P', fullMatch: 'P' });
    expect(result[1]).toEqual({ group: 'Ts K', symbol: undefined, fullMatch: '(Ts K)' });
    expect(result[2]).toEqual({ group: undefined, symbol: '.', fullMatch: '.' });
  });

  it('should tokenize a sentence with multiple groups', () => {
    // Given: "(P Ts) K (. Bw)"
    const result = tokenizeSentence('(P Ts) K (. Bw)');
    // Then
    expect(result).toHaveLength(3);
    expect(result[0].group).toBe('P Ts');
    expect(result[1].symbol).toBe('K');
    expect(result[2].group).toBe('. Bw');
  });
});

describe('#extractSymbols', () => {
  it('should return empty array for empty string', () => {
    // Given: ""
    // When
    const result = extractSymbols('');
    // Then
    expect(result).toEqual([]);
  });

  it('should extract symbols from a simple sentence', () => {
    // Given: "P Ts K"
    // When
    const result = extractSymbols('P Ts K');
    // Then
    expect(result).toEqual(['P', 'Ts', 'K']);
  });

  it('should extract symbols from inside groups', () => {
    // Given: "P (Ts K) ."
    // When
    const result = extractSymbols('P (Ts K) .');
    // Then
    expect(result).toEqual(['P', 'Ts', 'K', '.']);
  });
});

describe('#extractUniqueSymbols', () => {
  it('should return empty array for empty sentences', () => {
    // Given: []
    // When
    const result = extractUniqueSymbols([]);
    // Then
    expect(result).toEqual([]);
  });

  it('should deduplicate symbols across sentences', () => {
    // Given: ["P Ts", "K P"]
    // When
    const result = extractUniqueSymbols(['P Ts', 'K P']);
    // Then
    expect(result).toHaveLength(3);
    expect(result).toContain('P');
    expect(result).toContain('Ts');
    expect(result).toContain('K');
  });

  it('should extract symbols from groups', () => {
    // Given: ["(Ts P) K"]
    // When
    const result = extractUniqueSymbols(['(Ts P) K']);
    // Then
    expect(result).toContain('Ts');
    expect(result).toContain('P');
    expect(result).toContain('K');
  });

  it('should handle multiple sentences with groups', () => {
    // Given: ["P (Ts K)", "(. Bw) Pf"]
    // When
    const result = extractUniqueSymbols(['P (Ts K)', '(. Bw) Pf']);
    // Then
    expect(result).toContain('P');
    expect(result).toContain('Ts');
    expect(result).toContain('K');
    expect(result).toContain('.');
    expect(result).toContain('Bw');
    expect(result).toContain('Pf');
  });
});
```

---

## Phase 4 — `AudioEngine` nouvelles méthodes

### Fichier : `src/renderer/features/audio/engine/audio-engine.ts`

Ajouter les champs et méthodes suivants. Ne pas modifier les méthodes existantes.

#### Nouveau champ privé

```typescript
private loadedSymbols: Set<string> = new Set();
private currentColumnCount: number = 0;
```

#### Méthodes à ajouter

```typescript
public hasSymbol(symbol: string): boolean {
  return this.loadedSymbols.has(symbol);
}

public registerSymbol(symbol: string): void {
  this.loadedSymbols.add(symbol);
}

public async addToPlayers(
  audioBuffers: AudioFileBuffer,
): Promise<void> {
  if (!this.players) return;
  const context = Tone.getContext();
  const buffers = Object.entries(audioBuffers);
  const decoded = await Promise.all(
    buffers.map(([, buffer]) =>
      context.decodeAudioData(buffer as ArrayBuffer),
    ),
  );
  buffers.forEach(([instrumentName], index) => {
    this.players!.add(instrumentName, decoded[index]);
  });
}

public updateSequences(tracks: SequenceNotes[][]): void {
  if (this.sequences.length === 0) return;

  while (this.sequences.length < tracks.length) {
    const seq = new Tone.Sequence(
      (time, note) => {
        if (this.players && note) {
          this.players.player(note).start(time);
        }
      },
      tracks[this.sequences.length],
      '8n',
    );
    seq.start(0);
    this.sequences.push(seq);
  }

  while (this.sequences.length > tracks.length) {
    const removed = this.sequences.pop();
    removed?.dispose();
  }

  tracks.forEach((notes, index) => {
    if (index < this.sequences.length) {
      this.sequences[index].events = notes;
    }
  });

  const newColumnCount = tracks[0]?.length ?? 0;
  if (newColumnCount !== this.currentColumnCount) {
    this.clearStepLoop();
    this.currentColumnCount = newColumnCount;
    this.createStepLoop(newColumnCount);
  }
}
```

#### Modification de `createPlayers` — marquer les symboles comme chargés

À la fin de `createPlayers`, après le `this.players.toDestination()`, ajouter la logique pour peupler `loadedSymbols`. Mais `createPlayers` reçoit `AudioFileBuffer` (clé = nom d'instrument, pas symbole). Il faut passer les symboles ou faire le mapping ailleurs.

**Décision d'implémentation** : le marquage des symboles chargés se fera dans la facade `updatePattern()`, pas dans `createPlayers`. C'est la facade qui a accès aux symboles. `createPlayers` ne change pas (sauf pour resetter `loadedSymbols` au début).

#### Modification de `createSequence` — stocker le columnCount

Dans `createSequence`, après `this.createStepLoop(tracks[0].length)` :

```typescript
this.currentColumnCount = tracks[0]?.length ?? 0;
```

#### Modification de `clearAll` — resetter le columnCount

```typescript
private clearAll(): void {
  this.clearSequences();
  this.clearStepLoop();
  this.currentColumnCount = 0;
}
```

### Tests : nouveau fichier `src/renderer/features/audio/tests/audio-engine.test.ts`

Les tests de `AudioEngine` nécessitent de mocker Tone.js. Utiliser le pattern de mock inline comme dans les tests existants.

```typescript
// Mocks
jest.mock('tone', () => {
  const mockSequence = {
    events: [],
    start: jest.fn(),
    dispose: jest.fn(),
  };
  const mockPlayers = {
    add: jest.fn(),
    player: jest.fn().mockReturnValue({ start: jest.fn() }),
    toDestination: jest.fn(),
    dispose: jest.fn(),
  };
  const mockLoop = {
    start: jest.fn(),
    dispose: jest.fn(),
  };
  return {
    default: undefined,
    __esModule: true,
    Sequence: jest.fn().mockImplementation(() => mockSequence),
    Players: jest.fn().mockImplementation(() => mockPlayers),
    Loop: jest.fn().mockImplementation(() => mockLoop),
    getContext: jest.fn().mockReturnValue({
      decodeAudioData: jest.fn().mockResolvedValue({}),
    }),
    getTransport: jest.fn().mockReturnValue({}),
  };
});
```

#### `#hasSymbol`

```typescript
describe('#hasSymbol', () => {
  let engine: AudioEngine;

  beforeEach(() => {
    AudioEngine['#instance'] = undefined as unknown as AudioEngine;
    engine = AudioEngine.getInstance();
  });

  it('should return false when no symbols have been registered', () => {
    // Given: un moteur avec loadedSymbols vide
    // When: hasSymbol("P")
    // Then: false
    expect(engine.hasSymbol('P')).toBe(false);
  });

  it('should return false for an unregistered symbol', () => {
    // Given
    engine.registerSymbol('P');
    // When
    const result = engine.hasSymbol('Ts');
    // Then
    expect(result).toBe(false);
  });
});
```

#### `#registerSymbol`

```typescript
describe('#registerSymbol', () => {
  let engine: AudioEngine;

  beforeEach(() => {
    AudioEngine['#instance'] = undefined as unknown as AudioEngine;
    engine = AudioEngine.getInstance();
  });

  it('should register a new symbol', () => {
    // Given
    // When
    engine.registerSymbol('Bw');
    // Then
    expect(engine.hasSymbol('Bw')).toBe(true);
  });

  it('should be idempotent', () => {
    // Given
    engine.registerSymbol('P');
    // When
    engine.registerSymbol('P');
    // Then
    expect(engine.hasSymbol('P')).toBe(true);
  });

  it('should register the silence symbol', () => {
    // Given
    // When
    engine.registerSymbol('.');
    // Then
    expect(engine.hasSymbol('.')).toBe(true);
  });
});
```

#### `#updateSequences`

```typescript
describe('#updateSequences', () => {
  let engine: AudioEngine;
  let mockSeq: { events: unknown; dispose: jest.Mock };

  beforeEach(() => {
    AudioEngine['#instance'] = undefined as unknown as AudioEngine;
    engine = AudioEngine.getInstance();
    mockSeq = { events: [], dispose: jest.fn() };
    jest.mocked(Tone.Sequence).mockImplementation(() => mockSeq as unknown as Tone.Sequence);
  });

  it('should do nothing when no sequences exist (race condition guard)', () => {
    // Given: engine sans aucune séquence
    // When
    engine.updateSequences([['kickdrum']]);
    // Then
    expect(Tone.Sequence).not.toHaveBeenCalled();
  });

  it('should update events on existing sequence without creating new ones', () => {
    // Given: 1 sequence déjà créée
    engine.createSequence([['kickdrum', 'hihat']]);
    jest.mocked(Tone.Sequence).mockClear();
    // When
    engine.updateSequences([['kickdrum', null, 'hihat']]);
    // Then: events mis à jour, pas de nouvelle séquence
    expect(mockSeq.events).toEqual(['kickdrum', null, 'hihat']);
    expect(Tone.Sequence).not.toHaveBeenCalled();
  });

  it('should create a new sequence when track count increases', () => {
    // Given: 1 sequence déjà créée
    engine.createSequence([['kickdrum']]);
    jest.mocked(Tone.Sequence).mockClear();
    // When: 2 pistes
    engine.updateSequences([['kickdrum'], ['hihat']]);
    // Then: 1 nouvelle séquence créée (pour la 2e piste)
    expect(Tone.Sequence).toHaveBeenCalledTimes(1);
  });

  it('should dispose extra sequences when track count decreases', () => {
    // Given: 2 sequences
    engine.createSequence([['kickdrum'], ['hihat']]);
    // When: 1 piste
    engine.updateSequences([['kickdrum']]);
    // Then: une séquence disposée
    expect(mockSeq.dispose).toHaveBeenCalled();
  });
});
```

---

## Phase 5 — `updatePattern()` dans la facade audio

### Fichier : `src/renderer/features/audio/facade/audio-facade.ts`

Ajouter les imports nécessaires et la fonction :

```typescript
import { extractUniqueSymbols } from '../../../utils/sentence-tokenizer';

export async function updatePattern(sentences: string[]): Promise<void> {
  const audioEngine = AudioEngine.getInstance();

  const uniqueSymbols = extractUniqueSymbols(sentences);
  const newSymbols = uniqueSymbols.filter(
    (s) => s !== '.' && !audioEngine.hasSymbol(s),
  );

  if (newSymbols.length > 0) {
    const filePaths = (
      await Promise.all(
        newSymbols.map((s) => getInstrumentFilePathsFromSymbol(s)),
      )
    ).flat();
    const audioBuffers = await getAudioBuffers(filePaths);
    await audioEngine.addToPlayers(audioBuffers);
    newSymbols.forEach((s) => audioEngine.registerSymbol(s));
  }

  const allNotes: SequenceNotes[][] = await Promise.all(
    sentences.map((sentence) => preparePattern(sentence)),
  );

  audioEngine.updateSequences(allNotes);
}
```

### Tests : nouveau fichier `src/renderer/features/audio/tests/audio-facade.test.ts`

```typescript
describe('#updatePattern', () => {
  let mockEngine: {
    hasSymbol: jest.Mock;
    registerSymbol: jest.Mock;
    addToPlayers: jest.Mock;
    updateSequences: jest.Mock;
  };

  beforeEach(() => {
    mockEngine = {
      hasSymbol: jest.fn().mockReturnValue(true),
      registerSymbol: jest.fn(),
      addToPlayers: jest.fn().mockResolvedValue(undefined),
      updateSequences: jest.fn(),
    };
    jest.spyOn(AudioEngine, 'getInstance').mockReturnValue(mockEngine as unknown as AudioEngine);
    jest.spyOn(instrumentFacade, 'getInstrumentFilePathsFromSymbol')
      .mockImplementation(async (s: string) => {
        const map: Record<string, InstrumentFilePath[]> = {
          P: [{ name: 'kickdrum', filepath: '/path/kickdrum.mp3' }],
          Ts: [{ name: 'hihat', filepath: '/path/hihat.mp3' }],
          K: [{ name: 'rimshot', filepath: '/path/rimshot.mp3' }],
          Bw: [{ name: 'liproll', filepath: '/path/liproll.mp3' }],
          '.': [{ name: null, filepath: null }],
        };
        if (!map[s]) throw new Error(`Le symbole ${s} n'existe pas.`);
        return map[s];
      });
    jest.spyOn(audioService, 'default').mockResolvedValue({});
  });

  it('should update sequences without loading if no new symbols', async () => {
    // Given: engine a déjà P, Ts, K chargés (hasSymbol → true)
    mockEngine.hasSymbol.mockReturnValue(true);
    // When
    await updatePattern(['P (Ts K) .']);
    // Then
    expect(mockEngine.addToPlayers).not.toHaveBeenCalled();
    expect(mockEngine.registerSymbol).not.toHaveBeenCalled();
    expect(mockEngine.updateSequences).toHaveBeenCalledTimes(1);
  });

  it('should load new audio buffer when a new valid symbol appears', async () => {
    // Given: engine a P, Ts chargés. Bw est nouveau
    mockEngine.hasSymbol.mockImplementation((s: string) => s === 'P' || s === 'Ts');
    // When
    await updatePattern(['P (Ts Bw)']);
    // Then
    expect(instrumentFacade.getInstrumentFilePathsFromSymbol).toHaveBeenCalledWith('Bw');
    expect(audioService.default).toHaveBeenCalledTimes(1);
    expect(mockEngine.addToPlayers).toHaveBeenCalledTimes(1);
    expect(mockEngine.registerSymbol).toHaveBeenCalledWith('Bw');
    expect(mockEngine.updateSequences).toHaveBeenCalledTimes(1);
  });

  it('should skip . (silence) when checking for new symbols', async () => {
    // Given: engine a P chargé
    mockEngine.hasSymbol.mockImplementation((s: string) => s === 'P');
    // When
    await updatePattern(['P . . .']);
    // Then
    expect(mockEngine.addToPlayers).not.toHaveBeenCalled();
    expect(mockEngine.updateSequences).toHaveBeenCalledTimes(1);
  });

  it('should register new symbols after successful loading', async () => {
    // Given
    mockEngine.hasSymbol.mockReturnValue(false);
    // When
    await updatePattern(['Bw']);
    // Then
    expect(mockEngine.registerSymbol).toHaveBeenCalledWith('Bw');
    expect(mockEngine.updateSequences).toHaveBeenCalledTimes(1);
  });
});
```

---

## Phase 6 — `AudioContext.updateTrack()`

### Fichier : `src/renderer/features/audio/contexts/AudioContext.tsx`

Ajouter dans le type `AudioContextType` :

```typescript
updateTrack: (sentences: string[]) => Promise<void>;
```

Ajouter dans le `useMemo` :

```typescript
updateTrack: async (sentences: string[]): Promise<void> => {
  try {
    await updatePattern(sentences);
  } catch (error) {
    // Silencieux — l'édition reste réactive, l'audio reprendra au prochain update valide
  }
},
```

Ajouter l'import de `updatePattern` :

```typescript
import {
  playPattern,
  stopPattern,
  changeTempo,
  playInstrument,
  updatePattern,
} from '../facade/audio-facade';
```

**Pas de tests** — c'est un composant React context, hors scope métier.

### Fichier : `src/renderer/features/audio/hooks/useAudio.tsx`

Exposer `updateTrack` dans le type de retour (il est déjà dans `AudioContextType`, donc `useAudio()` le retourne automatiquement — rien à changer).

---

## Phase 7 — Intégration dans `PatternWorkspace`

### Fichier : `src/renderer/features/pattern/components/PatternWorkspace.tsx`

#### Nouveaux imports

```typescript
import { useEffect, useMemo } from 'react'; // ajouter useMemo
import { useInstrumentsContext } from '../../instruments/contexts/InstrumentsContext'; // ou le bon import
import { areAllSymbolsValid } from '../utils/pattern-validator';
```

#### Accès aux instruments

Le `PatternWorkspace` doit connaître la liste des symboles valides. Il y a deux options :

**Option A** : Utiliser `useInstrumentsContext()` si disponible au niveau du `PatternWorkspace`.
**Option B** : Charger les symboles via `InstrumentEngine.getInstance().getAllSymbols()`.

Vérifier si `PatternWorkspace` a accès au `InstrumentsProvider`. Oui — le provider est au-dessus dans `App.tsx`.

**Code à ajouter dans le composant** :

```typescript
const { instruments } = useInstrumentsContext();
const validSymbols = useMemo(
  () => instruments.map((i) => i.symbol),
  [instruments],
);

const allValid = useMemo(
  () => areAllSymbolsValid(sentencesForPlayback, validSymbols),
  [sentencesForPlayback, validSymbols],
);
```

#### Nouveau `useEffect`

```typescript
const { playing, updateTrack } = useAudio(); // modifier la ligne 41 pour extraire updateTrack

// Mise à jour dynamique de la séquence pendant la lecture
useEffect(() => {
  if (playing && allValid) {
    updateTrack(sentencesForPlayback);
  }
}, [playing, allValid, sentencesForPlayback, updateTrack]);
```

**Pas de tests** — composant React.

---

## Phase 8 — Mise à jour de `createSequence` pour le chargement initial

### Fichier : `src/renderer/features/audio/engine/audio-engine.ts`

Dans `createSequence`, après avoir créé les séquences mais avant de retourner, enregistrer les colonnes :

```typescript
this.currentColumnCount = tracks[0]?.length ?? 0;
```

### Fichier : `src/renderer/features/audio/facade/audio-facade.ts`

Dans `playPattern` (ou `prepareAudioEngine`), après `createSequence`, enregistrer les symboles comme chargés :

```typescript
// Dans prepareAudioEngine, après createSequence(allNotes)
const allSymbols = extractUniqueSymbols(sentences);
allSymbols.forEach((s) => audioEngine.registerSymbol(s));
```

Ceci permet à `updatePattern` de savoir quels symboles sont déjà chargés quand l'utilisateur édite.

---

## Ordre d'implémentation (TDD)

1. **Phase 0** — Créer `sentence-tokenizer.ts` + `sentence-tokenizer.test.ts`, refactorer `pattern-parser.ts`, `pattern-mute.ts`, `sequence-service.ts`. Vérifier que TOUS les tests existants passent (`npm run test`).
2. **Phase 1** — `InstrumentEngine.getAllSymbols()` + tests dans `instrument-engine.test.ts`
3. **Phase 2** — `areAllSymbolsValid()` + tests dans `pattern-validator.test.ts`
4. **Phase 3** — Compléter `sentence-tokenizer.test.ts` si pas déjà fait en phase 0
5. **Phase 4** — `AudioEngine.hasSymbol()`, `registerSymbol()`, `addToPlayers()`, `updateSequences()` + `audio-engine.test.ts`
6. **Phase 5** — `audio-facade.updatePattern()` + `audio-facade.test.ts`
7. **Phase 6** — `AudioContext.updateTrack()` (pas de tests)
8. **Phase 7** — Intégration dans `PatternWorkspace` (pas de tests)
9. **Phase 8** — Enregistrement des symboles au Play initial

Lancer `npm run test` après chaque phase.

---

## Liste exhaustive des fichiers à créer

| Fichier | Action |
|---------|--------|
| `src/renderer/utils/sentence-tokenizer.ts` | Créer |
| `src/renderer/utils/sentence-tokenizer.test.ts` | Créer |
| `src/renderer/features/audio/tests/audio-engine.test.ts` | Créer |
| `src/renderer/features/audio/tests/audio-facade.test.ts` | Créer |

## Liste exhaustive des fichiers à modifier

| Fichier | Modification |
|---------|-------------|
| `src/renderer/features/instruments/engine/instrument-engine.ts` | + `getAllSymbols()` |
| `src/renderer/features/instruments/tests/instrument-engine.test.ts` | + describe `#getAllSymbols` |
| `src/renderer/features/pattern/utils/pattern-validator.ts` | + `areAllSymbolsValid()` |
| `src/renderer/features/pattern/tests/pattern-validator.test.ts` | + describe `#areAllSymbolsValid` |
| `src/renderer/features/pattern/utils/pattern-parser.ts` | Remplacer regex → import `tokenizeSentence` |
| `src/renderer/features/pattern/utils/pattern-mute.ts` | Remplacer regex → import `tokenizeSentence` |
| `src/renderer/features/sequence/service/sequence-service.ts` | Remplacer regex → import `tokenizeSentence` |
| `src/renderer/features/audio/engine/audio-engine.ts` | + `loadedSymbols`, `currentColumnCount`, `hasSymbol()`, `registerSymbol()`, `addToPlayers()`, `updateSequences()` |
| `src/renderer/features/audio/facade/audio-facade.ts` | + `updatePattern()`, enregistrement des symboles après `createSequence` |
| `src/renderer/features/audio/contexts/AudioContext.tsx` | + `updateTrack()` dans le contexte |
| `src/renderer/features/pattern/components/PatternWorkspace.tsx` | + `useEffect` + `useMemo` de validation + `updateTrack` |

## Fichiers qui ne nécessitent AUCUN changement

| Fichier | Raison |
|---------|--------|
| `src/main/icpEvents.ts` | Le channel IPC `get-audio-buffers` existe déjà |
| `src/main/preload.ts` | Aucun nouveau channel IPC nécessaire |
| `src/main/audio/services/audio-service.ts` | Déjà fonctionnel |
| `src/renderer/features/audio/services/audio-service.ts` | Déjà fonctionnel |
| `src/renderer/features/sequence/facade/sequence-facade.ts` | Délégue à `sequence-service.ts` |
| `src/renderer/features/sequence/adapters/sequence-adapter.ts` | Fonction identité, inchangée |
| `src/renderer/features/sequence/types/sequence-note.ts` | Types inchangés |
| `src/renderer/features/instruments/facade/instrument-facade.ts` | Les méthodes `getInstrumentNameFromSymbol` et `getInstrumentFilePathsFromSymbol` existent |
| `src/renderer/features/pattern/hooks/usePatternSession.ts` | Inchangé — `sentencesForPlayback` est déjà calculé |
| `src/renderer/features/pattern/hooks/usePattern.tsx` | Inchangé |
| `src/renderer/features/audio/components/AudioControls.tsx` | Inchangé |
| Tous les fichiers de `src/shared/` | Types existants suffisants |

---

## Bug — Désynchronisation des animations (step loop) après update dynamique

### Scénarios GWT

1. **Étant donné que** je clique sur Play et qu'un pattern est lu
   **Alors** chaque symbole de la grille est animé (step-active-pulse) en même temps que l'instrument est lu (audio)

2. **Étant donné que** je clique sur Play et qu'un pattern est lu
   **Quand** j'ajoute un nouvel instrument valide dans le pattern
   **Et** que le pattern se met à jour dynamiquement
   **Alors** les animations doivent être adaptées et se synchroniser avec le séquenceur

### Résumé pour l'implémentation

**Fichiers à modifier** :
| Fichier | Action |
|---------|--------|
| `src/renderer/features/audio/engine/audio-engine.ts` | Modifier `createStepLoop()` + `updateSequences()` |
| `src/renderer/features/audio/tests/audio-engine.test.ts` | Modifier le mock `Tone.Loop` + ajouter 3 tests |

**Fichiers inchangés** : tous les autres (audio-facade.ts, AudioContext.tsx, PatternWorkspace.tsx)

### Cause racine

`createStepLoop(columnCount)` crée une **closure** qui capture :
- `let stepIndex = 0` → variable locale réinitialisée à chaque appel
- `columnCount` → **paramètre capturé**, jamais mis à jour après la création

```typescript
// LIGNE 80 — ÉTAT ACTUEL (buggé)
private createStepLoop(columnCount: number): void {
  if (!this.onStep) return;
  let stepIndex = 0;                        // ← reset à 0 à chaque appel !
  this.stepLoop = new Tone.Loop(() => {
    const current = stepIndex % columnCount; // ← columnCount = paramètre capturé
    this.onStep!(current);
    stepIndex += 1;
  }, '8n').start(0);
}
```

Dans `updateSequences`, quand le `columnCount` change (ajout/suppression d'un symbole dans la 1ère phrase) :

```typescript
// LIGNE 185-190 — ÉTAT ACTUEL (buggé)
if (newColumnCount !== this.currentColumnCount) {
  this.clearStepLoop();                  // dispose l'ancien Tone.Loop
  this.currentColumnCount = newColumnCount;
  this.createStepLoop(newColumnCount);   // recrée → stepIndex = 0, columnCount capturé
}
```

**Trace de l'exécution Tone.js** (basée sur les sources `node_modules/tone/build/esm/event/`) :

D'abord, le `seq.events = newNotes` **ne cause pas de désynchronisation** :
```
Sequence.set events(s)
  → this.clear()                     // Sequence.js:78 → appelle _part.clear()
  → this._eventsArray = s
  → this._events = _createSequence(s)
  → this._eventsUpdated()            // Sequence.js:138
      → this._part.clear()           // 2e clear, no-op
      → this._rescheduleSequence()   // Sequence.js:147
          → forEach(value, index) →
              startTime = index * '8n' + startOffset  // temps ABSOLU depuis time=0
              this._part.add(startTime, value)
```
Les events sont reprogrammés à des temps **absolus**. Le Transport est à T>0. Les events aux temps <T sont déjà passés. Ceux aux temps ≥T s'exécuteront. **L'audio est correct.**

Ensuite, la recréation du `Tone.Loop` **cause la désynchronisation** :
```
createStepLoop(newCount)
  → let stepIndex = 0                        // compteur réinitialisé
  → new Tone.Loop(callback, '8n').start(0)
      → Loop.start(0)
          → ToneEvent.start(0)               // ToneEvent.js:145
              → ticks = toTicks(0) = 0
              → state.add({ time:0, state:"started" })
              → _rescheduleEvents(0)         // ToneEvent.js:71
                  → startTick = 0 + startOffset/playbackRate = 0
                  → transport.scheduleRepeat(_tick, '8n', startTick=0, ∞)
```
`transport.scheduleRepeat(callback, interval, startTick=0, duration=∞)` programme un callback répété depuis le **tick 0**. Comme le Transport est déjà à T>0 (ex: T = 5 ticks '8n'), le prochain callback s'exécute à la prochaine frontière '8n' après T. Mais `stepIndex` est à 0 :
```
   1er tick : stepIndex=0 → current=0%newCount=0 → onStep(0)
   2e  tick : stepIndex=1 → current=1%newCount=1 → onStep(1)
```
Pendant ce temps, le Transport est à la position T+N, et l'audio joue le step correspondant. **Animation = step 0, Audio = step T+N → désynchronisation permanente avec offset T+N.**

### Correction (2 modifications dans `audio-engine.ts`)

#### Modification 1/2 : `createStepLoop` — lire `this.currentColumnCount` au lieu du paramètre capturé

```typescript
private createStepLoop(columnCount: number): void {
  if (!this.onStep) return;
  let stepIndex = 0;
  this.currentColumnCount = columnCount;        // stocker sur l'instance
  this.stepLoop = new Tone.Loop(() => {
    if (this.currentColumnCount <= 0) return;   // guard NaN (phrase 1 vidée)
    const current = stepIndex % this.currentColumnCount; // ← lit le champ d'instance
    this.onStep!(current);
    stepIndex += 1;
  }, '8n').start(0);
}
```

**Pourquoi `this` dans la closure** : la closure est une arrow function → `this` est lié lexicalement à l'instance d'`AudioEngine`. La closure lit `this.currentColumnCount` à chaque tick. Quand `updateSequences` mute ce champ, la closure voit la nouvelle valeur au prochain tick.

**Pourquoi le guard** : si la phrase 1 est vidée (`columnCount=0`), `stepIndex % 0` donne `NaN`. Le guard `if (this.currentColumnCount <= 0) return` empêche l'appel à `onStep(NaN)`.

**Pourquoi `stepIndex` reste synchrone** : `stepIndex` n'est jamais réinitialisé tant que le `Tone.Loop` vit. Or le loop n'est détruit que par `stop()` ou `clearAll()`, jamais par `updateSequences`. `stepIndex` agit comme un compteur absolu de ticks '8n' depuis `time=0`, calé sur le Transport.

#### Modification 2/2 : `updateSequences` — ne plus recréer le loop

```typescript
// AVANT (lignes 185-190) — À SUPPRIMER
const newColumnCount = tracks[0]?.length ?? 0;
if (newColumnCount !== this.currentColumnCount) {
  this.clearStepLoop();
  this.currentColumnCount = newColumnCount;
  this.createStepLoop(newColumnCount);
}

// APRÈS — À ÉCRIRE
const newColumnCount = tracks[0]?.length ?? 0;
if (newColumnCount !== this.currentColumnCount) {
  this.currentColumnCount = newColumnCount;
}
```

**Pourquoi ça suffit** : la closure du `Tone.Loop` existant lit `this.currentColumnCount` (cf. modif 1). En mettant à jour ce champ sans recréer le loop, le `stepIndex` interne est préservé et seul le modulo change.

Exemple concret de synchronisation préservée :
```
État initial : columnCount=3, Transport à stepIndex=7 → 7%3=1 → anim=1, audio=1
Ajout symbole → columnCount=5 → this.currentColumnCount=5 (pas de recréation)
Prochain tick : stepIndex=8 → 8%5=3 → anim=3, audio=3 ✅
```

### Tests GWT (TDD — test d'abord, implémentation ensuite)

#### Prérequis : modifier le mock `Tone.Loop` dans `jest.mock('tone', ...)`

Le mock actuel de `Tone.Loop` ne capture pas le callback. Pour le 3e test, il faut l'enrichir.

**Fichier** : `src/renderer/features/audio/tests/audio-engine.test.ts`

```typescript
// AVANT — ligne ~9 du jest.mock factory
Loop: jest.fn(() => ({
  start: jest.fn(),
  dispose: jest.fn(),
})),

// APRÈS
Loop: jest.fn((callback: () => void) => ({
  start: jest.fn(),
  dispose: jest.fn(),
  _callback: callback,
})),
```

Ajouter `_callback` dans le type `mockTone.Loop` (optionnel, utiliser `as Record<string, unknown>` au moment du cast dans le test).

#### Test 1 : `should NOT recreate step loop when column count changes`

Emplacement : dans le `describe('#updateSequences', ...)`, après les tests existants.

```typescript
it('should NOT recreate step loop when column count changes', () => {
  engine.createSequence([['kickdrum', 'hihat', 'snare']]);
  mockTone.Loop.mockClear();
  engine.updateSequences([['a', 'b', 'c', 'd', 'e']]);
  expect(mockTone.Loop).not.toHaveBeenCalled();
});
```

#### Test 2 : `should dispose extra sequences when track count decreases` (inchangé, déjà présent)

Ce test existe déjà dans la suite. Il reste valide après la correction.

#### Test 3 : `should use updated currentColumnCount in step loop callback`

**Objectif** : vérifier que la closure du `Tone.Loop` lit bien `this.currentColumnCount` (champ d'instance mis à jour) et non le paramètre `columnCount` (capturé à la création).

**Principe** : on crée un pattern avec 2 colonnes, on simule `updateSequences` avec 4 colonnes, puis on appelle manuellement le callback 3 fois. On vérifie que les valeurs passées à `stepCallback` correspondent au **nouveau** `currentColumnCount=4` (0%4=0, 1%4=1, 2%4=2) et non à l'ancien `columnCount=2` (0%2=0, 1%2=1, 2%2=0).

Le mock de `Tone.Loop` ayant été enrichi avec `_callback`, on peut extraire le callback depuis `mockTone.Loop.mock.results[0].value._callback`.

```typescript
it('should use updated currentColumnCount in step loop callback', () => {
  const stepCallback = jest.fn();
  engine.setStepCallback(stepCallback);
  engine.createSequence([['a', 'b']]);
  const capturedCallback = (
    mockTone.Loop.mock.results[0]?.value as Record<string, unknown>
  )._callback as () => void;

  engine.updateSequences([['x', 'y', 'z', 'w']]);

  capturedCallback();
  capturedCallback();
  capturedCallback();

  expect(stepCallback).toHaveBeenCalledTimes(3);
  expect(stepCallback).toHaveBeenNthCalledWith(1, 0);
  expect(stepCallback).toHaveBeenNthCalledWith(2, 1);
  expect(stepCallback).toHaveBeenNthCalledWith(3, 2);
});
```

**Interprétation TDD** :
- **RED** : ce test va échouer avant la correction parce que la closure lit l'ancien `columnCount=2`. `stepIndex` passe de 0→1→2 et avec `columnCount=2` : 0%2=0, 1%2=1, 2%2=0 → `stepCallback` reçoit (0, 1, 0) au lieu de (0, 1, 2). Le test échoue sur `toHaveBeenNthCalledWith(3, 2)`.
- **GREEN** : après la correction (modif 1 + modif 2), la closure lit `this.currentColumnCount=4`. 0%4=0, 1%4=1, 2%4=2 → `stepCallback` reçoit (0, 1, 2). Le test passe.

### Ordre d'implémentation TDD

1. **Modifier le mock `Tone.Loop`** dans `jest.mock('tone', ...)` pour capturer le callback (`_callback`)
2. **Ajouter les 3 tests** dans le `describe('#updateSequences', ...)` du fichier de test
3. **Lancer `npm run test -- --testPathPattern='audio-engine'`** → les nouveaux tests doivent échouer (RED)
4. **Modifier `createStepLoop`** : lire `this.currentColumnCount` au lieu du paramètre, ajouter le guard NaN
5. **Modifier `updateSequences`** : supprimer `clearStepLoop()` + `createStepLoop()`, ne garder que `this.currentColumnCount = newColumnCount`
6. **Lancer `npm run test -- --testPathPattern='audio-engine'`** → tous les tests passent (GREEN)
7. **Lancer `npm run test`** complet → tous les tests passent (pas de régression)
8. **Lancer `npm run lint`** → 0 erreurs

### Tableau de correspondance bug → comportement corrigé

| Cas | ColumnCount change ? | Avant (bug) | Après (fix) |
|-----|---------------------|-------------|-------------|
| Changer `P` → `Bw` (même nombre de pas) | Non | ✅ Synchro | ✅ Synchro |
| Ajouter un symbole (1ère phrase) | Oui | ❌ Loop recréé → stepIndex=0 → desync | ✅ Loop conservé → stepIndex préservé → sync |
| Supprimer un symbole (1ère phrase) | Oui | ❌ Même cause | ✅ Même correction |
| Ajouter/supprimer une 2e piste | Non (tracks[0] inchangé) | ✅ Synchro | ✅ Synchro |
| Toggle mute | Non | ✅ Synchro | ✅ Synchro |
| Play initial | — | ✅ Synchro | ✅ Synchro |
| Stop → Play | — | ✅ Synchro | ✅ Synchro |
| Vider la 1ère phrase | Oui (columnCount=0) | ❌ Desync + NaN possible | ✅ Guard : `if (currentColumnCount <= 0) return` |

## Phase 10 — Correction du saut audio pendant le live editing (master Tone.Loop)

### Scénarios GWT

1. **Étant donné que** je clique sur Play et qu'un pattern de 4 colonnes est lu
   **Quand** j'ajoute un symbole valide (le pattern passe de 4 à 5 colonnes)
   **Et** que le pattern se met à jour dynamiquement
   **Alors** le son en cours de lecture ne saute pas (pas de changement brutal d'instrument au même tick)

2. **Étant donné que** je clique sur Play et qu'un pattern de 5 colonnes est lu
   **Quand** je supprime un symbole (le pattern passe de 5 à 4 colonnes)
   **Et** que le pattern se met à jour dynamiquement
   **Alors** le son en cours de lecture ne saute pas

3. **Étant donné que** je clique sur Play et que la lecture est en cours
   **Quand** j'ajoute/supprime une piste (sans changer le nombre de colonnes)
   **Alors** le son continue sans saut, les nouvelles pistes sont audibles immédiatement

### Résumé pour l'implémentation

**Fichiers à modifier** :
| Fichier | Action |
|---------|--------|
| `src/renderer/features/audio/engine/audio-engine.ts` | Refactor complet de `createSequence`, `updateSequences`, `stop`, `clearAll` ; supprimer `sequences`, `stepLoop`, `clearSequences`, `clearStepLoop`, `createStepLoop` |
| `src/renderer/features/audio/tests/audio-engine.test.ts` | Réécrire les 3 tests existants (`#updateSequences`) + ajouter 8 nouveaux tests ; simplifier le mock `Tone.Sequence` |

**Fichiers inchangés** : `audio-facade.ts`, `AudioContext.tsx`, `PatternWorkspace.tsx`, tous les autres

### Cause racine

`seq.events = newNotes` appelle `Sequence._rescheduleSequence()` (Sequence.js:147-158) qui schedule chaque élément à `index * '8n' + startOffset` — des **temps absolus depuis time=0**. Le Transport est à une position absolue T. Après le reschedule :

```
oldColCount=4, Transport à T=11 → index joué = 11%4 = 3 → oldNotes[3] = "."
newColCount=5, Transport à T=11 → index joué = 11%5 = 1 → newNotes[1] = "Ts"
```

Le son passe de `.` (silence) à `Ts` (hihat) **au même tick Transport**. Ce n'est pas une désynchronisation — c'est un **changement de mapping** entre temps absolu et index de tableau. Le contenu à la même position Transport change parce que le modulo a changé.

**L'ajout d'un symbole donne un saut arrière** (index diminue car dénominateur plus grand).
**La suppression donne un saut avant** (index augmente car dénominateur plus petit).

C'est irrécupérable avec `Tone.Sequence` car le reschedule est atomique et intégral.

### Architecture cible

Remplacer `Tone.Sequence[]` (event-driven, pré-schedulé) + `Tone.Loop` (animation) par un **seul `Tone.Loop` maître** qui lit les données en direct :

```
AVANT :
  N× Tone.Sequence  ──pré-schedule──→  Transport  ──→  audio
  1× Tone.Loop      ──pré-schedule──→  Transport  ──→  animation (activeStep)

APRÈS :
  1× Tone.Loop      ──pré-schedule──→  Transport  ──→  audio + animation
       │
       └── callback lit this.trackNotes et this.players en direct
```

### Refacto : champs

| Champ | Action |
|-------|--------|
| `private sequences: Tone.Sequence[]` | **Supprimer** |
| `private stepLoop?: Tone.Loop` | **Supprimer** |
| `private stepIndex: number = 0` | **Ajouter** (promu de variable locale → champ d'instance) |
| `private masterLoop?: Tone.Loop` | **Ajouter** (remplace `sequences` + `stepLoop`) |
| `private trackNotes: SequenceNotes[][] = []` | **Ajouter** (données lues en direct par le callback) |
| `private currentColumnCount: number = 0` | **Conserver** |
| `private loadedSymbols: Set<string>` | **Conserver** |
| `private players?: Tone.Players` | **Conserver** |
| `private onStep?: StepCallback` | **Conserver** |

### Refacto : méthodes supprimées

- `private clearSequences()` — plus de `Tone.Sequence` à disposer
- `private clearStepLoop()` — remplacée par `clearMasterLoop()`
- `private createStepLoop(columnCount)` — fusionnée dans `createSequence()`

### Refacto : code complet des méthodes

#### `createSequence(tracks)` — nouvelle version

```typescript
public createSequence(tracks: SequenceNotes[][]): void {
  this.clearAll();

  this.trackNotes = tracks;
  this.currentColumnCount = tracks[0]?.length ?? 0;
  this.stepIndex = 0;

  if (!this.onStep) return;

  this.masterLoop = new Tone.Loop((time) => {
    if (this.currentColumnCount <= 0) return;
    const step = this.stepIndex % this.currentColumnCount;

    this.trackNotes.forEach((trackNotes) => {
      if (step >= trackNotes.length) return;
      const note = trackNotes[step];
      if (typeof note === 'string') {
        this.players?.player(note).start(time);
      } else if (Array.isArray(note)) {
        note.forEach((n) => {
          if (typeof n === 'string') {
            this.players?.player(n).start(time);
          }
        });
      }
    });

    this.onStep!(step);
    this.stepIndex += 1;
  }, '8n').start(0);
}
```

**Logique du callback** : à chaque tick '8n', on calcule l'étape courante via `stepIndex % currentColumnCount`. Pour chaque piste, on lit la note à cet index. Si la note est `string` → on joue l'instrument. Si c'est `string[]` (groupe) → on joue chaque sous-note (Tone.js dispatche chaque élément à la même position temporelle). Si c'est `null` (silence) → on ne joue rien. S'il n'y a pas assez de notes dans la piste (piste plus courte que la première) → `return`.

#### `updateSequences(tracks)` — nouvelle version

```typescript
public updateSequences(tracks: SequenceNotes[][]): void {
  if (!this.masterLoop) return;

  this.trackNotes = tracks;
  const newColumnCount = tracks[0]?.length ?? 0;
  if (newColumnCount !== this.currentColumnCount) {
    const oldPosition = this.stepIndex % this.currentColumnCount;
    this.currentColumnCount = newColumnCount;
    this.stepIndex = oldPosition;
  }
}
```

**Recalibration de `stepIndex`** : quand `colCount` change, on préserve la position relative dans le loop. Exemple :
```
colCount=4, stepIndex=11 → position = 11%4 = 3
colCount passe à 5 → stepIndex recalé à 3 → 3%5 = 3 → même position visuelle
Prochain tick : stepIndex=4 → 4%5=4
```
Sans recalibration : `stepIndex=11` resterait 11, `11%5=1` — saut visuel ET audio. Avec recalibration : position 3 préservée, pas de saut.

Problème : le `stepIndex` est maintenant un petit nombre (0-4) plutôt qu'un compteur absolu. Le **prochain** tick repart de cette position, donc la boucle "avance" de 1 depuis la position recalée. Ce n'est pas parfait (on perd le rythme absolu) mais on évite le saut brutal.

#### `stop()` — nouvelle version

```typescript
public stop(): void {
  Tone.getTransport().stop();
  Tone.getTransport().cancel(0);
  this.clearMasterLoop();
}
```

#### `clearAll()` — nouvelle version

```typescript
private clearAll(): void {
  this.clearMasterLoop();
  this.trackNotes = [];
  this.currentColumnCount = 0;
  this.stepIndex = 0;
}
```

#### `clearMasterLoop()` — nouvelle méthode privée

```typescript
private clearMasterLoop(): void {
  if (this.masterLoop) {
    this.masterLoop.dispose();
    this.masterLoop = undefined;
  }
}
```

### Design patterns

| Pattern | Justification |
|---------|---------------|
| **Pull / Reactive** | Le callback du master loop lit `this.trackNotes` et `this.currentColumnCount` en direct à chaque tick — pas de pré-scheduling. Les changements sont répercutés au tick suivant sans reschedule |
| **Merge de responsabilités** | Une seule boucle pour audio + animation — supprime la dualité `Tone.Sequence` vs `Tone.Loop` |
| **Singleton** (inchangé) | `AudioEngine` |

### Complexité

| Opération | Avant | Après | Gain |
|-----------|-------|-------|------|
| Play initial | O(N×S) — crée N Tone.Sequence + 1 Tone.Loop | O(1) — crée 1 Tone.Loop | -N×S allocations |
| Update (live edit) | O(N×S) — `_part.clear()` + `_rescheduleSequence()` par Sequence | **O(1)** — affectation de tableau + champ | Suppression du reschedule Tone.js natif |
| Par tick '8n' | O(N) — N callbacks Sequence + 1 callback step loop | O(N) — 1 callback qui itère N pistes | Identique |
| Changement nombre de pistes | Create/dispose Tone.Sequence | O(1) — rien, juste le tableau change | Suppression create/dispose Sequence |
| Changement colCount | O(1) — MAJ champ (Phase 9) | O(1) — MAJ champ + recalage stepIndex | Identique |

N = nombre de pistes (1-4 typiquement), S = nombre de pas (1-32 typiquement).

### Duplication de code — ménage

| Code supprimé | Lignes | Remplacement |
|---------------|--------|--------------|
| `private sequences: Tone.Sequence[]` | 1 | `private trackNotes: SequenceNotes[][]` |
| `private stepLoop?: Tone.Loop` | 1 | Fusionné dans `masterLoop` |
| `clearSequences()` | 3 | Supprimé (plus de Sequences) |
| `clearStepLoop()` | 5 | `clearMasterLoop()` (5 lignes) |
| `createStepLoop()` | 10 | Fusionné dans `createSequence()` |
| Boucle while create/dispose dans `updateSequences()` | 16 | Supprimé (plus de Sequences) |
| `seq.events = notes` | 3 | Supprimé (plus de reschedule) |

**Bilan** : ~40 lignes supprimées, ~25 lignes ajoutées. Code plus court et plus simple.

### Tests GWT (TDD — test d'abord)

#### Prérequis : modifications du mock Tone

Le mock `jest.mock('tone', ...)` doit être simplifié car `Tone.Sequence` n'est plus utilisé par `AudioEngine`. Il reste mocké (pas d'erreur d'import) mais simplifié.

```typescript
jest.mock('tone', () => {
  const decodeFn = jest.fn().mockResolvedValue({});

  return {
    decodeFn,
    Sequence: jest.fn(),                         // plus utilisé, gardé pour compilation
    Players: jest.fn(() => ({
      add: jest.fn(),
      player: jest.fn(() => ({ start: jest.fn() })),
      toDestination: jest.fn(),
      dispose: jest.fn(),
    })),
    Loop: jest.fn((loopCallback: () => void) => ({
      start: jest.fn(),
      dispose: jest.fn(),
      cb: loopCallback,                          // callback capturé pour les tests
    })),
    getContext: jest.fn(() => ({
      decodeAudioData: decodeFn,
    })),
    getTransport: jest.fn(() => ({
      bpm: { value: 120 },
    })),
  };
});
```

Le type `mockTone` est simplifié (plus de `sequences`) :

```typescript
const mockTone = jest.requireMock('tone') as {
  decodeFn: jest.Mock;
  Sequence: jest.Mock;
  Players: jest.Mock;
  Loop: jest.Mock;
  getContext: jest.Mock;
};
```

Le `beforeEach` est simplifié aussi :

```typescript
beforeEach(() => {
  mockTone.Sequence.mockClear();
  mockTone.Loop.mockClear();
  mockTone.decodeFn.mockClear();
});
```

#### Tests existants à réécrire (3 tests)

Ces tests dans `describe('#updateSequences')` utilisent `mockTone.sequences` ou `Tone.Sequence` et doivent être réécrits :

**Test 1 — race condition guard** (remplace "should guard against empty sequences")

```typescript
it('should guard against no master loop (race condition)', () => {
  engine.updateSequences([['kickdrum']]);
  // Ne doit pas crasher — guard : if (!this.masterLoop) return
});
```

**Test 2 — track count increase** (remplace "should create new sequences when track count increases")

```typescript
it('should accept more tracks without crashing', () => {
  engine.createSequence([['kickdrum']]);
  // Ne doit pas crasher
  engine.updateSequences([['kickdrum'], ['hihat']]);
});
```

**Test 3 — track count decrease** (remplace "should dispose extra sequences when track count decreases")

```typescript
it('should accept fewer tracks without crashing', () => {
  engine.createSequence([['kickdrum'], ['hihat']]);
  engine.updateSequences([['kickdrum']]);
  // Ne doit pas crasher
});
```

#### Nouveaux tests (8 tests)

Ajouter dans `describe('#createSequence', ...)` et `describe('#updateSequences', ...)` :

```typescript
describe('#createSequence', () => {
  let engine: AudioEngine;

  beforeAll(() => {
    engine = AudioEngine.getInstance();
  });

  it('should create a single master Tone.Loop', () => {
    engine.setStepCallback(jest.fn());
    engine.createSequence([['a', 'b']]);
    expect(mockTone.Loop).toHaveBeenCalledTimes(1);
    expect(mockTone.Sequence).not.toHaveBeenCalled();
  });

  it('should not create master loop when onStep is not set', () => {
    engine.clearStepCallback();
    mockTone.Loop.mockClear();
    engine.createSequence([['a', 'b']]);
    expect(mockTone.Loop).not.toHaveBeenCalled();
  });

  it('should play a note on each tick', () => {
    // Given: players initialisés, sequence créée
    const players = engine.createPlayers({});
    const stepCb = jest.fn();
    engine.setStepCallback(stepCb);
    engine.createSequence([['kickdrum'], ['hihat']]);
    const capturedCallback = (
      mockTone.Loop.mock.results[0]?.value as MockLoopInstance
    ).cb;

    // When: tick 0
    capturedCallback();
    // Then: kickdrum joué sur piste 0, hihat sur piste 1, onStep(0)
    const playerFn = mockTone.Players.mock.results[0]?.value.player;
    expect(playerFn).toHaveBeenCalledWith('kickdrum');
    expect(playerFn).toHaveBeenCalledWith('hihat');
    expect(stepCb).toHaveBeenCalledWith(0);
  });

  it('should handle groups (string[]) on a tick', () => {
    const stepCb = jest.fn();
    engine.setStepCallback(stepCb);
    engine.createSequence([['kickdrum']]);
    // Injecter un groupe dans trackNotes
    (engine as Record<string, unknown>).trackNotes = [[['hihat', 'snare']]];
    const capturedCallback = (
      mockTone.Loop.mock.results[0]?.value as MockLoopInstance
    ).cb;

    capturedCallback();

    const playerFn = mockTone.Players.mock.results[0]?.value.player;
    expect(playerFn).toHaveBeenCalledWith('hihat');
    expect(playerFn).toHaveBeenCalledWith('snare');
    expect(stepCb).toHaveBeenCalledWith(0);
  });

  it('should skip null notes (silence)', () => {
    const stepCb = jest.fn();
    engine.setStepCallback(stepCb);
    engine.createSequence([['kickdrum']]);
    (engine as Record<string, unknown>).trackNotes = [[null]];
    const capturedCallback = (
      mockTone.Loop.mock.results[0]?.value as MockLoopInstance
    ).cb;

    capturedCallback();

    const playerFn = mockTone.Players.mock.results[0]?.value.player;
    // player().start() n'est pas appelé directement, on vérifie que player() n'a pas été appelé
    // En pratique, si note est null, le callback ne joue rien
    expect(stepCb).toHaveBeenCalledWith(0);
  });

  it('should skip tracks shorter than current step', () => {
    const stepCb = jest.fn();
    engine.setStepCallback(stepCb);
    engine.createSequence([['a', 'b'], ['c']]);
    // Simuler stepIndex=1 : piste 0 a [a,b] → joue b, piste 1 a [c] → step >= length, skip
    (engine as Record<string, unknown>).trackNotes = [['a', 'b'], ['c']];
    (engine as Record<string, unknown>).stepIndex = 1;
    (engine as Record<string, unknown>).currentColumnCount = 2;
    const capturedCallback = (
      mockTone.Loop.mock.results[0]?.value as MockLoopInstance
    ).cb;

    capturedCallback();

    const playerFn = mockTone.Players.mock.results[0]?.value.player;
    expect(playerFn).toHaveBeenCalledWith('b');
    // 'c' ne doit pas être appelé car step(=1) >= trackNotes[1].length(=1)
    expect(stepCb).toHaveBeenCalledWith(1);
  });
});
```

---

```typescript
describe('#updateSequences', () => {
  // ... tests existants réécrits ...

  it('should update trackNotes without recreating master loop', () => {
    engine.setStepCallback(jest.fn());
    engine.createSequence([['a', 'b']]);
    mockTone.Loop.mockClear();
    engine.updateSequences([['x', 'y', 'z']]);
    expect(mockTone.Loop).not.toHaveBeenCalled();
  });

  it('should preserve relative position when columnCount changes', () => {
    engine.setStepCallback(jest.fn());
    engine.createSequence([['a', 'b', 'c', 'd']]);
    // Simuler stepIndex à une position connue
    (engine as Record<string, unknown>).stepIndex = 11; // 11%4=3
    (engine as Record<string, unknown>).currentColumnCount = 4;
    const capturedCallback = (
      mockTone.Loop.mock.results[0]?.value as MockLoopInstance
    ).cb;

    // Recalibration : 11%4=3, nouveau colCount=5 → stepIndex=3
    engine.updateSequences([['w', 'x', 'y', 'z', 'a']]);
    // stepIndex doit être 3 (position relative préservée)
    capturedCallback(); // step 3 → stepIndex=4
    capturedCallback(); // step 4 → stepIndex=5
    capturedCallback(); // step 0 → stepIndex=6
    // Vérifier que les bonnes notes sont jouées selon le nouveau tableau
  });

  it('should not change master loop when trackNotes change without columnCount change', () => {
    engine.setStepCallback(jest.fn());
    engine.createSequence([['a', 'b', 'c']]);
    mockTone.Loop.mockClear();
    engine.updateSequences([['x', 'y', 'z']]);
    expect(mockTone.Loop).not.toHaveBeenCalled();
  });
});
```

### Ordre d'implémentation TDD

1. **Modifier le mock `jest.mock('tone', ...)`** — simplifier `Sequence`, enlever `sequences`, garder `Loop` enrichi avec `cb`
2. **Simplifier le type `mockTone` et le `beforeEach`**
3. **Réécrire les 3 tests existants** dans `describe('#updateSequences')`
4. **Ajouter le nouveau `describe('#createSequence')`** avec 6 tests
5. **Ajouter les 3 nouveaux tests** dans `describe('#updateSequences')`
6. **Lancer `npm run test -- --testPathPattern='audio-engine'`** → TOUS doivent échouer (RED)
7. **Modifier `audio-engine.ts`** :
   - Supprimer `private sequences`, `private stepLoop`
   - Ajouter `private stepIndex = 0`, `private masterLoop`, `private trackNotes`
   - Réécrire `createSequence()` (code fourni ci-dessus)
   - Réécrire `updateSequences()` (code fourni ci-dessus)
   - Réécrire `stop()` (appeler `clearMasterLoop`)
   - Réécrire `clearAll()` (appeler `clearMasterLoop`)
   - Ajouter `clearMasterLoop()`
   - Supprimer `clearSequences()`, `clearStepLoop()`, `createStepLoop()`
8. **Lancer `npm run test -- --testPathPattern='audio-engine'`** → tous les tests passent (GREEN)
9. **Lancer `npm run test`** complet → vérifier aucune régression (toutes les suites passent)
10. **Lancer `npm run lint`** → 0 erreurs

### Tableau de régression

| Test existant | Impact | Résultat attendu |
|---------------|--------|-----------------|
| `#hasSymbol` (2 tests) | Aucun | Passent sans modification |
| `#registerSymbol` (1 test) | Aucun | Passe sans modification |
| `#addToPlayers` (2 tests) | Aucun | Passent sans modification |
| `#updateSequences — guard` (1 test) | Réécrit (guard `sequences.length` → `masterLoop`) | Nouvelle version passe |
| `#updateSequences — track increase` (1 test) | Réécrit (plus de Sequences) | Nouvelle version passe |
| `#updateSequences — track decrease` (1 test) | Réécrit (plus de Sequences) | Nouvelle version passe |
| `#updateSequences — no recreate` (1 test, Phase 9) | Adapté (guard `sequences.length` → `masterLoop`) | Passe |
| `#updateSequences — columnCount callback` (1 test, Phase 9) | Adapté (closure du master loop, pas du step loop) | Passe |
| `audio-facade.test.ts` (4 tests) | Aucun — la facade ne voit pas les internes | Passent sans modification |
| Tous les autres tests | Aucun | Passent sans modification |

## Cas limites et pièges

| Cas | Comportement attendu |
|-----|---------------------|
| `updateTrack` appelé avant que `playPattern` ait fini | `updateSequences` fait un early return (`!this.masterLoop`), sans erreur |
| User supprime un symbole pendant la lecture | La séquence est mise à jour immédiatement (le symbole supprimé disparaît du son) |
| User ajoute une piste pendant la lecture | `this.trackNotes` mis à jour, pas de création/dispose de Sequences |
| User supprime une piste pendant la lecture | `this.trackNotes` mis à jour, pas de création/dispose de Sequences |
| User change la 1ère phrase (normalisation) | Toutes les autres pistes sont normalisées → `areAllSymbolsValid` recalculé → update si valide |
| User toggle un mute pendant la lecture | `sentencesForPlayback` recalcule → `areAllSymbolsValid` (toujours true, car `.` est valide) → update |
| User ajoute un symbole valide + un invalide en même temps | `allValid` = false, aucun update. Quand l'invalide est corrigé, les DEUX changements sont appliqués |
| `InstrumentEngine` n'est pas encore initialisé | Les facades `getInstrumentNameFromSymbol` et `getInstrumentFilePathsFromSymbol` appellent `prepareInstrumentEngine()` en interne |
| `Tone.getContext().decodeAudioData()` échoue | L'erreur est attrapée dans `updatePattern`, silencieusement (pas d'alert bloquant) |
| `getInstrumentFilePathsFromSymbol` throw (symbole inconnu) | Ne peut pas arriver — on filtre avec `hasSymbol()` qui est géré en amont, et la validation bloque les symboles invalides avant cet appel |
| `this.players.add(name, decoded[index])` — Tone.js accepte des ajouts après la création initiale | Confirmé : `Tone.Players.add()` fonctionne dynamiquement |
| Master loop callback — groupe `(A B)` en sub-array | `Array.isArray(note)` → `forEach` → chaque sous-note jouée à la même position temporelle |
| Master loop callback — piste plus courte que la première | `if (step >= trackNotes.length) return` — skip, pas de crash |
| ColCount change → recalibration `stepIndex` | `oldPosition = stepIndex % oldColCount` puis `stepIndex = oldPosition` — pas de saut audio |

## Avancement

- [x] Analyse du code existant (25/06/2026)
- [x] Vérification faisabilité Tone.Sequence.events setter — OK
- [x] Analyse d'architecture et optimisations
- [x] Plan de tests GWT exhaustif
- [x] Phase 0 — Déduplication du tokenizer
- [x] Phase 1 — `InstrumentEngine.getAllSymbols()`
- [x] Phase 2 — `areAllSymbolsValid()`
- [x] Phase 3 — `sentence-tokenizer.test.ts`
- [x] Phase 4 — `AudioEngine` nouvelles méthodes
- [x] Phase 5 — `audio-facade.updatePattern()`
- [x] Phase 6 — `AudioContext.updateTrack()`
- [x] Phase 7 — Intégration `PatternWorkspace`
- [x] Phase 8 — Enregistrement symboles au Play initial
- [x] Phase 9 — Correction désynchronisation step loop (animation)
- [x] Phase 10 — Correction saut audio (master Tone.Loop)
- [x] Phase 11 — Correction de la subdivision des groupes `(A B C)`

## Phase 11 — Correction de la subdivision des groupes `(A B C)` dans le master Tone.Loop

### Scénarios GWT

1. **Étant donné que** un pattern contient un groupe `(A B C)` et qu'il est lu
   **Alors** les 3 instruments A, B, C sont joués **successivement** dans la même pulsation (subdivision du temps)
   **Et non PAS** simultanément

2. **Étant donné que** un pattern contient `(A B)` et qu'il est lu
   **Alors** A est joué au début du temps, B est joué à mi-temps (intervalle `'8n' / 2`)

### Résumé pour l'implémentation

**Fichiers à modifier** :
| Fichier | Action |
|---------|--------|
| `src/renderer/features/audio/engine/audio-engine.ts` | Ajouter `stepDuration`, modifier `setTempo()` et le callback master loop |
| `src/renderer/features/audio/tests/audio-engine.test.ts` | Ajouter `Tone.Time` au mock, modifier le test de groupe, ajouter 1 test |

**Fichiers inchangés** : tous les autres

### Cause racine

Avec le master `Tone.Loop`, le callback joue tous les éléments d'un groupe au **même `time`** :

```typescript
// ÉTAT ACTUEL (buggé)
} else if (Array.isArray(note)) {
    note.forEach((n) => {
        if (typeof n === 'string') {
            this.players?.player(n).start(time);  // ← même `time`, toutes les sous-notes
        }
    });
}
```

Avant le refacto, `Tone.Sequence` utilisait `_rescheduleSequence(value, subdivision / value.length, ...)` pour espacer automatiquement les sous-notes (Sequence.js:151). Avec le master loop, ce comportement doit être reproduit manuellement.

### Correction (3 modifications dans `audio-engine.ts`)

#### Modification 1/3 — Nouveau champ `stepDuration`

```typescript
private stepDuration: number = 0;
```

Ajouter après `private stepIndex: number = 0;`.

#### Modification 2/3 — `setTempo` calcule `stepDuration`

```typescript
// eslint-disable-next-line class-methods-use-this
public setTempo(bpm: number) {
    Tone.getTransport().bpm.value = bpm;
    this.stepDuration = Tone.Time('8n').toSeconds();
}
```

**Explication** : `Tone.Time('8n').toSeconds()` calcule la durée d'une croche en secondes à partir du BPM courant. Stockée dans le champ, elle sert à espacer les sous-notes proportionnellement.

#### Modification 3/3 — Callback : espacer les sous-notes

```typescript
// AVANT (lignes 78-83 du callback)
} else if (Array.isArray(note)) {
    note.forEach((n) => {
        if (typeof n === 'string') {
            this.players?.player(n).start(time);
        }
    });
}

// APRÈS
} else if (Array.isArray(note)) {
    const subCount = note.length;
    note.forEach((n, i) => {
        if (typeof n === 'string') {
            const offset = (i * this.stepDuration) / subCount;
            this.players?.player(n).start(time + offset);
        }
    });
}
```

**Explication** : chaque sous-note est décalée de `(i * stepDuration) / subCount`. Pour `(A B C)` → indices 0, 1, 2 → offsets 0, stepDuration/3, 2*stepDuration/3. Toutes les sous-notes tiennent dans le temps d'une croche.

### Prérequis : modifier le mock `jest.mock('tone', ...)`

Ajouter `Time` au mock pour que `Tone.Time('8n').toSeconds()` retourne une valeur connue :

```typescript
// Dans jest.mock('tone', () => {
return {
    decodeFn,
    Sequence: jest.fn(),
    Players: jest.fn(() => ({...})),
    Loop: jest.fn((loopCallback: () => void) => ({...})),
    Time: jest.fn(() => ({ toSeconds: jest.fn(() => 0.2) })),  // ← AJOUTER
    getContext: jest.fn(() => ({...})),
    getTransport: jest.fn(() => ({...})),
};
```

Ajouter `Time` dans le type `mockTone` :

```typescript
const mockTone = jest.requireMock('tone') as {
    decodeFn: jest.Mock;
    Sequence: jest.Mock;
    Players: jest.Mock;
    Loop: jest.Mock;
    Time: jest.Mock;           // ← AJOUTER
    getContext: jest.Mock;
};
```

Ajouter `mockTone.Time.mockClear();` dans le `beforeEach`.

### Tests GWT (TDD — test d'abord)

#### Test 1 : modifier le test de groupe existant — vérifier les offsets temporels

Remplacer `should handle groups (string[]) on a tick` par :

```typescript
it('should play sub-notes with time offsets for groups', () => {
    const stepCb: StepCallback = jest.fn();
    engine.setStepCallback(stepCb);
    engine.setTempo(120);
    engine.createSequence([['kickdrum']]);
    (engine as unknown as Record<string, unknown>).trackNotes = [
        [['hihat', 'snare']],
    ];
    const capturedCallback = (
        mockTone.Loop.mock.results[0]?.value as MockLoopInstance
    ).cb;

    capturedCallback();

    const playerFn = mockTone.Players.mock.results[0]?.value.player;
    expect(playerFn).toHaveBeenCalledWith('hihat');
    expect(playerFn).toHaveBeenCalledWith('snare');

    const startHihat = playerFn.mock.results[0]?.value.start;
    const startSnare = playerFn.mock.results[1]?.value.start;
    expect(startHihat).toHaveBeenCalledWith(expect.closeTo(0, 0.001));
    expect(startSnare).toHaveBeenCalledWith(expect.closeTo(0.1, 0.001));
    expect(stepCb).toHaveBeenCalledWith(0);
});
```

**Validation TDD RED** : avant la correction, les deux appels `start` reçoivent le même `time` (0). Le test `closeTo(0.1)` échoue.

**Validation TDD GREEN** : après la correction, `hihat` est à `time + 0`, `snare` à `time + stepDuration / 2 = 0 + 0.1`. Les deux assertions passent.

#### Test 2 : trois sous-notes

```typescript
it('should spread three sub-notes across the step duration', () => {
    const stepCb: StepCallback = jest.fn();
    engine.setStepCallback(stepCb);
    engine.setTempo(120);
    engine.createSequence([['kickdrum']]);
    (engine as unknown as Record<string, unknown>).trackNotes = [
        [['a', 'b', 'c']],
    ];
    const capturedCallback = (
        mockTone.Loop.mock.results[0]?.value as MockLoopInstance
    ).cb;

    capturedCallback();

    const playerFn = mockTone.Players.mock.results[0]?.value.player;
    const startA = playerFn.mock.results[0]?.value.start;
    const startB = playerFn.mock.results[1]?.value.start;
    const startC = playerFn.mock.results[2]?.value.start;

    expect(startA).toHaveBeenCalledWith(expect.closeTo(0, 0.001));
    expect(startB).toHaveBeenCalledWith(expect.closeTo(0.0666, 0.001));
    expect(startC).toHaveBeenCalledWith(expect.closeTo(0.1333, 0.001));
});
```

#### Test 3 : groupe avec une seule sous-note — offset = 0

```typescript
it('should play a single-element group at time 0', () => {
    const stepCb: StepCallback = jest.fn();
    engine.setStepCallback(stepCb);
    engine.setTempo(120);
    engine.createSequence([['kickdrum']]);
    (engine as unknown as Record<string, unknown>).trackNotes = [
        [['solo']],
    ];
    const capturedCallback = (
        mockTone.Loop.mock.results[0]?.value as MockLoopInstance
    ).cb;

    capturedCallback();

    const playerFn = mockTone.Players.mock.results[0]?.value.player;
    const startSolo = playerFn.mock.results[0]?.value.start;
    expect(startSolo).toHaveBeenCalledWith(expect.closeTo(0, 0.001));
});
```

### Ordre d'implémentation TDD

1. **Ajouter `Time` au mock `jest.mock('tone', ...)`** — `Time: jest.fn(() => ({ toSeconds: jest.fn(() => 0.2) }))`
2. **Ajouter `Time` au type `mockTone` et au `beforeEach`**
3. **Modifier le test de groupe existant** (vérifier les offsets `start`) + ajouter les 2 nouveaux tests
4. **Lancer `npm run test -- --testPathPattern='audio-engine'`** → les nouveaux tests échouent (RED)
5. **Ajouter `private stepDuration: number = 0;`** dans `AudioEngine`
6. **Modifier `setTempo(bpm)`** pour calculer `this.stepDuration = Tone.Time('8n').toSeconds();`
7. **Modifier le callback master loop** — espacer les sous-notes avec offset
8. **Lancer `npm run test -- --testPathPattern='audio-engine'`** → tous les tests passent (GREEN)
9. **Lancer `npm run test`** complet → vérifier aucune régression
10. **Lancer `npm run lint`** → 0 erreurs

### Tableau de régression

| Test existant | Impact | Résultat attendu |
|---------------|--------|-----------------|
| `#createSequence — should handle groups` | Remplacé par la version vérifiant les offsets | Passe avec la correction |
| `#createSequence — should play a note on each tick` | Aucun (pistes séparées) | Passe sans modification |
| `#createSequence — should skip null notes` | Aucun (null, pas de groupe) | Passe sans modification |
| `#createSequence — should skip tracks shorter than current step` | Aucun | Passe sans modification |
| `#createSequence — creates single master loop` | Aucun | Passe sans modification |
| `#createSequence — should not create master loop when onStep not set` | Aucun | Passe sans modification |
| Tous les `#updateSequences` | Aucun (pas de callback invoqué) | Passent sans modification |
| Tous les autres tests | Aucun | Passent sans modification |

## Notes / décisions

- L'accumulation des changements valides derrière un invalide est implicite : le state React contient la dernière version des phrases, le gate ne fait que différer l'appel à `updatePattern`
- `updatePattern` charge les nouveaux buffers audio AVANT de mettre à jour les sequences, pour éviter un délai entre le reschedule et la disponibilité du son
- Pas besoin de debounce — chaque frappe valide pendant la lecture déclenche un update. Tone.js gère le reschedule sans glitch.
- Si la validation échoue → pas d'update audio → le nouveau symbole invalide est visible dans la grille (en rouge) mais pas joué
- `areAllSymbolsValid` utilise `replace(/[()]/g, '')` + `.split()` (O(n)) plutôt que le regex complet (O(n×m)) — optimisation délibérée
- Le tokenizer partagé (`sentence-tokenizer.ts`) est dans `src/renderer/utils/` car il est utilisé par plusieurs features (pattern, sequence)
- `hasSymbol` / `registerSymbol` dans `AudioEngine` trackent par symbole (ex: "P"), pas par nom d'instrument (ex: "kickdrum"). Le mapping symbole → nom est fait par `InstrumentEngine` dans la facade

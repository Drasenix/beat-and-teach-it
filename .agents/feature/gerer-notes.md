# Feature: Gérer les notes (pitch shifting par instrument)

## Problème

Un instrument = un fichier audio fixe. Pour les percussions (kick, snare) c'est
suffisant. Mais pour un son mélodique (ex: un "Hum" fredonné), il faudrait un
fichier par note, ce qui est ingérable.

## Solution cible

Enregistrer un seul sample, et le **moduler à la volée** (pitch shifting via
`playbackRate`) pour produire différentes notes. Un même échantillon audio joué
à des hauteurs différentes.

---

## Flux de pensée & analyses contradictoires

### Round 1 — Première intuition (analyse #1)

L'approche naïve : stocker les fréquences dans un tableau `frequencies: (number
| null)[][]` séparé, en miroir de `highlights`.

- Nouvelle colonne `frequencies TEXT` dans la table `patterns`
- Nouvel état `useState<(number | null)[][]>()` dans `usePatternSession`
- Synchronisation `sentences ↔ frequencies` à chaque `changeSentence`
- Molette : modifier `frequencies[i][j]` directement
- Notation texte : `Hum@440`, parser → extraire la fréquence → stocker dans le
  tableau, nettoyer le texte

**Problèmes identifiés :**
1. Double source de vérité (texte ET tableau) → risque de désynchronisation
2. Le textarea affiche `Hum@440`, mais `changeSentence` nettoie → le `@440`
   disparaît visuellement → UX confuse
3. Migration BDD nécessaire sur `patterns`
4. Duplication de la logique de synchronisation (déjà complexe pour
   `highlights`)

### Round 2 — Remise en question (analyse #2)

En explorant `SentenceInput`, découverte clé : le composant est **fully
controlled** — `value={sentence}` + `onChange={(value) => changeSentence(index,
value)}`. Aucun état local.

**Nouvelle direction : le texte est la source unique de vérité.**

- Les `@freq` vivent DANS les `sentences` (ex: `"Hum@440 P (Ts@220 K) ."`)
- **Aucun** tableau `frequencies` séparé
- **Aucune** migration BDD pour `patterns`
- **Aucun** état supplémentaire dans `usePatternSession`
- La molette modifie le texte directement → `changeSentence` → re-render
  automatique du textarea ET de la grille

### Comparaison des deux approches

```
Option B (tableaux séparés)              Option D (texte source unique)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Stockage : 2 colonnes JSON               Stockage : 1 colonne (sentences)
State    : 2 tableaux à synchroniser     State    : 0 tableau supplémentaire
Molette  : Modifier frequencies[i][j]    Molette  : Modifier la phrase texte
Migration : OUI (ALTER TABLE patterns)   Migration : NON (patterns inchangés)
Risque   : Désynchronisation             Risque   : Bug dans updateTokenFrequency
Complexité globale : ÉLEVÉE             Complexité globale : MOYENNE
```

### Corrections apportées par l'analyse #2

1. **"Deux pistes, même instrument, même step, playbackRate différents"**
   → Non-problème. Tone.js crée un `AudioBufferSourceNode` distinct par
   `player.start(time)`, chacun capture son propre `playbackRate`.

2. **Type `SequenceNotes` cassant**
   → Acceptable. Le type n'est consommé que par `preparePattern` →
   `AudioEngine.createSequence`, deux fonctions sous notre contrôle.

3. **Complexité de `updateTokenFrequency`**
   → Fonction pure isolée d'environ 30 lignes, parfaite pour le TDD.

### Décision finale

**Option D retenue — Texte source unique.**

---

## Décisions actées

| # | Décision | Raison |
|---|----------|--------|
| 1 | Pas de flag "mélodique" | Tout instrument est pitchable, c'est l'utilisateur qui décide d'appliquer ou non une fréquence |
| 2 | Notation `Hum@440` | `@` n'est pas utilisé dans la syntaxe actuelle, se lit "at" |
| 3 | Fréquence brute (Hz) stockée dans le texte | Stockage simple, pas de mapping note→fréquence |
| 4 | Molette = ±1 demi-ton | Musical, intuitif. `freq * 2^(±1/12)` |
| 5 | Détection hauteur en V2 (autocorrélation) | Via un bouton "Détecter" dans le formulaire instrument |
| 6 | Champ `referenceFrequency` dans `instruments` | Stocké en BDD, détecté une fois, modifiable manuellement |
| 7 | `PatternStep` enrichi de `frequency?: number` | Extraite du texte par le parser |
| 8 | `SequenceNotes` évolue vers `{ name, playbackRate }` | Le playbackRate est pré-calculé avant l'envoi à AudioEngine |
| 9 | Pas de migration `patterns` | Les `@freq` sont dans les `sentences` existantes |
| 10 | Migration `instruments` : `ALTER TABLE ADD reference_frequency REAL` | Seule migration BDD nécessaire |
| 11 | Manifest library : version 1 → 2 | Ajout de `referenceFrequency` dans les instruments exportés |
| 12 | `playbackRate` clampé à 0.25–4 | Évite les taux extrêmes (-2 à +2 octaves) |
| 13 | Pas de time-stretch en V1 | Accepté : la durée du sample change avec le pitch |
| 14 | `@` interdit dans les symboles d'instruments | Validation ajoutée pour éviter les ambiguïtés |

---

## Flux de données

```
Création instrument
  ├── Formulaire : symbole, nom, fichier audio
  ├── [Bouton "Détecter"] → autocorrélation → 440 Hz
  ├── Champ "Fréquence de référence" : 440 Hz (modifiable)
  └── Sauvegarde → INSERT instruments (..., reference_frequency = 440)

Saisie pattern
  ├── Textarea : "Hum@440 P (Ts@220 K) ."
  ├── SentenceInput.onChange → changeSentence(index, "Hum@440 P (Ts@220 K) .")
  ├── usePattern → stocke le texte brut dans pattern.sentences
  └── Grid : parseSteps → PatternStep[] avec frequency extraite

Molette sur StepBadge
  ├── onWheel → deltaY → ±1 demi-ton → nouvelle fréquence
  ├── updateTokenFrequency(sentence, flatTokenIndex, newFreq)
  │     → "Hum@440 P (Ts@220 K) ."  devient  "Hum@466 P (Ts@220 K) ."
  └── changeSentence(index, newSentence) → re-render automatique

Lecture (Play)
  ├── sentencesForPlayback = transformSentencesWithMute(sentences, mutedSteps)
  ├── preparePattern("Hum@A#4 P (Ts@C4 K) .")
  │     ├── tokenizeSentence → ["Hum@A#4", "P", "(Ts@C4 K)", "."]
  │     ├── parseToken("Hum@A#4") → { symbol: "Hum", frequency: 466.16 }
  │     ├── getInstrumentReferenceFrequency("Hum") → 220
  │     ├── playbackRate = clamp(466.16 / 220, 0.25, 4) = 2
  │     ├── semitoneOffset = round(12 * log2(2)) = 12
  │     └── SequenceNote { name: "hum", playbackRate: 2, semitoneOffset: 12 }
  └── AudioEngine.createSequence
        └── Loop callback : playNote → getPoolPlayer("hum", offset=12, rate=2)
              ├── 1er usage → new Tone.Player(buffer).toDestination()
              │               player.playbackRate = 2  (fixé UNE FOIS)
              │               stocké dans playerPool[0]["hum"].get(12)
              └── player.start(time)  (pas de setter)
```

---

## Architecture cible

### Modèle de données

```
Instrument
  + referenceFrequency: number | null   (Hz, détecté ou saisi)

Pattern
  (inchangé — les @freq sont dans sentences)

PatternStep
  + frequency?: number                  (extraite du token @freq)
```

### AudioEngine — pool lazy

```
createPlayers(audioBuffers)
  → décode les buffers → this.decodedBuffers: Map<name, AudioBuffer>

createSequence(tracks)
  → 1 Tone.Loop par piste (localStep par closure)
  → pas de buildTrackPlayers (les Players sont créés lazily)

Loop callback:
  → playNote(trackIndex, note, time)
    → getPoolPlayer(trackIndex, name, semitoneOffset, playbackRate)
      → existe déjà ? → retourne le Player
      → nouveau ? → new Tone.Player(buffer).toDestination()
                   → player.playbackRate = playbackRate (UNIQUEMENT à la création)
                   → stocké dans playerPool[track][name].get(semitoneOffset)
    → player.start(time)  ← pas de setter, pas de timeline modifiée

updateSequences(tracks)
  → swap this.trackNotes
  → si nb pistes change → rebuildTrackLoops()
  → le pool lazy crée de nouveaux Players au 1er usage
```

### Types

```typescript
// Sequence note enrichi
type SequenceNote = { name: string | null; playbackRate: number; semitoneOffset?: number } | null;
type SequenceNotes = SequenceNote | SequenceNote[];
```

### Nouveaux fichiers

| Fichier | Rôle |
|---------|------|
| `src/renderer/utils/detect-pitch.ts` | Autocorrélation → fréquence (Hz) |
| `src/renderer/utils/update-token-frequency.ts` | `updateTokenFrequency(sentence, flatIdx, freq): string` |
| `src/renderer/utils/frequency-to-note.ts` | `frequencyToNoteName(freq): string` (A4=440) |
| `src/renderer/utils/note-name-to-frequency.ts` | `noteNameToFrequency("A#4"): number` (reverse) |
| `src/renderer/features/instruments/hooks/useDetectPitch.ts` | Hook partagé pour le bouton Détecter |
| `src/renderer/features/pattern/components/StepTooltip.tsx` | Note strip + couleurs + mute + reset |

### Fichiers modifiés

| Fichier | Modification |
|---------|-------------|
| `src/shared/models/instrument-dto.ts` | Ajout `referenceFrequency: number \| null` |
| `src/renderer/features/instruments/models/instrument-model.ts` | Ajout `referenceFrequency` |
| `src/renderer/features/instruments/types/instrument-types.ts` | Hérite de Instrument |
| `src/renderer/features/instruments/components/form/InstrumentForm.tsx` | Champ fréquence + bouton Détecter |
| `src/renderer/features/instruments/utils/instrument-validator.ts` | Validation : `@` interdit dans symbole |
| `src/main/db/repositories/instrument-repository.ts` | INSERT/UPDATE avec `referenceFrequency` |
| `src/main/db/migrations/003_pitch.ts` | `ALTER TABLE instruments ADD COLUMN reference_frequency REAL` |
| `src/main/db/database.ts` | Importer et appeler migration 003 |
| `src/renderer/features/pattern/types/pattern-types.ts` | `PatternStep.frequency?: number` |
| `src/renderer/features/pattern/utils/pattern-parser.ts` | `parseToken()`, `createStep`/`createGroup` → extraire `@freq` |
| `src/renderer/features/instruments/engine/instrument-engine.ts` | `getInstrumentReferenceFrequency(symbol)` |
| `src/renderer/features/sequence/types/sequence-note.ts` | `SequenceNote` → `{ name, playbackRate }` |
| `src/renderer/features/sequence/adapters/sequence-adapter.ts` | Adapter enrichi |
| `src/renderer/features/sequence/service/sequence-service.ts` | `preparePattern` → calculer `playbackRate` |
| `src/renderer/features/audio/engine/audio-engine.ts` | `createSequence` → appliquer `playbackRate` |
| `src/renderer/features/pattern/components/StepBadge.tsx` | `onWheel` handler + affichage note |
| `src/renderer/features/pattern/components/StepCell.tsx` | Propager `onFrequencyChange` |
| `src/renderer/features/pattern/components/Column.tsx` | Propager `onFrequencyChange` |
| `src/renderer/features/pattern/components/PatternSteps.tsx` | Propager `onFrequencyChange` |
| `src/renderer/features/pattern/hooks/usePatternSession.ts` | `handleFrequencyChange` → `updateTokenFrequency` → `changeSentence` |
| `src/renderer/features/library/` (export/import) | Manifest v2, `referenceFrequency` |
| `src/renderer/features/guide/` | Documentation feature |

---

## Cas limites

| Cas | Comportement |
|-----|-------------|
| Instrument sans `referenceFrequency` | Molette inopérante sur ses steps. `@freq` dans le texte ignoré (token traité comme symbole pur, sera invalide si `@` présent) |
| `referenceFrequency` modifiée après création de patterns | Les fréquences stockées dans les phrases restent en Hz absolu. Si la référence change, le `playbackRate` est recalculé → la hauteur de sortie change. Documenté. |
| Fréquence = `0` dans le texte (`Hum@0`) | `parseFloat("0")` → `0`. Ignoré (fréquence invalide). |
| Silence `.` avec fréquence `.@440` | Ignoré : le silence reste silencieux. |
| Groupe avec fréquences mélangées `(Hum@440 Hum@554)` | OK — chaque token interne a sa propre fréquence. |
| `@` dans le symbole d'un instrument | Validateur refuse le symbole. |
| Trackpad (scroll continu) sur le badge | Accepté en V1. Ajustement possible en V2. |
| Export/import de `.beatpack` v1 | Les patterns v1 n'ont pas de `@freq` dans leurs phrases → import normal. |

---

## Avancement

- [x] Specs validées
- [x] Analyse contradictoire (analyses #1 et #2)
- [x] Décisions actées (14 décisions)
- [x] Pitch detection (`detectPitch` + tests)
- [x] DB migration (`reference_frequency` dans instruments)
- [x] Instrument Engine : `getInstrumentReferenceFrequency`
- [x] Instrument Form UI : champ fréquence + bouton Détecter
- [x] Pattern parser : `parseToken()`, `createStep`/`createGroup` mis à jour
- [x] `PatternStep.frequency` dans le type
- [x] `SequenceNotes` type enrichi
- [x] Sequence Service/Adapter : calcul du `playbackRate`
- [x] AudioEngine : `playbackRate` dans `createSequence`
- [x] `updateTokenFrequency` utilitaire + tests
- [x] StepBadge : molette + affichage nom de note
- [x] Propagation `onFrequencyChange` dans la chaîne de composants
- [x] `usePatternSession` : intégration molette → `changeSentence`
- [x] Library export/import : manifest v2
- [x] Notation `@A#4` (nom de note) + `noteNameToFrequency`
- [x] `StepTooltip` unifié (note strip + couleurs + mute + reset)
- [x] Double-clic reset pitch
- [x] `playingRef` guard contre double playTrack
- [x] `StepTooltip` portal + MuteIcon + note strip conditionnel
- [x] Suppression `ColorTooltip`
- [x] Suppression clic mute sur badge
- [x] 1 Tone.Loop par piste + `localStep` par Loop
- [x] Pool lazy : Player créé UNE FOIS avec `playbackRate`, réutilisé par offset
- [ ] Intégration bout en bout

## Notes / décisions
- Pitch shifting via `playbackRate` (simple, pas de time-stretch en V1)
- Pas de flag "mélodique" — tout instrument est pitchable, c'est le choix utilisateur qui active ou non
- Le texte est la source unique de vérité pour les fréquences (Option D)
- `@` est le séparateur symbole/fréquence, interdit dans les symboles d'instruments
- Molette = ±1 demi-ton. Affichage en nom de note (A4, C#3) dérivé de la fréquence
- Tone.js v15 : `playbackRate` est un getter/setter `Positive` (number), pas un Signal → assignation `=` correcte
- Notation : `@A#4` accepté (nom de note) en plus de `@440` (Hz), rétrocompatible. Affichage toujours en nom de note
- Molette : snap automatique au demi-ton le plus proche (pas de floats dans le texte)
- Reset pitch : double-clic sur le badge → supprime `@freq`

## Améliorations UX (v2)

### Notation texte `@A#4` au lieu de `@440.16376`
- `parseToken` accepte les deux formats : `@440` (Hz) et `@A#4` (nom de note)
- `updateTokenFrequency` écrit toujours le nom de note (via `frequencyToNoteName`)
- La molette produit des notes justes (snap au demi-ton), pas des floats
- Nouvel utilitaire : `noteNameToFrequency()`

### Reset de la hauteur
- Double-clic sur un badge → `onFrequencyChange(null)` → supprime le `@freq` du texte
- Retour au son original (fréquence de référence de l'instrument)

### Refonte du hover : `StepTooltip` unifié
- Remplace `ColorTooltip` (supprimé) + `MuteIcon` séparés
- **Note strip** : 17 notes (±8 autour de la note courante), clic pour sélectionner. Caché si l'instrument n'a pas de `referenceFrequency`
- **Color picker** : 5 couleurs, toujours visibles (même sans `referenceFrequency`)
- **Mute** : icône `MuteIcon` (SVG speaker barré), toujours visible. Le clic simple sur le badge ne mute plus — uniquement via le tooltip
- **Reset** : bouton ⟳, visible si le token a une fréquence
- **Portal React** : rendu dans `document.body` via `createPortal()` → isolation totale de l'animation `step-active-pulse` (pas de `transform: scale(1.08)` ni `filter: brightness(1.3)` hérités)
- **Positionnement** : `position: fixed` calculé via `badgeRef.getBoundingClientRect()` au `mouseenter`
- Tailles : `w-4 h-4` pastilles, `text-xs px-1.5` notes, `p-3` global
- Scroll horizontal du note strip à la molette seule (pas de Shift)
- Pastilles couleur → hex Tailwind-400 pour correspondre au texte du badge
- 150ms de délai de fermeture inchangé

## Bugs corrigés

### #1 — `extractUniqueSymbols` n'enlevait pas le `@freq`
**Impact** : `registerSymbol("Hum@440")` au lieu de `registerSymbol("Hum")`, ce qui cassait le hot-reload en cours de lecture.
**Fix** : `sentence-tokenizer.ts:extractSymbols` applique `parseToken()` pour nettoyer le symbole avant de le pousser.

### #2 — `areAllSymbolsValid` n'enlevait pas le `@freq`
**Impact** : `allValid === false` quand `@freq` présent → `updateTrack` jamais déclenché en cours de lecture → l'ancienne séquence continuait (pas de changement de pitch en live).
**Fix** : `pattern-validator.ts:areAllSymbolsValid` applique `parseToken()` avant de comparer le symbole.

### #3 — Race condition double `playTrack`
**Impact** : erreur Tone.js "The time must be greater than or equal to the last scheduled time" quand Ctrl+Enter pressé 2x rapidement.
**Cause** : `setPlaying(true)` était après `await playPattern()`. Pendant l'attente, `playing === false` → un second `playTrack` lançait un `createSequence()` qui tentait `masterLoop.start(0)` alors que le Transport tournait déjà.
**Fix** : `AudioContext.tsx` — `playingRef` immédiatement à `true` au début de `playTrack`, garde-fou `if (playingRef.current) return`. Reset à `false` dans `stopTrack` et en cas d'erreur.

### #4 — Race condition `updateSequences` + `start()` même instrument
**Impact** : erreur Tone.js "The time must be greater than or equal to the last scheduled time" quand 2 pistes utilisent le même instrument aligné sur le même temps.
**Fix** : `audio-engine.ts` — refacto `Tone.Players` partagé → 1 `Tone.Loop` par piste + 1 `Tone.Player` par instrument/piste. Chaque piste a ses propres instances de Player et de Loop → pas de collision de timeline `_state`.

### #5 — `playbackRate` setter crash à chaque cycle de piste
**Impact** : erreur Tone.js "The time must be greater than or equal to the last scheduled time" réapparaît à chaque cycle de piste pendant la lecture. Le `player.playbackRate = rate` dans le Loop callback modifie la timeline `_state` du Player (`cancel(stopEvent) + setStateAtTime("stopped", newTime)`), ce qui nettoie les events futurs et corrompt la timeline pour les cycles suivants.
**Stack trace** : `set playbackRate (Player.js:396)` → `StateTimeline.setStateAtTime` → `Timeline.add` → `assert(GTE)`.
**Fix** : `audio-engine.ts` — pool lazy. `playerPool: Map<name, Map<semitoneOffset, Player>>[]`. Le Player est créé UNE FOIS avec son `playbackRate` définitif (pas de setter pendant la lecture). `player.start(time)` uniquement dans le Loop callback, pas de `playbackRate =` ni `stop()`.

### #6 — Décalage temporel entre pistes (stepIndex partagé)
**Impact** : les pistes étaient désynchronisées (la piste 2 jouait avec 1 temps de retard) quand `stepIndex` était partagé entre les Loops.
**Fix** : `audio-engine.ts` — `localStep` par Loop (closure). Plus de `this.stepIndex` partagé.

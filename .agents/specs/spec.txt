# Documentation Métier — Beat & Teach

## 1. Présentation du domaine

**Beat & Teach** est une application de composition musicale rythmique assistée, destinée à l'apprentissage du beatboxing et des percussions. Elle permet à l'utilisateur de **noter des patterns rythmiques** via un langage symbolique textuel, d'y associer des **fichiers audio** (instruments), et de **jouer** le résultat via un séquenceur audio temps réel.

Public cible : musiciens, beatboxers, enseignants en éveil musical.

Concept : chaque symbole (ex: `P`, `Ts`, `K`) représente un son percussif. L'utilisateur compose des séquences de symboles, les organise en pistes, et écoute le résultat.

---

## 2. Modèle de données métier

### 2.1 Instrument

L'instrument est l'unité sonore de base. Il associe un **symbole** (identifiant textuel court) à un **fichier audio** et un **nom lisible**.

```
Instrument
├── id        : number       — Identifiant technique
├── slug      : string       — Identifiant unique lisible (snake_case, auto-généré)
├── symbol    : string       — Symbole de notation (ex: "P", "Ts", "A<")
├── name      : string|null  — Nom lisible (ex: "kickdrum", "hihat")
└── filepath  : string|null  — Chemin du fichier audio (.mp3/.wav/.ogg)
```

**Règles :**
- `symbol` et `slug` sont uniques (contrainte SQL + message d'erreur utilisateur)
- L'instrument spécial `symbol: "."`, `name: null`, `filepath: null` représente le **silence** — il est filtré de tous les affichages UI
- Le `slug` est auto-généré depuis le `name` via `toSnakeCase()` (minuscules, uniquement `[a-z0-9-]`)
- La résolution des chemins audio : les chemins relatifs sont résolus par rapport à `assets/audio/` (dev) ou `resources/assets/audio/` (production). Les chemins absolus sont utilisés tels quels s'ils existent.

**Données initiales (seed) — 20 instruments :**

| Slug | Symbole | Nom | Fichier |
|------|---------|-----|---------|
| silence | `.` | null | null |
| kickdrum | `P` | kickdrum | kickdrum.mp3 |
| hihat | `Ts` | hihat | hihat.mp3 |
| snare | `Pf` | snare | snare.mp3 |
| rimshot | `K` | rimshot | rimshot.mp3 |
| sonic-boom | `W` | sonic boom | sonic-boom.mp3 |
| air-A-inward | `A<` | air-A-inward | air-A-inward.mp3 |
| air-A | `A>` | air-A | air-A.mp3 |
| air-F-inward | `F<` | air-F-inward | air-F-inward.mp3 |
| air-F | `F>` | air-F | air-F.mp3 |
| air-S-inward | `S<` | air-S-inward | air-S-inward.mp3 |
| air-S | `S>` | air-S | air-S.mp3 |
| clock | `Lo` | clock | clock.mp3 |
| cough-snare | `Eh` | cough-snare | cough-snare.mp3 |
| kch-snare | `Kch` | kch-snare | kch-snare.mp3 |
| liproll-bass | `Bwr` | liproll-bass | liproll-bass.mp3 |
| liproll | `Bw` | liproll | liproll.mp3 |
| throat-bass | `Rr` | throat-bass | throat-bass.mp3 |
| tom-bass | `Tum` | tom-bass | tom-bass.mp3 |
| tongue-kick | `p` | tongue-kick | tongue-kick.mp3 |

### 2.2 Pattern

Le pattern est une **composition musicale**. Il contient un nom, une ou plusieurs **phrases** (pistes de notation), et des **couleurs de surbrillance** par token.

```
Pattern
├── id          : number              — Identifiant technique
├── slug        : string              — Identifiant unique (snake_case, auto-généré)
├── name        : string              — Nom du pattern
├── sentences   : string[]            — Phrases de notation (1 par piste)
└── highlights  : (string|null)[][]   — Couleurs par token [phrase][position]
```

**Règles :**
- Le `slug` est auto-généré depuis le `name` et doit être unique
- Toutes les phrases sont normalisées sur la longueur de la **première phrase** (padding avec `.`)
- Les phrases après la première sont bloquées si elles dépassent la longueur de la première
- Les `highlights` sont stockés par **position de token** (pas par step visuel) — les groupes `(…)` comptent comme 1 step mais N tokens pour les highlights

### 2.3 Phrase (Sentence)

Une phrase est une ligne de notation textuelle.

```
Exemple : "P Ts (K . P) Ts K ."
```

- Les tokens sont séparés par des espaces
- Chaque token est soit un **symbole d'instrument** (ex: `P`, `Ts`), soit un **groupe** `(…)`
- Le token `.` représente un silence
- Une phrase peut être vide

### 2.4 Groupe `(…)`

Sémantique précise (validée par comportement observé) :

> `(Ts K)` = une **succession rapide** d'instruments dans une même pulsation (subdivision du temps)

- `P (Lo P)` → kick sur le temps, puis Lo puis kick en subdivision dans le même temps
- Ce n'est PAS simultané — Tone.js dispatche chaque élément du tableau à la même position temporelle, et l'oreille perçoit une succession rapide (contretemps)
- Dans la grille visuelle : les groupes ont une bordure `border-primary` en pointillé (`step-cell-group`)
- Dans le flux audio : `SequenceNote[]` (tableau de `string`) positionné dans un `Tone.Sequence` au pas `'8n'`

Représentation interne :

```typescript
PatternStep = {
  id: string;         // Identifiant de position
  symbol: string;     // Texte affiché (ex: "P" ou "(Ts K)")
  valid: boolean;     // true si le symbole existe dans la base
  isGroup: boolean;   // true si c'est un groupe (...)
  steps?: PatternStep[];  // Sous-étapes si isGroup
}
```

---

## 3. Règles de gestion (RG)

| Réf | Règle | Implémentation |
|-----|-------|----------------|
| RG1 | Le symbole d'un instrument doit être unique dans la base | Contrainte UNIQUE SQL + message : "Un instrument avec ce symbole existe déjà." |
| RG2 | Le slug d'un pattern/instrument doit être unique | Contrainte UNIQUE SQL + message : "Un pattern/instrument avec ce nom existe déjà." |
| RG3 | L'instrument de slug `.` (silence) est filtré de tous les affichages UI | `instrument.symbol !== '.'` |
| RG4 | Les phrases non-racines sont normalisées sur la longueur de la première | `normalizeSentences()` — padding avec `.` |
| RG5 | Les groupes `(…)` comptent comme 1 step dans le décompte visuel | `countSentenceSteps()` utilise `execAll()` sur la regex |
| RG6 | Un symbole inconnu (aucun instrument trouvé) est marqué invalide (rouge) | `createStep().valid = false` |
| RG7 | Un instrument sans fichier audio peut exister mais sera silencieux | Résolu via `.` dans la phrase |
| RG8 | Les tokens muets (mute) sont remplacés par `.` pour la lecture uniquement | `transformSentencesWithMute()` — ne modifie pas les données sources |
| RG9 | Les highlights sont stockés par position de token (pas par step visuel) | `highlights[sentenceIndex][tokenIndex]` — les groupes `(…)` ont N tokens internes |
| RG10 | Le slug est auto-généré au format snake_case avec uniquement `[a-z0-9-]` | `toSnakeCase()` |
| RG11 | À l'import, la résolution des conflits peut écraser, ignorer ou renommer | `ConflictAction = 'overwrite' | 'skip' | 'rename'` |
| RG12 | Les fichiers audio importés sont copiés dans `userData/imported-audio/` | `import-library-service.ts` |
| RG13 | Le BPM est limité entre 1 et 300 (slider UI : 40-240, raccourcis : 1-300) | Clamp dans `useSlider` et `AudioControls` |
| RG14 | Les messages d'erreur IPC sont nettoyés pour l'affichage utilisateur | `extractIpcError()` — retire le préfixe technique |

---

## 4. Parcours utilisateur complet

### 4.1 Studio — `/workspace`

Écran principal de composition. Layout DAW :

```
┌─────────────────────────────────────────────────────┐
│ Transport Bar : ▶ ■ | BPM [====o====] 120           │
├──────────────┬──────────────────────────────────────┤
│  Sidebar     │  Zone principale                     │
│  (resizable) │                                      │
│              │  Sentence 1 : [P Ts K . P ...]       │
│  Patterns    │  Sentence 2 : [. K Ts . P ...]       │
│  ─────────   │                                      │
│  ○ pattern A │  Grille PatternSteps :               │
│  ● pattern B │  ┌──┬────┬──┬──┬────┬──┐             │
│  ○ pattern C │  │P │ Ts │ K│ .│ (K │ .│             │
│              │  │. │ K  │Ts│ .│ P) │ .│             │
│  Instruments │  └──┴────┴──┴──┴────┴──┘             │
│  ──────────  │                                      │
│  P kickdrum ▶│                                      │
│  Ts hihat  ▶│                                      │
│  K rimshot ▶│                                      │
└──────────────┴──────────────────────────────────────┘
```

#### Transport Bar

| Élément | Comportement |
|---------|-------------|
| ▶ Play | Déclenche la lecture. Désactivé si `playing || !allSentencesValid` |
| ■ Stop | Arrête la lecture. Désactivé si `!playing` |
| Slider BPM | Valeur 40-240. Molette : +1/-1. Ctrl+↑/↓ : +1/-1 (1-300) |
| Valeur BPM | Affichage numérique (min-width 36px, aligné droite) |

État actif : le bouton Play passe en `transport-btn active` (fond primary, texte background).

#### Sidebar (redimensionnable)

- Largeur : 140px min, 340px max (drag avec `useResizable` ou touches ←/→ ±10px)
- **Patterns** (section repliable) :
  - Liste des patterns triés alphabétiquement
  - Pattern sélectionné : classe `.selected` (bordure primary)
  - "＋" Nouveau pattern
  - "Sauvegarder" (ouvre `SavePatternModal`)
  - Boutons désactivés si `playing`
- **Instruments** (section repliable) :
  - Liste des instruments triés par symbole
  - Symbole en primary, nom en texte secondaire
  - Bouton ▶ play (pré-écoute) — désactivé si `playing`

#### Grille PatternSteps

Visualisation colonne par colonne (pulsation par pulsation) :

- **Step atomique** : `step-cell-atomic` (bordure pointillée `border-border`)
- **Step groupe `(…)`** : `step-cell-group` (bordure pointillée `border-primary`) — rendu récursif des `StepBadge` internes
- **Step vide** : `step-cell-empty` → "---" en gris (opacity-30) — quand une piste a moins de steps
- **Badge valide** : `step-badge-valid` (texte primary sur fond background)
- **Badge invalide** : `step-badge-invalid` (texte rouge, bordure rouge)
- **Step muet** : `opacity-40` + icône `MuteIcon` (haut-parleur barré)
- **Highlight** : couleur de bordure + texte (voir section 7)
- **Colonne active** (lecture en cours) : animation `step-active-pulse` (scale 1→1.08, brightness 1→1.3)

Interactions sur un badge-step :
- **Clic** : toggle mute (désactivé si `onToggleMute` non fourni)
- **Survol** : `ColorTooltip` apparaît au-dessus (150ms délai de fermeture)
  - 5 couleurs : 🔴 rouge, 🔵 bleu, 🟢 vert, 🟡 jaune, 🟠 orange
  - Sélection : désactive le tooltip et applique la couleur
- **Survol d'une swatch** : scale 1.25

#### Phrases (textarea)

- Une `SentenceInput` par piste
- Placeholder : "P Ts K . P (Ts P) K"
- Autocomplétion : filtrage insensible à la casse sur le début du symbole
- Wrapping `(…)` : si du texte est sélectionné, `(` ou `)` le wrapper automatiquement
- Limite de tokens (pistes non-racines) : impossible de dépasser la longueur de la première piste
- "＋ Ajouter une piste" / ✕ Supprimer une piste (min 1)

### 4.2 Configuration des instruments — `/configuration/instruments`

CRUD list :

```
┌─────────────────────────────────────────────────────────┐
│ INSTRUMENTS                                              │
├─────────────────────────────────────────────────────────┤
│ ▶ P      kickdrum    /assets/audio/kickdrum.mp3  ✎ ✕   │
│ ▶ Ts     hihat       /assets/audio/hihat.mp3      ✎ ✕   │
│ ▶ K      rimshot     /assets/audio/rimshot.mp3    ✎ ✕   │
│ ...                                                    │
│ [＋ Ajouter un instrument]                              │
└─────────────────────────────────────────────────────────┘
```

- Ligne : ▶ play + symbole + nom + chemin fichier + actions
- Formulaire d'ajout/édition : symbole, nom, sélecteur de fichier (dialogue natif, formats : mp3/wav/ogg)
- Suppression : confirmation "Oui/Non" (bouton rouge `btn-confirm-delete`)
- Validation : symbole requis, nom requis, fichier audio requis
- L'instrument `.` (silence) est filtré

### 4.3 Configuration des patterns — `/configuration/patterns`

CRUD list :

```
┌─────────────────────────────────────────────────────────┐
│ PATTERNS                                                 │
├─────────────────────────────────────────────────────────┤
│ drum and bass                              ✎ ✕         │
│   P Ts K P Ts K P .                                    │
│ dubstep                                   ✎ ✕         │
│   P (Ts P) Ts P K (Ts P) Ts P K Bw                    │
│ ...                                                    │
│ [＋ Ajouter un pattern]                                 │
└─────────────────────────────────────────────────────────┘
```

- Ligne : nom + phrases (multiligne) + actions
- Formulaire d'ajout/édition : saisie du pattern avec prévisualisation en direct de la grille
- Suppression : confirmation "Oui/Non"

### 4.4 Bibliothèque — `/library`

#### Export

1. L'utilisateur coche les patterns et/ou instruments à exporter (checkboxes)
2. Master checkbox (indéterminé si sélection partielle, type Gmail)
3. Clic "Exporter (N)" → dialogue de sauvegarde natif → fichier `.beatpack`

#### Import

1. Clic "Importer" → dialogue d'ouverture natif → sélection d'un `.beatpack`
2. Analyse du fichier → `ImportPreviewModal` :
   - Liste des patterns et instruments avec leurs métadonnées
   - Pour chaque élément en conflit (slug existant déjà en base) : menu déroulant `Écraser / Ignorer / Renommer`
   - Si "Renommer" : champ texte pour le nouveau nom
3. Clic "Importer" → exécution → résumé (importés/ignorés/erreurs)

#### Structure du fichier `.beatpack`

```
bibliotheque.beatpack/
├── manifest.json
└── audio/
    ├── kickdrum.mp3
    ├── hihat.mp3
    └── ... (un par instrument exporté avec fichier)
```

```json
{
  "version": 1,
  "exportDate": "2026-01-01T00:00:00.000Z",
  "patterns": [
    {
      "slug": "drum-and-bass",
      "name": "drum and bass",
      "sentences": ["P Ts K P Ts K P ."],
      "highlights": [[null, null, null, null, null, null, null, null]]
    }
  ],
  "instruments": [
    {
      "slug": "kickdrum",
      "symbol": "P",
      "name": "kickdrum",
      "audioFile": "audio/kickdrum.mp3"
    }
  ]
}
```

### 4.5 Guide — `/guide`

- Documentation de l'application avec syntaxe, exemples, raccourcis
- 4 boutons de **visites guidées** (driver.js, 16/7/5/5 étapes) : Studio, Instruments, Patterns, Bibliothèque
- Les tours sont joués une seule fois (flag localStorage), réactivables depuis le Guide
- Raccourci `F1` : ouvre `GuideModal` avec la référence des raccourcis clavier

---

## 5. Raccourcis clavier complets

### Studio / Workspace

| Touche | Action |
|--------|--------|
| `F1` | Ouvrir l'aide (GuideModal) |
| `Ctrl + Enter` | Play / Stop |
| `Ctrl + ↑` | BPM +1 (clamp 300) |
| `Ctrl + ↓` | BPM -1 (clamp 1) |

### Saisie (SentenceInput)

| Touche | Action |
|--------|--------|
| `(` avec sélection | Wrapper la séléction dans `()` |
| `)` avec sélection | Wrapper la sélection dans `()` |
| `↑` / `↓` (autocomplete) | Navigation entre suggestions |
| `Space` / `Enter` / `Tab` (autocomplete) | Valider la suggestion sélectionnée |
| `)` (autocomplete) | Valider la suggestion + ajouter `)` |
| `Escape` (autocomplete) | Fermer la liste de suggestions |
| `Ctrl + Espace` | Afficher / Masquer l'autocomplétion |

### Sidebar

| Touche | Action |
|--------|--------|
| `←` / `→` (sur le resizer) | Redimensionner -10 / +10 px |

### Slider BPM

| Action | Résultat |
|--------|----------|
| Molette vers le haut | +1 |
| Molette vers le bas | -1 |

### Modales

| Touche | Action |
|--------|--------|
| `Escape` | Fermer la modale (GuideModal, ImportPreviewModal) |
| Clic en dehors | Fermer la modale |

---

## 6. Flux de données critiques

### 6.1 Flux de composition (Studio → Audio)

```
Saisie texte utilisateur
  │
  ▼
usePatternSession
  ├── changeSentence()       → met à jour la phrase + normalisation (si index 0)
  ├── toggleMute()           → Set<string> de clés "sentenceIndex-tokenIndex"
  └── sentencesForPlayback() → transformSentencesWithMute() remplace muets par '.'
         │
         ▼
parseMultiTrackSteps()
  ├── parseSteps(sentence, validSymbols)   → PatternStep[]
  ├── parseMultiTrackSteps(sentences, symbols) → TrackColumn[]
  │      │
  │      ▼
  │   Grid UI (StepBadge, Column, ColorTooltip, mute)
  │
  ▼ (à la lecture)
SequenceService.preparePattern(sentence)
  ├── Regex /\(([^)]*)\)|(\S+)/g → tokens
  ├── toSequenceNotes(match)
  │     ├── groupe → toGroupNotes(group) → SequenceNote[] (tableau)
  │     └── simple → toSequenceNote(symbol) → SequenceNote (string)
  └── Result : SequenceNotes[] = (string | null | string[])[]
         │
         ▼
AudioEngine.createSequence(tracks)
  ├── Nouveau Tone.Sequence par piste (interval '8n')
  │     callback (time, note) → this.players.player(note).start(time)
  │     * Tone.js dispatche les tableaux (string[]) à la même position
  │
  ├── Tone.Players créés depuis AudioFileBuffer
  └── Tone.Loop (stepCallback → activeStep pour UI)
         │
         ▼
AudioFacade.playPattern() → Tone.Transport.start('+0.1')
```

### 6.2 Flux de résolution des symboles

```
Symbole (ex: "P")
  │
  ▼
InstrumentEngine (singleton)
  ├── getInstrumentNameFromSymbol(symbol) → InstrumentName (string | null)
  ├── getInstrumentFilePathsFromSymbol(symbol) → InstrumentFilePath[]
  │
  ▼
Adapter : toSequenceNote(name) → SequenceNote (identity, string | null)
  │
  ▼
Utilisé par :
  ├── SequenceService.preparePattern() → résolution des noms pour le séquenceur
  └── SequenceService.prepareFilePaths() → résolution des chemins pour le chargement audio
```

### 6.3 Flux d'import/export

```
Export :
  IDs patterns/instruments
    → Repository.getPatternsByIds() / getInstrumentsByIds()
      → Conversion en LibraryPattern / LibraryInstrument
        → archiver (ZIP store level)
          → manifest.json + audio/* → .beatpack

Import :
  .beatpack
    → AdmZip → parseLibraryFile() → LibraryManifest
      → UI : ImportPreviewModal (sélection + conflits)
        → ConflictResolution[]
          → importLibrary()
            → Pour chaque pattern : delete si existant → create
            → Pour chaque instrument : delete si slug/symbol existant → copy audio → create
              → ImportResult { compteurs, erreurs }
```

### 6.4 Flux de chargement audio

```
Renderer :
  [symbole → InstrumentEngine → InstrumentFilePath[]]
    → IPC 'get-audio-buffers'
      → Main : fs.readFileSync() → ArrayBuffer
        → AudioFileBuffer { [instrumentName]: ArrayBuffer }
          → AudioEngine.createPlayers()
            → Tone.Players.add(name, AudioBuffer décodé)
```

---

## 7. Palette de couleurs et thème

### Thème sombre

| Token CSS | Hex | Usage |
|-----------|-----|-------|
| `primary` | `#679ff9` | Accents, symbole, boutons, bordures actives, animations |
| `background` | `#030712` | Fond principal de l'application |
| `surface` | `#1d273c` | Cartes, sections, sidebar |
| `border` | `#1f2937` | Bordures, séparateurs |
| `field` | `#111827` | Fonds de champs, inputs, textareas |
| `text-primary` | `#f3f4f6` | Texte principal (blanc cassé) |
| `text-secondary` | `#6b7280` | Texte secondaire (gris) |
| `text-accent` | `#679ff9` | Texte d'accentuation |
| `button-surface` | `#111827` | Fond des boutons |
| `button-delete` | `#990033` | Survol bouton supprimer |
| `button-edit` | `#679ff9` | Survol bouton éditer |
| `button-confirm-delete` | `#770000` | Fond bouton confirmer suppression |

### Couleurs de highlight (grille)

| Couleur | Rôle |
|---------|------|
| `red` | Marquer un temps fort |
| `blue` | Marquer une liaison |
| `green` | Marquer un ghost note |
| `yellow` | Marquer un accent |
| `orange` | Marquer un syncope |

---

## 8. Animations et retours visuels

| Animation | Déclencheur | Effet |
|-----------|-------------|-------|
| `step-active-pulse` | Colonne active pendant la lecture | scale 1→1.08, brightness 1→1.3, 150ms |
| `glow-pulse-high` | Titre "Beat & Teach" (Home), logo "B" (Header) | text-shadow pulsant 3s |
| `glow-pulse-low` | Titres de section, sidebar-title | text-shadow pulsant subtil 1.5s |
| `rotate` | Cartes Home, wrapper modal | Bordure gradient tournante 8s linéaire |
| Scale 1.05 | Cartes Home au survol | Agrandissement progressif |
| Scale 1.25 | Swatch ColorTooltip au survol | Agrandissement de la pastille |
| Opacity + couleur | Resizer survol/drag | Passe de transparent à primary |
| Opacity 0.4 | Badge mute, boutons disabled | Diminution d'opacité |

---

## 9. Autocomplétion

### Déclenchement
- Dès que l'utilisateur tape dans le textarea, le dernier token (mot après le dernier espace) sert de filtre
- Le préfixe `(` au début du token est ignoré (pour la saisie de groupes)

### Filtrage
- Insensible à la casse
- Match sur le **début** du symbole (`startsWith`)
- Si le token courant est vide → aucune suggestion

### UI
- Liste positionnée absolument aux coordonnées du curseur (via `useCaretPosition` — mirror div)
- Fond surface, bordure border, ombre, arrondi
- Item sélectionné : fond primary, texte background
- Items non sélectionnés : texte secondaire, primary hover

### Navigation
- ↑/↓ pour naviguer (wrap autour)
- Space/Enter/Tab pour confirmer → le dernier token est remplacé, espace ajouté
- `)` pour confirmer et ajouter la parenthèse fermante (saisie groupée)
- Escape pour masquer
- Le dropdown se réaffiche automatiquement au focus ou si le token change

---

## 10. Données initiales (seed)

### 12 patterns

| Slug | Nom | Phrases |
|------|-----|---------|
| drum-and-bass | drum and bass | `["P Ts K P Ts K P ."]` |
| dubstep | dubstep | `["P (Ts P) Ts P K (Ts P) Ts P K Bw"]` |
| funk | funk | `["P Ts P Ts Kch Ts Ts Kch Ts Kch P Kch Kch P Ts Kch"]` |
| funk-2 | funk 2 | `["P (. Kch) . P . P Kch Ts (P Ts) (Ts Kch) (. Ts) (P Ts) (Ts Ts) (P P) (Kch .) Bw"]` |
| reggae | reggae | `["Tum (Ts K) (. K) . (Ts K) (. K) Tum (Ts K) (. K) . (Ts K) (. Tum)"]` |
| jazz | jazz | `["P (P Ts) (Ts P) (Ts Ts) (P K) . Ts (P Ts) (Ts P) K (. Ts) ."]` |
| boom-bap | boom bap | `["P Ts K (Ts P) P P K Ts"]` |
| 23 | 2/3 | `["P Ts Pf", "Eh (. Eh) ."]` |
| 43 | 4/3 | `["P Ts Pf Ts", "Eh (. Eh . ) ( . . Eh) ."]` |
| inward-drag | inward drag | `["P A< F> S< A> F< Eh A< F> S< A> F<"]` |
| drop-it-like-its-hot | drop it like its hot | `["P (Lo P) (Pf Lo) P Lo (P Lo) Pf ."]` |
| alexinho-drum-and-bass | alexinho drum and bass | `["Bwr . Pf (Ts P) (K Kch) (Ts Kch) Pf (Ts P) (K Kch) (Ts Kch) Pf (Ts P) (K Kch) (Ts Kch) Pf Ts"]` |

### 20 instruments

(Voir section 2.1)

---

## 11. Contraintes et validations

| Validation | Niveau | Message | Code |
|------------|--------|---------|------|
| Symbole requis | Instrument | "Le symbole est requis." | `instrument-repository.ts` |
| Nom requis | Instrument | "Le nom est requis." | `instrument-repository.ts` |
| Fichier audio requis | Instrument | "Le fichier audio est requis." | `instrument-repository.ts` |
| Symbole unique (DB) | Instrument | "Un instrument avec ce symbole existe déjà." | `instrument-repository.ts` |
| Slug unique (DB) | Instrument | "Un instrument avec ce nom existe déjà." | `instrument-repository.ts` |
| Nom requis | Pattern | "Le nom est requis." | `pattern-repository.ts` |
| Phrases requises | Pattern | "Les phrases sont requises." | `pattern-repository.ts` |
| Slug unique (DB) | Pattern | "Un pattern avec ce nom existe déjà." | `pattern-repository.ts` |
| Au moins 1 phrase | Pattern | "Au moins une phrase est requise." | `pattern-validator.ts` |
| Phrase non vide | Pattern | "La phrase N est requise." | `pattern-validator.ts` |
| Nom requis (form) | Pattern | Messages du validateur | `pattern-validator.ts` |
| manifest.json absent | Library | "Fichier manifest.json introuvable dans le .beatpack" | `import-library-service.ts` |
| Version non supportée | Library | "Version de manifest non supportée : N" | `import-library-service.ts` |
| Erreur générique IPC | Tous | Message nettoyé via `extractIpcError()` | `util.ts` |

---

## 12. Architecture et organisation du code

### 12.1 Structure par domaine (feature-sliced)

```
src/
├── main/            # Processus main Electron (backend, fichiers, DB, IPC)
│   ├── main.ts      # Point d'entrée, fenêtre, menu
│   ├── preload.ts   # contextBridge : 16 channels typés
│   ├── icpEvents.ts # Handlers IPC (CRUD, audio, library)
│   ├── db/          # SQLite (database.ts, migrations, repositories, services)
│   ├── audio/       # Lecture fichiers audio
│   └── library/     # Import/export ZIP
│
├── renderer/        # Processus renderer React (frontend)
│   ├── App.tsx      # Root : MemoryRouter + 4 providers imbriqués
│   ├── components/  # UI partagée (Header, SideBar, Home, Modal, etc.)
│   ├── hooks/       # Hooks partagés (useResizable, useSlider, etc.)
│   ├── utils/       # Utilitaires (toSnakeCase, removeDuplicates, etc.)
│   └── features/    # Domaines métier
│       ├── pattern/     # Composition, CRUD, grille, parser, validator, mute
│       ├── instruments/ # CRUD, moteur de résolution, légende
│       ├── sequence/    # Résolution pattern→notes audio
│       ├── audio/       # Moteur Tone.js, transport, contexte
│       ├── library/     # Import/export UI
│       ├── autocomplete/# Saisie assistée
│       ├── guide/       # Documentation, raccourcis, modale
│       └── onboarding/  # Visites guidées driver.js
│
└── shared/          # Partagé main/renderer
    ├── models/      # DTOs (PatternDTO, InstrumentDTO, LibraryManifest)
    └── types/       # Types (FilePath, InstrumentFilePath, AudioFileBuffer)
```

### 12.2 Providers (ordre d'imbrication)

```
MemoryRouter
  └── AudioProvider        (playing, activeStep, playTrack, stopTrack, changeBpm)
      └── PatternsProvider (patterns[], setPatterns, error)
          └── InstrumentsProvider (instruments[], setInstruments, error)
              └── GuideModalProvider (isOpen, showGuideModal, hideGuideModal)
                  └── Routes + GuideModal
```

### 12.3 Couches par domaine

Chaque feature suit le même pattern :

```
facade/       → API publique du domaine (ce que les composants appellent)
services/     → Appels IPC (communication main/renderer)
adapters/     → DTO → Modèle frontend
models/       → Interfaces de données
types/        → Types utilitaires spécifiques
engine/       → Logique métier pure (singleton pour InstrumentEngine, AudioEngine)
hooks/        → Logique React (état, effets)
contexts/     → Providers React (état global)
components/   → UI
utils/        → Fonctions pures (parser, validator, mute...)
tests/        → Tests unitaires Jest (Given/When/Then)
```

### 12.4 IPC — 16 channels typés

| Channel | Direction | But |
|---------|-----------|-----|
| `get-all-patterns` | renderer→main | Récupérer tous les patterns |
| `create-pattern` | renderer→main | Créer un pattern |
| `update-pattern` | renderer→main | Modifier un pattern |
| `delete-pattern` | renderer→main | Supprimer un pattern |
| `get-all-instruments` | renderer→main | Récupérer tous les instruments |
| `create-instrument` | renderer→main | Créer un instrument |
| `update-instrument` | renderer→main | Modifier un instrument |
| `delete-instrument` | renderer→main | Supprimer un instrument |
| `open-file-dialog` | renderer→main | Sélecteur de fichier natif |
| `get-audio-buffers` | renderer→main | Charger les fichiers audio |
| `export-library` | renderer→main | Exporter .beatpack |
| `parse-library-file` | renderer→main | Lire le manifest d'un .beatpack |
| `import-library` | renderer→main | Importer .beatpack |
| `save-library-file` | renderer→main | Dialogue de sauvegarde |
| `open-library-file` | renderer→main | Dialogue d'ouverture |
| `get-imported-audio-path` | renderer→main | Chemin de destination audio |

---

## 13. Schémas SQL

```sql
CREATE TABLE instruments (
  id       INTEGER PRIMARY KEY AUTOINCREMENT,
  slug     TEXT NOT NULL UNIQUE,
  symbol   TEXT NOT NULL UNIQUE,
  name     TEXT,
  filepath TEXT
);

CREATE TABLE patterns (
  id         INTEGER PRIMARY KEY AUTOINCREMENT,
  slug       TEXT NOT NULL UNIQUE,
  name       TEXT NOT NULL,
  sentences  TEXT,
  highlights TEXT
);
```

- `sentences` : stocké comme `JSON.stringify(string[])`
- `highlights` : stocké comme `JSON.stringify((string|null)[][])`
- Base de données : `userData/database.db` (WAL mode, foreign keys ON)

---

## 14. Tests

- Framework : Jest avec jsdom
- ts-jest configuré avec `tsconfig.renderer.json`
- Structure : Given / When / Then
- Tests existants :
  - `util.test.ts` — `#removeDuplicates`, `#removeParenthesis`, `#toSnakeCase`, `#extractIpcError`
  - `pattern-parser.test.ts` — `#createStep`, `#createGroup`, `#parseSteps`
  - `sequence-service.test.ts` — `#prepareFilePaths`, `#preparePattern`
  - `sequence-adapter.test.ts` — `#toSequenceNote`
- Commande : `npm run test`
- Mocks nécessaires pour les tests main process : `electron`, `better-sqlite3`

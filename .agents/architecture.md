# Architecture — Beat & Teach

> Spécialisation projet du pack `electron-react-ts`.

## Arborescence

```
src/
├── main/                          # Electron main process
│   ├── main.ts                    # Entry point
│   ├── preload.ts                 # contextBridge
│   ├── icpEvents.ts               # IPC handlers
│   ├── menu.ts                    # App menu
│   ├── util.ts                    # resolveHtmlPath
│   ├── audio/                     # Audio file reading (main process)
│   ├── db/                        # SQLite (migrations, repositories, services)
│   └── library/                   # Import/export services
├── renderer/                      # React renderer process
│   ├── App.tsx                    # Root : createMemoryRouter + 5 providers
│   ├── App.css                    # Global styles, Tailwind components, scrollbar, variables de thème
│   ├── preload.d.ts               # Window.electron type declaration
│   ├── components/                # Shared UI (Home, Header, SideBar, Modal)
│   ├── hooks/                     # Shared hooks (useResizable, useSlider, etc.)
│   ├── utils/                     # Utility functions
│   └── features/                  # Feature modules (domain-driven):
│       ├── audio/                 # Audio engine, Tone.js, context
│       ├── autocomplete/          # Autocomplete with symbol filtering
│       ├── instruments/           # Instrument CRUD, engine, validator
│       ├── pattern/               # Pattern composer, parser, validator
│       ├── sequence/              # Sequence service, adapter, facade
│       ├── library/               # Import/export, preview modal
│       ├── guide/                 # Documentation, shortcuts, tours
│       ├── onboarding/            # driver.js guided tours
│       ├── recorder/              # Enregistrement micro, waveform, WAV
│       └── theme/                 # Thèmes d'accent, ThemeContext, switcher
└── shared/
    ├── models/                    # DTOs (pattern-dto, instrument-dto, library-dto)
    └── types/                     # Types (FilePath, InstrumentFilePath, AudioFileBuffer)
```

## Providers (ordre d'imbrication)

```
ThemeProvider                 (theme, setTheme, data-theme sur <html>)
  └── AudioProvider            (playing, activeStep, playTrack, stopTrack, changeBpm)
      └── PatternsProvider     (patterns[], setPatterns, error)
          └── InstrumentsProvider (instruments[], setInstruments, error)
              └── GuideModalProvider (isOpen, showGuideModal, hideGuideModal)
                  └── RouterProvider (createMemoryRouter)
```

Le thème est lu depuis `localStorage` (`beat-and-teach:theme`) ; le script anti-FOUC de `index.ejs` pose `data-theme` avant le rendu pour `green`/`pink`.

## Communication main ↔ renderer

- **18 channels IPC typés** dans `preload.ts` (type `Channels`).
- Tous les handlers sont enregistrés dans `icpEvents.ts`.
- Exposé via `window.electron.ipcRenderer.invokeMessage(channel, ...args)`.
- Toujours typer le channel — ne pas utiliser de `string` brut.
- Utiliser `extractIpcError()` dans les `catch` renderer pour nettoyer le message avant affichage.

| Channel                   | Direction     | But                                               |
| ------------------------- | ------------- | ------------------------------------------------- |
| `get-all-patterns`        | renderer→main | Récupérer tous les patterns                       |
| `create-pattern`          | renderer→main | Créer un pattern                                  |
| `update-pattern`          | renderer→main | Modifier un pattern                               |
| `delete-pattern`          | renderer→main | Supprimer un pattern                              |
| `get-all-instruments`     | renderer→main | Récupérer tous les instruments                    |
| `create-instrument`       | renderer→main | Créer un instrument                               |
| `update-instrument`       | renderer→main | Modifier un instrument                            |
| `delete-instrument`       | renderer→main | Supprimer un instrument                           |
| `open-file-dialog`        | renderer→main | Sélecteur de fichier natif                        |
| `get-audio-buffers`       | renderer→main | Charger les fichiers audio                        |
| `export-library`          | renderer→main | Exporter `.beatpack`                              |
| `parse-library-file`      | renderer→main | Lire le manifest d'un `.beatpack`                 |
| `import-library`          | renderer→main | Importer `.beatpack`                              |
| `save-library-file`       | renderer→main | Dialogue de sauvegarde                            |
| `open-library-file`       | renderer→main | Dialogue d'ouverture                              |
| `get-imported-audio-path` | renderer→main | Chemin de destination audio                       |
| `save-recorded-audio`     | renderer→main | Écrire un enregistrement WAV sur disque           |
| `open-audio-save-dialog`  | renderer→main | Dialogue de sauvegarde d'un enregistrement `.wav` |

## Base de données

- SQLite (`better-sqlite3`), fichier `database.db` dans le dossier `userData` d'Electron (WAL mode, foreign keys ON ; fichiers `-wal`/`-shm` associés).
- Migrations : `001_initial` (tables + seed), `002_highlights`, `003_reference_frequency`. **Ne jamais modifier une migration existante** : en créer une nouvelle.
- Tables : `instruments` (dont `reference_frequency REAL` nullable), `patterns` (dont `highlights TEXT`) — voir `.agents/specs/spec.txt` §13, amendé par `SP-NOTE-02`.
- `sentences` et `highlights` sont stockés en `TEXT` (JSON) — parser/stringifier systématiquement.
- Seed : 12 patterns et 20 instruments au premier démarrage, uniquement si les tables sont vides.

## Configs TypeScript

| Fichier                      | Rôle                                                                    |
| ---------------------------- | ----------------------------------------------------------------------- |
| `tsconfig.json`              | Base : ES2022, strict, types Jest                                       |
| `tsconfig.main.json`         | Main : `module: node16`, `src/main/**` + `src/shared/**`                |
| `tsconfig.renderer.json`     | Renderer : `module: esnext`, `dom`, `src/renderer/**` + `src/shared/**` |
| `src/renderer/tsconfig.json` | Étend le renderer, fixe `rootDir`                                       |

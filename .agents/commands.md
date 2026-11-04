# Commandes — Beat & Teach

> Spécialisation projet du pack `electron-react-ts`. Le sous-ensemble toujours chargé en contexte est `.agents/workflow-commands.md`.

## Dev

```bash
npm start          # dev complet (webpack + electronmon)
```

## Build / package

```bash
npm run build      # build production (main + renderer)
npm run package    # packaging electron-builder
```

## Tests

```bash
npm test                                # tous les tests (jest)
npx jest --runInBand <fichier|pattern>  # test ciblé
```

## Lint / format

```bash
npm run lint           # eslint
npm run lint:fix       # eslint --fix
npx prettier --write <fichiers>   # format ciblé
```

## Typecheck

```bash
npm run typecheck      # tsc -p tsconfig.renderer.json && tsc -p tsconfig.main.json
```

## Base de données

```bash
# SQLite (better-sqlite3) — migrations gérées dans src/main/db/
```

- Base : `database.db` dans le dossier `userData` (WAL mode ; fichiers `-wal`/`-shm` associés), créée et migrée au démarrage.
- Repartir du seed : supprimer `database.db`, `database.db-wal` et `database.db-shm` ; les migrations re-seedent 12 patterns et 20 instruments.
- **Ne jamais modifier une migration existante** : en créer une nouvelle.

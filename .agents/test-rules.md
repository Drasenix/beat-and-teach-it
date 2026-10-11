# Règles d'écriture des tests — Beat & Teach

> Spécialisation projet du pack `electron-react-ts`.

## Périmètre du TDD

- **Oui** (logique métier) : parser, validators, mute, adapters, engines, services, **utils** (fonctions pures).
- **Non** (frontend) : composants React, hooks UI, contexts — tests après ou pas de tests.

## Framework et config

- **Jest** + **jsdom**, transform `ts-jest` pointant sur `tsconfig.renderer.json` (`jest.config.js`).
- `moduleNameMapper` : assets (`fileMock.js`) et styles (`identity-obj-proxy`).
- Tests main process : mocker `electron` et `better-sqlite3`.

## Localisation et structure

- Fichiers `*.test.ts` / `*.test.tsx`.
- Blocs `describe('#nomDeFonction')`, pattern **Given / When / Then**.
- Une assertion principale par test ; messages explicites.

## Exécution

- **Ciblé** (subagents `@test` / `@dev`) : `npx jest --runInBand <fichier|pattern>`.
- **Global** (Build uniquement, fin de workflow) : `npm test`.
- **Vérification systématique** : après toute création/modification de test ou de code métier, lancer `npm test`.
- Détails et format de rapport : `.agents/skills/tests/SKILL.md`.

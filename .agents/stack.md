# Stack — Beat & Teach

> Spécialisation projet du pack `electron-react-ts`. Versions issues de `package.json`.

## Runtime

- **Electron** : 35.x
- **Node.js** : `>=14` (socle `electron-react-boilerplate`)
- **Langage** : TypeScript 5.8 (`strict`)

## Dépendances principales

| Package | Version | Rôle |
|---|---|---|
| `react` | 19.x | UI renderer |
| `react-dom` | 19.x | Rendu DOM |
| `react-router-dom` | 7.x | Routing (`MemoryRouter`) |
| `tone` | 15.x | Moteur audio (Web Audio / séquenceur) |
| `better-sqlite3` | — | Base de données locale (main) |
| `archiver` | 7.x | Export `.beatpack` (ZIP) |
| `adm-zip` | 0.5.x | Import `.beatpack` |
| `lucide-react` | — | Icônes |
| `driver.js` | 1.4.x | Visites guidées |

## Outillage

| Outil | Rôle |
|---|---|
| `webpack` (`.erb/`) | Bundle main / renderer / preload |
| `electron-builder` | Packaging |
| `tailwindcss` 3 | Styles |
| `jest` + `ts-jest` + `jest-environment-jsdom` | Tests |
| `eslint` (`eslint-config-erb`) + `prettier` | Lint / format |

# Conventions UI — Beat & Teach

> Spécialisation projet du pack `electron-react-ts`.

- **Tailwind CSS 3** avec thème custom dans `tailwind.config.js`.
- Styles globaux, composants (`@layer components`) et scrollbar custom dans `App.css` (scrollbar 3px, couleur primaire).
- `.content-page` : `bg-background h-screen overflow-y-auto`.
- `.section-collapsible.open` : `flex flex-col flex-1 min-h-0` (le scroll doit être sur `.section-collapsible.open`, pas sur `.sidebar-list`).

## Palette (thème sombre)

| Token | Hex | Usage |
|---|---|---|
| `primary` | `#679ff9` | Accents, boutons, bordures actives |
| `background` | `#030712` | Fond principal |
| `surface` | `#1d273c` | Cartes, sections, sidebar |
| `border` | `#1f2937` | Bordures, séparateurs |
| `field` | `#111827` | Fonds de champs |
| `text-primary` | `#f3f4f6` | Texte principal |
| `text-secondary` | `#6b7280` | Texte secondaire |
| `button-delete` | `#990033` | Survol supprimer |
| `button-confirm-delete` | `#770000` | Confirmer suppression |

## Couleurs de highlight (grille)

`red` (temps fort) · `blue` (liaison) · `green` (ghost note) · `yellow` (accent) · `orange` (syncope).

## Animations

`step-active-pulse` (colonne active) · `glow-pulse-high` / `glow-pulse-low` (titres) · `rotate` (bordures gradient) · scale au survol (cartes, swatches).

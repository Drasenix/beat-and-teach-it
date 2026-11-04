# Conventions UI — Beat & Teach

> Spécialisation projet du pack `electron-react-ts`.

## Tailwind et variables de thème

- **Tailwind CSS 3** avec thème custom dans `tailwind.config.js`.
- Les tokens couleur sont des **variables CSS** : `tailwind.config.js` mappe sur `rgb(var(--bt-*) / <alpha-value>)`, ce qui conserve les opacités (`bg-primary/10`).
- `App.css` définit `:root` (thème bleu par défaut), puis `[data-theme='green']` et `[data-theme='pink']`.
- Les clés `text.*` et `button.*` produisent des classes à double préfixe (`text-text-secondary`, `bg-button-surface`) : c'est normal.
- **Ne jamais écrire une couleur hex en dur dans le JSX** : utiliser un token (`bg-primary`, `text-text-secondary`, `border-border`…). Exceptions : les 5 couleurs de highlight (constantes sémantiques fixées dans `StepTooltip`) et les rendus canvas (`waveform-renderer`), qui ne consomment pas les variables CSS.

## Thèmes

| Thème         | `data-theme`   | Accent                                    | Hex       |
| ------------- | -------------- | ----------------------------------------- | --------- |
| Bleu (défaut) | aucun attribut | `primary` / `text.accent` / `button.edit` | `#679ff9` |
| Vert          | `green`        | idem                                      | `#4ade80` |
| Rose          | `pink`         | idem                                      | `#f472b6` |

- Seuls les accents changent ; les gris (fonds, surfaces, bordures, champs, textes) sont identiques dans les trois thèmes.
- `ThemeProvider` applique `data-theme` sur `<html>` ; pour `blue`, il retire l'attribut (valeurs `:root`).
- Le choix est persisté dans `localStorage` (`beat-and-teach:theme`) ; le script anti-FOUC de `index.ejs` ne pose que `green`/`pink`.
- Une valeur absente ou invalide retombe sur `blue`, sans erreur.
- **Hors thème** : les 5 couleurs de highlight de la grille, le rouge d'erreur et le backdrop des modales.
- Transition de couleur ciblée 150–200 ms ; jamais de transition universelle sur `*`.

## Palette (thème bleu par défaut)

| Token                   | Hex       | Usage                              |
| ----------------------- | --------- | ---------------------------------- |
| `primary`               | `#679ff9` | Accents, boutons, bordures actives |
| `background`            | `#030712` | Fond principal                     |
| `surface`               | `#1d273c` | Cartes, sections, sidebar          |
| `border`                | `#1f2937` | Bordures, séparateurs              |
| `field`                 | `#111827` | Fonds de champs                    |
| `text-primary`          | `#f3f4f6` | Texte principal                    |
| `text-secondary`        | `#6b7280` | Texte secondaire                   |
| `text-accent`           | `#679ff9` | Texte d'accentuation               |
| `button-surface`        | `#111827` | Fond des boutons                   |
| `button-delete`         | `#990033` | Survol supprimer                   |
| `button-edit`           | `#679ff9` | Survol éditer                      |
| `button-confirm-delete` | `#770000` | Confirmer suppression              |

## Couleurs de highlight (grille)

`red` (temps fort) · `blue` (liaison) · `green` (ghost note) · `yellow` (accent) · `orange` (syncope).

Sémantique pédagogique **fixe** : ces couleurs ne suivent pas le thème et ne se personnalisent pas.

## Layout

- `.content-page` : `bg-background h-screen overflow-y-auto`.
- `.section-collapsible.open` : `flex flex-col flex-1 min-h-0` (le scroll doit être sur `.section-collapsible.open`, pas sur `.sidebar-list`).

## Animations

`step-active-pulse` (colonne active) · `glow-pulse-high` / `glow-pulse-low` (titres) · `rotate` (bordures gradient) · scale au survol (cartes, swatches) · transition de thème 150–200 ms ciblée.

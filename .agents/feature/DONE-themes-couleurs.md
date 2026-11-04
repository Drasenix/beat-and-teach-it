# Feature: Thèmes de couleurs néon

> Clôturé le 2026-08-05 · test global 260 verts · version de travail : `git show 66eeb90:.agents/feature/DONE-themes-couleurs.md`

## Résumé métier

Trois thèmes d'accent (bleu par défaut, vert, rose) choisis depuis un sélecteur palette en bas de la sidebar. Seuls les accents changent ; les gris, les highlights, le rouge d'erreur et le backdrop des modales restent fixes. Le choix est persisté localement (`beat-and-teach:theme`) et restauré au démarrage sans flash, avec repli silencieux sur le bleu si la valeur est absente ou invalide. La bascule est immédiate, avec une transition de 150–200 ms ciblée.

## Scénarios évolution — nouveau comportement

> Photo de l'étape 2 ; les corrections de clôture (dont la suppression de `swatchClass`) sont dans « Leçons de réalisation ».

Périmètre des cas : logique pure testable uniquement (utils du slice `features/theme/` : registry `themes.ts` et helpers de persistance localStorage). Pas de cas pour les composants React (ThemeSwitcher, ThemeContext) ni pour le CSS (frontend, hors TDD). Clé localStorage : `beat-and-teach:theme` (tranchée).

#### Registry `utils/themes.ts` — liste des thèmes

**Cas nominaux**

- Given: le registry des thèmes est chargé
  When: on lit la liste des thèmes
  Then: elle contient exactement 3 thèmes, d'ids `blue`, `green`, `pink`, chacun avec un label, un `accentHex` et une classe de pastille
- Given: le registry des thèmes est chargé
  When: on lit les accents des 3 thèmes
  Then: `blue` a l'accent `#679ff9`, `green` a l'accent `#4ade80`, `pink` a l'accent `#f472b6` (valeurs exactes, distinctes entre elles)
- Given: le registry des thèmes est chargé
  When: on appelle `getThemeById('blue')`
  Then: elle retourne le thème `blue` (id, label, accentHex, swatchClass complets)
- Given: le registry des thèmes est chargé
  When: on appelle `getThemeById('green')` puis `getThemeById('pink')`
  Then: chacune retourne le thème correspondant (mêmes champs complets que `blue`)
- Given: le registry des thèmes est chargé
  When: on appelle `isValidThemeId('blue')`, `isValidThemeId('green')` et `isValidThemeId('pink')`
  Then: les trois retournent `true`

**Cas limites**

- Given: le registry des thèmes est chargé
  When: on appelle `getThemeById('purple')` (id inconnu)
  Then: elle retourne `null` (aucun thème), sans erreur
- Given: le registry des thèmes est chargé
  When: on appelle `isValidThemeId('purple')` (id inconnu)
  Then: elle retourne `false`
- Given: le registry des thèmes est chargé
  When: on appelle `getThemeById('')` (chaîne vide)
  Then: elle retourne `null`, sans erreur
- Given: le registry des thèmes est chargé
  When: on appelle `isValidThemeId('')` (chaîne vide)
  Then: elle retourne `false`
- Given: le registry des thèmes est chargé
  When: on appelle `getThemeById('Blue')` (casse différente)
  Then: elle retourne `null` (comparaison exacte, sensible à la casse)
- Given: le registry des thèmes est chargé
  When: on appelle `isValidThemeId('blue ')` (espace de fin)
  Then: elle retourne `false` (aucune normalisation de la valeur)
- Given: le registry des thèmes est chargé
  When: on lit la liste des thèmes puis on la modifie
  Then: la liste source du registry n'est pas affectée (immutabilité)

**Cas d'erreur**

- Given: le registry des thèmes est chargé
  When: on appelle `getThemeById(undefined)` puis `getThemeById(null)` (valeurs non-string)
  Then: chaque appel retourne `null`, sans lever d'exception
- Given: le registry des thèmes est chargé
  When: on appelle `isValidThemeId(undefined)` puis `isValidThemeId(123)` (valeurs non-string)
  Then: chaque appel retourne `false`, sans lever d'exception

#### Persistance — helpers localStorage (`getStoredTheme` / `saveTheme`)

**Cas nominaux**

- Given: la clé `beat-and-teach:theme` contient la valeur `'green'`
  When: on appelle `getStoredTheme()`
  Then: elle retourne `'green'`
- Given: un thème valide `'pink'`
  When: on appelle `saveTheme('pink')`
  Then: la clé `beat-and-teach:theme` contient désormais la valeur `'pink'`
- Given: la clé `beat-and-teach:theme` contient `'blue'`
  When: on appelle `saveTheme('green')`
  Then: la clé contient `'green'` (écrasement de la valeur précédente)

**Cas limites**

- Given: la clé `beat-and-teach:theme` est absente du localStorage
  When: on appelle `getStoredTheme()`
  Then: elle retourne `null` (aucun thème enregistré)
- Given: la clé `beat-and-teach:theme` contient `'purple'` (id inconnu)
  When: on appelle `getStoredTheme()`
  Then: elle retourne `null` (valeur invalide non retournée)
- Given: la clé `beat-and-teach:theme` contient une valeur vide `''`
  When: on appelle `getStoredTheme()`
  Then: elle retourne `null`
- Given: la clé `beat-and-teach:theme` contient une valeur malformée (ex. `'42'`, objet JSON)
  When: on appelle `getStoredTheme()`
  Then: elle retourne `null`, sans lever d'exception
- Given: un id invalide `'purple'`
  When: on appelle `saveTheme('purple')`
  Then: rien n'est écrit dans le localStorage (la clé reste dans son état antérieur), sans erreur

**Cas d'erreur**

- Given: la lecture du localStorage échoue (exception levée par l'API)
  When: on appelle `getStoredTheme()`
  Then: elle retourne `null`, sans propager l'exception
- Given: l'écriture dans le localStorage échoue (exception levée par l'API, ex. quota)
  When: on appelle `saveTheme('blue')`
  Then: l'exception n'est pas propagée (l'appel se termine sans erreur, la préférence n'est simplement pas persistée)

#### Résolution du thème initial (lecture au démarrage + registry)

**Cas nominaux**

- Given: la clé `beat-and-teach:theme` contient `'green'` et le registry est chargé
  When: on résout le thème initial de l'application
  Then: le thème résolu est `green`
- Given: la clé `beat-and-teach:theme` contient `'pink'`
  When: on résout le thème initial de l'application
  Then: le thème résolu est `pink`

**Cas limites**

- Given: aucune valeur enregistrée pour `beat-and-teach:theme`
  When: on résout le thème initial de l'application
  Then: le thème résolu est `blue` (défaut, RG15)
- Given: la clé `beat-and-teach:theme` contient `'purple'` (id inconnu)
  When: on résout le thème initial de l'application
  Then: le thème résolu est `blue`, sans erreur ni log d'échec (RG17)
- Given: la clé `beat-and-teach:theme` contient une valeur vide ou malformée
  When: on résout le thème initial de l'application
  Then: le thème résolu est `blue`, sans erreur (RG17)

**Cas d'erreur**

- Given: la lecture du localStorage échoue (exception)
  When: on résout le thème initial de l'application
  Then: le thème résolu est `blue` (fallback), sans propager l'exception (RG17)

## Dette connue

| #    | Edge case / sujet                                                                                       | Repris par    | Justification                                                                |
| ---- | ------------------------------------------------------------------------------------------------------- | ------------- | ---------------------------------------------------------------------------- |
| EC-1 | Canvas waveform du recorder non thématisé (couleurs bleues par défaut)                                  | V2            | Le canvas ne consomme pas les variables CSS ; un re-render serait nécessaire |
| EC-2 | `docs/index.html` (page statique) non thématisée                                                        | —             | Hors bundle, hors périmètre V1                                               |
| EC-3 | `.agents/specs/spec.txt` §7 non modifié                                                                 | `SP-THEME-01` | Le spec est immuable ; l’amendement vit dans le spec parallèle               |
| EC-4 | Thèmes supplémentaires, thème clair, thème par pattern, personnalisation libre, import/export de thèmes | —             | Exclus V1                                                                    |
| EC-5 | Vérification visuelle runtime (contraste, FOUC, popover, transition) non automatisée                    | —             | Contrôle manuel à la clôture                                                 |

Couverts : les cas GWT du registry, de la persistance et du fallback (32 tests).

## Arbre de flux

**Démarrage**

```
index.ejs (script inline, avant le bundle)
  localStorage.getItem('beat-and-teach:theme')
    ├─ 'green' | 'pink' → document.documentElement.dataset.theme = id   (pas de flash)
    └─ sinon (absent/invalide) → rien (blue implicite, :root)

App.tsx → <ThemeProvider> (externe)
  useState(() => getThemeById(resolveInitialTheme()) ?? DEFAULT_THEME)
    ├─ resolveInitialTheme() → getStoredTheme() ?? 'blue'   (try/catch muet)
    │    └─ localStorage.getItem → isValidThemeId → ThemeId | null
    └─ useEffect([theme.id]) → dataset.theme = id | delete (si blue)   (idempotent avec le script)

CSS (App.css)
  :root (blue) / [data-theme='green'|'pink'] (3 variables d'accent)
    → tailwind : rgb(var(--bt-*) / <alpha-value>)  (classes @apply)
    → App.css  : var(--bt-primary)                 (keyframes glow, scrollbar, resizer, slider, accent-color)
    → HomeLogoSvg : rgb(var(--bt-*))               (fill/stroke SVG)
  → premier rendu React déjà dans le bon thème
```

**Changement de thème**

```
ThemeSwitcher (Header, bas de sidebar)
  clic pastille (accentHex)
    → useThemeContext().setTheme(themeId)
      ├─ setThemeState(getThemeById(themeId) ?? DEFAULT_THEME)   → re-render
      ├─ saveTheme(themeId)                                      → localStorage.setItem (try/catch)
      └─ useEffect → document.documentElement.dataset.theme = id (ou delete si blue)
        → CSS : [data-theme] surcharge --bt-primary/-text-accent/-button-edit
          → classes tailwind + var(--bt-*) + SVG recalculés     → transition 150ms ciblée
  fermeture popover (clic extérieur / Escape)
```

## Décisions & Justifications — ADR

| #   | Décision                                                                                                                                                                                                                                                                                                                                                                                                                        | Raison                                                                                                                                                                           |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | 13 variables CSS `--bt-*` en **triplets RGB** (`--bt-primary: 103 159 249`), définies sur `:root` dans App.css ; blocs `[data-theme='green']` et `[data-theme='pink']` ne surchargent que `--bt-primary`, `--bt-text-accent`, `--bt-button-edit` (RG18)                                                                                                                                                                         | `rgb(var(--bt-*) / <alpha-value>)` exige des triplets ; les gris restent identiques                                                                                              |
| 2   | `data-theme` posé sur `document.documentElement` ; **jamais `'blue'`** (blue = absence d'attribut, valeurs `:root`)                                                                                                                                                                                                                                                                                                             | Évite un bloc `[data-theme='blue']` dupliquant `:root` ; scrollbars/body couverts                                                                                                |
| 3   | `tailwind.config.js` : 13 tokens → `rgb(var(--bt-*) / <alpha-value>)`                                                                                                                                                                                                                                                                                                                                                           | Compatible avec l'unique usage alpha `bg-background/70`                                                                                                                          |
| 4   | App.css : les ~25 `theme('colors.primary')` → `var(--bt-primary)` (keyframes glow incluses) ; 5 hex slider BPM → `var(--bt-primary)`                                                                                                                                                                                                                                                                                            | Les keyframes résolvent les var() dynamiquement ; suppression de tous les hex applicatifs                                                                                        |
| 5   | `HomeLogoSvg.tsx` : les 7 hex → `rgb(var(--bt-*))` en attributs fill/stroke (jamais `var(--bt-*)` seul)                                                                                                                                                                                                                                                                                                                         | Un triplet nu est invalide en attribut SVG ; `rgb(var())` est résolu par le moteur CSS                                                                                           |
| 6   | **Script inline anti-FOUC dans `index.ejs`** (autorise `'unsafe-inline'` déjà en place) : lit `beat-and-teach:theme`, pose `data-theme` si `'green'`/`'pink'`, try/catch muet                                                                                                                                                                                                                                                   | Zéro flash bleu au démarrage ; duplication volontaire minimale (2 ids en dur)                                                                                                    |
| 7   | `ThemeProvider` = **provider le plus externe** dans App.tsx (au-dessus d'AudioProvider)                                                                                                                                                                                                                                                                                                                                         | ThemeSwitcher dans Header (layout du routeur) ; préférence globale sans dépendance aux données                                                                                   |
| 8   | State du contexte = `ThemeEntry` complet ; lecture initiale dans `useState(() => getThemeById(resolveInitialTheme()) ?? DEFAULT_THEME)` ; application `data-theme` dans un `useEffect` de synchronisation ; `setTheme(id)` = setState + `saveTheme(id)`                                                                                                                                                                         | Lecture hors effet (pas de dérivation), synchro DOM légitime ; une seule source d'écriture CSS                                                                                   |
| 9   | `themes` exporté via **Proxy getter-copie** (chaque méthode de tableau s'exécute sur `[...registry]`) ; `getThemeById(id: unknown): ThemeEntry \| null` ; `isValidThemeId(id: unknown): boolean` ; `DEFAULT_THEME` exporté                                                                                                                                                                                                      | Contrainte du test de mutation ; signatures tolérantes `unknown` ; pas de cast dans le contexte                                                                                  |
| 10  | `theme-storage.ts` : `getStoredTheme(): ThemeId \| null` (valide via `getThemeById(raw)?.id ?? null`), `saveTheme(theme: string): void` (no-op si invalide), `resolveInitialTheme(): ThemeId` (= `getStoredTheme() ?? 'blue'`), try/catch muet partout                                                                                                                                                                          | GWT cas limites/erreurs ; `saveTheme` typé `string` (le test passe `'purple'`)                                                                                                   |
| 11  | Slice `features/theme/` : `types/theme-types.ts` (ThemeId, ThemeEntry), `utils/themes.ts`, `utils/theme-storage.ts`, `contexts/ThemeContext.tsx` (provider + `useThemeContext`), `components/ThemeSwitcher.tsx`. **Pas de `hooks/useTheme.ts` ni de facade/service** (pas d'IPC, pas de DTO)                                                                                                                                    | Pattern existant `InstrumentsContext.tsx` (provider + hook même fichier) ; les composants appellent le contexte directement (feature-sliced sans façade quand pas de couche IPC) |
| 12  | ThemeSwitcher : bouton `Palette` (lucide-react) en bas de `aside.bar-aside` (wrapper `mt-auto` dans Header), popover absolu **vers le haut** (`bottom-full`), 3 pastilles `swatchClass` (tailwind arbitraire `bg-[#679ff9]` etc.), pastille active marquée (ring), fermeture clic extérieur + Escape                                                                                                                            | Bas d'écran → le popover doit monter ; pattern Autocomplete (absolu) + Modal (Escape/clic extérieur)                                                                             |
| 13  | Transition 150ms : règle CSS ciblée dans App.css sur une **liste explicite de classes de conteneurs/zones** (`.bar-aside`, `.content-page`, `.transport-bar`, `.daw-sidebar`, `.form-card`, `.item-row`, `.library-section`, `.modal-content`, `.sidebar-item`, `.instrument-card`, `.step-badge-valid`, etc.) avec `transition: background-color 150ms ease, color 150ms ease, border-color 150ms ease` — pas de `*` universel | Tranche GO étape 2 (classes ciblées) ; les éléments interactifs ont déjà `transition-colors` 100-200ms (dans la fourchette demandée)                                             |
| 14  | Clé localStorage `beat-and-teach:theme` (littéral partagé entre utils et script FOUC — dupliqué volontairement dans le script inline)                                                                                                                                                                                                                                                                                           | Tranchée au GWT ; pattern onboarding existant (`*_tour_seen`)                                                                                                                    |

**Corrections d’implémentation (étape 5 et rounds de clôture).** Le registry Tailwind et `:root` comptent **12** variables, pas 13 (le chiffre « 13 » de la décision 1 est une erreur de comptage). Les usages de `var(--bt-*)` dans une propriété couleur sont en réalité `rgb(var(--bt-*))` (les variables sont des triplets RGB). La décision 12 a été corrigée au Round 3 : `swatchClass` a été supprimé, la couleur des pastilles vient de `accentHex` en style inline.

## Leçons de réalisation

- Popover du sélecteur coupée à gauche au premier rendu (centrage sur un bouton de 40 px dans une sidebar de 64 px) → repositionnement à droite de la sidebar (`left-full ml-2`), pastilles réduites → pour un popover ancré dans une barre étroite, ancrer sur le côté, pas au centre.
- Pastilles invisibles : `swatchClass` en classes Tailwind arbitraires dans `themes.ts` (fichier `.ts` hors du scan `src/**/*.tsx`) → suppression de `swatchClass`, couleur via `style={{ backgroundColor: accentHex }}` → ne jamais mettre de classes Tailwind générées dynamiquement dans un fichier non scanné ; une couleur de donnée passe par un style inline.
- `var(--bt-*)` nu invalide dans `text-shadow`/`accent-color` (variables en triplets RGB) → `rgb(var(--bt-*))` partout → pour un token triplet, toujours l’envelopper dans `rgb()`.
- Lint : catch muet interdit (`no-empty`, `no-useless-return`) → helper `writeStoredTheme(): boolean` avec double `return false` → garder un retour non-void dans un catch pour satisfaire le lint sans `eslint-disable`.
- Prettier a reformaté `themes.test.ts` (`.some(...)`) avant le lint global → aucun changement sémantique → formater avant de lancer le lint global.

## Vigilances

- Classes Tailwind arbitraires dans un fichier `.ts` : jamais générées (scan `src/**/*.tsx`). Une couleur de donnée → `style` inline ou token existant.
- Le script anti-FOUC de `index.ejs` duplique la clé `beat-and-teach:theme` et les ids `green`/`pink` : toute évolution de la clé ou du registre doit être répercutée.
- Le canvas waveform du recorder reste hors thème (couleurs explicites) : ne pas supposer qu’il suit `data-theme`.

## Traçabilité

- **Spec et US** : §7 (palette), RG15–RG18.
- **Spec parallèle** : `SP-THEME-01` (AMENDE).
- **Conventions établies** : `.agents/ui-rules.md` (3 thèmes, variables `--bt-*`, highlights fixes) · `.agents/component-rules.md` (ordre des providers) · `.agents/architecture.md` (providers, `theme/`) · `.agents/conventions.md` (Thème).
- **Utilitaires référencés** : `themes.ts`, `theme-storage.ts` (voir `.agents/reference-utilitaires.md`).
- **Tests** : `themes.test.ts`, `theme-storage.test.ts` (32 cas) ; test global 260 verts.

> Écarté à la compaction (texte complet : version de travail) : Contexte (→ Résumé métier) · Étape 1 — Specs (pilotage du développement ; périmètre → Dette connue) · Étape 2 — GWT (→ Scénarios évolution) · Étape 3 — Tests rouges (pilotage du développement ; scénarios → Scénarios évolution) · Étape 4 — Architecture (pilotage du développement ; décisions → ADR, flux → Arbre de flux, cas limites → Dette connue) · Étape 5 — Développement (pilotage du développement ; divergences et rounds → Leçons) · Étape 6 — Review (trace de revue) · Avancement (pilotage du développement) · Notes / décisions (→ ADR / Résumé métier)

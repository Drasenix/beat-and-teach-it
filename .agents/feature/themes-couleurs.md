# Feature: Thèmes de couleurs néon

## Contexte
Demande : imaginer plusieurs thèmes de couleurs restant dans l'esprit néon (gris/bleu existant, ajout de gris/vert et gris/rose), avec un moyen propre et élégant de changer de thème, inspiré d'applications existantes (Linear, Discord, GitHub). Décisions utilisateur actées : icône palette + popover swatches en bas de la sidebar, accent vert #4ade80, accent rose #f472b6, transition douce 150-200ms.

## Étape 1 — Specs
STATUS: OK

### Problème
- L'application a un thème unique gris/bleu (accent `primary: #679ff9`) codé en dur : tokens Tailwind dans `tailwind.config.js` (13 tokens) et hex littéraux dans `App.css` (~30 usages de `theme('colors.primary')`, slider BPM `#679ff9` ×5) et `HomeLogoSvg.tsx` (7 hex).
- Aucun moyen pour l'utilisateur de personnaliser l'ambiance visuelle ; changer le thème aujourd'hui exigerait de modifier du code.
- Objectif : offrir 3 thèmes "néon" (gris/bleu existant + gris/vert + gris/rose) et un sélecteur élégant, sans dupliquer les styles.

### Solution cible

**1. Mécanisme CSS variables + `data-theme`**
- Les couleurs deviennent des **CSS custom properties** définies dans `App.css` : `:root` (valeurs du thème bleu par défaut) + blocs `[data-theme='green']` et `[data-theme='pink']` qui surchargent les variables.
- Une seule variable par token couleur (ex. `--bt-primary`, `--bt-background`, ...) — l'accent change (primary, text.accent, button.edit), les gris (background, surface, border, field, text.primary, text.secondary, button.surface) restent **identiques** entre les 3 thèmes.
- `tailwind.config.js` : les 13 tokens mappent sur `rgb(var(--bt-*) / <alpha-value>)` pour conserver les opacités existantes (`bg-primary/10` etc.).
- `App.css` : les ~30 usages de `theme('colors.primary')` (keyframes glow, scrollbar, resizer, boutons) et les hex en dur (slider BPM) migrent vers `var(--bt-*)`. Les variables étant définies sur `:root`/`[data-theme]`, les keyframes CSS restent valides (résolues à l'exécution).
- `HomeLogoSvg.tsx` : les 7 hex du SVG migrent vers `var(--bt-*)` (le SVG étant dans le DOM, les variables s'y appliquent).

**2. Slice `features/theme/`** (feature-sliced)
- `utils/themes.ts` : **registry pur** des thèmes — `{ id: 'blue' | 'green' | 'pink', label, accentHex, swatchClass }[]` + helpers (`getThemeById`, `isValidThemeId`). Seule logique testable unitairement (TDD).
- `contexts/ThemeContext.tsx` : provider qui applique `data-theme` sur `<html>` (via `document.documentElement.dataset.theme`), expose `{ theme, setTheme }`, et gère la persistance. Rendu côté renderer (pas de `useEffect` de dérivation : la lecture initiale se fait dans le state initial, l'application du `data-theme` est un effet de synchronisation DOM, légitime).
- `components/ThemeSwitcher.tsx` : UI du sélecteur (icône + popover).

**3. Switcher UI (inspiration Linear/Discord/GitHub)**
- Icône **palette** (lucide-react) en **bas de la barre latérale** (`aside.bar-aside` dans `Header.tsx`).
- Clic → **popover** avec 3 pastilles circulaires (bleu `#679ff9` / vert `#4ade80` / rose `#f472b6`), pastille active marquée (bordure/check).
- Clic sur une pastille → changement immédiat de thème ; fermeture du popover (clic extérieur ou Escape, cohérent avec les modales existantes).

**4. Transition douce**
- Transition 150-200ms sur les propriétés de couleur des éléments affectés par le thème (background, color, border, box-shadow pertinents) — à définir proprement (classe utilitaire ou règles ciblées dans App.css), pas de transition universelle sur `*` (risque de perf/artefacts).

**5. Persistance**
- Clé localStorage dédiée (pattern existant dans `OnboardingDriver.tsx`) : ex. `beat-and-teach:theme`.
- Au démarrage : lecture de la clé → application du thème ; valeur absente → bleu (défaut) ; valeur invalide → bleu sans erreur.

### Périmètre

**Inclus (V1)**
- 3 thèmes : `blue` (défaut, existant), `green` (accent `#4ade80`), `pink` (accent `#f472b6`).
- Bascule complète des couleurs applicatives : tokens Tailwind + App.css (glow, scrollbar, resizer, boutons, slider BPM) + HomeLogoSvg.
- Sélecteur palette en bas de sidebar + popover 3 pastilles.
- Transition douce 150-200ms.
- Persistance localStorage + fallback défaut.
- Slice `features/theme/` avec registry pur (testable).

**Exclu (hors V1)**
- Les 5 couleurs de highlight sémantiques du `StepTooltip.tsx` (rouge/bleu/vert/jaune/orange) : fixes, hors thème (sémantique pédagogique).
- Le rouge d'erreur (`red-400`/`text-red-*` dans les validations UI) : fixe, hors thème (sémantique d'erreur).
- Le fond/overlay des modales (backdrop) : reste tel quel.
- Thèmes supplémentaires, thème clair, mode sombre/clair, thème par pattern ou par session.
- Personnalisation utilisateur (choix libre de couleur), import/export de thèmes.

### Règles de gestion impactées ou nouvelles

| Réf | Règle | Statut |
|-----|-------|--------|
| RG15 | Le thème par défaut est `blue` (gris/bleu existant) — appliqué si aucune préférence enregistrée | **Nouvelle** |
| RG16 | Le choix de thème est persistant en localStorage (clé dédiée) et restauré au démarrage | **Nouvelle** |
| RG17 | Si la valeur localStorage est absente ou invalide (id inconnu), le thème `blue` est appliqué sans erreur | **Nouvelle** |
| RG18 | Le thème ne modifie que les variables d'accent (primary, text.accent, button.edit) et le logo ; les gris restent identiques dans les 3 thèmes | **Nouvelle** |
| — | `documentation-métier.md` §7 (palette de couleurs) : mise à jour proposée pour documenter les 3 thèmes et le mécanisme CSS variables — à valider avec l'utilisateur (ne pas modifier sans accord, cf. agent.md §14) | Impact documentation |

### Points ouverts / questions restantes
1. **Dégradés du logo** : `HomeLogoSvg.tsx` contient des dégradés multicolores (7 hex) — tous les hex doivent-ils devenir des variables (logo accent-é en entier) ou seulement la partie accent, en gardant les autres teintes fixes ? → à trancher (hypothèse : thémer l'intégralité pour la cohérence "accent = couleur du thème").
2. **Portée de la transition** : appliquer la transition sur tous les éléments à fond/couleur (plus fluide mais risque de flash/perf) ou seulement sur les zones principales (sidebar, header, boutons, grille) ? → hypothèse : classes ciblées sur les conteneurs et éléments interactifs majeurs.
3. **Nom de la clé localStorage** : confirmer `beat-and-teach:theme` (pattern de nommage des clés existantes à vérifier).
4. **Contraste** : valider visuellement le contraste du texte secondary sur les fonds des 3 thèmes (les gris restent identiques, donc impact faible — vérification purement visuelle).

## Étape 2 — GWT
STATUS: OK

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

---

Rappels pour l'étape 3 : les tests ciblent `utils/themes.ts` (registry) et les helpers de persistance (`getStoredTheme`/`saveTheme`) + la fonction de résolution du thème initial — jsdom fournit `localStorage` (mockable pour les cas d'échec). Aucun test frontend (ThemeSwitcher, ThemeContext, CSS) n'est attendu.

## Étape 3 — Tests rouges
STATUS: OK

### Fichiers créés

| Fichier | Couvre |
|---------|--------|
| `src/renderer/features/theme/tests/themes.test.ts` | Registry `utils/themes.ts` : liste `themes`, `getThemeById`, `isValidThemeId` — 18 cas |
| `src/renderer/features/theme/tests/theme-storage.test.ts` | Persistance `utils/theme-storage.ts` : `getStoredTheme`, `saveTheme`, `resolveInitialTheme` — 18 cas |

Total : 36 cas (4 describe `#methodName`, structure Given/When/Then dans les noms de `it`).

### Chemins cibles des modules (implémentation étape 5)

- `src/renderer/features/theme/utils/themes.ts` → exports attendus : `themes` (liste), `getThemeById(id: unknown)`, `isValidThemeId(id: unknown)`
- `src/renderer/features/theme/utils/theme-storage.ts` → exports attendus : `getStoredTheme()`, `saveTheme(theme: string)`, `resolveInitialTheme()`

### Vérification ROUGE

Commande : `npm run test` (suite complète).

Résultat : **2 suites en échec, 21 suites passent (228 tests existants intacts)** — aucun test existant cassé.

Échecs attendus (modules métier non encore implémentés — pas d'erreur de syntaxe de test) :

```
FAIL src/renderer/features/theme/tests/themes.test.ts
  TS2307: Cannot find module '../utils/themes' or its corresponding type declarations.

FAIL src/renderer/features/theme/tests/theme-storage.test.ts
  TS2307: Cannot find module '../utils/theme-storage' or its corresponding type declarations.
```

Points de vigilance pour l'implémentation (étape 5) :
- Signatures tolérantes : `getThemeById(undefined)`, `getThemeById(null)`, `isValidThemeId(123)` et `saveTheme('purple')` doivent compiler → paramètres typés `unknown` (ou `string` pour `saveTheme`), validation interne.
- Immutabilité du registry : le test mute la liste exposée (`themes as Array<ThemeEntry>`) puis relit — la liste source ne doit pas être affectée → l'export `themes` doit être une copie à chaque accès (getter) ou équivalent, pas un simple `export const` mutable ni un tableau `Object.freeze` (la mutation lèverait une exception en strict mode et ferait échouer le test).
- Exceptions localStorage : `getStoredTheme`/`resolveInitialTheme`/`saveTheme` doivent catcher (`error: unknown` + `instanceof Error` si log nécessaire — ici pas de log attendu) et ne jamais propager.
- Clé : littéral `'beat-and-teach:theme'` (RG15-17), pas de normalisation des valeurs lues (casse sensible, espace final invalide).
- Labels et `swatchClass` : testés non vides (pas de valeur exacte imposée).

## Étape 4 — Architecture
STATUS: OK

### Analyse contradictoire

**Round 1 — Faisabilité CSS variables + `theme()` Tailwind**

Vérifié dans `tailwind.config.js` (13 tokens hex) et `App.css` :
- Les 13 tokens sont consommés par `@apply` (ex. `bg-primary`, `text-text-secondary`) **et** par ~25 usages directs de `theme('colors.primary')` dans App.css (keyframes glow ×11, btn-hover, gradient `.div-animated-wrapper::before`, `.section-toggle-arrow`, `.daw-resizer`, `.sidebar-item.selected`, `.sidebar-btn-save`, `.instrument-play-btn:hover`, `.transport-label`, `.transport-bpm-value`, accent-color checkboxes ×2, scrollbar thumb ×2) **et** par 5 hex en dur du slider BPM (ligne 471 `accent-color` + lignes 606-635 track/thumb webkit/moz).
- Migration en `rgb(var(--bt-*) / <alpha-value>)` : compatible Tailwind 3.4. Seul usage avec alpha détecté : `bg-background/70` dans `Modal.tsx` → génère `rgb(var(--bt-background) / 0.7)`. **Contrainte découverte : les variables doivent être des triplets RGB (`103 159 249`), pas des hex** — sinon `rgb(var(--bt-primary))` est invalide.
- Keyframes : les `var()` dans `text-shadow` des keyframes glow sont résolues au point d'utilisation → valides et réactives au changement de thème. Idem pour `accent-color`, scrollbar (pseudo-éléments globaux), `background` de gradient. Aucune incompatibilité bloquante.

**Round 2 — Le piège du SVG (contradiction détectée)**

`HomeLogoSvg.tsx` contient 7 hex uniques : `#030712` (rect fond), `#679ff9` (stroke waveform + rect + texte B ×2), `#1d273c` (rect), `#f3f4f6` (titre), `#6b7280` (tagline). Les variables étant des **triplets**, `fill="var(--bt-primary)"` produirait `fill: 103 159 249` → valeur invalide pour un attribut de présentation SVG.
- Option A : double jeu de variables (`--bt-primary` triplet + `--bt-primary-hex` hex) → duplication ×13.
- Option B : `fill="rgb(var(--bt-primary))"` / `stroke="rgb(var(--bt-primary))"` → les attributs de présentation SVG acceptent `rgb()` avec `var()` imbriquée, résolue au point d'utilisation (le SVG est dans le DOM React, hérite du `data-theme`).
- **Retenue : Option B** — zéro duplication, réactivité immédiate au switch. Le fond du rect (actuellement `#030712`) devient `rgb(var(--bt-background))`, cohérent avec le fond applicatif.

**Round 3 — Placement des variables : `<html>` vs `<div>`**

Le specs propose `data-theme` sur `<html>`. Vérifié : les scrollbars (`::-webkit-scrollbar` globaux), le fond `body` et tout élément hors du root React ne voient que les variables définies sur `:root`. Un `<div data-theme>` laisserait les scrollbars et le body en valeurs bleues. **Retenu : `data-theme` sur `document.documentElement`** (html), blocs CSS sur `:root` + `[data-theme='green']` + `[data-theme='pink']` dans App.css (seul fichier CSS du projet).

**Round 4 — FOUC au démarrage**

Le template est `src/renderer/index.ejs` (HtmlWebpackPlugin, configs `.erb/configs/webpack.config.renderer.{dev,prod}.ts`), CSP `script-src 'self' 'unsafe-inline'` → un script inline **est autorisé**.
- Option A : défaut `:root` seul → l'utilisateur green/pink voit un flash bleu entre le premier paint et l'effet React (le bundle charge vite en Electron, mais le flash existe, surtout en dev).
- Option B : script inline minimal dans `index.ejs` qui pose `data-theme` avant le bundle → zéro FOUC. Duplication volontaire et limitée : 1 clé + 2 ids (`'green'`, `'pink'`) en dur, avec try/catch. La logique complète (validation, fallback) reste dans `theme-storage.ts` (testée) ; le script ne fait qu'un miroir dégradé.
- **Retenue : Option B** — cohérent avec l'inspiration Linear/GitHub, coût quasi nul.

**Round 5 — Ordre des providers et état**

Vérifié dans `App.tsx` : `AudioProvider > PatternsProvider > InstrumentsProvider > GuideModalProvider > RouterProvider`. Le `ThemeSwitcher` vit dans `Header` (rendu dans le layout du routeur) → le `ThemeProvider` doit englober `RouterProvider`, donc être **le provider le plus externe**. Justification : préférence d'apparence globale, aucune dépendance aux données (instruments/patterns) ni à l'audio ; à l'inverse, tout le monde (Header, Modales, canvas éventuels) doit pouvoir lire le thème. Le state initial est lu dans l'initialiseur paresseux de `useState` (`resolveInitialTheme()`) — pas de dérivation ; l'application de `data-theme` est un `useEffect` de synchronisation DOM (légitime, cf. specs étape 1 ; la règle "pas de useEffect pour les dérivations calculables" ne s'applique pas à une synchro avec `document.documentElement`).

**Round 6 — Exigence des tests rouges : `themes` en getter-copie**

Le test `themes.test.ts` fait `themes.map/forEach/some`, `expect(themes).toHaveLength(3)` **et** mute la liste exposée (`exposed.push(...)`) puis vérifie que la source n'est pas affectée. Un simple `export const themes = [...]` échoue (même référence mutée) ; `Object.freeze` échoue aussi (push → TypeError en strict mode, signalé par l'étape 3). Il faut une copie fraîche **à chaque accès** :
- Option A : `Object.defineProperty(exports, 'themes', { get })` → ne compile pas en ESM (`exports` inexistant), incompatible `module: esnext` du typecheck.
- Option B : **Proxy getter-copie** — `themes` est un `Proxy` typé `ThemeEntry[]` sur le registry interne ; le trap `get` retourne, pour toute propriété fonction, une fonction qui s'exécute sur `[...registry]` (copie fraîche). `themes.length` lit le registry, `themes.push()` mute une copie jetable, `themes.map()` itère une copie → immutabilité garantie, typage transparent pour les consommateurs. **Retenue : Option B.**

**Round 7 — Périmètre et cohérence avec les tests**

- Les chemins cibles des tests (`../utils/themes`, `../utils/theme-storage`) **sont exactement** les fichiers cibles retenus → aucune divergence de tests à signaler.
- `waveform-renderer.ts` (recorder, canvas) a des hex par défaut (`#679ff9`/`#111827`) passés en options non fournies par `useWaveformEditor` : **hors périmètre V1** (les specs listent explicitement tokens Tailwind + App.css + HomeLogoSvg ; un canvas ne réagit pas au switch sans re-render). Documenté en cas limite — le canvas garde ses couleurs bleues en V1.
- `docs/index.html` (page de téléchargement statique, hors bundle) : hors périmètre, non modifié.
- `StepTooltip.tsx` (5 couleurs de highlight) : confirmé hors thème (sémantique pédagogique, tranché étape 2).
- Opacités existantes : `bg-background/70` (Modal) est le seul usage alpha → le format `rgb(var()/ <alpha-value>)` est requis mais n'a qu'un point de vérification.

### Décisions actées

| # | Décision | Raison |
|---|----------|--------|
| 1 | 13 variables CSS `--bt-*` en **triplets RGB** (`--bt-primary: 103 159 249`), définies sur `:root` dans App.css ; blocs `[data-theme='green']` et `[data-theme='pink']` ne surchargent que `--bt-primary`, `--bt-text-accent`, `--bt-button-edit` (RG18) | `rgb(var(--bt-*) / <alpha-value>)` exige des triplets ; les gris restent identiques |
| 2 | `data-theme` posé sur `document.documentElement` ; **jamais `'blue'`** (blue = absence d'attribut, valeurs `:root`) | Évite un bloc `[data-theme='blue']` dupliquant `:root` ; scrollbars/body couverts |
| 3 | `tailwind.config.js` : 13 tokens → `rgb(var(--bt-*) / <alpha-value>)` | Compatible avec l'unique usage alpha `bg-background/70` |
| 4 | App.css : les ~25 `theme('colors.primary')` → `var(--bt-primary)` (keyframes glow incluses) ; 5 hex slider BPM → `var(--bt-primary)` | Les keyframes résolvent les var() dynamiquement ; suppression de tous les hex applicatifs |
| 5 | `HomeLogoSvg.tsx` : les 7 hex → `rgb(var(--bt-*))` en attributs fill/stroke (jamais `var(--bt-*)` seul) | Un triplet nu est invalide en attribut SVG ; `rgb(var())` est résolu par le moteur CSS |
| 6 | **Script inline anti-FOUC dans `index.ejs`** (autorise `'unsafe-inline'` déjà en place) : lit `beat-and-teach:theme`, pose `data-theme` si `'green'`/`'pink'`, try/catch muet | Zéro flash bleu au démarrage ; duplication volontaire minimale (2 ids en dur) |
| 7 | `ThemeProvider` = **provider le plus externe** dans App.tsx (au-dessus d'AudioProvider) | ThemeSwitcher dans Header (layout du routeur) ; préférence globale sans dépendance aux données |
| 8 | State du contexte = `ThemeEntry` complet ; lecture initiale dans `useState(() => getThemeById(resolveInitialTheme()) ?? DEFAULT_THEME)` ; application `data-theme` dans un `useEffect` de synchronisation ; `setTheme(id)` = setState + `saveTheme(id)` | Lecture hors effet (pas de dérivation), synchro DOM légitime ; une seule source d'écriture CSS |
| 9 | `themes` exporté via **Proxy getter-copie** (chaque méthode de tableau s'exécute sur `[...registry]`) ; `getThemeById(id: unknown): ThemeEntry \| null` ; `isValidThemeId(id: unknown): boolean` ; `DEFAULT_THEME` exporté | Contrainte du test de mutation ; signatures tolérantes `unknown` ; pas de cast dans le contexte |
| 10 | `theme-storage.ts` : `getStoredTheme(): ThemeId \| null` (valide via `getThemeById(raw)?.id ?? null`), `saveTheme(theme: string): void` (no-op si invalide), `resolveInitialTheme(): ThemeId` (= `getStoredTheme() ?? 'blue'`), try/catch muet partout | GWT cas limites/erreurs ; `saveTheme` typé `string` (le test passe `'purple'`) |
| 11 | Slice `features/theme/` : `types/theme-types.ts` (ThemeId, ThemeEntry), `utils/themes.ts`, `utils/theme-storage.ts`, `contexts/ThemeContext.tsx` (provider + `useThemeContext`), `components/ThemeSwitcher.tsx`. **Pas de `hooks/useTheme.ts` ni de facade/service** (pas d'IPC, pas de DTO) | Pattern existant `InstrumentsContext.tsx` (provider + hook même fichier) ; les composants appellent le contexte directement (feature-sliced sans façade quand pas de couche IPC) |
| 12 | ThemeSwitcher : bouton `Palette` (lucide-react) en bas de `aside.bar-aside` (wrapper `mt-auto` dans Header), popover absolu **vers le haut** (`bottom-full`), 3 pastilles `swatchClass` (tailwind arbitraire `bg-[#679ff9]` etc.), pastille active marquée (ring), fermeture clic extérieur + Escape | Bas d'écran → le popover doit monter ; pattern Autocomplete (absolu) + Modal (Escape/clic extérieur) |
| 13 | Transition 150ms : règle CSS ciblée dans App.css sur une **liste explicite de classes de conteneurs/zones** (`.bar-aside`, `.content-page`, `.transport-bar`, `.daw-sidebar`, `.form-card`, `.item-row`, `.library-section`, `.modal-content`, `.sidebar-item`, `.instrument-card`, `.step-badge-valid`, etc.) avec `transition: background-color 150ms ease, color 150ms ease, border-color 150ms ease` — pas de `*` universel | Tranche GO étape 2 (classes ciblées) ; les éléments interactifs ont déjà `transition-colors` 100-200ms (dans la fourchette demandée) |
| 14 | Clé localStorage `beat-and-teach:theme` (littéral partagé entre utils et script FOUC — dupliqué volontairement dans le script inline) | Tranchée au GWT ; pattern onboarding existant (`*_tour_seen`) |

### Flux de données

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
  clic pastille (swatchClass)
    → useThemeContext().setTheme(themeId)
      ├─ setThemeState(getThemeById(themeId) ?? DEFAULT_THEME)   → re-render
      ├─ saveTheme(themeId)                                      → localStorage.setItem (try/catch)
      └─ useEffect → document.documentElement.dataset.theme = id (ou delete si blue)
        → CSS : [data-theme] surcharge --bt-primary/-text-accent/-button-edit
          → classes tailwind + var(--bt-*) + SVG recalculés     → transition 150ms ciblée
  fermeture popover (clic extérieur / Escape)
```

### Fichiers cibles (créés)

| Fichier | Rôle |
|---------|------|
| `src/renderer/features/theme/types/theme-types.ts` | `ThemeId = 'blue' \| 'green' \| 'pink'` ; `ThemeEntry = { id, label, accentHex, swatchClass }` (structurellement identique au type local des tests) |
| `src/renderer/features/theme/utils/themes.ts` | Registry pur : `themes` (Proxy getter-copie `ThemeEntry[]`), `getThemeById(id: unknown): ThemeEntry \| null`, `isValidThemeId(id: unknown): boolean`, `DEFAULT_THEME: ThemeEntry` (blue) |
| `src/renderer/features/theme/utils/theme-storage.ts` | Persistance : `getStoredTheme(): ThemeId \| null`, `saveTheme(theme: string): void`, `resolveInitialTheme(): ThemeId` — clé `'beat-and-teach:theme'`, try/catch muet (`error: unknown`) |
| `src/renderer/features/theme/contexts/ThemeContext.tsx` | `ThemeProvider` (state initial paresseux, useEffect synchro DOM, `setTheme`), exporte `useThemeContext()` (pattern InstrumentsContext) |
| `src/renderer/features/theme/components/ThemeSwitcher.tsx` | Bouton palette + popover 3 pastilles (swatchClass, ring actif), fermeture clic extérieur + Escape, aria-label |

Les fichiers cibles sont **strictement cohérents** avec les imports des tests rouges (étape 3) : aucun ajustement de test nécessaire.

### Fichiers modifiés

| Fichier | Modification |
|---------|--------------|
| `tailwind.config.js` | 13 tokens → `rgb(var(--bt-*) / <alpha-value>)` (mêmes noms de tokens) |
| `src/renderer/App.css` | Ajout `:root { --bt-*: <triplets> }` + `[data-theme='green']` / `[data-theme='pink']` (surcharge 3 variables d'accent) ; ~25 `theme('colors.primary')` → `var(--bt-primary)` ; slider BPM : `accent-color` (l.471) et track/thumb webkit/moz (l.606-635) → `var(--bt-primary)` ; règle transition 150ms ciblée (liste de classes) ; styles éventuels du popover swatches si non couverts par Tailwind |
| `src/renderer/components/Header.tsx` | Import + rendu `<ThemeSwitcher />` en bas de l'aside (wrapper `mt-auto` pour pousser en bas du flex-col `h-screen`) |
| `src/renderer/components/HomeLogoSvg.tsx` | 7 hex → `rgb(var(--bt-*))` : fond rect `--bt-background`, waveform/rect/B `--bt-primary`, rect `--bt-surface`, titre `--bt-text-primary`, tagline `--bt-text-secondary` |
| `src/renderer/App.tsx` | Wrap `<ThemeProvider>` le plus externe (au-dessus d'AudioProvider) |
| `src/renderer/index.ejs` | Script inline anti-FOUC (3 lignes, try/catch, pose `data-theme` pour green/pink) |
| `documentation-métier.md` (§7) | Proposé (3 thèmes + mécanisme CSS variables) — **accord utilisateur requis** (agent.md §14), hors V1 automatique |

### Cas limites

| Cas | Comportement |
|-----|--------------|
| Valeur localStorage absente / inconnue / vide / malformée | `getStoredTheme()` → `null` ; `resolveInitialTheme()` → `'blue'` sans erreur (RG17) ; script FOUC ne pose rien |
| localStorage indisponible (lecture ou écriture lève) | try/catch muet (`error: unknown`) → fallback blue, préférence simplement non persistée |
| `getThemeById` / `isValidThemeId` appelés avec `null` / `undefined` / nombre | Paramètre `unknown`, validation interne → `null` / `false`, jamais d'exception |
| Mutation de la liste `themes` par un consommateur | Proxy getter-copie : chaque méthode travaille sur `[...registry]`, la source n'est jamais affectée |
| `data-theme` en `'blue'` | Aucun attribut posé (ni script, ni provider) — blue = valeurs `:root`, pas de bloc CSS dédié |
| FOUC au démarrage | Script inline dans `index.ejs` pose `data-theme` avant le bundle ; `:root` bleu en secours |
| SVG avec triplet dans un attribut | Interdit `fill="var(--bt-primary)"` (triplet invalide) ; toujours `fill="rgb(var(--bt-primary))"` |
| Canvas waveform du recorder (`waveform-renderer.ts`) | **Hors périmètre V1** : garde ses défauts hex bleus ; amélioration possible (lecture `getComputedStyle` des variables + re-render) à évaluer en V2 |
| `docs/index.html` (page de téléchargement statique) | Hors périmètre : page standalone non bundleisée, non modifiée |
| Highlight sémantiques StepTooltip + rouges d'erreur | Fixes, hors thème (tranché étape 2) |
| SSR / absence de `document` | Toute lecture DOM est dans `useEffect` ou initialiseur (jamais au rendu d'un composant enfant) ; les utils n'utilisent que `localStorage` |

### Points de vigilance pour le dev (étape 5)

- **Proxy `themes`** : trap `get` → si `typeof value === 'function'`, retourner `(...args) => value.apply([...target], args)` ; ne pas oublier le narrowing avant `.apply` (pas d'implicit `any`).
- **Triplets RGB** partout dans les blocs CSS (jamais d'hex dans les variables utilisées par `rgb(var(--bt-*) / <alpha-value>)`).
- `saveTheme` typé `string` et valide via `isValidThemeId` (no-op si invalide, état antérieur conservé — testé).
- Vérifier au runtime l'unique usage alpha : `bg-background/70` (Modal) doit rendre `rgb(var(--bt-background) / 0.7)`.
- Le script `index.ejs` est hors lint (ESLint ne couvre pas `.ejs`) ; le garder volontairement minimal et sans commentaire.
- `useThemeContext` doit lancer si utilisé hors provider (pattern contexte strict) ou fournir un défaut — trancher côté dev en cohérence avec le reste du codebase (les contextes existants fournissent un défaut silencieux).
- Ne pas thématiser : rouge d'erreur, highlights, backdrop modale, `docs/index.html`.

## Étape 5 — Développement
STATUS: OK

### Implémentation

**Créés (slice `features/theme/`)**

| Fichier | Contenu |
|---------|---------|
| `src/renderer/features/theme/types/theme-types.ts` | `ThemeId = 'blue' \| 'green' \| 'pink'`, `ThemeEntry { id, label, accentHex, swatchClass }` |
| `src/renderer/features/theme/utils/themes.ts` | Registry : `themes` (Proxy getter-copie `ThemeEntry[]` — chaque méthode de tableau s'exécute sur `[...registry]`), `getThemeById(id: unknown)`, `isValidThemeId(id: unknown)`, `DEFAULT_THEME` (blue) |
| `src/renderer/features/theme/utils/theme-storage.ts` | `getStoredTheme(): ThemeId \| null`, `saveTheme(theme: string): void`, `resolveInitialTheme(): ThemeId` — clé `'beat-and-teach:theme'`, exceptions avalées, fallback blue |
| `src/renderer/features/theme/contexts/ThemeContext.tsx` | `ThemeProvider` + `useThemeContext()` (pattern InstrumentsContext) : state initial paresseux `useState(() => getThemeById(resolveInitialTheme()) ?? DEFAULT_THEME)`, `useEffect` de synchro DOM (pose/supprime `data-theme` sur `document.documentElement`, jamais `'blue'`), `setTheme` = setState + `saveTheme` |
| `src/renderer/features/theme/components/ThemeSwitcher.tsx` | Bouton `Palette` (lucide-react, classe `nav-item`) + popover `bottom-full` : 3 pastilles `swatchClass` avec ring sur l'active, fermeture clic extérieur (`mousedown`) + Escape, `aria-label` |

**Modifiés**

| Fichier | Modification |
|---------|--------------|
| `tailwind.config.js` | 13 tokens → `rgb(var(--bt-*) / <alpha-value>)` (mêmes noms) |
| `src/renderer/App.css` | Blocs `:root` (13 triplets RGB) + `[data-theme='green'\|'pink']` (surcharge `--bt-primary`, `--bt-text-accent`, `--bt-button-edit`) ; 37 usages `theme('colors.*')` → `rgb(var(--bt-*))` (keyframes glow incluses) ; 5 hex slider BPM → `rgb(var(--bt-primary))` ; règle transition 150ms ciblée (liste de 11 classes : `.bar-aside`, `.content-page`, `.transport-bar`, `.daw-sidebar`, `.form-card`, `.item-row`, `.library-section`, `.modal-content`, `.sidebar-item`, `.instrument-card`, `.step-badge-valid`) |
| `src/renderer/components/Header.tsx` | `<ThemeSwitcher />` dans un wrapper `mt-auto` en bas de l'aside |
| `src/renderer/components/HomeLogoSvg.tsx` | 7 hex → `rgb(var(--bt-*))` (fond `--bt-background`, accent `--bt-primary` ×4, `--bt-surface`, `--bt-text-primary`, `--bt-text-secondary`) |
| `src/renderer/App.tsx` | `<ThemeProvider>` le plus externe (au-dessus d'AudioProvider) |
| `src/renderer/index.ejs` | Script inline anti-FOUC dans `<head>` : lit `beat-and-teach:theme`, pose `data-theme` si `'green'`/`'pink'`, `try/catch` muet |

### Divergences justifiées (vs architecture étape 4)

1. **Décision 4 ajustée — `rgb(var(--bt-*))` dans App.css (au lieu de `var(--bt-*)` nu)** : les variables étant des triplets RGB (décision 1), un `var(--bt-primary)` nu produit une valeur invalide dans `text-shadow`/`accent-color`/`background`/`color`/`border-color` (le triplet n'est une couleur que dans une fonction `rgb()`). Même raison que la décision 5 (SVG) — application homogène. Les keyframes glow restent résolues dynamiquement au point d'utilisation (Round 1 inchangé). Vérifié : `rgb(var(--bt-primary))` compile dans le CSS généré.
2. **Catch muet de `saveTheme` réécrit** : les règles lint actives (`no-empty`, `no-useless-return`, `no-void`, `@typescript-eslint/no-unused-vars` avec `caughtErrors` actif) interdisent `catch {}`, `catch { return; }`, `catch (error) { return; }` et `void error`. Solution : fonction d'aide privée `writeStoredTheme(theme): boolean` (`catch (error: unknown)` + `return false` dans les deux branches — `return false` n'est jamais "useless" pour un retour non-void). Comportement identique au GWT : exception avalée, aucune propagation, `saveTheme` reste `void`.
3. **Reformatage prettier de `themes.test.ts` (étape 3)** : la ligne 47 (`.some(...)`) ne respectait pas prettier → erreur lint bloquante. Reformattée par `prettier --write` — **aucune modification sémantique**, aucun test affaibli.
4. **Écart de comptage des cas** : l'étape 3 annonçait 36 cas (18+18) ; les suites en contiennent réellement 32 `it` (16+16). Sans impact — les 32 passent.

### Vérifications (ordre du pipeline)

| Vérification | Résultat |
|--------------|----------|
| `npm run test` | **VERT** — 23 suites, 260 tests (228 existants + 32 nouveaux) |
| `npm run typecheck` | **0 erreur** (tsconfig.renderer + tsconfig.main) |
| `npm run lint` | **0 erreur** (5 warnings préexistants hors périmètre : `audio-engine.ts` ×4 no-console, `audio-engine.test.ts` func-names) |

### Points de vigilance pour la review

- `themes` est un Proxy getter-copie : le trap `get` retourne un wrapper pour toute propriété fonction, exécuté sur `[...target]` (copie fraîche) — immutabilité garantie, typage `ThemeEntry[]` transparent (vérifié par les 16 tests du registry).
- La règle de transition 150ms est hors `@layer` → cascade gagnante sur `transition-colors duration-100` de `.sidebar-item` (100ms → 150ms, dans la fourchette 150-200ms actée) : voulu.
- Vérification visuelle runtime (contraste, FOUC, popover, smoothness) non couverte par les tests unitaires — à valider à la clôture (lancement `npm start`).

### Round 2 — Correction popover (retour utilisateur, clôture)

- **Problème signalé** : popover coupée à gauche (hors écran), seul le cercle du thème actuel visible, pastilles trop grosses.
- **Cause** : centrage `left-1/2 -translate-x-1/2` sur un bouton de 40px centré dans une sidebar de 64px → bord gauche de la popup (~116px) à -26px hors écran ; le seul cercle visible était le premier (bleu, actuel).
- **Correctif** (`ThemeSwitcher.tsx` uniquement) : popover repositionné à droite de la sidebar (`left-full ml-2 top-1/2 -translate-y-1/2`, pattern Discord/Linear) ; pastilles `w-7 h-7` → `w-4 h-4` ; ring actif `ring-2 ring-offset-2` → `ring-1 ring-offset-1` ; `gap-2`/`p-2` → `gap-1.5`/`p-1.5`. Les 3 cercles sont désormais visibles.
- **Vérifications** : `npm run test` VERT (260), `npm run typecheck` 0 erreur, `npm run lint` 0 erreur. Validation visuelle à confirmer par l'utilisateur au runtime.

### Round 3 — Correction pastilles invisibles (retour utilisateur, clôture)

- **Problème signalé** : seul le cercle du thème actuel visible, les autres en gris (invisibles).
- **Cause racine** : `swatchClass` du registry contient des classes Tailwind arbitraires (`bg-[#679ff9]` etc.) définies dans `utils/themes.ts` (fichier `.ts`) — or `tailwind.config.js` ne scanne que `./src/**/*.tsx` → classes jamais générées dans le CSS. Pastilles non-actives sans fond (transparent sur surface) ; seule l'active était visible via son `ring-primary` (classe issue de la config, non du registry). Le bug datait du Round 1, masqué par la popup coupée.
- **Correctif — source de vérité unique** : suppression de `swatchClass` de `ThemeEntry` (types, registry, tests) ; `ThemeSwitcher.tsx` applique `style={{ backgroundColor: entry.accentHex }}` (l'`accentHex` du registry, déjà testé, devient la seule source de la couleur).
- **Vérifications** : `npm run test` VERT (260), `npm run typecheck` 0 erreur, `npm run lint` 0 erreur. Validation visuelle à confirmer par l'utilisateur au runtime.
- `bg-background/70` (Modal) compile bien en `rgb(var(--bt-background) / 0.7)` — vérifié par compilation Tailwind isolée.
- Le script `index.ejs` duplique volontairement la clé et les 2 ids (décision 6) ; hors lint (`.ejs` non couvert).

## Étape 6 — Review
STATUS: OK

### Résultat

- Point 1 — Conformité aux specs : ✅ 3 thèmes (`blue`/`green`/`pink`, accents `#679ff9`/`#4ade80`/`#f472b6`), 12 variables CSS triplets RGB sur `:root` + surcharge `[data-theme='green'|'pink']` de 3 variables d'accent (RG18), `data-theme` sur `<html>` via `document.documentElement`, persistance localStorage `beat-and-teach:theme` (RG15-17), switcher palette en bas de la sidebar (wrapper `mt-auto` dans `aside.bar-aside` qui est `flex flex-col h-screen`), transition 150ms ciblée (11 classes, hors `@layer` → cascade gagnante), script anti-FOUC dans `index.ejs`. Exclus V1 respectés : rouge d'erreur, highlights StepTooltip, backdrop modale (`rgba(0,0,0,0.6)` l.579, seul rgba restant), `waveform-renderer.ts`, `docs/index.html` — tous non modifiés.
- Point 2 — Conformité aux GWT : ✅ 32 `it` (16+16) couvrent **tous** les cas GWT (nominaux, limites, erreurs — certains regroupés dans un même `it` : casse, espace final, `null`/`undefined`/`123`, valeurs malformées, exceptions read/write via spies). Les 3 helpers + registry sont testés : `getThemeById`/`isValidThemeId` tolérants `unknown`, `saveTheme('purple')` no-op, immutabilité du registry vérifiée par mutation d'une copie Proxy.
- Point 3 — Conformité à l'architecture : ✅ Décisions 1-14 respectées. Vérifications ciblées : Proxy getter-copie fonctionnel (trap `get` narrow `typeof value === 'function'` puis `apply([...target])` — pas d'implicit `any`) ; `data-theme` jamais `'blue'` (script FOUC ne pose que green/pink, provider fait `delete dataset.theme`) ; ThemeProvider le plus externe dans `App.tsx` (au-dessus d'AudioProvider) ; state initial paresseux `useState(() => getThemeById(resolveInitialTheme()) ?? DEFAULT_THEME)` ; `setTheme` = setState + `saveTheme` ; slice sans facade/service (pas d'IPC, pattern InstrumentsContext, décision 11) ; popover `bottom-full` avec fermeture mousedown extérieur + Escape ; `key={entry.id}` (ID stable) ; transition 150ms sur la liste exacte de 11 classes.
- Point 4 — Conventions (`agent.md`) : ✅ pas de `for...of`, pas de commentaires dans le slice, pas d'implicit `any` (rg vérifié), pas d'eslint-disable, `error: unknown` + narrowing dans les catch, immutabilité (copies fraîches), `useEffect` uniquement pour synchro DOM/listeners (légitime), `useMemo` pour la valeur du contexte. Composants n'appelant que le contexte (pas de service direct).
- Point 5 — Règles de gestion : ✅ RG15 (défaut blue), RG16 (persistance), RG17 (fallback silencieux), RG18 (seules 3 variables d'accent changent) — toutes couvertes par les tests. `bg-background/70` (Modal) compile bien en `rgb(var(--bt-background) / 0.7)` (seul usage alpha).
- Point 6 — Vérifications techniques : ✅ `npm run test` → 23 suites / 260 tests VERT ; `npm run typecheck` → 0 erreur ; `npm run lint` → 0 erreur, 5 warnings préexistants hors périmètre (`audio-engine.ts` ×4, `audio-engine.test.ts` ×1 — identiques à l'étape 5).
- Point 7 — Diff scope : ✅ exactement les 6 fichiers modifiés actés (App.css, App.tsx, Header.tsx, HomeLogoSvg.tsx, index.ejs, tailwind.config.js) + le slice `features/theme/` (7 fichiers). `documentation-métier.md` non modifié (accord requis, conforme agent.md §14). Aucun fichier hors périmètre.

### Divergences déclarées (étape 5) — vérifiées, justifiées, sans impact

1. `rgb(var(--bt-*))` au lieu de `var(--bt-*)` nu dans App.css : ✅ justifié — les variables sont des triplets (décision 1), un `var()` nu est invalide dans `text-shadow`/`accent-color`/`background`/`color`/`border-color`. 42 usages `var(--bt-*)`, aucun hex applicatif restant (seul `rgba(0,0,0,0.6)` = backdrop, hors périmètre).
2. `writeStoredTheme(theme): boolean` avec double `return false` : ✅ justifié — contourne `no-empty`/`no-useless-return`/`no-unused-vars`(caughtErrors) tout en gardant le comportement GWT (exception avalée, `saveTheme` reste `void`, no-op sur invalide). Vérifié au lint : 0 erreur.
3. Reformatage prettier de `themes.test.ts` : ✅ purement cosmétique (`.some(...)` multi-lignes), aucun cas modifié ni affaibli — les 16 tests passent.
4. 32 `it` au lieu de 36 annoncés : ✅ comptage réel vérifié (16+16), tous les cas GWT couverts, aucun cas manquant.

### Corrections requises

Aucune. (Nitpick documentaire sans impact : la décision 1 annonce "13 variables" mais le registry Tailwind n'en contient que 12 — cohérence parfaite entre `tailwind.config.js` et `:root`, uniquement une erreur de comptage dans le doc, non bloquante.)

### Points de vigilance pour la clôture (non bloquants, validation visuelle)

- Vérification runtime `npm start` : FOUC au démarrage (green/pink), contraste des 3 thèmes, smoothness de la transition 150ms, positionnement du popover en bas de sidebar, ring de la pastille active.

## Avancement
- [x] 1. Specs rédigées
- [x] 2. GWT rédigés
- [x] 3. Tests rouges
- [x] 4. Architecture validée
- [x] 5. Développement (vert)
- [x] 6. Review OK
- [ ] 7. Clôture (archivée)

## Notes / décisions
- Décisions utilisateur (avant specs) : approche popover swatches dans la sidebar ; accents vert #4ade80 et rose #f472b6 ; transition douce 150-200ms.
- Les 5 couleurs de highlight (StepTooltip) sont sémantiques et restent fixes, hors thème.
- Persistance en localStorage (pattern onboarding existant).
- Tranches GO étape 2 : **logo** — tout le SVG de `HomeLogoSvg.tsx` est thématisé (accent = couleur du thème, les 7 hex deviennent des variables) ; **transition** — classes ciblées sur les zones/éléments affectés, pas de transition universelle `*` ; **clé localStorage** — `beat-and-teach:theme` ; **contraste** — vérification visuelle pure en fin de feature (clôture), les gris restant identiques.
- Helpers de persistance envisagés dans le GWT : `getStoredTheme` / `saveTheme` (utils purs, testables en jsdom).

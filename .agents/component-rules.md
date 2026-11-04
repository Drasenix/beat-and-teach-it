# Conventions composants — Beat & Teach

> Spécialisation projet du pack `electron-react-ts`.

## Structure

- Un composant par fichier, `PascalCase.tsx`, export par défaut.
- Props typées explicitement (`interface XxxProps`).
- Logique dérivée → `useMemo` (jamais `useEffect` pour un calcul dérivable).

## Règles React

- **Pas d'index de tableau comme `key`** : utiliser un ID stable et unique.
- État global réservé aux données partagées (listes patterns/instruments, session audio) ; le reste en `useState` local.
- Providers dans l'ordre : `ThemeProvider > AudioProvider > PatternsProvider > InstrumentsProvider > GuideModalProvider > RouterProvider`.
- Le thème est appliqué par `ThemeProvider` (`data-theme` sur `<html>`) : ne jamais manipuler `data-theme` depuis un composant ; passer par `useThemeContext()`.
- Pas de logique métier dans les composants : déléguer aux facades / hooks.

## Hooks

- Hooks partagés dans `hooks/` (ex. `useResizable`, `useSlider`, `useCaretPosition`).
- Un hook = une responsabilité ; nommage `useXxx`.
- Hooks de domaine dans `features/<domaine>/hooks/`.

## Accessibilité

- Éléments interactifs = vrais boutons/liens ; gérer le focus clavier.
- Raccourcis clavier documentés au niveau du composant concerné (cf. `.agents/specs/spec.txt` §5).

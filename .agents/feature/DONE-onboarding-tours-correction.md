# Feature: Correction des tours guidés (onboarding)

> Clôturé le 2026-06-30 · version de travail : `git show 66eeb90:.agents/feature/DONE-onboarding-tours-correction.md`

## Résumé métier

Correction et consolidation des visites guidées driver.js : suppression des bugs bloquants du tour Studio (`onNextClick`, sélecteurs vides, courses entre `localStorage` et `setTimeout`), factorisation de la création des tours (`createTour()`), réduction du tour Studio de 16 à 9 étapes, réécriture des textes, suppression du tour Patterns et enrichissement du Guide (édition des steps, raccourcis, icônes Lucide).

## Scénarios évolution — nouveau comportement

> Reconstitués depuis la liste des corrections (fichier antérieur au gabarit).

1. Étant donné le tour Studio, quand je clique sur « Suivant », alors le tour avance d'une étape sans sauter d'étape (l'état courant est lu avant le clic).
2. Étant donné un élément pas encore monté dans le DOM, quand le tour attend, alors `waitForElement()` attend son apparition au lieu d'un `setTimeout` fixe.
3. Étant donné le Guide, quand j'ouvre la liste des tours, alors le tour Studio compte 9 étapes et le tour Patterns n'existe plus.
4. Étant donné les raccourcis du Guide, quand je lis la section « Outils », alors le tooltip est documenté.

## Dette connue

| #    | Edge case / sujet                                              | Repris par                  | Justification                                  |
| ---- | -------------------------------------------------------------- | --------------------------- | ---------------------------------------------- |
| EC-1 | Attributs `data-tour` non ajoutés sur les éléments ciblés      | Au fil des corrections (P1) | Non prioritaire, sélecteurs actuels suffisants |
| EC-2 | Tooltip forcé du tour (`data-tour-force-tooltip`) non supporté | —                           | Approche annulée (hors overlay driver.js)      |

Couverts : aucun test automatisé (frontend, hors TDD).

## Décisions & Justifications — ADR

| #   | Décision                                                 | Raison                                                         |
| --- | -------------------------------------------------------- | -------------------------------------------------------------- |
| 1   | Suppression du tour Patterns (Option A)                  | Moins de bruit pédagogique ; le tour Studio couvre l'essentiel |
| 2   | `data-tour-force-tooltip` abandonné                      | Le tooltip du tour est hors de l'overlay driver.js             |
| 3   | Couleurs de highlight citées en exemple, pas comme règle | Éviter de transformer une illustration en norme                |
| 4   | Normalisation des pistes retirée de la documentation     | Information obsolète                                           |
| 5   | `eslint-disable` conservé dans `GuideModalProvider.tsx`  | Toujours utile ; le retirer casserait le lint                  |

## Leçons de réalisation

- `onNextClick` sautait des étapes → lecture de l'état courant via `setSectionState()` avant le clic → pour un tour, toujours lire l'état réel avant de simuler une action.
- Courses entre `localStorage` et `setTimeout(500)` → remplacement par `waitForElement()` → ne pas attendre un délai fixe pour un élément dont l'apparition est observable.
- Duplication de la création des tours → factory `createTour()` dans `onboarding/utils/createTour.ts` → une seule fabrique pour tous les tours.

## Vigilances

- Les sélecteurs des tours doivent tolérer un DOM non prêt (`waitForElement`) : ne pas revenir à un `setTimeout` fixe.
- Ne pas réintroduire le tour Patterns ni les émojis dans les popups : les textes et les icônes Lucide sont le rendu validé.

## Traçabilité

- **Spec et US** : aucune section de spec officielle ; parcours Guide / onboarding (feature hors spec).
- **Spec parallèle** : aucun.
- **Conventions établies** : `.agents/architecture.md` (`onboarding/`, `guide/`) · `.agents/conventions.md` (composants, hooks).
- **Utilitaires référencés** : `createTour.ts` (factory des tours).
- **Tests** : aucun (frontend).

> Écarté à la compaction (texte complet : version de travail) : Specs (→ Résumé métier, Scénarios, ADR, Dette connue) · Avancement (pilotage du développement) · Notes / décisions (→ ADR, Dette connue)

# Feature: dynamic-pattern

> Clôturé le 2026-06-16 · version de travail : `git show 66eeb90:.agents/feature/DONE-dynamic-pattern.md`

## Résumé métier

Le pattern s'entend pendant qu'on l'édite : ajouter ou retirer un symbole, ajouter ou retirer une phrase, ou muter un pas modifie la lecture en cours sans Stop + Play. La grille, les textarea et le mute restent réactifs en toutes circonstances ; seule la poussée vers le moteur audio est conditionnée à la validité — un symbole inconnu (même à l'intérieur d'un groupe) gèle la mise à jour sonore, le symbole fautif reste visible et éditable, et dès qu'il est corrigé ou supprimé, tous les changements en attente (y compris ceux qui étaient déjà valides) sont appliqués d'un coup. Un instrument entendu pour la première fois voit son fichier audio chargé avant d'être joué.

Le moteur audio ne pré-programme plus les notes à l'avance : une seule boucle maître lit le pattern courant à chaque pas `'8n'`, ce qui rend l'édition live possible sans décalage et sans saut de son quand la longueur du pattern change. L'animation de la grille et le son sont servis par cette même boucle, et les groupes `(A B C)` rejouent leur succession rapide à l'intérieur d'un pas, comme avant le refacto.

Le découpage des phrases (regex « groupe | token ») est extrait dans un tokenizer partagé, qui remplace trois copies et sert désormais au parsing de la grille, au mute, à la préparation des notes et à la validation.

## Scénarios évolution — nouveau comportement

Périmètre testé : logique pure et moteur audio (`sentence-tokenizer`, `pattern-validator`, `instrument-engine`, `audio-engine`, `audio-facade`). Les composants React (`PatternWorkspace`, `AudioContext`) ne sont pas couverts.

#### Édition live pendant la lecture

- Given: la séquence est en cours de lecture
  When: j'ajoute un symbole valide (instrument existant) dans le pattern
  Then: la séquence est mise à jour dynamiquement
  And: l'instrument est joué sans Stop + Play
- Given: la séquence est en cours de lecture
  When: j'ajoute un symbole non valide (aucun instrument dans la base)
  Then: la séquence n'est pas mise à jour
  And: l'édition de l'interface reste réactive
- Given: la séquence est en cours de lecture
  When: j'ajoute un symbole non valide puis un symbole valide
  Then: aucune mise à jour n'a lieu
  And: quand je corrige ou supprime le symbole non valide, la séquence est mise à jour et inclut tous les changements valides restés en attente
- Given: la séquence est en cours de lecture
  When: je mute ou démute un pas
  Then: la séquence est mise à jour dynamiquement (le silence `.` est toujours valide)
- Given: un symbole valide jamais joué apparaît dans le pattern pendant la lecture
  Then: son fichier audio est chargé (IPC + decode) avant d'être joué

#### Synchronisation de l'animation avec le son

- Given: un pattern est en cours de lecture
  Then: chaque symbole de la grille est animé en même temps que l'instrument est lu
- Given: un pattern est en cours de lecture et j'ajoute un instrument valide qui change le nombre de pas
  Then: les animations sont recalées et restent synchronisées avec le séquenceur

#### Pas de saut de son quand la longueur change

- Given: un pattern de 4 colonnes est lu, j'ajoute un symbole valide (5 colonnes)
  Then: le son en cours ne saute pas (pas de changement brutal d'instrument au même tick)
- Given: un pattern de 5 colonnes est lu, je supprime un symbole (4 colonnes)
  Then: le son en cours ne saute pas
- Given: la lecture est en cours, j'ajoute ou supprime une piste sans changer le nombre de colonnes
  Then: le son continue sans saut et les nouvelles pistes sont audibles immédiatement

#### Subdivision des groupes

- Given: un pattern contient `(A B C)` et il est lu
  Then: A, B, C sont joués successivement dans la même pulsation, et non simultanément
- Given: un pattern contient `(A B)` et il est lu
  Then: A est joué au début du pas, B à mi-pas (`'8n' / 2`)

## Dette connue

| #    | Edge case / sujet                                                                                                                                                                    | Repris par | Justification                                                                       |
| ---- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------- | ----------------------------------------------------------------------------------- |
| EC-1 | Recalibration de `stepIndex` : la position relative est préservée, le compteur absolu est perdu — le rythme repart de la position recalée                                            | —          | Compromis assumé pour supprimer le saut ; le décalage résiduel est inaudible        |
| EC-2 | `updateTrack` reçu avant la fin de `playPattern` : `updateSequences` sort tôt (`!masterLoop`), la modification est perdue jusqu'à la prochaine édition valide                        | —          | Guard de course ; pas de rejeu différé                                              |
| EC-3 | `loadedSymbols` n'est jamais purgé alors que `createPlayers` reconstruit `Tone.Players` (nouveau Play, aperçu d'instrument) : un symbole peut être cru chargé sans buffer disponible | —          | Écart au plan de la phase 4 (reset prévu, non implémenté) ; non couvert par un test |
| EC-4 | Échec de `decodeAudioData` ou de `getInstrumentFilePathsFromSymbol` : erreur avalée par `updateTrack`, aucun signal utilisateur, l'instrument reste muet                             | —          | Choix « l'audio ne bloque jamais l'édition » (ADR-6)                                |
| EC-5 | Pistes de longueurs différentes : la longueur de référence est celle de la première phrase, les autres sont tronquées ou silencieuses au-delà (skip, pas de bouclage)                | —          | Masqué par la normalisation des phrases sur la première                             |
| EC-6 | Gate de validation et branchement `updateTrack` non testés (composants React hors TDD)                                                                                               | —          | Vérifiés manuellement à la livraison                                                |
| EC-7 | Pas de debounce : chaque frappe valide déclenche un chargement de buffers + une reconstruction des notes                                                                             | —          | Assumé : Tone.js reschedule sans glitch ; patterns courts                           |

Couverts : tokenizer (3 fonctions), `areAllSymbolsValid` (groupes, silences, phrases vides, multi-phrases), `getAllSymbols`, `hasSymbol` / `registerSymbol`, `addToPlayers`, `updateSequences` (guard, recalibration, pas de recréation), callback du master loop (notes, groupes et offsets, silences, pistes courtes), `updatePattern` (rien à charger, nouveau symbole, silence, enregistrement).

## Arbre de flux

```
PatternWorkspace
  │ (inchangé) usePatternSession() → sentencesForPlayback
  │ (inchangé) useAudio() → { playing, updateTrack, ... }
  │ (NOUVEAU) useMemo → allValid = areAllSymbolsValid(sentencesForPlayback, validSymbols)
  │ (NOUVEAU) useEffect → if playing && allValid → updateTrack(sentencesForPlayback)
  ▼
AudioContext.updateTrack(sentences)
  ▼
audio-facade.updatePattern(sentences)
  ├── extractUniqueSymbols(sentences) → string[] (via sentence-tokenizer)
  ├── Filtrer les symboles NON déjà chargés dans AudioEngine
  ├── Si nouveaux → IPC get-audio-buffers → audioEngine.addToPlayers()
  ├── preparePattern(sentence) → SequenceNotes[] (via sentence-tokenizer)
  └── audioEngine.updateSequences(allNotes)
        ├── Guard: si this.sequences.length === 0 → return (race condition)
        ├── Ajout/suppression de Tone.Sequence si nombre de pistes changé
        ├── seq.events = newNotes pour chaque Tone.Sequence existant
        └── Recrée Tone.Loop (step) si columnCount a changé
```

## Arbre de flux — lecture temps réel (boucle maître)

```
Play
  prepareAudioEngine → setTempo (stepDuration = Tone.Time('8n').toSeconds())
    ├── createPlayers (Tone.Players reconstruits depuis les buffers)
    ├── createSequence(trackNotes) → 1 seul Tone.Loop maître ('8n')
    └── registerSymbol(s) pour chaque symbole du pattern

Chaque tick '8n' (masterLoop callback)
  ├── currentColumnCount <= 0 → return (première phrase vidée)
  ├── step = stepIndex % currentColumnCount
  ├── pour chaque piste :
  │     ├── step >= trackNotes.length → skip (piste plus courte)
  │     ├── note string   → players.player(note).start(time)
  │     ├── note string[] → sous-note i à time + i * stepDuration / subCount
  │     └── note null     → silence
  ├── onStep(step) → animation de la grille
  └── stepIndex += 1

Édition live
  sentencesForPlayback change
    ├─ areAllSymbolsValid = false → rien n'est poussé (l'UI continue)
    └─ true → updateTrack → updatePattern
                ├── nouveaux symboles : IPC buffers → addToPlayers → registerSymbol
                └── updateSequences : trackNotes = tracks
                      └── columnCount changé → stepIndex = stepIndex % ancienColumnCount (pas de recréation de boucle)
```

## Décisions & Justifications — ADR

| #   | Décision                                                                                                                                                                                                                                                                  | Raison                                                                                                                                                                                                           |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Regex de découpage des phrases et boucle `exec` extraits dans `src/renderer/utils/sentence-tokenizer.ts` (`tokenizeSentence`, `extractSymbols`, `extractUniqueSymbols`) ; `pattern-parser`, `pattern-mute` et `sequence-service` l'importent au lieu de recopier le regex | Trois copies existaient déjà, la validation en aurait ajouté une 4e ; placé dans `renderer/utils/` car partagé par plusieurs features                                                                            |
| 2   | `areAllSymbolsValid()` ajoutée à `pattern-validator.ts` (pas de nouveau fichier) ; implémentation O(n) par `replace(/[()]/g, '')` + `split` ; `.` toujours valide et ensemble vide valide                                                                                 | Même domaine que `validatePattern()` ; les captures de groupes sont inutiles ici                                                                                                                                 |
| 3   | Gate dans `PatternWorkspace` : `useMemo` de validité + `useEffect([playing, allValid, sentencesForPlayback, updateTrack])` qui ne pousse que si `playing && allValid`                                                                                                     | L'état React porte la dernière version des phrases : la retenue derrière un invalide est implicite, aucun tampon « en attente » à maintenir                                                                      |
| 4   | `updatePattern()` charge les buffers des nouveaux symboles **avant** de mettre les séquences à jour ; `AudioEngine` suit les symboles chargés (`hasSymbol` / `registerSymbol`), le mapping symbole → nom d'instrument reste dans `InstrumentEngine`                       | Pas de trou entre l'édition et la disponibilité du son ; les `Tone.Players` ne connaissent que des noms d'instruments                                                                                            |
| 5   | Les symboles du pattern sont enregistrés au Play initial dans la facade, après `createSequence`, et non dans `createPlayers`                                                                                                                                              | `createPlayers` ne reçoit que des noms d'instruments ; seule la facade dispose du mapping symbole → fichier                                                                                                      |
| 6   | `AudioContext.updateTrack` attrape l'erreur sans la propager (catch muet)                                                                                                                                                                                                 | L'édition ne doit jamais être bloquée par l'audio ; la prochaine édition valide reprend l'état courant                                                                                                           |
| 7   | Un seul `Tone.Loop` maître remplace N `Tone.Sequence` + 1 `Tone.Loop` d'animation ; le callback lit `trackNotes` et `currentColumnCount` en direct (pull / réactif)                                                                                                       | `Sequence.events =` reprogramme chaque note à un temps absolu : à tick constant, changer le modulo change la note jouée (saut décrit en Leçons) ; update O(1), boucle unique = audio et animation indissociables |
| 8   | `updateSequences` se limite à affecter `trackNotes` et, si la longueur change, à recaler `stepIndex = stepIndex % ancienColumnCount` ; guard `!masterLoop`                                                                                                                | Ne jamais recréer la boucle (la recréation remet le compteur à 0 et désynchronise) ; la position relative est préservée                                                                                          |
| 9   | La boucle maître n'est créée que si `onStep` est posé                                                                                                                                                                                                                     | Elle sert à la fois l'audio et l'animation ; sans callback d'interface, aucune boucle n'est nécessaire                                                                                                           |
| 10  | Groupes : subdivision manuelle `offset = i * stepDuration / subCount`, avec `stepDuration = Tone.Time('8n').toSeconds()` calculé dans `setTempo()`                                                                                                                        | `Tone.Sequence` faisait cette subdivision nativement ; la boucle maître doit la reproduire, et elle dépend du BPM courant                                                                                        |
| 11  | `stepIndex` et `currentColumnCount` vivent sur l'instance ; la closure de la boucle ne capture que `this`                                                                                                                                                                 | Une valeur capturée à la création ne peut plus être mise à jour ; un champ d'instance est relu à chaque tick                                                                                                     |
| 12  | Tests du moteur : mock `tone` inline qui capture le callback du `Loop` (`cb`) et expose `Time`                                                                                                                                                                            | Permet de piloter les ticks et d'asserter les offsets temporels sans Transport réel                                                                                                                              |

## Leçons de réalisation

- **Animation désynchronisée après une édition live** : à chaque changement du nombre de pas, le `Tone.Loop` d'animation était disposé puis recréé — `stepIndex` repartait à 0 alors que le Transport continuait, et la closure gardait le `columnCount` capturé à la création ; le décalage devenait permanent. Correction : lire `this.currentColumnCount` au tick, guard `<= 0` contre le `NaN` (première phrase vidée), ne plus recréer la boucle. → Une boucle ne capture jamais une donnée mutable ; elle lit l'état d'instance à chaque tick.
- **Saut de son au changement de longueur** : `Sequence.events =` reprogramme toutes les notes à des temps absolus ; au même tick Transport, `11 % 4` puis `11 % 5` désignent deux notes différentes (`. → Ts`) — saut arrière à l'ajout, avant à la suppression. La correction de l'animation seule (leçon précédente) avait masqué le problème : c'est le modèle de programmation qui était en cause. Correction : boucle maître qui lit les données en direct + recalibration de la position. → Pré-scheduling et édition live sont incompatibles ; on ne répare pas un reschedule, on change de modèle.
- **Subdivision des groupes perdue** : après le passage à la boucle maître, toutes les sous-notes d'un groupe démarraient au même `time` (l'ancien test de groupe ne vérifiait que les appels à `player()`, jamais les temps, donc il restait vert). Correction : `stepDuration` calculé dans `setTempo()` + offset `i * stepDuration / subCount`, et assertions `expect.closeTo` sur les appels `start`. → Quand on remplace un mécanisme natif de Tone.js, en reproduire explicitement les comportements implicites ; un test audio asserte le temps, pas seulement l'instrument.
- **Mocks Tone** : les tests ont dû être enrichis au fil des corrections (callback du `Loop`, puis `Tone.Time`) pour observer le comportement temporel. → Un mock qui n'expose pas le callback ne peut pas tester une horloge.

## Vigilances

- `.agents/specs/spec.txt` §6.1 décrit encore l'ancien `Tone.Sequence` par piste : `SP-AUDIO-01` prime, et `.agents/conventions.md` § Audio porte l'état courant du moteur (un `Tone.Loop` par piste).
- `stepDuration` n'est recalculé que par `setTempo()` : tout chemin de changement de tempo qui ne passe pas par la facade laisse les offsets de groupes calés sur l'ancien BPM.
- Le verrou anti-symbole-inconnu vit uniquement dans `PatternWorkspace` et n'a pas de test : le contourner enverrait des symboles inconnus au moteur, où l'erreur est avalée par `updateTrack` (panne silencieuse, grille animée mais pas de son).
- Le registre `loadedSymbols` de `AudioEngine` est indépendant des `Tone.Players` réellement en place : toute nouvelle reconstruction des Players doit le purger (cf. EC-3), sinon un symbole est cru chargé sans buffer.

## Traçabilité

- **Spec et US** : §6.1 (flux de composition Studio → Audio), §6.4 (chargement audio), §2.3–§2.4 (phrases et groupes), §8 (animations) ; pas de lot ni d'US numéroté.
- **Spec parallèle** : `SP-AUDIO-01` (AMENDE §6.1).
- **Conventions établies** : `.agents/conventions.md` (audio, pattern, sequence ; la partie audio est antérieure au refacto) · `.agents/reference-utilitaires.md` (`sentence-tokenizer`, `pattern-validator`, `AudioEngine`).
- **Utilitaires référencés** : `sentence-tokenizer.ts` (`tokenizeSentence`, `extractSymbols`, `extractUniqueSymbols`) · `pattern-validator.ts` (`areAllSymbolsValid`) · `instrument-engine.ts` (`getAllSymbols`) · `audio-engine.ts` (`hasSymbol`, `registerSymbol`, `addToPlayers`, `updateSequences`, boucle maître) · `audio-facade.ts` (`updatePattern`) · `sequence-service.ts` (`preparePattern`) — signatures dans `.agents/reference-utilitaires.md`.
- **Tests** : `sentence-tokenizer.test.ts` · `pattern-validator.test.ts` (`#areAllSymbolsValid`) · `instrument-engine.test.ts` (`#getAllSymbols`) · `audio-engine.test.ts` (`#hasSymbol`, `#registerSymbol`, `#addToPlayers`, `#createSequence`, `#updateSequences`) · `audio-facade.test.ts` (`#updatePattern`).

> Écarté à la compaction (texte complet : version de travail) : Règle d'implémentation (pilotage du développement) · Specs (→ Résumé métier, Scénarios évolution) · État du codebase — ce qui existe (pilotage du développement ; flux Play initial → Arbre de flux, fichiers et fonctions → Traçabilité) · Flux de données complet (→ Arbre de flux) · Phase 0 — Déduplication du tokenizer (pré-requis) (pilotage du développement ; → ADR-1) · Phase 1 — `InstrumentEngine.getAllSymbols()` (pilotage du développement ; → Traçabilité) · Phase 2 — `areAllSymbolsValid()` (pilotage du développement ; → ADR-2) · Phase 3 — `sentence-tokenizer.test.ts` (pilotage du développement ; → Traçabilité) · Phase 4 — `AudioEngine` nouvelles méthodes (pilotage du développement ; → ADR-4, ADR-5, ADR-8, EC-3) · Phase 5 — `updatePattern()` dans la facade audio (pilotage du développement ; → ADR-4) · Phase 6 — `AudioContext.updateTrack()` (pilotage du développement ; → ADR-6) · Phase 7 — Intégration dans `PatternWorkspace` (pilotage du développement ; → ADR-3) · Phase 8 — Mise à jour de `createSequence` pour le chargement initial (pilotage du développement ; → ADR-5) · Ordre d'implémentation (TDD) (pilotage du développement) · Liste exhaustive des fichiers à créer (pilotage du développement ; → Traçabilité) · Liste exhaustive des fichiers à modifier (pilotage du développement ; → Traçabilité) · Fichiers qui ne nécessitent AUCUN changement (pilotage du développement) · Bug — Désynchronisation des animations (step loop) après update dynamique (→ Leçons, ADR-7, ADR-11) · Phase 10 — Correction du saut audio pendant le live editing (master Tone.Loop) (→ ADR-7, ADR-8, Leçons, Arbre de flux — lecture temps réel) · Cas limites et pièges (→ Dette connue) · Avancement (pilotage du développement) · Phase 11 — Correction de la subdivision des groupes `(A B C)` dans le master Tone.Loop (→ ADR-10, Leçons) · Notes / décisions (→ ADR-1, ADR-4, ADR-5, ADR-7)

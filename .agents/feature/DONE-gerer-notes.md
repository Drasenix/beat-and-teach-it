# Feature: Gérer les notes (pitch shifting par instrument)

> Clôturé le 2026-06-23 · version de travail : `git show 66eeb90:.agents/feature/DONE-gerer-notes.md`

## Résumé métier

Un instrument peut porter une **fréquence de référence** : sa hauteur naturelle, détectée par autocorrélation sur son sample ou saisie à la main. Une phrase peut ensuite préciser la hauteur de chaque token avec un suffixe `@` — en hertz (`Hum@440`) ou en nom de note (`Hum@A#4`) — sans créer de fichier audio par note. À la lecture, le sample est rejoué à une vitesse calculée (`playbackRate = cible / référence`, bornée à ±2 octaves) : la hauteur change, la durée du sample aussi (pas de time-stretch en V1). Sur un step, un cran de molette monte ou descend d'un demi-ton (le texte est réécrit en note juste), le double-clic retire la hauteur, et le survol ouvre un tooltip unique qui réunit la bande de 17 notes, les 5 couleurs, le mute et le reset. La fréquence vit dans le texte de la phrase, seule source de vérité : aucune colonne ni aucun état miroir n'est stocké, et `patterns` reste inchangé. L'export `.beatpack` passe en manifest `version: 2` avec `referenceFrequency` ; un manifest v1 reste importable.

## Scénarios évolution — nouveau comportement

> Comportement reconstruit à la compaction depuis les décisions actées, le flux de données, les cas limites et les corrections (la version de travail n'avait pas de scénarios GWT).

#### Instrument — fréquence de référence

**Cas nominaux**

- Given: un instrument existant sans référence
  When: on saisit `440` dans le champ fréquence et on sauvegarde
  Then: `instruments.reference_frequency = 440`, l'instrument devient pitchable
- Given: un sample d'environ 1 s contenant un son stable
  When: on clique sur « Détecter »
  Then: le champ est pré-rempli avec la fréquence détectée (Hz), modifiable avant sauvegarde

**Cas limites**

- Given: un sample trop court (< 0,02 s), silencieux ou de confiance < 0,5
  When: on clique sur « Détecter »
  Then: aucune fréquence n'est proposée, sans erreur
- Given: le formulaire de création d'instrument
  When: on saisit un symbole contenant `@`
  Then: la validation refuse le symbole (ambiguïté avec le suffixe de fréquence)

#### Saisie — notation `@fréquence`

**Cas nominaux**

- Given: une phrase vide
  When: on saisit `Hum@440 P (Ts@220 K) .`
  Then: la grille affiche Hum pitché à 440, P sans fréquence, Ts pitché à 220 dans le groupe, et la phrase stockée garde le texte exact (aucun tableau de fréquences)
- Given: une phrase `Hum@A#4`
  When: la lecture prépare le pattern
  Then: la fréquence vaut 466,16 Hz (tempérament égal, A4 = 440), comme pour `Hum@466.16`

**Cas limites**

- Given: `P@0`, `P@abc` ou `P@`
  When: la grille parse le token
  Then: P est affiché sans fréquence, le suffixe est ignoré sans erreur et le token reste valide
- Given: un groupe `(Hum@440 Hum@554)`
  When: la grille l'affiche
  Then: chaque sous-token porte sa propre fréquence

#### Molette et tooltip sur un step

**Cas nominaux**

- Given: un step `Hum@440` d'un instrument à référence 220
  When: on fait un cran de molette vers le haut
  Then: le texte devient `Hum@A#4` (466,16 Hz, +1 demi-ton, pas un flottant) et le textarea comme la grille se mettent à jour ensemble
- Given: un token dans un groupe `(A B C)`
  When: on tourne la molette sur le 2e sous-step
  Then: seul B est réécrit (index de token aplati)
- Given: un token pitché
  When: on double-clique sur son badge
  Then: le `@fréquence` est retiré et le step reprend la hauteur naturelle de l'instrument
- Given: un step dont l'instrument a une référence
  When: on survole le badge
  Then: le tooltip affiche la bande de 17 notes (±8 demi-tons autour de la note courante), les 5 couleurs, le mute et le reset (si le token est pitché)

**Cas limites**

- Given: un instrument sans fréquence de référence
  When: on tourne la molette sur ses steps
  Then: rien ne change (aucun playbackRate calculable)
- Given: un instrument sans fréquence de référence
  When: on survole un de ses steps
  Then: la bande de notes est masquée, les couleurs et le mute restent disponibles
- Given: un step affiché dans la grille
  When: on clique simplement sur le badge
  Then: le mute ne bascule pas : il ne se commande que par le tooltip

#### Lecture pitchée

**Cas nominaux**

- Given: un instrument de référence 220 et un token à 466,16 Hz
  When: on lance la lecture
  Then: `playbackRate = clamp(466,16 / 220, 0,25, 4) = 2`, `semitoneOffset = 12`, le sample est joué 2× plus vite (durée divisée par 2)
- Given: deux pistes, même instrument, même step, rates différents
  When: la lecture démarre
  Then: chaque piste joue sa propre vitesse (aucun rate partagé)
- Given: une lecture en cours et une hauteur modifiée dans la grille
  When: le cycle suivant démarre
  Then: la nouvelle hauteur est prise en compte sans arrêter la lecture (le symbole nettoyé du `@fréq` reste reconnu)

**Cas limites**

- Given: une fréquence cible très éloignée de la référence
  When: la lecture prépare le pattern
  Then: le playbackRate est borné à `[0,25, 4]`, pas rejeté
- Given: un instrument sans référence (`null` ou ≤ 0)
  When: la lecture lit un token pitché
  Then: `playbackRate = 1`, aucun décalage, aucune erreur
- Given: deux pistes alignées sur le même temps avec le même instrument
  When: la lecture tourne plusieurs cycles
  Then: les pistes restent synchronisées (un Loop et un compteur de step par piste)

**Cas d'erreur**

- Given: `Ctrl+Enter` pressé deux fois rapidement
  When: la seconde frappe arrive pendant le démarrage
  Then: aucune seconde lecture n'est lancée, aucune erreur Tone.js

#### Bibliothèque

- Given: un instrument à référence 440
  When: on exporte un `.beatpack`
  Then: le manifest est en `version: 2` avec `referenceFrequency` par instrument
- Given: un `.beatpack` v1 (sans `referenceFrequency`)
  When: on l'importe
  Then: les instruments sont importés avec `referenceFrequency = null`, sans erreur
- Given: des patterns v1 dont les phrases n'ont pas de `@fréquence`
  When: on les importe puis on les lit
  Then: la lecture est inchangée (aucun playbackRate)

## Dette connue

| #    | Edge case / sujet                                                                                                                          | Repris par | Justification                                                                  |
| ---- | ------------------------------------------------------------------------------------------------------------------------------------------ | ---------- | ------------------------------------------------------------------------------ |
| EC-1 | Scroll trackpad continu sur un badge : chaque cran vaut ±1 demi-ton                                                                        | V2         | Ajustement du pas au scroll continu repoussé                                   |
| EC-2 | Deux fréquences distinctes d'un même demi-ton réutilisent le Player créé au premier usage, donc son rate exact                             | —          | Clé du pool = `semitoneOffset` ; seul le playbackRate de création est appliqué |
| EC-3 | Pas de time-stretch : la durée du sample varie avec la hauteur                                                                             | —          | Exclu V1 (décision 13)                                                         |
| EC-4 | Changer la fréquence de référence après création de patterns : les Hz stockés dans les phrases ne bougent pas, la hauteur de sortie change | —          | Comportement documenté, pas un bug                                             |
| EC-5 | Parcours de bout en bout (enregistrer → détecter → pitcher → jouer) sans test automatisé                                                   | —          | Case « Intégration bout en bout » restée non cochée dans la version de travail |
| EC-6 | Instrument sans fréquence de référence : non pitchable, molette inopérante                                                                 | —          | Comportement voulu (`SP-NOTE-02`)                                              |

Couverts : parsing `@fréquence`, molette et reset, calcul du playbackRate, manifest v2 (tests listés en Traçabilité) ; cas limites détaillés dans `SP-NOTE-01` et `SP-NOTE-02`.

## Arbre de flux

**Flux de données**

```
Création instrument
  ├── Formulaire : symbole, nom, fichier audio
  ├── [Bouton "Détecter"] → autocorrélation → 440 Hz
  ├── Champ "Fréquence de référence" : 440 Hz (modifiable)
  └── Sauvegarde → INSERT instruments (..., reference_frequency = 440)

Saisie pattern
  ├── Textarea : "Hum@440 P (Ts@220 K) ."
  ├── SentenceInput.onChange → changeSentence(index, "Hum@440 P (Ts@220 K) .")
  ├── usePattern → stocke le texte brut dans pattern.sentences
  └── Grid : parseSteps → PatternStep[] avec frequency extraite

Molette sur StepBadge
  ├── onWheel → deltaY → ±1 demi-ton → nouvelle fréquence
  ├── updateTokenFrequency(sentence, flatTokenIndex, newFreq)
  │     → "Hum@440 P (Ts@220 K) ."  devient  "Hum@466 P (Ts@220 K) ."
  └── changeSentence(index, newSentence) → re-render automatique

Lecture (Play)
  ├── sentencesForPlayback = transformSentencesWithMute(sentences, mutedSteps)
  ├── preparePattern("Hum@A#4 P (Ts@C4 K) .")
  │     ├── tokenizeSentence → ["Hum@A#4", "P", "(Ts@C4 K)", "."]
  │     ├── parseToken("Hum@A#4") → { symbol: "Hum", frequency: 466.16 }
  │     ├── getInstrumentReferenceFrequency("Hum") → 220
  │     ├── playbackRate = clamp(466.16 / 220, 0.25, 4) = 2
  │     ├── semitoneOffset = round(12 * log2(2)) = 12
  │     └── SequenceNote { name: "hum", playbackRate: 2, semitoneOffset: 12 }
  └── AudioEngine.createSequence
        └── Loop callback : playNote → getPoolPlayer("hum", offset=12, rate=2)
              ├── 1er usage → new Tone.Player(buffer).toDestination()
              │               player.playbackRate = 2  (fixé UNE FOIS)
              │               stocké dans playerPool[0]["hum"].get(12)
              └── player.start(time)  (pas de setter)
```

## Décisions & Justifications — ADR

| #   | Décision                                                                                          | Raison                                                                                                                       |
| --- | ------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| 1   | Pas de flag « mélodique »                                                                         | Tout instrument est pitchable, c'est l'utilisateur qui décide d'appliquer ou non une fréquence                               |
| 2   | Notation `Hum@440`                                                                                | `@` n'est pas utilisé dans la syntaxe actuelle, se lit « at »                                                                |
| 3   | Fréquence brute (Hz) stockée dans le texte                                                        | Stockage simple, pas de mapping note → fréquence                                                                             |
| 4   | Molette = ±1 demi-ton                                                                             | Musical, intuitif. `freq * 2^(±1/12)`                                                                                        |
| 5   | Détection de hauteur en V2 (autocorrélation)                                                      | Via un bouton « Détecter » dans le formulaire instrument                                                                     |
| 6   | Champ `referenceFrequency` dans `instruments`                                                     | Stocké en BDD, détecté une fois, modifiable manuellement                                                                     |
| 7   | `PatternStep` enrichi de `frequency?: number`                                                     | Extraite du texte par le parser                                                                                              |
| 8   | `SequenceNotes` évolue vers `{ name, playbackRate }`                                              | Le playbackRate est pré-calculé avant l'envoi à AudioEngine                                                                  |
| 9   | Pas de migration `patterns`                                                                       | Les `@freq` vivent dans les `sentences` existantes                                                                           |
| 10  | Migration `instruments` : `ALTER TABLE ADD reference_frequency REAL`                              | Seule migration BDD nécessaire (`003_reference_frequency.ts`)                                                                |
| 11  | Manifest library : version 1 → 2                                                                  | Ajout de `referenceFrequency` dans les instruments exportés                                                                  |
| 12  | `playbackRate` clampé à 0,25–4                                                                    | Évite les taux extrêmes (−2 à +2 octaves)                                                                                    |
| 13  | Pas de time-stretch en V1                                                                         | Accepté : la durée du sample change avec le pitch                                                                            |
| 14  | `@` interdit dans les symboles d'instruments                                                      | Validation ajoutée pour éviter les ambiguïtés                                                                                |
| 15  | Le texte de la phrase est la **source unique de vérité** des fréquences (Option D)                | Un état dérivable du texte ne se stocke pas à côté : le miroir se désynchronise et le `@freq` nettoyé disparaît de la saisie |
| 16  | `@A#4` (nom de note) accepté en plus de `@440` (Hz), rétrocompatible ; affichage toujours en note | Lisibilité : on ne lit pas `@440.16376` dans une grille                                                                      |
| 17  | La molette et la saisie écrivent une note juste (snap au demi-ton le plus proche)                 | Pas de flottants dans le texte, la grille reste lisible                                                                      |
| 18  | Double-clic sur un badge = reset de la hauteur                                                    | Retour au son naturel sans passer par la saisie                                                                              |

**Corrections d'implémentation.** La décision 5 annonce la détection en V2 : elle a été livrée dans cette feature (`detectPitch`, hook `useDetectPitch`, bouton « Détecter »). Le type de la décision 8 porte aussi `semitoneOffset`, qui sert de clé au pool de Players.

## Leçons de réalisation

- L'analyse #1 stockait les fréquences dans un tableau miroir `frequencies` plus une colonne BDD, et `changeSentence` nettoyait le `@440` du texte (le textarea perdait l'affichage) → direction Option D : les `@freq` vivent dans `sentences` → un état dérivable du texte ne se stocke pas à côté (ADR-15).
- Crainte « deux pistes, même instrument, même step, playbackRate différents » → vérifié : chaque `player.start()` crée sa propre source audio → ne pas complexifier sur une crainte non vérifiée.
- `@freq` cassait les consommateurs de symboles : `extractUniqueSymbols` enregistrait `Hum@440` (hot-reload cassé en lecture) et `areAllSymbolsValid` renvoyait `false` (aucune mise à jour de séquence en lecture) → `parseToken()` appliqué avant comparaison → tout consommateur de symbole nettoie via `parseToken` (`.agents/reference-utilitaires.md`).
- Trois erreurs Tone.js « The time must be greater than or equal to the last scheduled time » : double `playTrack` (Ctrl+Enter), `updateSequences` + `start()` sur le même instrument, puis le setter `playbackRate` à chaque cycle → `playingRef` posé avant l'`await`, puis 1 `Tone.Loop` + 1 `Tone.Player` par piste et pool lazy (rate fixé à la création) → ne jamais modifier la timeline d'un Player en lecture.
- Les pistes jouaient avec un temps de retard (compteur `stepIndex` partagé) → `localStep` en closure de chaque Loop → un compteur de pas ne se partage pas entre pistes.
- La molette écrivait des fréquences flottantes (`@440.16376`) → passage à `@A#4`, snap au demi-ton, `noteNameToFrequency` / `frequencyToNoteName` → une hauteur s'écrit en note.

## Vigilances

- `Tone.Player.playbackRate` est un setter qui réécrit la `StateTimeline` du Player (`cancel(stopEvent)` + `setStateAtTime`) : une assignation pendant la lecture annule les events futurs et lève « The time must be greater than or equal to the last scheduled time ». Le pool lazy fixe le rate une seule fois à la création, puis `start(time)` seulement.
- Le pool est indexé par demi-ton (`semitoneOffset`), pas par fréquence : deux fréquences du même demi-ton partagent le Player du premier usage et donc son rate. Changer le calcul du rate sans changer la clé fausse la hauteur.
- 1 `Tone.Loop` par piste avec son `localStep` : un compteur de step partagé entre pistes les décale d'un temps.
- `playingRef` doit être posé avant le premier `await` de `playTrack` et remis à `false` dans `stopTrack` et en cas d'erreur : sinon un double Ctrl+Enter lance deux séquences.
- Le tooltip de step est rendu dans `document.body` (`createPortal`, `position: fixed` calculée au `mouseenter`) : le replacer dans le badge lui ferait hériter du `transform` / `filter` de `step-active-pulse`.

## Traçabilité

- **Spec et US** : §2.1 (Instrument), §2.3 (Phrase), §6.1 (flux de composition), §13 (schémas SQL), §4.4 / §6.3 (`.beatpack`).
- **Spec parallèle** : `SP-NOTE-01`, `SP-NOTE-02`, `SP-AUDIO-01` (AMENDE §6.1).
- **Conventions établies** : `.agents/conventions.md` (Instruments, Audio, Sequence — index aplati, pitch, `@` interdit) · `.agents/architecture.md` (§13 amendé par `SP-NOTE-02`) · `.agents/domain-glossary.md` (Fréquence de référence, `.beatpack`) · `.agents/ui-rules.md` (5 couleurs de highlight dans `StepTooltip`).
- **Utilitaires référencés** : `token-parser.ts`, `sentence-tokenizer.ts`, `pattern-parser.ts`, `pattern-validator.ts`, `detect-pitch.ts`, `note-name-to-frequency.ts`, `frequency-to-note.ts`, `update-token-frequency.ts` (détails et pièges dans `.agents/reference-utilitaires.md`).
- **Tests** : `token-parser.test.ts`, `sentence-tokenizer.test.ts`, `pattern-parser.test.ts`, `update-token-frequency.test.ts`, `note-name-to-frequency.test.ts`, `detect-pitch.test.ts`, `sequence-service.test.ts`, `export-library-service.test.ts`, `import-library-service.test.ts`, `audio-engine.test.ts`.

> Écarté à la compaction (texte complet : version de travail) : Problème (→ Résumé métier) · Solution cible (→ Résumé métier / ADR-15) · Flux de pensée & analyses contradictoires (→ ADR-15 / Leçons de réalisation) · Décisions actées (→ ADR-1 à 14) · Flux de données (→ Arbre de flux) · Cas limites (→ Dette connue / Scénarios) · Architecture cible (pilotage du développement ; Modèle de données → ADR-7, ADR-10 · AudioEngine — pool lazy → Arbre de flux / Vigilances · Types → ADR-8 · Nouveaux fichiers → Traçabilité · Fichiers modifiés → Traçabilité ; la migration réelle est `003_reference_frequency.ts`) · Avancement (pilotage du développement) · Notes / décisions (→ ADR-15 à 18) · Améliorations UX (v2) (→ Résumé métier / Scénarios : comportement livré) · Bugs corrigés (→ Leçons de réalisation / Vigilances)

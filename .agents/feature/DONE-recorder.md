# Feature: recorder

> Clôturé le 2026-06-16 · version de travail : `git show 66eeb90:.agents/feature/DONE-recorder.md`

## Résumé métier

Écran `/recorder` pour enregistrer un instrument directement au microphone, éditer la forme d'onde (trim début/fin), pré-écouter et sauvegarder en WAV via le dialogue natif. La création de l'instrument reste dans l'écran Instruments : le recorder produit un fichier, il n'ajoute pas l'instrument au catalogue.

## Scénarios évolution — nouveau comportement

> Reconstitués depuis le flux utilisateur (fichier antérieur au gabarit).

1. Étant donné l'écran `/recorder` à l'état `idle`, quand je clique « Enregistrer » et j'accorde le micro, alors l'état passe à `recording` (cercle rouge + timer).
2. Étant donné un enregistrement terminé, quand je déplace les curseurs de trim puis clique « Rogner », alors la waveform et la pré-écoute reflètent la portion conservée.
3. Étant donné un enregistrement rogné, quand je clique « Sauvegarder », alors le dialogue natif s'ouvre et le WAV est écrit au chemin choisi ; l'état passe à `saved`.
4. Étant donné le fichier WAV sauvegardé, quand je vais dans Instruments → « Ajouter un instrument », alors je peux le sélectionner ; le recorder ne crée pas l'instrument lui-même.

## Dette connue

| #    | Edge case / sujet                                                       | Repris par        | Justification                          |
| ---- | ----------------------------------------------------------------------- | ----------------- | -------------------------------------- |
| EC-1 | `ScriptProcessorNode` est déprécié                                      | V2                | Remplaçable par `AudioWorklet`         |
| EC-2 | Pas d'auto-détection des silences, pas de découpe multiple, pas de zoom | —                 | Hors V1, trim manuel assumé            |
| EC-3 | Playhead visuelle sans seek                                             | —                 | Hors V1                                |
| EC-4 | Création d'instrument non gérée par le recorder                         | Écran Instruments | Le formulaire existant fait le travail |
| EC-5 | Tests limités à `wav-encoder` ; waveform et recorder non couverts       | —                 | Frontend, hors TDD                     |

Couverts : `wav-encoder.test.ts` (encodage WAV).

## Arbre de flux

```
/recorder
  ├─ idle → « Enregistrer » → recording (pulse + timer)
  ├─ recording → « Arrêter » → recorded (waveform + trim)
  ├─ recorded → « Rogner » → pré-écoute (playhead)
  ├─ recorded → « Sauvegarder » → dialogue natif → IPC → saved
  └─ saved → « Effacer » / « Nouvel enregistrement » → idle
```

## Décisions & Justifications — ADR

| #   | Décision                                                                      | Raison                                                                                        |
| --- | ----------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| 1   | WAV PCM 16 bits mono, encodé sans dépendance                                  | Pas de dépendance MP3 (lamejs) ; format lisible par le formulaire instrument                  |
| 2   | `getUserMedia` → `AudioContext` → `MediaStreamSource` → `ScriptProcessorNode` | `AudioWorklet` trop complexe pour le besoin ; `ScriptProcessorNode` déprécié mais fonctionnel |
| 3   | Trim manuel uniquement (curseurs début/fin)                                   | Pas d'auto-détection des silences ; l'utilisateur règle visuellement                          |
| 4   | Sauvegarde via `open-audio-save-dialog` puis IPC `save-recorded-audio`        | Emplacement et nom libres, pattern IPC existant                                               |
| 5   | Playhead visuelle sans seek, boucle `requestAnimationFrame` (60 FPS)          | Mouvement fluide ; `timeupdate` natif trop lent ; pas de seek demandé                         |
| 6   | Contrôles audio custom avec les tokens du thème                               | Cohérence UI ; les contrôles natifs ne suivent pas le thème                                   |
| 7   | Le recorder ne crée pas d'instrument                                          | L'utilisateur passe par le formulaire Instruments (`open-file-dialog`, filtre `.wav`)         |

## Vigilances

- `ScriptProcessorNode` est déprécié : ne pas le remplacer par `AudioWorklet` sans re-tester l'encodage WAV et le trim.
- Le recorder ne crée pas d'instrument : ne pas ajouter une création implicite dans l'écran d'enregistrement ; le parcours reste Recorder → Instruments.
- Les canaux `save-recorded-audio` et `open-audio-save-dialog` sont déjà dans `preload.ts` : ne pas les dupliquer.

## Traçabilité

- **Spec et US** : aucune section de spec officielle ; parcours `/recorder` (feature hors spec).
- **Spec parallèle** : aucun.
- **Conventions établies** : `.agents/architecture.md` (18 IPC, `recorder/`) · `.agents/conventions.md` (Recorder) · `.agents/reference-utilitaires.md` (`wav-encoder.ts`, `waveform-renderer.ts`).
- **Utilitaires référencés** : `wav-encoder.ts`, `waveform-renderer.ts`, `useRecorder`, `useWaveformEditor`.
- **Tests** : `wav-encoder.test.ts`.

> Écarté à la compaction (texte complet : version de travail) : Specs (→ Résumé métier, Scénarios, ADR, Dette connue) · Avancement (pilotage du développement) · Notes / décisions (→ ADR, Dette connue)

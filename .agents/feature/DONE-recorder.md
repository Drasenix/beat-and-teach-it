# Feature: recorder

## Specs

Nouvel écran permettant d'enregistrer un instrument directement depuis l'application via le microphone.

### Contexte
- Route : `/recorder`
- Icône nav : `⏺` (U+23FA), titre "Enregistreur"
- Position dans la sidebar : 3e (après Instruments, avant Patterns)
- Navigation : Header.tsx + Home.tsx

### Stack technique
- Capture : `getUserMedia` → `AudioContext` → `MediaStreamSource` → `ScriptProcessorNode`
- Format : WAV (PCM 16-bit mono), encodé côté rendu sans dépendance
- Sauvegarde : dialogue natif `open-audio-save-dialog` → l'utilisateur choisit emplacement + nom → IPC `save-recorded-audio` → main écrit au chemin choisi
- Création instrument : **non gérée par le recorder** — l'utilisateur passe par l'écran Instruments
  (`/configuration/instruments`) via le formulaire existant + `open-file-dialog` (filtre déjà `.wav`)

### Flux utilisateur
1. Page `/recorder` → état `idle` → bouton "⏺ Enregistrer"
2. Clic → demande permission micro → état `recording` (cercle rouge + timer)
3. Clic "⏹ Arrêter" → arrêt micro → état `recorded`
4. **Waveform editor** apparaît : rendu canvas du signal audio avec deux curseurs de trim
5. L'utilisateur déplace les curseurs début/fin pour sélectionner la zone à conserver
6. Clic "Rogner" → génère un nouveau `Float32Array` avec la portion sélectionnée → pré-écoute
7. Clic "Sauvegarder" → dialogue natif `open-audio-save-dialog` (choix emplacement + nom) → IPC `save-recorded-audio` → état `saved`
8. Clic "🗑 Effacer" ou "⏺ Nouvel enregistrement" pour recommencer
9. L'utilisateur va dans Instruments → "Ajouter un instrument" → sélectionne le fichier `.wav`
   sauvegardé via le dialogue natif

### Waveform editor
- **Rendu** : `<canvas>` remplissant la largeur du conteneur, hauteur fixe (ex: 160px)
- **Signal** : downsampling du `Float32Array` pour affichage (1 pixel = N samples), trait blanc sur fond field
- **Curseurs** : deux poignées verticales draggable (début = position 0, fin = position 100%)
- **Zone sélectionnée** : fond semi-transparent primary entre les deux curseurs
- **Bouton "Rogner"** : slice le `Float32Array` entre les indices début/fin, recrée un waveform affiché
- **Pré-écoute** : `<audio>` caché comme moteur, contrôles customs stylés (play/pause + barre progression + temps)
- **Playhead** : ligne verticale mobile sur le canvas qui suit `currentTime / duration` du `<audio>` en temps réel, visuelle uniquement (pas de seek au clic)
- **Lissage** : boucle `requestAnimationFrame` (60 FPS) plutôt que `timeupdate` natif (4-20 Hz) pour un mouvement fluide
- Pas d'auto-détection des silences, pas de découpe milieu, pas de zoom

### UI
- Layout : `content-page` + `workspace-section-content` (conforme au style existant)
- Boutons : `btn-primary`, `btn-secondary`
- Animation : pulse rouge custom `@keyframes record-pulse`

### Fichiers à créer
- `src/renderer/features/recorder/models/recorder-model.ts`
- `src/renderer/features/recorder/services/recorder-service.ts`
- `src/renderer/features/recorder/utils/wav-encoder.ts`
- `src/renderer/features/recorder/utils/waveform-renderer.ts` — rendu canvas du waveform
- `src/renderer/features/recorder/hooks/useRecorder.ts`
- `src/renderer/features/recorder/hooks/useWaveformEditor.ts` — logique trim + curseurs
- `src/renderer/features/recorder/components/RecorderScreen.tsx`
- `src/renderer/features/recorder/components/WaveformEditor.tsx` — composant canvas + curseurs

### Fichiers à modifier
- `src/main/preload.ts` : ajout channels `save-recorded-audio` + `open-audio-save-dialog`
- `src/main/icpEvents.ts` : handler `save-recorded-audio` + `open-audio-save-dialog`
- `src/renderer/features/recorder/services/recorder-service.ts` : appelle `open-audio-save-dialog` avant `save-recorded-audio`
- `src/renderer/App.tsx` : route `/recorder`
- `src/renderer/components/Header.tsx` : nav item position 3
- `src/renderer/components/Home.tsx` : lien home

## Avancement
- [ ] Specs validées
- [x] Tests wav-encoder
- [x] waveform-renderer.ts (downsample + renderWaveform + trimSamples)
- [x] useWaveformEditor hook (canvas ref + drag trim)
- [x] RecorderScreen + states (idle/recording/recorded/saving/saved)
- [x] WaveformEditor component (canvas + curseurs + Rogner)
- [x] IPC save-recorded-audio (déjà existant)
- [x] Navigation et route (déjà existant)

## Notes / décisions
- Encodage WAV plutôt que MP3 (pas de dépendance lamejs)
- ScriptProcessorNode (déprécié mais fonctionnel) plutôt qu'AudioWorklet (trop complexe pour ce besoin)
- Trim manuel uniquement (pas d'auto-détection des silences) — l'utilisateur règle visuellement
- Pas de découpe multiple, pas de zoom — basique et efficace
- Sauvegarde via dialogue natif (emplacement + nom libre) plutôt que chemin automatique
- Playhead visuelle uniquement (pas de seek) — line couleur primary plus lumineuse
- RAF loop pour playhead fluide (60 FPS) plutôt que `timeupdate` natif (4-20 Hz)
- Contrôles audio customs (play/pause, barre progression, temps) stylés avec les tokens du thème (primary, field, border, text-secondary) au lieu des contrôles natifs


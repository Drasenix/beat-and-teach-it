# Todo — Évolutions & Correctifs

## Structure

Chaque entrée suit ce format :

```md
- [ ] **Titre court** — *priorité* — `tag1, tag2`
  Description courte, contexte, pistes d'implémentation.
```

**Priorités :** `P0` (bloquant/critique) → `P1` (important) → `P2` (souhaitable) → `P3` (nice to have)

**Tags :** `frontend`, `backend`, `db`, `audio`, `library`, `ux`, `archi`, `bug`, `doc`

---

## 🐞 Bugs

...

---

## ✨ Features

- [ ] **Écran de paramétrage** — *P2* — `frontend, ux`
  Nouvel écran `/settings` permettant de personnaliser l'interface :
  - **Thèmes de couleurs** : choix entre plusieurs palettes (sombre actuel, clair, daltonien, personnalisé) — stockage dans `localStorage` ou base
  - **Taille du texte** : slider ou preset (petit/normal/grand) appliqué via CSS custom properties ou classe racine
  - Piste : `ThemeProvider` ou context dédié, variables CSS redéfinies dynamiquement, synchronisation persistante

- [ ] **Ajout d'instrument par clic dans la sidebar (Studio)** — *P2* — `frontend, ux`
  Sur l'écran Studio, cliquer sur un instrument dans la sidebar l'insère dans la phrase active à la position du curseur.
  - Piste : `SentenceInput` expose une ref + une méthode `insertAtCursor(text)`, la sidebar appelle cette méthode via contexte ou callback
  - Il faut connaître la `SentenceInput` active (focus ou index stocké dans `usePatternSession`)
  - Gérer le cas où aucun `SentenceInput` n'a le focus

- [ ] **Wiki / tutoriel par instrument** — *P2* — `frontend, ux`
  Ajouter un éditeur de contenu riche (texte, images, vidéos embarquées) pour chaque instrument, permettant de créer un mini tutoriel (ex: technique de frappe, contexte d'utilisation).
  - Piste : éditeur WYSIWYG léger (TinyMCE, TipTap/ProseMirror, ou Slate)
  - Vidéos : embed YouTube/Vimeo ou upload + stockage local
  - À décider : écran dédié `/wiki` ou intégration dans l'écran de configuration des instruments (`/configuration/instruments`)
  - Stockage : nouvelle table SQL `instrument_notes` ou colonne `notes` dans `instruments`

---

## ⚡ Améliorations

- [ ] **Améliorer le Guide — notes et détection de fréquences** — *P2* — `frontend, doc`
  Enrichir l'écran `/guide` avec une section dédiée à la théorie des notes (hauteur, fréquence, timbre) et à la détection de fréquences (comment l'appli analyse/utilise le pitch, lien avec les instruments).
  - Piste : nouvelle page ou section dans le Guide existant, schémas, exemples audio, lien vers les visites guidées

...

---

## 🔧 Technique / Architecture

...

---

## 📚 Documentation

...

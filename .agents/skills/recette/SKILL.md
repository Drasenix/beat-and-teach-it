---
name: 'recette'
description: 'Recette manuelle locale après une US passée en DONE-* : scénarios utilisateur dérivés de la feature et base SQLite jetable. À charger avant toute recette manuelle.'
---

# recette

Charger cette skill après une US passée en `DONE-*`, avant de tester l'application à la main.

## Déclenchement

- `/recette`, « recette manuelle », « je dois tester » après une US livrée.

## Garde-fous

1. **Local uniquement** : pas de build packagé pour la recette, sauf si le bug suspecté est dans le packaging (`npm run package`).
2. **Jamais la base réelle** : sauvegarder puis mettre de côté `database.db`, `database.db-wal` et `database.db-shm` avant la recette ; les restaurer à la fin.
3. **Jeu de données jetable** : une base neuve est re-seedée par les migrations (12 patterns, 20 instruments). Les audio importés vont dans `userData/imported-audio/`.
4. Ne rien commiter des artefacts de recette (captures, WAV, base temporaire).

## Workflow

1. **Lire la DONE** : `## Résumé métier`, scénarios, dette connue. Ne pas relire tout le fichier.
2. **Dériver les scénarios** au format « Étant donné / Quand je / Alors je dois voir », un scénario par comportement observable.
3. **Préparer un `userData` isolé** :
   - chemin : `app.getPath('userData')` (sous Linux, généralement `~/.config/<nom de l'app>`) — le vérifier, ne pas le deviner ;
   - sauvegarder `database.db` (+ `-wal`, `-shm`) dans un dossier temporaire ;
   - supprimer ces fichiers pour repartir du seed ;
   - si le lancement accepte `--user-data-dir=<dossier jetable>`, le préférer à la manipulation de la base réelle.
4. **Lancer** : `npm start` (dev). Pour un bug de packaging : `npm run build` puis `npm run package`.
5. **Exécuter** les scénarios dans l'ordre, noter observé vs attendu, faire une capture pour chaque écart.
6. **Restaurer** : remettre la base sauvegardée, supprimer le `userData` jetable et les captures de travail.

## Format de rapport

| Scénario | Attendu                | Observé                 | Verdict |
| -------- | ---------------------- | ----------------------- | ------- |
| <nom>    | <comportement attendu> | <comportement constaté> | OK / KO |

- Pour chaque KO : étapes de reproduction, capture, message d'erreur éventuel.
- Conclure par : « Prêt à livrer », « Bloquant », ou « À reprendre en `/raffiner` / `/retouche` ».
- **Ne pas corriger le code pendant la recette** : ouvrir une feature (ou une retouche) avec le symptôme observé.

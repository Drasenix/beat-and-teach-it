# beat-and-teach-it : Agent Instructions

> Fichier canonique lu par tous les outils IA (opencode, Claude Code).
>
> **Ce document est un index.** Les détails vivent dans `.agents/*.md` et le pack techno. Lire les sous-fichiers à la demande.

## Project

Application desktop **Electron + React + TypeScript** de composition rythmique symbolique : l'utilisateur note des patterns via un langage symbolique textuel, associe des fichiers audio (instruments) et joue le résultat via un séquenceur temps réel (Tone.js). Persistance locale SQLite (better-sqlite3).

→ Stack : **`.agents/stack.md`**

## Quick start

```bash
npm install
npm start
```

→ Commandes : **`.agents/commands.md`**

## Architecture

Application desktop à deux processus (main / renderer) communiquant par **IPC typé**, organisée en **feature-sliced** (domain-driven) sous `src/renderer/features/`.

→ Détails : **`.agents/architecture.md`**

## Conventions

- Logique métier dans les `engine/`, API publique via les `facade/`, persistance/IPC via les `services/`, transfert typé via les DTO/adapter.
- Jamais de logique métier dans un composant : déléguer aux facades.
- Formatter : **Prettier**. Tests : **Jest** (ts-jest, jsdom).

→ Règles : **`.agents/conventions.md`**

## Domain

**Avant de toucher aux modèles métier, lire `.agents/domain-glossary.md`.**

Les règles de comportement / flux / calculs validées hors spec vivent dans `.agents/specs/spec-parallele.md` (entrées `SP-*`) : elles **priment** sur le spec officiel.

En cas d'ambiguïté, **demander à l'utilisateur** plutôt que supposer.

## Règles dures

### Ne jamais
- Modifier une migration existante. Créer une nouvelle migration.
- Mettre de logique métier dans un composant / contrôleur.
- Toucher aux fichiers de config sensibles existants sans accord explicite.
- Committer des secrets.
- Inventer des nuances métier.

### Toujours
- Lancer les tests avant de considérer un changement fini.
- Formater le code après modification.
- Utiliser le fallback documenté si l'outil principal n'est pas accessible localement.
- Valider les données côté main **et** côté renderer.
- Respecter l'immutabilité (nouvelle référence pour tableaux/objets).
- En fin de conversation, proposer de commit, ou de commit et push.

## Rapport d'avancement client

Quand l'utilisateur demande un « point » destiné à un interlocuteur non technique :

1. **Sourcer depuis git** (`git log`, `git show --stat`, diff si besoin).
2. **Traduire en langage métier** : pas de noms de classes/méthodes/fichiers/colonnes/branches/hash.
3. **Pour chaque correction : contexte → problème → solution.**
4. **Pour une évolution** : besoin puis apport, sans inventer de problème antérieur.
5. **Regrouper par thème métier**, signaler le visible immédiatement.
6. **Ne rien inventer** ; si l'impact n'est pas déductible du code, le dire et demander.
7. **Jamais d'heure ni de tranche horaire** : uniquement la date.

## Aide-mémoire

| Je veux… | Lire |
|---|---|
| Versions / deps | `.agents/stack.md` |
| Arborescence et IPC | `.agents/architecture.md` |
| Organiser du nouveau code | `.agents/conventions.md` |
| Retrouver une commande | `.agents/commands.md` |
| Comprendre une entité métier | `.agents/domain-glossary.md` |
| Comprendre un flux / statut / règle | `.agents/specs/spec.txt` (+ index `.agents/specs/README.md`) |
| Vérifier une règle `SP-*` | `.agents/specs/spec-parallele.md` |
| Comprendre le workflow de dev | `workflow-kit/core/index.md` + `workflow-kit/core/agent-workflow.md` |

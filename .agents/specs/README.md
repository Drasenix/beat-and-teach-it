# Beat & Teach : Specs fonctionnelles (index)

> **Source de lecture** : `.agents/specs/spec.txt` (documentation métier de référence).
>
> Cet index n'est PAS chargé automatiquement. Il sert à trouver quelle section lire, puis à déclencher une **lecture ciblée** (grep/offset). **Ne jamais dumper le `.txt` entier.**

## Comment lire le fichier de specs

```
# 1. Trouver la section par mot-clé
Grep(pattern="<mot-clé métier>", path=".agents/specs/spec.txt")

# 2. Lire autour du match
Read(file_path=".agents/specs/spec.txt", offset=<ligne>, limit=<N>)
```

## Table des matières (mappée aux sections)

- **§1 Présentation du domaine** : concept, public cible.
- **§2 Modèle de données métier** : Instrument, Pattern, Phrase, Groupe `(…)`.
- **§3 Règles de gestion (RG1–RG14)**.
- **§4 Parcours utilisateur** (Studio, Instruments, Patterns, Bibliothèque, Guide).
- **§5 Raccourcis clavier**.
- **§6 Flux de données critiques** (composition, résolution symboles, import/export, chargement audio).
- **§7 Palette de couleurs et thème** (amendé par `SP-THEME-01`).
- **§8 Animations et retours visuels**.
- **§9 Autocomplétion**.
- **§10 Données initiales (seed)** : 12 patterns, 20 instruments.
- **§11 Contraintes et validations**.
- **§12 Architecture et organisation du code**.
- **§13 Schémas SQL**.
- **§14 Tests**.

> **Sections amendées** : §2.1, §2.3, §6, §7 et §13 sont précisées par des `SP-*`. Lire l'index du spec parallèle avant ces sections.

## Quick lookup : par concept métier

| Concept                        | Sections à lire                               |
| ------------------------------ | --------------------------------------------- |
| Instrument                     | §2.1, §3 (RG1, RG3, RG7), §13 + `SP-NOTE-02`  |
| Pattern / Phrase               | §2.2, §2.3, §3 (RG2, RG4, RG5) + `SP-NOTE-01` |
| Groupe `(…)`                   | §2.4, §6.1                                    |
| Mute                           | §3 (RG8)                                      |
| Highlights                     | §2.2, §3 (RG9), §7                            |
| Thème / palette                | §7 + `SP-THEME-01`                            |
| Notes / hauteur (`@fréquence`) | §2.1, §2.3, §6.1 + `SP-NOTE-01`, `SP-NOTE-02` |
| Fréquence de référence         | §2.1, §13 + `SP-NOTE-02`                      |
| Import / export `.beatpack`    | §4.4, §6.3, §3 (RG11, RG12) + `SP-NOTE-02`    |
| BPM                            | §3 (RG13), §5                                 |
| Audio / séquenceur             | §6.1, §6.4 + `SP-AUDIO-01`                    |

## Quick lookup : spec parallèle (`SP-*`)

| Concept             | Entrées       |
| ------------------- | ------------- |
| Thèmes              | `SP-THEME-01` |
| Notation de hauteur | `SP-NOTE-01`  |
| Lecture pitchée     | `SP-NOTE-02`  |
| Audio live          | `SP-AUDIO-01` |

## Workflow conseillé pour agent IA

0. **Spec parallèle d'abord** : lire l'index de `.agents/specs/spec-parallele.md` et les `SP-*` liés (priment sur le spec officiel).
1. **Question métier** → chercher le concept dans la table ci-dessus.
2. **Lire uniquement les passages indiqués** (grep → lecture ciblée).
3. **Nuance validée** → entité : `.agents/domain-glossary.md` ; règle de flux/calcul : proposer un `SP-*` (écriture Build après accord, recette étape 6).
4. **Ne jamais** résumer tout le spec « au cas où ».

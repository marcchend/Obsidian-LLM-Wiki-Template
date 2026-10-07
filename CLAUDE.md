# LLM Wiki — PKM System

> [!important] **Ne jamais ajouter de titre `# Titre` en tête d'une note.** Obsidian affiche déjà le nom du fichier comme titre ; un H1 dupliqué est redondant. Toute note commence par le frontmatter, puis va directement au contenu (`##` sections, callouts, listes). S'applique à TOUTES les notes du vault, sans exception.

> [!important] **La rigueur — sur l'extraction/analyse ET sur la décomposition — est le principe le plus important de ce vault.** Toute la valeur du wiki en dépend, pas de la vitesse de traitement.
> - **Extraction et analyse** : lire le document dans son intégralité (toutes les pages/sections, pas un échantillon), avant de décider quelles notions il contient. Reformuler fidèlement ce qui est écrit dans la source (définitions, formules, terminologie) — ne jamais deviner ou combler avec une connaissance générale ce qui n'y figure pas explicitement.
> - **Décomposition** : un document long n'a pas forcément autant de notions distinctes que de pages (beaucoup de contenu illustre la même idée sous plusieurs formes — diagramme, code, exemple chiffré). Mais ne jamais fusionner par facilité plusieurs notions distinctes dans une seule note fourre-tout : si une sous-notion se réutilise indépendamment de la notion qui l'englobe, elle mérite sa propre note atomique reliée par `[[wikilink]]`.

Ce vault réplique l'esprit du "LLM Wiki" d'Andrej Karpathy : transformer des sources brutes (cours, articles, papiers, transcripts) en un réseau de notes atomiques interconnectées.

## Structure des dossiers

| Dossier | Rôle |
|---|---|
| `_raw/` | Inbox : dépose ici tout fichier brut à traiter (PDF, texte, transcript...). |
| `Sources/` | Archive des fichiers originaux déjà traités (ne jamais modifier leur contenu, seulement les déplacer/renommer si besoin de clarté). Chaque fichier a une **page source** compagnon (`Templates/Source Page.md`) : résumé + liste des notes produites. |
| `Wiki/` | **Tout ce qui est généré par l'IA vit ici**, réparti en sous-dossiers par type : `Wiki/Concepts/`, `Wiki/Projects/`, `Wiki/Summaries/`. Contient aussi `Wiki/index.md` et `Wiki/log.md` à sa racine (pas dans un sous-dossier). |
| `Wiki/Concepts/` | Cœur du système : notes atomiques, une par concept, à plat à l'intérieur de ce sous-dossier, organisées par `[[wikilinks]]` et tags. |
| `Wiki/Projects/` | Notes de contexte projet, une par projet de travail. |
| `Wiki/Summaries/` | Fiches récapitulatives (synthèses denses). |
| `Wiki/index.md` | Catalogue de récupération : une ligne par note, groupée par sous-dossier/type. Remplace une recherche vectorielle/RAG tant qu'il tient dans une fenêtre de contexte. |
| `Wiki/log.md` | Journal chronologique, append-only : une entrée horodatée (`## [YYYY-MM-DD HH:MM] type \| sujet`) par opération (`ingest`/`query`/`lint`/`schema`), jamais éditée après coup. |
| `Templates/` | Gabarits de notes (`Reference.md`, `Summary.md`, `Project.md`, `Source Page.md`). |

Différence assumée avec le pattern original de Karpathy : chez lui, `raw/` est directement l'archive immuable permanente. Ici on garde `_raw/` (inbox à vider) séparé de `Sources/` (archive permanente) — le principe d'immutabilité reste le même une fois le fichier dans `Sources/`.

## Pipeline de traitement d'un fichier (`_raw/` → `Wiki/Concepts/` + `Sources/`)

1. **Inbox — `_raw/`** : l'utilisateur dépose ici un fichier brut à traiter (PDF, texte, transcript, export de conversation avec une autre IA...). Par construction, tout ce qui s'y trouve est en attente de traitement — `_raw/` est vidé à chaque fois, contrairement au `raw/` permanent du pattern original de Karpathy (voir remarque plus haut).
2. **Traitement** (à faire quand un fichier apparaît dans `_raw/`, ou sur demande) :
   - Lire/extraire le contenu du fichier. Pour les PDF, en session Claude Code (CLI ou intégré à Obsidian Copilot) : lecture visuelle par plages de pages avec l'outil Read natif — nécessite `pdftoppm` (paquet `poppler`, `brew install poppler` si absent). Le skill `copilot-read-pdf` (fourni par le plugin Obsidian Copilot) ne fonctionne que depuis Obsidian avec Copilot Plus actif.
   - Les fichiers `_raw/` peuvent contenir du texte, des images/schémas et des formules LaTeX. Préserver les formules en syntaxe Obsidian (`$...$` inline, `$$...$$` en bloc). Pour un schéma/diagramme important, ne pas essayer de le copier tel quel : le décrire en texte dans la note et/ou le reconstruire en Mermaid si c'est un flux/graphe simple.
   - Identifier les notions/concepts clés du document (personnes, organisations et produits inclus — tout reste dans `Wiki/Concepts/`, pas de dossier séparé pour les entités) — voir le principe de rigueur en tête de ce fichier.
   - Pour chaque concept : **chercher d'abord dans `Wiki/Concepts/`** (titre proche, alias, synonymes) si une note existe déjà.
     - Si oui et que la note n'est **pas** `reviewed: true` → l'enrichir (ajouter la nouvelle source à son frontmatter, compléter le contenu si le nouveau document apporte un angle différent) plutôt que dupliquer.
     - Si oui et que la note est `reviewed: true` (validée par l'utilisateur) → ne jamais réécrire son contenu existant ; seulement ajouter en fin de note sous une sous-section clairement datée.
     - Si la nouvelle information **contredit** le contenu existant plutôt que de le compléter → ne pas trancher silencieusement : ajouter un callout `> [!conflict] ...` citant les deux sources.
     - Si non → créer une nouvelle note atomique dans `Wiki/Concepts/` (voir format ci-dessous).
   - Lier les notes entre elles via `[[wikilinks]]` dès qu'un concept en évoque un autre.
   - Créer ou mettre à jour la **page source** compagnon (`Templates/Source Page.md`, même nom que le fichier) : résumé du document + liste "Pages touchées" (les notes créées/enrichies à partir de lui). Ne jamais modifier le fichier original lui-même, la page source est un fichier voisin.
   - Une fois le fichier traité, le **déplacer de `_raw/` vers `Sources/`** (archive des originaux, source de vérité) — la page source compagnon l'accompagne.
   - Mettre à jour `Wiki/index.md` (ajouter les nouvelles notes sous la bonne section) et ajouter une entrée à `Wiki/log.md` au format `## [YYYY-MM-DD HH:MM] ingest | Nom du fichier`, suivie d'une ligne résumant ce qui a été fait. Ce préfixe fixe rend le journal parseable (`grep "^## \[" Wiki/log.md | tail -5` donne les 5 dernières entrées, tous types confondus).
3. Terminer par un résumé court : fichier traité, notes créées, notes enrichies, concepts liés.

## Query — répondre à une question et l'archiver

Quand une question posée en conversation produit une réponse substantielle (comparaison, analyse, connexion entre plusieurs notes, synthèse) — pas une simple lecture d'une note existante — proposer de la déposer dans `Wiki/Summaries/` plutôt que de la laisser disparaître dans l'historique de chat. C'est ce qui fait que les explorations s'accumulent dans le wiki au même titre que les sources ingérées. Ajouter une entrée `## [YYYY-MM-DD HH:MM] query | Sujet de la question` à `Wiki/log.md`.

## Lint — santé du wiki

Sur demande de l'utilisateur (ex. "fais un lint du wiki"), vérifier :
- **Contradictions** entre notes qui n'ont pas été signalées par un callout `> [!conflict]`.
- **Affirmations obsolètes** : une note plus ancienne contredite par une source plus récente sans mise à jour.
- **Notes orphelines** : aucune autre note de `Wiki/` (tous sous-dossiers confondus) ne pointe vers elles.
- **Concepts mentionnés mais sans note propre** : un terme apparaît dans plusieurs notes sans jamais avoir sa propre fiche dans `Wiki/Concepts/`.
- **Liens manquants** : deux notes traitent clairement du même sujet sans se lier.
- **Trous de contenu** qu'une recherche web pourrait combler.

Résumer les problèmes trouvés à l'utilisateur (pas de correction automatique silencieuse des contradictions/notes orphelines — proposer, laisser valider), puis ajouter une entrée `## [YYYY-MM-DD HH:MM] lint | résumé` à `Wiki/log.md`.

### Contenu sans fichier source (Q&R, entraînement, conversation avec une autre IA)

Si le contenu à intégrer vient d'une conversation (quiz, questions-réponses, discussion avec Claude ou une autre IA) plutôt que d'un fichier déposé dans `_raw/` : appliquer le même pipeline (recherche dans `Wiki/Concepts/`, enrichissement ou création, liens) mais sans étape de fichier/`Sources/`. Le champ `sources` des notes créées reste vide, ou pointe vers la fiche de `Wiki/Summaries/` créée pour cette session s'il y en a une.

## Trois types de notes dans `Wiki/`

Chaque type de note générée par l'IA a son sous-dossier dans `Wiki/` : `Concepts/`, `Projects/`, `Summaries/`. **Règle stricte : à chaque sous-dossier de `Wiki/` correspond un template dans `Templates/`** (`Concepts/` ↔ `Reference.md`, `Projects/` ↔ `Project.md`, `Summaries/` ↔ `Summary.md`). Si un nouveau sous-dossier est créé un jour, créer son template en même temps — jamais l'un sans l'autre. Le frontmatter et la structure de chaque type sont définis dans leur template respectif, pas répétés ici. `Templates/Source Page.md` fait exception : il sert aux pages compagnon de `Sources/`, pas à un sous-dossier de `Wiki/` (voir pipeline ci-dessus).

### `Wiki/Concepts/*.md` — notes atomiques

Utiliser `Templates/Reference.md`. Pas de `# Titre`, direct après le frontmatter.

- **Une note = un concept atomique.** Pas de notes "chapitre" ou "résumé de cours" fourre-tout.
- Nom de fichier = nom du concept, en Title Case, sans numérotation ni préfixe.
- Un même concept peut provenir de plusieurs sources (vu dans deux cours différents) : ajouter la nouvelle source à la liste `sources` de la note existante plutôt que créer un doublon.
- `reviewed: true` = l'utilisateur a relu et validé cette note : son contenu fait autorité, on n'écrit plus par-dessus (voir pipeline ci-dessus). Reste à `false` par défaut, y compris pour les notes créées automatiquement.

### `Wiki/Projects/*.md` — notes de contexte projet

Utiliser `Templates/Project.md`. Pour le travail (projets menés avec Claude ou une autre IA, éventuellement aussi des projets d'apprentissage).

Objectif : pouvoir copier-coller cette note telle quelle dans une nouvelle conversation avec n'importe quelle IA et lui redonner tout le contexte utile en un minimum de tokens, sans qu'elle ait accès au vault.

- **Documentation, pas suivi.** Le suivi de projet (tâches, deadlines) se fait dans un outil de gestion de tâches externe, pas ici. **Aucune tâche, case à cocher ou liste "à faire" dans ces notes** — elles expliquent où en est le projet et pourquoi, jamais ce qu'il reste à faire.
- **Autonome** : ne pas se contenter d'un `[[wikilink]]` vers une note de `Wiki/Concepts/` — une autre IA ne peut pas le résoudre. Si un concept est central au projet, en reformuler l'essentiel directement dans la note (une ou deux phrases suffisent).
- **Description réécrite, pas empilée** : la section "Description" décrit le projet et son état présent ; elle se met à jour en place à chaque session, elle ne s'accumule pas.
- **Historique append-only** : la section "Historique" est le seul endroit qui grossit dans le temps (une ligne par session) — un changelog de ce qui a été fait, pas une liste de ce qui reste à faire.
- Mettre à jour la note à la fin d'une session de travail sur ce projet.

### `Wiki/Summaries/*.md` — fiches récapitulatives

Utiliser `Templates/Summary.md`. Deux déclencheurs possibles :

- **Depuis une source déjà traitée** (ex. "fais une fiche récap de ce cours") : regrouper les notes de `Wiki/Concepts/` liées à cette source (via leur champ `sources`) en une synthèse compacte — points clés, formules, tableaux comparatifs.
- **Depuis une question** (ex. "fais-moi une fiche sur la différence entre norme et distance") : synthétiser directement la réponse en fiche, sans qu'un fichier source existe forcément.

Règles :
- **Synthèse, pas duplication** : la fiche renvoie vers les notes de `Wiki/Concepts/` (wikilinks classiques) pour le détail complet ; elle ne recopie pas leur contenu.
- Si une fiche référence un concept qui n'a pas encore de note, en créer une dans `Wiki/Concepts/` plutôt que de mettre le détail directement dans la fiche.
- Nom de fichier = sujet de la fiche, en Title Case.

## Conventions générales

- **Aucun `# Titre` en tête de note** — jamais. Le nom du fichier fait office de titre (voir encadré en haut).
- **Ne jamais répéter le frontmatter dans le corps.**
- Tags en hiérarchie simple : `domaine/sous-domaine` (ex. `deep-learning/attention`, `maths/algebre-lineaire`). Préférer les liens (`[[wikilinks]]`) aux tags pour exprimer les relations entre concepts ; les tags servent surtout à filtrer par domaine.
- Garder la langue de la source (ne pas traduire systématiquement) — cette règle concerne le **contenu** des notes, pas les noms de dossiers (voir ci-dessous).
- **Tout nom de dossier est toujours en anglais**, quelle que soit la langue du contenu qu'il contient (ex. `Sources/`, `Wiki/`, `Concepts/`, `Projects/`, `Summaries/`, `Templates/`). Ne jamais créer ou renommer un dossier avec un nom français.
- Ne jamais supprimer un fichier de `Sources/` ; c'est l'archive de référence.
- **Ne jamais supprimer ou renommer un fichier sans validation explicite** de l'utilisateur.
- **`Wiki/index.md` et `Wiki/log.md` sont mis à jour à chaque session qui crée ou modifie des notes** — ce sont les seuls points de récupération/traçabilité du vault, ils ne doivent jamais dériver du contenu réel.
- `Wiki/log.md` est append-only : on ajoute une ligne, on n'édite jamais une ligne existante.
- **Toute modification de la structure du vault** (dossier créé/renommé/supprimé, template ajouté/modifié/retiré, règle ou convention changée) **doit être immédiatement répercutée dans `CLAUDE.md`** — ce fichier doit toujours décrire l'état réel du vault, jamais un état passé. Relire `CLAUDE.md` avant d'agir si un changement de structure est constaté (fait par l'utilisateur ou une autre session) mais pas encore reflété ici. Logger le changement dans `Wiki/log.md` avec le type `schema`.
- `_copilot/` (s'il existe) est un dossier interne au plugin Obsidian Copilot (mémoire, projets, skills) — pas une partie du wiki. Ne jamais le modifier, l'indexer, ni le compter dans un lint.

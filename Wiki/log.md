---
title: Journal d'ingestion
tags: [index]
---

Journal chronologique, append-only — jamais éditer une ligne existante. Chaque entrée commence par un préfixe fixe pour rester parseable en ligne de commande (ex. `grep "^## \[" Wiki/log.md | tail -5`).

---
description: Провести анализ фичи сейчас - G1, analyst, architect, G2 (без петли оркестратора)
agent: orchestrator
---
Проведи анализ фичи $ARGUMENTS.

Загрузи skills `board` и `quality-gates`. Фича должна быть в Sprint или Analysis. Если она в
Backlog, скажи, что сначала нужен `/sprint add`.

Выполни G1 (если фича в Sprint), затем раздел «Analysis» и G2. Итог: вердикт G2, путь к spec
и arch doc, число AC и stories, ADR. Если нужна `/approve`, скажи это.

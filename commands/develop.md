---
description: Провести одну story через разработку и ревью сейчас - G3, developer, G4, ревьюеры, G5
agent: orchestrator
---
Проведи story $ARGUMENTS через разработку и Code Review.

Загрузи skills `board` и `quality-gates`. Story должна быть в Sprint, Ready for Dev,
In Development или Code Review. Выполни шаги от её текущего статуса до G5 включительно:
G3 → developer → G4 → ревьюеры → G5. На FAIL верни story в In Development и повтори
разработку, пока не исчерпан `quality.fix_loop_max`.

Итог: статус story, вердикты ревью с SHA, покрытие, замечания, если остались.

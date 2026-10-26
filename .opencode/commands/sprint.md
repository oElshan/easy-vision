---
description: "Спринт: /sprint add <id…> взять в работу, /sprint remove <id…> вернуть в Backlog, /sprint без аргументов показать, что в работе"
agent: orchestrator
---
Команда спринта. Аргументы: $ARGUMENTS

Загрузи skill `board`.

- `add <id> [<id> …]`: каждую карточку из Backlog переведи в Sprint, строка журнала
  «взято в работу человеком». Карточку не из Backlog не трогай и скажи почему.
- `remove <id> [<id> …]`: карточку из Sprint верни в Backlog. Карточку, которая уже дальше
  Sprint, не трогай: её останавливает `/stage` или парковка.
- без аргументов: карточки в Sprint и всё, что в работе (статусы от Analysis до Deploy),
  с прогрессом фич.

Режим `github`: push изменённых карточек (skill `github-sync`). В конце предложи
`/orchestrate`.

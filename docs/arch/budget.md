# Architectural Budget: ratchet сложности

Самозапрет через ресурс: каждая единица архитектурной сложности учитывается здесь.
При достижении threshold новый ресурс требует **ADR** (в arch doc фичи или в `docs/adr/`).
Файл обновляют analyst и architect в том же коммите, что spec и arch doc. Оркестратор
проверяет его на G2, java-reviewer сверяет с ним код.

## Threshold → ADR обязателен

| Событие | ADR |
|---|---|
| Новый микросервис / модуль | ВСЕГДА |
| Новый external call (третья сторона) | ВСЕГДА |
| Новая очередь / exchange / topic | ВСЕГДА |
| Новая БД / схема / кэш-технология | ВСЕГДА |
| Новый слой абстракции внутри модуля | Если модуль уже имеет ≥ 3 слоёв |
| Новая библиотека в shared-contracts | ВСЕГДА |
| Новый CronJob / background worker | Если их уже ≥ 5 |

## Текущий учёт

> Стартовое состояние заполняет `/setup` сканом проекта, дальше файл ведут analyst и architect.

| Категория | Кол-во | Список |
|---|---|---|
| Микросервисы / модули | 1 | `vision-assistant` (backend) |
| External calls | 1 | AI vision/OCR/TTS провайдер (вендор — по ADR) |
| Очереди / topics | 0 | — |
| БД / схемы | 1 | PostgreSQL (схема `public`, таблица `requests`) |
| Cron / workers | 0 | — |

## Журнал изменений

| Дата | Story | Ресурс | ADR |
|---|---|---|---|
| 2026-10-27 | 1-vision-assistant (spec) | модуль `vision-assistant` | нужен (architect) |
| 2026-10-27 | 1-vision-assistant (spec) | внешний AI vision/OCR/TTS провайдер | нужен (architect) |
| 2026-10-27 | 1-vision-assistant (spec) | БД PostgreSQL (схема `public`) | нужен (architect) |

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
| External calls | 1 | AI vision/OCR/TTS провайдер — Yandex Cloud (Vision + SpeechKit) |
| Очереди / topics | 0 | — |
| БД / схемы | 1 | PostgreSQL (схема `public`, таблица `requests`) |
| Объектные хранилища | 1 | MinIO (S3-compatible): buckets `photos`, `audio` |
| Cron / workers | 1 | `VisionProcessor` (in-process `@Scheduled` poller) |

## Журнал изменений

| Дата | Story | Ресурс | ADR |
|---|---|---|---|
| 2026-10-27 | 1-vision-assistant (spec) | модуль `vision-assistant` | ADR-1 (arch) |
| 2026-10-27 | 1-vision-assistant (spec) | внешний AI vision/OCR/TTS провайдер | ADR-2 (arch) |
| 2026-10-27 | 1-vision-assistant (spec) | БД PostgreSQL (схема `public`) | ADR-4 (arch) |
| 2026-10-27 | 1-vision-assistant (arch) | объектное хранилище MinIO | ADR-3 (arch) |
| 2026-10-27 | 1-vision-assistant (arch) | background worker `VisionProcessor` | не нужен (< 5) |

# Architecture: 1 — Приложение для слабовидящих: голосовое описание фото (документы и окружение)

Spec: `docs/specs/1-vision-assistant.md`

## 1. Обзор

Первая фича проекта (greenfield — кода в репозитории ещё нет). Вводит мобильно-ориентированное
PWA для слабовидящих: пользователь фотографирует документ или окружение, система распознаёт
текст (OCR) или описывает сцену и голосом зачитывает результат.

Что появляется:

- **Backend** — единственный Spring Boot 3 сервис `vision-assistant` (модульный монолит на
  один деплой): REST API `/requests`, асинхронный in-process worker (DB-backed очередь через
  `FOR UPDATE SKIP LOCKED`), интеграция с внешним AI-провайдером (vision/OCR + TTS),
  PostgreSQL + Flyway, JWT (ресурс-сервер).
- **Хранилище объектов** — MinIO (S3-совместимое) для фото (временно) и аудио (постоянно).
  Фото в БД не пишется (AC-1-13), бэкенд stateless (≥ 2 экземпляра).
- **Frontend** — Angular PWA: съёмка/выбор фото, выбор режима, отправка, статус обработки,
  экран результата с воспроизведением озвучки, история обращений.

Топология — один backend-сервис (ADR-1), асинхронная обработка без брокера (БД-очередь),
внешний провайдер — Yandex Cloud (ADR-2), хранилище — MinIO (ADR-3), БД — PostgreSQL (ADR-4).

## 2. Затронутые модули → SDC 1 OWNERSHIP

| Модуль / сервис | Изменение | New / Modified | Владеет ресурсами |
|---|---|---|---|
| `vision-assistant` (backend, Spring Boot 3) | Каркас + JWT + API + worker + AI-интеграция + MinIO-клиент | New | таблица `requests`; endpoints `/requests`, `/requests/{id}`, `/requests/{id}/audio`; buckets `photos`, `audio` |
| frontend (Angular PWA) | Съёмка/выбор фото, режим, отправка, статус, результат + озвучка, история | New | — (consumer API) |

Владелец данных — `vision-assistant` (совпадает с `docs/arch/ownership-map.md`). Внешний
AI-провайдер — третья сторона (не владелец, только обработка по контракту). Запись в чужую
таблицу отсутствует (таблица одна). Consumer API — только Angular frontend.

## 3. Модель данных → SDC 1, 4

Одна таблица `requests`: запись обращения и его результат в одной строке (результат — колонки
`result_text` / `audio_object_key`). Отдельная таблица `results` не вводится: обращение 1:1 с
результатом, разделение не даёт ничего, кроме лишнего JOIN.

### Новые сущности

`requests`:

| Колонка | Тип | Ограничения / смысл |
|---|---|---|
| `id` | `UUID` | PK = `requestId` (неугадываемый, генерируется в приложении) |
| `owner_id` | `TEXT` | `sub` из JWT (идентификатор пользователя; object-level scope) |
| `mode` | `TEXT` | `CHECK (mode IN ('DOCUMENT','SCENE'))` |
| `status` | `TEXT` | `CHECK (status IN ('RECEIVED','PROCESSING','DONE','FAILED'))` |
| `idempotency_key` | `TEXT` | значение заголовка `Idempotency-Key` или случайный UUID |
| `photo_object_key` | `TEXT` | ключ объекта в bucket `photos`; `NULL` запрещён в `RECEIVED`; удаляется объект после терминального статуса |
| `result_text` | `TEXT` | текст OCR/описания; непустой при `DONE` |
| `audio_object_key` | `TEXT` | ключ объекта в bucket `audio`; непустой при `DONE` |
| `failure_reason` | `TEXT` | `PROVIDER_UNAVAILABLE` \| `STORAGE_ERROR` \| `INVALID_INPUT`; заполнен при `FAILED` |
| `created_at` | `TIMESTAMPTZ` | `NOT NULL DEFAULT now()` |
| `updated_at` | `TIMESTAMPTZ` | `NOT NULL DEFAULT now()`, обновляется при смене статуса |

Ограничения и индексы:

- `UNIQUE (owner_id, idempotency_key)` — дедупликация повторного POST (SDC 2).
- `INDEX idx_requests_owner_created ON requests (owner_id, created_at DESC)` — история
  `GET /requests` (пагинация, сортировка по убыванию времени).
- `INDEX idx_requests_status_received ON requests (status) WHERE status = 'RECEIVED'` —
  частичный индекс под выборку worker (SDC 7, нет hot-сканов по всей таблице).

Байтов фото/аудио в БД нет (AC-1-13): хранятся объекты в MinIO, в БД — только ключи и текст.
`audioUrl` формируется на чтение как presigned URL по `audio_object_key` (не хранится в БД,
т.к. истекает).

### Миграции

- Файл: `V1__create_requests.sql` (Flyway, каталог `src/main/resources/db/migration/`).
- DDL: `CREATE TABLE requests` + 2 индекса + `UNIQUE` constraint (см. выше).
- Блокировки: мгновенно — greenfield, таблица пустая, `CREATE INDEX CONCURRENTLY` не нужен.
- Rollback: `DROP TABLE IF EXISTS requests;` (greenfield; forward-миграция, rollback-скрипт
  фиксируется здесь, undo-файл Flyway не обязателен).
- Совместимость с текущей версией приложения: N/A — первая версия, старого кода нет.

### Изменения существующих сущностей

Нет — существующей схемы нет.

## 4. API → SDC 6 BLAST RADIUS

### Новые endpoints

| Method | Path | Роль | Request | Response | Коды ошибок |
|---|---|---|---|---|---|
| `POST` | `/requests` | `USER` | multipart/form-data: `file` (JPEG/PNG, ≤ 4 МБ, ≤ 8 МП), `mode` (`DOCUMENT`\|`SCENE`); header `Idempotency-Key` (опционально) | `202` `{ requestId, status: "RECEIVED" }` | `400` (невалидный mode / не-изображение / > 4 МБ), `401`, `413`, `415`, `502` (недоступно хранилище) |
| `GET` | `/requests/{id}` | `USER` | — | `200` `{ requestId, mode, status, result?: { text, audioUrl }, failureReason? }` | `401`, `404` (чужая/несуществующая) |
| `GET` | `/requests` | `USER` | query `page`, `size` | `200` `{ items: [...], page, size, total }`, сортировка `created_at DESC` | `400` (невалидный `page`/`size`), `401` |
| `GET` | `/requests/{id}/audio` | `USER` | — | `302` на presigned URL MinIO (TTL 15 мин) | `401`, `404` |

Идемпотентный `POST`: при повторе с тем же `Idempotency-Key` возвращается `202` с тем же
`requestId` (AC-1-05). `mode` — только `DOCUMENT|SCENE` (AC-1-03). Object-level: все чтения
фильтруются по `owner_id = sub(JWT)`; чужое обращение → `404`, не `403` (AC-1-07).

### Изменённые endpoints

Нет.

### Consumers

Greenfield — потребителей в коде нет. Единственный потребитель API — Angular frontend
(stories S5/S6, ещё не написаны). `grep -rn "RequestDto\|ResultDto"` по репозиторию: N/A
(кода нет). Контракт DTO фиксируется здесь и в `ownership-map.md` (секция Shared contracts);
контрактный тест DTO запланирован в spec (раздел 9).

### Версионирование

Non-breaking + default — первая версия API, версионирование не требуется.

## 5. Сервисный слой → SDC 3 CONSISTENCY

- `RequestService.create(ownerId, mode, file, idempotencyKey) → RequestDto` — валидация,
  загрузка фото в MinIO, вставка строки `RECEIVED` (идемпотентно), возврат `202`.
- `RequestService.get(ownerId, id) → RequestDto` — `404`, если строка не принадлежит `ownerId`.
- `RequestService.list(ownerId, page, size) → Page<RequestDto>` — только свои, пагинация.
- `VisionProcessor` — in-process worker (см. раздел 6/7): приём `RECEIVED` → обработка →
  `DONE|FAILED`, удаление фото.
- `MinioStorage` — `put/get/delete` объектов (фото, аудио), генерация presigned URL.
- `YandexVisionClient` / `YandexSpeechKitClient` — единый внешний вызов к Yandex Cloud
  (конкретные клиенты, без интерфейса-«на-будущее», см. ADR-2 и раздел 12).

### Границы транзакций

- **POST**: загрузка фото в MinIO — вне транзакции; `INSERT … ON CONFLICT DO NOTHING` —
  отдельная короткая транзакция. Один ресурс — конфликтов между БД и object storage нет
  (идемпотентность закрыта unique constraint + удалением orphan-фото, раздел 7).
- **Worker**: приём пачки (`FOR UPDATE SKIP LOCKED` + `UPDATE … SET status='PROCESSING'`) —
  одна короткая транзакция; внешние AI-вызовы — вне транзакции; финализация
  (`UPDATE … SET status='DONE|FAILED', result_text, audio_object_key, failure_reason`) —
  отдельная транзакция. Больше одного ресурса в транзакции нет → outbox/saga не нужны.

### Инварианты

- `id` уникален (PK); `(owner_id, idempotency_key)` уникален.
- Статус движется строго `RECEIVED → PROCESSING → DONE | FAILED` (без обратных переходов).
- `result_text` и `audio_object_key` заполнены тогда и только тогда, когда `status='DONE'`;
  `failure_reason` — только при `FAILED`.
- Фото-объект в MinIO удалён после перехода в терминальный статус (AC-1-13); в `RECEIVED`
  `photo_object_key` всегда заполнен.
- Пользователь не может прочитать чужое обращение (все чтения через `owner_id`).

## 6. Интеграции → SDC 5 RESILIENCE

| Вызов | Connect TO | Read TO | Retry | Circuit breaker | Fallback |
|---|---|---|---|---|---|
| Yandex Vision (OCR для `DOCUMENT` / captioning для `SCENE`) | 2 с | 25 с | 2× exponential backoff (1 с, 2 с) | `visionCB` 50 % / 10 с | `FAILED` + `failure_reason=PROVIDER_UNAVAILABLE` |
| Yandex SpeechKit (TTS, русские neural-голоса) | 2 с | 25 с | 2× exponential backoff (1 с, 2 с) | `ttsCB` 50 % / 10 с | `FAILED` + `failure_reason=PROVIDER_UNAVAILABLE` |
| MinIO (object storage: put/get/delete, presign) | 2 с | 10 с | 2× | `minioCB` 50 % / 10 с | POST → `502`; worker → `FAILED` + `STORAGE_ERROR` |

Retry только для идемпотентных операций (HTTP `GET`/`PUT`/`DELETE` объекта по фиксированному
ключу, повтор безопасен). После 2 повторных попыток — `FAILED` с причиной `PROVIDER_UNAVAILABLE`
(AC-1-12). Пользователю — терминальный статус, не зависание.

Брокер: не вводится (см. раздел 12). Кэш: не вводится (аудио не кэшируется, spec NOT-DOING).

## 7. Идемпотентность и сбои → SDC 2

- Ключ дедупликации: `(owner_id, idempotency_key)` · хранение: `UNIQUE` constraint · окно:
  всё время жизни строки (повторный POST возвращает тот же `requestId`, AC-1-05).
- Повтор POST: `INSERT … ON CONFLICT (owner_id, idempotency_key) DO NOTHING`; если вставки не
  было — удаляется orphan-фото (если загружено) и возвращается существующая строка.
- Worker at-least-once: переход `RECEIVED→PROCESSING` атомарен (`FOR UPDATE SKIP LOCKED`),
  повторная обработка детерминирована (перезапись результата той же строкой); застрявшие
  `PROCESSING` возвращает reaper (см. ниже).

| Сбой | Что происходит | Восстановление |
|---|---|---|
| timeout / 5xx AI-провайдера | 2 retry с backoff → `FAILED` `PROVIDER_UNAVAILABLE` | пользователь видит статус; повтор возможен новым POST |
| падение worker после `PROCESSING` | строка остаётся `PROCESSING` | reaper: `PROCESSING` со `updated_at` старше порога (90 с) → обратно `RECEIVED` для повторной обработки |
| дубль обработки (2 экземпляра) | `SKIP LOCKED` исключает гонку; повторная обработка перезаписывает результат | терминальный статус достигается один раз |
| смена порядка сообщений | N/A — очереди/брокера нет, порядок задан `created_at` | — |
| падение после загрузки фото, до INSERT | orphan-объект в `photos` | reaper/cleanup: объекты `photos/*` старше порога удаляются |

## 8. Sequence

**Основной поток (DOCUMENT):**

1. Angular: съёмка/выбор файла, выбор `mode`, `POST /requests` (multipart + JWT +
   `Idempotency-Key`).
2. Backend: валидация (формат/размер/режим) → `requestId=UUID` → `PUT photos/{requestId}.{ext}`.
3. `INSERT requests (RECEIVED, photo_object_key)` `ON CONFLICT DO NOTHING` → `202 {requestId, RECEIVED}`.
4. Worker: `SELECT … FOR UPDATE SKIP LOCKED WHERE status='RECEIVED'` → `PROCESSING`.
5. Yandex Vision OCR → `result_text`; Yandex SpeechKit TTS → `PUT audio/{requestId}.mp3`.
6. `UPDATE … SET status='DONE', result_text, audio_object_key` → удалить `photos/{requestId}`.
7. Frontend: polling `GET /requests/{id}` до `DONE` → показать текст, `audioUrl` (presigned)
   → воспроизведение `<audio>`.

**Путь ошибки (AI недоступен):**

1. Worker: OCR/TTS → timeout/5xx → retry 2× (backoff) → всё неуспешно.
2. `UPDATE … SET status='FAILED', failure_reason='PROVIDER_UNAVAILABLE'` → удалить фото.
3. Frontend polling видит `FAILED` + `failureReason`, предлагает повторить (новый POST).

## 9. Безопасность

- **Auth**: Spring Security OAuth2 Resource Server, JWT access token (подпись проверяется
  ресурс-сервером). Роль единственная — `USER` (claim `roles`/`scope`). Выпуск токена — вне
  scope `vision-assistant` (dev-эндпоинт `/auth/token` для стенда или внешний IdP; открытый
  вопрос, spec раздел 13).
- **Object-level доступ**: `owner_id = sub(JWT)`; чужое/несуществующее → `404` (факт
  существования не раскрывается, AC-1-07).
- **Данные**: фото и аудио шифруются at rest (MinIO server-side encryption); по сети — TLS
  1.2+ (termination на edge/ingress); байты фото/аудио в логи не пишутся.
- **Валидация ввода**: тип MIME + magic bytes (JPEG/PNG), лимит 4 МБ / 8 МП, `mode` из
  закрытого множества; отказ → `400`/`413`/`415` (AC-1-03, AC-1-04).
- **Аудит**: события «создание» и «чтение» обращения пишутся в структурированный аудит-лог
  (JSON: `who` = `sub`, `when`, `action`, `resourceId`, `result`). Tamper-evident хранилище
  аудита — вне MVP (раздел 12).

## 10. Производительность и Prod AC → SDC 7

- Нагрузка: 10 RPS на создание, масштаб до 50 RPS без перепроектирования; фото ≤ 4 МБ, ≤ 8 МП.
- Узкие места:
  - внешний AI-вызов — доминирует в p95 цикла → асинхронность + таймауты + CB (раздел 6);
  - выборка worker → частичный индекс `idx_requests_status_received` + `LIMIT` батча (нет
    скана по всей таблице);
  - история → `idx_requests_owner_created` (нет сортировки в памяти по большой таблице).
- Индексы: `idx_requests_owner_created (owner_id, created_at DESC)`,
  `idx_requests_status_received (status) WHERE status='RECEIVED'`.
- Цели: `POST /requests` p95 ≤ 500 мс / p99 ≤ 1 с; `GET /requests/{id}` p95 ≤ 200 мс;
  `GET /requests` p95 ≤ 300 мс; полный цикл p95 ≤ 30 с / p99 ≤ 45 с; error rate < 1 %;
  10→50 RPS. Метрики (Micrometer): `vision.cycle.p95` (timer: `created_at` → терминальный
  статус), `request.status.failed` (counter, tag `failure_reason`), `vision.request.created`
  (counter), `vision.provider.latency` / `vision.provider.retries` (timer/counter).
- Kill switch: `features.vision-assistant` (env `FEATURES_VISION_ASSISTANT`, по умолчанию
  `true`); при `false` контроллер отдаёт `503`, worker отключён — фича выключается без деплоя.

## 11. ADR

### ADR-1: Топология — единый backend-сервис (модульный монолит)
- Контекст: первая фича, один домен, выбор между микросервисами и одним сервисом.
- Решение: один Spring Boot 3 сервис `vision-assistant` (пакеты по фиче), готовый к
  модульному монолиту.
- Альтернативы: 1) микросервисы (`requests-api` + `worker` + `ai-gateway`) — отклонено:
  преждевременно, операционные издержки (2+ деплоя, сетевые контракты) без выгоды на
  1 фиче; 2) расширить существующее — нечего (greenfield).
- Последствия: плюсы — один деплой/одна БД, простота; минусы — границы модулей дисциплинируют
  только конвенцией; риски — при росте возможен сплит worker в отдельный сервис (не блокер).

### ADR-2: Внешний AI-провайдер — Yandex Cloud (Vision + SpeechKit)
- Контекст: нужны OCR документов, описание сцены и TTS; MVP — русский язык.
- Решение: Yandex Cloud — Vision API (OCR для `DOCUMENT`, captioning для `SCENE`) +
  SpeechKit (TTS, русские neural-голоса). Один вендор, один SDK/auth, лучшая русская OCR/TTS.
- Альтернативы: 1) расширить существующее / self-hosted (Tesseract OCR + локальный TTS) —
  отклонено: качество OCR на фото низкое, нет описания сцены, GPU-инфраструктура, не
  укладывается в p95 30 с без масштабирования; 2) Google Cloud Vision + Cloud TTS — хорошая
  OCR/TTS, но русские neural-голоса слабее и доступность/стоимость в RU-контексте хуже.
- Последствия: плюсы — русское качество, один контракт; минусы — vendor lock-in; риски —
  смена вендора локализована в одном клиенте `YandexVisionClient`/`YandexSpeechKitClient`.

### ADR-3: Хранилище фото/аудио — MinIO (S3-совместимое)
- Контекст: фото временно, аудио постоянно; бэкенд stateless ≥ 2 экземпляра; фото не в БД
  (AC-1-13); шифрование at rest.
- Решение: MinIO (S3-совместимый object storage), buckets `photos` + `audio`; фото удаляется
  после терминального статуса; аудио отдаётся presigned URL (TTL 15 мин).
- Альтернативы: 1) PostgreSQL `bytea` (расширить существующее) — отклонено: AC-1-13 запрещает
  байты фото в БД, большие объекты раздувают БД/бэкапы; 2) локальная ФС на volume — отклонено:
  нарушает stateless при ≥ 2 экземплярах.
- Последствия: плюсы — stateless, стандартный S3-API (замена на AWS S3 без кода); минусы —
  ещё один сервис в окружении; риски — SSE at rest обязателен (включён).

### ADR-4: БД — PostgreSQL + Flyway
- Контекст: реляционные данные (статусная машина, unique constraint, транзакции); стек
  зафиксирован с PO (PostgreSQL + Flyway).
- Решение: PostgreSQL (схема `public`), миграции Flyway, `V1__create_requests.sql`.
- Альтернативы: 1) MySQL — отклонено (PO зафиксировал PostgreSQL); 2) документная БД (MongoDB)
  — отклонено: нужны транзакции, `UNIQUE`, `FOR UPDATE SKIP LOCKED` для worker.
- Последствия: плюсы — ACID, транзакционная статусная машина; минусы — горизонтальный
  масштаб через stateless-инстансы + одна БД (достаточно на 10→50 RPS); риски — рост БД —
  будущий сплит на чтение/запись (не блокер).

## 12. Не вводим → SDC 8

- **Брокер/очередь (Kafka/RabbitMQ)** — асинхронность закрыта БД-очередью
  (`FOR UPDATE SKIP LOCKED`); на 10→50 RPS брокер не нужен.
- **Интерфейс провайдера «на будущее»** — конкретные клиенты `YandexVisionClient` /
  `YandexSpeechKitClient` (интерфейс с одной реализацией запрещён).
- **Таблицу `results` отдельно** — обращение 1:1 с результатом, JOIN не нужен.
- **Таблицу `users`/регистрацию** — MVP только валидация JWT (роль `USER`), `owner_id` из
  `sub`; выпуск токена вне scope.
- **Кэширование аудио / CDN** — аудио отдаётся presigned URL, кэш не вводится (spec NOT-DOING).
- **Автоопределение режима, перевод текста, multi-tenancy, нативный клиент, экспорт истории** —
  spec NOT-DOING.
- **Tamper-evident аудит-хранилище** — аудит ведётся структурированным логом; выделенное
  append-only хранилище — tech-debt на будущее.

## 13. Kubernetes

N/A: `infra.orchestration` в профиле не `kubernetes` — локальный стенд и staging поднимаются
через Docker Compose (spec, раздел 9). Манифесты k8s не создаются; при переходе на k8s
требуется доп. секция.

## 14. High-load

N/A: `quality.high_load: false` в профиле. Требования масштаба (stateless ≥ 2, 10→50 RPS)
покрыты в разделе 10; отдельный high-load чек-лист не применяется.

## 15. Frontend (Angular)

Приложение: Angular PWA (standalone components, signals, typed reactive forms, `inject()`).

Структура (`<app>/src/app/`):

| Артефакт | core / shared / features | Назначение |
|---|---|---|
| `core/interceptors/auth.interceptor.ts` | core | добавляет `Authorization: Bearer`; обрабатывает `401` |
| `core/services/auth.service.ts` | core | хранение токена, `isAuthenticated` |
| `core/services/api.service.ts` | core | `HttpClient`-обёртка с типизированными DTO (`RequestDto`/`ResultDto`) |
| `shared/components/status-badge/` | shared | бейдж статуса (`RECEIVED/PROCESSING/DONE/FAILED`) |
| `shared/components/audio-player/` | shared | `<audio>`-плеер с `data-testid="audio-play"` |
| `shared/components/error-state/`, `spinner/` | shared | состояние загрузки/ошибки |
| `features/requests/create-request/` | features/requests | съёмка (`getUserMedia`) / выбор файла, выбор режима, отправка |
| `features/requests/request-list/` | features/requests | история (пагинация, статусы) |
| `features/requests/request-detail/` | features/requests | результат: текст + воспроизведение озвучки |

Маршруты (lazy loading через `loadComponent`/`loadChildren`):

- `/requests` → `request-list` (история)
- `/requests/new` → `create-request`
- `/requests/:id` → `request-detail`
- `/login` → получение/ввод токена (MVP-заглушка под внешний IdP)

Состояние: формы и статус — `signal`/`computed`; данные с бэка — `toSignal()`; polling статуса
`GET /requests/{id}` каждые 2 с до терминального статуса — RxJS `interval` + `switchMap` с
`takeUntilDestroyed()`. Потоки RxJS — подписки только с `takeUntilDestroyed()` или `async` pipe.
Шаблоны: `@for (… track item.id)`, `@if`; без вызовов функций в шаблоне (computed/pure pipe).
PWA: манифест + базовый service worker (офлайн-оболочка); кэш аудио — нет (spec NOT-DOING).
`data-testid` на интерактивных элементах для e2e.

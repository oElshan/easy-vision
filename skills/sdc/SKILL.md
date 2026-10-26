---
name: sdc
description: System Design Checklist - 8 слотов (ownership, идемпотентность, согласованность, миграция, устойчивость, blast radius, prod AC, anti-scope), когда блок обязателен, как его заполняет analyst, детализирует architect и сверяет с кодом java-reviewer.
---

# System Design Checklist (SDC)

Восемь системных вопросов, на которые фича отвечает до начала разработки. analyst заполняет
слоты в spec, architect детализирует их в arch doc, java-reviewer сверяет код со слотами,
tester проверяет слоты 2, 5, 8 на стенде, devops проверяет 4 и 7 на staging.

## Когда обязателен

- любая фича;
- изменение модуля shared-contracts (`topology.shared_contracts`);
- новая миграция БД (слоты 4 и 7 обязательно);
- новый внешний вызов (слот 5);
- новая очередь, exchange, topic;
- новый микросервис или Deployment в k8s (все слоты).

Не обязателен для задачи размера xs или s внутри одного модуля без новых таблиц, очередей и
внешних вызовов. Тогда в spec или карточке одна строка: `SDC: N/A — <причина>`.

## Слоты

1. **OWNERSHIP.** Кто владеет данными (модуль или сервис из `docs/arch/ownership-map.md`),
   кто consumers, tenant-scope, если проект multi-tenant. Запись в чужие данные только через
   API или событие владельца.
2. **IDEMPOTENCY.** Что при повторной доставке, повторном вызове, retry клиента. Где живёт
   ключ (заголовок `Idempotency-Key`, unique constraint, таблица). Как consumer at-least-once
   дедуплицирует.
3. **CONSISTENCY.** Граница транзакции (какие таблицы в одной транзакции). Между сервисами:
   saga, outbox или eventual с reconciliation. Какой инвариант защищаем и что его нарушает.
4. **MIGRATION.** Forward и rollback, либо `forward-only: <причина>`. Совместимость старого
   кода с новой схемой на время rolling update (expand → migrate → contract). Объём данных,
   время, блокировки; для high-load онлайн-миграция без lock таблицы.
5. **RESILIENCE.** Для каждого внешнего вызова: connect и read timeout, retry (сколько,
   backoff, только для идемпотентных операций), circuit breaker (имя, пороги), fallback.
   Degrade или fail fast при недоступности зависимости.
6. **BLAST RADIUS.** Затронутые DTO, события, API и их consumers с доказательством (`grep`).
   Breaking или non-breaking; для breaking стратегия версий (`/v2`, новая версия схемы,
   dual-write). Затронутые k8s-объекты.
7. **PROD AC.** Числа: p95, p99, error rate, RPS. Как проверить идемпотентность в проде.
   Kill switch: флаг, env или ключ ConfigMap, который выключает фичу без деплоя. Имена
   метрик.
8. **NOT-DOING.** Что сознательно не делаем и почему; что станет tech-debt.

## Формат в spec

```markdown
| # | Слот | Решение |
|---|---|---|
| 1 | OWNERSHIP | order-service владеет `orders`; consumers: billing (событие OrderCreated) |
| 2 | IDEMPOTENCY | Idempotency-Key + unique(orders.external_id) |
| 3 | CONSISTENCY | orders + order_items в одной транзакции; billing через outbox |
| 4 | MIGRATION | V42__add_orders.sql + rollback в arch doc; expand-contract |
| 5 | RESILIENCE | billing-client: timeout 2s/5s, retry 2× backoff, CB billingCB 50%/10s |
| 6 | BLAST RADIUS | OrderDto: +optional поле → non-breaking; consumers: grep-список в arch doc |
| 7 | PROD AC | p95 < 200 ms, err < 0.1 %, 300 RPS; kill switch features.order-v2 |
| 8 | NOT-DOING | без пересчёта истории заказов |
```

Пустой слот недопустим: решение или `N/A: <причина>`.

## Сверка кода (java-reviewer)

Только для слотов не `N/A` и только по AC текущей story:

| ID | Слот | HIGH, если |
|---|---|---|
| SDC-01 | 2 | заявлен ключ дедупликации, а в коде нет unique constraint, `ON CONFLICT`, проверки `existsBy…` или SETNX |
| SDC-02 | 3 | заявлена одна транзакция, а `@Transactional` нет, разорван или вызывается через `this.`; заявлен outbox, а событие шлётся напрямую внутри транзакции |
| SDC-03 | 4 | обещан rollback, а его нет; `forward-only` без причины |
| SDC-04 | 5 | на внешнем вызове нет заявленных timeout, retry или circuit breaker |
| SDC-05 | 6 | заявлено non-breaking, а поле DTO в shared-contracts удалено, переименовано или сменило тип |
| SDC-06 | 7 | MEDIUM: на критичном endpoint нет метрики (Micrometer Timer/Counter) |
| SDC-07 | 8 | в коде появилось то, что слот NOT-DOING исключил |

# Ownership Map: единый владелец на ресурс

Каждая таблица / очередь / endpoint / topic имеет ровно ОДНОГО владельца (модуль или
микросервис). Cross-module запись идёт только через API/event владельца. Прямой доступ к
чужой таблице запрещён (проверяет java-reviewer по слоту OWNERSHIP в SDC).

Обновляют analyst и architect при появлении нового ресурса в spec. Стартовое состояние
заполняет `/setup` сканом миграций, контроллеров и listeners.

## Таблицы БД

| Таблица | Owner (модуль/сервис) | Пишут | Читают |
|---|---|---|---|

## Очереди / Topics / Exchanges

| Ресурс | Producer (owner) | Consumers |
|---|---|---|

## API Endpoints (внешние контракты)

| Endpoint | Owner | Consumers |
|---|---|---|

## Shared contracts (DTO / Events)

| Контракт | Owner | Consumers (grep evidence при изменении) |
|---|---|---|

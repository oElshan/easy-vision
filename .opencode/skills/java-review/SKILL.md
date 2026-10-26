---
name: java-review
description: Чек-листы java-reviewer - дерево severity и проверки SEC, CA, SOL, CC, ERR, SPR, JPA, VT, API, MS, LOAD, CONC, OBS, MJ, TEST с ID для находок, ключевые grep по delta. Сверка с SDC в skill sdc, миграции в skill db-migration.
---

# Java + System Design review

Только delta. Каждая находка: ID чека, `file:line`, severity, confidence (high, medium, low),
рекомендация. False positive допустим, если указан confidence.

Сначала определи версии: Java (`sourceCompatibility`, `java.version`) и Spring Boot. Java
ниже 16: чеки MJ = N/A. Java ниже 21: чеки VT = N/A. Находку про несоответствие стеку ставь,
только если код отходит от фреймворка, заявленного в `stack.backend.framework`.

## Severity (первое совпадение сверху)

1. **CRITICAL**: уязвимость (SEC), потеря данных, гонка в hot path, проглоченное исключение
   в критичном потоке.
2. **HIGH**: нарушение Dependency Rule, LSP, N+1, внешний вызов без timeout/retry/
   идемпотентности, изменение схемы без миграции, исчерпание пула, нет backpressure, нет
   outbox при «БД + событие», HIGH из блока SDC.
3. **MEDIUM**: SOLID/DRY/KISS без немедленного риска, observability, современные идиомы,
   тесты, stampede кэша, GC pressure вне hot path.
4. **LOW**: стиль, именование, text blocks, контрактные тесты.

## SEC: маркеры безопасности (полный аудит у security-reviewer)
- SEC-01 [CRIT] SQL-конкатенация `"SELECT" + var` в `@Query`/JdbcTemplate → параметры
- SEC-02 [CRIT] литерал секрета `password=`, `apiKey=`, `token=` → vault/env
- SEC-03 [CRIT] PII, пароли, токены в логах → маскирование
- SEC-04 [CRIT] `@RequestBody` без `@Valid` → валидация
- SEC-05 [CRIT] `new File(userInput)`, `Paths.get(userInput)` без нормализации → whitelist
- SEC-06 [CRIT] `ObjectInputStream`, Jackson default typing → запрет десериализации недоверенного

## CA: Clean Architecture (infrastructure → application → domain)
- CA-01 [HIGH] domain импортирует infrastructure/web/persistence → порт
- CA-02 [HIGH] в domain `@Entity`, `@Component`, `jakarta.persistence.*`, `org.springframework.*` (если entity-as-domain не закреплён ADR)
- CA-03 [HIGH] application зависит от реализации, а не от порта
- CA-04 [MED] сервис-комбайн на весь домен вместо сфокусированных use case
- CA-05 [HIGH] entity или domain-объект отдаётся из контроллера → DTO/record
- CA-06 [HIGH] циклы между пакетами или модулями
- CA-07 [MED] значимое архитектурное решение без ADR
- CA-08 [MED] command и query смешаны в одном методе со сложной логикой

## SOL: SOLID
- SOL-S [HIGH god / MED] класс > 500 строк, метод > 50, сервис «валидирует + маппит + БД + нотификации»
- SOL-O [MED] switch/if-каскад по типу в бизнес-логике → полиморфизм, sealed + pattern matching
- SOL-L [HIGH] override бросает `UnsupportedOperationException`, ужесточает предусловия
- SOL-I [MED] толстый интерфейс, пустые реализации
- SOL-D [MED] `new RestTemplate()`, `new ObjectMapper()` в бизнес-логике

## CC: Clean Code, DRY, KISS
- CC-01 [MED] дублирование > 5 строк между классами
- CC-02 [MED] magic numbers и strings → константа, enum
- CC-03 [MED] интерфейс с одной реализацией «на будущее», generic-фабрика без второго случая
- CC-04 [LOW] имена `data`, `temp`, `process()`, `Manager`, `Util`
- CC-05 [LOW] комментарий «что» вместо «почему», закомментированный код
- CC-06 [MED] метод > 50 строк, вложенность > 3, > 4 параметров
- CC-07 [LOW] boolean-флаги в параметрах
- CC-08 [HIGH] побочный эффект в геттере или проверке

## ERR: обработка ошибок
- ERR-01 [CRIT в критичном потоке / HIGH] пустой `catch`
- ERR-02 [HIGH] `Optional.get()` без проверки
- ERR-03 [HIGH] нет `@RestControllerAdvice`, обработка размазана
- ERR-04 [MED] везде `RuntimeException` вместо доменной иерархии
- ERR-05 [HIGH] неверный HTTP-статус (200 вместо 404, нет 201/204)
- ERR-06 [MED] rethrow без cause
- ERR-07 [MED] исключения как управление потоком

## SPR: Spring
- SPR-01 [MED] `@Autowired` на поле → конструктор
- SPR-02 [HIGH] бизнес-логика в контроллере
- SPR-03 [HIGH] `@Transactional` на контроллере или repository
- SPR-04 [MED] чтение без `@Transactional(readOnly = true)`
- SPR-05 [MED] россыпь `@Value` → `@ConfigurationProperties`
- SPR-06 [HIGH] self-invocation `@Transactional`, `@Cacheable`, `@Async` через `this.`
- SPR-07 [HIGH] состояние в singleton-бине
- SPR-08 [MED] `@SpringBootTest` для unit

## JPA: БД
- JPA-01 [HIGH] N+1: `FetchType.EAGER` на коллекциях, lazy-доступ в цикле
- JPA-02 [HIGH] `List<T>` без `Pageable`
- JPA-03 [HIGH] мутирующий `@Query` без `@Modifying` и `@Transactional`
- JPA-04 [MED] `CascadeType.ALL` + `orphanRemoval` без явной причины
- JPA-05 [HIGH] долгий запрос без timeout
- JPA-06 [HIGH] новая entity или изменение схемы без миграции; правка применённой миграции = CRITICAL
- JPA-07 [MED] значимая entity без аудита (`@CreatedDate`, `@LastModifiedDate`)
- JPA-08 [LOW] многострочный native SQL без text block
- JPA-09 [HIGH] новый запрос по неиндексированной колонке
- JPA-10 [MED] OFFSET-пагинация большой таблицы в hot path → keyset
- JPA-11 [HIGH] массовый `saveAll` без `hibernate.jdbc.batch_size` и `order_inserts`

## VT: Java 21+ и современный Spring Boot
- VT-01 [HIGH] virtual threads + `synchronized` вокруг блокирующего I/O → `ReentrantLock`
- VT-02 [HIGH] `InheritableThreadLocal` с virtual threads → `ScopedValue`
- VT-03 [HIGH] нет `server.shutdown=graceful` и `spring.lifecycle.timeout-per-shutdown-phase`
- VT-04 [HIGH для native / N/A] reflection без регистрации
- VT-05 [HIGH] Spring Batch без явного chunk

## API: system design
- API-01 [HIGH] breaking change без `/v2` или версии в заголовке
- API-02 [HIGH] повторяемый POST (платёж, генерация) без `Idempotency-Key`
- API-03 [MED] нестандартная пагинация, дорогой `total`
- API-04 [HIGH в критичной части / MED] анемичный домен, неверные границы агрегатов
- API-05 [MED] новые endpoints без OpenAPI-аннотаций
- API-06 [HIGH] удаление, переименование, смена типа поля существующего DTO

## MS: микросервисы (только `topology.type: microservices`)
- MS-01 [HIGH] межсервисный вызов без клиента с timeout, retry, circuit breaker
- MS-02 [CRIT] доступ к БД чужого сервиса (сверь с `docs/arch/ownership-map.md`)
- MS-03 [HIGH] изменение контракта события или API без обратной совместимости
- MS-04 [MED] нет distributed tracing между сервисами

## LOAD: высокая нагрузка (вопрос к каждому изменению: что при ×100 RPS и ×100 данных?)
При `quality.high_load: true` пограничные находки поднимай на уровень.
- LOAD-01 [HIGH] `CompletableFuture`/`@Async` без своего bounded executor
- LOAD-02 [HIGH] долгий метод в `@Scheduled`
- LOAD-03 [HIGH] HTTP-клиент без connect/read timeout
- LOAD-04 [HIGH] внешний вызов без retry с backoff и jitter; retry неидемпотентной операции
- LOAD-05 [HIGH] критичная интеграция без circuit breaker
- LOAD-06 [HIGH] один пул на все интеграции → bulkhead
- LOAD-07 [HIGH] HTTP или долгий I/O внутри `@Transactional`
- LOAD-08 [HIGH] `@Cacheable` без TTL/evict, кэш без max size
- LOAD-09 [MED] горячий ключ без защиты от stampede
- LOAD-10 [HIGH] consumer без ограничения concurrency/prefetch, producer без обработки отказа брокера
- LOAD-11 [MED] дорогой публичный endpoint без rate limit, если SLA требует
- LOAD-12 [HIGH] запись в БД и событие без общей гарантии → outbox
- LOAD-13 [MED] большие аллокации на запрос в hot path
- LOAD-14 [HIGH] глобальный lock в hot path
- LOAD-15 [HIGH] загрузка всей таблицы в память для подсчёта или фильтра
- LOAD-16 [HIGH] критичная секция между инстансами на локальном `synchronized` → ShedLock, Redis lock, optimistic locking

## CONC: конкурентность
- CONC-01 [HIGH в hot path / MED] изменяемые поля в `@Service`/`@Component`
- CONC-02 [HIGH] общий `HashMap`/`ArrayList` между потоками
- CONC-03 [MED] double-checked locking без `volatile`
- CONC-04 [MED] `SimpleDateFormat` в полях singleton
- CONC-05 [MED] check-then-act → `computeIfAbsent`, `putIfAbsent`

## OBS: наблюдаемость
- OBS-01 [MED] `log.info` на каждый запрос; ERROR без stack trace
- OBS-02 [MED] нет traceId/requestId в MDC
- OBS-03 [MED] критичный endpoint без Micrometer timer/counter
- OBS-04 [MED] зависимость без HealthIndicator
- OBS-05 [MED] межсервисные вызовы без propagation контекста
- OBS-06 [LOW] конкатенация в логах вместо `log.info("x={}", x)`

## MJ: современная Java (16+)
- MJ-01 [MED] DTO как POJO с getters/setters → record
- MJ-02 [MED] закрытая иерархия без sealed
- MJ-03 [LOW] `instanceof` + cast → pattern matching
- MJ-04 [LOW] многострочный JSON/SQL без text block
- MJ-05 [LOW] switch statement с мутацией → switch expression
- MJ-06 [MED] сервис возвращает null → `Optional` или исключение
- MJ-07 [LOW] конкатенация в цикле, raw types

## TEST: тесты
- TEST-01 [MED] `@SpringBootTest` для unit-логики
- TEST-02 [MED] `Thread.sleep()` → Awaitility
- TEST-03 [LOW] имя `testFindUser` → `should_return_404_when_user_not_found`
- TEST-04 [MED] критичный метод из delta без теста
- TEST-05 [LOW] внешняя интеграция без контрактного теста
- TEST-06 [MED] только happy path

## Grep по delta

```bash
git diff --name-only <base>...<story> -- '*.java' '*.kt' '*.sql'
grep -rnE '"\s*\+\s*\w' <files> | grep -iE 'select|where|insert|update|delete'      # SEC-01
grep -rnE '(password|apiKey|token|secret)\s*=\s*"' <files>                         # SEC-02
grep -rn 'Runtime.exec\|ProcessBuilder\|ObjectInputStream\|new File(\|Paths.get(' <files>
grep -rn '@Autowired\|FetchType.EAGER\|@Query' <files>
grep -rn '\.get()' <files> | grep -i 'optional\|findBy'                            # ERR-02
for f in $(grep -ln 'RestTemplate\|WebClient\|RestClient\|FeignClient' <files>); do
  grep -l '@Transactional' "$f" && echo "LOAD-07? $f"; done
grep -rn '@Async\|supplyAsync\|synchronized\|@Cacheable' <files> | head -20
grep -rnE 'class .*(DTO|Dto|Response|Request)\b' <files> | grep -v 'record '      # MJ-01
```

## Принципы

- Без догматизма: абстракция оправдана при второй реализации, внешнем ресурсе или границе
  слоя. Иначе KISS (CC-03) важнее DIP. При конфликте принципов назови компромисс.
- Сборку и тесты не запускай: их запускает оркестратор на G4.

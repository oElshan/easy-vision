---
name: java-spring
description: Как developer пишет backend на Java/Spring Boot - порядок реализации, тесты (JUnit 5, Mockito, Testcontainers), покрытие JaCoCo, безопасность, Spring и JPA паттерны, self-review чек-лист перед коммитом с примерами BAD/GOOD.
---

# Backend: Java / Spring Boot

Для другого фреймворка (Quarkus, Micronaut) применяй эквиваленты тех же правил.

## Порядок реализации

1. Данные: entity → миграция (skill `db-migration`) → repository.
2. Сервис: бизнес-логика, валидация, ошибки.
3. API: controller → DTO (record) → маппинг.
4. Тесты: unit на сервисы, интеграционные на endpoint и БД, если в проекте есть базовый
   класс (`Glob("**/*IntegrationTest*.java")`, `BaseIntegrationTest`).

Перед новым классом прочитай 2–3 существующих того же типа и повтори их структуру.

## Тесты

- Unit: JUnit 5 + Mockito, по тесту на ветку логики, имена
  `should_<результат>_when_<условие>`.
- Интеграционные: Testcontainers с реальными зависимостями из `stack.data`, REST-assured или
  MockMvc. Запуск `commands.backend.integration_test`.
- Ожидание асинхронности через Awaitility, не `Thread.sleep`.
- Unit-логику не поднимают через `@SpringBootTest`: `@WebMvcTest`, `@DataJpaTest` или чистый
  Mockito.
- Покрытие изменённых классов ≥ `quality.unit_coverage_min`: проверь отчётом
  `commands.backend.coverage`, цифру бери из отчёта.

## Безопасность (CRITICAL, блокирует коммит)

- Нет секретов в коде: ключи, пароли, токены, connection strings только из env или vault.
- Только параметризованные запросы:
  ```java
  // BAD
  String jpql = "SELECT u FROM User u WHERE u.email = '" + email + "'";
  // GOOD
  String jpql = "SELECT u FROM User u WHERE u.email = :email";
  query.setParameter("email", email);
  ```
- Каждый новый endpoint защищён: `@PreAuthorize` с конкретной ролью или правило в
  `SecurityConfig`. `isAuthenticated()` там, где нужна роль, недостаточно.
- `@RequestBody` всегда с `@Valid` и Bean Validation на DTO.
- Пути к файлам не строятся из пользовательского ввода без нормализации и whitelist.
- В логах нет паролей, токенов, PII.
- Multi-tenant (если есть в `AGENTS.md`): запросы к tenant-данным только через механизм
  изоляции проекта.

## Spring и JPA (HIGH)

- Constructor injection, не `@Autowired` на полях.
- Бизнес-логика в сервисе, не в контроллере. `@Transactional` только на сервисном слое,
  чтение `@Transactional(readOnly = true)`.
- Вызов `@Transactional`, `@Cacheable`, `@Async` через `this.` не работает (прокси): вынеси
  в другой бин.
- Списки на внешнем API только с `Pageable` или `LIMIT`.
- Без N+1:
  ```java
  // BAD
  orderRepository.findAll().forEach(o -> o.getCustomer().getPayments());
  // GOOD
  @Query("SELECT o FROM Order o JOIN FETCH o.customer c LEFT JOIN FETCH c.payments")
  List<Order> findAllWithCustomers();
  ```
- Внешний HTTP-вызов: connect и read timeout, retry только для идемпотентных операций,
  circuit breaker для критичных интеграций (Resilience4j). HTTP внутри `@Transactional`
  запрещён.
- Запись в БД и публикация события под одной гарантией: outbox.
- Ошибки через `@RestControllerAdvice`, клиенту без stack trace. Коды: 201 на создание,
  204 на удаление, 404 на отсутствие, 400 на валидацию.
- `Optional.get()` без проверки запрещён: `orElseThrow`.
- Entity наружу не отдаётся: DTO или record на границе.
- Новый запрос по колонке без индекса: индекс в миграции.

## Качество кода (HIGH)

- Метод до ~50 строк, вложенность до 3, early return вместо пирамиды `if`.
- Нет пустых `catch`, `System.out.println`, закомментированного кода, неиспользуемых импортов.
- Магические числа и строки в константы или enum.
- TODO только со ссылкой на задачу: `// TODO(#123): …`.
- Без новых библиотек, которых нет в arch doc. Если добавил, напиши в отчёте.

## Самопроверка перед коммитом

Пройди `git diff --staged` по разделам выше. Отмечай только то, в чём уверен больше чем на
80 %. CRITICAL и HIGH исправь. MEDIUM и LOW исправь или впиши в `Not-tested` коммита.

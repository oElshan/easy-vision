---
name: e2e-playwright
description: Процедура e2e-tester - отдельный проект e2e-tests на Java (Playwright, JUnit 5, AssertJ, Page Object Model), структура пакетов, base-классы, только data-testid селекторы, тестовые пользователи из env, генерация тестов из TC в docs/qa, компиляция и запуск.
---

# E2E: Java + Playwright

## Структура `e2e-tests/`

Build tool тот же, что у backend (`stack.backend.build_tool`), если проекта ещё нет. Если
есть, следуй его build-файлу. Версии зависимостей: актуальные стабильные с Maven Central.

```
e2e-tests/
  build.gradle | pom.xml
  src/main/java/<group>/e2e/
    config/E2EConfig.java       BASE_URL, API_URL, HEADLESS, таймауты из system properties и env
    config/TestUsers.java       логины по ролям из env (E2E_<ROLE>_EMAIL, E2E_PASSWORD)
    pages/BasePage.java         navigate, waitFor, isVisible, getText
    pages/<Page>Page.java       по Page Object на страницу
    utils/ApiClient.java        setup и teardown данных через API
  src/test/java/<group>/e2e/scenarios/
    BaseE2ETest.java            Playwright и Browser на класс, Context и Page на тест, loginAs(role)
    feature<F>/<Feature>E2ETest.java
```

Зависимости: `com.microsoft.playwright:playwright`, `org.junit.jupiter:junit-jupiter`,
`org.assertj:assertj-core`, HTTP-клиент для ApiClient, Jackson. `test { useJUnitPlatform() }`,
`BASE_URL`, `API_URL`, `HEADLESS` из env с дефолтами на порты из профиля.

## Правила

- Один TC из `docs/qa/<F>-<slug>.md` → один `@Test` с `@DisplayName("TC-<F>-NNN: …")` и
  Javadoc со ссылкой на AC.
- Селекторы только `[data-testid='…']`, они живут в Page Object как константы. В тестах
  селекторов нет.
- Методы Page Object называются по действию: `clickCreate()`, `fillTitle(String)`,
  `getErrorText()`. Логики в Page Object нет.
- Пароли и токены только из env, в коде их нет.
- Данные для теста создаются через API в `@BeforeEach` и удаляются в `@AfterEach`.
- Структура теста: Given (подготовка, навигация) → When (действия) → Then (AssertJ).

## Пример

```java
@DisplayName("Feature 12: Экспорт заказов")
class OrderExportE2ETest extends BaseE2ETest {
    private OrdersPage orders;

    @BeforeEach
    void setUp() {
        loginAs(TestUsers.MANAGER_EMAIL, TestUsers.PASSWORD);
        orders = new OrdersPage(page);
    }

    /** TC-12-001 · AC-12-01 */
    @Test
    @DisplayName("TC-12-001: менеджер выгружает заказы в CSV")
    void managerExportsOrders() {
        orders.navigate("/orders");
        var download = orders.clickExport();
        assertThat(download.suggestedFilename()).endsWith(".csv");
    }
}
```

## Проверка

```bash
cd e2e-tests && ./gradlew compileTestJava        # Maven: ./mvnw test-compile
E2E_BASE_URL=<url frontend> <commands.e2e.run>     # если стенд поднят
```
Не компилируется: исправь до коммита. Стенда нет: «compiled OK, run skipped: нет окружения».

В QA-документе отметь у автоматизированных TC: `E2E: E2E-<F>-NNN`.

---
name: project-setup
description: Подключение проекта к агентной команде (/setup) - детект стека по файлам сборки, заполнение .agents/profile.md, создание AGENTS.md, docs/arch/budget.md и ownership-map.md, .gitignore, выбор режима доски local или github. Раздел «Детект» нужен любому агенту, если профиль не заполнен.
---

# Подключение проекта

Шаблоны лежат рядом: `templates/profile.md`, `templates/AGENTS.md`, `templates/budget.md`,
`templates/ownership-map.md`.

## Шаги `/setup`

1. **Проверки.** Это git-репозиторий (`git rev-parse --show-toplevel`), дерево чистое.
   Определи default branch: `git symbolic-ref --short refs/remotes/origin/HEAD` (без
   `origin/`), иначе текущая ветка; подтверди у человека и переключись на неё
   (`git switch <default>`). Если `.agents/profile.md` уже есть и `initialized: true`,
   покажи текущие значения и обновляй только то, что человек попросил.
2. **Детект** по таблице ниже. Команды бери из реальных файлов: wrapper (`./gradlew`,
   `./mvnw`) раньше глобальных, npm-скрипты из `package.json`, а не угаданные.
3. **Покрытие.** Гейт G4 читает CSV JaCoCo. Проверь, что отчёт CSV включён:
   - Gradle: в `jacocoTestReport { reports { csv.required = true } }`;
   - Maven: `jacoco-maven-plugin` с goal `report` (CSV `target/site/jacoco/jacoco.csv`
     создаётся по умолчанию).
   Не включён: предложи человеку правку build-файла, сам её не делай.
4. **Профиль.** Скопируй `templates/profile.md` в `.agents/profile.md` и заполни. Покажи
   человеку итог одной таблицей (стек, команды, стенд, деплой, пороги) и спроси:
   - режим доски: `local` или `github`;
   - утверждение spec человеком (`gates.spec_approval`): `human` (по умолчанию) или `auto`;
   - пороги покрытия, если 95/50 не подходят.
   После подтверждения поставь `initialized: true`, `updated`.
5. **AGENTS.md.** Если файла нет, создай по `templates/AGENTS.md`: модули из структуры
   каталогов, домен из сущностей и README, роли из security-конфига. Есть `CLAUDE.md`:
   возьми знание оттуда. Есть `AGENTS.md`: допиши только раздел «Агентная команда».
6. **Карты.** `docs/arch/budget.md` и `docs/arch/ownership-map.md` из шаблонов, стартовое
   состояние сканом:
   - таблицы: имена из миграций (`CREATE TABLE`) или `@Table`/`@Entity`;
   - очереди и topics: `@KafkaListener`, `@RabbitListener`, конфиги брокера;
   - endpoints: `@RequestMapping`/`@GetMapping`… по контроллерам (префиксы);
   - внешние вызовы: `RestClient`, `WebClient`, `FeignClient`, `RestTemplate`.
7. **.gitignore.** Добавь строки, если их нет:
   ```
   .agents/board/
   .agents/cache/
   ```
8. **Доска.** `mkdir -p .agents/board`. Режим `github`: выполни настройку из skill
   `github-sync`.
9. **Коммит** на default branch: `.agents/profile.md`, `AGENTS.md`, `docs/arch/*`,
   `.gitignore`: `chore: подключение агентной команды`.
10. **Итог** человеку: что найдено, что не удалось определить, следующий шаг
    (`/feature <описание>`).

## Детект

| Признак | Вывод |
|---|---|
| `gradlew`, `build.gradle(.kts)` | gradle: `./gradlew build -x test`, `./gradlew test`, покрытие `./gradlew test jacocoTestReport` |
| `mvnw`, `pom.xml` | maven: `./mvnw -q package -DskipTests`, `./mvnw test`, покрытие `./mvnw verify` |
| `settings.gradle` с несколькими `include`, несколько `pom.xml` | многомодульный: тесты модуля `./gradlew :{service}:test` или `./mvnw -pl {service} test` |
| `spring-boot` в build-файле | framework spring-boot, health `/actuator/health` |
| `sourceCompatibility`, `java.version`, `<java.version>` | версия Java |
| `angular.json` | frontend angular: `ng build`, `ng test --watch=false` |
| `package.json` с `react` | frontend react: скрипты из `package.json` |
| `pnpm-lock.yaml` / `yarn.lock` / `package-lock.json` | pnpm / yarn / npm |
| `src/main/resources/db/migration` | flyway |
| `liquibase` в build-файле, `db/changelog` | liquibase |
| `docker-compose*.yml`, `compose*.yml` | local_env: `docker compose up -d` / `docker compose down` |
| `Chart.yaml`, `helm/` | k8s helm |
| `kustomization.yaml` | k8s kustomize |
| манифесты с `kind: Deployment` | kubernetes |
| `.github/workflows/` | ci github-actions |
| `Testcontainers` в зависимостях, задача `integrationTest` | `integration_test` |

Агент, который нашёл незаполненный профиль во время работы, определяет команды по этой
таблице, работает по ним и пишет в отчёте: «профиль не заполнен, запустите /setup».

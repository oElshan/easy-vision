---
name: security-review
description: Чек-лист security-reviewer по delta - OWASP Top 10 для Spring Boot, Angular и React, JWT и RBAC, object-level доступ, зависимости (OWASP dependency-check, npm audit), k8s-секреты, AI-риски (prompt injection, утечки RAG), классификация severity.
---

# Security review

Только delta ветки story. Стек и пути приложений из `.agents/profile.md`, модель auth и роли
из `AGENTS.md`.

## OWASP Top 10

| # | Категория | Что проверить |
|---|---|---|
| A01 | Broken Access Control | `@PreAuthorize`/`@Secured` или правило в `SecurityConfig` на каждом новом endpoint; роль конкретная; пользователь получает только свои объекты (object-level); multi-tenant: фильтр по tenant на каждом запросе |
| A02 | Cryptographic Failures | нет секретов в коде и конфиге; пароли через bcrypt/argon2; ключ JWT не в коде |
| A03 | Injection | SQL только параметризованный; нет `Runtime.exec`/`ProcessBuilder` с вводом; Angular: нет `innerHTML` и `bypassSecurityTrust*` с вводом; React: нет `dangerouslySetInnerHTML` с вводом, нет `javascript:` в href, нет `eval` |
| A04 | Insecure Design | обход бизнес-правил, нет rate limit на auth-endpoints |
| A05 | Security Misconfiguration | CORS не `*` с credentials; security headers; actuator и debug-endpoints закрыты |
| A06 | Vulnerable Components | новые зависимости проверены (ниже) |
| A07 | Auth Failures | проверка подписи, срока, issuer и audience JWT; ротация refresh; logout инвалидирует |
| A08 | Data Integrity | CSRF там, где cookie-auth; безопасная десериализация; `@Valid` на входе |
| A09 | Logging | нет PII и токенов в логах; события безопасности логируются; клиенту нет stack trace |
| A10 | SSRF | URL для серверных HTTP-вызовов не берётся из ввода без whitelist |

## JWT и RBAC

- Матрица ролей из `AGENTS.md` или arch doc совпадает с аннотациями в коде.
- Токен на фронте не хранится в `localStorage` (HttpOnly cookie или память), если проект не
  закрепил иное.

## Зависимости

Если в delta изменились `build.gradle(.kts)`, `pom.xml`, `package.json`:
```bash
./gradlew dependencyCheckAnalyze || ./mvnw org.owasp:dependency-check-maven:check   # если плагин подключён
cd <stack.frontend[].path> && npm audit --json | head -50                          # pnpm audit / yarn audit
```
Плагин не подключён: напиши `dependency-check не настроен`, это не находка.

## Kubernetes и конфиги (если в delta манифесты, Dockerfile, CI)

- Секреты не в ConfigMap и не в `env` открытым текстом → Secret или внешний менеджер.
- Креды не печатаются в entrypoint-скриптах и аргументах.
- Контейнер не от root, если в проекте так принято.
- Новый сервис без NetworkPolicy: обычно MEDIUM.

## AI-компоненты (только если в проекте есть LLM-вызовы)

- Пользовательский ввод не склеивается с системным промптом.
- Результаты RAG фильтруются по правам пользователя (и tenant) до отправки в модель.
- Ответ модели санитизируется перед выводом.
- Задан лимит токенов на вызов.

## Severity

| Severity | Примеры | Вердикт |
|---|---|---|
| CRITICAL | обход auth, SQL injection, RCE, утечка данных, секрет в коде | FAIL |
| HIGH | broken access control, XSS, обход CSRF, prompt injection | FAIL |
| MEDIUM | нет валидации, подробные ошибки, слабая криптография | PASS, tech-debt |
| LOW | best practices | PASS |

## Grep по delta

```bash
grep -rnE '(password|secret|apiKey|token)\s*[=:]\s*"[^"]+"' <files>
grep -rn 'permitAll\|csrf().disable\|csrf(AbstractHttpConfigurer::disable)\|allowedOrigins("\*")' <files>
grep -rn 'bypassSecurityTrust\|dangerouslySetInnerHTML\|innerHTML\|eval(\|javascript:' <files>
grep -rn 'localStorage.setItem' <files> | grep -i 'token\|jwt'
grep -rn '@RequestBody' <files> | grep -v '@Valid'
```

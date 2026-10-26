---
name: frontend-review
description: Чек-листы frontend-reviewer - дерево severity и проверки ARCH, TS, ANG, RCT, ST, PERF, A11Y, API, SEC, TEST с ID для находок, ключевые grep по frontend-delta. Правила структуры в skills react-fsd и angular.
---

# Frontend review

Только frontend-delta. К файлу применяй чеки его фреймворка: ANG только к Angular, RCT только
к React. Находка: ID, `file:line`, severity, confidence, рекомендация.

## Severity

1. **CRITICAL**: XSS-маркер (SEC), потеря пользовательских данных, сломанный критичный сценарий.
2. **HIGH**: утечка подписок, нарушение правил hooks, два источника правды, запрос без
   обработки ошибок, обход границ архитектуры, нет пагинации или виртуализации при
   `quality.high_load`.
3. **MEDIUM**: типизация, нормализация state, a11y, debounce, недостающие тесты.
4. **LOW**: стиль, именование, микрооптимизации.

## ARCH: структура и границы
- ARCH-01 [HIGH] React: импорт против слоёв FSD; Angular: feature лезет во внутренности другой области
- ARCH-02 [HIGH] циклические зависимости между модулями или областями
- ARCH-03 [MED] бизнес-логика в компоненте вместо сервиса, hook или store
- ARCH-04 [MED] доменная логика в `shared/`

## TS: TypeScript
- TS-01 [HIGH] `any`, `as any` без обоснования
- TS-02 [MED] loading/success/error набором boolean → discriminated union
- TS-03 [MED] non-null `!` без гарантии
- TS-04 [MED] дублируемые типы ответов API → один типизированный API-слой
- TS-05 [LOW] `@ts-ignore` без причины

## ANG: Angular
- ANG-01 [MED] новый компонент не standalone
- ANG-02 [HIGH] тяжёлый компонент без OnPush и без signals
- ANG-03 [HIGH] `*ngFor` без `trackBy`, `@for` без `track`
- ANG-04 [HIGH] `subscribe()` без `takeUntilDestroyed()` или `async`
- ANG-05 [MED] маршрут области без lazy loading
- ANG-06 [MED] вычисления и вызовы функций в шаблоне

## RCT: React
- RCT-01 [HIGH] условный вызов hooks, hooks в циклах
- RCT-02 [HIGH] неверные или подавленные зависимости `useEffect`
- RCT-03 [MED] нет мемоизации там, где частота рендера её оправдывает, или мемоизация «на всякий случай»
- RCT-04 [HIGH] `key={index}` в изменяемом списке
- RCT-05 [HIGH] server-state через `useState` + `useEffect`
- RCT-06 [MED] лишние ререндеры: литералы в props горячего компонента

## ST: состояние
- ST-01 [HIGH] один факт в двух местах (store, локальный state, URL)
- ST-02 [MED] ненормализованные коллекции
- ST-03 [HIGH] server-state продублирован в глобальном store
- ST-04 [MED] прямая мутация состояния

## PERF: производительность
- PERF-01 [HIGH] крупный маршрут в основном бандле
- PERF-02 [MED] изображения без размеров и lazy loading
- PERF-03 [HIGH] дорогое вычисление на каждый рендер
- PERF-04 [MED] ввод или поиск без debounce
- PERF-05 [HIGH при high_load, иначе MED] длинный список без виртуализации
- PERF-06 [HIGH при high_load] списочный запрос без пагинации

## A11Y: доступность
- A11Y-01 [MED] `div` с onClick вместо `button`, нет доступа с клавиатуры
- A11Y-02 [MED] input без label или `aria-label`
- A11Y-03 [MED] модалка без управления фокусом
- A11Y-04 [LOW] явные проблемы контраста

## API: запросы
- API-01 [HIGH] нет loading, error или empty у запроса
- API-02 [MED] устаревшие запросы не отменяются
- API-03 [MED] retry на неидемпотентной мутации или нет retry там, где UX требует

## SEC: маркеры (полный аудит у security-reviewer)
- SEC-01 [CRIT] Angular: `bypassSecurityTrust*`, `innerHTML` с вводом
- SEC-02 [CRIT] React: `dangerouslySetInnerHTML` с несанитизированным вводом
- SEC-03 [CRIT] `javascript:` в href/src, `eval`, `new Function`
- SEC-04 [HIGH] токен в `localStorage`

## TEST: тесты
- TEST-01 [MED] новый компонент, hook или сервис без unit-теста
- TEST-02 [MED] интерактивный элемент без `data-testid`
- TEST-03 [LOW] тест на детали реализации

## Grep по delta

```bash
git diff --name-only <base>...<story> -- '*.ts' '*.tsx' '*.jsx' '*.html' '*.scss' '*.css'
grep -rnE ':\s*any\b|as any' <files> | head -20
grep -rn 'bypassSecurityTrust\|dangerouslySetInnerHTML\|innerHTML\|eval(\|javascript:' <files>
grep -rn 'localStorage.setItem' <files> | grep -i 'token\|jwt'
grep -rn '\.subscribe(' <files> | grep -v takeUntilDestroyed
grep -rnE '\*ngFor|@for' <files> | grep -v 'trackBy\|track '
grep -rn 'key={index}\|key={i}' <files>
grep -rn 'eslint-disable.*exhaustive-deps' <files>
```

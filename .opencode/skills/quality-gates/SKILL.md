---
name: quality-gates
description: Гейты G1–G8 и гейт фичи для оркестратора - чек-листы, команды проверки (сборка, тесты, покрытие из JaCoCo, миграции, свежесть вердиктов по SHA, дрейф spec), подготовка веток, шаблоны задач для субагентов, обработка FAIL.
---

# Гейты

Гейт проверяет оркестратор сам. Каждый пункт подтверждается выводом команды, который ты
получил в этой сессии, или строкой отчёта субагента с SHA. Итог гейта: `PASS`, `FAIL`
(вернуть на доработку) или `PARK` (нужен человек). Результат записывается в карточку
(skill `board`, раздел «Запись»).

Обозначения: `<F>` фича, `<S>` story или задача, `<base>` ветка, от которой создана ветка
story (feature-ветка или default branch), `<story>` ветка story, `<default>` =
`project.default_branch` из профиля.

## Общие проверки

**Чистое дерево** перед любым `git switch` и в G4:
```bash
git status --porcelain
```
Пусто: можно. Иначе остановись и перечисли файлы.

**Свежесть вердикта** `<VERDICT>@<sha>` относительно HEAD ветки story:
```bash
git diff --name-only <sha> <story> | grep -vE '(^|/)docs/qa/|(^|/)src/(test|integrationTest|it)/'
```
Пустой вывод: вердикт действителен. Иначе вердикт устарел, стадию нужно пройти заново.

**Дрейф spec** у story фичи:
```bash
git show <feature-branch>:<spec path> | git hash-object --stdin
```
Должно совпасть с `spec_sha` карточки фичи. Не совпало: `PARK human`, причина «spec изменена
после утверждения, нужен /approve <F>».

**Применённые миграции** (пути из `paths.migrations` профиля, например `**/db/migration/**`):
```bash
git diff --name-status --diff-filter=MDR <base>...<story> -- ':(glob)**/db/migration/**' ':(glob)**/db/changelog/**'
```
Любая строка вывода = FAIL: файл существовал в базовой ветке и был изменён, удалён или
переименован.

**Покрытие изменённых классов** (JaCoCo CSV). Команда `commands.backend.coverage`, отчёты
`commands.backend.coverage_report` (путь или glob):
```bash
mkdir -p .agents/cache && touch .agents/cache/coverage-start
<commands.backend.coverage>
find <coverage_report> -newer .agents/cache/coverage-start     # отчёт обязан быть свежим
git diff --name-only --diff-filter=AM <base>...<story> -- '*.java' '*.kt' \
  | grep -E 'src/main/(java|kotlin)/' \
  | sed -E 's#.*src/main/(java|kotlin)/##; s#\.(java|kt)$##' > .agents/cache/changed-classes.txt
awk -F, '
  FILENAME == ARGV[1] { want[$0] = 1; next }
  FNR > 1 {
    cls = $3; sub(/\..*/, "", cls)
    pkg = $2; gsub(/\./, "/", pkg)
    key = (pkg == "" ? cls : pkg "/" cls)
    if (key in want) { missed += $8; covered += $9 }
  }
  END { if (missed + covered == 0) print "n/a"; else printf "%.1f\n", 100 * covered / (missed + covered) }
' .agents/cache/changed-classes.txt <coverage_report>
```
Колонки JaCoCo CSV: `$2` PACKAGE, `$3` CLASS, `$8` LINE_MISSED, `$9` LINE_COVERED. Если
`commands.backend.coverage` пуст, запиши `coverage: n/a` и предупреждение в журнал: гейт это
не блокирует, но в PR уйдёт строка «покрытие не измерено».

**TODO без ссылки** в добавленных строках:
```bash
git diff <base>...<story> | grep -E '^\+.*\b(TODO|FIXME)\b' | grep -vE '(TODO|FIXME)\((#?[0-9]+|[A-Z]+-[0-9]+)\)'
```

## G1 · Sprint → Analysis (feature)

Проверки:
- в карточке есть бизнес-ценность и хотя бы высокоуровневые критерии, задан `priority`;
- нет дубля среди карточек и spec (`grep -il` по ключевым словам);
- дерево чистое.

Действия: `git switch <default>`, затем `git switch -c feature/<F>-<slug>` (если ветка уже
есть: `git switch feature/<F>-<slug>`). В карточку: `branch`, `spec: docs/specs/<F>-<slug>.md`,
`arch: docs/arch/<F>-<slug>.md`, `qa_doc: docs/qa/<F>-<slug>.md`, статус Analysis.
Нет бизнес-ценности: FAIL, карточка остаётся в Sprint, спроси человека.

## Analysis

1. analyst (шаблон ниже) → отчёт `ANALYST DONE`.
2. architect → отчёт `ARCHITECT DONE`. `BLOCKED: spec неполная` → снова analyst с этим списком.
3. G2.

## G2 · Analysis → Ready for Dev (feature)

Проверки на feature-ветке:
- spec содержит обязательные разделы шаблона (skill `write-spec`);
- AC: заголовки `### AC-<F>-NN`, ID уникальны, у каждого Given/When/Then и строка
  `Проверка: stand | integration | unit`:
  ```bash
  grep -oE '^### AC-<F>-[0-9]+' <spec> | sort | uniq -d          # дубли: должно быть пусто
  grep -cE '^### AC-<F>-[0-9]+' <spec>; grep -c '^Проверка:' <spec> # числа равны
  ```
- SDC: восемь слотов, в каждом решение или `N/A: <причина>`; либо весь блок
  `SDC: N/A — <причина>`, если фича под это подпадает (skill `sdc`);
- «Декомпозиция»: каждый AC фичи покрыт хотя бы одной story, у каждой story есть AC,
  размер xs, s или m, зависимости ссылаются на существующие ключи и не образуют цикл;
- arch doc есть и содержит разделы шаблона (skill `design-change`);
- новый модуль, сервис, очередь или внешняя интеграция при пороге из `budget.md`: в arch doc
  есть ADR; новые таблицы, очереди, endpoints внесены в `ownership-map.md`;
- изменения закоммичены, дерево чистое.

FAIL: снова analyst или architect с перечнем замечаний. После `quality.fix_loop_max` неудачных
G2 подряд: `PARK attempts`.

PASS:
- `gates.spec_approval: human` (по умолчанию): `parked: human`, причина «ждёт /approve <F>».
  Человеку выведи: путь к spec, число AC и stories, ADR, открытые вопросы analyst.
- `gates.spec_approval: auto`: сразу выполни шаги утверждения ниже.

## Утверждение (`/approve <F>` или auto)

**Первое утверждение**, фича в Analysis:

1. Проверь, что последний G2 у фичи PASS.
2. `spec_sha` = `git show <feature-branch>:<spec> | git hash-object --stdin`, `approved` =
   сегодняшняя дата, сними парковку.
3. Создай карточки stories по таблице «Декомпозиция» (skill `board`).
4. Фича → Ready for Dev.

**Повторное утверждение**, фича в Ready for Dev или In Development, а хэш spec на
feature-ветке не совпадает со `spec_sha` (spec изменили после утверждения):

1. Проверь изменённую spec по чек-листу G2, без analyst и architect. FAIL: откажи и перечисли
   замечания, spec правит человек.
2. Обнови `spec_sha` и `approved`.
3. Сверь таблицу «Декомпозиция» с карточками фичи по `spec_key`:
   - строка без карточки: создай story в Ready for Dev;
   - у карточки не в Done обнови `ac`, `deps`, `priority`, `size` из таблицы. Если у story в
     Code Review или Testing изменились AC, сбрось её вердикты `review` и `qa` и верни её в
     In Development;
   - карточку, чьей строки в таблице больше нет, и карточку в Done с изменёнными AC не
     трогай, перечисли человеку. Изменённые AC готовых stories проверит регрессия в гейте
     фичи.
4. Сними со stories фичи парковку «spec изменена после утверждения». Статус фичи не меняй.
5. В журнал фичи: `повторно утверждена · spec_sha <старый> → <новый>`.

## G3 · Ready for Dev или Sprint → In Development

Проверки:
- нет другой активной story (In Development, Code Review или Testing без парковки);
- все `deps` в Done;
- у story фичи: фича утверждена, дрейфа spec нет, каждый AC из `ac` есть в spec;
- у самостоятельной задачи: в теле карточки есть AC с ID и строкой «Проверка»;
- дерево чистое.

Действия:
- `git switch <base>` и сборка базы `commands.backend.build` (и frontend build затронутых
  приложений). Сломана: `PARK human`, причина «база не собирается», вывод в журнал.
- `git switch -c story/<S>-<slug>`. Если ветка уже есть, это возобновление (раздел ниже).
- В карточку `branch`, статус In Development. Если родительская фича в Ready for Dev, переведи
  её в In Development.
- Запусти developer.

## Возобновление story

Story после парковки или после ожидания, пока шла другая story, могла отстать от базы.

1. `git switch <story>`, затем:
   ```bash
   git merge-base --is-ancestor <base> <story> && echo up-to-date
   ```
2. Отстала: `git merge --no-ff <base> -m "merge(<S>): update from <base>"`.
   - Конфликт: merge не прерывай, статус In Development, запусти developer с задачей
     «разреши конфликты» (шаблон developer, последняя строка).
   - Без конфликта: вердикты ревью и QA устарели. Статус In Development и сразу G4 без
     developer: если G4 PASS, story идёт в Code Review заново.
3. Не отстала: продолжай с текущего статуса.

## G4 · In Development → Code Review

Developer вернул `BLOCKED`: `PARK human` с его причиной.

Проверки на ветке story:
- дерево чистое, есть коммиты `git log --oneline <base>..<story>`;
- применённые миграции не тронуты;
- сборка и lint: `commands.backend.build`, `commands.backend.lint` (если задан), для
  затронутых frontend-приложений `build` и `lint`;
- тесты: `commands.backend.test` (или `test_one_service` по каждому затронутому сервису),
  frontend `test` затронутых приложений; все SUCCESS;
- покрытие изменённых классов ≥ `quality.unit_coverage_min` (hotfix:
  `quality.hotfix_coverage_min`);
- нет TODO/FIXME без ссылки.

PASS: `coverage: <pct>@<sha>`, статус Code Review, сброс `review` в пустые значения.
FAIL: In Development, `attempts + 1`, замечания с выводом команд.

## Code Review (ревьюеры)

Delta: `git diff --name-only <base>...<story>`.
- security-reviewer: всегда;
- java-reviewer: есть файлы `*.java *.kt *.sql *.gradle *.kts pom.xml` или `src/main/resources/**`;
- frontend-reviewer: есть `*.ts *.tsx *.jsx *.html *.scss *.css` в путях `stack.frontend[].path`.

Неприменимому ревьюеру сразу пиши `n/a`. Применимых запускай параллельно, несколькими
вызовами task в одном сообщении. `BLOCKED: SHA изменился` → запусти этого ревьюера заново.

## G5 · Code Review → Testing

Проверки:
- `review.security` = `PASS@<sha>`;
- `review.java` и `review.frontend` = `PASS@<sha>`, `WARNING@<sha>` или `n/a`;
- все SHA свежие;
- `git status --porcelain` пуст и `git rev-parse HEAD` равен `<sha>` ревью.

PASS: статус Testing. MEDIUM/LOW находки запиши в журнал как tech-debt: они попадут в PR.
FAIL: In Development, `attempts + 1`, каждая CRITICAL/HIGH находка в замечания.
Дерево грязное или HEAD сдвинулся после ревью: `PARK human`, в причину вывод
`git status --porcelain`. Ревьюерам запрещён `edit`, но перенаправление `>` в shell OpenCode
не блокирует, поэтому изменение файлов ловит этот пункт. Откат решает человек.

## G6 · Testing → merge → Ready to Deploy или Done

Проверки:
- tester вернул `VERDICT: PASS`, его SHA равен HEAD ветки story;
- в отчёте `AUTOTESTS ... SUCCESS` и `BUILD ... SUCCESS`;
- вердикты ревью всё ещё свежие, `coverage` записан;
- QA-документ закоммичен.

`FAIL`: In Development, `attempts + 1`, в замечания каждый проваленный AC с repro, expected,
actual.
`NOT_VERIFIED`: `PARK human`, причина из отчёта (что нужно для стенда). После `/unpark`
story снова идёт в Testing.

PASS:
- story фичи: `git switch <feature-branch>`, `git merge --no-ff <story> -m "merge(<S>): <title>"`.
  Конфликт: `git merge --abort`, `PARK conflict`. После мёржа `git branch -d <story>`.
- самостоятельная задача: не мёржится, PR откроется после Done.
- `qa: PASS@<sha>`. Если в профиле есть `commands.deploy.staging` и стадия deploy не на паузе:
  Ready to Deploy. Иначе Done.

## G7 · Ready to Deploy → Deploy

Пакет: все карточки в Ready to Deploy одной ветки (feature-ветка или ветка задачи).

Проверки:
- у каждой карточки пакета `qa: PASS@<sha>`, и этот коммит входит в ветку деплоя:
  `git merge-base --is-ancestor <sha> <branch>` (ветка story после мёржа уже удалена,
  поэтому проверяется вхождение, а не свежесть);
- на HEAD ветки зелёные `commands.backend.build` и `commands.backend.test` (один прогон на пакет);
- дерево чистое.

PASS: карточки пакета → Deploy, запусти devops с веткой и SHA.

## G8 · Deploy → Done

Проверки по отчёту devops:
- `VERDICT: PASS`, деплой и раскатка OK, health каждого сервиса OK, smoke OK, логи чистые;
- репетиция отката выполнена, либо `skipped` с причиной: нет `commands.deploy.rollback` или
  SDC #4 = forward-only;
- kill switch проверен, если заявлен в SDC #7.

PASS: каждой карточке пакета `deploy: staging@<sha>`, статус Done.
FAIL: карточки остаются в Deploy, `PARK human`, причина и вывод devops в журнал.

## Гейт фичи · все stories в Done

Проверки:
- все карточки с `parent: <F>` в Done;
- на HEAD feature-ветки `commands.backend.build` и `commands.backend.test` зелёные;
- tester в режиме `regression`: все AC фичи на HEAD feature-ветки.

Регрессия упала: на каждый сломанный AC создай баг в Sprint (skill `board`), фича остаётся
In Development. Tester вернул FAIL, потому что не поднялся стенд: баги не создавай, `PARK
human` с выдержкой логов из отчёта. Отдельной story, которую можно вернуть на доработку, у
фичи здесь нет.

PASS: draft PR (ниже). PR создан: фича → Done, поле `pr`. PR не создан (нет `gh` или remote,
человек отклонил push): фича остаётся In Development, `PARK human`, причина «PR не создан:
<почему>», в журнал `гейт фичи PASS@<sha>`. После `/unpark`, если HEAD feature-ветки всё ещё
`<sha>`, повтори только раздел «Draft PR», без регрессии.

## Spike

Исследование без продуктового кода. Из Sprint: ветка `story/<id>-<slug>` от `<default>`,
статус In Development, developer с задачей «исследование по карточке, результат в
`docs/spikes/<id>-<slug>.md`: вопрос, варианты, вывод, рекомендация; продуктовый код не
менять». Проверка: документ закоммичен, `git diff --name-only <default>...<story>` содержит
только `docs/spikes/`. PASS: Done и draft PR. Иначе FAIL с замечаниями.

## Draft PR

Тело PR собери в `.agents/cache/pr-<id>.md`: ссылка на spec и arch doc, stories с вердиктами
ревью и QA, таблица AC → story → QA, tech-debt (MEDIUM/LOW находки), покрытие, статус staging,
план отката из spec, что не проверено.

```bash
gh auth status && git remote get-url origin
git push -u origin <branch>                      # подтверждает человек
gh pr create --draft --base <default> --head <branch> --title "<id>: <title>" --body-file .agents/cache/pr-<id>.md
```
Нет `gh` или remote: PR не создавай, выведи человеку путь к файлу с описанием и команды выше,
фичу запаркуй по разделу «Гейт фичи».
Самостоятельная задача после Done получает PR так же, из своей ветки в `<default>`.

## Шаблоны задач для субагентов

Подставь значения. Тексты карточек передавай путём к файлу, не пересказом.

**analyst**
```
Spec для фичи <F> «<title>».
Карточка: .agents/board/<F>.md
Ветка: feature/<F>-<slug> (активна). Spec: docs/specs/<F>-<slug>.md
Замечания G2: <список или «нет»>
Верни отчёт ANALYST.
```

**architect**
```
Arch doc для фичи <F> «<title>».
Spec: docs/specs/<F>-<slug>.md. Arch doc: docs/arch/<F>-<slug>.md
Ветка: feature/<F>-<slug> (активна). Замечания G2: <список или «нет»>
Верни отчёт ARCHITECT.
```

**developer**
```
Story <S> «<title>». Карточка: .agents/board/<S>.md
Ветка: <story> (активна), база: <base>. AC: <AC-…>
Spec: <путь или «AC в карточке»>. Arch doc: <путь или «нет»>
Попытка: <attempts + 1>. Замечания к исправлению: <раздел карточки или «нет»>
<если идёт merge с конфликтами: «Идёт merge <base> в <story>, конфликты: <файлы>. Разреши и закоммить.»>
Верни отчёт DEVELOPER.
```

**security-reviewer / java-reviewer / frontend-reviewer**
```
Ревью story <S> «<title>».
Ветка: <story>, база: <base>, SHA: <sha>
Diff: git diff <base>...<story>
Spec (для SDC): <путь или «нет»>. AC story: <AC-…>
Верни отчёт <SECURITY|JAVA|FRONTEND> REVIEW.
```

**tester**
```
QA story <S> «<title>».
Ветка: <story> (активна), SHA: <sha>. AC: <AC-…>
Где AC: <spec или карточка>. QA-документ: <docs/qa/…>
Режим: story | regression (все AC фичи на HEAD <feature-branch>)
Верни отчёт QA REPORT.
```

**devops**
```
Деплой на staging. Ветка: <branch> (активна), SHA: <sha>.
Карточки в пакете: <id, …>. SDC #4/#7 фичи: <spec или «нет»>.
Верни отчёт DEPLOY REPORT.
```

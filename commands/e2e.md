---
description: Написать E2E-тесты (Java Playwright) по тест-кейсам QA-документа фичи
agent: orchestrator
---
Напиши E2E-тесты для фичи $ARGUMENTS.

Загрузи skill `board`. У фичи должен быть QA-документ (`qa_doc` в карточке) с тест-кейсами.
Дерево должно быть чистым. Переключись на feature-ветку фичи и запусти e2e-tester с id фичи,
путём к QA-документу и веткой. Если он вернул список недостающих `data-testid`, предложи
создать по нему задачу (`type: tech-debt`, `component: [frontend]`) в Backlog.

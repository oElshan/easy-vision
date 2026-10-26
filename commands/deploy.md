---
description: Задеплоить на staging всё, что в Ready to Deploy - G7, devops, G8
agent: orchestrator
---
Деплой на staging. Аргументы: $ARGUMENTS

Загрузи skills `board` и `quality-gates`. Если в профиле нет `commands.deploy.staging`,
скажи, что стадии деплоя в проекте нет, и остановись. Аргумент `production` не выполняй:
prod вне автоматизации, объясни это.

Возьми пакет карточек из Ready to Deploy (или только перечисленные id), выполни G7, запусти
devops, проверь G8. Итог: что задеплоено, SHA, health, smoke, откат.

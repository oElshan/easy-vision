---
name: deploy-staging
description: Процедура devops для деплоя на staging - проверки манифестов для high-load, сборка артефакта, деплой командой профиля (helm, kustomize, kubectl, docker compose), rollout status, health, smoke, логи, репетиция отката и kill switch, немедленный откат при провале health.
---

# Деплой на staging

Команды только из `.agents/profile.md`. Production не трогается.

## 1. Перед деплоем

- HEAD равен SHA из задачи, дерево чистое.
- Если `quality.high_load: true`, проверь манифесты затронутых сервисов
  (`infra.k8s_manifests_path`): заданы requests и limits, readiness и liveness, replicas ≥ 2
  у критичных сервисов, HPA там, где он заявлен в arch doc. Нарушение: FAIL до деплоя.

## 2. Артефакт

Docker: `docker build` по Dockerfile сервиса или `docker compose build`, тег = короткий SHA.
Без Docker: `commands.backend.build`.

## 3. Деплой

```bash
<commands.deploy.staging>
```
Kubernetes: дождись раскатки каждого затронутого сервиса:
```bash
kubectl rollout status deployment/<service> -n <commands.deploy.namespace_staging> --timeout=180s
```
docker compose: `docker compose ps` и состояние healthy.

## 4. Проверка

```bash
curl -sf <url сервиса><topology.services[].health>            # каждый затронутый сервис
<commands.smoke.staging>
kubectl logs deployment/<service> -n <ns> --since=60s | grep -iE 'error|exception|fatal' || echo "чисто"
# compose: docker compose logs --since=60s <service> | grep -iE 'error|exception|fatal' || echo "чисто"
```

Health не OK: сразу `<commands.deploy.rollback>`, проверь health после отката, вердикт FAIL.

## 5. Репетиция отката

Если задан `commands.deploy.rollback` и SDC 4 не `forward-only`:
1. выполни откат;
2. проверь health и, если миграция обратима, состояние схемы;
3. верни новую версию повторным `<commands.deploy.staging>`;
4. снова health и smoke.

Нет команды отката или forward-only: `skipped` с этой причиной.

## 6. Kill switch

Если SDC 7 заявляет kill switch: переключи его (флаг, env, ключ ConfigMap), покажи эффект
(лог, ответ endpoint, метрика), верни обратно.

## 7. Отчёт

По формату из промпта devops: каждая строка с реальным выводом команды.

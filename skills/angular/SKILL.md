---
name: angular
description: Правила Angular-приложений для developer, architect и frontend-reviewer - структура core, shared, features, standalone components, signals, inject(), новый control flow с track, typed reactive forms, HttpClient с interceptors, отписка takeUntilDestroyed, self-review чек-лист.
---

# Angular

Путь приложения `stack.frontend[].path`, команды `commands.frontend[]` того же приложения.
FSD к Angular не применяется.

## Структура (`<path>/src/app/`)

```
core/              singleton-сервисы, HttpClient interceptors (auth, 401, ошибки), guards, конфиг
shared/            переиспользуемые standalone-компоненты, pipes, directives без домена
features/<area>/   маршрутные области с lazy loading (loadChildren / loadComponent)
```

Каждая область владеет своим поддеревом маршрутов и отдаёт наружу только публичные
сервисы и компоненты.

## Обязательно в новом коде

- Standalone components, без NgModule.
- Состояние: `signal`, `computed`, `effect`; данные с бэка можно через `toSignal()`.
- RxJS для потоков; подписки только с `takeUntilDestroyed()` или через `async` pipe.
- `@for (item of items; track item.id)` и `@if` вместо `*ngFor`/`*ngIf`; `track` обязателен.
- Typed Reactive Forms; template-driven формы на 5+ полей запрещены.
- `HttpClient` с типом `http.get<T>()`, interceptors из `core/`.
- DI через `inject()`.
- Тяжёлые компоненты: `ChangeDetectionStrategy.OnPush` или signals.

## Self-review

- нет утечек подписок;
- нет прямой работы с DOM (`document.getElementById`, `nativeElement` без нужды);
- нет `any` без причины;
- у async-операций есть обработка ошибок и состояние загрузки;
- в шаблонах нет вызовов функций и вычислений: computed signal или pure pipe;
- у интерактивных элементов `data-testid`.

```typescript
// BAD: утечка
ngOnInit() { this.service.getData().subscribe(d => this.data = d); }

// GOOD
private destroyRef = inject(DestroyRef);
ngOnInit() {
  this.service.getData().pipe(takeUntilDestroyed(this.destroyRef)).subscribe(d => this.data.set(d));
}
```

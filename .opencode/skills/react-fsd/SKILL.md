---
name: react-fsd
description: Правила React-приложений по Feature-Sliced Design для developer, architect и frontend-reviewer - 6 слоёв, импорты сверху вниз через public API, структура слайса, куда класть код, запреты, стек по умолчанию (TanStack Query, RHF + Zod), self-review чек-лист.
---

# React: Feature-Sliced Design

Путь приложения `stack.frontend[].path`, команды `commands.frontend[]` того же приложения.

## Слои (сверху вниз)

```
app/       инициализация: router, providers (QueryClient, Theme, Auth), стили, error boundary
pages/     страницы, которые монтируются в route
widgets/   крупные самостоятельные UI-блоки (Header, OrderList)
features/  пользовательские сценарии (AuthForm, OrderFilters)
entities/  бизнес-сущности (Order, Customer): model + api + базовые ui
shared/    ui-kit, api-клиент, interceptors, lib, config, types
```

## Импорты

- Только сверху вниз: `app → pages → widgets → features → entities → shared`.
- Слайсы одного слоя не импортируют друг друга. Нужна связка: поднимай в `features/` или
  `widgets/`. Связь entities допускается только через `@x` public API с ADR.
- Только через public API слайса (`index.ts`), не вглубь (`features/auth-form/ui/Form.tsx`).
- `eslint-plugin-boundaries` или `@feature-sliced/eslint-config`: упавший линт чинится
  структурой, правило не отключается.

## Структура слайса

```
<layer>/<slice>/
  ui/       компоненты
  model/    состояние: queries и mutations, store, селекторы
  api/      HTTP (только в shared/api и entities/<x>/api)
  lib/      внутренние утилиты
  config/   константы
  index.ts  public API
```

## Куда класть код

| Код | Место |
|---|---|
| axios instance, interceptors (JWT refresh, 401) | `shared/api` |
| обёртки UI-kit | `shared/ui/<name>` |
| утилиты, общие zod-схемы | `shared/lib` |
| DTO из OpenAPI | `shared/api/generated` или `shared/types` |
| сущность + её API + карточка | `entities/<name>` |
| сценарий (verb-noun) | `features/<name>` |
| композитный блок | `widgets/<name>` |
| страница | `pages/<name>` |
| router, providers | `app/` |

## Запреты (HIGH)

- Нет каталогов `core/`, `services/`, `common/`: их роль у `shared/`.
- Нет `fetch`/`axios` в `features/`, `widgets/`, `pages/`.
- Нет компонентов в `model/`, бизнес-состояния в `shared/`, роутинга вне `app/router`.
- Нет barrel-файлов, которые реэкспортируют чужой слайс.

## Стек по умолчанию (отход только с ADR)

TanStack Query для server-state, Zustand только для состояния, общего для несвязанных
компонентов, React Hook Form + Zod для форм от 5 полей, React Router, axios через
`shared/api/http.ts`, Vitest + Testing Library.

## Self-review

- каждый файл в правильном слое, импорты по правилам;
- данные с бэка в `useQuery`/`useMutation`, не в `useState` после `fetch`;
- нет `any` без причины, ответы типизированы;
- у async UI есть loading, error, empty;
- `useEffect` с подписками и таймерами возвращает cleanup, запросы отменяются;
- `key` в списках стабильный id, не индекс;
- у интерактивных элементов `data-testid` для e2e.

```tsx
// BAD: запрос и состояние прямо в feature
const [items, setItems] = useState([]);
useEffect(() => { axios.get('/api/orders').then(r => setItems(r.data)); }, []);

// GOOD
// entities/order/api/orders.ts
export const fetchOrders = () => http.get<OrderDto[]>('/api/orders').then(r => r.data);
// entities/order/model/use-orders.ts
export const useOrders = () => useQuery({ queryKey: ['orders'], queryFn: fetchOrders });
// features/order-list/ui/OrderList.tsx
const { data, isLoading, error } = useOrders();
```

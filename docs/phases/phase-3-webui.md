# Фаза 3: Web UI (единый интерфейс)

**Оценка:** 80–100 чел·часов  
**Предусловие:** [Фаза 1](phase-1-api-rbac.md) минимум, [Фаза 2](phase-2-engine.md) желательна  
**Зависимости:** Node.js 20+, React + Vite (или HTMX для упрощения)  
**Результат:** Пользователь заказывает ВМ в браузере; админ видит дашборд (mock capacity)

---

## Цель

Единое веб-приложение: после логина user видит «Мои ВМ», admin — расширенный интерфейс + дашборд ресурсов.

---

## Выбор стека

| Вариант | Плюсы | Минусы | Время |
|---------|-------|--------|-------|
| **React + Vite** | Масштабируемо, богатый UX | Больше кода | 80–100 ч |
| **HTMX + FastAPI templates** | Быстрый старт | Менее гибкий UI | 40–60 ч |

Ниже — React; для HTMX замените шаги 2–4 на Jinja2-шаблоны в `vmorch/api/templates/`.

---

## Шаг 1. CORS и static (2–4 ч)

### 1.1 FastAPI CORS

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:5173"],  # Vite dev
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)
```

### 1.2 Production

Собранный `web/dist` отдавайте через `StaticFiles` или nginx.

---

## Шаг 2. Инициализация frontend (4–6 ч)

```bash
cd vm_orchestrator
npm create vite@latest web -- --template react-ts
cd web
npm install
npm install react-router-dom @tanstack/react-query axios
```

Структура:

```text
web/src/
├── api/client.ts       # axios + token interceptor
├── auth/AuthContext.tsx
├── pages/
│   ├── Login.tsx
│   ├── MyInstances.tsx
│   ├── OrderVM.tsx
│   ├── InstanceDetail.tsx
│   └── admin/
│       ├── Dashboard.tsx
│       ├── AllInstances.tsx
│       ├── Users.tsx
│       └── Catalog.tsx
├── components/
│   ├── Layout.tsx
│   ├── InstanceTable.tsx
│   └── CapacityCards.tsx
└── App.tsx
```

---

## Шаг 3. Auth в UI (8–10 ч)

### 3.1 `api/client.ts`

```typescript
const api = axios.create({ baseURL: "http://localhost:8000/api/v1" });

api.interceptors.request.use((config) => {
  const token = localStorage.getItem("token");
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});
```

### 3.2 Login flow

1. Форма username/password → `POST /auth/login`
2. Сохранить token + role в localStorage
3. `AuthContext` предоставляет `user`, `isAdmin`, `logout`

### 3.3 Protected routes

```tsx
<Route path="/" element={<RequireAuth><MyInstances /></RequireAuth>} />
<Route path="/admin/*" element={<RequireAdmin><AdminLayout /></RequireAdmin>} />
```

---

## Шаг 4. Страницы пользователя (16–20 ч)

### 4.1 Мои ВМ (`/')

- Таблица: имя, profile, status, IP, TTL, кнопка «Сдать»
- Polling каждые 3 сек, пока есть `provisioning` / `configuring`
- Пустое состояние + кнопка «Заказать ВМ»

### 4.2 Заказ ВМ (`/order`)

Форма:

- name (валидация: DNS-safe)
- profile (dropdown из `GET /catalog/profiles` — добавьте endpoint в API)
- ttl (обязательно для user: 1d–30d)

Submit → `POST /instances` → redirect на детали.

### 4.3 Детали (`/instances/:name`)

- Timeline статусов
- IP, labels, owner (admin видит owner)
- Кнопка Release с confirm modal

### 4.4 Release

```typescript
await api.post(`/instances/${name}/release`);
// toast «ВМ сдаётся»
```

---

## Шаг 5. Admin UI (20–24 ч)

### 5.1 Все ВМ (`/admin/instances`)

- Таблица всех ВМ
- Фильтры: owner, status, profile
- Mass actions (опционально): release selected

### 5.2 Дашборд (`/admin/dashboard`)

Карточки из `GET /admin/capacity`:

```text
┌─────────────┐ ┌─────────────┐ ┌─────────────┐
│ vCPU        │ │ RAM         │ │ ВМ ready    │
│ 8 / 64      │ │ 32 / 256 GB │ │ 12          │
│ ████░░░░░░  │ │ ██░░░░░░░░  │ │             │
└─────────────┘ └─────────────┘ └─────────────┘
```

- Progress bar: used / total
- Кнопка Refresh
- Таблица «Top users by usage» (добавьте endpoint или считайте на клиенте из instances)

### 5.3 Пользователи (`/admin/users`)

- Список users + квоты
- Форма редактирования max_instances, max_vcpu, max_memory_mb

### 5.4 Каталог (`/admin/catalog`)

- Список profiles (read-only в MVP)
- Позже: CRUD

---

## Шаг 6. API дополнения (6–8 ч)

Добавьте в FastAPI, если ещё нет:

```text
GET /api/v1/catalog/profiles     — для dropdown (user: только allowed)
GET /api/v1/catalog/profiles/{name}
```

---

## Шаг 7. UX-полировка (8–12 ч)

- Toast-уведомления (успех / ошибка / квота)
- Loading skeletons
- Обработка 401 → redirect на login
- Обработка 409 QuotaExceeded с текстом ошибки
- Тёмная тема (опционально)
- i18n ru (опционально)

---

## Шаг 8. Сборка и деплой dev (4–6 ч)

```bash
# terminal 1
uvicorn vmorch.api.main:app --reload --port 8000

# terminal 2
cd web && npm run dev
```

Production build:

```bash
cd web && npm run build
# FastAPI mounts web/dist at /
```

---

## Критерии готовности

- [ ] Login/logout работает
- [ ] alice видит только свои ВМ
- [ ] admin видит `/admin/dashboard` с capacity
- [ ] user не может открыть `/admin/*` (redirect 403)
- [ ] Заказ ВМ из формы → появляется в списке
- [ ] Release с подтверждением работает
- [ ] Статус обновляется при provisioning (polling)
- [ ] Ошибка квоты показывается пользователю

---

## Частые ошибки

| Проблема | Решение |
|----------|---------|
| CORS error | Проверьте `allow_origins` и credentials |
| 401 после refresh | Token в localStorage, interceptor на axios |
| Admin route доступен user | Проверяйте role на клиенте И на сервере |

---

## Что дальше

[Фаза 4](phase-4-proxmox.md) — подключите Proxmox VE, заменив mock Terraform module.
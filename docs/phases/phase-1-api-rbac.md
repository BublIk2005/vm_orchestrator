# Фаза 1: БД + API + RBAC

**Оценка:** 60–80 чел·часов  
**Предусловие:** [Фаза 0](phase-0-mock.md) завершена  
**Зависимости:** FastAPI, python-jose (JWT), passlib  
**Результат:** REST API с ролями user/admin, mock-провайдер остаётся backend

---

## Цель

Перенести логику из CLI в API-сервер. Пользователь видит только свои ВМ; админ — все. Квоты проверяются перед заказом.

---

## Шаг 1. Зависимости (1 ч)

Добавьте в `pyproject.toml`:

```toml
dependencies = [
    # ... существующие
    "fastapi>=0.110",
    "uvicorn[standard]>=0.27",
    "python-jose[cryptography]>=3.3",
    "passlib[bcrypt]>=1.7",
    "python-multipart>=0.0.9",
]
```

```bash
pip install -e ".[dev]"
```

---

## Шаг 2. Расширение схемы БД (6–8 ч)

### 2.1 Новые таблицы — `vmorch/db/schema.sql`

```sql
CREATE TABLE users (
    id            TEXT PRIMARY KEY,
    username      TEXT UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    role          TEXT NOT NULL DEFAULT 'user',  -- user | admin
    created_at    TEXT NOT NULL
);

CREATE TABLE quotas (
    user_id       TEXT PRIMARY KEY REFERENCES users(id),
    max_instances INTEGER NOT NULL DEFAULT 3,
    max_vcpu      INTEGER NOT NULL DEFAULT 8,
    max_memory_mb INTEGER NOT NULL DEFAULT 16384
);

CREATE TABLE audit_log (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    actor       TEXT NOT NULL,
    action      TEXT NOT NULL,
    target      TEXT,
    details     TEXT,
    created_at  TEXT NOT NULL
);
```

### 2.2 Seed-данные

При `init_db()` создайте:

| username | password | role |
|----------|----------|------|
| alice | alice | user |
| admin | admin | admin |

**Важно:** только для dev; в production — сменить пароли.

---

## Шаг 3. Auth (10–12 ч)

### 3.1 `vmorch/api/auth.py`

- `hash_password` / `verify_password` (bcrypt)
- `create_access_token(user_id, role, expires=24h)` — JWT
- `get_current_user` — FastAPI dependency из `Authorization: Bearer`
- `require_admin` — dependency, 403 если role != admin

### 3.2 Endpoints

```text
POST /api/v1/auth/login     → { access_token, role, username }
GET  /api/v1/auth/me        → текущий пользователь
```

**Проверка:**

```bash
curl -X POST http://localhost:8000/api/v1/auth/login \
  -d "username=alice&password=alice"
```

---

## Шаг 4. Instance API (12–16 ч)

### 4.1 `vmorch/api/routes/instances.py`

| Method | Path | Кто | Действие |
|--------|------|-----|----------|
| POST | `/api/v1/instances` | user | Создать (owner = current_user) |
| GET | `/api/v1/instances` | user | Список своих |
| GET | `/api/v1/instances/{name}` | user | Детали (только своя) |
| POST | `/api/v1/instances/{name}/release` | user | Сдать свою |

**Тело POST /instances:**

```json
{
  "name": "myvm",
  "profile": "dev-docker",
  "ttl": "7d",
  "labels": { "project": "backend" }
}
```

### 4.2 Owner-scoping

В каждом handler для role=user:

```python
instance = db.get_by_name(name)
if instance.owner != current_user.username:
    raise HTTPException(403, "Not your instance")
```

Admin обходит фильтр через отдельные routes (шаг 5).

### 4.3 Интеграция с Engine

API не дублирует логику — вызывает `Engine` из фазы 0.

Для долгого provisioning возвращайте `202 Accepted` + `status=pending`, клиент поллит `GET /instances/{name}`.

---

## Шаг 5. Admin API (8–10 ч)

### 5.1 `vmorch/api/routes/admin.py`

Все routes с `dependencies=[Depends(require_admin)]`:

| Method | Path | Действие |
|--------|------|----------|
| GET | `/api/v1/admin/instances` | Все ВМ (фильтры: owner, status) |
| POST | `/api/v1/admin/instances/{name}/release` | Сдать любую |
| GET | `/api/v1/admin/capacity` | Mock capacity из провайдера |
| GET | `/api/v1/admin/users` | Список пользователей |
| PUT | `/api/v1/admin/users/{id}/quota` | Изменить квоту |

**Ответ `/admin/capacity`:**

```json
{
  "cpu": { "total": 64, "used": 8, "free": 56 },
  "memory_mb": { "total": 262144, "used": 32768, "free": 229376 },
  "instances_by_status": { "ready": 5, "provisioning": 1 }
}
```

---

## Шаг 6. Quota engine (8–10 ч)

### 6.1 `vmorch/core/quota.py`

Перед `engine.create()`:

```python
def check_quota(user_id: str, spec: VMSpec, db) -> None:
    quota = db.get_quota(user_id)
    usage = db.get_user_usage(user_id)  # sum vcpu, memory, count

    if usage.instance_count >= quota.max_instances:
        raise QuotaExceeded("max instances")
    if usage.vcpu + spec.vcpu > quota.max_vcpu:
        raise QuotaExceeded("max vcpu")
    # ...
```

Возвращайте HTTP 409 с понятным сообщением.

### 6.2 Profile visibility

User может заказать только profile с `allowed_for_users: true`. Admin — любой.

---

## Шаг 7. FastAPI app (4–6 ч)

### 7.1 `vmorch/api/main.py`

```python
app = FastAPI(title="VM Orchestrator")
app.include_router(auth_router, prefix="/api/v1")
app.include_router(instances_router, prefix="/api/v1")
app.include_router(admin_router, prefix="/api/v1")
```

### 7.2 Запуск

```bash
uvicorn vmorch.api.main:app --reload --port 8000
```

Swagger: `http://localhost:8000/docs`

---

## Шаг 8. CLI как API-клиент (6–8 ч)

Перепишите `vmorch/cli/main.py`:

- Флаг `--api-url http://localhost:8000`
- `vmorch login alice` → сохранить token в `~/.vmorch/credentials`
- Все команды → HTTP-запросы с Bearer token

```bash
vmorch login alice
vmorch create myvm --profile dev-docker
vmorch list
```

Сохраните режим `--local` для отладки без API (опционально).

---

## Шаг 9. Audit log (4–6 ч)

Записывайте в `audit_log`:

- `instance.create`, `instance.release`
- `admin.quota.update`
- actor = username, target = instance name

Admin endpoint: `GET /api/v1/admin/audit?limit=100`

---

## Критерии готовности

- [ ] Login возвращает JWT
- [ ] alice создаёт ВМ, bob не видит её в `GET /instances`
- [ ] bob получает 403 при `GET /instances/alice-vm`
- [ ] admin видит все ВМ в `/admin/instances`
- [ ] Квота: 4-я ВМ при max_instances=3 → 409
- [ ] `/admin/capacity` возвращает mock-метрики
- [ ] CLI работает через API
- [ ] Audit log пишется при create/release

---

## Частые ошибки

| Проблема | Решение |
|----------|---------|
| 401 на всех запросах | Проверьте заголовок `Authorization: Bearer <token>` |
| CORS в браузере | Добавьте `CORSMiddleware` в FastAPI (нужно для фазы 3) |
| Пароли в открытом виде | Только bcrypt hash в БД |

---

## Что дальше

[Фаза 2](phase-2-engine.md) — подключите Terraform (mock module) и Ansible в pipeline.
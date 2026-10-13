# Фаза 0: Mock MVP

**Оценка:** 40–50 чел·часов  
**Зависимости:** Python 3.11+, никаких внешних сервисов  
**Результат:** CLI, который создаёт «виртуальные» машины через заглушку и сохраняет их в SQLite

---

## Цель

Получить рабочий скелет системы: каталог профилей, заказ ВМ, список, сдача — всё на `MockProvider` без vCenter, Terraform и Ansible.

---

## Шаг 1. Инициализация проекта (4–6 ч)

### 1.1 Создайте структуру

```bash
mkdir -p vm_orchestrator/{vmorch/{cli,core,db,providers},profiles,flavors,stacks,instances}
cd vm_orchestrator
```

### 1.2 `pyproject.toml`

```toml
[project]
name = "vm-orchestrator"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = [
    "typer>=0.12",
    "rich>=13.0",
    "pydantic>=2.0",
    "pyyaml>=6.0",
]

[project.scripts]
vmorch = "vmorch.cli.main:app"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

### 1.3 Установка

```bash
pip install -e .
```

**Проверка:** `vmorch --help` выводит справку (пока пустую).

---

## Шаг 2. Интерфейс провайдера (4–6 ч)

### 2.1 Модели данных — `vmorch/core/models.py`

Определите Pydantic-модели:

- `Flavor`: `name`, `vcpu`, `memory_mb`, `disk_gb`
- `Profile`: `name`, `provider`, `image`, `flavor`, `stack: list[str]`
- `InstanceManifest`: `name`, `profile`, `ttl`, `owner`, `labels`
- `VMSpec`: объединённая spec после resolve profile + manifest
- `InstanceStatus`: enum `pending | provisioning | ready | releasing | released | failed`
- `ProviderInstance`: `id`, `name`, `ip`, `status`

### 2.2 Протокол — `vmorch/providers/base.py`

```python
from typing import Protocol

class CloudProvider(Protocol):
    name: str

    def create_vm(self, spec: VMSpec) -> ProviderInstance: ...
    def destroy_vm(self, instance_id: str) -> None: ...
    def get_status(self, instance_id: str) -> InstanceStatus: ...
    def get_capacity(self) -> ClusterCapacity: ...
```

Этот интерфейс **не меняйте** в следующих фазах — только добавляйте реализации (`proxmox`, позже `vsphere` в [extensions/](extensions/)).

---

## Шаг 3. Каталог YAML (4–6 ч)

### 3.1 `flavors/medium.yaml`

```yaml
name: medium
vcpu: 2
memory_mb: 4096
disk_gb: 40
```

### 3.2 `profiles/dev-docker.yaml`

```yaml
name: dev-docker
provider: mock
image: ubuntu-22.04
flavor: medium
network: default
stack:
  - docker
  - git
allowed_for_users: true
```

### 3.3 `vmorch/core/catalog.py`

Реализуйте:

- `load_flavor(name) -> Flavor`
- `load_profile(name) -> Profile`
- `resolve(manifest: InstanceManifest) -> VMSpec` — подставляет vCPU/RAM из flavor

**Проверка:** unit-тест или `python -c "from vmorch.core.catalog import load_profile; print(load_profile('dev-docker'))"`.

---

## Шаг 4. SQLite state (6–8 ч)

### 4.1 Схема — `vmorch/db/schema.sql`

```sql
CREATE TABLE instances (
    id          TEXT PRIMARY KEY,
    name        TEXT UNIQUE NOT NULL,
    owner       TEXT NOT NULL,
    profile     TEXT NOT NULL,
    status      TEXT NOT NULL,
    ip          TEXT,
    provider    TEXT NOT NULL DEFAULT 'mock',
    ttl_expires_at TEXT,
    created_at  TEXT NOT NULL,
    updated_at  TEXT NOT NULL
);
```

### 4.2 `vmorch/db/sqlite.py`

Методы:

- `init_db(path="~/.vmorch/state.db")`
- `insert_instance(...)`
- `get_by_name(name)`
- `list_instances(owner=None)` — если `owner` задан, фильтр
- `update_status(name, status, ip=None)`
- `delete_instance(name)`

**Проверка:** создайте запись вручную, прочитайте через `list_instances`.

---

## Шаг 5. MockProvider (8–10 ч)

### 5.1 `vmorch/providers/mock.py`

Логика `create_vm`:

1. Сгенерировать `id = uuid4()`
2. Назначить fake IP: `10.255.{random}.{random}`
3. Вернуть `status=provisioning`

Логика фонового перехода (в engine):

- Через 3 сек → `ready` (симуляция provisioning)
- `destroy_vm` → сразу `released`

`get_capacity()` — фиксированные значения:

```python
ClusterCapacity(
    cpu_total=64, cpu_used=8, cpu_free=56,
    memory_total_mb=262144, memory_used_mb=32768, memory_free_mb=229376,
)
```

### 5.2 Фабрика провайдеров — `vmorch/providers/__init__.py`

```python
def get_provider(name: str) -> CloudProvider:
    if name == "mock":
        return MockProvider()
    raise ValueError(f"Unknown provider: {name}")
```

---

## Шаг 6. Engine (6–8 ч)

### 6.1 `vmorch/core/engine.py`

```python
class Engine:
    def create(self, manifest: InstanceManifest) -> Instance:
        # 1. resolve profile
        # 2. check name unique in DB
        # 3. provider.create_vm(spec)
        # 4. save to DB (status=provisioning)
        # 5. simulate_ready(name) — threading.Timer или asyncio
        # return instance

    def release(self, name: str, owner: str | None = None) -> None:
        # 1. load instance, check owner if set
        # 2. status → releasing
        # 3. provider.destroy_vm
        # 4. delete from DB or status → released

    def list(self, owner: str | None = None) -> list[Instance]: ...
    def status(self, name: str) -> Instance: ...
```

**Идемпотентность:** `create` с существующим именем → ошибка `InstanceAlreadyExists`.

---

## Шаг 7. CLI (6–8 ч)

### 7.1 `vmorch/cli/main.py`

Команды Typer:

```bash
vmorch catalog list
vmorch catalog show dev-docker
vmorch create myvm --profile dev-docker --owner alice
vmorch list [--owner alice]
vmorch status myvm
vmorch release myvm [--yes]
```

Используйте `rich` для таблиц. Флаг `--json` для машинного вывода.

### 7.2 Пример сессии

```bash
vmorch create web-01 --profile dev-docker --owner alice
# подождать 3 сек
vmorch list --owner alice
vmorch status web-01    # ready, IP 10.255.x.x
vmorch release web-01 --yes
vmorch list --owner alice   # пусто
```

---

## Шаг 8. Манифесты и apply (4–6 ч)

### 8.1 `instances/myvm.yaml`

```yaml
name: myvm
profile: dev-docker
ttl: 7d
owner: alice
```

### 8.2 Команда `vmorch apply`

```bash
vmorch apply -f instances/myvm.yaml
vmorch apply -f instances/    # все файлы в каталоге
```

Парсинг YAML → `InstanceManifest` → `engine.create()`.

---

## Критерии готовности (чеклист)

- [ ] `pip install -e .` работает
- [ ] `vmorch catalog list` показывает profiles
- [ ] `vmorch create` создаёт ВМ, через ~3 сек статус `ready`
- [ ] `vmorch list --owner X` фильтрует по владельцу
- [ ] `vmorch release` удаляет ВМ из БД
- [ ] `vmorch apply -f instances/` читает YAML
- [ ] Повторный `create` с тем же именем — ошибка
- [ ] Интерфейс `CloudProvider` определён и используется

---

## Частые ошибки

| Проблема | Решение |
|----------|---------|
| БД не создаётся | Вызывайте `init_db()` при старте CLI |
| Статус застревает на provisioning | Проверьте Timer/callback в engine |
| Путь к profiles не найден | Ищите относительно корня проекта или `VMORCH_HOME` env |

---

## Что дальше

Переходите к [phase-1-api-rbac.md](phase-1-api-rbac.md): оберните engine в FastAPI и добавьте авторизацию.
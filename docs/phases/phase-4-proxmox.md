# Фаза 4: Подключение Proxmox VE

**Оценка:** 50–70 чел·часов  
**Предусловие:** [Фазы 0–3](README.md) завершены  
**Зависимости:** Proxmox VE 8+, Terraform, cloud-init шаблон в Proxmox  
**Результат:** `provider: proxmox` в profile создаёт реальные ВМ

---

## Цель

Подключить бесплатный Proxmox как целевой backend. Mock-провайдер остаётся для dev; vSphere/VCD — опционально позже ([extensions/](extensions/)).

---

## Шаг 0. Подготовка Proxmox (6–10 ч)

### Минимальная инсталляция

- Один узел Proxmox VE достаточен для dev.
- Установка: [Proxmox VE ISO](https://www.proxmox.com/en/downloads) на bare metal или nested в KVM.

### Чеклист

- [ ] Web UI доступен (`https://pve.example:8006`)
- [ ] API token создан: Datacenter → Permissions → API Tokens  
  - User: `vmorch@pve`  
  - Role: `VM.Administrator` или custom с `VM.Allocate`, `VM.Clone`, `VM.Config.*`
- [ ] QEMU-шаблон с cloud-init (см. шаг 0.2)
- [ ] Bridge `vmbr0` с DHCP для гостей
- [ ] Storage: `local-lvm` или ZFS

### Шаг 0.2. Golden template

1. Скачайте cloud image (Ubuntu 22.04 nocloud).
2. Создайте VM (например VMID **9000**), подключите cloud-init drive.
3. Установите `qemu-guest-agent`.
4. Convert to template (ПКМ → Convert to template).

Запишите **VMID** в profile: `image: "9000"`.

### Переменные окружения

```bash
export PROXMOX_VE_ENDPOINT="https://pve.example:8006"
export PROXMOX_VE_API_TOKEN="vmorch@pve!tokenid=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
# или user/password:
export PROXMOX_VE_USERNAME="vmorch@pve"
export PROXMOX_VE_PASSWORD="..."
```

---

## Шаг 1. Terraform proxmox module (14–18 ч)

### 1.1 Provider

Используйте [`bpg/proxmox`](https://registry.terraform.io/providers/bpg/proxmox/latest) (активно поддерживается).

`terraform/modules/proxmox-vm/versions.tf`:

```hcl
terraform {
  required_providers {
    proxmox = {
      source  = "bpg/proxmox"
      version = "~> 0.50"
    }
  }
}
```

### 1.2 Ресурс VM (clone from template)

```hcl
resource "proxmox_virtual_environment_vm" "vm" {
  name      = var.name
  node_name = var.node_name
  vm_id     = var.vm_id   # опционально: фиксированный или auto

  clone {
    vm_id = var.template_id   # 9000
    full  = true
  }

  cpu {
    cores = var.vcpu
  }

  memory {
    dedicated = var.memory_mb
  }

  disk {
    datastore_id = var.datastore_id
    size         = var.disk_gb
    interface    = "scsi0"
  }

  network_device {
    bridge = var.network_bridge
    model  = "virtio"
  }

  initialization {
    ip_config {
      ipv4 {
        address = "dhcp"
      }
    }
    user_account {
      username = "ubuntu"
      keys     = [var.ssh_public_key]
    }
  }

  agent {
    enabled = true
  }
}
```

### 1.3 Outputs

```hcl
output "ip" {
  value = proxmox_virtual_environment_vm.vm.ipv4_addresses[1][0]
  # индекс может отличаться — проверьте на реальной ВМ
}
```

**Проверка вручную:** `terraform apply` с одной ВМ вне оркестратора.

---

## Шаг 2. Provider registry (6–8 ч)

### 2.1 `vmorch/providers/registry.py`

```python
PROVIDERS: dict[str, type[CloudProvider]] = {
    "mock": MockProvider,
    "proxmox": ProxmoxProvider,
    # "vsphere": VsphereProvider,   # extensions
}

def get_provider(name: str) -> CloudProvider:
    cls = PROVIDERS.get(name)
    if not cls:
        raise UnknownProviderError(name)
    return cls()
```

### 2.2 `vmorch/providers/proxmox.py`

Обёртка над `TerraformRunner` с module `./modules/proxmox-vm` — **не дублируйте** Proxmox API в Python.

### 2.3 Renderer

В `instances.tf.json` для каждой ВМ:

```json
{
  "myvm": {
    "provider": "proxmox",
    "vcpu": 2,
    "memory_mb": 4096,
    "template_id": 9000,
    "node_name": "pve",
    "network_bridge": "vmbr0"
  }
}
```

Engine выбирает TF module по `spec.provider`.

---

## Шаг 3. Capacity collector (8–10 ч)

### `vmorch/core/capacity_proxmox.py`

Через Proxmox API (`GET /api2/json/nodes/{node}/status`):

| Метрика | Поле |
|---------|------|
| CPU total/used | `cpuinfo.cpus`, load |
| RAM | `memory.total`, `memory.used` |
| Disk | per-storage `maxdisk`, `disk` |

Кэш 5 мин. `GET /admin/capacity` → реальные данные если Proxmox настроен, иначе mock.

---

## Шаг 4. Ansible на реальных хостах (6–8 ч)

1. `unset VMORCH_ANSIBLE_MOCK`
2. SSH user `ubuntu` (из cloud-init template)
3. `wait_for_connection` timeout 300s
4. QEMU guest agent помогает получить IP быстрее

Dynamic inventory читает IP из TF output / БД.

---

## Шаг 5. Конфигурация (4–6 ч)

### `config.yaml`

```yaml
proxmox:
  endpoint: https://pve.example:8006
  node: pve
  datastore: local-lvm
  network_bridge: vmbr0
  template_id: 9000
  insecure: false   # true только dev с self-signed cert

default_provider: proxmox   # mock для локальной разработки без PVE
```

### Profile

```yaml
name: dev-docker
provider: proxmox
image: "9000"
flavor: medium
network: vmbr0
stack:
  - docker
```

---

## Шаг 6. Reconcile (4–6 ч)

`POST /admin/reconcile`:

1. `terraform refresh`
2. Сравнить TF state vs `instances/` vs Proxmox API
3. `--prune`: удалить ВМ в PVE, отсутствующие в манифестах

---

## Критерии готовности

- [ ] Template VMID 9000 существует в Proxmox
- [ ] TF module клонирует ВМ вручную
- [ ] API/UI создаёт реальную ВМ через `provider: proxmox`
- [ ] cloud-init: hostname + SSH key
- [ ] Ansible configure по SSH
- [ ] `release` удаляет ВМ из Proxmox
- [ ] Admin dashboard — реальный CPU/RAM
- [ ] `provider: mock` по-прежнему работает

---

## Частые ошибки

| Проблема | Решение |
|----------|---------|
| TLS certificate verify failed | `insecure: true` или добавьте CA |
| Clone failed: storage full | Проверьте `local-lvm` free space |
| Нет IP в output | Включите qemu-guest-agent, подождите DHCP |
| Permission denied | API token role VM.Administrator |
| VMID conflict | Не задавайте фиксированный vm_id или ведите пул |

---

## Что дальше

[Фаза 5](phase-5-hardening.md) — CI, remote state, production.

Опционально: [extensions/vsphere.md](extensions/vsphere.md) — второй провайдер для VMware DC.
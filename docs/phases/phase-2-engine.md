# Фаза 2: Engine + Terraform + Ansible (mock infra)

**Оценка:** 50–70 чел·часов  
**Предусловие:** [Фаза 1](phase-1-api-rbac.md) завершена  
**Зависимости:** Terraform >= 1.5, ansible-core >= 2.15 (опционально в mock-mode)  
**Результат:** Pipeline render → terraform apply → ansible configure при создании ВМ

---

## Цель

Заменить прямой вызов `MockProvider.create_vm()` на полноценный pipeline. Terraform управляет жизненным циклом (пока через mock module), Ansible настраивает stacks.

---

## Шаг 1. Terraform mock module (10–12 ч)

### 1.1 Структура

```text
terraform/
├── main.tf
├── variables.tf
├── outputs.tf
├── modules/
│   └── mock-vm/
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
└── generated/
    └── instances.tf.json   # генерируется Python
```

### 1.2 `modules/mock-vm/main.tf`

```hcl
variable "name" { type = string }
variable "vcpu" { type = number }
variable "memory_mb" { type = number }

resource "null_resource" "vm" {
  triggers = {
    name = var.name
  }

  provisioner "local-exec" {
    command = "sleep 2 && echo 'mock provisioned ${var.name}'"
  }
}

output "ip" {
  value = "10.255.${random_integer.octet1.result}.${random_integer.octet2.result}"
}

resource "random_integer" "octet1" { min = 0 max = 255 }
resource "random_integer" "octet2" { min = 1 max = 254 }
```

### 1.3 Корневой `main.tf`

```hcl
terraform {
  required_providers {
    null = { source = "hashicorp/null" }
    random = { source = "hashicorp/random" }
  }
}

module "vm" {
  for_each = var.instances
  source   = "./modules/mock-vm"
  name     = each.key
  vcpu     = each.value.vcpu
  memory_mb = each.value.memory_mb
}

output "instances" {
  value = { for k, m in module.vm : k => { ip = m.ip } }
}
```

**Проверка вручную:**

```bash
cd terraform
terraform init
terraform apply -var='instances={"test":{"vcpu":2,"memory_mb":4096}}'
terraform output
```

---

## Шаг 2. Renderer (10–12 ч)

### 2.1 `vmorch/core/renderer.py`

Читает все `instances/*.yaml` + resolve profiles → генерирует:

**`terraform/generated/instances.tf.json`:**

```json
{
  "variable": {
    "instances": {
      "default": {
        "myvm": { "vcpu": 2, "memory_mb": 4096, "provider": "mock" }
      }
    }
  }
}
```

Или auto.tfvars.json — на ваш выбор; главное — стабильный формат.

### 2.2 Вызывайте renderer

- Перед каждым `apply` / `release`
- При `release` — убираете ключ из `instances` → TF destroy на следующем apply

---

## Шаг 3. Terraform runner (8–10 ч)

### 3.1 `vmorch/runners/terraform.py`

```python
class TerraformRunner:
    def __init__(self, workdir: Path): ...

    def init(self) -> None:
        subprocess.run(["terraform", "init"], cwd=workdir, check=True)

    def apply(self) -> dict:
        # terraform apply -auto-approve -json
        # parse outputs → { "myvm": { "ip": "10.255.1.2" } }

    def destroy_target(self, name: str) -> None:
        # terraform destroy -target='module.vm["myvm"]' -auto-approve
```

Обрабатывайте ошибки: ненулевой exit code → `status=failed` в БД.

---

## Шаг 4. Ansible (10–14 ч)

### 4.1 Структура

```text
ansible/
├── ansible.cfg
├── inventory/
│   └── vmorch.py          # dynamic inventory
├── playbooks/
│   ├── configure.yml
│   └── cleanup.yml
├── roles/
│   ├── common/
│   ├── docker/
│   └── git/
└── requirements.yml
```

### 4.2 `playbooks/configure.yml`

```yaml
- hosts: vmorch_managed
  gather_facts: false
  tasks:
    - name: Apply stack roles
      include_role:
        name: "{{ item }}"
      loop: "{{ stack_roles }}"
      when: not ansible_check_mode
```

### 4.3 Mock-mode

Переменная окружения `VMORCH_ANSIBLE_MOCK=1`:

- `AnsibleRunner.run()` просто логирует «would run configure» и возвращает ok
- Полезно без SSH и реальных хостов

### 4.4 `vmorch/runners/ansible.py`

```python
class AnsibleRunner:
    def configure(self, host: str, ip: str, stack: list[str]) -> None: ...
    def cleanup(self, host: str, ip: str) -> None: ...
```

---

## Шаг 5. Обновлённый Engine (10–12 ч)

### 5.1 Pipeline create

```text
1. resolve manifest + profile
2. check quota
3. renderer.render_all()
4. status → provisioning
5. terraform.init() + terraform.apply()
6. ip = outputs[name].ip
7. status → configuring
8. ansible.configure(name, ip, stack)  # или mock skip
9. status → ready, save ip
```

### 5.2 Pipeline release

```text
1. status → releasing
2. ansible.cleanup()  # если не mock
3. renderer.render_all()  # без этой ВМ
4. terraform.apply()  # destroy via for_each removal
5. delete from DB / status → released
6. audit log
```

Запускайте долгие шаги в `BackgroundTasks` (FastAPI) или `asyncio.to_thread`.

---

## Шаг 6. Статусы в API (4–6 ч)

`GET /instances/{name}` возвращает:

```json
{
  "name": "myvm",
  "status": "configuring",
  "profile": "dev-docker",
  "ip": null,
  "pipeline_step": "ansible_configure"
}
```

Web UI (фаза 3) будет поллить этот endpoint.

---

## Критерии готовности

- [ ] `terraform init/apply` работает из Python
- [ ] Renderer генерирует корректный JSON из `instances/`
- [ ] Create: provisioning → configuring → ready
- [ ] Release: TF убирает ресурс из state
- [ ] `VMORCH_ANSIBLE_MOCK=1` пропускает SSH
- [ ] При ошибке TF статус → `failed`
- [ ] Outputs IP попадают в БД

---

## Частые ошибки

| Проблема | Решение |
|----------|---------|
| TF не находит module | Запускайте из `terraform/` с правильным cwd |
| for_each меняется — recreate all | Нормально для mock; позже — `lifecycle` |
| Ansible не находит хост | Dynamic inventory читает TF output или БД |

---

## Что дальше

[Фаза 3](phase-3-webui.md) — постройте Web UI поверх API.
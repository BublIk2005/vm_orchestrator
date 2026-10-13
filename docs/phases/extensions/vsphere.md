# Расширение: vSphere / vCenter (опционально)

**Оценка:** 60–80 чел·часов  
**Предусловие:** MVP завершён ([фазы 0–5](../README.md)), `ProviderRegistry` работает с Proxmox  
**Когда нужно:** корпоративный VMware DC, уже есть vCenter и лицензии

> Proxmox остаётся основным целевым провайдером. vSphere — **плагин**, не замена архитектуры.

---

## Что добавить

1. `terraform/modules/vsphere-vm/` — provider [`vmware/vsphere`](https://registry.terraform.io/providers/vmware/vsphere/latest)
2. `vmorch/providers/vsphere.py` — обёртка над TerraformRunner
3. Запись в `registry.py`: `"vsphere": VsphereProvider`
4. `vmorch/core/capacity_vsphere.py` — pyvmomi для admin dashboard
5. Profile: `provider: vsphere`, `image: ubuntu-22.04-template`

## Подключение в registry

```python
# vmorch/providers/registry.py
PROVIDERS = {
    "mock": MockProvider,
    "proxmox": ProxmoxProvider,
    "vsphere": VsphereProvider,   # + эта строка
}
```

Renderer выбирает module:

```python
MODULES = {
    "mock": "./modules/mock-vm",
    "proxmox": "./modules/proxmox-vm",
    "vsphere": "./modules/vsphere-vm",
}
```

## Подготовка vCenter

- [ ] vCenter host, datacenter, cluster, datastore, network
- [ ] Service account: VM provisioning
- [ ] Golden template (Packer): cloud-init + open-vm-tools
- [ ] cloud-init via `guestinfo.userdata` (vSphere 7.0 U3+)

```bash
export VSPHERE_SERVER="vcenter.example.com"
export VSPHERE_USER="svc-vmorch@corp.local"
export VSPHERE_PASSWORD="..."
```

## Terraform module (кратко)

```hcl
resource "vsphere_virtual_machine" "vm" {
  name             = var.name
  resource_pool_id = var.resource_pool_id
  datastore_id     = var.datastore_id
  num_cpus         = var.vcpu
  memory           = var.memory_mb

  clone {
    template_uuid = var.template_uuid
  }

  extra_config = {
    "guestinfo.userdata"          = base64encode(var.user_data)
    "guestinfo.userdata.encoding" = "base64"
  }
}
```

## Profile example

```yaml
name: corp-linux
provider: vsphere
image: ubuntu-22.04-template
flavor: medium
network: VLAN-Prod
stack:
  - docker
```

## Admin dashboard

`GET /admin/capacity` агрегирует несколько провайдеров:

```json
{
  "providers": [
    { "name": "proxmox", "cpu_free": 56 },
    { "name": "vsphere", "cpu_free": 120 }
  ]
}
```

## Критерии готовности

- [ ] TF module клонирует ВМ в vCenter вручную
- [ ] `provider: vsphere` работает через API/UI
- [ ] release удаляет ВМ из vCenter
- [ ] Proxmox и vSphere работают параллельно

## Частые ошибки

| Проблема | Решение |
|----------|---------|
| Clone timeout | Увеличить TF timeout, проверить datastore |
| cloud-init не применился | guestinfo encoding, VMware Tools в template |
| Permission denied | RBAC service account в vCenter |
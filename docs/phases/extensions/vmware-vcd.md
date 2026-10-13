# Расширение: VMware Cloud Director (опционально)

**Оценка:** 50–70 чел·часов  
**Предусловие:** MVP + [ProviderRegistry](../phase-4-proxmox.md#шаг-2-provider-registry-6-8-ч)  
**Когда нужно:** hosted VMware cloud, мультитенант через VCD

---

## Что добавить

1. `terraform/modules/vcd-vm/` — provider [`vmware/vcd`](https://registry.terraform.io/providers/vmware/vcd/latest)
2. `vmorch/providers/vmware_vcd.py`
3. Registry: `"vmware_vcd": VmwareVcdProvider`
4. `capacity_vcd.py` для admin dashboard

## Profile example

```yaml
name: vcd-dev
provider: vmware_vcd
image: ubuntu-22.04-vcd-template
flavor: medium
network: org-vdc-net-1
stack:
  - docker
```

## config.yaml

```yaml
vmware_vcd:
  url: https://vcd.example.com
  org: my-org
  vdc: my-vdc
  catalog: Public Catalog
```

## Подключение

Тот же паттерн, что Proxmox и vSphere:

- Renderer добавляет ключ в `instances.tf.json`
- Engine вызывает `registry.get("vmware_vcd")`
- Ansible configure/cleanup **без изменений** (SSH на гостевую ОС)

## Критерии готовности

- [ ] TF module создаёт vApp/VM в VCD
- [ ] release удаляет ресурс в VCD
- [ ] Capacity отображается в admin dashboard

## См. также

- [vSphere extension](vsphere.md) — on-prem альтернатива в экосистеме VMware
- [Общий план](../../PLAN.md)
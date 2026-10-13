# Пособия по фазам реализации

Пошаговые руководства для самостоятельной реализации VM Orchestrator.

**Целевой гипервизор:** Proxmox VE. Опциональные расширения (vSphere, VCD) — в [extensions/](extensions/).

## Порядок (MVP)

```text
Фаза 0 → Фаза 1 → Фаза 2 → Фаза 3 → Фаза 4 → Фаза 5
 mock     API       engine    Web UI    Proxmox   prod
```

Каждая фаза завершается **рабочим артефактом**, который можно демонстрировать.

## Документы MVP

| Фаза | Файл | Оценка | Внешние зависимости |
|------|------|--------|---------------------|
| 0 | [phase-0-mock.md](phase-0-mock.md) | 40–50 ч | нет |
| 1 | [phase-1-api-rbac.md](phase-1-api-rbac.md) | 60–80 ч | нет |
| 2 | [phase-2-engine.md](phase-2-engine.md) | 50–70 ч | Terraform CLI, Ansible (опционально) |
| 3 | [phase-3-webui.md](phase-3-webui.md) | 80–100 ч | Node.js |
| 4 | [phase-4-proxmox.md](phase-4-proxmox.md) | 50–70 ч | Proxmox VE |
| 5 | [phase-5-hardening.md](phase-5-hardening.md) | 40–60 ч | CI runner |

## Опциональные расширения

| Провайдер | Файл | Когда подключать |
|-----------|------|------------------|
| vSphere / vCenter | [extensions/vsphere.md](extensions/vsphere.md) | Корп. VMware DC |
| VMware Cloud Director | [extensions/vmware-vcd.md](extensions/vmware-vcd.md) | Hosted VMware cloud |

## Общий план

См. [../PLAN.md](../PLAN.md).

## Как пользоваться пособием

1. Прочитайте раздел «Цель» и «Критерии готовности».
2. Выполняйте шаги по порядку.
3. Отмечайте чеклист в конце документа.
4. Не переходите к следующей фазе, пока не выполнены критерии текущей.
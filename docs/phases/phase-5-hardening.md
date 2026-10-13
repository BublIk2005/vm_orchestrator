# Фаза 5: Production hardening

**Оценка:** 40–60 чел·часов  
**Предусловие:** [Фазы 0–4](README.md) завершены (минимум 0–3 для dev-only)  
**Результат:** Система готова к эксплуатации: CI, тесты, remote state, мониторинг

---

## Цель

Закрыть технический долг перед production: надёжность, безопасность, наблюдаемость, автоматические тесты.

---

## Шаг 1. Remote Terraform state (6–8 ч)

Локальный `terraform.tfstate` не подходит для команды.

| Backend | Когда |
|---------|-------|
| S3 + DynamoDB lock | AWS доступен |
| etcd / Consul | on-prem |
| Terraform Cloud | SaaS |

```bash
terraform init -migrate-state
```

**Проверка:** два параллельных `apply` — второй ждёт lock.

---

## Шаг 2. Секреты (6–8 ч)

| Секрет | Где |
|--------|-----|
| `PROXMOX_VE_API_TOKEN` | env / Vault |
| `JWT_SECRET` | env |
| SSH private key | `~/.vmorch/keys/` chmod 600 |

Не хранить в git. Ansible Vault для чувствительных group_vars.

---

## Шаг 3. Тесты (10–14 ч)

```text
tests/
├── test_catalog.py
├── test_quota.py
├── test_renderer.py
├── test_registry.py     # провайдеры mock + proxmox
└── test_api.py
```

- Фазы 0–3 — полностью на mock
- Contract tests на `CloudProvider`
- `terraform validate` + `terraform fmt -check`
- Molecule для Ansible roles

---

## Шаг 4. CI pipeline (6–8 ч)

```yaml
# .github/workflows/ci.yml
- run: pytest tests/
- run: terraform fmt -check -recursive terraform/
- run: cd web && npm ci && npm run build
```

Отдельный workflow `integration-proxmox` — manual dispatch с secrets.

---

## Шаг 5. TTL sweep и reconcile (6–8 ч)

```bash
# cron
0 */6 * * * vmorch sweep --yes
```

`reconcile --prune` — sync TF state / instances / Proxmox API.

---

## Шаг 6. Observability (6–8 ч)

- Structured logging (`structlog`)
- `GET /health` → db, terraform, proxmox reachability
- Prometheus `/metrics` (опционально)

---

## Шаг 7. Auth hardening (4–6 ч)

- Сменить дефолтные пароли
- Короткий JWT TTL или refresh tokens
- OIDC/LDAP (опционально)
- Rate limit на login

---

## Шаг 8. Деплой (6–8 ч)

Docker Compose или systemd + `docs/DEPLOY.md`.

Оркестратор может работать на отдельной ВМ/контейнере, не на Proxmox-ноде.

---

## Критерии готовности

- [ ] Remote TF state с locking
- [ ] Секреты не в git
- [ ] CI green
- [ ] sweep + reconcile работают
- [ ] DEPLOY.md написан

---

## Production checklist

- [ ] HTTPS (reverse proxy)
- [ ] Proxmox API token least privilege
- [ ] Backup: DB + TF state + `instances/`
- [ ] Runbook на failed creates

---

## Расширение

Новые провайдеры — см. [extensions/](extensions/): vSphere, VMware VCD, AWS.
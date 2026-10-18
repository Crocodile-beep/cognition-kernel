# ADR: Session Decisions — 5 June 2026

> **Context:** Полный день архитектурной сессии. Исследование рынка MCP/A2A/Gateway, first-principles анализ, позиционирование EPP, business model.
> **Участники:** Исследователь + Архитектор + Decision Maker
> **Статус:** Accepted

---

## 1. Архитектурное решение: Sync PEP Gate + Event Bus (гибрид)

### Проблема
Чистый event-native (всё async subscribers) делает EPP observer'ом, а не interceptor'ом. Без sync enforcement EPP — audit tool, а не governance standard.

### Решение
Sync PEP (gate) как единственный entry point для агента. Event Bus — для observability и provenance, НЕ для control flow.

```
Agent ──→ Sync PEP (gate)
             ├── PDP check inline (Cerbos)
             ├── Если DENY → publish event (audit) → return 403
             ├── Если ALLOW → publish event → execute → return result
             └── Event Bus → Provenance DAG (async subscribers)
```

### Что меняет в рефакторинге
- PDP subscriber — **убирается. PDP — inline.**
- PEP subscriber — **убирается. PEP — inline.**
- Provenance subscriber — **остаётся** (но как async observer)
- L7 subscribers — **остаются** (как async observers)
- main.py — не "publisher + subscribers", а **gate с inline checks**

### Что не меняет
- Пять атомов (Event, Capability, Decision, Policy, ProvenanceRecord) — **остаются**
- Четыре операции (RECORD, DECIDE, ENFORCE, PROVE) — **остаются**
- Пять законов композиции — **остаются** (sequential, parallel, conditional, scoped, supervised)
- Event-native — **остаётся** (события — first-class citizens для state)
- BSL 1.1 — **остаётся**

---

## 2. Продуктовое решение: EPP как MCP Governance Layer

### Проблема
Рынок подтверждает: MCP без governance непригоден для production. MCP Gateways (Portkey, Cloudflare, TrueFoundry) — коммерческие, proprietary, per-call pricing. Никто не сделал open source governance standard.

### Решение
EPP — **open source MCP Governance Layer**. Не новый протокол, не Gateway, не платформа. Слой между MCP (tool access) и A2A (agent collaboration).

```
Позиционирование:
  "MCP connects agents to tools.
   A2A connects agents to agents.
   EPP governs what happens when they do."
```

### Что НЕ делаем
- ❌ EPP Protocol (unified MCP+A2A) — слишком рано, разная семантика
- ❌ Chain topology built-in — не Critical Path
- ❌ A2A adapter — после MCP governance, не до
- ❌ L7 Lens как product — сначала adoption core

### Что делаем (порядок)
1. Phase 0: Refactoring main.py → Sync PEP gate
2. Phase 1: MCP Governance Layer (MCP adapter, Provenance, Policy Editor, Dashboard)
3. Phase 2: A2A, Chain, Ecosystem (если demand)

---

## 3. Бизнес-решение: Open Source Everything + Managed Cloud

### Проблема
Закрытый код enterprise features — классический Open Core, но создаёт paywall. Services/support — не масштабируется до $10M/год.

### Решение
**Всё open source. Без paywall. Revenue на managed service.**

| Слой | Лицензия | Revenue |
|------|----------|---------|
| EPP Protocol Spec | CC-BY | $0 — adoption |
| EPP Core (reference impl) | BSL 1.1 / MIT | $0 — adoption |
| Sync PEP runtime | BSL 1.1 / MIT | $0 — adoption |
| L7 modules | BSL 1.1 / MIT | $0 — adoption |
| UI/Dashboard | BSL 1.1 / MIT | $0 — adoption |
| **EPP Cloud** (managed hosting) | SaaS | **Usage-based** |
| **EPP Enterprise** (SSO, RBAC, SIEM, SLA) | Proprietary | **$100K+/yr** |
| **EPP Compliance** (SOC2/HIPAA pack) | Proprietary | **$50K+/yr add-on** |
| **Marketplace** (community integrations) | Platform | **20-30% commission** |

### Ключевой принцип
> **Разница между free и paid — не в коде. В сервисе.**

Free user: self-host, сам настраивает, сам обновляет, без SLA.
Paid user: managed hosting, compliance reports, SSO, SLA, контракт.

Small org использует EPP бесплатно и вечно. Enterprise платит за managed service.

### Revenue target ($10M/год)
50 enterprise × $200K ACV = $10M

Packaging:
- EPP Cloud (managed) — $50K/yr
- EPP Compliance add-on — $50K/yr
- EPP Enterprise (SSO, SIEM, SLA) — $100K/yr

### Почему не services
Services — $150K/employee, не масштабируется.
SaaS/Enterprise license — $400-500K/employee, масштабируется.

---

## 4. Компромисс, который мы принимаем

**Choose 2 out of 3:**

| Хотим | Что выбираем |
|-------|-------------|
| ✅ Real revenue ($10M/год) | Managed Cloud + Enterprise license |
| ✅ No vendor lock-in | Всё open source, self-hostable |
| ❌ No paywall | **Принимаем:** small org не может платить, но и не должна. Они используют open source бесплатно. |

**Enterprise платит за managed service, а не за доступ к фичам.**
Small org получает всё бесплатно — код, фичи, свободу.

---

## 5. Constitutional Principles (из external review)

### Проблема
Policy mutable, events immutable. Без causal link между ними replay недетерминирован, audit неполный, explainability фальшивая.

### Решение
Три конституционных принципа, которые нельзя нарушать:

| Принцип | Суть | Нарушение = смерть |
|---------|------|-------------------|
| **Events immutable** | Bus — append-only. Никакого update/delete. | Replay недетерминирован |
| **Policy versioned** | Каждый decision хранит `policy_version` (git commit hash) — minimal causal link | Невозможно объяснить "почему система тогда разрешила это?" |
| **Replay side-effect free** | Replay читает Bus + policy version, анализирует, НЕ исполняет. Replay — model, not runtime. | Temporal paradoxes, accidental re-execution |

### Как это выглядит в архитектуре

```
Gate публикует:
  execution.allowed {
    policy_version: "abc123",     ← causal link к policy на момент решения
    decision_id: "...",
    ...
  }

Policy хранится в git (код):
  /policies/deny-agent-x-bash/v3.rego
  version = git commit hash

Replay:
  читает Bus
  для каждого event: подтягивает policy_version из git
  воспроизводит анализ с той же policy, что была в оригинале
  НЕ вызывает execution (side-effect free)
```

### Что НЕ делаем
- ❌ Policy snapshot внутри event — раздувает события, дублирует state
- ❌ Bus как authoritative policy store — превращает bus в distributed config platform
- ❌ Replay как execution — temporal paradoxes

### Guardrail
> **Replay объясняет, не исполняет. Policy version — minimal causal link. Bus — append-only immutable log.**

---

## 6. Зафиксированные инсайты сессии

| Инсайт | Суть |
|--------|------|
| **MCP и A2A — разная семантика** | MCP = capability invocation, A2A = task delegation. Unification преждевременна. |
| **Gateway ≠ Proxy ≠ Router** | Разные уровни вопросов (bytes vs capability vs policy). |
| **MCP Gateway — костыль** | Все gateway коммерческие, решают проблему отсутствия governance в MCP. |
| **Sync PEP — Critical Path** | Без синхронного enforcement EPP — audit tool, не governance. |
| **EPP — не новый протокол** | EPP — governance layer поверх существующих протоколов. |
| **Всё open source — сильнее Open Core** | Защита не в коде, а в managed service. |
| **Enterprise платит за сервис, не за код** | SSO, compliance, SLA — то, что enterprise не хочет делать сам. |
| **Causal governance > audit log** | Без temporal causality governance неполный. Policy_version — minimal causal link. |
| **Three constitutional principles** | Events immutable + Policy versioned + Replay side-effect free. |
| **Counterfactual governance (future)** | "What if policy v7 existed at t=0?" — simulation engine на основе history. |

---

## 7. Следующие шаги

| Что | Когда |
|-----|-------|
| Начать Phase 0 (refactoring main.py → gate) | Завтра |
| Sync PEP design doc | В течение недели |
| Characterization tests (14 шт) | До начала рефакторинга |
| REFACTORING_PROCESS.md update (sync PEP модель) | После design doc |
| MONETIZATION_STRATEGY.md update | После фиксации решений |

---

*Session closed. Decisions captured. Moving to execution.*

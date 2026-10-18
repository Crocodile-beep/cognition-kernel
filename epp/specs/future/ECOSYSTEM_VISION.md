# EPP Ecosystem Vision

> **Назначение:** Описать EPP как платформу — экосистему вертикалей на L3 Event Bus и L7 Semantic Runtime.
> **Принцип:** Platform Business Model — протокол как основа, вертикали как продукты.
> **Дата:** 4 июня 2026.

---

## 1. First Principle: Что такое платформа?

Платформа отличается от продукта **одним фундаментальным свойством**:

```
Продукт: Я решаю твою проблему.
Платформа: Ты решаешь проблему твоего клиента, используя мою инфраструктуру.
```

Примеры:

| Инструмент | Это продукт или платформа? | Почему |
|-----------|---------------------------|--------|
| Slack | Платформа | Через Slack Apps кто угодно может построить своего бота |
| Stripe | Платформа | Через Stripe Connect кто угодно может принимать платежи |
| VS Code | Платформа | Через extensions кто угодно может добавить язык |
| OpenCode | Платформа | Через MCP кто угодно может добавить tools |
| Обычный SaaS | Продукт | Ты используешь то, что сделали мы |

**EPP становится платформой, когда:**
1. Через EPP проходит **весь AI-execution трафик** организации (MCP как стандарт)
2. На этом трафике можно строить **вертикальные продукты** (L7 Lens)
3. Эти продукты может строить не только команда EPP, но и **сторонние разработчики** (L7 Lens SDK)

---

## 2. Фундамент: Три слоя экосистемы

```
┌─────────────────────────────────────────────────────────────────────┐
│                        EPP ECOSYSTEM                                 │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Layer 1: Protocol (MCP)                                     │   │
│  │  ── Открытый стандарт подключения AI-агентов                  │   │
│  │  ── Любой агент, который говорит на MCP, может быть подключён │   │
│  │  ── Единый entry point для всех agent frameworks              │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                               │                                     │
│                               ▼                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Layer 2: Event Spine (L3)                                   │   │
│  │  ── Единый поток ВСЕХ governance-событий                     │   │
│  │  ── tool.invocation, decision.issued, execution.completed    │   │
│  │  ── approval.created, delegation.granted                     │   │
│  │  ── Все события связаны через parent_event_id (provenance)  │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                               │                                     │
│                               ▼                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Layer 3: Semantic Runtime (L7)                              │   │
│  │  ── Lens SDK для построения проекций над event stream'ом    │   │
│  │  ── Встроенные Lens: AgentDNA, Lattice, TraceLink           │   │
│  │  ── Ecosystem Lens: строятся сторонними разработчиками      │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                               │                                     │
│                               ▼                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │  Layer 4: Verticals (Продукты)                               │   │
│  │  ── Каждый Lens = отдельный продукт или модуль              │   │
│  │  ── Billing, Compliance, Analytics, Security                │   │
│  │  ── Marketplace Lens: сообщество публикует свои Lens        │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Четыре встроенные вертикали (Built-in Lens)

### Vertical 1: AgentDNA

**Что делает:** Аналитика поведения AI-агентов в организации.

**Проблема:** CTO не знает, чем на самом деле занимаются AI-агенты его команды. Сколько вызовов, какие tools, какие ошибки, какие паттерны.

**L7 Lens:**

| Метрика | Описание | Источник из L3 |
|---------|----------|----------------|
| Tool популярность | Какие tools вызываются чаще всего | tool.invocation |
| Agent behaviour profile | Каждый агент — свой профиль действий | Все события по agent_id |
| Deny rate | Сколько запросов отклонено / всего | decision.issued |
| Hold rate | Сколько запросов требует approval | decision.issued: HOLD |
| Anomaly detection | Агент запрашивает то, что не запрашивал раньше | Исторический профиль |
| Active hours | В какое время суток агенты активны | timestamp |

**Пример использования:**

```python
# AgentDNA Lens — usage report для CTO

report = agent_dna.team_report(team="backend", timeframe="7d")

print(f"Агенты команды backend сделали {report.total_calls} вызовов")
print(f"Топ-3 tools: {report.top_tools}")
print(f"Отклонено: {report.deny_rate}%")
print(f"Требуют approval: {report.hold_rate}%")

# → "Агенты команды backend сделали 1,247 вызовов"
# → "Топ-3 tools: bash(34%), read(28%), kubectl(12%)"
# → "Отклонено: 2.3%"
# → "Требуют approval: 8.1%"
```

**Кому продаётся:** CTO, VP Engineering.
**Ценность:** Понимание, чем занимаются AI-агенты в организации.

---

### Vertical 2: Lattice

**Что делает:** Граф доверия и делегирования между агентами и людьми.

**Проблема:** В organisation с delegation (агент А может делегировать capability агенту Б) никто не видит, как формируется цепочка доверия. Кто кому может делегировать? Кто кого approves? Кто эскалирует?

**L7 Lens:**

| Метрика | Описание | Источник из L3 |
|---------|----------|----------------|
| Trust graph | Кто кому может делегировать | delegation.granted |
| Delegation chains | Как capability прошла от A к B к C | Цепочка delegation |
| Approval network | Кто чаще всего approve, кто чаще всего denied | approval.* |
| Escalation patterns | Кто эскалирует, к кому уходит escalation | escalation.* |
| Weak links | Где цепочка слишком длинная или рискованная | Анализ графа |
| Role discovery | Какие роли реально существуют в системе | Principal aggregation |

**Пример использования:**

```python
# Lattice Lens — анализ графа доверия для CISO

graph = lattice.trust_graph(organization="acme-corp")

print(f"Граф содержит {graph.nodes} узлов и {graph.edges} связей")
print(f"Средняя длина цепочки делегирования: {graph.avg_chain_length}")
print(f"Слабые звенья: {graph.weak_links}")
print(f"Рекомендация: сократить цепочки длиннее 3 шагов")

# → "Граф содержит 47 узлов и 156 связей"
# → "Средняя длина цепочки делегирования: 2.3"
# → "Слабые звенья: agent-23 (7 эскалаций за неделю)"
# → "Рекомендация: сократить цепочки длиннее 3 шагов"
```

**Кому продаётся:** CISO, Security Team.
**Ценность:** Понимание графа доверия между AI-агентами.

---

### Vertical 3: TraceLink

**Что делает:** Полная forensic provenance — восстановление любой цепочки событий.

**Проблема:** Если происходит инцидент (агент сделал что-то опасное), нужно восстановить полную цепочку: кто запросил, какое решение было принято, кто approve, какой sandbox использовался, какой был результат.

**L7 Lens:**

| Метрика | Описание | Источник из L3 |
|---------|----------|----------------|
| Event trace | Полный DAG событий для одного trace_id | Все события по trace_id |
| Tool history | Все случаи использования tool за период | Все tool.invocation |
| Agent timeline | Всё, что делал агент за период | Все события по agent_id |
| Decision audit | Почему было принято решение (с reasoning) | decision.issued |
| Compliance report | Готовый отчёт для аудитора | Все события по фильтру |

**Пример использования:**

```python
# TraceLink Lens — forensic investigation

trace = tracelink.trace_event(event_id="evt_abc123")

print(f"Цепочка содержит {len(trace.events)} событий")
print(f"Время: {trace.start_time} → {trace.end_time}")
print(f"Участники: {trace.participants}")
print(f"Решения: {trace.decisions}")
print(f"")
print(f"Полная цепочка:")
for event in trace.chain:
    print(f"  {event.timestamp} | {event.type} | {event.actor} | {event.outcome}")

# → "Цепочка содержит 8 событий"
# → "Время: 14:32:01 → 14:32:47"
# → "Участники: agent-finance-01, tech-lead-carol, cto-bob"
# → "Решения: ALLOW (policy: production-deploy), HOLD → APPROVED"
# → ""
# → "Полная цепочка:"
# → "  14:32:01 | tool.invocation | agent-finance-01 | kubectl apply -f prod.yaml"
# → "  14:32:02 | decision.issued | PDP | HOLD (policy: production-deploy)"
# → "  14:32:15 | approval.created | tech-lead-carol | pending"
# → "  14:32:30 | approval.approved | tech-lead-carol | approved"
# → "  14:32:32 | execution.started | PEP | sandbox-id=42"
# → "  14:32:47 | execution.completed | PEP | success (deployment v2.3.1)"
```

**Кому продаётся:** Compliance Officer, Auditor, CISO.
**Ценность:** Расследования и compliance отчёты.

---

### Vertical 4: Billing Engine

**Что делает:** Usage-based billing для AI-агентов.

**Проблема:** Команды платят за AI-агентов, но никто не знает, кто сколько использует. Сколько tool calls сделала команда backend vs frontend? Какие capability самые дорогие?

**L7 Lens:**

| Метрика | Описание | Источник из L3 |
|---------|----------|----------------|
| Usage by team | Сколько вызовов сделала каждая команда | Principal → team mapping |
| Cost by capability | Какие capability самые ресурсоёмкие | execution.time |
| Forecast | Прогноз использования | Historical trend |
| Budget alerts | Превышение лимита | Текущее vs лимит |
| Cost allocation | Распределение затрат по командам | team_id |

**Пример использования:**

```python
# Billing Lens — cost report

billing = billing_engine.usage_by_team(month="2026-06")

for team in billing.teams:
    print(f"{team.name}: {team.total_calls} calls, ${team.cost:.2f}")

print(f"\nТоп-3 дорогих capability:")
for cap in billing.most_expensive_capabilities(limit=3):
    print(f"  {cap.name}: {cap.total_cost}")

# → "backend: 1,247 calls, $623.50"
# → "frontend: 892 calls, $446.00"
# → "data-science: 2,103 calls, $1,051.50"
# → ""
# → "Топ-3 дорогих capability:"
# → "  production.deploy: $2,100.00"
# → "  database.query: $890.00"
# → "  kubernetes.manage: $670.00"
```

**Кому продаётся:** Finance, Engineering Manager.
**Ценность:** Cost allocation и budgeting для AI-agent usage.

---

### Vertical 5: Compliance Suite

**Что делает:** Автоматические compliance reports для SOC2, HIPAA, SOX.

**Проблема:** Enterprise должен показать аудитору, что AI-агенты работают под контролем. Без EPP — это ручной сбор логов. С EPP — один клик.

**L7 Lens:**

| Метрика | Описание | Источник из L3 |
|---------|----------|----------------|
| SOC2 report | Готовый отчёт для SOC2 аудита | Все события |
| HIPAA report | Доступ к PHI контролируется | Фильтр по capability |
| SOX report | Все financial transactions audit | Фильтр по trust_zone |
| User access review | Кто имеет доступ к каким capability | delegation, policy |
| Change history | Все изменения политик | policy.updated |

**Кому продаётся:** Compliance Officer.
**Ценность:** Готовые отчёты для аудиторов.

---

## 4. Ecosystem Model: Как это масштабируется

### Phase 1: Built-in Lens (мы пишем)

Первые 5 Lens пишет команда EPP. Они:
1. Доказывают, что модель работает
2. Дают базовую ценность пользователям
3. Создают API и SDK для сторонних разработчиков

### Phase 2: Lens SDK (сторонние разработчики могут писать)

```python
# Lens SDK — любой разработчик может написать свой Lens

from epp.l7 import Lens, EventStream, Snapshot

class MyCustomLens(Lens):
    """
    Кастомный Lens, который считает статистику по моим tools.
    """
    
    lens_id = "my-custom-analytics"
    version = "1.0.0"
    
    def projection(self, event: EventStream, prev: Snapshot) -> Snapshot:
        # Считаем статистику
        if event.type == "tool.invocation":
            tool = event.data.tool_name
            prev.state[tool] = prev.state.get(tool, 0) + 1
        return prev
    
    def query(self, snapshot: Snapshot, params: dict) -> dict:
        # Отдаём результат через API
        return {
            "total_calls": sum(snapshot.state.values()),
            "by_tool": snapshot.state,
            "top_tool": max(snapshot.state, key=snapshot.state.get)
        }
```

**Marketplace Lens:**
```
EPP Cloud Lens Marketplace
├── Built-in: AgentDNA, Lattice, TraceLink, Billing, Compliance
├── Verified: HashiCorp Lens (Terraform compliance)
│             AWS Lens (cloud cost analysis)
│             Sentry Lens (error tracking correlation)
├── Community: (любой может опубликовать)
│             Slacks Lens, Jira Lens, ...
└── Enterprise: (платные, с поддержкой)
```

### Phase 3: Network Effects

```
Больше MCP-агентов → Больше событий в L3 → Ценнее L7 Lens
                                            ├── AgentDNA точнее
                                            ├── Lattice полнее
                                            ├── TraceLink глубже
                                            └── Billing точнее
                                                   │
                                                   ▼
                                    Больше причин подключить EPP
                                    Больше причин купить Team/Enterprise
                                                   │
                                                   ▼
                                    Больше разработчиков пишут Lens
                                    (marketplace эффект)
                                                   │
                                                   ▼
                                    Больше причин подключить EPP
                                    ──────────────────────────→ Loop
```

---

## 5. Платформа vs Продукт — что мы не теряем

Важно: EPP Core — **продукт** (security governance). Ecosystem — **платформа** (вертикали на L3+L7).

```
┌─────────────────────────────────────────────────────────────┐
│  Что мы продаём              │  Что создаёт экосистему       │
├──────────────────────────────┼───────────────────────────────┤
│  Безопасность AI-агентов     │  Аналитика AI-агентов         │
│  Контроль исполнения         │  Граф доверия                 │
│  Audit trail                 │  Forensic provenance          │
│  Sandbox изоляция            │  Compliance отчёты            │
│                              │  Billing metering             │
│                              │  Кастомные Lens от сообщества  │
├──────────────────────────────┼───────────────────────────────┤
│  Это причина купить EPP      │  Это причина НЕ уходить с EPP │
│  (core value proposition)    │  (lock-in через ecosystem)    │
└──────────────────────────────┴───────────────────────────────┘
```

**Core = причина прийти. Ecosystem = причина остаться.**

---

## 6. Как это выглядит для пользователя

### Solo Developer

```
OpenCode → EPP (security + sandbox)
   └── Ему не нужны Lens. Ему нужна безопасность.
   └── AgentDNA было бы интересно, но он solo.
   └── Free tier: без Lens.
```

### Team (через EPP Cloud)

```
EPP Cloud Dashboards:
├── AgentDNA: "Ваши агенты сделали 5,342 вызова на этой неделе"
├── Lattice: "Граф доверия: 12 человек, 23 агента, 47 связей"
├── TraceLink: "Полный audit для любого события за 90 дней"
├── Billing: "Распределение затрат по командам"
└── Policy Dashboard: "Политики, изменения, конфликты"

Каждый Dashboard = один Lens = одна причина не уходить.
```

### Enterprise (self-hosted + compliance)

```
Enterprise Compliance Portal:
├── SOC2 Report: готовый отчёт за Q2 2026
├── HIPAA Report: все доступы к PHI за период
├── TraceLink: 7-летний retention для forensic
└── Custom Lens:
    ├── Splunk Lens (данные уходят в Splunk)
    └── Custom Compliance Lens (под их стандарты)
```

---

## 7. Что построить, чтобы это заработало

### Нужно для Phase 1 (Built-in Lens)

| Компонент | Что делает | Когда |
|-----------|-----------|-------|
| L7 Lens SDK | Интерфейс для написания Lens | Phase 1 |
| Lens Registry | Хранилище Lens манифестов | Phase 1 |
| Lens Runtime | Жизненный цикл Lens (start, snapshot, query) | Phase 1 |
| Lens API | REST API для доступа к Lens данным | Phase 1 |
| AgentDNA Lens | Вертикаль #1 | Phase 1 |
| TraceLink Lens | Вертикаль #2 | Phase 1 |

### Нужно для Phase 2 (Ecosystem)

| Компонент | Что делает | Когда |
|-----------|-----------|-------|
| Lens Marketplace | UI для поиска и установки Lens | Phase 2 |
| Lens Verification | Проверка безопасности сторонних Lens | Phase 2 |
| Lens Billing | Monetization для платных Lens | Phase 2 |
| Lens SDK docs | Документация для сторонних разработчиков | Phase 2 |
| Lattice Lens | Вертикаль #3 | Phase 2 |
| Billing Lens | Вертикаль #4 | Phase 2 |
| Compliance Lens | Вертикаль #5 | Phase 2 |

---

## 8. Почему это сложно скопировать

Конкуренты могут скопировать MCP Server. Они не могут скопировать **экосистему**:

```
Скопировать можно:
  ✅ MCP Protocol Handler
  ✅ PDP + Cerbos
  ✅ PEP + Landlock + seccomp

Нельзя скопировать:
  ❌ L3 Event Stream — заполняется реальными событиями от реальных агентов
  ❌ L7 Lens — работают только на твоём Event Stream
  ❌ Сетевой эффект — чем больше у тебя агентов, тем ценнее твоя платформа
```

**Копируется функциональность. Не копируется инерция сети.**

---

## 9. One-Sentence Takeaway

> **EPP — это не просто security-продукт. Это платформа, на которой L3 Event Stream (все governance-события от всех AI-агентов) и L7 Semantic Runtime (проекции над этим stream'ом) позволяют строить экосистему вертикальных продуктов: от аналитики поведения агентов до compliance отчётов.**

---

*Документ создан 4 июня 2026.*
*Следующий шаг: спецификация L7 Lens SDK.*

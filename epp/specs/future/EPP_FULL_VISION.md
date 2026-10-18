# EPP Full Vision: Execution Provenance Platform

> **Дата:** 4 июня 2026  
> **Статус:** Architectural & Product Concept — синтез сессии  
> **Принцип:** Event-native, MCP-friendly, Verticals-as-Platform  
> **Лицензия:** BSL 1.1 (Business Source License)  

---

## Executive Summary

**EPP (Execution Provenance Platform)** — это платформа управления исполнением AI-агентов. Она обеспечивает Safety (агент не может навредить), Audit (полная цепочка решений) и Control (централизованное управление политиками) для любого AI-агента, который говорит на MCP (Model Context Protocol).

EPP — это не прокси, не API Gateway и не мониторинг. Это **локальный security-агент**, который встраивается между AI-ассистентом и операционной системой: проверяет каждое действие, исполняет в sandbox и записывает полную цепочку решений для аудита.

Архитектура: **event-native** (4 слоя: Event Bus, PDP, PEP, Semantic), **MCP-friendly** (любой MCP-агент подключается за 1 минуту), **платформа с вертикалями** (5 встроенных L7 Lens + marketplace для сторонних).

Модель: **Open Source Core (BSL) + Enterprise Subscription** — как Docker, GitLab, HashiCorp. Free для индивидуальных разработчиков, Team ($15/user/mo) для команд, Enterprise (custom) для compliance.

---

## Part 1: The Problem

### 1.1 Универсальная истина

```
AI-агенты выполняют действия от имени человека.
Эти действия могут быть опасными: удаление файлов, запись в БД, деплой на production.
Ни одна существующая система не гарантирует, что агент не сделает 
то, что ему не разрешено.
```

### 1.2 Почему существующие решения не работают

| Решение | Проблема |
|---------|----------|
| OpenCode Plugin Hooks | UX-level permission, не security boundary. Subagent-ы не trigger'ят hooks |
| Sandlock MCP | Per-tool sandbox, но нет governance, нет semantic, нет provenance |
| Docker-in-Docker | Тяжёлый (секунды), нет policy engine |
| gVisor/Firecracker | Изоляция есть, governance нет |
| OPA/Cedar | Governance есть, enforcement нет |
| RBAC | Не работает для dynamic AI-сценариев |

**Никто не соединяет governance + sandbox + semantic + provenance в одной системе.**

### 1.3 Три потребности пользователя

| # | Потребность | Что это значит |
|---|-------------|----------------|
| 1 | **Safety** | Агент не может выполнить действие, которое мне навредит |
| 2 | **Audit** | Я могу точно сказать, что агент сделал и почему (с reasoning chain) |
| 3 | **Control** | Я могу управлять тем, что агент может делать, централизованно |

---

## Part 2: The First Principles

### 2.1 Пять атомов EPP

| Атом | Определение | Пример |
|------|-------------|--------|
| **Event** | Атомарный факт, что что-то произошло | `tool.invocation`, `decision.issued` |
| **Capability** | Атомарное право на действие | `fs:read:/src/`, `shell:exec:npm` |
| **Decision** | Атомарный акт выбора | `allow`, `deny`, `hold` |
| **Policy** | Правило, мапящее Event в Decision | `"deploy to prod → HOLD"` |
| **ProvenanceRecord** | Связь между Event и Decision | `tool.call → decision.allow` |

### 2.2 Четыре атомарные операции

| Операция | Input | Output |
|----------|-------|--------|
| **RECORD** | Event | ProvenanceRecord |
| **DECIDE** | Event + Policy[] | Decision |
| **ENFORCE** | Decision + Capability | Result |
| **PROVE** | Event ID | ProvenanceRecord[] |

### 2.3 Пять законов композиции

| Закон | Суть |
|-------|------|
| **Event causality** | Каждый Enforcement порождает 2 события (start, complete) |
| **Decision precedes enforcement** | Enforcement не начинается до решения |
| **Capability is output** | Policy выдаёт capability, enforcement использует |
| **Provenance append-only** | Только INSERT, никогда UPDATE |
| **Boundariness** | Каждый слой знает только о своих атомах |

---

## Part 3: The Architecture

### 3.1 Event-native ядро

EPP построен на **event-native архитектуре**, а не event-sourced. Разница:

```
Event-Sourced (как НЕ надо):
  main.py → Cerbos (gRPC) → NATS.publish (одно событие) → Response
  События сохраняются, но НЕ управляют потоком

Event-Native (как надо):
  Event Bus → PDP (subscriber) → Event Bus → PEP (subscriber) → Event Bus
  События управляют потоком — каждый шаг = событие
```

### 3.2 Четыре слоя

```
Layer 0: Event Bus
  └── In-process event bus. Все компоненты общаются через события.
  └── NATS JetStream для persistence и guaranteed delivery.
  └── Пользователь не знает о его существовании.

Layer 1: PDP — Policy Decision Point
  └── Принимает решения: ALLOW, DENY, HOLD.
  └── Pluggable engine: Cerbos, OPA/Rego, custom.
  └── Пользователь настраивает через политики (YAML/Rego).

Layer 2: PEP — Policy Enforcement Point
  └── Исполняет действие в sandbox с ограничениями.
  └── Sandbox Strategy Pattern: LinuxNative, Docker, macOS, Windows.
  └── Пользователь не настраивает — работает автоматически.

Layer 3: Semantic Layer
  └── Транслирует семантические capability (source_code.read) 
      в операционные (fs:read:/src/**).
  └── Определяет intent пользователя (read vs modify vs deploy).
  └── Пользователь опционально настраивает онтологию.

Layer 4: Provenance Store
  └── Запоминает все события, решения, результаты.
  └── Append-only DAG. Полная реконструируемость.
  └── Пользователь читает через Audit Dashboard.
```

### 3.3 Sandbox Strategy Pattern

Sandbox — не конкретная реализация, а интерфейс:

```
interface SandboxStrategy {
  name: string;
  isolationLevel: 1-6;
  canRun(request): boolean;
  run(request): SandboxResult;
  terminate(id): void;
}

Strategies:
├── LinuxNative  — Landlock + seccomp + namespaces (level 6)
├── Docker       — Docker container (level 5)
├── MacOSSandbox — macOS sandbox-exec (level 3)
├── WindowsJobObject — Windows Job Objects (level 2)
└── Fallback     — chroot + ulimit (level 1)
```

---

## Part 4: MCP Integration

### 4.1 Почему MCP

MCP (Model Context Protocol) — открытый стандарт от Anthropic (Apache 2.0). 

Поддерживается всеми современными AI-агентами:
- **OpenCode** — нативная поддержка MCP servers
- **Claude Desktop** — первый MCP-совместимый клиент
- **Cline** — VS Code MCP-клиент
- **Continue** — open-source MCP-клиент
- **Cody** — Sourcegraph MCP-клиент

**EPP — MCP Server.** Любой MCP-клиент подключается без единой строки адаптера.

### 4.2 Как подключается

```bash
# Solo developer — одна команда
opencode mcp add cognition-kernel \
  --command "cognition-kernel"

# Или для Claude Desktop
# claude_desktop_config.json:
{
  "mcpServers": {
    "epp-governance": {
      "command": "cognition-kernel"
    }
  }
}
```

### 4.3 Что покрывает MCP

| Операция | Через MCP | Как |
|----------|-----------|-----|
| Tool execution | ✅ `tools/call` | EPP проверяет → исполняет |
| Capability registry | ✅ `tools/list` | Возвращает доступные tools |
| HOLD lifecycle | ✅ `tools/call` → "pending_approval" | Агент ждёт подтверждения |
| Delegation | ✅ Custom tool `epp.delegation.create` | Через EPP tool |
| **Admin/analytical** | **❌ EPP REST API** | Для dashboard, audit, настроек |

---

## Part 5: The Product

### 5.1 Solo Developer

**Установка:** `pip install cognition-kernel` — 1 минута. Всё локально.

**Работа:** Агент → MCP (stdio) → PDP → PEP (sandbox) → Result.

**Audit:** Локальный SQLite.

**Цена:** $0 (Free, Open Source Core).

### 5.2 Team

**Установка:** Каждый разработчик ставит локальный EPP + `cognition-kernel connect --team`.

**Работа:** Агент → MCP (stdio) → PDP (политики синхронизированы из Cloud) → PEP.

**Audit:** Локально + EPP Cloud Dashboard (все события команды).

**Управление:** CTO пишет политики в Git → Cloud пушит на всех членов команды.

**Цена:** $15/user/мес.

### 5.3 Enterprise

**Установка:** Self-hosted EPP Server на своей инфраструктуре (Kubernetes/VM).

**Работа:** Agent Backend → MCP (HTTP) → PDP (on-prem Cerbos) → PEP → Audit.

**Audit:** Собственный PostgreSQL + SIEM (Splunk/Datadog).

**Compliance:** SOC2, HIPAA, SOX отчёты — готовые для аудитора.

**Цена:** Custom (зависит от числа агентов и уровня поддержки).

---

## Part 6: The Monetization

### 6.1 Модель: Open Source Core + Enterprise Subscription

Точно как Docker, GitLab, HashiCorp, Grafana, CrowdStrike.

### 6.2 Лицензия: BSL 1.1

```
Business Source License 1.1

Разрешает:
  ✅ Free: individual developer, team ≤3 users
  ✅ Internal use, modification, distribution
  ✅ Через 4 года → Apache 2.0 (полная open source)

Запрещает:
  ❌ SaaS/managed service без коммерческой лицензии
  ❌ Production-использование командами >3 человек без подписки
```

BSL — это **не AGPL** (не пугает enterprise). **Не MIT** (не даёт AWS забрать). **Проверенная модель** (CockroachDB, HashiCorp, Sentry).

### 6.3 Product Tiers

| Tier | Цена | Кому | Что входит |
|------|------|------|-----------|
| **Free** | $0 | Individual dev | Local MCP, single user, local audit, Linux sandbox |
| **Team** | $15/user/mo | Команда 3-50 чел | Free + Cloud audit, team policies, SSO, dashboard |
| **Enterprise** | Custom | Enterprise 50+ | Team + Self-hosted, compliance, 7yr retention, SLA, on-prem PDP |

### 6.4 Revenue Model

```
Free:  adoption channel (чем больше, тем лучше)
Team:  основной revenue (subscription, $15/user/mo)
Enterprise: high-ticket (custom pricing, annual contracts)
Marketplace: 30% commission on paid Lens (Phase 3)
```

---

## Part 7: The Ecosystem

### 7.1 Три слоя платформы

```
Layer 1: MCP Protocol — открытый стандарт подключения AI-агентов
Layer 2: L3 Event Bus — единый поток всех governance-событий
Layer 3: L7 Semantic Runtime — проекции над event stream'ом (Lens)
```

### 7.2 Пять встроенных вертикалей (Lens)

| Вертикаль | Что даёт | Кому |
|-----------|----------|------|
| **AgentDNA** | Аналитика поведения AI-агентов: кто, что, когда, как часто | CTO |
| **Lattice** | Граф доверия и делегирования между агентами и людьми | CISO |
| **TraceLink** | Полная forensic provenance — восстановление любой цепочки | Compliance Officer |
| **Billing** | Usage-based billing — кто сколько использует AI-агентов | Finance |
| **Compliance Suite** | SOC2/HIPAA/SOX отчёты — один клик до аудитора | Auditor |

### 7.3 Network Effect

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
                                    Ещё больше агентов → LOOP
```

### 7.4 Lens SDK (Phase 2)

```python
from epp.l7 import Lens, EventStream, Snapshot

class MyLens(Lens):
    lens_id = "custom-analytics"
    
    def projection(self, event, prev):
        # Любая логика над event stream'ом
        return prev
    
    def query(self, snapshot, params):
        return {"result": snapshot.state}
```

**Marketplace:** Free, Verified (платные с поддержкой), Community.

---

## Part 8: The Evolution (Three Tracks)

```
Поток A: Текущий EPP (Python)
  └── Существующая кодовая база (~1800 строк)
  └── Рефакторинг: event-sourced → event-native
  └── Изменение: только main.py (~400 строк)
  └── Сохраняется: Cerbos, NATS, Ingest, PG, все тесты

Поток B: CognitionKernel Plugin (TypeScript)
  └── Новая кодовая база (с нуля)
  └── Event-native с первого дня
  └── MCP entry point
  └── Sandbox Strategy Pattern
  └── Reference implementation для EPP Core 2.0

Поток C: EPP Protocol (будущее)
  └── Слияние опыта Потока A и архитектуры Потока B
  └── Federation: несколько PDP, PEP, общий Provenance DAG
  └── Semantic Intelligence Layer
  └── Универсальный протокол управления исполнением
```

---

## Part 9: From User's Eyes — One-Page Summary

### Solo Developer

```
INSTALL:
  pip install cognition-kernel
  opencode mcp add cognition-governance --command "cognition-kernel"

FLOW:
  Agent → MCP (stdio) → PDP → PEP (sandbox) → Result

YOU SEE:
  - Агент работает как обычно
  - Если опасно → запрос подтверждения
  - Всё записано в локальный audit

COST: $0
LICENSE: BSL 1.1 (Free for individual use)
```

### Team Lead

```
SETUP:
  cognition-kernel cloud init --team "Acme"
  Каждый dev: cognition-kernel connect --team acme

FLOW:
  Ты пишешь политики → Cloud пушит на всех
  Каждый dev имеет локальный EPP + общие политики

YOU SEE:
  - Audit Dashboard: все действия агентов команды
  - Кто что делал, какие решения приняты
  - SSO, team management, общие политики

COST: $15/user/month
```

### CISO / Compliance Officer

```
INFRASTRUCTURE:
  Self-hosted EPP на своей инфраструктуре (или Cloud)

FLOW:
  Любой agent backend → MCP (HTTP) → EPP → on-prem PDP → sandbox → audit

YOU SEE:
  - SOC2/HIPAA/SOX отчёты — готовые, один клик
  - TraceLink — forensic provenance для любого события
  - Lattice — граф доверия и делегирования
  - 7-year retention, append-only audit, SIEM integration

COST: Custom (annual contract)
```

---

## Part 10: Architectural Decisions (Recorded)

| AD | Решение | Альтернативы | Обоснование |
|----|---------|-------------|-------------|
| **AD-001** | Event Bus как фундамент | RPC (как Sandlock) | Реактивная композиция, наблюдаемость, provenance |
| **AD-002** | Явное разделение PDP/PEP | Смешаны в main.py | Security best practice, заменяемость |
| **AD-003** | Sandbox Strategy Pattern | Жёсткая реализация | Кроссплатформенность, заменяемость |
| **AD-004** | Semantic Layer отдельно | В Rego-политиках | Онтология независима от политик |
| **AD-005** | Refactoring, не Rewrite | Rewrite с нуля | 1400 рабочих строк, тесты, knowledge |
| **AD-006** | Python для EPP, TS для Plugin | Один язык для всего | Экосистема OpenCode (TS), существующий стек (Python) |
| **AD-007** | MCP для agent-facing, API для admin | Всё через MCP | MCP не покрывает admin операции |
| **AD-008** | Replay = поведение, не состояние | Только чтение из PG | Можно перезапустить Cerbos на истории |
| **AD-009** | Event Schema Registry | Один NATS subject | Масштабирование event types |
| **AD-010** | Guaranteed delivery + DLQ | Fire-and-forget | Надёжность цепочки подписчиков |
| **AD-011** | BSL 1.1 лицензия | MIT, AGPL, Apache | Защита от AWS, adoption, enterprise |

---

## Part 11: Why This Will Work

### 11.1 Рынок существует

AI-агенты входят в enterprise. Каждая компания с AI-агентами **нуждается** в:
- Safety (чтобы агенты не навредили)
- Audit (чтобы знать, что агенты делают)
- Control (чтобы управлять, что агенты могут делать)

### 11.2 Правильная модель монетизации

Open Source Core (BSL) + Enterprise Subscription — проверенная модель. Docker, GitLab, HashiCorp, Grafana, CrowdStrike — все работают так. Free — adoption channel. Team — основной revenue. Enterprise — high-ticket.

### 11.3 Сетевой эффект

Чем больше агентов на EPP → тем больше событий → тем ценнее аналитика → тем больше причин подключить ещё агентов. Это loop, который сложно скопировать.

### 11.4 Открытый стандарт

MCP — открытый протокол (Apache 2.0). EPP не создаёт vendor lock-in. Любой MCP-агент подключается за минуту. Enterprise не боится закрытого протокола.

### 11.5 Архитектурная честность

Мы признали, что текущий EPP — event-sourced, не event-native. Мы спроектировали путь исправления. Никаких иллюзий, никакого wishful thinking. Только инженерная правда.

---

*EPP Full Vision v1.0*
*4 июня 2026*
*Следующий шаг: спецификация CognitionKernel Plugin (Поток B) — первая строчка кода.*

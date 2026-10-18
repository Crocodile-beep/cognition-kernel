# EPP Monetization Strategy

> **Контекст:** Архитектурная сессия Chief Architect EPP и Product Lead.
> **Принцип:** Open Source Core + Enterprise Subscription.
> **Модель:** Как Docker, GitLab, VS Code, HashiCorp, Grafana, CrowdStrike.
> **Дата:** 4 июня 2026.

---

## 1. Что мы продаём

EPP — это **Execution Governance Platform**. Система, которая гарантирует, что AI-агенты выполняют только разрешённые действия, и сохраняет полную цепочку решений для аудита.

Мы продаём не «MCP сервер». Мы продаём четыре ценности:

| Ценность | Описание | Кому нужно |
|----------|----------|------------|
| **Безопасность** | Гарантия, что AI-агент не навредит системе | Individual dev |
| **Аудит** | Полная цепочка: кто, когда, почему выполнил действие | Compliance-офицер |
| **Compliance** | Соответствие SOC2, HIPAA, SOX для AI execution | Enterprise |
| **Контроль** | Кто что может делать в AI-агентах, централизованно | Team lead |

---

## 2. Модель: Open Source Core + Enterprise Subscription

### Концепция

**Ядро EPP — открыто.** Local MCP Server, базовый PDP, sandbox, локальный provenance — доступны всем бесплатно.

**Подписка — за то, что выше одного пользователя:** централизованное управление политиками, облачный аудит, team management, compliance reports.

```
┌─────────────────────────────────────────────────────────────────────┐
│                      OPEN SOURCE CORE                               │
│  Local MCP Server + PDP + Linux Sandbox + Local Provenance          │
│  Бесплатно. Навсегда.                                               │
└─────────────────────────────────────────────────────────────────────┘
                                │
           ┌────────────────────┴────────────────────┐
           ▼                                         ▼
┌──────────────────────┐              ┌──────────────────────────────┐
│   EPP Cloud Free      │              │   EPP Enterprise              │
│   (регистрация)       │              │   (self-hosted / cloud)       │
├──────────────────────┤              ├──────────────────────────────┤
│ • Cloud Audit         │              │ • Всё из Team                 │
│ • Manual Policy Push  │              │ • Self-hosted option          │
│ • 1 user free         │              │ • Compliance reports          │
│                       │              │   (SOC2, HIPAA, SOX)          │
│   Подписка: $0        │              │ • SLA 99.9% / 99.99%          │
├──────────────────────┤              │ • Custom integrations          │
│   UPGRADE             │              │ • SIEM (Splunk, Datadog)      │
│       ▼               │              │ • On-prem PDP                 │
│   EPP Team            │              │ • Свой sandbox (Docker, VM)   │
│   (cloud)             │              │ • Dedicated support           │
│                       │              │ • Команды любого размера      │
│ • Central Policies    │              │                               │
│ • Team Audit Dashboard│              │   Подписка: Custom pricing    │
│ • SSO / SAML / Okta   │              │                               │
│ • Team Management     │              └──────────────────────────────┘
│ • Shared Ontology     │
│ • До 50 пользователей │
│                       │
│   Подписка: $15/user  │
│              /мес     │
└──────────────────────┘
```

---

## 3. Три продукта

### Tier 1: Free (Open Source Core)

**Что входит:**
- Локальный MCP Server (stdio transport)
- PDP: локальный OPA или Cerbos (single instance)
- PEP: Linux Native sandbox (Landlock + seccomp) + Docker strategy
- Provenance: локальный SQLite / JSONL
- Semantic Layer: base ontology (pass-through)
- Single user (один Principal)
- Навсегда

**Как работает:**
```bash
pip install cognition-kernel
# или
npm install @cognition-kernel/mcp

opencode mcp add cognition-kernel \
  --command "cognition-kernel"
```

**Кто потребляет:** Individual developer.
**Adoption channel:** Разработчики ставят локально для себя. Никакого прокси, никакого облака.

**Почему это бесплатно:**
1. Это adoption channel — entry point в продукт
2. Single user не даёт ценности централизованного управления
3. Разработчик привыкает к EPP, потом приводит команду

---

### Tier 2: Team ($15/user/month)

**Что добавляет к Free:**

Компонент | Free (локально) | Team (облачная синхронизация)
-----------|----------------|------------------------------
**PDP** | Локальный, одна политика | Централизованные политики, push на все MCP серверы
**Audit** | Локальный SQLite | Cloud Audit Dashboard (все события команды)
**Team** | N/A | Team Management: roles, permissions, SSO
**Policies** | Policy as code (git) | Policy UI: create, edit, version, rollback
**Ontology** | Base (pass-through) | Shared semantic ontology для команды
**Delegation** | N/A | Delegation между членами команды
**Approval** | N/A | Team-wide approval chains
**Sandbox** | Linux native | + Docker, + macOS (по платформе члена команды)

**Как подключается Team:**

```
Team Member 1                    Team Member N
┌──────────────────────┐        ┌──────────────────────┐
│  Локальный EPP        │        │  Локальный EPP        │
│  ├── PDP: cloud sync  │◄──────►│  ├── PDP: cloud sync  │
│  ├── Audit: → cloud   │        │  ├── Audit: → cloud   │
│  └── Sandbox: local   │        │  └── Sandbox: local   │
└──────────┬───────────┘        └──────────┬───────────┘
           │                               │
           └───────────────┬───────────────┘
                           │
                  ┌────────▼────────┐
                  │   EPP Cloud      │
                  │  ├── Policies   │
                  │  ├── Audit DB   │
                  │  ├── Team Mgmt  │
                  │  └── SSO        │
                  └─────────────────┘
```

**Кто потребляет:** Команда разработчиков (2-50 человек), использующая AI-агентов.

**Почему платят:**
1. **Центральные политики** — CTO пишет одну политику, она применяется на всей команде
2. **Общий аудит** — кто из команды что делал через AI-агентов
3. **SSO** — Okta/Google Workspace — без этого enterprise не пускает
4. **Delegation** — один разработчик может делегировать capability другому

---

### Tier 3: Enterprise (Custom pricing)

**Что добавляет к Team:**

| Компонент | Team | Enterprise |
|-----------|------|------------|
| **Хостинг** | EPP Cloud (SaaS) | Self-hosted (на инфраструктуре клиента) |
| **SLA** | Best effort | 99.9% / 99.99% |
| **Compliance** | N/A | SOC2 Type II, HIPAA, SOX reports |
| **SIEM** | N/A | Splunk / Datadog / Elastic integration |
| **PDP** | Cloud Cerbos/OPA | On-premise PDP (их собственный Cerbos) |
| **Sandbox** | Linux, Docker, macOS | Firecracker, gVisor, custom (их инфраструктура) |
| **Support** | Slack community | Dedicated support engineer |
| **Team size** | ≤50 | Unlimited |
| **Audit retention** | 90 days | 7 years (compliance requirement) |
| **Escalation** | Basic | Multi-level escalation chains |
| **Delegation** | Basic | Audit-grade delegation с full provenance |

**Кто потребляет:** Enterprise (50-5000+ AI-агентов), которым нужен compliance.

**Как подключается Enterprise (self-hosted):**

```
Enterprise Network
┌───────────────────────────────────────────────────────────┐
│  Enterprise Kubernetes / VM Cluster                        │
│  ┌──────────────────────┐    ┌────────────────────────┐   │
│  │  EPP Server (self)   │    │  EPP Server (self)     │   │
│  │  ├── PDP: Cerbos     │    │  ├── PDP: Cerbos       │   │
│  │  ├── Audit: PG        │    │  ├── Audit: PG          │   │
│  │  └── API: internal   │    │  └── API: internal     │   │
│  └──────────────────────┘    └────────────────────────┘   │
│           │                            │                   │
│           └──────────┬───────────────┘                    │
│                      │                                    │
│  ┌───────────────────▼────────────────────────────────┐  │
│  │  Enterprise Agent Backend                           │  │
│  │  └── вызывает EPP для governance check              │  │
│  └────────────────────────────────────────────────────┘  │
│                                                           │
│  Compliance: SOC2, HIPAA, SOX — все данные внутри        │
└───────────────────────────────────────────────────────────┘
```

**Enterprise НЕ ХОЧЕТ:**
- Чтобы их AI-трафик шёл через чужой облачный прокси (leak данных)
- Чтобы их политики и audit хранились на чужих серверах

**Enterprise ХОЧЕТ:**
- Self-hosted — всё на их инфраструктуре
- Compliance — готовые отчёты для SOC2/HIPAA/SOX аудиторов
- Centralized management своей команды
- Интеграцию со своими SIEM/Splunk/Datadog

---

## 4. Почему эта модель работает

### Аналогии, которые её доказывают

| Компания | Core (free) | Подписка | Наша аналогия |
|----------|-------------|----------|---------------|
| **Docker** | Docker Desktop (local) | Docker Hub, Team, Business | EPP MCP Server (local) → EPP Cloud |
| **GitLab** | GitLab CE (self-host) | GitLab.com, Enterprise | EPP core (self-host) → EPP Enterprise |
| **HashiCorp** | Terraform (CLI) | Terraform Cloud, Enterprise | EPP (CLI) → EPP Cloud |
| **CrowdStrike** | Falcon sensor (local) | Falcon Cloud console | Local MCP → EPP Cloud |
| **Grafana** | Grafana (OSS) | Grafana Cloud, Enterprise | Local audit → Cloud audit |

**Общий паттерн:** Ядро — open source, локальное, бесплатное. Подписка — за централизованное управление, compliance, поддержку.

### Почему это выгодно нам

1. **Free != убыток.** Free — это adoption channel. Чем больше индивидуальных разработчиков используют EPP, тем больше команд приходят за подпиской.

2. **Локальный MCP Server — не проблема, а актив.** Enterprise сам хочет, чтобы всё было локально. Мы продаём не «прокси», а «централизованное управление локальными security-агентами».

3. **Cloud не обязателен.** Enterprise может поставить self-hosted. Это снимает главное возражение enterprise: «наши данные не уйдут».

4. **Upgrade path есть и очевиден.**
   ```
   Individual dev (Free) → присоединяется к команде (Team) →
   компания вырастает (Enterprise)
   ```

---

## 5. Архитектурные изменения для этой модели

**Хорошая новость:** Всё, что нужно — уже есть в архитектуре EPP. Не нужно переписывать ядро.

### Нужно добавить

| Компонент | Что делает | Где живёт |
|-----------|-----------|-----------|
| **EPP Cloud API** | REST API для синхронизации политик, сбора аудита, управления командой | Cloud (новый сервис) |
| **Cloud Sync Agent** | Модуль в локальном EPP, который пушит событий в Cloud и пуллит политики | Встраивается в core |
| **Team DB** | PostgreSQL для команд, ролей, SSO | Cloud |
| **Audit Aggregator** | Принимает события от всех локальных EPP, складывает в единую БД | Cloud |
| **Policy Pusher** | Публикует политики из Cloud на все локальные EPP подписчики | Cloud |
| **Billing** | Stripe / Chargebee интеграция | Cloud |
| **Dashboard UI** | React/Next.js для Team и Enterprise | Cloud |

### Что НЕ нужно менять

```
❌ Ядро EPP (PDP, PEP, Sandbox, Provenance, Semantic Layer)
❌ MCP Protocol Handler
❌ Event Bus (NATS) — остаётся локальным
❌ Existing tests
❌ Decision tree
```

**EPP Cloud — это НЕ новый EPP.** Это облачная проекция того же EPP. Локальный EPP уже генерирует события. Cloud — просто подписчик на эти события + API для управления.

---

## 6. Как это выглядит в коде (для понимания)

```python
# Локальный EPP — новый модуль: Cloud Sync Agent
# Он не меняет flow, а добавляет опциональную синхронизацию

class CloudSyncAgent:
    """
    Опциональный модуль. Если user не ввёл API key — не активен.
    """
    
    def __init__(self, api_key: str | None):
        self.api_key = api_key
        self.cloud_api = "https://api.cognition-kernel.com"
    
    async def on_decision(self, event: GovernanceEvent):
        """Вызывается после каждого decision.issued.
        
        Если есть подписка — шлёт событие в Cloud.
        Если нет — ничего не делает.
        """
        if not self.api_key:
            return  # Free user — всё локально
        
        await httpx.post(
            f"{self.cloud_api}/v1/events",
            json=event.dict(),
            headers={"Authorization": f"Bearer {self.api_key}"}
        )
    
    async def pull_policies(self) -> list[Policy]:
        """Забирает актуальные политики из Cloud.
        
        Если нет подписки — возвращает пустой список.
        """
        if not self.api_key:
            return []
        
        response = await httpx.get(
            f"{self.cloud_api}/v1/policies",
            headers={"Authorization": f"Bearer {self.api_key}"}
        )
        return [Policy(**p) for p in response.json()]
```

**Локальный EPP не знает про подписку.** Он просто вызывает `on_decision()` после каждого решения. Если CloudSyncAgent не активен — ничего не происходит. Если активен — события уходят в Cloud.

---

## 7. Roadmap монетизации

### Phase 0: Open Source Core (сейчас)
- Local MCP Server
- PDP + PEP + Provenance
- Free для всех
- **Цель:** adoption, community, feedback

### Phase 1: EPP Cloud MVP (1-2 месяца после core)
- Cloud Audit Dashboard (events из локальных EPP)
- Team Management (invite, roles)
- SSO (Google, GitHub)
- Policy sync (push из cloud на локальные EPP)
- **Цель:** Team tier ($15/user/mo)

### Phase 2: Enterprise (2-4 месяца после core)
- Self-hosted option (Docker image, helm chart)
- Compliance reports (SOC2)
- SIEM integration (Splunk, Datadog)
- SLA
- On-prem PDP
- **Цель:** Enterprise tier (custom pricing)

### Phase 3: Marketplace (6+ месяцев)
- Policy Marketplace — sharing политик для common tools
- Ontology Marketplace — sharing semantic онтологий
- **Цель:** Ecosystem (network effects)

---

## 8. Ключевые метрики (OKRs)

| Метрика | Free | Team | Enterprise |
|---------|------|------|------------|
| **Users** | Install count | Active seats | Active agents |
| **Revenue** | $0 | $15/user/mo | $X/custom |
| **Adoption** | DAU of MCP | % от free who upgrade | % от team who upgrade |
| **Retention** | — | Month-over-month churn < 3% | Annual contract |

**North star:** Количество AI-агентов, работающих под EPP governance.

---

## 9. Ответ на главный вопрос

> Продавать локальный MCP сервер по подписке?

**Да. Но не за «локальность» — за «централизованное управление».**

Локальный MCP Server — **бесплатный** и навсегда. Он работает сам по себе, без облака, для одного разработчика.

Подписка (Team/Enterprise) — за возможность **управлять тысячей таких MCP серверов централизованно**. За общие политики, общий аудит, compliance, SSO, поддержку.

**Это самая проверенная enterprise-модель.**
- Docker делает так.
- GitLab делает так.
- HashiCorp делает так.
- CrowdStrike делает так.
- Grafana делает так.

**EPP делает так.**

---

## 10. Что нужно сделать прямо сейчас

1. **Закончить core** — PDP, PEP, Provenance, Semantic Layer, MCP Adapter
2. **Сделать Free tier идеальным** — чтобы individual developer не хотел уходить
3. **Спроектировать Cloud Sync Agent** — как модуль в локальном EPP, который опционально шлёт события в Cloud
4. **Не писать Cloud** — пока нет product-market fit для Free

**Золотое правило:** Free — это продукт, а не демо-версия. Если Free не решает проблему individual developer'а — никто не дойдёт до Team.

# CognitionKernel: Архитектурное предложение (EPP-aligned)

> **Автор:** Главный Архитектор EPP  
> **Статус:** Черновик для обсуждения  
> **Основание:** Архитектурная рецензия README.md v1.0

---

## ⚠️ Архитектурная рецензия текущего README

Прочитав текущий документ как Главный Архитектор EPP, я фиксирую следующие **архитектурные нарушения**:

### 1. Event-native архитектура отсутствует

Текущий дизайн — **RPC-native** (MCP — это JSON-RPC). Вся коммуникация строится на request-response:
- OpenCode → MCP Server (RPC call)
- MCP Server → OPA (HTTP query)
- MCP Server → Audit (file write)

**Это нарушает принцип Event-native architecture.** В архитектуре нет событий как первоклассных граждан. Нет event bus, нет event sourcing, нет реактивной композиции.

### 2. Provenance — это bolt-on, не первоклассная система

Audit pipeline — JSONL-файл, который пишется **после** выполнения. Нет:
- Provenance chain как first-class entity
- Хэш-связывания событий (Merkle DAG)
- Возможности воспроизвести решение из событий
- Детерминированного воспроизведения (replay)

**Это нарушает принцип Provenance-first systems.**

### 3. Semantic contamination of execution layer

User Intent Layer упомянут в Stage 2 как отдельная фича, но семантика уже просачивается в execution layer:
- `input.user_intent == "modify"` в Rego-политиках
- Semantic validation как tool call, а не как слой

**Это нарушает принцип строгого разделения runtime / semantic layer.**

### 4. Operational concern leaking into semantic ontology

Capability модель использует физические пути (`/src/**`, `api.github.com:443`) как семантические атомы. Это операционные детали, привязанные к конкретной инфраструктуре.

**Должно быть:** семантическая capability (`read_source_code`, `deploy_artifact`), которая маппится в операционные права на уровне рантайма.

### 5. Primitive-first нарушен

Текущий дизайн определяет примитивы как:
- `fork() + Landlock + seccomp + cgroups` (это implementation details)
- `MCP tool` (это транспорт, не примитив)
- `OPA policy` (это механизм, не примитив)

**Настоящие примитивы должны быть:**
- `Event` — атомарный факт
- `Capability` — атомарное право
- `Decision` — allow/deny с provenance
- `ProvenanceRecord` — связанный граф решений
- `Policy` — правило, порождающее решения

### 6. Frankenstein architecture risk

Комбинация TypeScript (MCP сервер) + Rust (sandbox) + OPA/Rego + JSONL + SIEM создаёт **слишком много разнородных технологий** без чётких границ. Это типичный признак архитектурной энтропии.

---

## ♻️ Предлагаемая архитектура: CognitionKernel на Event-Native Substrate

### Принципиальное изменение

Вместо **MCP Server + OPA + Sandbox** как трёх независимых компонентов, я предлагаю:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    COGNITIONKERNEL                                   │
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    EVENT SUBSTRATE                           │   │
│  │  (Event Bus + Event Store + Event Replay)                   │   │
│  │  • Все коммуникации — события                               │   │
│  │  • Всё persistence — event store                             │   │
│  │  • Вся provenance — реконструкция из событий                │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                 ↓                                    │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    GOVERNANCE LAYER                          │   │
│  │  • Policy Decision Point (PDP) — слушает события           │   │
│  │  • Принимает решения: allow / deny / escalate              │   │
│  │  • Генерирует DecisionEvent                                │   │
│  │  • Чистый, без привязки к runtime                         │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                 ↓                                    │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    ENFORCEMENT LAYER                         │   │
│  │  • Policy Enforcement Point (PEP) — забирает решения       │   │
│  │  • Sandbox manager — runtime-механизм                       │   │
│  │  • Единственный слой, который знает про seccomp/Landlock   │   │
│  │  • Может быть заменён (Docker, Firecracker, ...)           │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                 ↓                                    │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    SEMANTIC LAYER                            │   │
│  │  • Онтология операций (читай, пиши, деплой, билд)          │   │
│  │  • Intent resolution — "что пользователь на самом деле     │   │
│  │    хочет сделать?"                                          │   │
│  │  • Semantic capability mapping: read_source → fs:read:/src  │   │
│  │  • Никогда не вызывает sandbox, не трогает syscalls        │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                 ↓                                    │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    OBSERVABILITY LAYER                       │   │
│  │  • Все события уже есть в Event Store                       │   │
│  │  • Provenance Graph — реконструируется из событий           │   │
│  │  • Audit — это проекция событий (не отдельный pipeline)     │   │
│  │  • SIEM интеграция — через event stream, не через JSONL    │   │
│  └─────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 📐 Слой 0: Event Substrate (фундамент)

### Что меняется

Вместо stdio MCP transport как основного канала — **внутренний event bus**.

```
Текущий:
OpenCode → [stdio JSON-RPC] → MCP Server

Предлагаемый:
OpenCode → [MCP transport] → Event Adapter → Event Bus
```

Event Bus — это **не внешняя система** (NATS/Kafka) на старте. Это in-process событийная шина с простым интерфейсом:

```typescript
// Примитив #1: Event
interface Event {
  id: string;                    // UUIDv7 (time-sortable)
  type: EventType;               // "tool.invocation" | "decision" | "sandbox.started" | ...
  source: string;                // "opencode.agent" | "governance.pdp" | "enforcement.pep"
  timestamp: number;             // nanosecond-precision
  data: Record<string, unknown>; // payload
  provenance: {
    parentEventId: string | null;
    traceId: string;
    chain: string[];             // ordered list of ancestor event IDs
  };
  signature?: string;            // optional: hash-chain link
}

// Event Bus — единая точка входа
interface EventBus {
  publish(event: Event): Promise<void>;
  subscribe(type: EventType, handler: EventHandler): Subscription;
  replay(from: string, to: string): AsyncIterable<Event>;
  getProvenanceGraph(eventId: string): Promise<ProvenanceGraph>;
}
```

### Почему это важно

1. **Все слои говорят на одном языке** — событиях
2. **Provenance строится автоматически** — каждый event знает своего родителя
3. **Audit — это query к Event Store**, не отдельный pipeline
4. **Replay возможен** — можно воспроизвести любое решение
5. **Sandbox не привязан к протоколу** — можно заменить на Firecracker, gVisor, Docker

### Реализация PoC

```typescript
// Не нужен NATS/Kafka на старте. Достаточно in-memory event bus.
class InMemoryEventBus implements EventBus {
  private store: Event[] = [];
  private subscribers = new Map<EventType, Set<EventHandler>>();

  async publish(event: Event): Promise<void> {
    // 1. Сохраняем
    this.store.push(event);
    // 2. Рассылаем подписчикам
    for (const handler of this.subscribers.get(event.type) ?? []) {
      await handler(event);
    }
    // 3. Если превышен лимит store — сбрасываем на диск
    if (this.store.length > 1000) this.flushToDisk();
  }

  async replay(fromId: string, toId: string): Promise<Event[]> {
    const fromIdx = this.store.findIndex(e => e.id === fromId);
    const toIdx = this.store.findIndex(e => e.id === toId);
    return this.store.slice(fromIdx, toIdx + 1);
  }

  getProvenanceGraph(eventId: string): ProvenanceGraph {
    // Реконструируем граф, проходя по parentEventId
    const graph = new ProvenanceGraph();
    let current = this.store.find(e => e.id === eventId);
    while (current) {
      graph.addNode(current);
      current = this.store.find(e => e.id === current.provenance.parentEventId);
    }
    return graph;
  }
}
```

---

## 📐 Слой 1: Governance Layer (Policy Decision Point)

### Что меняется

Вместо OPA/Rego как внешнего HTTP-сервиса — **внутренний компонент, слушающий события**.

```
Текущий:
MCP Server → HTTP → OPA (localhost:8181)

Предлагаемый:
Event Bus ──subscribe("tool.invocation")──→ Governance PDP
                                            │
                                            ├─→ Policy Engine (OPA/Rego или нативный)
                                            ├─→ Generates DecisionEvent
                                            └─→ Publishes DecisionEvent в Event Bus
```

### Примитивы

```typescript
// Примитив #2: Capability
interface Capability {
  id: string;
  semantic: string;       // "source_code.read" (семантическая)
  operational: string[];  // ["fs:read:/src/**", "fs:read:/workspace/**"] (операционные)
  constraints?: Record<string, unknown>;
}

// Примитив #3: Policy
interface Policy {
  id: string;
  name: string;
  description: string;
  rules: Rule[];
  priority: number;       // для разрешения конфликтов
}

// Примитив #4: Decision
interface Decision {
  eventId: string;
  requestEventId: string; // ссылка на событие-запрос
  result: "allow" | "deny" | "escalate";
  matchedPolicy: string;
  reasoning: string;
  capabilities: Capability[]; // разрешённые capability
  semantics?: {
    intent: string;
    confidence: number;
  };
}
```

### Почему OPA остаётся (но иначе)

OPA/Rego **не убирается**, но меняется его роль:
- OPA — это один из возможных policy engines
- Governance слой общается с OPA через внутренний интерфейс, не через HTTP
- На старте можно использовать Rego для гибкости
- В production — скомпилированные политики для производительности

```typescript
interface PolicyEngine {
  evaluate(request: PolicyRequest): Promise<Decision>;
  validate(policy: Policy): Promise<ValidationResult>;
}

// OPA implementation
class OpaPolicyEngine implements PolicyEngine {
  constructor(private opaClient: OpaClient) {}

  async evaluate(request: PolicyRequest): Promise<Decision> {
    const result = await this.opaClient.query("governance/allow", {
      action: request.action,
      resource: request.resource,
      intent: request.intent,       // semantic!
      plugin: request.plugin
    });
    return {
      eventId: generateId(),
      requestEventId: request.eventId,
      result: result.allowed ? "allow" : "deny",
      matchedPolicy: result.matched_policy,
      reasoning: result.reasoning,
      capabilities: result.capabilities
    };
  }
}
```

---

## 📐 Слой 2: Enforcement Layer (Policy Enforcement Point)

### Что меняется

Sandbox не вызывается напрямую из MCP tool. Он — подписчик событий.

```
Текущий:
MCP Tool → Sandbox.run(command)

Предлагаемый:
Event Bus ──subscribe("decision")──→ Enforcement PEP
                                      │
                                      ├─→ Sandbox Manager
                                      ├─→ Spawns isolated process
                                      ├─→ Publishes "sandbox.started" event
                                      ├─→ Publishes "sandbox.completed" event
                                      └─→ Publishes "tool.result" event
```

### Sandbox как стратегия, не как конкретная реализация

```typescript
// Sandbox Strategy — можно подменять
interface SandboxStrategy {
  name: string;                       // "linux-namespace" | "firecracker" | "docker"
  capability: string;                 // "isolation.level.6" | "isolation.level.3"
  
  spawn(command: string, caps: Capability[]): Promise<SandboxInstance>;
  
  // Проверка доступности
  checkAvailability(): Promise<boolean>;
}

// Linux-native sandbox (Landlock + seccomp + namespaces)
class LinuxNamespaceStrategy implements SandboxStrategy {
  name = "linux-namespace";
  capability = "isolation.level.6";

  async spawn(command: string, caps: Capability[]): Promise<SandboxInstance> {
    const policies = this.buildPolicies(caps);
    
    // fork + clone + Landlock + seccomp + cgroups
    const pid = await this.forkIsolated(policies);
    
    return new SandboxInstance(pid);
  }

  private buildPolicies(caps: Capability[]): SandboxPolicies {
    // Трансляция capabilities → Landlock rules
    // Трансляция capabilities → seccomp rules
    // Трансляция capabilities → cgroup limits
    return {
      landlock: this.toLandlockRules(caps),
      seccomp: this.toSeccompFilter(caps),
      cgroups: this.toCgroupLimits(caps)
    };
  }
}
```

### Ключевое изменение: separation of concerns

- **Governance слой** решает: `allow` или `deny`, и **какие** capabilities разрешены
- **Enforcement слой** знает: как транслировать capabilities в Landlock/seccomp
- Governance слой НЕ знает про Landlock
- Enforcement слой НЕ принимает решений (только исполняет)

Это **явное разделение PDP и PEP** — фундаментальный паттерн безопасности.

---

## 📐 Слой 3: Semantic Layer

### Что меняется

User Intent — не поле в Rego-политике, а **отдельный слой** между запросом и решением.

```
Текущий:
tool.call → OPA policy (intent == "modify")

Предлагаемый:
tool.call → Semantic Layer → [intent: "modify", confidence: 0.95] → Governance PDP → Decision
```

### Semantic Resolution

```typescript
class SemanticLayer {
  constructor(private ontology: Ontology) {}

  async resolveIntent(request: ToolInvocation): Promise<SemanticContext> {
    // Шаг 1: Классифицируем операцию на основе онтологии
    const operationClass = this.ontology.classifyOperation(
      request.toolName,
      request.parameters
    );
    
    // Шаг 2: Определяем семантическую capability
    const semanticCap = this.ontology.mapToSemanticCapability(
      operationClass,
      request.context
    );
    
    // Шаг 3: Маппинг в операционные capabilities
    const operationalCaps = this.ontology.mapToOperational(
      semanticCap,
      request.plugin.declaredCapabilities
    );
    
    return {
      intent: operationClass.intent,       // "read" | "modify" | "deploy" | "audit"
      confidence: operationClass.confidence,
      semanticCapability: semanticCap.id,   // "source_code.read"
      operationalCapabilities: operationalCaps
    };
  }
}
```

### Онтология как код

```yaml
# ontology/operations.yaml
ontology:
  version: "1.0"
  
  operations:
    - id: source_code.read
      intent: read
      description: "Чтение исходного кода"
      patterns:
        - tool: ["read", "glob", "grep"]
        - resource: ["**/*.ts", "**/*.rs", "**/*.py"]
      maps_to:
        - fs:read:/src/**
        - fs:read:/workspace/**
        
    - id: artifact.deploy
      intent: deploy
      description: "Деплой артефакта"
      patterns:
        - tool: ["bash", "exec"]
        - command_prefix: ["npm publish", "docker push", "deploy"]
      maps_to:
        - shell:exec:npm
        - network:connect:registry.npmjs.org:443
      required_approval: true
      audit_level: critical
```

**Этот слой — единственное место, где живёт семантика.** Он не вызывает sandbox, не трогает syscalls, не знает про seccomp.

---

## 📐 Слой 4: Observability Layer (Provenance & Audit)

### Что меняется

**Нет отдельного audit pipeline.** Audit — это query к Event Store.

```
Текущий:
Каждый компонент пишет в JSONL

Предлагаемый:
Event Store (все события) → Provenance Graph → Audit View
                           ↓
                      SIEM Stream
```

### Provenance Graph

```typescript
class ProvenanceGraph {
  private events: Map<string, Event> = new Map();
  private children: Map<string, Set<string>> = new Map(); // parent → children

  addEvent(event: Event): void {
    this.events.set(event.id, event);
    
    if (event.provenance.parentEventId) {
      const siblings = this.children.get(event.provenance.parentEventId) ?? new Set();
      siblings.add(event.id);
      this.children.set(event.provenance.parentEventId, siblings);
    }
  }

  getChain(eventId: string): Event[] {
    // Полная цепочка от корня до этого события
    const chain: Event[] = [];
    let current = this.events.get(eventId);
    while (current) {
      chain.unshift(current);
      current = this.events.get(current.provenance.parentEventId);
    }
    return chain;
  }

  getTree(eventId: string): ProvenanceTree {
    // Всё дерево решений, включая дочерние события
    return {
      root: this.events.get(eventId)!,
      children: [...(this.children.get(eventId) ?? [])].map(
        childId => this.getTree(childId)
      )
    };
  }

  generateAuditReport(filter: AuditFilter): AuditReport {
    // Audit — это просто проекция графа
    const relevantEvents = this.query(filter);
    return {
      events: relevantEvents,
      summary: this.summarize(relevantEvents),
      decisions: this.extractDecisions(relevantEvents),
      violations: this.findViolations(relevantEvents)
    };
  }
}
```

---

## 🔄 Сравнение: Текущий дизайн vs Предлагаемый

| Аспект | Текущий (README) | Предлагаемый (EPP-aligned) |
|--------|------------------|---------------------------|
| **Коммуникация** | RPC (request-response) | Event-driven (publish-subscribe) |
| **Provenance** | JSONL лог (bolt-on) | Event Store (first-class) |
| **PDP/PEP** | Смешаны в MCP Server | Разделены явно |
| **Семантика** | В Rego-политиках | Отдельный Semantic Layer |
| **Sandbox** | Одна реализация (Landlock+seccomp) | Strategy pattern (подменяемый) |
| **Capability модель** | Операционная (пути) | Семантическая + operational mapping |
| **Audit** | Отдельный pipeline | Проекция Event Store |
| **Технологии** | TypeScript + Rust + OPA + JSONL | Единый event substrate поверх |
| **Архитектурная энтропия** | Высокая (Frankenstein) | Низкая (слоистая) |
| **EPP совместимость** | Stage 3 | C рождения |

---

## 🗺️ Roadmap (EPP-aligned)

### Stage 0: Event Substrate (2 недели)

| Задача | Описание |
|--------|----------|
| Event Bus MVP | In-memory event bus с publish/subscribe/replay |
| Event Schema | Базовые event types: `tool.invocation`, `decision`, `sandbox.*` |
| MCP Adapter | Адаптер, который принимает MCP вызовы и публикует как события |
| Governance PDP MVP | Простой decision engine: allow list / deny list |
| PoC Flow | OpenCode → MCP → Event Bus → PDP → Event Bus → console output |

**Success criteria:** Весь lifecycle tool call'а проходит через события.

### Stage 1: Governance & Enforcement (4 недели)

| Задача | Описание |
|--------|----------|
| Capability система | Capability как first-class entity |
| Policy Engine | OPA/Rego интеграция через внутренний интерфейс |
| Semantic Layer MVP | Онтология базовых операций (read/write) |
| Sandbox Strategy | LinuxNamespaceStrategy (Landlock + seccomp + namespaces) |
| PEP implementation | Sandbox Manager, слушающий DecisionEvent |
| Provenance Graph | Реконструкция chain of decisions из событий |

### Stage 2: Semantic & Observability (4 недели)

| Задача | Описание |
|--------|----------|
| Полная онтология | Semantic capability mapping для всех tool типов |
| Intent Resolution | LLM-augmented определение намерения |
| Audit проекция | Генерация compliance-отчётов из Event Store |
| SIEM Stream | Event stream в стандартном формате (OCSF) |
| Policy hot-reload | Без остановки сервиса |

### Stage 3: Production Hardening (4 недели)

| Задача | Описание |
|--------|----------|
| Event Store persistence | Disk-backed с WAL |
| Multiple Sandbox Strategies | Firecracker strategy для высоконадёжной изоляции |
| Performance optimization | Sub-10ms latency for PDP |
| Web dashboard | Визуализация provenance graph |
| EPP full integration | Semantic операционный интеллект поверх |

**Общий срок:** 14 недель (сопоставимо с текущим, но с принципиально другой архитектурой)

---

## 🛡️ Ответы на anticipated objections

### "Зачем event bus? MCP — это стандарт."

MCP — это **транспортный протокол**, не архитектура. Event bus — внутри CognitionKernel. MCP остаётся как внешний интерфейс. OpenCode по-прежнему общается через MCP. Но внутри — события.

### "Event bus — это overengineering для PoC."

In-memory event bus — это **~50 строк кода** (см. пример выше). Это не Kafka. Это простой pub/sub. Overengineering — это когда у вас MCP Server + OPA HTTP + JSONL audit + Rust sandbox без единой событийной модели.

### "Rust sandbox быстрее TypeScript."

Да, но это concern enforcement layer'а. Sandbox может быть на Rust. Governance и Semantic слои — на TypeScript. Чёткие границы позволяют использовать разные языки без хаоса.

### "Semantic layer — это преждевременно."

Согласен для Stage 0. Но он должен быть **спроектирован** с самого начала как отдельный слой, даже если реализован как заглушка (pass-through). Иначе semantic concern просочится во все остальные слои.

### "А что если OpenCode уберёт MCP?"

Тогда мы теряем транспорт, но не архитектуру. Event bus остаётся. MCP Adapter заменяется на другой адаптер. Все внутренние слои не меняются.

---

## ✅ Заключение

Текущий README содержит **хорошую мотивацию** и **правильную проблемную постановку**, но предлагает архитектуру, которая:
1. Нарушает event-native принцип
2. Не имеет provenance как first-class системы
3. Смешивает семантику и рантайм
4. Создаёт риски архитектурной энтропии

Моё предложение — **не переписывать README**, а **переосмыслить архитектуру** с учётом EPP-принципов, сохранив:
- MCP как внешний транспорт (не трогаем OpenCode интеграцию)
- OPA/Rego как policy engine (но через внутренний интерфейс)
- Landlock + seccomp как механизмы изоляции (но через strategy pattern)

Ключевое изменение: **Event Bus как фундамент**, а не MCP Server как точка интеграции.

---

*Предложение готово к обсуждению. Готов защищать каждую архитектурную решение.*

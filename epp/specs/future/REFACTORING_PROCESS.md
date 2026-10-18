# Процесс рефакторинга EPP: Event-Sourced → Event-Native

> **Назначение:** Практическое руководство по проведению рефакторинга.
> **Принцип:** Каждый шаг обратим. Система остаётся working после каждого шага.
> **Основание:** REFACTORING_REFERENCE.md — архитектурные решения и first principles.

---

## 0. Структура процесса

```
Фаза 0: Подготовка (не трогаем код, только думаем и измеряем)
  ↓ утверждение плана
Фаза 1: Инфраструктура (безопасно, ничего не ломает)
  ↓ новая схема готова
Фаза 2: Ядро (один шаг за раз, каждый обратим)
  ↓ старое ядро заменено
Фаза 3: Валидация (сравнение, зачистка, документация)
```

**Жёсткое правило:** Не переходи к следующей фазе, пока не пройдены все checkpoints текущей. Никаких «сделаем потом».

---

## Фаза 0: Подготовка (Pre-Flight Checklist)

### 0.1. Карта текущего состояния (do not skip)

Прежде чем что-то менять — **задокументируй текущий flow как есть**.

Возьми `main.py` и нарисуй **один лист** (бумага/Miro/diagram), где показано:

```ascii
ТЕКУЩИЙ FLOW (as-is):
HTTP POST /api/execute
  │
  ├─ 1. Registry.lookup(tool_name) → CapabilityInfo
  │
  ├─ 2. GovernanceEvent(event_type="tool.execution.requested", ...)
  │
  ├─ 3. Cerbos.check(event) → Decision
  │     ├─ HOLD → 3a. Approval.create_request(event) → ApprovalRequest
  │     └─ ALLOW → 3b. Executor.execute(event, decision) → Result
  │
  ├─ 4. NATS.publish("epp.governance", event)
  │
  └─ 5. HTTP Response
```

На этом же листе отметь:
- **Красным:** что меняем (main.py, approval/router.py)
- **Зелёным:** что не трогаем (Cerbos, Executor, Registry, Ingest, PG, NATS client)
- **Синим:** что добавляем (подписчики, новые NATS subjects)

**Checkpoint 0.1:** Лист утверждён. Все понимают, что красное, зелёное, синее.

### 0.2. Карта целевого состояния

Нарисуй **второй лист** — целевой flow:

```ascii
ЦЕЛЕВОЙ FLOW (to-be):
HTTP POST /api/execute  ← МЕНЯЕТСЯ: теперь публикует событие и всё
  │
  └─ NATS.publish("epp.tool.invocation.requested", event)
     │
     ├── [nats.consumer: pdp-handler]
     │     └─ Cerbos.check(event) → decision
     │        ├─ HOLD → NATS.publish("epp.approval.created")
     │        └─ ALLOW → NATS.publish("epp.decision.issued")
     │
     ├── [nats.consumer: approval-handler]  ← НОВЫЙ
     │     └─ on "epp.approval.created"
     │        └─ Approval.create_request(event)
     │
     ├── [nats.consumer: pep-handler]  ← НОВЫЙ
     │     └─ on "epp.decision.issued"
     │        └─ Executor.execute(event, decision)
     │           └─ NATS.publish("epp.execution.completed")
     │
     ├── [nats.consumer: provenance-handler]  ← НОВЫЙ
     │     └─ on ANY "epp.*"
     │        └─ ProvenanceGraph.add(event)
     │
     └── [nats.consumer: semantic-handler]  ← НОВЫЙ (pass-through v0)
           └─ on "epp.tool.invocation.requested"
              └─ IntentResolver.resolve(event)
                 └─ NATS.publish("epp.intent.resolved")
```

**Checkpoint 0.2:** Целевой лист утверждён. Все подписчики, subject'ы, event types — согласованы.

### 0.3. Определи Event Types и NATS Subject'ы

Создай таблицу. Это будет **Event Schema Registry v0**:

```yaml
# Схема: каждый тип события имеет имя, кто производит, кто потребляет
events:
  - type: tool.invocation.requested
    version: 1.0
    produced_by: [api-layer, mcp-adapter]
    consumed_by: [pdp-handler, semantic-handler, provenance-handler]
    fields:
      trace_id: string
      tool_name: string
      args: object
      principal: Principal
      timestamp: int

  - type: decision.issued
    version: 1.0
    produced_by: [pdp-handler]
    consumed_by: [pep-handler, provenance-handler]
    fields:
      trace_id: string
      parent_event_id: string
      policy_decision: ALLOW | DENY | HOLD
      matched_policy: string
      reasoning: string
      capabilities: Capability[]

  - type: approval.created
    version: 1.0
    produced_by: [pdp-handler]
    consumed_by: [approval-handler, provenance-handler]
    fields:
      trace_id: string
      parent_event_id: string
      approval_id: string
      status: pending

  - type: execution.started
    version: 1.0
    produced_by: [pep-handler]
    consumed_by: [provenance-handler]
    fields:
      trace_id: string
      parent_event_id: string
      started_at: int

  - type: execution.completed
    version: 1.0
    produced_by: [pep-handler]
    consumed_by: [provenance-handler, api-layer]
    fields:
      trace_id: string
      parent_event_id: string
      result: object
      completed_at: int

  - type: intent.resolved
    version: 1.0
    produced_by: [semantic-handler]
    consumed_by: [pdp-handler, provenance-handler]
    fields:
      trace_id: string
      parent_event_id: string
      capability: string
      confidence: float
```

**Правило:** Каждый event type имеет ровно одного producer. Может иметь N consumer'ов.

**Checkpoint 0.3:** Таблица согласована. Нет событий без consumer'а. Нет consumer'а без события.

### 0.4. Характеризационные тесты (Characterization Tests)

Прежде чем менять код — **напиши тесты, которые проверяют текущее поведение**.

Эти тесты не проверяют «правильность». Они проверяют: **«поведение не изменилось»**.

```python
# tests/characterization/test_execute_flow.py

async def test_simple_allow_flow(epp_client, mocker):
    """Текущее поведение: ALLOW → execution → success.
    
    Этот тест остаётся зелёным ДО и ПОСЛЕ рефакторинга.
    Если он пожелтел — мы сломали поведение."""
    
    response = await epp_client.post("/api/execute", json={
        "tool_name": "bash",
        "args": {"command": "echo hello"},
        "principal": {"id": "user-1", "role": "developer"}
    })
    
    assert response.status_code == 200
    assert response.json()["status"] == "success"
    assert "result" in response.json()

async def test_deny_flow(epp_client, mocker):
    """Текущее поведение: DENY → error."""
    mocker.patch("app.main.cerbos.check", return_value=Decision(decision="DENY"))
    
    response = await epp_client.post("/api/execute", json={
        "tool_name": "bash",
        "args": {"command": "rm -rf /"},
        "principal": {"id": "user-1", "role": "developer"}
    })
    
    assert response.status_code == 403
    assert response.json()["status"] == "denied"

async def test_hold_flow(epp_client, mocker):
    """Текущее поведение: HOLD → approval created."""
    mocker.patch("app.main.cerbos.check", return_value=Decision(decision="HOLD"))
    
    response = await epp_client.post("/api/execute", json={...})
    
    assert response.status_code == 202
    assert response.json()["status"] == "pending_approval"
    assert "approval_id" in response.json()
```

**Сколько тестов нужно:** Минимум 3 (ALLOW, DENY, HOLD). Максимум — по одному на каждый граничный случай, который ты знаешь.

**Checkpoint 0.4:** Все characterization tests зелёные. Они проверяют **поведение**, а не реализацию.

### 0.5. Карта зависимостей слоёв EPP

Нарисуй **третью карту** — как слои связаны. Это ответ на твой вопрос: *«все уровни EPP связаны между собой»*.

```
L1 (API) ──────────────────────→ L3 (NATS) ──→ L6 (Ingest + PG)
  ↓                              ↓                ↑
  └── вызывает L2 (Registry) ────┘                │
  ↓                                                 │
  └── вызывает L4 (Cerbos) ────→ L3 (NATS) ────────┘
  ↓                              ↓
  └── вызывает L5 (Executor) ────┘
```

**Что это значит для рефакторинга:**

| Слой | Меняется? | Как |
|------|-----------|-----|
| **L1 (API)** | ✅ Да | Вместо вызова Cerbos/Executor — публикует событие |
| **L2 (Registry)** | ❌ Нет | Только читается, интерфейс не меняется |
| **L3 (NATS)** | ✅ Да | Новые subjects, новая схема |
| **L4 (Cerbos)** | ❌ Нет | Тот же gRPC вызов, но из subscriber, не из main.py |
| **L4 (Approval)** | ✅ Да | Из router → subscriber |
| **L5 (Executor)** | ⚠️ Интерфейс | Вместо return result → publish event. Логика та же |
| **L6 (Ingest)** | ❌ Нет | Всё ещё читает из NATS. Только subject names меняются |
| **L7 (Semantic)** | ✅ Новый | Новый subscriber (pass-through v0) |
| **L8 (Observability)** | ✅ Новый | Метрики из subscriber'ов |

**Checkpoint 0.5:** Карта согласована. Ты знаешь, что трогаешь, а что нет.

### 0.6. Определи порядок изменений (Dependency Order)

Из карты зависимостей следует **жёсткий порядок**:

```
Шаг 1: L3 (новые NATS subjects)    ← не зависит ни от чего
Шаг 2: L7 (Semantic subscriber)     ← зависит только от L3
Шаг 3: L4 (PDP subscriber)         ← зависит от L3 + L2 (registry)
Шаг 4: L4 (Approval subscriber)    ← зависит от L3 + L4 (PDP)
Шаг 5: L5 (PEP subscriber)         ← зависит от L3 + L4 (PDP)
Шаг 6: L1 (API → thin publisher)   ← зависит от L3
Шаг 7: Удаление старого main.py    ← зависит от шагов 3-6
```

**Правило:** Шаг N не начинается, пока шаг N-1 не прошёл все checkpoint'ы.

**Checkpoint 0.6:** Порядок утверждён. Каждый шаг имеет чёткий scope.

### 0.7. План отката (Rollback Plan)

Для каждого шага — **как откатиться, если что-то пошло не так**:

```yaml
rollback_plan:
  step_1_new_nats_subjects:
    how: "Удалить новые subjects. Старые продолжают работать."
    impact: "Нет. Старый flow не использует новые subjects."
  
  step_2_semantic_subscriber:
    how: "Остановить процесс semantic-handler."
    impact: "Нет semantic resolution. PDP продолжает работать."
  
  step_3_pdp_subscriber:
    how: "Остановить pdp-handler. Вернуть main.py к старому flow."
    impact: "Временная потеря event-native для PDP."
  
  step_4_approval_subscriber:
    how: "Остановить approval-handler. Вернуть approval в main.py."
    impact: "Временная потеря event-native для approval."
  
  step_5_pep_subscriber:
    how: "Остановить pep-handler. Вернуть executor в main.py."
    impact: "Временная потеря event-native для execution."
  
  step_6_thin_api:
    how: "Вернуть main.py к старому flow."
    impact: "Система возвращается к исходному состоянию."
  
  step_7_remove_old:
    how: "Вернуть удалённый код из git."
    impact: "Код возвращается."
```

**Checkpoint 0.7:** Для каждого шага известно, как откатиться. Откат проверен (хотя бы мысленно).

### 0.8. Критерии готовности к Фазе 1

Прежде чем писать код — проверь:

- [ ] **0.1** — Карта as-is нарисована и утверждена
- [ ] **0.2** — Карта to-be нарисована и утверждена
- [ ] **0.3** — Event Schema Registry создана (все типы, producer'ы, consumer'ы)
- [ ] **0.4** — Characterization tests написаны и зелёные
- [ ] **0.5** — Карта слоёв и зависимостей нарисована
- [ ] **0.6** — Порядок шагов утверждён
- [ ] **0.7** — План отката для каждого шага существует
- [ ] **0.8** — Ты знаешь, сколько времени займёт КАЖДЫЙ шаг, а не весь проект

---

## Фаза 1: Инфраструктура (ничего не ломает)

### 1.1. NATS Subject Schema

Создай файл `app/events/subjects.py`:

```python
# app/events/subjects.py
# Централизованное определение всех NATS subjects

class Subjects:
    # === Existing subjects (не трогаем до Фазы 2) ===
    GOVERNANCE = "epp.governance"  # старый subject, пока живёт
    
    # === New subjects (добавляем, старый код их не использует) ===
    TOOL_INVOCATION_REQUESTED = "epp.tool.invocation.requested"
    DECISION_ISSUED = "epp.decision.issued"
    APPROVAL_CREATED = "epp.approval.created"
    EXECUTION_STARTED = "epp.execution.started"
    EXECUTION_COMPLETED = "epp.execution.completed"
    INTENT_RESOLVED = "epp.intent.resolved"
```

**Проверка:** Старый код продолжает писать в `epp.governance`. Новый код пишет в новые subjects. Они не пересекаются.

### 1.2. Event Schema (Pydantic models)

Создай файл `app/events/schemas.py`:

```python
# app/events/schemas.py
from pydantic import BaseModel
from typing import Literal, Optional
from uuid import uuid4
from datetime import datetime

class ToolInvocationRequested(BaseModel):
    event_type: Literal["tool.invocation.requested"] = "tool.invocation.requested"
    event_id: str = ""
    trace_id: str
    tool_name: str
    args: dict
    principal: dict
    timestamp: int = 0
    
    def __init__(self, **data):
        super().__init__(**data)
        if not self.event_id:
            self.event_id = str(uuid4())
        if not self.timestamp:
            self.timestamp = int(datetime.now().timestamp() * 1_000_000)

class DecisionIssued(BaseModel):
    event_type: Literal["decision.issued"] = "decision.issued"
    event_id: str = ""
    trace_id: str
    parent_event_id: str
    policy_decision: Literal["ALLOW", "DENY", "HOLD"]
    matched_policy: str
    reasoning: str
    capabilities: list = []
    timestamp: int = 0

# ... остальные схемы по тому же шаблону
```

**Проверка:** Схемы созданы, но никто их ещё не использует. Просто файлы.

### 1.3. Handler Stubs (пустые подписчики)

Создай структуру `app/events/handlers/`:

```python
# app/events/handlers/__init__.py
from .pdp_handler import PdpHandler
from .pep_handler import PepHandler
from .approval_handler import ApprovalHandler
from .provenance_handler import ProvenanceHandler
from .semantic_handler import SemanticHandler

__all__ = ["PdpHandler", "PepHandler", "ApprovalHandler", 
           "ProvenanceHandler", "SemanticHandler"]
```

```python
# app/events/handlers/pdp_handler.py
# Пока заглушка. В Фазе 2 сюда переедет Cerbos check.

class PdpHandler:
    def __init__(self, nats_client, cerbos_client, registry):
        self.nats = nats_client
        self.cerbos = cerbos_client
        self.registry = registry
    
    async def start(self):
        # Пока не подписывается. Запустится в Фазе 2.
        pass
```

**Проверка:** Хендлеры созданы, но не активны. Старый flow работает как работал.

### 1.4. DLQ + Error Handling Infrastructure

Создай `app/events/dlq.py`:

```python
# app/events/dlq.py
from dataclasses import dataclass
from datetime import datetime

@dataclass
class DeadLetterEvent:
    original_event: dict
    handler_name: str
    error: str
    attempts: int
    timestamp: int
    
class DeadLetterQueue:
    def __init__(self, nats_client):
        self.nats = nats_client
    
    async def send(self, event: DeadLetterEvent):
        # Публикует в NATS subject epp.dead.letter
        await self.nats.publish("epp.dead.letter", event.json())
    
    async def retry(self, dlq_event_id: str):
        # Re-publish оригинальное событие
        pass
```

**Проверка:** DLQ создан, но никто в него не шлёт. Просто инфраструктура.

### 1.5. Checkpoint Фазы 1

- [ ] **1.1** — NATS subjects определены (старые + новые)
- [ ] **1.2** — Pydantic схемы для всех event types созданы
- [ ] **1.3** — Handler stubs созданы, но не активны
- [ ] **1.4** — DLQ инфраструктура готова
- [ ] **Старый flow продолжает работать** (characterization tests зелёные)
- [ ] **Ничего не сломано** (мы ничего не включили)

---

## Фаза 2: Ядро (один шаг за раз, каждый обратим)

**Главное правило Фазы 2:** Каждый шаг — это отдельный PR / коммит. Если шаг сломал тесты — откатываешься и исправляешь, прежде чем идти дальше.

### Шаг 2.1: Semantic Handler (pass-through v0)

**Что делаешь:** Активируешь semantic-handler как подписчика на `epp.tool.invocation.requested`.

```python
# app/events/handlers/semantic_handler.py

class SemanticHandler:
    async def start(self):
        # Подписываемся на новые события
        # Старый flow пока не использует этот subject, 
        # так что никто не публикует сюда
        await self.nats.subscribe(
            Subjects.TOOL_INVOCATION_REQUESTED,
            cb=self.handle
        )
    
    async def handle(self, msg):
        event = ToolInvocationRequested.parse_raw(msg.data)
        
        # Pass-through v0: публикуем intent.resolved с заглушкой
        intent_event = IntentResolved(
            trace_id=event.trace_id,
            parent_event_id=event.event_id,
            capability="unknown",
            confidence=0.0
        )
        
        await self.nats.publish(
            Subjects.INTENT_RESOLVED,
            intent_event.json()
        )
        await msg.ack()
```

**Проверка:**
- Semantic handler запущен, но никто не публикует в его subject
- Тесты зелёные
- Откат: остановить процесс semantic-handler

**Checkpoint 2.1:** Semantic handler работает вхолостую. Старый flow не затронут.

### Шаг 2.2: PDP Handler (Cerbos check из main.py → subscriber)

**Что делаешь:** Создаёшь активного подписчика на `epp.tool.invocation.requested`, который вызывает Cerbos и публикует `epp.decision.issued`.

```python
# app/events/handlers/pdp_handler.py

class PdpHandler:
    async def start(self):
        await self.nats.subscribe(
            Subjects.TOOL_INVOCATION_REQUESTED,
            cb=self.handle
        )
    
    async def handle(self, msg):
        event = ToolInvocationRequested.parse_raw(msg.data)
        
        # Cerbos check (тот же код, что был в main.py)
        capability = await self.registry.lookup(event.tool_name)
        governance_event = self._build_governance_event(event, capability)
        decision = await self.cerbos.check(governance_event)
        
        # Публикуем решение
        decision_event = DecisionIssued(
            trace_id=event.trace_id,
            parent_event_id=event.event_id,
            policy_decision=decision.decision,
            matched_policy=decision.policy_id,
            reasoning=decision.reasoning,
            capabilities=decision.capabilities
        )
        
        await self.nats.publish(
            Subjects.DECISION_ISSUED,
            decision_event.json()
        )
        await msg.ack()
```

**Важно:** На этом шаге **никто ещё не публикует** в `epp.tool.invocation.requested`. Только semantic handler (pass-through) и PDP handler подписались, но publisher'ов нет. Это безопасно.

### Шаг 2.3: Approval Handler

**Что делаешь:** Создаёшь подписчика на `epp.decision.issued`, который обрабатывает HOLD.

```python
# app/events/handlers/approval_handler.py

class ApprovalHandler:
    async def start(self):
        await self.nats.subscribe(
            Subjects.DECISION_ISSUED,
            cb=self.handle
        )
    
    async def handle(self, msg):
        event = DecisionIssued.parse_raw(msg.data)
        
        if event.policy_decision != "HOLD":
            await msg.ack()
            return
        
        # Создаём approval request (тот же код из approval/router.py)
        approval = await self.approval.create_request(
            trace_id=event.trace_id,
            parent_event_id=event.event_id
        )
        
        await self.nats.publish(
            Subjects.APPROVAL_CREATED,
            ApprovalCreated(
                trace_id=event.trace_id,
                parent_event_id=event.event_id,
                approval_id=approval.approval_id,
                status="pending"
            ).json()
        )
        await msg.ack()
```

### Шаг 2.4: PEP Handler (Executor)

**Что делаешь:** Создаёшь подписчика на `epp.decision.issued`, который исполняет tool при ALLOW.

```python
# app/events/handlers/pep_handler.py

class PepHandler:
    async def start(self):
        await self.nats.subscribe(
            Subjects.DECISION_ISSUED,
            cb=self.handle
        )
    
    async def handle(self, msg):
        event = DecisionIssued.parse_raw(msg.data)
        
        if event.policy_decision != "ALLOW":
            await msg.ack()
            return
        
        # Публикуем execution.started
        await self.nats.publish(
            Subjects.EXECUTION_STARTED,
            ExecutionStarted(
                trace_id=event.trace_id,
                parent_event_id=event.event_id
            ).json()
        )
        
        # Executor (тот же код из main.py, но возвращает результат через событие)
        try:
            result = await self.executor.execute(event)
            await self.nats.publish(
                Subjects.EXECUTION_COMPLETED,
                ExecutionCompleted(
                    trace_id=event.trace_id,
                    parent_event_id=event.event_id,
                    result=result
                ).json()
            )
        except Exception as e:
            await self.nats.publish(
                Subjects.EXECUTION_COMPLETED,
                ExecutionCompleted(
                    trace_id=event.trace_id,
                    parent_event_id=event.event_id,
                    error=str(e)
                ).json()
            )
        
        await msg.ack()
```

### Шаг 2.5: Provenance Handler

**Что делаешь:** Создаёшь подписчика на **все** новые subjects, который строит Provenance Graph.

```python
# app/events/handlers/provenance_handler.py

class ProvenanceHandler:
    async def start(self):
        # Подписывается на все epp.* subjects
        # Использует NATS JetStream consumer для ordered delivery
        for subject in [
            Subjects.TOOL_INVOCATION_REQUESTED,
            Subjects.DECISION_ISSUED,
            Subjects.APPROVAL_CREATED,
            Subjects.EXECUTION_STARTED,
            Subjects.EXECUTION_COMPLETED,
            Subjects.INTENT_RESOLVED,
        ]:
            await self.nats.subscribe(subject, cb=self.handle)
    
    async def handle(self, msg):
        event = GovernanceEvent.parse_raw(msg.data)
        
        # Добавляем в Provenance Graph (in-memory + batch write)
        self.graph.add_event(event)
        
        # Каждые 100 событий — пишем в PG (или через Ingest)
        if len(self.graph.events) >= 100:
            await self.flush_to_pg()
        
        await msg.ack()
```

### Шаг 2.6: API Layer (main.py → thin publisher)

**Что делаешь:** Меняешь `main.py` — он больше не вызывает Cerbos, не исполняет, не создаёт approval. Он только публикует событие.

```python
# main.py — ПОСЛЕ рефакторинга (~30 строк вместо ~300)

@router.post("/execute")
async def execute_tool(request: ExecuteRequest):
    event = ToolInvocationRequested(
        trace_id=str(uuid4()),
        tool_name=request.tool_name,
        args=request.args,
        principal=request.principal.dict()
    )
    
    # Единственная ответственность: опубликовать событие
    await nats.publish(Subjects.TOOL_INVOCATION_REQUESTED, event.json())
    
    # Возвращаем trace_id — клиент может подписаться на результат
    return {"status": "accepted", "trace_id": event.trace_id}
```

**Важно:** Теперь **есть publisher** для `epp.tool.invocation.requested`. Все подписчики (PDP, Semantic, Provenance) получают событие. Цепочка замкнулась.

### Шаг 2.7: Удаление старого кода

**Что делаешь:** Удаляешь старый код из main.py (Cerbos check, approval creation, execution). Удаляешь approval/router.py (теперь это approval-handler subscriber).

```python
# Удаляем из main.py:
- # cerbos.check(...) — теперь в pdp-handler
- # approval.create_request(...) — теперь в approval-handler
- # executor.execute(...) — теперь в pep-handler
- # NATS.publish("epp.governance", ...) — заменено на новые subjects
```

**Проверка:** Старый NATS subject `epp.governance` больше никто не использует. Можно оставить для обратной совместимости или удалить.

### Checkpoint Фазы 2

- [ ] **2.1** — Semantic handler активен (pass-through)
- [ ] **2.2** — PDP handler активен (Cerbos из subscriber)
- [ ] **2.3** — Approval handler активен (HOLD flow)
- [ ] **2.4** — PEP handler активен (Execution из subscriber)
- [ ] **2.5** — Provenance handler активен (строит граф)
- [ ] **2.6** — API → thin publisher (публикует событие, ничего больше)
- [ ] **2.7** — Старый код удалён (main.py чист, approval/router.py удалён)
- [ ] **Characterization tests зелёные** (поведение не изменилось)
- [ ] **Возможность отката:** `git revert` последнего коммита возвращает старый flow

---

## Фаза 3: Валидация и зачистка

### 3.1. Сравнение поведения

Запусти characterization tests на **обоих** flow:
1. Старый flow (если оставил для обратной совместимости): тесты зелёные
2. Новый flow: тесты зелёные

Если тесты идентичны — поведение сохранено. Поздравляю, рефакторинг успешен.

### 3.2. Behaviour Replay (проверка)

Запусти replay на исторических данных через новых подписчиков:

```python
# replay через новых подписчиков
# Берём события из PG, подаём их на вход PDP/ PEP/ Semantic handler'ам
# Сравниваем решения с оригинальными

async def replay_events(from_date: str, to_date: str):
    events = await pg.query(
        "SELECT * FROM events WHERE timestamp BETWEEN $1 AND $2",
        from_date, to_date
    )
    
    for event in events:
        # Подаём событие в PDP handler как будто оно только что пришло
        await pdp_handler.handle(NatsMsg(data=event.json()))
        # Проверяем, что решение совпадает с оригинальным
```

### 3.3. Удаление старого NATS subject (опционально)

Если `epp.governance` больше никто не использует — удали consumer. Если есть внешние системы, которые в него пишут — оставь как deprecated.

### 3.4. Документация

Обнови:
- `ARCHITECTURE.md` — новый event flow, новые компоненты
- `README.md` — если нужно
- ADR (Architecture Decision Records) — запиши решения, принятые в процессе
- REFACTORING_REFERENCE.md — добавь секцию "Итоги рефакторинга"

### Checkpoint Фазы 3

- [ ] **3.1** — Behaviour comparison пройден (старый = новый)
- [ ] **3.2** — Behaviour replay работает
- [ ] **3.3** — Старый subject удалён (или объявлен deprecated)
- [ ] **3.4** — Документация обновлена
- [ ] **Можно открывать шампанское**

---

## Dependency Map: что с чем связано

```
L1 (API) ──────────────→ публикует: tool.invocation.requested
                           ↓
L3 (NATS) ←───────────── подписчики:
                           ├── PDP → публикует: decision.issued
                           ├── Semantic → публикует: intent.resolved
                           ├── Approval → публикует: approval.created
                           ├── PEP → публикует: execution.started/completed
                           └── Provenance → подписан на всё

Каждый подписчик:
  └── знает свои event types
  └── не знает другие подписчики
  └── не знает L1 (API)
  └── знает только L3 (NATS) + свои зависимости (Cerbos, Executor и т.д.)
```

**Золотое правило:** Если подписчику A нужно узнать результат подписчика B — он не вызывает B напрямую. Он подписывается на событие, которое B публикует. Никаких прямых вызовов между подписчиками.

---

## Шпаргалка: что делать, если...

| Ситуация | Действие |
|----------|----------|
| **Шаг сломал characterization test** | Откати шаг. Исправь. Перезапусти. |
| **Нужно добавить новый event type** | 1. Добавь в Event Schema Registry 2. Создай Pydantic модель 3. Создай/обнови подписчика |
| **Подписчик упал с ошибкой** | Событие идёт в DLQ. Retry. Если retry не помог — alert. |
| **Новый flow медленнее старого** | Измерь. Если >10% — профилируй. Обычно event-native быстрее за счёт параллелизма. |
| **Внешняя система пишет в старый subject** | Оставь `epp.governance` как deprecated. Добавь bridge, который пересылает в новые subjects. |
| **Хочешь добавить подписчика между PDP и PEP** | Создай подписчика на `epp.decision.issued`. Не меняй PDP. Не меняй PEP. |

---

## Принцип, которым нельзя поступаться

> **Каждый шаг — рабочий.**
> 
> Если после шага система не работает — шаг неправильный.
> Не «мы закоммитим и починим потом». Откатывайся и делай иначе.

Это единственный способ сделать рефакторинг безболезненным.

# Тестирование EPP: First Principles декомпозиция

> **Назначение:** Разложить тесты на атомы — их типы, назначение, что у нас есть, что нужно для рефакторинга.
> **Контекст:** REFACTORING_REFERENCE.md + REFACTORING_PROCESS.md + actual code `/home/msi/epp/`

---

## Атом 0: Что такое тест?

```
Тест = Утверждение (assertion) о поведении системы
        │
        ├── в определённых условиях (given)
        ├── при определённом действии (when)
        └── с ожидаемым результатом (then)
```

**Любой тест, независимо от типа, проверяет ровно одно утверждение.** Если тест проверяет два утверждения — это два теста, склеенных в одном.

---

## Атом 1: Измерения тестов (Dimensions)

У каждого теста есть **4 независимых измерения**:

### Измерение A: Scope (что покрывает)

| Scope | Что тестирует | Пример в EPP |
|-------|--------------|--------------|
| **Unit** | Одну функцию/класс изолированно | `CerbosClient.check()` с моком gRPC |
| **Integration** | Взаимодействие 2-3 компонентов | PdpHandler + NATS + Cerbos |
| **System** | Весь EPP как единое целое | POST /events/tool-call → обработка → response |
| **E2E** | EPP + внешние системы | EPP → OpenCode агент → результат |

### Измерение B: Nature (что проверяет)

| Nature | Что проверяет | Пример |
|--------|--------------|--------|
| **Behavior** | Система делает правильные действия | `ALLOW` → execution запущен |
| **State** | Состояние после операции | ProvenanceGraph содержит `execution.completed` |
| **Contract** | Интерфейс соответствует спецификации | PdpHandler принимает `ToolInvocationRequested`, возвращает `DecisionIssued` |
| **Property** | Инвариант выполняется для всех случаев | Для любого DENY: execution НЕ запущен |

### Измерение C: Purpose (цель)

| Purpose | Когда пишется | Когда исполняется | Пример |
|---------|--------------|-------------------|--------|
| **Development** | До/во время написания кода | Каждый коммит | TDD |
| **Regression** | После фикса бага | Каждый коммит | «Баг #42: DENY не возвращает 403» |
| **Characterization** | **До рефакторинга** | Каждый шаг рефакторинга | Фиксация as-is поведения |
| **Exploration** | Когда поведение неясно | Один раз | «Что если NATS недоступен?» |

### Измерение D: Longevity (время жизни)

| Longevity | Когда удаляется | Пример |
|-----------|----------------|--------|
| **Permanent** | Никогда | Unit-тесты, Contract-тесты |
| **Temporary** | После завершения рефакторинга | Characterization tests старого поведения |
| **One-shot** | Сразу после исполнения | Exploration-тест |

---

## Атом 2: Таксономия тестов (полная карта)

```
ТЕСТЫ EPP
│
├── Unit Tests (Scope: Unit, Nature: Behavior/State, Purpose: Development, Longevity: Permanent)
│   ├── CerbosClient: gRPC вызов
│   ├── Executor: запуск процесса
│   ├── Approval Store: create/approve/reject
│   ├── Registry: lookup capability
│   ├── Delegation Store: create/validate/revoke
│   └── Event Schemas: сериализация/десериализация
│
├── Integration Tests (Scope: Integration, Nature: Behavior/Contract, Purpose: Development, Longevity: Permanent)
│   ├── PdpHandler + NATS: событие → решение
│   ├── PepHandler + Executor: решение → исполнение
│   ├── ApprovalHandler + Store: HOLD → approval
│   ├── SemanticHandler: запрос → intent
│   └── ProvenanceHandler: события → граф
│
├── Characterization Tests (Scope: System, Nature: Behavior, Purpose: Characterization, Longevity: TEMPORARY)
│   ├── По одному на каждый observable outcome main.py
│   └── См. Атом 4 — точный список
│
├── Contract Tests (Scope: Integration, Nature: Contract, Purpose: Regression, Longevity: Permanent)
│   ├── Event Schema Contract
│   ├── NATS Subject Contract
│   └── API Contract
│
├── Property-Based Tests (Scope: Unit/Integration, Nature: Property, Purpose: Regression, Longevity: Permanent)
│   ├── DENY → execution не запущен
│   ├── ALLOW → execution запущен
│   ├── HOLD → approval создан
│   └── Provenance DAG: parent_event_id всегда есть (кроме root)
│
├── Existing Tests (все, что уже есть — остаются как regression)
│   └── Ничего не удаляем, ничего не переписываем
│
├── Performance Tests (Scope: System, Nature: Behavior, Purpose: Exploration, Longevity: Permanent)
│
└── Smoke Tests (Scope: System, Nature: Behavior, Purpose: Regression, Longevity: Permanent)
    ├── GET /health → 200
    └── NATS check
```

---

## Атом 3: Что есть сейчас в EPP (по реальному коду)

### Unit Tests (`/home/msi/epp/tests/unit/`)
```
✅ approval/          — ~20 тестов на StubApprovalStore
✅ approval_router/   — ~10 тестов на approval API
✅ epp_core_router/   — ~25 тестов на main.py flow
✅ policy/            — ~12 тестов на PolicyClient
✅ registry/          — ~20 тестов на Registry
✅ delegation_router/ — тесты на delegation API
✅ delegation_store/  — тесты на DelegationStore
✅ execution/         — тесты на StubExecutor
✅ governance/        — ~15 тестов на модели
✅ nats/              — тесты на NATS publisher
✅ escalation/        — тесты на EscalationStore
✅ l7/                — тесты на L7 Semantic Runtime
✅ ingest/            — тесты на Ingest
✅ admin_router/      — тесты на admin API
✅ notification/      — тесты на Notification
✅ provenance/        — тесты на Provenance
```

### Integration Tests (`/home/msi/epp/tests/integration/`)

Существующие тесты покрывают следующие сценарии POST /events/tool-call:

| Тест | Что проверяет | Статус |
|------|--------------|--------|
| `test_allow_returns_decision` | ALLOW → 200, decision=ALLOW, trace_id совпадает | ✅ |
| `test_allow_with_multiple_tools` | ALLOW для 3 разных tool | ✅ |
| `test_nonexistent_tool_returns_404` | Неизвестный tool → 404 | ✅ |
| `test_empty_tool_name_returns_400` | Пустой tool_name → 400 | ✅ |
| `test_missing_trace_id_returns_400` | Нет trace_id → 400 | ✅ |
| `test_missing_agent_id_returns_400` | Нет agent_id → 400 | ✅ |
| `test_health_returns_ok` | GET /health → 200 | ✅ |
| `test_hold_returns_hold_with_hold_id` | HOLD → 200, decision=HOLD, hold_id | ✅ |
| `test_hold_creates_approval_request` | HOLD → approval создан в БД | ✅ |
| `test_hold_writes_provenance` | HOLD → событие в NATS | ✅ |
| `test_deny_returns_deny` | DENY → 200, decision=DENY | ✅ |
| `test_deny_writes_provenance` | DENY → событие в NATS | ✅ |
| `test_tool_not_found_returns_404` | Неизвестный tool → 404 (TestClient) | ✅ |
| `test_tool_not_mapped_returns_404` | Tool без capability → 404 | ✅ |
| `test_policy_unavailable_returns_deny` | Cerbos down → 200, decision=DENY | ✅ |
| `test_policy_unavailable_writes_provenance` | Cerbos down → NATS event | ✅ |

**Что НЕ покрыто существующими тестами:**

| Сценарий | Почему важен |
|----------|-------------|
| NATS publish → ServiceUnavailableError → HTTP 503 | Отказ инфраструктуры |
| Approval.create_request → ServiceUnavailableError → HTTP 503 | HOLD при отказе approval |
| Delegation store → ServiceUnavailableError → HTTP 503 | Отказ delegation |
| Invalid delegation token → HTTP 403 | Безопасность |
| Delegation capability mismatch → HTTP 403 | Безопасность |
| Unhandled exception → HTTP 500 | Аварийный сценарий |

---

## Атом 4: Реальный decision tree main.py (точный)

На основе кода `/home/msi/epp/app/main.py` (строки 351-479):

```
POST /events/tool-call
│
├── Шаг 0: FastAPI Validation
│   └── RequestValidationError (Pydantic) → HTTP 400
│
├── Шаг 1: Registry.lookup_capability(tool_name)
│   ├── CapabilityNotMappedError → HTTP 404
│   │   { "error": "HTTPException", "detail": "Tool 'X' not mapped to capability" }
│   ├── NotFoundError → HTTP 404
│   │   { "error": "HTTPException", "detail": "Tool 'X' not found" }
│   └── OK → cap_info
│
├── Шаг 2: GovernanceEvent construction (всегда OK)
│
├── Шаг 3: Delegation check (только если delegation_token передан)
│   ├── delegation_store.validate_token(token)
│   │   ├── ServiceUnavailableError → HTTP 503
│   │   │   { "error": "HTTPException", "detail": "Delegation store unavailable" }
│   │   ├── token is None → HTTP 403
│   │   │   { "error": "HTTPException", "detail": "Invalid, expired, or revoked..." }
│   │   └── OK → token_data
│   ├── token.capability != cap_info.capability → HTTP 403
│   │   { "detail": "Delegation token capability 'X' does not match..." }
│   └── OK → delegation_attrs
│
├── Шаг 4: Policy.check_policy(event, delegation_attrs)
│   ├── ServiceUnavailableError → fail-closed:
│   │   HTTP 200, decision=DENY, reason="Policy engine unavailable..."
│   └── OK → decision { ALLOW | DENY | HOLD }
│
├── Шаг 5: NATS.publish(event, raise_on_error=True)
│   ├── ServiceUnavailableError → HTTP 503
│   │   { "error": "HTTPException", "detail": "NATS unavailable. Request rejected." }
│   └── OK
│
├── Шаг 6: Dispatch
│   ├── decision="ALLOW" → executor.execute()
│   │   └── HTTP 200, decision=ALLOW
│   │
│   ├── decision="HOLD" → approval.create_request(event)
│   │   ├── ServiceUnavailableError → HTTP 503
│   │   │   { "detail": str(e) }
│   │   └── OK → HTTP 200, decision=HOLD, hold_id="appr_..."
│   │
│   └── decision="DENY" → HTTP 200, decision=DENY
│
└── Шаг 99: Unhandled Exception → HTTP 500
    { "error": "InternalError", "detail": "Internal server error" }
```

---

## Атом 5: Полный список characterization tests (14 тестов)

Каждый тест — **один наблюдаемый outcome** из decision tree.

```python
# tests/characterization/test_main_py_behavior.py
# TEMPORARY: удалить после завершения рефакторинга

class TestMainPyCharacterization:
    """Фиксирует ТЕКУЩЕЕ поведение POST /events/tool-call.
    
    Не проверяет «правильность». Проверяет: «поведение не изменилось».
    """
    
    # ─── Happy path: ALLOW ───────────────────────────────────────
    
    def test_allow_returns_200_decision_allow(self, client, registry, mocker):
        """Leaf: executor.execute() → HTTP 200, decision=ALLOW"""
        mocker.patch("app.main.get_policy", ...decision="ALLOW")
        response = client.post("/events/tool-call", json=make_request(...))
        
        assert response.status_code == 200
        assert response.json()["decision"] == "ALLOW"
        assert response.json()["trace_id"] == request["trace_id"]
    
    # ─── Happy path: DENY ────────────────────────────────────────
    
    def test_deny_returns_200_decision_deny(self, client, registry, mocker):
        """Leaf: policy DENY → HTTP 200, decision=DENY"""
        mocker.patch("app.main.get_policy", ...decision="DENY")
        response = client.post("/events/tool-call", json=make_request(...))
        
        assert response.status_code == 200
        assert response.json()["decision"] == "DENY"
    
    # ─── Happy path: HOLD ────────────────────────────────────────
    
    def test_hold_returns_200_with_hold_id(self, client, registry, mocker):
        """Leaf: approval.create_request() → HTTP 200, decision=HOLD, hold_id"""
        mocker.patch("app.main.get_policy", ...decision="HOLD")
        response = client.post("/events/tool-call", json=make_request(...))
        
        assert response.status_code == 200
        assert response.json()["decision"] == "HOLD"
        assert response.json()["hold_id"].startswith("appr_")
    
    # ─── Fail-closed: Cerbos unavailable ─────────────────────────
    
    def test_policy_unavailable_fail_closed(self, client, registry, mocker):
        """Leaf: Policy.check_policy → ServiceUnavailableError → HTTP 200, decision=DENY"""
        mocker.patch("app.main.get_policy", ...side_effect=ServiceUnavailableError())
        response = client.post("/events/tool-call", json=make_request(...))
        
        assert response.status_code == 200  # !!! Не 503, а 200
        assert response.json()["decision"] == "DENY"
        assert "unavailable" in response.json()["reason"]
    
    # ─── NATS unavailable ────────────────────────────────────────
    
    def test_nats_unavailable_returns_503(self, client, registry, mocker):
        """Leaf: NATS.publish → ServiceUnavailableError → HTTP 503"""
        mocker.patch("app.main.get_nats", ...side_effect=ServiceUnavailableError())
        response = client.post("/events/tool-call", json=make_request(...))
        
        assert response.status_code == 503
        assert "NATS" in response.json()["detail"]
    
    # ─── Approval store unavailable (on HOLD) ────────────────────
    
    def test_approval_unavailable_returns_503(self, client, registry, mocker):
        """Leaf: approval.create_request → ServiceUnavailableError → HTTP 503"""
        mocker.patch("app.main.get_policy", ...decision="HOLD")
        mocker.patch("app.main.get_approval", ...side_effect=ServiceUnavailableError())
        response = client.post("/events/tool-call", json=make_request(...))
        
        assert response.status_code == 503
    
    # ─── Registry: tool not found ────────────────────────────────
    
    def test_tool_not_found_returns_404(self, client, mocker):
        """Leaf: Registry.lookup → NotFoundError → HTTP 404"""
        mocker.patch("app.main.get_registry", ...side_effect=NotFoundError())
        response = client.post("/events/tool-call", json=make_request(...))
        
        assert response.status_code == 404
    
    def test_tool_not_mapped_returns_404(self, client, mocker):
        """Leaf: Registry.lookup → CapabilityNotMappedError → HTTP 404"""
        mocker.patch("app.main.get_registry", ...side_effect=CapabilityNotMappedError())
        response = client.post("/events/tool-call", json=make_request(...))
        
        assert response.status_code == 404
    
    # ─── Delegation failures ─────────────────────────────────────
    
    def test_delegation_invalid_token_returns_403(self, client, registry, mocker):
        """Leaf: delegation token is None → HTTP 403"""
        mocker.patch("app.main.get_delegation", ...validate_token=None)
        response = client.post("/events/tool-call", json={
            **make_request(...), "delegation_token": "bad"
        })
        
        assert response.status_code == 403
        assert "Invalid" in response.json()["detail"]
    
    def test_delegation_capability_mismatch_returns_403(self, client, registry, mocker):
        """Leaf: token capability != requested capability → HTTP 403"""
        mocker.patch("app.main.get_delegation", ...validate_token={
            "capability": "wrong.capability"
        })
        response = client.post("/events/tool-call", json={
            **make_request(...), "delegation_token": "tok"
        })
        
        assert response.status_code == 403
        assert "does not match" in response.json()["detail"]
    
    def test_delegation_store_unavailable_returns_503(self, client, registry, mocker):
        """Leaf: delegation_store.validate_token → ServiceUnavailableError → HTTP 503"""
        mocker.patch("app.main.get_delegation", ...side_effect=ServiceUnavailableError())
        response = client.post("/events/tool-call", json={
            **make_request(...), "delegation_token": "tok"
        })
        
        assert response.status_code == 503
        assert "Delegation store" in response.json()["detail"]
    
    # ─── Validation ──────────────────────────────────────────────
    
    def test_validation_error_returns_400(self, client):
        """Leaf: Pydantic validation → HTTP 400"""
        response = client.post("/events/tool-call", json={
            "tool_name": "",  # empty = invalid
            "args": {},
            "agent_id": "",
            "trace_id": "",
        })
        assert response.status_code == 400
    
    # ─── Unhandled exception ─────────────────────────────────────
    
    def test_unhandled_exception_returns_500(self, client, mocker):
        """Leaf: unhandled Exception → HTTP 500"""
        mocker.patch("app.main.get_registry", side_effect=Exception("boom"))
        response = client.post("/events/tool-call", json=make_request(...))
        
        assert response.status_code == 500
        assert response.json()["error"] == "InternalError"
    
    # ─── Health check (не refactoring, но поведение) ─────────────
    
    def test_health_returns_200(self, client):
        """GET /health → 200"""
        response = client.get("/health")
        assert response.status_code == 200
        assert response.json()["status"] == "healthy"
```

### Итого: 14 characterization tests

| # | Outcome | HTTP Status | Что проверяет |
|---|---------|-------------|---------------|
| 1 | ALLOW | 200 | decision=ALLOW, trace_id |
| 2 | DENY | 200 | decision=DENY |
| 3 | HOLD | 200 | decision=HOLD, hold_id |
| 4 | Fail-closed | 200 | Cerbos down → decision=DENY, reason |
| 5 | NATS error | 503 | NATS publish error |
| 6 | Approval error | 503 | Approval store error on HOLD |
| 7 | Tool not found | 404 | NotFoundError |
| 8 | Tool not mapped | 404 | CapabilityNotMappedError |
| 9 | Invalid delegation | 403 | Token is None |
| 10 | Capability mismatch | 403 | Token capability != request |
| 11 | Delegation store error | 503 | Delegation store unavailable |
| 12 | Validation error | 400 | Pydantic validation |
| 13 | Internal error | 500 | Unhandled exception |
| 14 | Health check | 200 | GET /health |

---

## Атом 6: Contract Tests — развёрнутая спецификация

### 6.1. PDP Handler Contract

```
Component: PDP Handler (будущий subscriber)
Input:  NATS subject "epp.tool.invocation.requested", schema ToolInvocationRequested
Output: NATS subject "epp.decision.issued", schema DecisionIssued
```

**Structural Contract (4 теста):**

```python
class TestPdpHandlerStructuralContract:
    
    def test_event_type_is_decision_issued(self, event):
        """Тип события строго определён."""
        assert event.event_type == "decision.issued"
    
    def test_required_fields_present(self, event):
        """Все обязательные поля присутствуют."""
        assert hasattr(event, "trace_id")
        assert hasattr(event, "parent_event_id")
        assert hasattr(event, "policy_decision")
        assert hasattr(event, "matched_policy")
        assert hasattr(event, "reasoning")
    
    def test_field_types_correct(self, event):
        """Типы полей строго соблюдены."""
        assert isinstance(event.trace_id, str)
        assert isinstance(event.parent_event_id, str)
        assert event.policy_decision in ("ALLOW", "DENY", "HOLD")
        assert isinstance(event.reasoning, str)
    
    def test_no_extra_unexpected_fields(self, event):
        """Нет полей, не предусмотренных схемой."""
        expected = {"trace_id", "parent_event_id", "policy_decision",
                    "matched_policy", "reasoning", "capabilities",
                    "event_id", "event_type", "timestamp"}
        actual = set(event.dict(exclude_none=True).keys())
        assert actual <= expected
```

**Behavioral Contract (3 теста):**

```python
class TestPdpHandlerBehavioralContract:
    
    def test_one_input_produces_one_output(self, nats, pdp_handler):
        """На каждый вход — ровно один выход."""
        events = []
        async def collect(msg):
            events.append(msg)
        
        await nats.subscribe("epp.decision.issued", cb=collect)
        await pdp_handler.start()
        await nats.publish("epp.tool.invocation.requested", valid_event.json())
        await asyncio.sleep(0.1)
        
        assert len(events) == 1
    
    def test_invalid_event_goes_to_dlq(self, nats, pdp_handler):
        """Невалидное событие → не падает, а идёт в DLQ."""
        dlq_events = []
        async def collect_dlq(msg):
            dlq_events.append(msg)
        
        await nats.subscribe("epp.dead.letter", cb=collect_dlq)
        await pdp_handler.start()
        await nats.publish("epp.tool.invocation.requested", b"invalid json")
        await asyncio.sleep(0.1)
        
        assert len(dlq_events) == 1
    
    def test_latency_within_bounds(self, nats, pdp_handler):
        """99% событий обрабатываются быстрее 500ms."""
        latencies = []
        for _ in range(100):
            start = time.monotonic()
            await nats.publish("epp.tool.invocation.requested", event.json())
            await wait_for_event("epp.decision.issued", timeout=5.0)
            latencies.append((time.monotonic() - start) * 1000)
        
        p99 = sorted(latencies)[99]
        assert p99 < 500
```

**Identity Contract (3 теста):**

```python
class TestPdpHandlerIdentityContract:
    """Самый важный контракт для provenance chain."""
    
    def test_output_trace_id_equals_input_trace_id(self, nats, pdp_handler):
        """trace_id сквозит через весь pipeline."""
        input_event = ToolInvocationRequested(trace_id="trace-42", ...)
        
        await nats.publish("epp.tool.invocation.requested", input_event.json())
        output = await wait_for_event("epp.decision.issued")
        
        assert output.trace_id == "trace-42"
    
    def test_output_parent_event_id_links_to_input(self, nats, pdp_handler):
        """parent_event_id указывает на конкретный входной event."""
        input_event = ToolInvocationRequested(...)
        input_id = input_event.event_id
        
        await nats.publish("epp.tool.invocation.requested", input_event.json())
        output = await wait_for_event("epp.decision.issued")
        
        assert output.parent_event_id == input_id
    
    def test_output_has_own_unique_event_id(self, nats, pdp_handler):
        """У выходного события свой уникальный event_id."""
        input_event = ToolInvocationRequested(...)
        
        await nats.publish("epp.tool.invocation.requested", input_event.json())
        output = await wait_for_event("epp.decision.issued")
        
        assert output.event_id is not None
        assert output.event_id != input_event.event_id
```

**Causality Contract (2 теста):**

```python
class TestPdpHandlerCausalityContract:
    
    def test_decision_issued_before_execution_started(self, nats, pdp, pep):
        """decision.issued публикуется ДО execution.started."""
        events = []
        
        async def collect(msg):
            events.append(msg)
        
        await nats.subscribe("epp.decision.issued", cb=collect)
        await nats.subscribe("epp.execution.started", cb=collect)
        
        await pdp.start()
        await pep.start()
        await nats.publish("epp.tool.invocation.requested", event.json())
        await asyncio.sleep(0.2)
        
        decision_idx = next(i for i, e in enumerate(events)
                          if e.event_type == "decision.issued")
        execution_idx = next(i for i, e in enumerate(events)
                           if e.event_type == "execution.started")
        
        assert decision_idx < execution_idx
    
    def test_no_orphan_events_except_root(self, provenance_store):
        """Кроме tool.invocation.requested, каждое событие имеет parent."""
        for event in provenance_store.get_all():
            if event.event_type != "tool.invocation.requested":
                assert event.parent_event_id is not None
```

### 6.2. Полный список contract tests (на всех подписчиков)

| Подписчик | Input | Output | Contract Tests |
|-----------|-------|--------|----------------|
| PDP Handler | ToolInvocationRequested | DecisionIssued | 12 (4+3+3+2) |
| PEP Handler | DecisionIssued | ExecutionStarted, ExecutionCompleted | 12 |
| Approval Handler | DecisionIssued | ApprovalCreated | 12 |
| Semantic Handler | ToolInvocationRequested | IntentResolved | 12 |
| Provenance Handler | All events | ProvenanceGraph update | 8 (без Identity) |

**Итого contract tests:** 56 тестов — permanent, живут навсегда.

---

## Атом 7: Сводная таблица: что есть → что будет

| Тип теста | Сейчас | После рефакторинга | Longevity |
|-----------|--------|-------------------|-----------|
| **Существующие unit/integration** | ~150 тестов | ~150 тестов (не трогаем) | Permanent |
| **Characterization** | 0 | **14 тестов** (до рефакторинга) | **Temporary** — удалить после |
| **Contract (на подписчиков)** | 0 | **56 тестов** | Permanent |
| **Property-based** | 0 | **4 теста** | Permanent |
| **Итого временных** | 0 | **14** | Удалить |
| **Итого постоянных** | ~150 | **~210** | Навсегда |

---

## Атом 8: Процесс работы с characterization tests

### ДО рефакторинга
```bash
# Пишешь 14 тестов
pytest tests/characterization/  # → ВСЕ ЗЕЛЁНЫЕ
git add tests/characterization/
git commit -m "chore: add 14 characterization tests for POST /events/tool-call"
```

### ВО ВРЕМЯ каждого шага рефакторинга
```bash
# Если ты изменил main.py или добавил/изменил subscriber:
pytest tests/characterization/  # → ВСЕ ЕЩЁ ЗЕЛЁНЫЕ?
# Если нет → откат (ты изменил поведение)
```

### ПОСЛЕ рефакторинга
```bash
# 1. Убедился, что characterization зелёные
pytest tests/characterization/  # → ✅

# 2. Написал contract tests для новых подписчиков
pytest tests/contract/  # → ✅

# 3. Удалил characterization tests
rm -rf tests/characterization/
git commit -m "chore: remove characterization tests, refactoring complete"
```

---

## Резюме

**14 characterization tests** — минимальная достаточность для refactoring.
**56 contract tests** — permanent страховка для event-native архитектуры.

Characterization tests проверяют: *«main.py ведёт себя ТАК ЖЕ»*
Contract tests проверяют: *«каждый подписчик соблюдает свой контракт»*

Первые — временные и удаляются. Вторые — навсегда.

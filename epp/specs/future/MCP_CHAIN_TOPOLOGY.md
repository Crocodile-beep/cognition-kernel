# MCP Chain Topology: Server-to-Server в Client-Server Архитектуре

> **Дата:** 4 июня 2026  
> **Контекст:** Анализ концепции MCP 2.0 с chaining серверов  
> **Связь с EPP:** Позиционирование EPP как узла в chain  

---

## 1. Как MCP работает сейчас (v1.0)

Текущий MCP — это **плоская топология**:

```
        ┌───────────────────┐
        │    MCP Client      │
        │  (OpenCode/Claude) │
        └──┬────┬────┬───────┘
           │    │    │
     ┌─────┘    │    └─────┐
     ▼          ▼          ▼
┌─────────┐ ┌─────────┐ ┌─────────┐
│ Server A │ │ Server B │ │ Server C │
│ (bash)   │ │ (files)  │ │ (web)    │
└──────────┘ └─────────┘ └──────────┘
```

**Характеристики:**
- Client подключается к N серверам (1:N)
- Каждый сервер — независимый процесс со своими tools
- Серверы НЕ общаются между собой
- Client агрегирует tools от всех серверов
- Client решает, какой server вызвать для какого tool

---

## 2. Что ты предлагаешь: MCP 2.0 с цепочкой серверов

```
        ┌────────────────────────────────┐
        │         MCP Client              │
        │  (OpenCode/Claude)              │
        └─────────────┬───────────────────┘
                      │ tools/list, tools/call
                      ▼
              ┌───────────────┐
              │  MCP Server 1  │  ← Governance Layer (EPP)
              │  (policy check)│
              └───────┬───────┘
                      │ tools/call (если ALLOW)
                      ▼
              ┌───────────────┐
              │  MCP Server 2  │  ← Sandbox / Execution Layer
              │  (sandbox)     │
              └───────┬───────┘
                      │ tools/call (если sandbox разрешил)
                      ▼
              ┌───────────────┐
              │  MCP Server 3  │  ← Real Tool (bash/files/web)
              │  (execution)   │
              └───────────────┘
```

**Ключевое отличие:** Client подключается только к Server 1. Server 1 сам решает, вызывать ли Server 2, Server 3 и т.д. Серверы образуют **chain of responsibility**.

---

## 3. First Principles: что даёт chain topology

### От Client-Native Aggregation → Server-Native Delegation

Сейчас в MCP v1:
```
Client знает про Server A, B, C.
Client сам решает, какой server вызвать.
Client не может контролировать, что server делает.
Server не знает про другие server'ы.
```

С MCP chain:
```
Client знает только про Server 1.
Server 1 решает: можно ли выполнить, какой server дальше.
Server 1 может: проверить политики, записать audit, изменить запрос.
Server 2 может: добавить sandbox, ограничить ресурсы.
Server 3 делает реальную работу.
Client НЕ знает про Server 2 и 3.
```

### Это middleware chain для AI execution

Точно так же, как в веб-фреймворках:

```python
# Express.js / FastAPI middleware chain
@app.middleware("http")
async def auth_middleware(request, call_next):    # ← MCP Server 1 (Governance)
    if not is_authenticated(request):             # ← Policy check
        return Response(status_code=403)
    response = await call_next(request)           # ← Forward to next
    
@app.middleware("http")
async def rate_limit_middleware(request, call_next):  # ← MCP Server 2 (Sandbox)
    if is_rate_limited(request):                     # ← Resource control
        return Response(status_code=429)
    response = await call_next(request)              # ← Forward to next

@app.get("/api/data")                               # ← MCP Server 3 (Execution)
async def get_data():
    return {"data": "real response"}
```

MCP Chain — это **middleware chain**, где каждый сервер:
1. Принимает запрос
2. Может обработать/изменить/отклонить
3. Может вызвать следующий сервер в цепи
4. Может обработать ответ от следующего сервера
5. Возвращает результат предыдущему

---

## 4. Архитектурные паттерны, которые открывает chain

### Pattern 1: Governance Pipeline

```
Agent → Governance MCP Server (policy check)
         → Sandbox MCP Server (isolation)
           → Execution MCP Server (real tool)
             → Result
```

Каждый сервер добавляет свою ценность:
- Governance: разрешить или нет
- Sandbox: как исполнять (Landlock, Docker, gVisor)
- Execution: исполнение

### Pattern 2: Delegation Chain

```
Agent A → EPP MCP Server
          → Agent B MCP Server (делегировано)
            → Agent C MCP Server (суб-делегировано)
              → Execution MCP Server
```

Полная цепочка provenance: A → B → C → Execution.
Каждый delegation записывается как событие.

### Pattern 3: Observability Sidecar

```
     ┌───────────────────────────────┐
     │         MCP Chain              │
     │  Server 1 → Server 2 → Server 3│
     └──────────────┬────────────────┘
                    │ события дублируются
                    ▼
           ┌──────────────────┐
           │ Observability MCP │
           │ Server (sidecar)  │
           │ (audit, metrics)  │
           └──────────────────┘
```

Chain не меняется. На каждый вызов Observability Server получает копию события для аудита.

### Pattern 4: Split-Execute

```
Agent → EPP MCP Server
        ├── PDP: ALLOW
        ├── EPP публикует событие
        ├── Возвращает агенту: "accepted, trace_id=42"
        │
        └── Асинхронно: Sandbox MCP Server → Execute MCP Server
              Результат доступен через tools/call с trace_id
```

Синхронное решение. Асинхронное исполнение. Агент не ждёт.

---

## 5. Как это соотносится с EPP

### EPP как первый сервер в chain

```
MCP Client (OpenCode)
    │
    ▼
┌──────────────────────────────┐
│  EPP MCP Server (Governance) │  ← Первый в цепи
│                              │
│  ├── PDP: policy check       │
│  ├── PEP: sandbox strategy   │
│  └── Provenance: audit       │
└──────────┬───────────────────┘
           │ если ALLOW вызывает
           ▼
┌──────────────────────────────┐
│  Tool MCP Server (execution) │  ← Второй в цепи
│  (bash, files, web, kubectl)│
└──────────────────────────────┘
```

**EPP не прокси.** EPP — **node в MCP chain**. Он принимает запрос, применяет политики, и если ALLOW — вызывает следующий сервер в цепи, который реально исполняет tool.

### Chain topology решает проблему "прокси"

Помнишь, мы обсуждали: «EPP как прокси — enterprise не хочет, чтобы трафик шёл через нас»?

В chain topology:
- **Агент НЕ отправляет трафик "через EPP"**
- Агент отправляет запрос **на EPP** (а EPP дальше решает, что делать)
- EPP — это governance node, не proxy
- Enterprise может поставить EPP как первый сервер в своей локальной сети
- Весь трафик внутри enterprise — EPP только принимает решения

---

## 6. Что нужно для MCP 2.0 Chain

### Protocol change: Server-to-Server capabilities

Текущий MCP не поддерживает server-to-server вызовы. Нужно:

```json
// tools/list response — новый флаг "forwardable"
{
  "tools": [
    {
      "name": "bash",
      "forwardable": true,
      // MCP Server может forward'ить этот tool другому серверу
    },
    {
      "name": "read",
      "forwardable": false,
      // Tool может быть вызван только напрямую
    }
  ]
}
```

```json
// tools/call — новый параметр для chaining
{
  "tool": "bash",
  "arguments": {"command": "npm run build"},
  "chain": {
    "previous": "server-gov-01",
    "trace_id": "trace-42",
    "depth": 1
  }
}
```

### Reference implementation

EPP может стать **reference implementation** MCP chain:
- EPP — первый сервер
- EPP вызывает downstream MCP servers
- EPP доказывает, что chain работает

---

## 7. Риски chain topology

### Latency accumulation
```
Server 1 (5ms) → Server 2 (10ms) → Server 3 (100ms) = 115ms
```
Каждый сервер добавляет latency. Для short chains (3-4 сервера) — приемлемо.

### Error propagation
```
Server 1 → Server 2 → Server 3 (ошибка)
                → Server 2 должен обработать ошибку от Server 3
                → Server 1 должен обработать ошибку от Server 2
```
Нужен стандартный error format для chain.

### Complexity
Chain сложнее flat topology. Нужно протокольное решение, а не "каждый реализует как хочет".

---

## 8. Но: EPP не требует MCP 2.0

Важное уточнение. EPP работает и с текущим MCP 1.0:

```
MCP 1.0 (сейчас):
  OpenCode → EPP (MCP Server)
             ├── EPP принимает tools/call
             ├── EPP сам исполняет tool (через свой PEP)
             └── EPP возвращает результат

MCP 2.0 (chain):
  OpenCode → EPP → Tool Server
             ├── EPP принимает tools/call  
             ├── EPP проверяет политики
             ├── EPP вызывает Tool Server (server-to-server)
             └── Tool Server возвращает результат → EPP → Agent
```

MCP 2.0 даёт cleaner separation (EPP не исполняет, а делегирует). Но MCP 1.0 работает и даёт те же guarantees (safety, audit, control).

**Chain topology — не обязательное условие для EPP. Это evolution path.**

---

## 9. Что я думаю о твоей идее

**Это правильное направление для MCP 2.0.**

Текущий MCP (1.0) — плоский. Он не отражает реальность, где governance, sandbox и execution — разные ответственности, которые должны быть разделены.

Chain topology:
- **Отражает реальную архитектуру** governance → sandbox → execution
- **Решает проблему proxy** — EPP не прокси, а node в цепи
- **Позволяет delegation** — server-to-server вызовы
- **Создаёт middleware chain** для AI execution
- **Позиционирует EPP** как первый (governance) node в этой цепи

**Для EPP это означает:**
1. Сейчас EPP работает с MCP 1.0 — self-contained server
2. Когда MCP 2.0 появится — EPP станет первым сервером в chain
3. Архитектура EPP (PDP/PEP разделение) уже готова к chain — PDP решает, PEP либо сам исполняет, либо forward'ит

**Для статуса dev (твой контекст): MCP 1.0 — это нормально. Chain — evolution path, не requirement.**

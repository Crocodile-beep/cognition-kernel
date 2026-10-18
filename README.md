# Cognition Governance Layer для OpenCode
## Полное архитектурное описание и план реализации

**Версия:** 1.0  
**Дата:** 4 июня 2026  
**Автор:** Architecture Team  
**Статус:** Draft for Review


---

## 📋 Оглавление

1. [Executive Summary](#1-executive-summary)
2. [Проблемное поле и обоснование](#2-проблемное-поле-и-обоснование)
3. [Архитектурные принципы (First Principles)](#3-архитектурные-принципы-first-principles)
4. [Архитектура решения](#4-архитектура-решения)
5. [Компоненты и их взаимодействие](#5-компоненты-и-их-взаимодействие)
6. [Поток данных и lifecycle](#6-поток-данных-и-lifecycle)
7. [Реализация компонентов](#7-реализация-компонентов)
8. [Безопасность и изоляция](#8-безопасность-и-изоляция)
9. [Roadmap и этапы](#9-roadmap-и-этапы)
10. [Риски и митигации](#10-риски-и-митигации)
11. [Глоссарий](#11-глоссарий)


---

## 1. Executive Summary

### 1.1. Что мы строим

**Cognition Governance Layer** — это система capability-based security, policy-as-code enforcement и полного аудита для AI-агентов в OpenCode, реализованная как MCP сервер с kernel-level изоляцией.

### 1.2. Ключевые тезисы

| Аспект | Решение |
|--------|---------|
**Проблема** | OpenCode не sandboxит агентов — permission system это UX feature, не security boundary |
**Подход** | MCP сервер с per-tool-call sandboxing на основе Landlock + seccomp |
**Уникальность** | Каждый вызов инструмента — отдельный sandbox с capability-политикой |
**Изоляция** | 6 слоёв: namespaces, cgroups, pivot_root, Landlock, capabilities, seccomp |
**Управление** | Policy-as-code через OPA/Rego, полный provenance audit |

### 1.3. Почему MCP, а не нативный плагин

OpenCode plugins имеют критическое ограничение: `tool.execute.before` hooks **не перехватывают вызовы subagent-ов**. Это позволяет обойти любую policy.

> **Ключевой architectural decision:** MCP сервер — единственный способ получить гарантированную изоляцию и перехват всех вызовов.


## 2. Проблемное поле и обоснование

### 2.1. Официальная позиция OpenCode

Из `SECURITY.md`:

> *"OpenCode does NOT sandbox the agent. The permission system exists as a UX feature — it is not designed to provide security isolation. If you need true isolation, run OpenCode inside a Docker container or VM."*

### 2.2. Реальные векторы атак

| Вектор | Описание | Доказательство |
|--------|----------|----------------|
**Plugin hooks bypass** | Subagent-ы не trigger-ят `tool.execute.before` hooks | [Issue #5894] |
**Supply chain** | Любой npm plugin имеет полный FS доступ | [SECURITY.md] |
**Prompt injection** | Агент может быть скомпрометирован через внешние данные | [Sandlock article] |
**Cross-tool attack** | Web search tool может скомпрометировать shell tool | [Sandlock article] |

### 2.3. Требования к решению

1. **Перехват всех tool calls** — включая subagent-ы
2. **Per-tool capability model** — каждый инструмент получает минимум прав
3. **Zero trust by default** — deny если не разрешено явно
4. **Полный audit trail** — с reasoning chain для compliance
5. **Без изменений OpenCode core** — использование extension points


## 3. Архитектурные принципы (First Principles)

### 3.1. Базовые атомы безопасности

```
┌─────────────────────────────────────────────────────────────────────┐
│                     FIRST PRINCIPLES LAYER                          │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Атом 1: ВСЕ операции проходят через syscalls                       │
│  └── read(), write(), openat(), execve(), socket()                  │
│                                                                      │
│  Атом 2: Перехват на уровне ядра = единственный надежный способ     │
│  └── seccomp-bpf (syscall filtering)                                │
│  └── Landlock LSM (path-based FS control)                           │
│                                                                      │
│  Атом 3: Изоляция требует отдельного процесса                       │
│  └── Невозможно в том же процессе из-за shared memory space         │
│  └── Worker threads не дают syscall изоляции                         │
│                                                                      │
│  Атом 4: Capability-based > RBAC                                    │
│  └── Не "роль администратора", а "может читать /src/**"             │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2. Почему нативный OpenCode plugin невозможен

```typescript
// ❌ Так НЕ работает для полной изоляции
export const SecurityPlugin: Plugin = {
  "tool.execute.before": async (input) => {
    // Проблема 1: Не перехватывает subagent-ы
    // Проблема 2: Плагин в том же процессе — нет syscall isolation
    // Проблема 3: require() дает полный доступ к FS
    if (isDangerous(input.tool)) throw new Error("Blocked");
  }
}
```

### 3.3. Правильная архитектура: MCP + kernel isolation

```typescript
// ✅ Так РАБОТАЕТ полная изоляция
// MCP сервер — отдельный процесс с seccomp/landlock
const mcp = new FastMCP("Governance Layer");

mcp.tool("evaluate_permission", async (action, resource) => {
  // Каждый вызов — в своем sandbox
  return await sandboxedEvaluate(action, resource);
});
```


## 4. Архитектура решения

### 4.1. High-level architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           OPENCODE PROCESS                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Built-in MCP Client                                                │   │
│  │  - Поддерживает MCP servers из коробки                  │   │
│  │  - stdio transport для локальной коммуникации                       │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                   │                                         │
│                           stdio (inherited)                                 │
│                                   ↓                                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │              COGNITION GOVERNANCE MCP SERVER                         │   │
│  │  ┌───────────────────────────────────────────────────────────────┐ │   │
│  │  │  Process Isolation (6 layers)                     │ │   │
│  │  │  ┌─────────────────────────────────────────────────────────┐ │ │   │
│  │  │  │  1. Namespaces (PID, mount, net, user)                  │ │ │   │
│  │  │  │  2. cgroup v2 (CPU, memory, PID limits)                 │ │ │   │
│  │  │  │  3. pivot_root (host FS недоступен)                     │ │ │   │
│  │  │  │  4. Landlock (path-based FS rules)         │ │ │   │
│  │  │  │  5. Capabilities (drop ALL, add specific)               │ │ │   │
│  │  │  │  6. seccomp-bpf (syscall allowlist)                     │ │ │   │
│  │  │  └─────────────────────────────────────────────────────────┘ │ │   │
│  │  └───────────────────────────────────────────────────────────────┘ │   │
│  │                                                                      │   │
│  │  ┌───────────────────────────────────────────────────────────────┐ │   │
│  │  │  Tool Registry                                                 │ │   │
│  │  │  - evaluate_permission (основной)                             │ │   │
│  │  │  - get_policy_status                                          │ │   │
│  │  │  - audit_query                                                │ │   │
│  │  │  - policy_override (with approval)                            │ │   │
│  │  └───────────────────────────────────────────────────────────────┘ │   │
│  │                                                                      │   │
│  │  ┌───────────────────────────────────────────────────────────────┐ │   │
│  │  │  OPA Policy Engine                                            │ │   │
│  │  │  - Rego policies from Git                                     │ │   │
│  │  │  - Semantic validation                                        │ │   │
│  │  │  - Intent verification                                        │ │   │
│  │  └───────────────────────────────────────────────────────────────┘ │   │
│  │                                                                      │   │
│  │  ┌───────────────────────────────────────────────────────────────┐ │   │
│  │  │  Audit Pipeline                                               │ │   │
│  │  │  - JSONL structured logs                                      │ │   │
│  │  │  - Reasoning chain preservation                               │ │   │
│  │  │  - SIEM integration                                           │ │   │
│  │  └───────────────────────────────────────────────────────────────┘ │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2. Per-tool-call sandboxing (ключевая инновация)

Вдохновлено `sandlock.mcp`:

```python
# Каждый tool call — в своем sandbox с capability-политикой
@mcp.tool()
def sandboxed_execute(command: str, capabilities: dict) -> str:
    """
    Запускает команду в изолированном sandbox.
    capabilities определяет, что КОНКРЕТНО этот вызов может делать.
    """
    policy = build_policy_from_capabilities(capabilities)
    
    with Sandbox(policy) as sandbox:
        # Sandbox имеет:
        # - Отдельный process namespace
        # - Landlock FS rules
        # - seccomp syscall filter
        # - cgroup limits (CPU, memory)
        return sandbox.run(command)
```

**Почему это важно:** Даже если LLM скомпрометирована через prompt injection, инструмент не может делать ничего, что не было явно разрешено в `capabilities`.


## 5. Компоненты и их взаимодействие

### 5.1. Component diagram

```mermaid
graph TB
    subgraph "OpenCode"
        A[Agent Loop]
        B[MCP Client]
        C[Plugin Hooks<br/>(NOT for security)]
    end
    
    subgraph "Cognition Governance MCP Server"
        D[MCP Server Core<br/>FastMCP/stdio]
        E[Sandbox Manager]
        F[Policy Engine<br/>OPA/Rego]
        G[Audit Pipeline]
        
        subgraph "Per-tool Sandbox"
            H[Landlock Filter]
            I[seccomp Filter]
            J[cgroup Limits]
        end
    end
    
    subgraph "External"
        K[Git: Policies]
        L[SIEM / Audit Logs]
    end
    
    A -->|tool call| B
    B -->|stdio| D
    D -->|evaluate| F
    F -->|policy check| K
    D -->|spawn| E
    E -->|enforce| H
    E -->|enforce| I
    E -->|enforce| J
    D -->|log| G
    G -->|stream| L
```

### 5.2. Коммуникационная модель

| Компонент | Протокол | Данные |
|-----------|----------|--------|
**OpenCode → MCP Server** | stdio (JSON-RPC) | Tool calls, context |
**MCP Server → OPA** | HTTP localhost | Policy queries |
**MCP Server → Sandbox** | fork() + IPC | Isolated execution |
**MCP Server → Audit** | JSONL file | Structured logs |

### 5.3. Интеграция с OpenCode (без изменений core)

```bash
# Регистрация MCP сервера в OpenCode
opencode mcp add cognition-governance \
  --command "/usr/local/bin/governance-mcp-server" \
  --args "--policy-dir", "/etc/cognition/policies"

# OpenCode автоматически:
# 1. Запускает MCP сервер как дочерний процесс
# 2. Устанавливает stdio связь
# 3. Регистрирует все tools
# 4. LLM может вызывать их как нативные
```


## 6. Поток данных и lifecycle

### 6.1. Sequence diagram: нормальное выполнение

```
┌────────┐     ┌──────────┐     ┌──────────┐     ┌────────┐     ┌─────────┐
│  User  │     │ OpenCode │     │   MCP    │     │  OPA   │     │ Sandbox │
│        │     │  Agent   │     │ Server   │     │        │     │         │
└───┬────┘     └────┬─────┘     └────┬─────┘     └───┬────┘     └────┬────┘
    │               │                │               │               │
    │ "deploy"      │                │               │               │
    │──────────────>│                │               │               │
    │               │                │               │               │
    │               │ tool.call      │               │               │
    │               │───────────────>│               │               │
    │               │                │               │               │
    │               │                │ policy.query  │               │
    │               │                │──────────────>│               │
    │               │                │               │               │
    │               │                │    allow      │               │
    │               │                │<──────────────│               │
    │               │                │               │               │
    │               │                │ sandbox.spawn │               │
    │               │                │───────────────────────────────>│
    │               │                │               │               │
    │               │                │   result      │               │
    │               │                │<───────────────────────────────│
    │               │                │               │               │
    │               │                │ audit.log     │               │
    │               │                │──────────────>│ (file)        │
    │               │                │               │               │
    │               │    result      │               │               │
    │               │<───────────────│               │               │
    │               │                │               │               │
    │   ответ       │                │               │               │
    │<──────────────│                │               │               │
    │               │                │               │               │
```

### 6.2. Sequence diagram: запрещенное действие

```
┌────────┐     ┌──────────┐     ┌──────────┐     ┌────────┐
│  User  │     │ OpenCode │     │   MCP    │     │  OPA   │
│        │     │  Agent   │     │ Server   │     │        │
└───┬────┘     └────┬─────┘     └────┬─────┘     └───┬────┘
    │ "read ssh"    │                │               │
    │──────────────>│                │               │
    │               │                │               │
    │               │ tool.call      │               │
    │               │───────────────>│               │
    │               │                │               │
    │               │                │ policy.query  │
    │               │                │──────────────>│
    │               │                │               │
    │               │                │    deny       │
    │               │                │<──────────────│
    │               │                │               │
    │               │                │ audit.critical│
    │               │                │──────────────>│ (alert)
    │               │                │               │
    │               │   ERROR: denied│               │
    │               │<───────────────│               │
    │               │                │               │
    │   "Access     │                │               │
    │    denied"    │                │               │
    │<──────────────│                │               │
```

### 6.3. Capability declaration flow

```yaml
# 1. Plugin manifest (поставщик декларирует)
plugin: my-tool@1.0.0
capabilities:
  - fs:read:/src/**
  - fs:write:/src/output/**
  - network:connect:api.github.com:443
  - shell:exec:npm,node

# 2. При установке — верификация и approval
governance verify my-tool@1.0.0 --manifest ./manifest.yaml
# → Checking signatures (Sigstore)
# → Validating capability syntax
# → Requesting approval for network:connect
# → Policy stored in /etc/cognition/policies/tools/my-tool.yaml

# 3. При каждом вызове — runtime evaluation
# Каждый tool call проверяется против сохраненной политики
```


## 7. Реализация компонентов

### 7.1. MCP сервер (TypeScript)

```typescript
// governance-mcp-server/src/index.ts
import { FastMCP } from "fastmcp";
import { Sandbox } from "./sandbox";
import { OpaClient } from "./opa-client";
import { AuditLogger } from "./audit";

const mcp = new FastMCP({
  name: "Cognition Governance",
  version: "1.0.0"
});

// Основной tool для permission evaluation
mcp.addTool({
  name: "evaluate_permission",
  description: "Evaluate if an agent action should be allowed",
  parameters: {
    type: "object",
    properties: {
      action: { type: "string", enum: ["fs:read", "fs:write", "shell:exec", "network:connect"] },
      resource: { type: "string" },
      context: { type: "object" }
    }
  },
  execute: async (args) => {
    // 1. Policy evaluation через OPA
    const opa = new OpaClient("http://localhost:8181");
    const decision = await opa.query("governance/allow", args);
    
    // 2. Audit logging
    const audit = new AuditLogger("/var/log/cognition");
    await audit.log({
      timestamp: new Date().toISOString(),
      ...args,
      decision: decision.allowed,
      reasoning: decision.reasoning
    });
    
    // 3. При необходимости — sandboxed execution
    if (decision.allowed && args.action === "shell:exec") {
      const sandbox = new Sandbox({
        capabilities: decision.capabilities,
        timeout: 30000,
        memoryLimit: "512MB"
      });
      const result = await sandbox.run(args.resource);
      return { allowed: true, result };
    }
    
    return { allowed: decision.allowed, reason: decision.reason };
  }
});

// Policy management tools
mcp.addTool({
  name: "get_policy_status",
  description: "Get current governance policy status",
  execute: async () => {
    return await new OpaClient().getStatus();
  }
});

mcp.addTool({
  name: "audit_query",
  description: "Query audit logs",
  parameters: {
    type: "object",
    properties: {
      plugin: { type: "string" },
      action: { type: "string" },
      from: { type: "string" }
    }
  },
  execute: async (args) => {
    return await new AuditLogger().query(args);
  }
});

mcp.start({ transportType: "stdio" });
```

### 7.2. Sandbox implementation (Rust/Go for performance)

```rust
// sandbox/src/lib.rs
use landlock::{Access, PathBeneath, Ruleset, RulesetAttr};
use seccompiler::{BpfProgram, SeccompAction};
use nix::sched::{clone, CloneFlags};
use nix::unistd::{fork, ForkResult};

pub struct Sandbox {
    capabilities: Capabilities,
    limits: ResourceLimits,
}

impl Sandbox {
    pub fn new(caps: Capabilities) -> Self {
        Self {
            capabilities: caps,
            limits: ResourceLimits::default(),
        }
    }
    
    pub fn run<F>(&self, func: F) -> Result<String>
    where
        F: FnOnce() -> String,
    {
        // 1. Fork with namespaces
        match clone(
            Box::new(|| {
                // 2. Apply Landlock rules
                self.apply_landlock()?;
                
                // 3. Apply seccomp filter
                self.apply_seccomp()?;
                
                // 4. Apply cgroup limits
                self.apply_cgroups()?;
                
                // 5. Execute function
                let result = func();
                Ok(result)
            }),
            CloneFlags::CLONE_NEWNS | CloneFlags::CLONE_NEWNET | CloneFlags::CLONE_NEWPID,
            None,
        ) {
            Ok(pid) => wait_for_result(pid),
            Err(e) => Err(e.into()),
        }
    }
    
    fn apply_landlock(&self) -> Result<()> {
        let mut ruleset = Ruleset::new();
        ruleset.handle_access(Access::from_bits(
            if self.capabilities.fs_write {
                Access::WriteFile | Access::ReadFile
            } else {
                Access::ReadFile
            }
        ));
        
        for path in &self.capabilities.allowed_paths {
            ruleset.add_rule(PathBeneath::new(path, Access::ReadFile)?)?;
        }
        
        ruleset.restrict_self()?;
        Ok(())
    }
    
    fn apply_seccomp(&self) -> Result<()> {
        let allowlist = vec![
            "read", "write", "openat", "close", "exit", "exit_group"
        ];
        
        if self.capabilities.network_allowed {
            allowlist.extend(["socket", "connect", "sendto", "recvfrom"]);
        }
        
        let filter = BpfProgram::from_allowlist(&allowlist, SeccompAction::Kill)?;
        filter.load()?;
        Ok(())
    }
}
```

### 7.3. OPA Policies (Rego)

```rego
# policies/governance.rego
package governance

# Deny by default
default allow = false

# Allow fs:read для declared paths
allow {
    input.action == "fs:read"
    input.plugin.capabilities[_] == sprintf("fs:read:%s", [input.resource])
    not data.blocked_paths[input.resource]
}

# Allow fs:write только если user intent = modify
allow {
    input.action == "fs:write"
    input.user_intent == "modify"
    input.plugin.capabilities[_] == sprintf("fs:write:%s", [input.resource])
}

# Allow network только с audit
allow {
    input.action == "network:connect"
    input.plugin.capabilities[_] == sprintf("network:%s", [input.host])
    audit.required("network_access", input)
}

# Semantic validation: проверка intent
allow {
    input.action == "shell:exec"
    input.user_intent == "build"
    input.resource == "npm run build"
    input.plugin.capabilities[_] == "shell:exec:npm"
}

# Reasoning for decision
reason = msg {
    not allow
    msg = sprintf("Capability '%s' not declared for %s", [input.action, input.plugin.name])
}

reason = msg {
    allow
    msg = "Policy matched"
}
```

### 7.4. Audit pipeline

```typescript
// audit/logger.ts
export class AuditLogger {
    private stream: WriteStream;
    
    async log(entry: AuditEntry): Promise<void> {
        const enriched = {
            ...entry,
            session_id: this.getSessionId(),
            provenance_parent: this.getLastActionId(),
            reasoning_chain: await this.getReasoningChain(entry),
            hash: this.calculateHash(entry)
        };
        
        this.stream.write(JSON.stringify(enriched) + "\n");
        
        if (entry.severity === "critical") {
            await this.notifySecurityTeam(enriched);
        }
    }
    
    async query(filter: AuditFilter): Promise<AuditEntry[]> {
        // Streaming query over JSONL files
        const results = [];
        for await (const line of this.readLines()) {
            const entry = JSON.parse(line);
            if (this.matchesFilter(entry, filter)) {
                results.push(entry);
            }
        }
        return results;
    }
}
```


## 8. Безопасность и изоляция

### 8.1. Шестиуровневая модель изоляции

Адаптировано из HiveBox:

| Уровень | Механизм | Что контролирует |
|---------|----------|------------------|
**1** | Linux Namespaces | Процессы, сеть, mount points, users |
**2** | cgroup v2 | CPU, memory, PID limits; swap disabled |
**3** | pivot_root | Host filesystem становится недоступен |
**4** | Landlock LSM | Path-based FS access control |
**5** | Capabilities | Minimal privilege set + NO_NEW_PRIVS |
**6** | seccomp-bpf | Syscall allowlist (~80 default, ~40 strict) |

### 8.2. Per-tool capability declaration

```yaml
# Пример: безопасный web search tool
tool: web_search
capabilities:
  # Только чтение из workspace
  fs_read: ["/workspace/**"]
  # Нет записи
  fs_write: []
  # Только GitHub API
  network_allow_hosts: ["api.github.com:443"]
  # Только эти команды
  shell_exec: []
  # Очистка всех env, кроме указанных
  env_allow: []
  # Resource limits
  max_memory: "256MB"
  max_cpu_seconds: 30
```

### 8.3. Предотвращение cross-tool атак

```python
# Сценарий: LLM скомпрометирована через web_search
# Prompt injection заставляет вызвать bash с curl на attacker.com

# ❌ Без per-tool sandboxing
# bash имеет network доступ (нужен для web_search) → атака успешна

# ✅ С per-tool sandboxing
bash_tool = SandboxedTool(
    name="bash",
    capabilities={"fs_writable": ["/workspace"], "network_allow_hosts": []}
)
# curl attacker.com блокируется на уровне Landlock
# → атака не проходит, несмотря на скомпрометированную LLM
```

### 8.4. Supply chain security

```bash
# Подпись манифеста через Sigstore
cosign sign-blob --key cosign.key manifest.yaml --output-signature manifest.sig

# Верификация при установке
cosign verify-blob --key cosign.pub --signature manifest.sig manifest.yaml

# Политика требует valid signature для production
governance policy set --require-signature=true --level=production
```


## 9. Roadmap и этапы

### 9.1. Stage 0: PoC (2-3 недели)

| Задача | Описание | Статус |
|--------|----------|--------|
**MCP сервер** | FastMCP на TypeScript с 1 tool | 🟡 In Progress |
**Sandbox MVP** | Простой seccomp + Landlock через fork() | 🔴 Not Started |
**OPA интеграция** | Базовый Rego policy для fs:read | 🔴 Not Started |
**Audit** | JSONL логирование | 🔴 Not Started |
**OpenCode тест** | Регистрация и вызов tool | 🔴 Not Started |

**Success criteria:** `opencode mcp add` → вызов `evaluate_permission` → возвращает allow/deny

### 9.2. Stage 1: Alpha (4-6 недель)

| Задача | Описание |
|--------|----------|
**Полная capability модель** | fs:read/write, network, shell с path globs |
**6-слойная изоляция** | Namespaces, cgroups, pivot_root, Landlock, caps, seccomp |
**Per-tool sandboxing** | Каждый tool call — свой sandbox |
**Policy as Code** | Git OPA policies + hot reload |
**Audit с reasoning** | Полная provenance chain |

### 9.3. Stage 2: Beta (8-10 недель)

| Задача | Описание |
|--------|----------|
**User Intent Layer** | Анализ намерения пользователя (analyze/modify/audit) |
**Semantic validation** | Проверка соответствия архитектурным инвариантам |
**Anomaly detection** | ML на audit логах |
**Enterprise features** | SIEM интеграция, compliance reports (SOC2) |

### 9.4. Stage 3: Production (12-14 недель)

| Задача | Описание |
|--------|----------|
**Web dashboard** | Визуализация политик, аудит, инциденты |
**Multi-tenancy** | Поддержка teams и RBAC |
**Policy marketplace** | Sharing policies for common plugins |
**Full EPP integration** | Связь с Cognitive Kernel vision |


## 10. Риски и митигации

### 10.1. Технические риски

| Риск | Вероятность | Митигация |
|------|-------------|-----------|
**OpenCode меняет MCP API** | Низкая | MCP — стандартизированный протокол |
**Performance overhead** | Средняя | Per-tool sandbox ~10-50ms; кэширование политик |
**Subagent bypass остается** | Низкая | MCP перехватывает ВСЕ вызовы, включая subagent-ы |
**Landlock требует новых kernel** | Низкая | Fallback на seccomp-only для старых kernel |

### 10.2. Product риски

| Риск | Вероятность | Митигация |
|------|-------------|-----------|
**Отвлечение от core vision** | Средняя | Stage 0 = 3 months max, затем return to EPP |
**Community не принимает** | Низкая | MCP servers supported natively |
**Enterprise требует большего** | Средняя | Stage 2 включает compliance features |

### 10.3. Безопасность sandbox

| Риск | Митигация |
|------|-----------|
**seccomp bypass** | Landlock дополнительный слой; регулярный аудит syscall allowlist |
**Landlock escape** | Второй слой с pivot_root + namespaces |
**Resource exhaustion** | cgroup v2 limits (CPU, memory, PIDs) |


## 11. Глоссарий

| Термин | Определение |
|--------|-------------|
**MCP (Model Context Protocol)** | Протокол для подключения инструментов к LLM; "USB-интерфейс для AI-агентов" |
**Governance Layer** | Слой authority, trust, approvals, semantic validation |
**Capability** | Атомарное право: `fs:read:/src/**` |
**Landlock** | Linux LSM для path-based filesystem access control |
**seccomp-bpf** | Linux syscall filtering; allowlist-based isolation |
**OPA/Rego** | Policy-as-code engine от CNCF |
**Provenance** | Полная цепочка решений с reasoning |
**User Intent** | Semantic намерение пользователя (analyze vs modify) |
**Sandbox** | Изолированная среда выполнения с kernel-level ограничениями |


## ✅ Approval Checklist

- [ ] Architecture Review
- [ ] Security Review  
- [ ] Performance Review
- [ ] OpenCode Compatibility Review
- [ ] Resource Allocation (3 months, 2 engineers)
- [ ] Community RFC published

---

**Следующие шаги:**
1. Утверждение архитектуры
2. Найм/аллокация 2 инженеров (Rust + TypeScript)
3. Stage 0 PoC implementation (2 weeks)
4. Security audit после Stage 1

**Contact:** Architecture Team | `arch@cognition.sh`
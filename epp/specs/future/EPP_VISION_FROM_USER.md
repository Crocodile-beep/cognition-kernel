# EPP Vision: From User's Eyes

> **Назначение:** Описать EPP таким, каким его увидит пользователь.
> **Принцип:** First Principle Sinking — от факта использования вниз к архитектуре.
> **Аудитория:** Соло-разработчик и Team lead, которые впервые слышат про EPP.

---

## Part 1: The First Principle — What Problem Does EPP Solve?

### The Universal Truth

```
AI-агенты выполняют действия от имени человека.
Эти действия могут быть опасными.
Ни одна существующая система не гарантирует, что агент
не сделает то, что ему не разрешено.
```

### The User's Need (from first principles)

Любой пользователь AI-агента (будь то соло-разработчик с OpenCode или enterprise с кастомным agent backend) имеет **три фундаментальные потребности**:

| # | Потребность | Что это значит | Почему это не решено сейчас |
|---|-------------|----------------|---------------------------|
| 1 | **Safety** | Агент не может выполнить действие, которое мне навредит | OpenCode permission system — UX feature, не security |
| 2 | **Audit** | Я могу точно сказать, что агент сделал и почему | Логи не дают цепочки решений с reasoning |
| 3 | **Control** | Я могу управлять тем, что агент может делать | RBAC не работает для dynamic AI-сценариев |

**EPP — это система, которая решает все три потребности одновременно.**

---

## Part 2: Solo Developer Flow (The "How It Works")

### What the user sees

Ты — соло-разработчик. У тебя стоит OpenCode на ноутбуке. Ты слышал, что AI-агенты могут быть опасны, и хочешь контролировать, что твой агент делает.

**Шаг 1: Установка**

```bash
# Ты устанавливаешь EPP одной командой
pip install cognition-kernel

# Регистрируешь в OpenCode как MCP Server
opencode mcp add cognition-governance \
  --command "cognition-kernel"
```

**Всё. Это заняло 1 минуту.**

### What happens under the hood

После установки на твоей машине работает **локальный процесс**:

```
Твой ноутбук:
┌─────────────────────────────────────────────────────────────┐
│  OpenCode (AI-агент)                                         │
│    │                                                         │
│    │ MCP Protocol (stdio) — локальный канал                  │
│    ▼                                                         │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Cognition Kernel (локальный процесс)                  │   │
│  │                                                         │   │
│  │  ┌──────────┐  ┌──────────┐  ┌────────────────────┐   │   │
│  │  │  PDP     │  │  PEP     │  │  Provenance Store   │   │   │
│  │  │ (решает) │→│ (исполняет)│→│  (запоминает)       │   │   │
│  │  └──────────┘  └──────────┘  └────────────────────┘   │   │
│  │         │            │              │                   │   │
│  │         ▼            ▼              ▼                   │   │
│  │  ┌──────────┐  ┌──────────┐  ┌────────────────────┐   │   │
│  │  │  Policy  │  │ Sandbox  │  │  Local Database    │   │   │
│  │  │  Engine  │  │ (Landlock│  │  (SQLite/JSONL)     │   │   │
│  │  │  Cerbos  │  │  + ns)   │  │                     │   │   │
│  │  └──────────┘  └──────────┘  └────────────────────┘   │   │
│  └──────────────────────────────────────────────────────┘   │
│                                                             │
│  Всё локально. Нет облака. Нет прокси. Все данные — твои.   │
└─────────────────────────────────────────────────────────────┘
```

**Ничего не идёт в интернет.** EPP — локальный security-слой между агентом и твоей файловой системой.

### The Daily Flow

**Сценарий: ты просишь агента задеплоить приложение.**

```
Ты: "Задеплой приложение на production"

OpenCode получает твой запрос.
    │
    ├── OpenCode решает: "нужно вызвать bash"
    │
    ├── OpenCode → EPP (через MCP):
    │   {"action": "shell:exec", "command": "kubectl apply -f prod.yaml"}
    │
    ├── EPP.PDP проверяет политику:
    │   ├── "Ты — solo developer, твой default — allow"
    │   ├── "deploy to production требует approval"
    │   └── Решение: HOLD (нужно твоё подтверждение)
    │
    ├── EPP отправляет тебе уведомление:
    │   "Агент хочет выполнить: kubectl apply -f prod.yaml"
    │   "Подтвердить? [Y/n]"
    │
    ├── Ты подтверждаешь.
    │
    ├── EPP.PEP выполняет команду в sandbox:
    │   ├── drop all capabilities, add specific
    │   ├── seccomp: allow only kubectl syscalls
    │   ├── cgroup: limit CPU/memory
    │   └── kubectl apply -f prod.yaml (в sandbox)
    │
    ├── EPP.Provenance записывает:
    │   ├── кто запросил: ты
    │   ├── что: kubectl apply -f prod.yaml
    │   ├── решение: HOLD → подтверждено
    │   ├── кто подтвердил: ты
    │   ├── результат: success (pod created)
    │   └── timestamp: 2026-06-04T14:32:00Z
    │
    └── OpenCode возвращает результат тебе.
```

**Ты видишь:** агент работает как обычно, но если он пытается сделать что-то опасное — ты получаешь запрос на подтверждение. Всё, что он делает, записывается.

### What the user doesn't see (but matters)

| Компонент | Что делает | Как пользователь о нём узнаёт |
|-----------|-----------|------------------------------|
| **PDP** | Решает allow/deny/hold | Когда агент получает отказ или запрос подтверждения |
| **PEP** | Исполняет в sandbox | Когда команда выполняется, но не может выйти за рамки |
| **Sandbox** | Landlock+seccomp+namespaces | Никогда — это работает автоматически |
| **Policy Engine** | Проверяет политики | Когда ты меняешь политику через config file |
| **Provenance** | Пишет события | Когда ты открываешь audit log |
| **Event Bus** | Передаёт события между компонентами | Никогда — это внутренняя шина |

---

## Part 3: Team Flow (The "How It Works")

### What the user sees

Ты — CTO небольшой компании (15 разработчиков). Все используют AI-агентов. Ты хочешь:
- Чтобы разработчики могли работать свободно
- Чтобы CTO мог контролировать, что агенты делают
- Чтобы у компании был audit на случай compliance проверки

**Шаг 1: Ты (CTO) создаёшь Team в EPP Cloud**

```bash
# Один раз настраиваешь центральное управление
cognition-kernel cloud init --team "Acme Corp" --admin your@email.com

# Подключаешь SSO (Google Workspace)
cognition-kernel cloud sso setup --provider google --domain acme.com
```

**Шаг 2: Каждый разработчик устанавливает EPP локально**

```bash
# Каждый разработчик делает одну команду
pip install cognition-kernel
cognition-kernel connect --team acme-corp
```

**Шаг 3: Ты пишешь политики централизованно**

```yaml
# Ты (CTO) создаёшь policies/deploy.yaml в Git-репозитории команды.
# EPP Cloud автоматически пушит их на все локальные EPP разработчиков.

policies:
  - name: "production-deploy"
    applies_to: ["developer", "senior"]
    rules:
      - action: shell:exec
        command_prefix: "kubectl apply -f prod"
        decision: HOLD  # production deploy требует approval
        approved_by: ["tech-lead", "cto"]

  - name: "database-read"
    applies_to: ["all"]  
    rules:
      - action: shell:exec
        command_prefix: "psql"
        decision: ALLOW
        audit: true

  - name: "database-write"
    applies_to: ["senior"]
    rules:
      - action: shell:exec
        command_prefix: "psql -c INSERT"
        decision: HOLD
        approved_by: ["tech-lead"]
```

### What happens under the hood

```
Ты (CTO) пишешь политику → EPP Cloud
                              │
                    push политики
                              │
    ┌─────────────────────────┼─────────────────────────┐
    ▼                         ▼                         ▼
┌──────────────┐       ┌──────────────┐       ┌──────────────┐
│  Developer 1  │       │  Developer 2  │       │  Developer N  │
│  Локальный   │       │  Локальный   │       │  Локальный   │
│  EPP         │       │  EPP         │       │  EPP         │
│              │       │              │       │              │
│  PDP: cloud  │       │  PDP: cloud  │       │  PDP: cloud  │
│  sync        │       │  sync        │       │  sync        │
│              │       │              │       │              │
│  Audit: cloud│       │  Audit: cloud│       │  Audit: cloud│
│  → EPP Cloud │       │  → EPP Cloud │       │  → EPP Cloud │
└──────────────┘       └──────────────┘       └──────────────┘
```

**Каждый разработчик имеет свой локальный EPP, но:**
- Политики общие (ты написал — у всех применилось)
- Audit общий (ты видишь, что делают все агенты команды)
- SSO общее (кто есть кто — из Google Workspace)

### The Team Flow

**Сценарий: Junior developer просит агента изменить production БД.**

```
Джуниор: "Обнови запись пользователя в production БД"

Агент джуниора → Локальный EPP джуниора
    │
    ├── PDP: проверяет политику "database-write"
    │   ├── Политика: applies_to = ["senior"]
    │   ├── Junior не senior
    │   └── Решение: DENY (junior не может писать в БД)
    │
    ├── Агент получает: "Action denied by governance policy"
    │
    └── Ты (CTO) видишь в Audit Dashboard:
        "Junior попытался изменить production БД в 14:32 — DENY по политике"
```

**Сценарий: Senior developer просит агента задеплоить production.**

```
Сеньор: "Задеплой новую версию"

Агент сеньора → Локальный EPP сеньора
    │
    ├── PDP: проверяет политику "production-deploy"
    │   ├── Senior подпадает под applies_to
    │   ├── Решение: HOLD (нужен tech-lead или CTO)
    │   └── EPP отправляет запрос tech-lead'у
    │
    ├── Tech-lead получает уведомление:
    │   "Senior хочет задеплоить версию 2.3.1 на production"
    │   "Подтвердить? [Y/n]"
    │
    ├── Tech-lead подтверждает
    │
    ├── PEP: исполняет deploy в sandbox
    │
    └── Ты (CTO) видишь в Audit Dashboard:
        "Senior задеплоил v2.3.1 — approved by Tech-lead — success"
```

---

## Part 4: The Architecture (user-visible only)

### Layered Model — What the user needs to know

```ascii
EPP для пользователя — это "чёрный ящик" с тремя функциями:
┌─────────────────────────────────────────────────────────────┐
│  EPP = Decision(allow/deny/hold) → Execution(sandbox) → Audit(provenance)
└─────────────────────────────────────────────────────────────┘
```

Внутри этого ящика есть слои, которые пользователь **конфигурирует**, но **не видит в работе**:

### Layer 0: Event Bus (пользователь не знает про него)
```
Event Bus — это "кровеносная система".
Все компоненты общаются через события.
Пользователь не настраивает, не видит, не трогает.
```

### Layer 1: PDP — Policy Decision Point (пользователь настраивает)
```
Что делает: Решает, можно ли выполнить действие.
Как пользователь влияет: Пишет политики (YAML/Rego).
Что видит в работе: 
  - При allow: агент работает
  - При deny: агент получает отказ
  - При hold: пользователь получает запрос подтверждения
```

### Layer 2: PEP — Policy Enforcement Point (пользователь не настраивает)
```
Что делает: Исполняет действие в sandbox с ограничениями.
Что видит в работе: Команда выполняется, но не может выйти за рамки.
Что НЕ может сделать: Команда не может получить доступ к тому, 
что не разрешено политикой.
```

### Layer 3: Semantic Layer (пользователь опционально настраивает)
```
Что делает: Понимает, что пользователь на самом деле хочет сделать.
Как пользователь влияет: Определяет онтологию (маппинг tool → intent).
Что видит в работе: Политики пишутся на языке бизнеса 
("нельзя деплоить в production"), а не на языке команд 
("нельзя вызывать kubectl").
```

### Layer 4: Provenance Store (пользователь читает)
```
Что делает: Запоминает всё, что произошло.
Как пользователь влияет: Не влияет.
Что видит в работе: Audit Dashboard.
```

---

## Part 5: The Invariants (First Principles of EPP User Experience)

Эти утверждения **всегда истинны** для пользователя EPP, независимо от того, solo он или team:

### Invariant 1: Local-first
```
Локальный EPP работает без облака.
Облако нужно только для team-фич (общие политики, общий audit).
Если облако отключилось — локальный EPP продолжает работать.
```

### Invariant 2: Fail-closed
```
Если EPP не может принять решение (ошибка, недоступность) — 
действие блокируется, а не разрешается.
Безопасность важнее доступности.
```

### Invariant 3: Audit never lies
```
Provenance — append-only.
Никто не может удалить или изменить запись о том, что произошло.
Даже CTO. Даже админ.
```

### Invariant 4: Policy as code
```
Политики — это код (YAML/Rego).
Хранятся в Git. Версионируются. Review через PR.
Никаких UI-галочек "разрешить всё".
```

### Invariant 5: Provenance before execution
```
Сначала событие — потом решение — потом исполнение.
Provenance записывается ДО того, как действие выполнено
(запись о намерении), и ПОСЛЕ (запись о результате).
```

---

## Part 6: What EPP Is NOT (and why this matters)

| EPP — это НЕ | А это | Почему путают |
|-------------|-------|---------------|
| **Security proxy** | Локальный процесс | Потому что говорят "перехватывает вызовы" |
| **API Gateway** | Decision Point + Execution Point | Потому что принимает запросы и возвращает решения |
| **CI/CD system** | Governance system | Потому что может блокировать деплой |
| **Monitoring** | Provenance (не метрики, а цепочки решений) | Потому что сохраняет события |
| **RBAC system** | Capability-based security | Потому что проверяет права |
| **MCP Server** | Протокол, а не система | Потому что MCP — транспорт, EPP — содержание |

---

## Part 7: The Standard Sense Flow (Summary)

### Solo Developer

```ascii
УСТАНОВКА:
  pip install cognition-kernel
  opencode mcp add cognition-governance
  
РАБОТА:
  Агент → MCP (stdio) → PDP (policy check) → PEP (sandbox) → Result
  
AUDIT:
  Все события → локальный SQLite
  
УПРАВЛЕНИЕ:
  Локальный config file ~/.cognition-kernel/config.yaml
  
ЦЕНА: $0
```

### Team

```ascii
УСТАНОВКА:
  Каждый разработчик: pip install cognition-kernel + connect
  
РАБОТА:
  Агент → MCP (stdio) → PDP (cloud-synced policy) → PEP → Result
  
AUDIT:
  Все события → локально + cloud audit dashboard
  
УПРАВЛЕНИЕ:
  CTO пишет политики в Git → Cloud пушит на всех
  
ЦЕНА: $15/user/month
```

### Enterprise

```ascii
УСТАНОВКА:
  Self-hosted EPP Server на своей инфраструктуре
  Или EPP Cloud с выделенным инстансом
  
РАБОТА:
  Agent Backend → MCP (HTTP) → PDP (on-prem Cerbos) → PEP → Result
  
AUDIT:
  Все события → собственный PostgreSQL + SIEM (Splunk/Datadog)
  
УПРАВЛЕНИЕ:
  CISO + CTO управляют политиками
  Compliance reports (SOC2, HIPAA, SOX)
  
ЦЕНА: Custom
```

---

## Part 8: The User's One-Sentence Takeaway

> **EPP — это локальный security-агент, который встраивается между AI-ассистентом и файловой системой, проверяет каждое действие по политикам, исполняет в sandbox и записывает всю цепочку решений для аудита.**

Для соло — бесплатно, локально, без облака.
Для команды — централизованное управление и общий аудит.
Для enterprise — self-hosted с compliance.

---

*Документ создан 4 июня 2026 на основе архитектурной сессии.*
*Следующий шаг: спецификация CognitionKernel Plugin (Поток B).*

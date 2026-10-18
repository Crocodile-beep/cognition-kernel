# EPP Refactoring Reference: Event-Sourced → Event-Native

> **Контекст:** Сессия архитектурного обсуждения между Chief Architect EPP и Product Lead.
> **Дата:** 4 июня 2026
> **Назначение:** Референс для будущих сессий рефакторинга EPP Core и разработки CognitionKernel Plugin.

---

## Оглавление

1. [Контекст сессии](#1-контекст-сессии)
2. [Фундаментальный инсайт: Event-Sourced ≠ Event-Native](#2-фундаментальный-инсайт-event-sourced--event-native)
3. [First Principles декомпозиция EPP](#3-first-principles-декомпозиция-epp)
4. [Проблемы текущей архитектуры EPP](#4-проблемы-текущей-архитектуры-epp)
5. [Стратегия трёх потоков (Three-Track Evolution)](#5-стратегия-трёх-потоков-three-track-evolution)
6. [Архитектурные решения (Architectural Decisions)](#6-архитектурные-решения-architectural-decisions)
7. [Отвергнутые альтернативы](#7-отвергнутые-альтернативы)
8. [Модель зрелости EPP Core](#8-модель-зрелости-epp-core)
9. [Глоссарий: синхронизация терминов](#9-глоссарий-синхронизация-терминов)
10. [Приложение: Критический анализ подходов Sandlock и DeepSeek MCP](#10-приложение-критический-анализ-подходов)

---

## 1. Контекст сессии

### Исходная ситуация
Существующий EPP имеет Layer 1-6, построенные на Python (FastAPI, NATS, Cerbos, PostgreSQL). Система работает, но имеет архитектурный дефект: **внешне выглядит как event-native, внутренне — как RPC**.

### Триггер обсуждения
Разработка **CognitionKernel Plugin** — Security Governance Layer для OpenCode, описанного в `CognitionKernel/README.md`. Плагин позиционируется как:

1. Самостоятельный продукт (security для AI-агентов)
2. Экспериментальная ветка event-native архитектуры
3. Reference implementation для будущего EPP Core 2.0

### Ключевой внешний референс
Анализ DeepSeek показал, что MCP (Model Context Protocol) является правильным entry point для adoption. MCP — стандарт Anthropic, поддерживаемый Claude, Cline, Continue, OpenCode, Cody.

### Инсайты, полученные в сессии
1. Event-sourced ≠ Event-native (различие в управлении потоком исполнения)
2. main.py — god object с 6 ответственностями
3. Replay восстанавливает состояние, но не поведение
4. Rewrite ≠ Refactoring (rewrite опасен, refactoring необходим)
5. Python — правильный стек для event-native ядра EPP
6. TypeScript — правильный стек для плагина (экосистема OpenCode)

---

## 2. Фундаментальный инсайт: Event-Sourced ≠ Event-Native

Это самый ценный architectural insight сессии.

### Определения

**Event-Sourced (текущее состояние EPP)**
```
Компонент A → вызывает компонент B напрямую (gRPC/HTTP)
  Компонент B делает работу
    → Публикует событие: «я сделал работу» (запись, а не управление)
```

События **сохраняются**, но **не управляют потоком исполнения**. Компоненты вызывают друг друга напрямую. Событие — это лог, а не управляющий сигнал.

**Event-Native (целевое состояние)**
```
Компонент A → публикует событие «я хочу, чтобы это произошло»
  Event Bus доставляет событие подписчикам
    → Компонент B получает событие, делает работу
      → Публикует событие «я сделал работу»
```

События **управляют потоком исполнения**. Компоненты не знают друг о друге. Они знают только события и свои типы.

### Ключевое различие

| Характеристика | Event-Sourced | Event-Native |
|----------------|---------------|--------------|
| **Коммуникация** | Прямые вызовы (gRPC, HTTP) | Через Event Bus |
| **Управление потоком** | Код компонента A вызывает B | Событие «запрос» → подписчик B |
| **Событие — это** | Запись о прошлом | Запрос, ответ, запись — всё в одном |
| **Добавление компонента** | Изменение god object | Новый подписчик |
| **Наблюдаемость** | Конечный результат | Каждый шаг — событие |
| **Replay** | Восстанавливает состояние | Воспроизводит поведение |

### Почему это важно

Текущий EPP **думал**, что он event-native. У него есть NATS, события, Ingest. Но реальный flow:

```
Agent → POST /api/execute (HTTP)
  → main.py: Cerbos (gRPC, синхронно)
  → main.py: NATS.publish (одно событие)
  → main.py: Approval/Execution (синхронно)
  → HTTP response
```

Это **RPC с событием в середине**. Не event-native. Осознание этого — первый шаг к исправлению.

---

## 3. First Principles декомпозиция EPP

### Аксиома 0
Мы строим систему управления исполнением. Она:
1. Принимает запрос на операцию
2. Решает, можно ли её выполнить
3. Выполняет (или отклоняет)
4. Запоминает всё так, чтобы можно было восстановить цепочку решений

### Атом 1: Event (Событие)
**Определение:** Атомарный факт, что что-то произошло в определённый момент времени.
**Неделимость:** Нельзя разделить на меньшие факты.
**Примеры:** `tool.invocation`, `decision.issued`, `execution.completed`
**Свойства:** ID, тип, timestamp, payload, source, parentEventId

### Атом 2: Capability (Возможность)
**Определение:** Атомарное право на совершение класса действий.
**Неделимость:** Нельзя дать "половину" capability.
**Типы:** Семантическая (`source_code.read`) и операционная (`fs:read:/src/**`).
**Свойства:** ID, тип, ресурс, ограничения

### Атом 3: Decision (Решение)
**Определение:** Атомарный акт выбора между `allow`, `deny`, `escalate`.
**Неделимость:** Решение принимается мгновенно и целиком.
**Свойства:** Результат, matched policy, reasoning, выданные capabilities

### Атом 4: Policy (Политика)
**Определение:** Правило, которое мапит Event + Context в Decision.
**Неделимость:** Одно правило проверяет одно условие.
**Свойства:** Условие, результат, приоритет, ID

### Атом 5: ProvenanceRecord (Запись происхождения)
**Определение:** Атомарная связь между Event и Decision (или Event и Event).
**Неделимость:** Одно ребро графа между двумя узлами.
**Свойства:** From ID, To ID, тип связи, доказательство (хэш)

### Атомарные операции

| Операция | Input | Output | Описание |
|----------|-------|--------|----------|
| **RECORD** | Event | ProvenanceRecord | Зафиксировать факт |
| **DECIDE** | Event + Context + Policy[] | Decision | Принять решение |
| **ENFORCE** | Decision + Capability[] | Result | Исполнить с ограничениями |
| **PROVE** | Event/Decision ID | ProvenanceRecord[] | Восстановить цепочку |

### Законы композиции

**Закон 1 — Event causality:** Каждый Enforcement порождает минимум 2 события (start, complete/fail). Каждое решение связано событием с запросом.

**Закон 2 — Decision precedes enforcement:** Enforcement не может начаться до решения. Governance — причина, по которой run происходит.

**Закон 3 — Capability is a decision output:** Policy принимает решение на основе Context. Decision выдаёт Capability. Enforcement использует Capability. Не наоборот.

**Закон 4 — Provenance is append-only DAG:** Records образуют DAG. Только INSERT, никогда UPDATE.

**Закон 5 — Boundariness (закон границ):** Каждый слой знает только о своих атомах.
- Semantic Layer: Event, Capability (semantic)
- Governance: Event, Policy, Decision
- Enforcement: Decision, Capability (operational)
- Provenance: Event, Decision, ProvenanceRecord

---

## 4. Проблемы текущей архитектуры EPP

### P1: main.py — god object (6 ответственностей в одной функции)
Текущий `main.py` (~300 строк) делает:
1. Registry lookup (L2)
2. GovernanceEvent construction
3. Cerbos check (L4)
4. NATS publish (L3)
5. Approval creation (если HOLD)
6. Execution (если ALLOW)

**Следствие:** Любое изменение требует модификации main.py. Невозможно заменить компонент независимо.

### P2: Event-sourced, не event-native
Компоненты вызывают друг друга напрямую (Cerbos — gRPC, Executor — прямой вызов). Событие публикуется **после** того, как работа сделана. События не управляют потоком — они только архивируются.

**Следствие:** Нет реактивной композиции. Нет полной наблюдаемости промежуточных шагов.

### P3: Нет подписчиков, есть только один consumer (Ingest)
NATS используется как "запись в архив", а не как event bus для композиции компонентов. Ingest — единственный подписчик.

**Следствие:** Нельзя добавить новый подписчик (semantic resolver, pre-check validator) без изменения main.py.

### P4: Replay восстанавливает состояние, не поведение
Replay читает события из PG и восстанавливает состояние. Но он **не воспроизводит**: Cerbos check, Approval creation, Execution. Нельзя перезапустить Cerbos на исторических данных и сравнить.

**Следствие:** Невозможно протестировать "что если бы мы применили новую политику к прошлым событиям".

### P5: Нет separation PDP/PEP
main.py и принимает решение (PDP), и исполняет (PEP). Это security anti-pattern: enforcement должен следовать за decision, а не быть с ним в одном модуле.

**Следствие:** Нельзя гарантировать, что enforcement соответствует решению. Нельзя заменить enforcement независимо.

### P6: Нет Semantic Layer как отдельного слоя
User Intent / Semantic Resolution отсутствует как отдельный компонент. Онтология операций не формализована. Capability модель использует операционные пути (`/src/**`) вместо семантических имён (`source_code.read`).

**Следствие:** Семантика просачивается в Rego-политики. Онтология не может эволюционировать независимо.

### P7: Нет формальной Event Schema
NATS subject — один (`epp.governance`). Нет разделения event types. Нет схемы для каждого типа. Нет информации о том, кто производит и кто потребляет события.

**Следствие:** Сложно добавлять новые типы событий. Сложно отслеживать因果 chain.

### P8: Нет guaranteed delivery / DLQ / error handling
Если подписчик упал при обработке события — событие потеряно. Нет retry, нет dead letter queue, нет механизма восстановления.

**Следствие:** Ненадёжная композиция. Ошибка в одном подписчике может разрушить цепочку.

---

## 5. Стратегия трёх потоков (Three-Track Evolution)

### Концепция
Мы не строим одну систему. Мы строим три параллельных потока, которые сойдутся в EPP Protocol.

```
Поток A: Текущий EPP (существующий Python-стек)
  ├── RPC-ядро (HTTP API + main.py = god object)
  ├── Цель: Production system
  ├── Задача: рефакторинг в event-native, не теряя функциональности
  └── Время: 2-3 недели на ядро, потом инкрементально

Поток B: Плагин CognitionKernel (новый TypeScript-стек)
  ├── Event-native с первого дня
  ├── Маленькое ядро (5 атомов, 4 операции)
  ├── MCP entry point (adoption через стандарт)
  ├── Sandbox Strategy Pattern (Linux, Docker, macOS, Windows)
  ├── Цель: Reference implementation + OpenCode security
  └── Время: пишется параллельно

Поток C: EPP Protocol (будущее)
  ├── Рождается из слияния опыта Потока A и архитектуры Потока B
  ├── Federation: несколько PDP, несколько PEP, общий Provenance DAG
  ├── Semantic Intelligence Layer
  └── Цель: Универсальный протокол управления исполнением
```

### Почему три потока, а не один

| Аспект | Поток A (EPP) | Поток B (Plugin) |
|--------|--------------|------------------|
| **Язык** | Python | TypeScript |
| **Задача** | Governance substrate для AI | Security plugin для OpenCode |
| **Архитектура** | Refactoring к event-native | Event-native с нуля |
| **Adoption** | Через API + MCP adapter | Через MCP protocol |
| **Риск** | Сломать существующий функционал | Ничего не сломать (новый проект) |
| **Скорость изменений** | Низкая (production) | Высокая (greenfield) |

Плагин — это **experimental branch**, который показывает, как должен выглядеть EPP Core 2.0. Когда Поток A (текущий EPP) будет готов к Phase 6 (перестройка), он не будет гадать, как делать event-native. Он возьмёт архитектуру плагина.

### Схождение: EPP Core Interface

Оба потока реализуют один интерфейс:

```typescript
// EPP Core Interface — то, к чему мы придём
interface EppCore {
  // Точка входа: принять событие
  accept(event: Event): Promise<void>;
  
  // Подписка на результаты
  subscribe(type: EventType, handler: Handler): Subscription;
  
  // Provenance
  getProvenance(eventId: string): Promise<ProvenanceGraph>;
  
  // Admin
  registerCapability(cap: Capability): Promise<void>;
  registerPolicy(policy: Policy): Promise<void>;
}
```

Поток A (Python) реализует через обёртку над существующим ядром.
Поток B (TypeScript) реализует нативно.
Оба говорят на одном языке — событиях.

---

## 6. Архитектурные решения (Architectural Decisions)

### AD-001: Event Bus как фундамент, MCP как entry point

**Решение:** Внутренняя архитектура строится на Event Bus (in-process на старте, NATS в production). MCP — внешний протокол для agent-facing операций, не внутренний механизм.

**Обоснование:** Event Bus даёт:
- Реактивную композицию компонентов
- Полную наблюдаемость (каждый шаг — событие)
- Provenance из коробки (все события хранятся)
- Легкость добавления новых компонентов (подписчиков)

MCP даёт:
- Zero-friction adoption (агенты уже говорят на MCP)
- Стандартный интерфейс для `tools/list` и `tools/call`
- Совместимость со всеми MCP-клиентами

**Контекст:** MCP = транспорт. Event Bus = содержание. Они не конкурируют.

### AD-002: Явное разделение PDP и PEP

**Решение:** Policy Decision Point (PDP) и Policy Enforcement Point (PEP) — разные компоненты, общающиеся через события. PDP не знает, как исполняется. PEP не принимает решений.

**Обоснование:** Security best practice. Позволяет:
- Заменить PDP (OPA → Cerbos → custom) независимо
- Заменить PEP (Linux → Docker → Firecracker) независимо
- Аудировать каждый шаг раздельно
- Тестировать PDP и PEP изолированно

**Контекст:** Текущий EPP нарушает это (main.py = PDP + PEP).

### AD-003: Sandbox как Strategy Pattern

**Решение:** Sandbox — не конкретная реализация, а интерфейс (Strategy). Конкретные стратегии: LinuxNative (Landlock+seccomp), Docker, macOS Sandbox, Windows JobObject.

**Обоснование:**
- Кроссплатформенность (не только Linux)
- Возможность выбирать уровень изоляции под задачу
- Замена без изменения PDP, Semantic Layer, Event Bus

**Контекст:** Sandlock использует Landlock+seccomp жёстко. EPP-плагин делает sandbox подменяемым с первого дня.

### AD-004: Semantic Layer — отдельный слой, не поле в Rego

**Решение:** Semantic Layer — независимый компонент между запросом и решением. Он транслирует семантические capabilities (`source_code.read`) в операционные (`fs:read:/src/**`).

**Обоснование:**
- Онтология может эволюционировать независимо от политик
- Семантика не захардкодена в Rego
- Semantic Layer может использовать LLM для intent resolution
- Чистое разделение concern'ов

**Контекст:** Текущий EPP не имеет Semantic Layer. Онтология не формализована.

### AD-005: Никакого Rewrite — только рефакторинг

**Решение:** Существующий EPP (Python) рефакторится в event-native, а не переписывается с нуля. Новый плагин (TypeScript) пишется с нуля как отдельная сущность.

**Обоснование:**
- Rewrite с нуля: переписываем 1400 рабочих строк, чтобы исправить 400 проблемных
- Рефакторинг: исправляем 400 строк (main.py), сохраняя 1400
- Rewrite = 4-6 недель до feature parity. Рефакторинг = 2-3 недели
- Rewrite теряет knowledge, тесты, граничные случаи
- Плагин — не rewrite, а новая сущность с другой задачей

**Контекст:** Rewrite с нуля оправдан, когда нет тестов, нет границ, нет понимания кода. Ни одно из этих условий не выполняется для текущего EPP.

### AD-006: Python — правильный стек для event-native ядра EPP

**Решение:** Event-native рефакторинг EPP делается на Python. TypeScript — для плагина (экосистема OpenCode).

**Обоснование:**
- Python (asyncio + nats-py) полностью поддерживает event-native:
  - Async pub/sub — нативная возможность
  - NATS JetStream — уже используется
  - Pydantic — уже используется
  - FastAPI — уже используется
- Не требует ни одной новой зависимости
- Тесты не переписываются
- Вся инфраструктура остаётся без изменений (NATS, Cerbos, PG)

**Контекст:** Язык — транспорт. Архитектура — содержание. Event-native — это архитектурный паттерн, не языковая фича.

### AD-007: MCP покрывает agent-facing, API — admin-facing

**Решение:** MCP — entry point для агентов (tools/list, tools/call). REST API — для администраторов и аналитиков (регистрация tools, управление политиками, query аудита, replay).

**Обоснование:**
- MCP не имеет стандартных методов для admin операций
- MCP не предназначен для чтения истории (replay, audit)
- API остаётся для сложных сценариев (multi-agent orchestration, кастомные адаптеры)

**Контекст:** DeepSeek справедливо отметил, что MCP покрывает 100% agent-facing функционала, но не admin-facing.

### AD-008: Replay должен воспроизводить поведение, не только состояние

**Решение:** Event-native replay подаёт события на вход тем же подписчикам, что и в реальном времени. Можно перезапустить Cerbos на исторических данных и сравнить решения.

**Обоснование:**
- "Что если бы мы применили новую политику к прошлому инциденту?"
- "Какое решение принял бы новый semantic resolver?"
- Поведенческий replay = детерминированное воспроизведение всей цепочки

**Контекст:** Текущий replay — только чтение из PG. Не воспроизводит Cerbos check.

### AD-009: Event Schema Registry — необходим для event-native

**Решение:** Каждый тип события имеет формальную схему: поля, кто производит, кто потребляет.

**Обоснование:**
- Без схемы event-native превращается в хаос
- Сложно добавлять новые типы
- Сложно отслеживать causality chain
- Сложно гарантировать обратную совместимость

**Контекст:** Текущий NATS subject — один для всех событий. Это не масштабируется на event-native с десятками типов.

### AD-010: Гарантированная доставка, DLQ, error handling — с первого дня event-native

**Решение:** Event-native архитектура включает at-least-once delivery, dead letter queue, retry policy, timeout handling.

**Обоснование:**
- Если PDP упал при обработке — событие не должно теряться
- Если PEP завис — должен быть timeout
- Ошибка в одном подписчике не должна разрушать цепочку
- NATS JetStream предоставляет эти механизмы из коробки

**Контекст:** Текущая архитектура не имеет guaranteed delivery — событие публикуется один раз и "надеется", что Ingest его подхватит.

---

## 7. Отвергнутые альтернативы

### Альтернатива A: Rewrite EPP с нуля на TypeScript

**Предлагалось:** Переписать EPP целиком на TypeScript с правильной event-native архитектурой.

**Отвергнуто, потому что:**
- Переписывается 1400 рабочих строк ради 400 проблемных
- Потеря тестов, knowledge, граничных случаев
- 4-6 недель без production системы
- Два codebase параллельно = двойная поддержка

**Вместо:** Рефакторинг main.py (400 строк) на том же стеке.

### Альтернатива B: MCP как внутренний event bus

**Предлагалось:** Использовать MCP Protocol не только как внешний транспорт, но и как внутренний механизм коммуникации компонентов.

**Отвергнуто, потому что:**
- MCP — JSON-RPC (request-response), не pub/sub
- MCP не имеет guaranteed delivery, DLQ, replay
- MCP — протокол для агентов, не для компонентов системы

**Вместо:** Event Bus (NATS) для внутренней коммуникации, MCP — только внешний entry point.

### Альтернатива C: Sandlock-style per-tool sandbox без governance layer

**Предлагалось:** Использовать Sandlock MCP как готовое решение для sandboxing.

**Отвергнуто, потому что:**
- Sandlock не имеет governance layer (policy engine, approval flows)
- Sandlock не имеет semantic layer
- Sandlock не имеет provenance (только логи)
- Sandlock жёстко Landlock+seccomp (только Linux)

**Вместо:** Sandbox Strategy Pattern с governance PDP над ним.

### Альтернатива D: OPA как единственный PDP

**Предлагалось:** Использовать только OPA/Rego для всех решений.

**Отвергнуто, потому что:**
- OPA не поддерживает approval flows (HOLD/ESCLATE)
- OPA не имеет встроенного semantic reasoning
- Cerbos уже работает в production и имеет преимущества для RBAC

**Вместо:** Policy Engine Interface с возможностью использовать Cerbos, OPA, или кастомный PDP.

### Альтернатива E: Добавить semantic layer как поле в Rego

**Предлагалось:** User Intent = поле `input.user_intent` в Rego-политиках.

**Отвергнуто, потому что:**
- Semantic concern просачивается в execution concern
- Онтология не может эволюционировать независимо от политик
- Нельзя использовать LLM для intent resolution в Rego

**Вместо:** Semantic Layer как отдельный подписчик Event Bus между запросом и PDP.

---

## 8. Модель зрелости EPP Core

### Уровень 0: RPC (текущее состояние)
- HTTP API как единственный entry point
- main.py — god object (PDP + PEP вместе)
- NATS — только для архивирования
- Replay — только чтение из PG
- Нет Semantic Layer
- Нет формальной Event Schema

### Уровень 1: Event-Enhanced (цель рефакторинга)
- main.py разделён: публикует событие, ничего больше
- Подписчики: PDP (Cerbos), PEP (Executor)
- NATS: управляющий поток + архив
- Event Schema: формальные схемы для каждого типа
- Гарантированная доставка + DLQ
- Semantic Layer: pass-through (заглушка)

### Уровень 2: Event-Native
- Semantic Layer: intent resolution
- Provenance: полный DAG из событий
- Replay: воспроизводит поведение (подаёт события подписчикам)
- Sandbox: strategy pattern (Linux, Docker, macOS)
- PDP: pluggable (Cerbos, OPA, custom)

### Уровень 3: Event-Native + Federation
- Несколько PDP (разные политики для разных доменов)
- Несколько PEP (разные стратегии для разных сред)
- Общий Provenance DAG
- EPP Protocol: сетевой протокол для распределённой архитектуры
- Semantic Intelligence: онтология + LLM-augmented resolution

---

## 9. Глоссарий: синхронизация терминов

Термины, которые мы синхронизировали в этой сессии. Использовать в будущих обсуждениях без переопределения.

| Термин | Определение | Контекст |
|--------|-------------|----------|
| **Event** | Атомарный факт, что что-то произошло | Первоклассная сущность, не лог |
| **Event-Native** | Архитектура, где события управляют потоком исполнения | Цель рефакторинга |
| **Event-Sourced** | Архитектура, где события сохраняются, но не управляют потоком | Текущее состояние EPP |
| **PDP** | Policy Decision Point — принимает решения | Отдельный компонент |
| **PEP** | Policy Enforcement Point — исполняет решения | Отдельный компонент |
| **Capability (Semantic)** | `source_code.read` — что операция значит | Semantic Layer |
| **Capability (Operational)** | `fs:read:/src/**` — что операция делает на уровне OS | Enforcement Layer |
| **Sandbox Strategy** | Интерфейс изоляции, реализуемый под платформу | PEP component |
| **Provenance** | DAG событий + решений, восстанавливающий цепочку | First-class entity |
| **Behaviour Replay** | Replay, который подаёт события подписчикам | Цель (не только состояние) |
| **State Replay** | Replay, который только читает из PG | Текущее состояние |
| **MCP** | Внешний протокол для agent-facing операций | Не внутренний механизм |
| **Semantic Layer** | Слой между запросом и решением, определяющий intent | Отдельный подписчик |
| **Boundariness** | Закон: каждый слой знает только о своих атомах | Architectural invariant |
| **main.py god object** | Функция, делающая registry + cerbos + nats + approval + execution | Что рефакторим |
| **Rewrite** | Полная перепись кода с потерей существующего | Отвергнуто для EPP |
| **Refactoring** | Изменение структуры без изменения функциональности | Выбрано для EPP |
| **Three-Track Evolution** | Поток A (EPP), Поток B (Plugin), Поток C (Protocol) | Стратегия развития |
| **Architectural Honesty** | Признание, что система не event-native, хотя думала иначе | Ценность, проявленная в сессии |

---

## 10. Приложение: Критический анализ подходов

### Sandlock MCP (что предлагает README.md проекта CognitionKernel)

**Сильные стороны:**
- Per-tool sandboxing — правильная идея
- Landlock + seccomp — правильные механизмы
- MCP — правильный entry point

**Проблемы:**
- Нет event-native архитектуры (RPC-native)
- Нет provenance как first-class системы
- Semantic в Rego-политиках (contamination)
- Франкенштейновская архитектура (TS + Rust + OPA + JSONL)
- Capability модель — операционная (пути), не семантическая

**Вердикт:** Хорошая мотивация, проблемная архитектура. Плагин нужно строить на event-native принципах, не на RPC.

### DeepSeek MCP-over-RPC (критический анализ)

**Сильные стороны:**
- Правильный диагноз: adoption через MCP
- Правильная оценка: MCP покрывает agent-facing
- Правильное решение: API для admin, MCP для агентов

**Проблемы:**
- "Ничего не менять L2-L6" — архитектурная ошибка (main.py = god object остаётся)
- Нет event-native ядра — MCP снаружи, RPC внутри
- Нет разделения PDP/PEP
- Нет Semantic Layer
- Sandbox = "Stub Executor" (нет strategy pattern)
- "3-5 дней" — опасное упрощение (реально 2-3 недели с event schema, DLQ, deployment)

**Вердикт:** DeepSeek прав в стратегии adoption, но неправ в архитектуре реализации. MCP снаружи — да. RPC-монолит внутри — нет.

### Наше решение (синтез)

| Аспект | Берём от DeepSeek | Берём от Sandlock | Добавляем от себя |
|--------|------------------|-------------------|-------------------|
| Entry point | MCP как стандарт | Per-tool sandboxing | Event Bus внутри |
| PDP | — | — | Отдельный подписчик |
| PEP | — | Landlock+seccomp | Strategy Pattern |
| Semantic | — | — | Отдельный слой |
| Provenance | — | — | DAG из событий |
| Capability | — | — | Semantic + Operational |
| Architecture | MCP adoption | Motivation | Event-native ядро |

---

## Примечание для будущих сессий

Этот документ — **живой референс**. Он фиксирует:
- Инсайты, которые мы не хотим потерять
- Решения, которые мы приняли (и почему)
- Альтернативы, которые мы отвергли (и почему)
- Термины, которые мы синхронизировали
- Проблемы, которые мы обнаружили

**Не переопределяй термины из этого документа без явного обсуждения.**
**Не принимай решений, противоречащих AD, без явного пересмотра.**
**Не предлагай rewrite EPP — это отвергнутая альтернатива.**

Следующие шаги после этой сессии:
1. **Спецификация рефакторинга main.py** — файлы, NATS subjects, подписчики
2. **Спецификация плагина CognitionKernel** — event-native ядро на TypeScript
3. **Event Schema Registry** — формальные схемы для всех типов событий
4. **PDP/PEP Interface** — контракт между точкой решения и точкой исполнения

---

*Документ создан 4 июня 2026 на основе архитектурной сессии Chief Architect EPP и Product Lead.*
*Следующая сессия: спецификация рефакторинга main.py.*

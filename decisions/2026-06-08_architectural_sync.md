# Architectural Sync — 2026-06-08

**Участники:** Product Lead / Chief Architect EPP  
**Повод:** Рецензия корпуса документов CognitionKernel (9 документов, ~5300 строк)  
**Статус:** Синхронизация завершена, решения приняты, точки расхождения зафиксированы

---

## 1. Что обсуждали

### 1.1. Исходные документы (CognitionKernel/)

| Файл | Строк | Суть |
|------|-------|------|
| `CognitionKernel/README.md` | 799 | Исходное предложение: Security Governance Layer для OpenCode |
| `CognitionKernel/ARCHITECTURE_PROPOSAL.md` | 634 | Архитектурная рецензия README (от Chief Architect) |
| `epp/specs/future/EPP_FULL_VISION.md` | 476 | EPP как Execution Provenance Platform |
| `epp/specs/future/ECOSYSTEM_VISION.md` | 469 | Платформенная модель: 5 вертикалей, marketplace |
| `epp/specs/future/EPP_VISION_FROM_USER.md` | 457 | User-centric описание продукта |
| `epp/specs/future/MONETIZATION_STRATEGY.md` | 387 | Бизнес-модель: Open Source Core + Enterprise Subscription |
| `epp/specs/future/REFACTORING_REFERENCE.md` | 607 | Архитектурные решения, event-sourced vs event-native |
| `epp/specs/future/REFACTORING_PROCESS.md` | 827 | Пошаговый план рефакторинга main.py |
| `epp/specs/future/TESTING_FIRST_PRINCIPLES.md` | 628 | Таксономия тестов для рефакторинга |

### 1.2. Ключевой инсайт (принят обеими сторонами)

**EPP — event-sourced, не event-native.**

События сохраняются (NATS → Ingest → PG), но НЕ управляют потоком исполнения.  
`main.py` — god object с 6 ответственностями подряд (registry → cerbos → nats → approval → execution → response).

Цель рефакторинга: **main.py → thin publisher**, все компоненты — подписчики на NATS/event bus.

---

## 2. Что приняли как решение ✅

### R1. Three-Track Evolution Strategy

| Поток | Что | Язык | Статус |
|-------|-----|------|--------|
| **A** | Рефакторинг текущего EPP (event-sourced → event-native) | Python | **Утверждён. Начинаем.** |
| **B** | CognitionKernel Plugin (reference implementation для Core 2.0) | TypeScript | **Утверждён концептуально. Старт — после стабилизации Потока A.** |
| **C** | EPP Protocol (федерация PDP/PEP) | TBD | **Отложен. Всего лишь концепт.** |

### R2. **Никакого Rewrite EPP**

Существующий EPP (Python) НЕ переписывается с нуля. Рефакторинг ~400 строк main.py, сохраняя ~1400 рабочих строк + тесты + knowledge.

### R3. **14 Characterization Tests до рефакторинга**

Перед любыми изменениями — написать 14 тестов (см. `TESTING_FIRST_PRINCIPLES.md` Атом 5), которые фиксируют as-is поведение `POST /events/tool-call`. Каждый шаг рефакторинга — тесты остаются зелёными.

### R4. **PDP/PEP — явное разделение**

Policy Decision Point (Cerbos) и Policy Enforcement Point (Executor) — разные подписчики, общаются только через события. Никаких прямых вызовов между ними.

### R5. **Semantic Layer — отдельный, но как L7 lens**

Semantic Layer (intent resolution, ontology mapping) — НЕ отдельный подписчик, а L7 lens на существующем Semantic Runtime. Уже есть live loop, lens registry, replay — расширяем, не дублируем.

### R6. **BSL 1.1 + Free/Team/Enterprise**

Модель монетизации принята:
- **Free** ($0) — локальный MCP Server, sandbox, локальный audit
- **Team** ($15/user/mo) — общие политики, облачный аудит, SSO
- **Enterprise** (custom) — self-hosted, compliance, on-prem PDP

### R7. **7 шагов рефакторинга (из REFACTORING_PROCESS.md)**

| Шаг | Что делаем | Код |
|-----|-----------|-----|
| 0 | **14 characterization tests** | `tests/characterization/` |
| 1 | NATS Subject Schema + Event Schemas | `app/events/subjects.py`, `app/events/schemas.py` |
| 2 | Semantic Handler (уже есть как L7 live loop) | `app/l7/` |
| 3 | PDP Handler (Cerbos из subscriber) | `app/events/handlers/pdp_handler.py` |
| 4 | Approval Handler (HOLD из subscriber) | `app/events/handlers/approval_handler.py` |
| 5 | PEP Handler (Executor из subscriber) | `app/events/handlers/pep_handler.py` |
| 6 | API → thin publisher (main.py чист) | `app/main.py` |
| 7 | Удаление старого кода | `app/main.py` (cleanup), `app/approval/router.py` (если нужно) |

---

## 3. Что скорректировали (относительно исходных документов) 🔧

### C1. L7 — уже существующий subscriber

В `REFACTORING_PROCESS.md` не учтён L7 Semantic Runtime, который уже является подписчиком на L3.  
**Поправка:** L7 — уже существующий subscriber, не строить с нуля. Интегрировать с новыми NATS subjects.

### C2. In-process event bus для PDP/PEP на старте

`REFACTORING_PROCESS.md` предлагает всё через NATS subjects.  
**Поправка:** На Phase 6 (рефакторинг) PDP и PEP — в том же процессе, через in-process event bus (asyncio.Queue). NATS — для внешних подписчиков (L6, L7, будущий L8). Гибрид, не всё через NATS.

### C3. 56 contract tests — premature

`TESTING_FIRST_PRINCIPLES.md` предлагает 56 contract tests для всех подписчиков.  
**Поправка:** Начинаем с 14 characterization tests. Contract tests (не 56, а ~20) — после стабилизации контрактов, не до.

### C4. Semantic Layer = L7 lens, не отдельный subscriber

Исходные документы рисуют Semantic Handler как отдельного подписчика.  
**Поправка:** Semantic Layer — это L7 lens. Intent resolution — ещё один lens в LensRegistry. Не строим новый слой, расширяем существующий L7.

### C5. Monetization не влияет на архитектуру Phase 6

Монетизация (Free/Team/Enterprise) — продуктовая стратегия, а не архитектурное решение.  
**Подтверждение:** Архитектура EPP Core не меняется от того, продаём мы подписку или нет. Cloud Sync Agent — опциональный модуль, не влияет на refactoring.

---

## 4. На чём остановились 🛑

### Immediate next step

**Начать с Шага 0:** написать 14 characterization tests для `POST /events/tool-call` против текущего EPP.

Эти тесты:
- Фиксируют as-is поведение (14 observable outcomes из decision tree main.py)
- Должны быть зелёными ДО начала рефакторинга
- Остаются зелёными НА КАЖДОМ шаге рефакторинга
- Удаляются ПОСЛЕ завершения всех 7 шагов

### Что НЕ делаем сейчас

- ❌ Не пишем 56 contract tests (только 14 characterization)
- ❌ Не строим новый Semantic Layer subscriber (расширяем L7 lens)
- ❌ Не трогаем CognitionKernel Plugin (Поток B — после стабилизации Потока A)
- ❌ Не трогаем Arch refactoring plan (остаётся как reference, начинаем с Шага 0)

### Что остаётся открытым (post-Phase 6)

- **EPP Cloud** (Team/Enterprise) — продукт, не архитектура. Когда будет готов Core.
- **CognitionKernel Plugin** — TypeScript reference implementation. После стабилизации Потока A.
- **EPP Protocol** — федерация. После Потока B.
- **56 contract tests** — после стабилизации event-native контрактов.

---

## 5. Важные architectural constraints (для памяти)

- **main.py** после рефакторинга: только publish event + return trace_id. Всё.
- **Cerbos** остаётся тем же gRPC вызовом, но из PDP subscriber, не из main.py.
- **NATS** — управляющая шина для подписчиков, не архив.
- **L3 (canonical truth)** — не меняется. Типы событий меняются, но immutable L3 принцип остаётся.
- **Нижние слои не зависят от верхних** — L3 не знает про L7, L7 не знает про L8 — инвариант остаётся.
- **Characterization tests** — временные. После рефакторинга — удалить.

---

*Документ создан 8 июня 2026. Замораживает состояние обсуждения перед началом Phase 6 рефакторинга.*

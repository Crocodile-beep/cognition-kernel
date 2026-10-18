# ADR: Event-Native Refactoring Before MCP Adapter

> **Дата:** 4 июня 2026  
> **Тип:** Architecture Decision Record  
> **Статус:** Accepted  
> **Context:** Сессия синхронизации видения EPP: плагин vs event-native refactoring  

---

## Контекст

В ходе архитектурной сессии обсуждался порядок реализации двух связанных потоков:

- **Поток A**: Event-native рефакторинг EPP Core (Python, main.py → PDP/PEP subscribers)
- **Поток B**: CognitionKernel Plugin (TypeScript, MCP Server для OpenCode)

Возник вопрос: что делать сначала?

### Альтернативы

| Альтернатива | Порядок | Суть |
|-------------|---------|------|
| **A** (рассматривалась) | Плагин → Refactoring | Сначала написать плагин на TypeScript с event-native архитектурой, потом refactor'ить EPP Core под его архитектуру |
| **B** (выбрана) | Refactoring → Плагин | Сначала стабилизировать EPP Core как event-native, потом сделать плагин как тонкий MCP-адаптер поверх EPP Event Bus |

---

## Решение

**Выбрана Альтернатива B:** Сначала event-native рефакторинг EPP Core, затем MCP адаптер как тонкий слой (wrapper) поверх.

```
Порядок:
  Шаг 1: EPP Refactoring (main.py → event-native)
  Шаг 2: EPP стабилизирован (14 char tests + 56 contract tests — зелёные)
  Шаг 3: EPP MCP Adapter (~200 строк, wrapper над Event Bus)
  Шаг 4: Плагин автоматически получает все фичи EPP Core
```

---

## Reasoning (почему именно так)

### R1: Один codebase — одна архитектура

Если плагин пишется до стабилизации EPP, то:
- Плагин имеет свою архитектуру (TypeScript, свой event bus, свои типы)
- EPP имеет другую архитектуру (Python, RPC, старые типы)
- Возникает **architectural drift** — два codebase говорят на разных языках
- Плагин не может стать reference, потому что он расходится с production

Если EPP стабилизирован первым:
- Одна архитектура на всю систему
- Плагин — wrapper, а не второй codebase
- Нет drift, потому что плагин ничего не решает — он только преобразует протокол

### R2: Плагин как validation suite refactoring'а

Когда EPP refactoring проходит шаг за шагом:
- Каждый шаг меняет EPP Core
- Плагин подключается к EPP Event Bus и вызывает tools из OpenCode
- Если refactoring сломал поведение — characterization tests падают
- Если тесты зелёные — плагин работает корректно **автоматически**

Плагин не «после refactoring». Плагин — **validation suite**, который доказывает, что refactoring успешен.

### R3: Плагин не может устареть

Плагин не содержит логики. Он содержит только:
- Преобразование MCP JSON-RPC → событие Event Bus
- Ожидание ответа из Event Bus
- Преобразование ответа → MCP JSON-RPC

Если EPP Core эволюционирует (новые event types, новый PDP, новый sandbox) — плагин эволюционирует автоматически, потому что он просто публикует события в тот же Event Bus.

### R4: Protocol agnostic

Если плагин — wrapper, то:
- MCP — не единственный возможный протокол
- Если завтра появится A2P, AgentProtocol, или другой — пишется новый adapter
- EPP Core не меняется
- Нет vendor lock-in на MCP

```
MCP Adapter     → EPP Event Bus
A2P Adapter     → EPP Event Bus (будущее)
Custom Protocol → EPP Event Bus (будущее)
```

### R5: Инженерная честность

Плагин до стабилизации EPP — это **product reasoning**:
- «Хочу показать продукт раньше»
- «Хочу revenue быстрее»
- «Хочу проверить архитектуру на реальном коде»

Refactoring до плагина — это **engineering reasoning**:
- «Сначала построить фундамент, потом надстройку»
- «Одна архитектура — меньше complexity»
- «Refactoring под нагрузкой (real agent calls) — лучшая validation»

Для стадии Dev (не revenue-driven) правильный выбор — engineering reasoning.

---

## Последствия

### Положительные

| Аспект | Результат |
|--------|-----------|
| Архитектура | Единая event-native архитектура на всю систему |
| Поддерживаемость | Плагин — 200 строк, не требует отдельной архитектуры |
| Protocol agnostic | Любой протокол может быть адаптирован |
| Testing | Refactoring валидируется сразу через MCP |
| Evolution | EPP Core меняется — плагин следует автоматически |

### Отрицательные

| Аспект | Риск |
|--------|------|
| Время до первого MCP demo | Плагин появится через 2-3 недели (после refactoring), а не через 2-3 дня |
| OpenCode community | Ранний доступ к плагину откладывается |
| Feedback loop | Реальная обратная связь от пользователей — позже |

### Mitigation

Плагин — 200 строк. После стабилизации EPP он пишется за 1-2 дня. Потеря времени — не месяцы, а дни.

---

## Согласованный план

```
Фаза 0: Подготовка (сейчас)
  └── Контекст: REFACTORING_REFERENCE.md, REFACTORING_PROCESS.md
  └── Тесты: TESTING_FIRST_PRINCIPLES.md (14 char + 56 contract)
  └── Видение: EPP_FULL_VISION.md

Фаза 1: EPP Refactoring (~2-3 недели)
  └── main.py → Event Bus publisher (30 строк)
  └── PDP subscriber (Cerbos)
  └── PEP subscriber (Sandbox)
  └── Provenance subscriber
  └── Semantic Layer (pass-through v0)
  └── 14 characterization tests — страховка
  └── 56 contract tests — permanent

Фаза 2: EPP Stable (checkpoint)
  └── Все characterization tests зелёные
  └── Все contract tests зелёные
  └── Существующие тесты зелёные
  └── Event-native ядро работает

Фаза 3: EPP MCP Adapter (~1-2 дня)
  └── MCP Protocol Handler (stdio)
  └── Преобразует MCP → Event Bus
  └── Публикует tool.invocation.requested
  └── Ждёт execution.completed / decision.issued
  └── Отдаёт результат агенту

Фаза 4: Итерации
  └── EPP: Sandbox Strategy Pattern
  └── EPP: Semantic Layer
  └── EPP: L7 Lens
  └── Плагин: автоматически получает всё
```

---

## Ключевая цитата

> *«Плагин — не отдельный продукт. Плагин — дверь к EPP для MCP-агентов. Дверь не строят до дома.»*

*Решение принято 4 июня 2026.*

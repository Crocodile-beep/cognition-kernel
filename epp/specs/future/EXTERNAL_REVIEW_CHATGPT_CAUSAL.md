# External Review: Causal Governance — Temporal Dimension

> **Источник:** ChatGPT (external AI review)  
> **Контекст:** Анализ решения о policy versioning для faithful replay  
> **Дата:** 5 июня 2026  
> **Статус:** Reviewed, principles accepted as constitutional

---

## Key Insight: Governance without temporal causality is incomplete

Enterprise не спрашивает "что произошло?". Enterprise спрашивает **"почему система тогда разрешила это?"**.

Большинство governance систем:
- логируют решение (ALLOW/DENY)
- логируют событие (что произошло)
- НО не могут восстановить причинность решения (какая policy, какой версии, в каком контексте)

Без этого: audit неполный, replay ложный, explainability фальшивая.

---

## The Shift

| Было | Стало |
|------|-------|
| Event-native governance runtime | **Causally replayable governance substrate** |
| Runtime governance | **Temporal governance** |
| Bus = observability | Bus = immutable timeline + causal anchors |

---

## Three Constitutional Principles

| # | Principle | Why | Violation |
|---|-----------|-----|-----------|
| 1 | **Events immutable** | Bus append-only. Replay foundation. | Недетерминированный replay |
| 2 | **Policy versioned** | Minimal causal link: `policy_version` in event. | Audit без "почему" |
| 3 | **Replay side-effect free** | Replay анализирует, не исполняет. | Temporal paradoxes |

---

## The Minimal Causal Link

```json
{
  "decision": "DENY",
  "policy_version": "abc123"
}
```

Почему это правильный minimax choice:
- Дешёвый (одно поле)
- Immutable (version — hash)
- Replayable (git checkout policy_version)
- Explainable ("вот policy на момент решения")
- Audit-safe ("вот почему")
- Не раздувает event
- Не делает Bus stateful
- Не тащит policy внутрь runtime

---

## Future Capabilities (not now, but enabled)

| Capability | Description |
|-----------|-------------|
| **Faithful replay** | Replay world state + governance state + authorization reality at T₀ |
| **Counterfactual governance** | "What if policy v7 existed at t=0?" — simulate without executing |
| **Governance diffing** | Policy v3 → v4: +12% allowed, -43 denied, 7 new escalation paths |
| **Compliance time-travel** | "All decisions under policy v12, Jan-Feb, affecting capability X" |

---

## Dangers (Guardrails)

| Danger | Why | Guard |
|--------|-----|-------|
| Policy snapshots in every event | Гигантские events, duplicated state, schema drift | ❌ Не делаем. Version only. |
| Bus becomes authoritative state | Distributed config platform, governance Kubernetes | ❌ Bus = timeline, not state. |
| Replay becomes execution | Temporal paradoxes, accidental re-execution, production impact | ❌ Replay = model, not runtime. |

---

## Strategic Validation

> *"Вы не просто решили 'как хранить policy'. Вы обнаружили, что governance without temporal causality is incomplete. Большинство AI governance стартапов до этого вообще не доходят."*

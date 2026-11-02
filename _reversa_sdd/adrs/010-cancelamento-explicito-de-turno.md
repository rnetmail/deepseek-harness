# ADR 010 — cancelamento explicito de turno

- **Status:** Retroativo
- **Data:** 2026-07-16
- **Fonte:** `.agents/notes/implemented/architecture/2026-07-16-explicit-turn-cancellation.md`
- **Confiança:** 🟢 nota de design implementada

## Contexto
O cancelamento precisa ser previsível em todas as fases do turno.

## Decisão
Um `AbortController` por turno, do claim do inbox até antes de `turn/end`. Causa `{user|parent}` só em runtime, desfecho durável `{aborted}`.

## Consequências
Ctrl+C interrompe o modelo, as ferramentas e os subagentes de forma coerente.

## Implicação para o harness Rust
No Rust: `CancellationToken` hierárquico (turno → step → tool → subprocesso).

# ADR 009 — compactacao por pressao pos chamada

- **Status:** Retroativo
- **Data:** 2026-07-10
- **Fonte:** `.agents/notes/implemented/architecture/2026-07-10-after-call-compaction-pressure-and-overflow-recovery.md`
- **Confiança:** 🟢 nota de design implementada

## Contexto
Janelas de contexto estouram em sessões longas.

## Decisão
A pressão é medida no próximo pre-step, com estado completo (sem quebrar pares tool-call/result). Compacta em 80% e retém 16%. Overflow do provedor dispara compactação e uma nova tentativa limitada.

## Consequências
Sessões longas sem perda de pares e com resumo estruturado.

## Implicação para o harness Rust
Manter, com limiar por modelo (os modelos gratuitos costumam ter janelas menores).

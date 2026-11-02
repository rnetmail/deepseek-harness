# ADR 001 — sessoes event sourced

- **Status:** Retroativo
- **Data:** 2026-06-11
- **Fonte:** `.agents/notes/implemented/architecture/2026-06-11-event-sourced-sessions.md`
- **Confiança:** 🟢 nota de design implementada

## Contexto
O estado da conversa precisa suportar replay, fork, resume e auditoria.

## Decisão
A Session é um log append-only de eventos tipados e a única fonte da verdade. O histórico de mensagens é derivado (`deriveMessages`). O append é síncrono e a persistência faz write-behind até o checkpoint `session/flush`.

## Consequências
Replay e fork viram "semear uma sessão nova com o log". As projeções são folds puros. Exige um formato versionado e migrações explícitas.

## Implicação para o harness Rust
Manter no Rust: `Vec<SessionEvent>` + writer JSONL zstd assíncrono com barreiras de flush.

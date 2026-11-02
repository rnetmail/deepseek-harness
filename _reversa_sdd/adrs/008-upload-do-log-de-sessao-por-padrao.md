# ADR 008 — upload do log de sessao por padrao

- **Status:** Retroativo (rejeitado para o Rust)
- **Data:** 2026-09-14
- **Fonte:** `.agents/notes/implemented/architecture/2026-09-14-session-log-upload-default.md`
- **Confiança:** 🟢 nota de design implementada

## Contexto
A DeepSeek queria diagnósticos de requisições oficiais.

## Decisão
`session-log-deepseek.enabled` = true por padrão em todo processo. Inventário de plugins e telemetria OTel `FEEDBACK_ONLY` também ligados.

## Consequências
O histórico completo da sessão sai da máquina sem opt-in explícito.

## Implicação para o harness Rust
**Rejeitado** pelo usuário em 2026-11-01. Ver ADR 101. O dsh atual foi neutralizado via patch e `DSH_TELEMETRY_DISABLED=1`.

# ADR 011 — browser e computer use como providers

- **Status:** Retroativo
- **Data:** 2026-09-12
- **Fonte:** `.agents/notes/implemented/architecture/2026-09-12-browser-use-provider-registration.md`
- **Confiança:** 🟢 nota de design implementada

## Contexto
O agente precisa operar navegador e desktop.

## Decisão
Serviços `browserUse` e `computerUse` registram um provider (Playwright MCP, Chrome DevTools MCP, Stagehand, Cua Driver). Experimentais e opt-in.

## Consequências
Recursos presos à sessão viva. Um browser externo fica reservado a uma sessão.

## Implicação para o harness Rust
No Rust: pré-registrar Playwright MCP / Chrome DevTools MCP (instalação via npm, como o REA). Desligados por padrão.

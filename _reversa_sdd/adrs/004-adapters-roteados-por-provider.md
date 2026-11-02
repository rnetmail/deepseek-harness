# ADR 004 — adapters roteados por provider

- **Status:** Retroativo
- **Data:** 2026-07-14
- **Fonte:** `.agents/notes/implemented/architecture/2026-07-14-provider-routed-llm-adapters.md`
- **Confiança:** 🟢 nota de design implementada

## Contexto
Vários provedores e modelos coexistem, e a sessão pode trocar de modelo.

## Decisão
O provider é a chave de registro do adapter. O par provider/model é obrigatório por requisição. Cada provider tem exatamente um dono. O catálogo de modelos é consultivo.

## Consequências
Troca de modelo por step. Conflito de dono = erro de configuração.

## Implicação para o harness Rust
No Rust: o mesmo registro, com catálogo de modelos gratuitos e de assinatura atualizável (`<bin> models`).

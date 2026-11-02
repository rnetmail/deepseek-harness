# ADR 012 — perfis e bundles

- **Status:** Retroativo
- **Data:** 2026-08-05
- **Fonte:** `.agents/notes/implemented/architecture/2026-08-05-profile-plugin-bundles.md`
- **Confiança:** 🟢 nota de design implementada

## Contexto
Um único launcher precisa servir web, headless, ACP e SDK.

## Decisão
Perfil = pilha ordenada de bundles + patches. `dsh --profile`.

## Consequências
Flexível, mas o padrão deixa hooks, MCP, LSP e terminal fora do `base`.

## Implicação para o harness Rust
No Rust: subcomandos do binário único + feature flags em config. Hooks, MCP, LSP e terminal ligados por padrão.

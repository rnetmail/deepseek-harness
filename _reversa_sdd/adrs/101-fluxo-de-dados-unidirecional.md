# ADR 101 — fluxo de dados unidirecional

- **Status:** Aceito (novo)
- **Data:** 2026-11-01
- **Fonte:** `decisão do usuário (state.json: no-outbound-deepseek, upstream-tracking)`
- **Confiança:** 🟢 decisão registrada com o usuário

## Contexto
O harness Rust é a alternativa ao Claude e lida com código corporativo (Totvs AGP).

## Decisão
Nenhum dado sai para a DeepSeek ou para terceiros sem opt-in explícito: sem upload de log, inventário ou telemetria. No sentido inverso, o projeto acompanha as releases do deepseek-harness e porta melhorias e correções.

## Consequências
Privacidade por padrão. Exige um processo de acompanhamento do upstream (diff de releases → issues de porte).

## Implicação para o harness Rust
Implementar o processo `upstream-sync` (Writer/Reconstructor).

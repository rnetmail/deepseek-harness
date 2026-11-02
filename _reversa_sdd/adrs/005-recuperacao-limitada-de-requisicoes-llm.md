# ADR 005 — recuperacao limitada de requisicoes llm

- **Status:** Retroativo
- **Data:** 2026-06-21
- **Fonte:** `.agents/notes/implemented/architecture/2026-06-21-bounded-llm-request-recovery.md`
- **Confiança:** 🟢 nota de design implementada

## Contexto
Provedores falham (429, 5xx, rede) e retries ilimitados geram custo e loops.

## Decisão
Falha é um fato serializável (`LlmFailure`). O retry é uma política separada e limitada (5×, 500 ms–10 s, jitter de 10%).

## Consequências
Comportamento previsível. A falha terminal chega ao usuário com o id da requisição do provedor.

## Implicação para o harness Rust
Acrescentar no Rust: **fallback de modelo** configurável (exemplo: assinatura esgotada → modelo gratuito), essencial para o objetivo de alternativa ao Claude.

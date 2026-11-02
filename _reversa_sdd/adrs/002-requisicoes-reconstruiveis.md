# ADR 002 — requisicoes reconstruiveis

- **Status:** Retroativo
- **Data:** 2026-07-05
- **Fonte:** `.agents/notes/implemented/architecture/2026-07-05-reconstructable-requests.md`
- **Confiança:** 🟢 nota de design implementada

## Contexto
É preciso auditar o que o modelo viu e maximizar o prefix cache.

## Decisão
Visível ao modelo ⟺ referenciado de forma durável. A requisição é função pura do log + objetos endereçados por conteúdo.

## Consequências
Prefix cache estável por construção, auditoria byte a byte, drift atribuível em resume e fork.

## Implicação para o harness Rust
Manter: crucial para o custo com provedores pagos e o cache de prompts da Anthropic e da OpenAI.

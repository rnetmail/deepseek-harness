# ADR 102 — integracoes mcp gerenciadas

- **Status:** Aceito (novo)
- **Data:** 2026-11-01
- **Fonte:** `decisões rea-integration, gbrain-integration`
- **Confiança:** 🟢 decisão registrada com o usuário

## Contexto
REA e GBrain trazem capacidades críticas (engenharia reversa e memória) em Node e Bun.

## Decisão
Pré-registrar como servidores MCP gerenciados pelo harness, que instala, atualiza e diagnostica (`doctor`). REA via npm (`rea-agents`), GBrain via `bun install -g github:garrytan/gbrain#latest-stable` com PGLite. Memorable desligado.

## Consequências
O binário Rust continua único. Node ≥ 22.23 e Bun ≥ 1.4.4 viram dependências opcionais detectadas.

## Implicação para o harness Rust
REA aceita como alvo uma URL ou um caminho local, em qualquer projeto.

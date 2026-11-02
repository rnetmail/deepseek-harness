# ADR 103 — compatibilidade nativa claude code

- **Status:** Aceito (novo)
- **Data:** 2026-11-01
- **Fonte:** `decisão claude-plugin-format + requisito Totvs AGP`
- **Confiança:** 🟢 decisão registrada com o usuário

## Contexto
Agentes e conteúdos do Totvs AGP (VSCode + Claude) devem ser reaproveitados automaticamente.

## Decisão
Ler nativamente `CLAUDE.md`, `.claude/agents/*.md`, `.claude/skills/`, `.claude/commands/`, `.claude/settings.json` (permissions, hooks), `.mcp.json` e os plugins/marketplaces do Claude Code. Usar os **mesmos nomes de ferramenta** do Claude Code.

## Consequências
Corrige as lacunas do dsh: skills fora de `.claude`, sem `.claude/agents`, sem `.mcp.json`, nomes de ferramenta em minúsculas.

## Implicação para o harness Rust
Spec de compatibilidade na Fase 4.

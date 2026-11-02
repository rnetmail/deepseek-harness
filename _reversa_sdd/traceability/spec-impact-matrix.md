# Spec Impact Matrix — componentes do harness Rust

Leitura: a linha **impacta** a coluna. ● forte (mudança de contrato quebra), ○ fraca (ajuste de configuração ou comportamento).

| ↓ impacta → | core | llm | tools | sandbox | mcp | compat | integr. | script | store | tui | acp | sdk |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| **core** (loop, inbox, eventos) | — | ● | ● | ○ | ● | ○ | | ● | ● | ● | ● | ● |
| **llm** (adapters, catálogo, OAuth) | ● | — | | | | ○ | | | ○ | ○ | ○ | ○ |
| **tools** (nomes Claude, schemas) | ● | | — | ● | | ● | | ● | ○ | ● | ○ | ○ |
| **sandbox** | | | ● | — | ○ | ○ | ○ | ● | | ○ | | |
| **mcp** (cliente) | ● | | ○ | | — | ● | ● | | | ○ | ○ | |
| **compat** (.claude/*) | ● | ○ | ● | ○ | ● | — | | | | ○ | ○ | |
| **integrations** (REA, GBrain…) | | | | ○ | ● | ○ | — | | | ○ | | |
| **script** (QuickJS workflow/PTC) | ● | | ● | ● | | | | — | | ○ | | |
| **store** (JSONL, SQLite, blobs) | ● | | ○ | | | | | | — | ○ | ● | ● |
| **tui** | | | | | | | | | | — | | |
| **acp** | ○ | ○ | | | ○ | | | | ○ | | — | |
| **sdk** | ○ | | | | | | | | ○ | | | — |

## Specs × componentes (para a Fase 4)
| Spec planejada | Componentes afetados | ADRs |
|---|---|---|
| Loop do agente e sessão | core, store | 001, 002, 009, 010 |
| Provedores e modelos (gratuitos + assinaturas + fallback + data_policy) | llm | 004, 005, 101 |
| Ferramentas compatíveis com o Claude Code | tools, sandbox, script | 003, 006, 103 |
| Compatibilidade Claude Code / Totvs AGP | compat, mcp, tools | 103 |
| Plugins e marketplaces | compat, integrations | 103 |
| MCP gerenciado: REA | mcp, integrations | 102 |
| Memória: GBrain + agent-memory | mcp, integrations, compat | 102 |
| Browser/computer use e agent teams | mcp, integrations, core | 011 |
| LSP (C#, PL/SQL) | tools, integrations | 104 |
| Segurança: sandbox, permissões, credenciais, gate de rede | sandbox, compat, llm | 006, 007 |
| Superfícies: TUI, headless/stream-json, ACP, SDK | tui, sdk, acp | 012 |
| Sincronização com o upstream | (processo) | 101 |
| Subagentes, skills, hooks, workflow, goal | core, compat, script, tools | 003, 103 |
| Compactação de contexto | core, llm, store | 009 |
| Jobs, agendamentos, terminais | core, tools, sandbox, store | 006 |
| Sincronização VSCode → dsh + CatalogoSKILLs | compat, store | 103 (decisão catalog-sync) |

> [Revisão 2026-11-01] Linhas acrescentadas para as specs 5, 13, 14 e 16, que faltavam. Os nomes das specs geradas estão em `traceability/code-spec-matrix.md`.

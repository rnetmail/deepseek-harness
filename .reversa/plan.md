# Plano de Exploração — deepseek-harness

**Objetivo final:** especificações executáveis (SDD) do deepseek-harness que sirvam de base para
reconstruir um **harness em Rust** (binário único), com provedores de modelos plugáveis
(gratuitos + assinaturas), paridade de capacidades com o Claude Code e reaproveitamento
automático do conteúdo de `D:\Projetos\Totvs AGP\VSCode` e incorporação das funcionalidades do **REA** ([morluto/rea](https://github.com/morluto/rea)) e de repositórios da organização **CoreOS** (ai-helpers, ferramentas Rust) e da memória **GBrain** ([garrytan/gbrain](https://github.com/garrytan/gbrain)).

**Escopo da análise:** monorepo TypeScript (pnpm), ~5.200 arquivos `.ts` em `apps/`, `packages/`,
`native/`, `vendor/` e `python/`; 17 mil commits. Ficam de fora: `node_modules`, `website`,
`benchmarks`, `snapshots` e os arquivos de i18n `*.zh.md`.

## Fase 1 — Reconhecimento
- [x] ✅ **Scout** — Inventário, dependências, entry points (`apps/cli`, `apps/desktop`, `apps/web`) e mapa de módulos

## Fase 2 — Escavação
- Prioridade A — núcleo do agente:
  - [x] ✅ **Archaeologist** — Análise do módulo `llm`
  - [x] ✅ **Archaeologist** — Análise do módulo `core`
  - [x] ✅ **Archaeologist** — Análise do módulo `session`
  - [x] ✅ **Archaeologist** — Análise do módulo `context`
  - [x] ✅ **Archaeologist** — Análise do módulo `compaction`
  - [x] ✅ **Archaeologist** — Análise do módulo `skill`
  - [x] ✅ **Archaeologist** — Análise do módulo `subagent`
  - [x] ✅ **Archaeologist** — Análise do módulo `hooks`
  - [x] ✅ **Archaeologist** — Análise do módulo `mcp`
  - [x] ✅ **Archaeologist** — Análise do módulo `sandbox + guard`
  - [x] ✅ **Archaeologist** — Análise do módulo `interaction`
  - [x] ✅ **Archaeologist** — Análise do módulo `shell + subprocess`
  - [x] ✅ **Archaeologist** — Análise do módulo `fs`
  - [x] ✅ **Archaeologist** — Análise do módulo `plan + todo`
  - [x] ✅ **Archaeologist** — Análise do módulo `workflow + goal`
  - [x] ✅ **Archaeologist** — Análise do módulo `settings + credentials`
  - [x] ✅ **Archaeologist** — Análise do módulo `preset`
- Prioridade B — suporte:
  - [x] ✅ **Archaeologist** — Análise do módulo `extensions + bundle + boot (composição cordis)`
  - [x] ✅ **Archaeologist** — Análise do módulo `acp`
  - [x] ✅ **Archaeologist** — Análise do módulo `jobs + schedule`
  - [x] ✅ **Archaeologist** — Análise do módulo `web (fetch/search)`
  - [x] ✅ **Archaeologist** — Análise do módulo `lsp + terminal`
  - [x] ✅ **Archaeologist** — Análise do módulo `spill + storage + session-query`
  - [x] ✅ **Archaeologist** — Análise do módulo `attachment + ptc-runtime`
  - [x] ✅ **Archaeologist** — Análise do módulo `typert + api + sdk`
  - [x] ✅ **Archaeologist** — Análise do módulo `vendor/cordis + util`
  - [x] ✅ **Archaeologist** — Análise do módulo `apps/cli`
- Fonte externa:
  - [x] ✅ **Archaeologist** — Análise externa `morluto/rea` (catálogo de ferramentas MCP, CLI, skill, formato .claude-plugin, requisitos de runtime)
  - [x] ✅ **Archaeologist** — Análise externa `coreos/ai-helpers` (formato marketplace/plugin Claude Code, commands, skills, hooks)
  - [x] ✅ **Archaeologist** — Análise externa das ferramentas Rust do CoreOS (crates, padrões de CLI, cargo-vendor-filterer)
  - [x] ✅ **Archaeologist** — Análise externa `garrytan/gbrain` (BrainEngine, verbos MCP, skills de memória, hooks, config em 7 camadas)
- Prioridade C — só se pedido: `client`, `apps/web`, `apps/desktop`, `host`, `experimental`, `ssh`, `webhook` e demais
- [x] ✅ **Data Master** — Formatos persistidos: sessões, settings, credenciais, storage

## Fase 3 — Interpretação
- [x] ✅ **Detective** — Regras de negócio implícitas, ADRs retroativos via Git, máquinas de estado (loop do agente, aprovações, jobs)
- [x] ✅ **Architect** — C4, mapa de integrações (provedores de LLM, MCP, LSP, ACP) e Spec Impact Matrix

## Fase 4 — Geração
- [x] ✅ **Writer** — Specs SDD por capacidade, com rastreabilidade para o código
- [x] ✅ **Writer (extra)** — Spec de suporte a plugins/marketplace Claude Code (ai-helpers, REA)
- [x] ✅ **Writer (extra)** — Spec de memória persistente via GBrain (MCP gerenciado + skills + instruções brain-first)
- [x] ✅ **Writer (extra)** — Spec de integração do REA no harness Rust (MCP gerenciado + skill + mapeamento de capacidades)
- [x] ✅ **Writer (extra)** — Spec de provedores: Gemini Pro (API nativa + OAuth + Gemini CLI via ACP), gratuitos, fallback, data_policy/allowed_providers e gate de rede corporativa
- [x] ✅ **Writer (extra)** — Spec de browser use (Playwright MCP, Chrome DevTools MCP), computer use e agent teams (obrigatórios)
- [x] ✅ **Writer (extra)** — Spec de LSP (csharp-ls com detecção do .NET; PL/SQL via índice tree-sitter)
- [x] ✅ **Writer (extra)** — Processo upstream-sync (porte de melhorias/correções do deepseek-harness, sem envio de dados)
- [x] ✅ **Writer (extra)** — Spec de sincronização automática VSCode → harness (CatalogoSKILLs gerado, watcher, manifesto, data gate)
- [x] ✅ **Writer (extra)** — Mapeamento de compatibilidade com o formato Claude Code usado em `D:\Projetos\Totvs AGP\VSCode` (CLAUDE.md, agents, skills, hooks, MCP)

## Fase 5 — Revisão
- [x] ✅ **Reviewer** — Revisão cruzada, reclassificação de confiança e perguntas de validação

## Depois da análise (opcional)
- [~] **Reconstructor** (plano criado: 25 tarefas) — Plano bottom-up de reimplementação em Rust, executado tarefa a tarefa

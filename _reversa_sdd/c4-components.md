# C4 — Nível 3: Componentes (containers críticos)

## 1. Agent plane (`core`) — hoje e alvo 🟢
```mermaid
flowchart TB
  IN[Inbox\nnextTurn / nextStep] --> PRE[agent/pre-step\nclaim + compaction pressure]
  PRE --> PROMPT[System prompt\nseções ordenadas: persona, CLAUDE.md, skills, plan, memória]
  PROMPT --> REQ[agent/request\nprovider+model, deriveMessages]
  REQ --> LLM[ctx.llm.stream\nStreamChunks → assistant/message]
  LLM --> DISP[Dispatcher de tools\nmaxParallel=10]
  DISP --> PIPE[Pipeline\npre-execute → guards → hooks → approval → execute → post-execute]
  PIPE --> RES[tool/result + spill/pruner]
  RES --> PRE
  LLM --> END[turn/end + flush]
  SESS[(Session log\nappend síncrono)] --- REQ & LLM & RES & END
  PROJ[Projeções\ntodo, goal, schedule, stats] --- SESS
```

## 2. LLM runtime 🟢 → 🟡
| Componente | Hoje (dsh) | Alvo (Rust) |
|---|---|---|
| Registro de adapters | `registerAdapter(providers)`, um dono por provider | Igual (`HashMap<ProviderId, Arc<dyn LlmAdapter>>`) |
| Protocolos | openai-completions, openai-responses, anthropic-messages | + **gemini generateContent** (nativo) |
| Catálogo | 39 provedores / 113 modelos de custo zero (pi-ai) | Catálogo próprio atualizável (`models.dev`/OpenRouter API) + manual |
| Auth | API key (YAML) + OAuth (Claude, ChatGPT, Copilot, xAI, Kimi, OpenRouter) | Cofre do SO + OAuth **Google (Gemini Pro)** + os demais |
| Retry | 5×, 500 ms–10 s, jitter de 10% | Igual + **cadeia de fallback** por workspace |
| Política de dados | — | `data_policy` por provedor + `allowed_providers` por workspace |
| Medição | chars/4 | tokenizer por família (tiktoken-rs) quando disponível |

## 3. Compatibilidade Claude Code (`harness-claude-compat`) 🟡
```mermaid
flowchart LR
  subgraph Fontes
    A[CLAUDE.md / CLAUDE.local.md]
    B[.claude/agents/*.md\nname, description, tools, model, memory]
    C[.claude/skills + ~/.claude/skills]
    D[.claude/commands/*.md]
    E[.claude/settings*.json\npermissions, hooks, env]
    F[.mcp.json + ~/.claude.json mcpServers]
    G[~/.claude/plugins\nmarketplaces]
    H[.claude/agent-memory/<agente>]
  end
  A --> P[Seção de prompt]
  B --> S[Catálogo subagent_type\n+ mapa de modelos opus/sonnet/haiku]
  C --> K[Catálogo de skills]
  D --> X[Slash commands]
  E --> R[Motor de permissões\nallow/deny/ask + hooks]
  F --> M[Registro MCP]
  G --> B & C & D & E & F
  H --> MEM[Memória por agente ↔ GBrain]
```

## 4. Integrações gerenciadas (`harness-integrations`) 🟡
| Integração | Pré-requisito | Instalação/atualização | Registro |
|---|---|---|---|
| REA | Node ≥ 22.23 | `npm i -g rea-agents` / `rea update` | `rea` → `node …/scripts/rea.mjs mcp` |
| GBrain | Bun ≥ 1.4.4 | `bun install -g github:garrytan/gbrain#latest-stable` | `gbrain serve --surface starter` |
| Playwright MCP | Node | npm `@playwright/mcp` | stdio |
| Chrome DevTools MCP | Node + Chrome | npm `chrome-devtools-mcp` | stdio |
| Computer use | Cua Driver | conforme upstream | MCP |
| csharp-ls | .NET SDK (detectado) | `dotnet tool install/update -g csharp-ls` | LSP |
| Gemini CLI (opcional) | Node | npm `@google/gemini-cli` | agente ACP |

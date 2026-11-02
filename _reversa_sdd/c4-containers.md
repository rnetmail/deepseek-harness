# C4 — Nível 2: Containers

## 1. Como é hoje (dsh) 🟢
```mermaid
flowchart LR
  subgraph Host["Processo Node (dsh --profile …)"]
    LAUNCH[Launcher apps/cli\nperfis, patches, sinais]
    LOADER[Loader cordis\nDI + fibers + HMR]
    CORE[Agent plane\nagent-loop, tools, prompt, compaction]
    LLMR[LLM runtime\npi-ai + deepseek adapters + retry]
    EXEC[Execução\nsubprocess, sandbox, bash/pwsh, PTC Node]
    EXT[Extensibilidade\nskills, subagents, hooks*, mcp*, workflow]
    PERS[Persistência\nsession JSONL zstd, storage, spill, attachments]
    SURF[Superfícies\nwebserver+gateway | ACP | SDK JSON-RPC | headless]
  end
  WEBUI[Cliente web React\nslots/UI cordis]
  FILES[(~/.dsh\nsessions, settings.yaml,\n.credentials.yaml, storages)]
  LAUNCH --> LOADER --> CORE
  CORE --> LLMR
  CORE --> EXEC
  CORE --> EXT
  CORE --> PERS --> FILES
  SURF --> CORE
  WEBUI <-- "HTTP + WS (token)" --> SURF
```
`*` hooks e mcp-client existem, mas **não** vêm montados no bundle `base`.

## 2. Alvo (harness Rust): workspace Cargo 🟡
```mermaid
flowchart LR
  subgraph BIN["dsh (binário único Rust, tokio)"]
    CLI[harness-cli\nclap: tui, -p, acp, serve, models, login, mcp, plugin, doctor]
    TUI[harness-tui\nratatui]
    ACP[harness-acp\nagent-client-protocol server+client]
    SDK[harness-sdk\nJSON-RPC stdio + stream-json]
    CORE[harness-core\nloop ReAct, inbox, turnos, cancelamento, compaction]
    LLM[harness-llm\nadapters: openai-chat, openai-responses, anthropic, gemini\nOAuth, catálogo, fallback]
    TOOLS[harness-tools\nRead/Write/Edit/Glob/Grep/Bash/PowerShell/WebFetch/WebSearch\nTodoWrite/Task/Skill/Workflow/LSP/Terminal/Jobs/Schedule]
    SBX[harness-sandbox\nJob Objects / AppContainer / Landlock+seccomp]
    MCPC[harness-mcp\nrmcp client stdio/http, progress, cancel]
    COMPAT[harness-claude-compat\nCLAUDE.md, .claude/agents, skills, commands,\nsettings permissions/hooks, .mcp.json, plugins]
    MGR[harness-integrations\ninstalador/atualizador REA, GBrain, Playwright,\nChrome DevTools, Cua, csharp-ls]
    JS[harness-script\nrquickjs: workflow + PTC]
    STORE[harness-store\nJSONL zstd + rusqlite WAL FTS5 + blobs]
  end
  KR[(Cofre do SO)]
  HOME[(~/.dsh)]
  CLI --> TUI & ACP & SDK --> CORE
  CORE --> LLM & TOOLS & MCPC & STORE & JS
  TOOLS --> SBX
  COMPAT --> CORE
  MGR --> MCPC
  LLM --> KR
  STORE --> HOME
```

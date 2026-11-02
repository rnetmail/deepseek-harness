# ERD completo (domínio lógico)

Fontes: `database/` (Data Master) + data-dictionary + inventário do Totvs AGP. 🟢 = dsh confirmado, 🟡 = entidades novas do alvo Rust.

```mermaid
erDiagram
  WORKSPACE ||--o{ SESSION : "cwd"
  WORKSPACE ||--o| WORKSPACE_POLICY : "allowed_providers, data_policy, gate de rede (🟡)"
  SESSION ||--o{ SESSION_EVENT : "append-only"
  SESSION ||--o{ SESSION : "fork / subagente (parentSession)"
  SESSION }o--|| AGENT_PRESET : "preset"
  SESSION ||--o| GOAL : "≤1 atual"
  SESSION ||--o{ TODO_ITEM : "projeção"
  SESSION ||--o{ SCHEDULE_RECORD : "eventos schedule/change"
  SESSION ||--o{ JOB : "dono (runtime)"
  SESSION ||--o{ TERMINAL_SESSION : "dono (runtime)"
  SESSION_EVENT }o--o{ ATTACHMENT : "ref por hash"
  SESSION_EVENT }o--o| SPILL_FILE : "locator"
  SESSION_EVENT }o--o| COMPACTION_CHECKPOINT : "substitui prefixo"
  SESSION ||--o| SEARCH_INDEX_ENTRY : "FTS derivado"
  PROVIDER ||--o{ MODEL : "catálogo"
  PROVIDER ||--o| CREDENTIAL : "credentialRef / OAuth record"
  SESSION_EVENT }o--|| MODEL : "request header provider+model"
  AGENT_DEFINITION }o--o| MODEL_ALIAS : "model: opus/sonnet/haiku (🟡)"
  MODEL_ALIAS }o--|| MODEL : "mapeado para"
  AGENT_DEFINITION ||--o| AGENT_MEMORY : "memory: project (🟡)"
  WORKSPACE ||--o{ AGENT_DEFINITION : ".claude/agents"
  WORKSPACE ||--o{ SKILL : ".claude/skills"
  WORKSPACE ||--o{ PERMISSION_RULE : "settings.json allow/deny/ask"
  WORKSPACE ||--o{ HOOK : "settings.json hooks"
  WORKSPACE ||--o{ MCP_SERVER : ".mcp.json / ~/.claude.json"
  PLUGIN ||--o{ SKILL : contém
  PLUGIN ||--o{ AGENT_DEFINITION : contém
  PLUGIN ||--o{ HOOK : contém
  PLUGIN ||--o{ MCP_SERVER : contém
  MARKETPLACE ||--o{ PLUGIN : lista
  MCP_SERVER ||--o{ MCP_TOOL : "tools/list → mcp__srv__tool"
  MANAGED_INTEGRATION ||--|| MCP_SERVER : "REA, GBrain, Playwright… (🟡)"

  SESSION { string id PK; int version; string cwd; string parentSession; bool isSeeded }
  SESSION_EVENT { int seq PK; string type; json data; int time }
  PROVIDER { string id PK; string protocol; string data_policy }
  MODEL { string id PK; int contextWindow; bool free; bool reasoning; bool images }
  CREDENTIAL { string ref PK; string source }
  AGENT_DEFINITION { string name PK; string description; string tools; string model; string memory }
  SKILL { string name PK; string description; bool userInvocable }
  PERMISSION_RULE { string pattern PK; string effect }
  MCP_SERVER { string name PK; string transport; string commandOrUrl }
  MANAGED_INTEGRATION { string name PK; string runtime; string minVersion; string installCmd }
  WORKSPACE_POLICY { string workspace PK; string allowedProviders; string networkGate }
```

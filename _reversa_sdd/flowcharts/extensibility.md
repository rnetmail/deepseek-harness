# Fluxogramas — `skill`, `subagent`, `hooks`, `mcp`

## 1. Skills: descoberta e uso 🟢

```mermaid
flowchart TD
  A[início de sessão] --> B[providers: .dsh/skills → .agents/skills → custom → ~/.dsh/skills → ~/.agents/skills → bundled]
  B --> C[parse frontmatter; menor rank vence por nome]
  C --> D{há skills model-invocable e tool skill visível?}
  D -- sim --> E[mensagem user-role: catálogo nome + descrição ≤500]
  E --> F[modelo chama skill name]
  F --> G[relê arquivo → skill_content]
  H[usuário digita /nome] --> I{user-invocable?} -- sim --> G
  J[watch: arquivo novo/removido] --> K[catálogo substituto completo]
```

## 2. Delegação a subagente 🟢

```mermaid
flowchart TD
  A[modelo chama subagent prompt, run_in_background?] --> B{profundidade < maxDepth?}
  B -- não --> X[erro]
  B -- sim --> C{capacidades pedidas suportadas pelo provider?}
  C -- não --> X2[falha no início]
  C -- sim --> D{backgroundMode / run_in_background}
  D -- foreground --> E[filho roda até finalizar] --> F[texto final como tool result]
  D -- background one-shot --> G[job id → job_output / job_kill]
  D -- continuable --> H[childId durável → aviso de settlement]
  H --> I[send_message: steer se ocupado, novo turno se ocioso]
  H --> J[interrupt_agent: só turno atual]
```

## 3. Hook PreToolUse (bridge Claude Code) 🟢

```mermaid
flowchart TD
  A[tools/pre-execute] --> B[grupos PreToolUse cujo matcher casa com o nome da ferramenta]
  B --> C[executa cada command em sequência, cwd = projeto, stdin = JSON do evento]
  C --> D{exit code}
  D -- 2 --> E[bloqueia; stderr = motivo]
  D -- 0 --> F{stdout JSON}
  F -- decision ask --> G[aprovação do usuário]
  F -- contexto --> H[anexa contexto no próximo request]
  F -- nada --> I[permite]
  D -- outro --> J[log de falha; permite]
```

## 4. Ciclo de vida do servidor MCP 🟢

```mermaid
flowchart LR
  A[plugin mcp-client] --> B{transport}
  B -- stdio --> C[probe de negociação + processo servidor]
  B -- streamable-http --> D[conexão HTTP]
  C --> E[initialize + tools/list]
  D --> E
  E --> F[registra mcp__server__tool no ctx.tools]
  E -. falha .-> G[log; harness segue sem as ferramentas]
  F --> H{conexão perdida}
  H --> I[reconnect 500ms ×2 até 30s, máx 10 tentativas]
```

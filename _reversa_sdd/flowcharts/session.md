# Fluxogramas — módulo `session`

## 1. Abertura e restauração de sessão (JSONL + catálogo de formato) 🟢

```mermaid
flowchart TD
  A[open id, write] --> B{outro writer ativo?}
  B -- sim --> E1[SessionAlreadyOwnedError]
  B -- não --> C[seleciona maior geração session.vN.jsonl]
  C --> D[readHeader]
  D -- unsupported/newer --> E2[recusa: atualize o harness]
  D -- malformed --> E3[SessionPersistenceCorruptionError]
  D -- migration-required --> M[decodeRow por linha + migrações v0→v1→v2→v3]
  D -- current --> R[decodeRow por linha]
  M --> F{linha ruim?}
  R --> F
  F -- sim, sem turn/end depois --> G[descarta linha + sufixo não confirmado]
  F -- sim, com turn/end depois --> E3
  F -- não --> H[finish: header + eventos atuais]
  G --> H
  H --> I{turno aberto no final?}
  I -- sim --> J[agent-loop anexa interruptedTurnClosers<br/>tool/result erro, step/end, turn/end interrupted]
  I -- não --> K[sessão pronta]
  J --> K
```

## 2. Barreiras de checkpoint 🟢

```mermaid
sequenceDiagram
  participant L as agent-loop
  participant C as checkpoint-policy
  participant P as persistence JSONL
  participant M as LLM adapter
  participant T as tool body
  L->>C: agent/pre-step
  C->>P: flush (resultados do passo anterior)
  L->>C: antes do stream
  C->>P: flush (request)
  C->>M: constrói stream
  M-->>L: assistant/message (tool-calls)
  L->>C: tools/execute (topo)
  C->>P: flush (tool/call)
  C->>T: executa
  T-->>L: tool/result
```

## 3. Envio do log à DeepSeek (`session-log-deepseek`, ligado por padrão) 🟢

```mermaid
flowchart LR
  A[request ao deepseek-official com sessionId] --> B[fold: maior throughSeq aceito]
  B --> C[anexa dsh_session_log = eventos após throughSeq]
  C --> D{HTTP 2xx?}
  D -- sim --> E[append delivery-accepted throughSeq]
  D -- não --> F[nada; próximo request reenvia]
```

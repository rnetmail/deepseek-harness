# ERD — dados persistidos do deepseek-harness

## 1. Visão lógica geral 🟢/🟡

```mermaid
erDiagram
  SESSION ||--o{ SESSION_EVENT : "contém (append-only)"
  SESSION ||--o{ SESSION : "pai de (fork/subagente)"
  SESSION }o--|| WORKSPACE : "pertence (cwd)"
  SESSION_EVENT }o--o{ ATTACHMENT : "referencia por id"
  SESSION_EVENT }o--o{ SPILL_FILE : "referencia por locator"
  SESSION ||--o{ PROJECTION_CACHE : "projeções derivadas"
  SESSION ||--o| INDEXED_SESSION : "indexada em (derivado)"
  INDEXED_SESSION ||--o{ FTS_DOC : "documentos"
  PROFILE ||--o{ BUNDLE : "empilha"
  PROFILE ||--o| PATCH : "cordis.patch.yml"
  SETTINGS ||--o{ NAMESPACE_SECTION : "por namespace"
  CREDENTIALS ||--o{ CREDENTIAL_RECORD : "segredos/OAuth"

  SESSION { string id PK; int version; int createdAt; string cwd; string parentSession; bool isSeeded }
  SESSION_EVENT { int seq PK; string type; json data; int time; string surface }
  ATTACHMENT { string hash PK; string displayName; string mime; int bytes }
  SPILL_FILE { string locator PK; int bytes }
  INDEXED_SESSION { string id PK; string revision; int generation; int delegation_depth; string agent_preset }
  FTS_DOC { string session_id FK; int seq; string type; text text }
  CREDENTIAL_RECORD { string key PK; string value }
```

## 2. Storage SQLite (físico) 🟢

```mermaid
erDiagram
  units ||--o| unit_globals : "unit -> name"
  units ||--o{ unit_table : "uma tabela por (unit, table)"
  units { TEXT name PK; INTEGER version }
  unit_globals { TEXT unit PK; TEXT value }
  unit_table { TEXT key PK; TEXT value }
```

## 3. Índice FTS (físico) 🟢

```mermaid
erDiagram
  search_state { INTEGER singleton PK; INTEGER global_generation }
  persisted_sessions ||--o{ persisted_docs : "session_id"
  live_sessions ||--o{ live_docs : "session_id (TEMP)"
  persisted_sessions { TEXT id PK; INTEGER version; INTEGER created_at; TEXT cwd; TEXT parent_session; INTEGER seed_length; INTEGER delegation_depth; TEXT agent_preset; TEXT revision; INTEGER generation }
  persisted_docs { TEXT text; TEXT session_id; INTEGER seq; TEXT type; INTEGER time; TEXT surface; INTEGER codepoint_length }
```

# Relacionamentos — dados persistidos

| Origem | Destino | Cardinalidade | Mecanismo | Confiança |
|---|---|---|---|---|
| Sessão | Evento | 1:N | Linhas do JSONL depois do header (append-only, `seq` crescente) | 🟢 |
| Sessão | Sessão (filha) | 1:N | `parentSession` no header. Fork herda `inheritedEventCount` eventos (`isSeeded`) | 🟢 |
| Sessão | Workspace | N:1 | `cwd` (diretório canônico) + `storages/workspace.json` | 🟢/🟡 |
| Evento | Attachment | N:M | Id/hash no payload. Eventos **não** carregam caminhos nem base64 | 🟢 |
| Evento | Spill | N:1 | Locator no resultado da ferramenta | 🟢 |
| Sessão | Projeções (todo, goal, schedule, stats…) | 1:N | Fold dos eventos. Cache em `storages/session_projcache` | 🟢 |
| Sessão | `persisted_sessions` | 1:0..1 | Índice derivado, invalidado por `revision`/`generation` | 🟢 |
| `persisted_sessions` | `persisted_docs` | 1:N | `session_id` UNINDEXED (sem FK física: FTS5 não suporta) | 🟢 |
| `units` | `unit_globals` | 1:0..1 | FK `unit → units(name)` | 🟢 |
| `units` | tabelas `<unit>__<table>` | 1:N | Convenção de nome validada por regex | 🟢 |
| Perfil | Bundles | 1:N ordenado | `package.json → dsh.profile.bundles` | 🟢 |
| Credencial | Provedor LLM | N:1 | `credentialRef(NOME)` ou record `<scope>/<id>` | 🟢 |

**Polimorfismo:** `SessionEvent.type` + `data` formam uma união discriminada, e cada plugin define seus tipos (exemplo: `schedule/change` v1).

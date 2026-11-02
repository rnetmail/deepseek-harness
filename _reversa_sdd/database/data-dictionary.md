# Dicionário de dados persistidos — deepseek-harness

O dsh **não usa banco relacional de domínio**. O estado durável fica dividido em seis meios:

| # | Meio | Local (padrão) | Formato | Autoridade |
|---|---|---|---|---|
| 1 | Log de sessão | `~/.dsh/sessions/--<cwd>--/<id>/session.v3.jsonl.zstd` | JSONL + zstd, append-only | **Fonte da verdade** da conversa |
| 2 | Settings | `~/.dsh/settings.yaml` | YAML (mapa por namespace) | Preferências do usuário |
| 3 | Credenciais | `~/.dsh/.credentials.yaml` | YAML 0600 | Segredos e records OAuth |
| 4 | Storage de domínio | `~/.dsh/storages/` (`workspace.json`, `session_projcache/`) | JSON (`single`/`per-record`) ou SQLite | Estado de app fora do log |
| 5 | Índice FTS | SQLite (`:memory:` no base) | SQLite STRICT + FTS5 | **Derivado** (reconstruível) |
| 6 | Blobs | `~/.dsh/` (attachments), diretório temporário (spill) | Arquivos endereçados por conteúdo / privados | Anexos e saídas grandes |

Além desses: perfis (`~/.dsh/profiles/<nome>/{package.json, cordis.patch.yml}`), `~/.dsh/cordis.patch.yml`, `~/.dsh/.env`, `~/.dsh/.anonymous-user-id`, presets (`~/.dsh/.agent-presets/`) e skills (`~/.dsh/skills/`).

---

## 1. Log de sessão (JSONL v3) 🟢
| Linha | Campo | Tipo | Obrig. | Observação |
|---|---|---|---|---|
| 1 (header) | version | int (=3) | sim | Igual ao sufixo do arquivo, `SESSION_FORMAT_VERSION` |
| | id | string | sim | Escapado no caminho |
| | createdAt | epoch ms | sim | |
| | cwd | string | não | Escopo de busca e autorização |
| | parentSession | string | não | Forks e subagentes |
| | isSeeded | bool | sim | Fork: eventos herdados |
| | origin | object | não | |
| 2..n | type | string | sim | Exemplos: `turn/start`, `message`, `tool/call`, `tool/result`, `compaction/checkpoint`, `schedule/change`, `approval/*`… |
| | data | object | sim | Payload por tipo |
| | seq / time / surface | metadados | — | Ordem, timestamp, superfície |

Leitura do header: `current | migration-required | unsupported | malformed`. Erros: SessionAlreadyOwned / AlreadyExists / ReadOnly / HandleClosed / OwnershipLost / PersistenceCorruption.

## 2. `settings.yaml` 🟢
Mapa `<namespace>: <objeto>`. O namespace usa minúsculas, dígitos e hífen. Exemplos: `llm-pi-ai.providers`, `agent-default-model { provider, model, reasoningEffort }`, `agent-loop { maxParallelToolCalls }`. A escrita é por diff de folhas YAML (preserva comentários e anchors) com lock de 2 s e substituição atômica 0600.

## 3. `.credentials.yaml` 🟢
| Chave | Valor |
|---|---|
| `<NOME_POSIX>` | segredo (exemplo: `DEEPSEEK_API_KEY`) |
| `<scope>/<id>` | record de fluxo (exemplo: `llm-pi-ai/openai-codex`: tokens OAuth) |

Resolução: store primeiro, depois a variável de ambiente. 🔴 Legível pelas ferramentas do agente, porque roda como o mesmo usuário do SO.

## 4. Storage SQLite (`storage-sqlite`) 🟢 DDL
| Tabela | Colunas | Restrições |
|---|---|---|
| `units` | name TEXT, version INTEGER NOT NULL | PK(name), STRICT |
| `unit_globals` | unit TEXT, value TEXT NOT NULL | PK(unit), FK unit → units(name) |
| `"<unit>__<table>"` (uma por tabela de domínio) | key TEXT, value TEXT (JSON) NOT NULL | PK(key), STRICT |

`PRAGMA user_version` = versão do schema, gravada **por último** (só com o layout completo). Versão do unit divergente do descritor → erro de incompatibilidade.

## 5. Índice FTS (`session-query-sqlite`) 🟢 DDL
| Tabela | Colunas | Restrições |
|---|---|---|
| `search_state` | singleton INTEGER, global_generation INTEGER | PK, CHECK(singleton = 1), linha única |
| `persisted_sessions` | id TEXT, version INT, created_at INT, cwd TEXT, parent_session TEXT, seed_length INT, delegation_depth INT, agent_preset TEXT, revision TEXT, generation INT | PK(id), STRICT |
| `persisted_docs` (FTS5) | text; UNINDEXED: session_id, seq, type, time, surface, codepoint_length | tokenize `unicode61` |
| `temp.live_sessions` | colunas iguais + fingerprint TEXT, persisted INT CHECK IN (0,1) | TEMP |
| `temp.live_docs` (FTS5) | igual a persisted_docs | TEMP |

`PRAGMA application_id` fixo e `user_version` = versão do schema. Journal: WAL.

## 6. Blobs 🟢
- **Attachments:** objeto por hash de conteúdo, com metadados (nome de exibição, mime, tamanho). A imagem normalizada fica em cache por rota. Nunca apagados automaticamente.
- **Spill:** `<root>/<sessão>/<nome aleatório>`, 0700. Limpeza no arranque após 30 dias.

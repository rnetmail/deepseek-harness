# Dicionário de Dados — deepseek-harness

> Gerado pelo **Reversa Archaeologist** (nível `completo`). Entidades extraídas dos tipos TypeScript (`interface`/`type`) e dos schemas zod/schemastery.
> Formatos persistidos (JSONL de sessão, SQLite, YAML) serão aprofundados pelo **Data Master**.
> Escala: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA

---

## Módulo `llm`

### `GenerateOptions` 🟢 — `packages/llm/llm/src/types.ts:453`
| Campo | Tipo | Obrig. | Padrão / observação |
|---|---|---|---|
| provider | string | sim | nome da rota (`deepseek-official`, `openai`, `openrouter`…) |
| model | string | sim | id do modelo |
| reasoningEffort | ReasoningEffortId | não | validado contra o modelo antes do I/O |
| messages | Message[] | sim | deep-frozen quando vem do loop |
| system | string | não | prompt de sistema |
| tools | ToolSchema[] | não | `{ name, description, parameters (JSON Schema) }` |
| temperature | number | não | |
| maxTokens | number | não | default do adapter é materializado |
| stop | string[] | não | rejeitado pelo pi-ai (`UNSUPPORTED_OPTION`) |
| signal | AbortSignal | não | cancelamento → `finish aborted` |
| sessionId | SessionId | não | |
| purpose | `'compaction' \| 'session-title'` | não | chamadas auxiliares |

### `ContentBlock` (união discriminada por `type`) 🟢 — `types.ts:61-137`
| type | Campos |
|---|---|
| `text` | text: string |
| `reasoning` | text: string |
| `image` | attachment: ImageAttachmentRef; offloaded?: true |
| `file` | attachment: FileAttachmentRef |
| `tool-call` | id: ToolCallId; name: string; arguments: string (JSON cru) |
| `tool-result` | toolCallId: ToolCallId; content: ContentBlock[]; isError?: boolean |

### `Message` 🟢 — `message.ts:131-180`
| Tipo | Papel |
|---|---|
| `UserMessage` | entrada do usuário |
| `AssistantMessage` | resposta do modelo, com metadados do provedor (`AssistantProviderMetadata`) |
| `SystemMessage` | `createSystemMessage(text, plugin)`: prompt de sistema atribuído a um plugin |
| `ToolResultMessage` | resultado de ferramenta |

Todas são imutáveis (`freezeMessage`). `MessageSource` indica a origem (modelo, ferramenta ou contexto). `CONTEXT_SUMMARY_MAX_CHARS = 120`.

### `StreamChunk` 🟢 — `types.ts:424`
| type | Campos |
|---|---|
| `block-start` | index, blockType |
| `text-delta` / `reasoning-delta` | index, text |
| `tool-call-delta` | index, id, name?, argumentsDelta |
| `block-end` | index, block: ContentBlock |
| `usage` | usage: TokenUsage |
| `finish` | reason: FinishReason; replayState?: ReplayEnvelope `{ response, blocks? }` |

### `FinishReason` 🟢
`stop` · `tool-calls` · `max-tokens` · `aborted { failure }` · `error { failure }`

### `TokenUsage` 🟢
| Campo | Tipo | Obrig. |
|---|---|---|
| inputTokens | number | sim |
| outputTokens | number | sim |
| totalTokens | number | não |
| cacheReadTokens | number | não |
| cacheWriteTokens | number | não |
| reasoningTokens | number | não |

### `LlmFailure` 🟢
| Campo | Tipo | Obrig. |
|---|---|---|
| message | string | sim |
| code | string (ver códigos em code-analysis §llm) | sim |
| status | number (HTTP) | não |
| providerRetryAfterMs | number | não |
| requestId | ProviderRequestId | não |
| offloadImages | number | não |

### `LlmResolvedModelInfo` 🟢 (estende `LlmModelInfo`)
| Campo | Tipo | Obrig. |
|---|---|---|
| provider, id, name | string | sim |
| description | string | não |
| inputModalities | (`'text' \| 'image'`)[] | não |
| context.contextWindow | number | não |
| defaultMaxTokens | number | não |
| reasoning | `{ efforts: {id,name,description?}[], defaultEffort? }` | não |
| systemPromptUpdate | `'in-history'` | não |

### `LlmConfigurableProvider` 🟢
provider, displayName, settingsNs, settingsPath: string[], declared?: boolean, error?: string (diagnóstico de configuração).

### `LlmDiscoveredModel` 🟢
id (obrig.), name?, contextWindow?, maxTokens?

### `LlmImageRequestBudget` 🟢
representation: `'raw' | 'base64'`; maxBytes?; maxImages?; byteQuantum?; countQuantum?

### Perfil de provedor `llm-pi-ai` (`providers.<rota>`) 🟢 — `llm-pi-ai/src/config.ts:320+`
| Campo | Tipo | Padrão |
|---|---|---|
| apiKeyEnv | string (referência de credencial) | ausente → descoberta ambiente do pi-ai |
| displayName | string | nome da rota |
| api | `openai-completions \| openai-responses \| anthropic-messages` | protocolo do catálogo |
| baseURL | string | endpoint do catálogo |
| models | ModelProfile[] (id, name, contextWindow, maxTokens, reasoningEfforts, input…) | catálogo instalado |
| modelOverrides | dict<ModelOverride> | — |
| compat | CompatProfile (papel do system, campo de max tokens, formato de thinking, `thinkingTokenBudgetField`, `vllmPriority`, `supportsMaxOutputTokens`) | detecção do catálogo |
| headers | dict<string> | — |
| reasoning | nível de thinking | — |
| thinkingBudgets | mapa nível → tokens | — |
| cacheRetention | `none \| short \| long` | — |
| transport | `sse \| websocket \| websocket-cached \| auto` | — |
| defaultContextWindow | int | 262.144 |
| defaultMaxTokens | int | 32.768 |
| defaultInput | modalidades | text |
| requestImagePixelBudget | int | 4.194.304 |
| requestImageMaxBytes | int | 1 MiB |
| maxRequestImageBytes | int | 20 MiB |
| streamIdleTimeoutMs | int | 300.000 |
| retryPolicy | RetryPolicyConfig | normal, 5 |

### Config `llm-deepseek` 🟢
protocol (`messages`), apiKeyEnv (`DEEPSEEK_API_KEY`), baseURL, thinking (`enabled`), reasoningEffort (`high`; `off|low|high|max`), maxTokens (256.000), defaultContextWindow (1.000.000), models, streamIdleTimeoutMs (300.000), maxRequestFilesBytes (128 MiB), maxInlineRequestImageBytes (20 MiB), maxImagesPerRequest (600), imageOffloadByteQuantum (64 MiB), inlineImageOffloadByteQuantum (10 MiB), imageOffloadCountQuantum (20), filesApiTimeoutMs (60.000), fileExpiresAfterSeconds (604.800), fileRefreshMarginSeconds (3.600), fileQuotaCleanupBatch (100), retryPolicy.

### `RetryPolicyConfig` 🟢 — `retry-policy.ts`
| Campo | Tipo | Padrão |
|---|---|---|
| mode | `normal \| always` | normal |
| maxRetries (normal) | number | 5 |
| codes (normal) | string[] | EMPTY_RESPONSE, RATE_LIMIT, SERVER, TIMEOUT, TRANSPORT |
| backoff.initialDelayMs | number | 500 |
| backoff.maxDelayMs | number | 10.000 |
| backoff.jitterRatio | number | 0,1 |

### Eventos de sessão emitidos 🟢
`llm/retry` (retryId, provider, mode, policyKey, failure, delay) · `llm/retry-started` · `image/offload`

### Projeções do token-meter 🟢
`tokenUsage { uncachedInputTokens, outputTokens, cacheReadTokens, cacheWriteTokens }` · `contextPressure { pressureTokens?, projectedTokens?, contextWindow? }` · `contextBreakdown { systemTokens, toolsTokens, messageTokens }`

---

## Módulo `core`

### `AgentOptions` 🟢 — `core/agent/src/types.ts`
| Campo | Tipo | Obrig. | Observação |
|---|---|---|---|
| provider | string | sim* | *pode vir de `agent/request` |
| model | string | sim* | |
| reasoningEffort | ReasoningEffortId | não | validado pelo adapter |
| maxTokens | number (>0) | não | |

### `CreateAgentOptions` 🟢
sessionId, agentOptions: AgentOptions, parentAgent?: Agent, setup?: (agentCtx, agent) => void | Promise.

### Config `dsh-agent-loop` 🟢
| Campo | Tipo | Padrão |
|---|---|---|
| maxParallelToolCalls | int | 10 (`constants.ts:6`) |
| agents[].id | string | obrigatório |
| agents[].provider / model | string | — |
| agents[].reasoningEffort | string | — |
| agents[].maxTokens | int > 0 | — |
| agents[].cwd | string | — |
| agents[].sessionId | string | `${id}-session-<uuid>` |
| agents[].resumeSessionId | string | exclusivo com sessionId |

### `AgentStatus` 🟢
`'idle' | 'running'`

### `PreStepDecision` 🟢
`{ kind: 'reject' }` | `{ kind: 'enter', messages: Message[], startsRequestSeries?: boolean }`

### `Inbox` (estrutural) 🟢
Filas `nextTurn: Message[]` e `nextStep: Message[]`. Operações: append, prepend, replace, remove, clear, splice. `MessageId` é único entre as duas filas.

### `ToolDefinition` 🟢 — `core/tools/src/types.ts`
| Campo | Tipo | Obrig. |
|---|---|---|
| name | string | sim |
| description | string | sim |
| parameters | ParameterSchemaSpec (DSL) ou JsonSchemaNode | sim |
| output | `{ schema, render(args, value) → ContentBlock[] }` | sim |
| execute | `(args, exec: { signal, … }) → Promise<value>` | sim |
| finalizeContent | função | não |
| timeoutMs | number | não |
| presentCall / presentResult | funções puras (host) | não |
| exclusive / parallel-safe | flag de concorrência 🟡 | não |

### Config `dsh-tools` 🟢
mode: `native | ptc | both` (padrão `native`); maxParallelSubCalls: 10.

### Códigos de erro de ferramenta 🟢
`UNKNOWN_TOOL`, `ABORTED_BEFORE_DISPATCH`, `ABORTED`, `TOOL_TIMEOUT`, além dos códigos `ToolErrorInfo` vindos de pre-execute.

### `SessionHeader` 🟢 — `core/session/src/types.ts:93`
| Campo | Tipo | Obrig. |
|---|---|---|
| version | SESSION_FORMAT_VERSION | sim |
| id | SessionId | sim |
| createdAt | number (epoch ms) | sim |
| cwd | string | não |
| parentSession | SessionId | não |
| isSeeded | boolean | sim |
| origin | `'subagent'` | não |

### `SessionEvent` — tipos centrais 🟢 (`SessionEventMap`)
| type | Superfície? | Conteúdo principal |
|---|---|---|
| `turn/start` / `turn/end` | não | turn id; `turn/end.reason` ∈ completed, aborted{reason}, blocked, error{LlmFailure} |
| `step/start` / `step/end` | não | step id |
| `system/message` | sim | prompt renderizado (nó 0 = system) |
| `user/message` | sim | UserMessage + source (`user` \| `plugin`) |
| `assistant/message` | sim | AssistantMessage + stream compacto embutido |
| `assistant/attempt` | não | tentativa que falhou, foi repetida ou cancelada |
| `tool/call` | não | id, nome, argumentos |
| `tool/result` | sim | ToolResultMessage; `data.error` só se isError |
| `request/header` | não | snapshot canônico: call config, adapterDefaults, tools; reason ∈ initial, resume, change, series |
| `request/context` | não | provider, model, contextWindow, systemPromptUpdate |
| `session/end-seed` | não | fim do prefixo herdado (fork) |

Metadado de superfície: `surfaceOp: 'append' | { op: 'replace', startSeq, endSeq }`.
Tipos numéricos: `SessionSeq` (evento existente), `SessionLogOffset` (posição de gap), `SessionSeqCursor` (−1 = nenhum).

### `SystemPromptSection` 🟢
| Campo | Tipo | Obrig. |
|---|---|---|
| name | string | sim |
| order | number finito | sim |
| text | string \| (ctx) → string | sim |
| interpolate | boolean (padrão true) | não |
| complete | boolean | não |

### Config `dsh-system-prompt` 🟢
includeHarnessIdentity (true), includeRuntimeContext (true), personaPrefix (''), personaSuffix (''), toolOrder (string[] com um `'<unlisted-tools>'`).

---

## Módulo `session`

### Arquivo de sessão JSONL (v3) 🟢
| Linha | Conteúdo |
|---|---|
| 1 | Header físico: `version` (= sufixo do arquivo), `id`, `createdAt`, `cwd?`, `parentSession?`, `isSeeded`, `origin?` |
| 2..n | Um `SessionEvent` por linha (`type`, `data`, metadados de superfície) |

Caminho: `<root>/--<cwd-normalizado>--/<id-escapado>/session.v3.jsonl[.zstd]`. Root padrão: `~/.dsh/sessions`.

### Config `dsh-session-persistence-jsonl` 🟢
root (obrigatório), compression (`zstd` | `none`, padrão `zstd`).

### `SessionPersistence` (serviço) 🟢
create(header) → SessionHandle (write) · open(id, 'write'|'read') · stat(id) → { header, revision, eventCount?, sizeBytes? } · list() · flush()

### `SessionHandle.read` 🟢
`{ eventState: 'detached' | 'shared-frozen', events: SessionEvent[] }`

### Erros de persistência 🟢
SessionAlreadyOwnedError · SessionAlreadyExistsError · SessionReadOnlyError · SessionHandleClosedError · SessionOwnershipLostError · SessionPersistenceCorruptionError

### Descritor de formato 🟢
`readHeader()` → `current | migration-required | unsupported | malformed`

### `ProjectionDefinition<S, V>` 🟢
| Campo | Tipo | Obrig. |
|---|---|---|
| key | string | sim |
| stateSchema | schema | sim |
| stateVersion | int | sim |
| init | (header, inheritedEventCount) → S | sim |
| apply | (S, SessionEvent) → S (síncrono; mesma referência se não mudou) | sim |
| wire | { viewSchema, view(S) → V } | não |

### `sessionStats` 🟢
turns, steps, llmMs, toolMs, ttftMs, ttftSteps, decodeMs, decodeTokens (todos começam em 0)

### `turnOutline[]` 🟢
turn (número), seq (seq do turn/start), prompt (≤ 50 caracteres), response (≤ 120 caracteres)

### Config `dsh-session-title` 🟢
fallbackMaxWords, fallbackMaxBytes (≤ maxTitleBytes), maxTitleBytes. Todos obrigatórios, sem default.

### Config `dsh-session-telemetry-otel` 🟢
mode (`FEEDBACK_ONLY` | `DISABLED`), shutdownTimeoutMillis (3000), exporter { url, headers }, processor {}

### Campo `dsh_session_log` (v1) enviado à DeepSeek 🟢
sessionFormatVersion, header bruto (`seedLength` se houver fork), afterSeq, sufixo de eventos. Evento de controle: `session-log-deepseek/delivery-accepted { throughSeq, sessionFormatVersion }`. Config: `enabled` (padrão **true**).

---

## Módulo `context`

### Config `dsh-agent-instructions` 🟢
| Campo | Tipo | Padrão |
|---|---|---|
| maxBytes | int | obrigatório (base: 65.536) |
| maxSourceBytes | int | 1.048.576 |
| projectRootMarkers | string[] | `['.git']` |
| instructionFileCandidates | string[] | `['AGENTS.md','CLAUDE.md']` |
| localInstructionFileCandidates | string[] | `['AGENTS.local.md','CLAUDE.local.md']` |
| dshHome | string | `$DSH_HOME` / `~/.dsh` |

### Config `dsh-session-reference` 🟢
maxReferences (3, máx. 3) · candidateLimit (50) · maxReferenceBytes (automático) · referenceContextFraction (0,2)

### Config `dsh-time-context` 🟢
timeZone (zona do processo) · refreshIntervalMs (0)

### Config `dsh-tmux-context` 🟢
refreshIntervalMs (0)

---

## Módulo `compaction`

### Config `dsh-compaction-basic` 🟢
| Campo | Tipo | Padrão |
|---|---|---|
| thresholdRatio | number | 0,8 |
| retainRatio | number | 0,16 (exclusivo com retainTokens) |
| retainTokens | int | — |
| summarizationProvider / summarizationModel | string | '' (último roteado) |
| maxTokens | int | 8.192 |
| compactionRetries | int | 1 |
| maxOverflowRetries | int | 1 |
| modelPolicies | `{provider, model, ...política parcial}[]` | [] |
| auto | boolean | true |

### Config `dsh-compaction-tool-result-pruner` 🟢
thresholdChars (8.192) · headChars (4.096) · tailChars (1.024), em code points Unicode. Regra: head + marcador + tail ≤ threshold.

### Mensagem de checkpoint 🟢
User-role: `CHECKPOINT_PREAMBLE` + `<compacted-summary>` + seções (Primary Request and Intent, Key Technical Concepts, Files and Code, Errors and Fixes, Pending Jobs, Current Work, Next Step, Critical Context) + `</compacted-summary>`. Gravada como `replace` do intervalo condensado.

### Evento `image/offload` 🟢
Ocorrências selecionadas por (seq do evento de mensagem, índice da imagem em profundidade).

---

## Módulo `skill`

### Frontmatter de skill (`SKILL.md` / `<nome>.md`) 🟢
| Campo | Tipo | Obrig. | Padrão |
|---|---|---|---|
| name | string kebab-case | sim | — |
| description | string | sim | — |
| whenToUse | string | não | — |
| metadata | objeto | não | — |
| disable-model-invocation | booleano tolerante | não | false |
| user-invocable | booleano tolerante | não | true |

### Config `dsh-skill-filesystem` 🟢
providerName (`filesystem`) · includeDefaultRoots (true) · dshHome · agentsHome (`~/.agents`) · customSkillDirs ([]) · bundledSkillDir

### Config `dsh-tool-skill` 🟢
catalogDescriptionMaxLength (500, mínimo 3)

## Módulo `subagent`

### Config `dsh-tool-subagent` 🟢
provider (obrig.) · toolName (`subagent`) · enableRunInBackground (true) · backgroundMode (`one-shot` | `continuable`) · agentOptions { provider, model, reasoningEffort, maxTokens } · persona · toolFilter · maxDepth (3 | `'provider-managed'`) · modelSelectionSettings (false)

### Config `dsh-subagent-claude-code` 🟢
providerName (`claude-code`) · model · env ({}) · permissionMode (`dontAsk` | `acceptEdits` | `auto` | `plan` | `bypassPermissions`) · disposeGraceMs (3000)

### Ferramentas de controle 🟢
`send_message { agent_id, … } → { messageId }` · `interrupt_agent { agent_id }` · `list_agents { scope: children | descendants } → [{ id, status: running | idle | ready, parentSessionId, depth }]`

## Módulo `hooks`

### Config `dsh-hooks-claude-code` 🟢
configPath (obrig.) · pluginRoot · projectDir · defaultTimeoutMs (600.000) · stderrSummaryMaxChars (500)

### Hook (formato Claude Code) 🟢
`{ "<Evento>": [ { "matcher": "<regex>", "hooks": [ { "type": "command", "command": "...", "timeout": <s> } ] } ] }`. Eventos suportados: SessionStart, UserPromptSubmit, PreToolUse, PostToolUse, Stop, SubagentStart, SubagentStop.

### Saída de hook 🟢
exit 2 = bloquear (stderr = motivo). stdout JSON: contexto adicional, decisão ask/deny/allow, `continue: false` (sem efeito), `updatedInput` (ignorado).

### Eventos de sessão 🟢
`hook/invoked`, `hook/result` (com stderr resumido)

## Módulo `mcp`

### Config `dsh-mcp-client` 🟢
| Campo | Tipo | Padrão |
|---|---|---|
| serverName | `[A-Za-z0-9_-]{1,32}` | obrigatório |
| transport | `stdio` \| `streamable-http` | obrigatório |
| command, args, env, cwd | stdio | — |
| url, headers | http | — |
| toolCallTimeoutMs | int | 60.000 |
| maxInstructionBytes | int | 32.768 |
| failOnStartupError | bool | false |
| reconnect.enabled / initialDelayMs / maxDelayMs / maxAttempts | — | true / 500 / 30.000 / 10 |

Nome das ferramentas: `mcp__<serverName>__<tool>`. Ferramentas de recursos: list_mcp_resources, list_mcp_resource_templates, read_mcp_resource.

---

## Módulos `sandbox`, `interaction`, `shell`, `fs`

### Config `dsh-sandbox-policy` 🟢
mode (`read-only` | `workspace-write` | `danger-full-access`; padrão `read-only`) · workspaceRoot (absoluto; padrão `process.cwd()`)

### Config `dsh-sandbox-local` 🟢
runnerCommand ([]) · runnerFailureSignatures ([]) · probeTimeoutMs (5000)

### Campos de escalonamento nas ferramentas de shell 🟢
sandbox_permissions (modo mais amplo) · justification (string obrigatória)

### Config `dsh-repeat-tool-reminder` 🟢
thresholds ([3,5,8]) · include ([]) · exclude ([]) · argumentsPreviewChars (500)

### Config `dsh-user-approval` 🟢
policy (`ask` | `never`; padrão `ask`)

### Config `dsh-permission-presets` 🟢
presets: { <nome>: { sandbox, approval, name?, description? } } · defaultPreset

### `CommandDefinition` 🟢
name · description · input?: { hint } · handler({ agent, rawInput }) → { kind: success | error, text? } · definitionId? · recordInput? (true)

### `ask_user_question` (args) 🟢
questions[]: { id, question, header, options[]: { label, description } }

### `ShellExecSpec` / resultado 🟢
Entrada: command, workdir/cwd, timeoutMs, env, run_in_background, description. Resultado: exitCode | signal, timedOut, aborted, stdout/stderr { text, truncated, spillPath? }

### Config de ferramentas de shell 🟢
tool-bash / tool-pwsh: enableRunInBackground (true). tool-bash-persistent: backendType (`shell`), timeoutMs (300.000), maxOutputChars (16.000), description.

### Variáveis injetadas 🟢
DSH_HOME, DSH_SHELL=1, DSH_SESSION_ID

### Ferramentas de arquivo (args) 🟢
read { file_path, offset?, limit? } · read_image { file_path } · write { file_path, content } · edit { file_path, old_string, new_string, replace_all? } · glob { pattern, path? } · grep { pattern, path?, include? } · str_replace_editor { command, path, file_text?, old_str?, new_str?, insert_line?, view_range? } · present { files[{ path, description? }] }

### Códigos de erro de FS 🟢
FS_NOT_OBSERVED · FS_STALE_VERSION · FS_NOT_FOUND · SANDBOX_UNAVAILABLE · marcador `[sandbox: file access denied under <mode> mode]`

### Tabela de nomes: Claude Code → deepseek-harness 🟢
| Claude Code | dsh |
|---|---|
| Read / Write / Edit | read / write / edit |
| Glob / Grep | glob / grep |
| Bash / PowerShell | bash / pwsh |
| TodoWrite | todo_write |
| WebFetch / WebSearch | web_fetch / web_search |
| AskUserQuestion | ask_user_question |
| Task / Agent | subagent |
| Skill | skill |
| mcp__srv__tool | mcp__srv__tool |

---

## Módulos `plan`, `todo`, `goal`, `workflow`, `settings`, `credentials`, `preset`

### Config `dsh-plan-mode` 🟢
section (obrigatório; texto de `plan:policy`). Ferramenta: `exit_plan_mode`. Comando: `/plan [msg|off]`.

### `todo_write` (args) 🟢
todos[]: { content: string (não vazio, único), status: pending | in_progress | completed }. Config: allowParallelInProgress (obrigatório).

### Goal 🟢
Fases: active | paused | blocked | complete (+ armado, local). Config: defaultMaxGoalRounds (256). Ferramentas: get_goal, create_goal, update_goal.

### `workflow` (args/resultado) 🟢
Entrada: meta { name, description, whenToUse?, phases? } · script (JS) · args? (objeto). Saída: { runId, agentsStarted, result }.
Config `dsh-workflow-ptc`: provider (spawn), maxConcurrentAgents (0), maxTotalAgents (1000), maxItemsPerCall (4096), syncTimeoutMs (5000). `dsh-tool-workflow`: toolName (workflow), maxResultChars (50.000).

### `ralph` 🟢
Entrada: { objective, maxRounds? }. Relatório por rodada: { status: continue | complete | blocked, summary, evidence, nextSteps, blocker }. Config: subagentProvider (spawn), maxRounds (256), maxHandoffChars (16.384), maxResultChars (16.384).

### Config `dsh-settings-file` 🟢
path (`~/.dsh/settings.yaml`) · dshHome · watch (true) · debounceMs (100)

### Documento `settings.yaml` 🟢
Mapa `<namespace>: <seção do usuário>`. Exemplos: `llm-pi-ai: { providers: {...} }`, `agent-default-model: { provider, model, reasoningEffort }`, `agent-loop: { maxParallelToolCalls }`.

### Config `dsh-credentials-local` 🟢
path (`~/.dsh/.credentials.yaml`) · dshHome · watch (true) · debounceMs (100)

### API de credenciais 🟢
credentialRef(nome POSIX) · resolve → { value, source } · describe → { configured, source?, writable } · set · unset · records `<scope>/<id>`

### Fluxo de autorização 🟢
{ key: '<scope>/<id>', label, methods: [{ id: oauth | api-key, label }], run(session) } · session.notify({ message, url })

### Preset de agente 🟢
Pasta `<id>/` com `preset.yml` (display name, description) e `agent.cordis.yml` (plugins do agent plane). Config `dsh-agent-presets`: default (obrigatório), roots[{ path, trust: system | user }], includeShippedRoot (true), includeUserRoot (true). Embutidos: standard, ptc, cordis, minimal.

### Config `dsh-persona` 🟢
prefix (obrigatório) · suffix ('') · complete (false) · includeRuntimeContext (true)

### Diretório do usuário `~/.dsh` (observado na máquina) 🟢
profiles/ · sessions/ · settings.yaml · storages/ · (.credentials.yaml, .agent-presets/, skills/, AGENTS.md quando existirem)

---

## Módulos `extensions`, `bundle`, `boot`

### Perfil (`$DSH_HOME/profiles/<nome>`) 🟢
bundles[] (pacotes com `dsh.bundle.patch`) · `cordis.patch.yml` · patchReload: live | startup (custom: live)

### Linha do patch (`cordis.patch.yml`) 🟢
Substituição: { id, config (inteira) | disabled }. Inserção: { id, name (pacote, caminho absoluto, `./` relativo ou file URL), config?, inject?[] }. Valores aceitam `!!js <expr>`. `[]` = camada vazia válida. Arquivo vazio = erro.

### Precedência 🟢
- `.env`: ambiente herdado > `./.env` > `$DSH_HOME/.env`. Rejeita `PATH`/`DSH_*`/`XDG_*`. Proxies só do home.
- Composição: bundles → patch do perfil → `$DSH_HOME/cordis.patch.yml`.

### Entradas obrigatórias 🟢
agent-loop · webserver · modules · connection · headless-runner · acp · sdk-jsonrpc-server

### Serviços do launcher 🟢
ctx.cmdlineArgs (string[] imutável) · ctx.appExit(code) · ctx.appReady

### Config `session-telemetry-otel` 🟢
mode: FEEDBACK_ONLY | DISABLED (env `DSH_TELEMETRY_MODE`) · exporter.url (env `DSH_TELEMETRY_OTLP_URL`, padrão harness-telemetry.deepseeksvc.com/v1/logs) · compression gzip · timeoutMillis 1000 · shutdownTimeoutMillis 3000 · processor { scheduledDelayMillis 10000, maxQueueSize 2048, maxExportBatchSize 2048, exportTimeoutMillis 1500 } · opt-out: env `DSH_TELEMETRY_DISABLED` não vazio · identidade: `$DSH_HOME/.anonymous-user-id` (UUID)

### Ferramentas `tool-cordis` 🟢
cordis_inspect_self · cordis_inspect_list · cordis_inspect_query · cordis_define · cordis_run · cordis_stop · cordis_undefine. Config host runner: vmTimeoutMs.

---

## Módulo `acp`

### Config `dsh-acp` 🟢
provider (opcional) · model (opcional) · sessionListPageSize (100)

### Opções de configuração da sessão 🟢
model (escolhas opacas provider/model do catálogo LLM vivo) · reasoning_effort (só se o modelo exato declarar)

### Entrada MCP em `session/new` 🟢
stdio: { command absoluto, args, env } · http: { url absoluta http(s), headers }. SSE e ACP-transport são rejeitados.

### Conteúdo de prompt 🟢
text · resource_link · image (PNG, JPEG, WebP, GIF; exige attachment store e rota com imagem)

---

## Módulos `jobs`, `schedule`

### Job 🟢
id `<kind>-N` · kind · status (running | stopping | settled…) · owner (sessão de agente) · output (stream ou final) · outputLimitBytes (do produtor)

### Config `tool-jobs` 🟢
waitTimeoutMs (30.000) · maxWaitTimeoutMs (600.000; acima disso, clamp) · completionDelivery (wakeup | quiet) · maxConsecutiveWakes (3). Config `jobs-local`: limite por dono (10).

### ScheduleRecord (evento `schedule/change` v1) 🟢
scheduleId (local à sessão, nunca reutilizado) · prompt (com trim) · scheduledAt (RFC 3339 UTC) · kind: after { afterSeconds } | at {} | every { everySeconds, acceptedAt na dispatch }

### Erros do schedule (união fechada v1) 🟢
persistence_uncertain · schedule_not_found · prompt vazio · mais de um seletor · fuso inválido · horário não futuro ou fora de faixa · intervalo < 5 min

---

## Módulo `web`

### Config `tool-web` 🟢
search (true) · fetch (true) · searchMaxResults (8) · searchMaxQueries (4) · fetchTimeoutMs (30.000) · searchTimeoutMs (30.000) · fetchMaxOutputChars (200.000)

### Config `web-fetch-http` 🟢
maxResponseBytes (5.000.000) · maxBodyChars (100.000) · timeoutMs (30.000) · maxRedirects (5, só mesma origem) · userAgent (`deepseek-harness/…`)

### Config `web` 🟢
searchProvider (sem valor = automático) · fetchProvider (sem valor = automático)

### Resultado de busca 🟢
answer? (content) · sources[]: { url, title?, snippet?, publishedAt? }

---

## Módulos `lsp`, `terminal`

### `lsp` (args) 🟢
operation: goToDefinition | findReferences | goToImplementation | hover · file_path · line (≥1, UTF-16) · character (≥1, UTF-16)

### Config `tool-lsp` 🟢
maxLocations (100) · maxResultChars (16.000) · timeoutMs (60.000)

### Config `lsp-stdio` 🟢
command (obrigatório) · extensionToLanguage (obrigatório) · args ([]) · env ({}, ambiente sem credenciais) · initializationOptions (null) · configuration (null) · maxMessageBytes (16.000.000) · maxStderrBytes (1.000.000) · maxDocumentBytes (4.000.000) · shutdownTimeoutMs (5.000) · killGraceMs (2.000)

### Config `terminal-bash` 🟢
backendType (shell) · shellDialect (bash | pwsh) · maxReadBytes (262.144) · timeoutMs (30.000) · disposeGraceMs (3.000)

### Config `tool-terminal` 🟢
enableRunInBackground (true) · maxResultBytes (262.144, mínimo 64)

---

## Módulos `spill`, `storage`, `session-query`

### Resultado de `spillStore.save` 🟢
locator (caminho opaco) · bytes (exato) · guidance (texto de recuperação)

### Config `spill-local` 🟢
root (diretório temporário privado 0700) · cleanupPeriodDays (30; 0 desliga)

### Config `spill-policy` 🟢
maxInlineBytes (sem valor = desligada; base = 50.000)

### Backends de storage 🟢
json: { root, layout: single | per-record } · sqlite: { path }. Evento: `domain/changed`.

### Config `session-query-sqlite` 🟢
path (obrigatório ou `:memory:`) · openAt (startup | first-search | never; base = never) · journalMode (wal) · defaultLimit (20) · maxLimit (100) · snippetChars (240) · readWindowMax (50) · persistedReadConcurrency (4) · preparedSessionCacheSize (5)

### Config `tool-session-query` 🟢
maxSearchResults (100) · searchTimeoutMs (30.000). Ferramentas: session_search · session_event_search · session_trace · session_event_trace · session_event_read

---

## Módulos `attachment`, `ptc-runtime`

### Config `attachment-local` 🟢
dshHome · maxImageBytes (20 MiB) · maxImagesPerMessage (20) · maxMessageImageBytes (200 MiB) · maxImagePixels (64.000.000) · maxImageDimension (8192) · normalizedImageMaxPixels (2048×2048) · normalizedImageMaxDimension (8192) · normalizedImageMaxBytes (4 MiB) · imageCompressionConcurrency (2)

### Resultado do PTC 🟢
completion: { value (JSON sem perda) } | failure: { error estruturado } · logs: canais ordenados (stdout/stderr)

### Config `ptc-runtime-node` 🟢
timeoutMs (120.000) · maxTimeoutMs (600.000) · maxOutputBytes (67.108.864) · maxOldGenerationSizeMb (512) · maxMessageBytes (134.217.728) · maxPendingCalls (128) · graceMs (3.000) · nodeExecutable · bootstrapPath

---

## Módulos `typert`, `api`, `sdk`

### Métodos do protocolo SDK (JSON-RPC 2.0, NDJSON em stdio) 🟢
initialize · session/prompt · shutdown (cliente→servidor) · session.event · session.status · subagent.started · subagent.finished (servidor→cliente)

### Namespaces Remote da API web 🟢
session · skills · fileReferences · settings · credentials · workspace · directoryPicker · workspaceFiles · terminal

### Artefatos Typert 🟢
reflexão de pacote · fábricas de schema Zod (preguiçosas, em cache) · InvocationDescriptor (Remote) · exports `./typert` e `./client/typert`

---

## Módulos `vendor/cordis`, `util`

### Pacotes vendored 🟢
cordis 4.0.0-rc.7 · cordis-plugin-loader 1.0.0-rc.5 · include 1.0.4 · group 1.0.0 · timer 1.1.2 · hmr 1.0.15 · logger-console 1.0.0 · cosmokit 1.8.1 · schemastery 3.18.0 (todos MIT, escopo `@deepseek-ai`)

### Entrada do Loader (`cordis.yml`) 🟢
{ id, name (pacote), config?, inject?[], disabled? (bool ou !!js), group? (entradas aninhadas) }

### Ordem de confiança do ambiente (`launch-environment`) 🟢
herdado > `<cwd>/.env` > `$DSH_HOME/.env` (camadas excluíveis por busca)

### Raiz de dados (`home-paths`) 🟢
caminho explícito > `$DSH_HOME` > `~/.dsh`

---

## Módulo `apps/cli`

### Flags do launcher 🟢
--profile <nome> · --from-default-profile <template> · --patch <path> (repetível) · --dump-config · --dump-default-config · -h/--help

### Manifesto do perfil (`package.json` → `dsh.profile`) 🟢
bundles[] (ordenado) · patchReload (live | startup) · dependências de plugins fora da árvore (pnpm)

### Códigos de saída 🟢
0 (sucesso, SIGTERM) · 130 (SIGINT) · ≠0 (falha de parse, schema, resolução ou boot). Drenagem graciosa de 5 s.

### Exemplo de MCP de memória (overlay) 🟢
{ id: memory-memorix, name: @deepseek-ai/dsh-mcp-client, config: { serverName, transport: stdio, command, args, cwd } }

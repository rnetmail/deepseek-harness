# Análise de Código — deepseek-harness

> Gerado pelo **Reversa Archaeologist** · nível `completo` · iniciado em 2026-11-01.
> Escala: 🟢 CONFIRMADO (lido no código/README do pacote) · 🟡 INFERIDO · 🔴 LACUNA
> Fluxogramas: `_reversa_sdd/flowcharts/<modulo>.md` · Entidades: `_reversa_sdd/data-dictionary.md`
>
> **Convenção do repositório:** cada pacote é um plugin cordis que expõe um serviço em `ctx.<nome>` e documenta seu contrato no README, nas seções "Use this package" e "Understand the implementation". Esta análise usa esses READMEs como fonte primária e o código-fonte para confirmar tipos, constantes e defaults.

---

## Módulo `llm` — camada de modelos e provedores

**Pacotes:** `llm/llm` (serviço `ctx.llm`, tipos canônicos), `llm/llm-deepseek` (rota `deepseek-official`), `llm/llm-pi-ai` (multi-provedor), `llm/llm-retry` (executor de retry), `llm/token-meter` (medição de contexto), `llm/deepseek-llm-api-extensions`, `llm/plugin-package-inventory-deepseek`.
**Tamanho:** ~14.500 linhas TS em `src/`. **Complexidade:** alta.

### 1. Contrato central (`llm/llm`) 🟢
- `LlmRuntime` (`llm/src/index.ts`) é o único caminho até os provedores. Funções principais:
  - `stream(options: GenerateOptions): AsyncIterable<StreamChunk>`: uma chamada de modelo, com chunks no nível de token.
  - `registerAdapter(adapter)`: registra rotas. Uma rota duplicada falha com `DUPLICATE_ADAPTER`. A troca do conjunto de rotas é atômica: se for recusada, o estado anterior continua servindo.
  - `listProviders()`, `listModels(provider)`, `discoverModels(req)`, `resolveModelInfo(provider, model)`.
  - `prepareCall()`: amarra os fatos do modelo, o contexto e a política de retry à *geração exata* do adapter (protege contra HMR e troca de settings no meio da chamada). Um `PreparedLlmCall` pode ser despachado uma única vez.
  - `imageRequestPricing(provider, model)` e `fileRequestText(ref)`, usados pelo token-meter.
- `LlmAdapter` (classe abstrata): cada adapter cuida só do próprio *wire format*. O vocabulário é neutro em relação ao provedor.
- **Protocolo de stream** (`types.ts`): `block-start → (text|reasoning|tool-call)-delta* → block-end → usage → finish`. Há sempre **exatamente um** `finish` terminal, e nada vem depois dele. Os argumentos de tool-call trafegam como string JSON crua.
- `BlockAssembler` (`assembler.ts`) monta os blocos a partir dos chunks. `AssistantStreamAccumulator` (`assistant-stream.ts`) guarda o stream compactado, com timestamps por token, e esse registro é embutido no evento durável da tentativa.

### 2. Regras e invariantes 🟢
| Regra | Onde |
|---|---|
| **Visível ao modelo ⟺ registrado no log**: o request é reconstruível a partir do log da sessão; requests montados pelo loop são *deep-frozen* | `llm/README.md#invariants` |
| O *replay state* (assinaturas de thinking etc.) só viaja quando o mesmo adapter é dono da rota histórica e da rota de destino; caso contrário, é descartado | idem |
| `FileBlock` nunca chega ao provedor: vira uma linha de texto determinística com o caminho read-only, e o modelo lê o arquivo com suas ferramentas | `content.ts: projectFilesToText` |
| Modelo só-texto recebe placeholder determinístico por imagem, inclusive em imagens aninhadas em tool-result, sem reescrever o histórico | `content.ts: projectImagesForTextModel` |
| Imagens acima do `LlmImageRequestBudget` geram `IMAGE_OFFLOAD_REQUIRED`, indicando as N mais antigas; a compactação faz o offload e repete | `content.ts: requiredImageOffload` |
| `reasoningEffort` é validado contra o modelo exato **antes** de qualquer I/O | `call-config.ts` |
| `systemPromptUpdate: 'in-history'`: o modelo lê o último `system` em qualquer posição, então o loop anexa o prompt alterado em vez de reescrever o início (preserva o cache) | `types.ts:381` |
| Consumidores roteiam pelo **código** da falha, nunca pelo texto | `error.ts` |

**Códigos de falha 🟢:** `NO_ADAPTER`, `DUPLICATE_ADAPTER`, `MISSING_CREDENTIAL`, `INVALID_CREDENTIAL`, `AUTH`, `RATE_LIMIT`, `QUOTA`, `SERVER`, `TIMEOUT`, `TRANSPORT`, `EMPTY_RESPONSE`, `CONTEXT_WINDOW_EXCEEDED`, `IMAGE_OFFLOAD_REQUIRED`, `UNKNOWN_MODEL`, `INVALID_CONFIG`, `UNSUPPORTED_OPTION`, `INVALID_MODEL_INFO`, `UNSTORABLE_PROVIDER_ID`. A detecção de estouro de contexto e de cota por texto do provedor fica centralizada em `isContextWindowExceededError` e `isQuotaExceededError`.

### 3. `llm-pi-ai`: multi-provedor (base do requisito "modelos gratuitos + assinaturas") 🟢
- **Configuração:** um dicionário `providers.<rota>` com `apiKeyEnv`, `displayName`, `api`, `baseURL`, `models[]`, `modelOverrides`, `compat`, `headers`, `reasoning`, `thinkingBudgets`, `cacheRetention` (`none|short|long`), `transport` (`sse|websocket|websocket-cached|auto`), limites de imagem e `retryPolicy`.
- **Protocolos de wire suportados pelo harness:** `openai-completions`, `openai-responses`, `anthropic-messages` (`provider.ts:47`). 🟢 O catálogo do pi-ai conhece mais protocolos (`google-generative-ai`, `google-vertex`, `bedrock-converse-stream`, `mistral-conversations`, `azure-openai-responses`, `openai-codex-responses`), mas `PROTOCOLS` só mapeia esses três. 🟡 Provedores com outros protocolos provavelmente só funcionam através de um endpoint compatível com OpenAI.
- **Catálogo instalado (pi-ai 0.85.1):** **39 provedores**: amazon-bedrock, ant-ling, anthropic, azure-openai-responses, baseten, cerebras, cloudflare-ai-gateway, cloudflare-workers-ai, deepseek, fireworks, github-copilot, google, google-vertex, groq, huggingface, kimi-coding, minimax(+cn), mistral, moonshotai(+cn), nvidia, openai, openai-codex, opencode, opencode-go, openrouter, qwen-token-plan(+cn, individual), together, vercel-ai-gateway, xai, xiaomi(+3 token-plans), zai, zai-coding-cn.
- **Modelos com custo zero no catálogo: 113**. Exemplos: `openrouter/*:free` (cerca de 20), `nvidia/*` (cerca de 20, incluindo deepseek-v4, kimi-k3 e gpt-oss-20b), `opencode/*-free`, `google/gemma-4-*`, `huggingface/GLM-4.7-Flash`, `mistral/labs-devstral-small`, além dos modelos incluídos em planos de assinatura (qwen-token-plan, xiaomi-token-plan, zai highspeed). → **Fonte direta para a "listagem automática de modelos gratuitos"**.
- **Login por assinatura (OAuth) disponível no pi-ai 🟢:** Anthropic (Claude Pro/Max), OpenAI (ChatGPT Plus/Pro via `openai-codex`), GitHub Copilot, xAI (SuperGrok / X Premium), Kimi Code, OpenRouter. A credencial fica em `llm-pi-ai/<provider id>` no credential store e é renovada sob um lock entre processos (`login.ts`, `auth.ts`).
- **Resolução de catálogo:** `models` *substitui* o catálogo da rota, e cada entrada herda os campos do modelo instalado com o mesmo id. `modelOverrides` ajusta modelos individuais, e é recusado se vier junto com `models`, em rota declarada à mão ou para um id inexistente.
- **Descoberta de modelos:** uma rota do catálogo é respondida **sem rede**. Uma rota declarada à mão usa `GET {baseURL}/models` com bearer (OpenAI) ou `GET /v1/models?limit=1000` com `x-api-key` e `anthropic-version` (Anthropic). O parser aceita tanto o array `data` quanto o mapa `models`. O resultado é só uma sugestão: nada é gravado, quem decide é o `settings.yaml`.
- **Hot-reload:** os perfis são relidos a cada operação, mesclando o perfil base com a seção `llm-pi-ai:` do usuário, provedor a provedor. Uma escrita inválida é rejeitada com `settings-rejected`. Uma falha posterior mantém o último valor válido.
- **Defaults 🟢:** `defaultContextWindow=262144`, `defaultMaxTokens=32768`, `requestImagePixelBudget=4194304`, `requestImageMaxBytes=1 MiB`, `maxRequestImageBytes=20 MiB`, `streamIdleTimeoutMs=300000`.
- `GenerateOptions.stop` é rejeitado com `UNSUPPORTED_OPTION`.

### 4. `llm-deepseek` 🟢
- Rota única `deepseek-official`. Protocolos: `messages` (padrão, `https://api.deepseek.com/anthropic` + `/v1/messages`) ou `chat-completions` (`https://api.deepseek.com` + `/chat/completions`).
- O id do modelo é repassado como está, então modelos novos não exigem re-registro. Catálogo padrão: `deepseek-flash` (texto e imagem, `systemPromptUpdate: in-history`), `deepseek-v4-flash`, `deepseek-v4-pro`, `deepseek-v4-flash-vision-exp`, todos com janela de 1M de tokens.
- Esforços de raciocínio: `off|low|high|max` (padrão `high`). Saída padrão de 256.000 tokens.
- Imagens via Files API, com upload, cache local, renovação (`fileRefreshMarginSeconds=3600`), expiração (7 dias) e limpeza por cota (lotes de 100).
- `baseURL` é resolvido nesta ordem: valor explícito → `$DEEPSEEK_BASE_URL` → padrão do protocolo.

### 5. `llm-retry`: algoritmo de retry 🟢
- **Modo `normal`** (padrão): até 5 retries para `EMPTY_RESPONSE`, `RATE_LIMIT`, `SERVER`, `TIMEOUT`, `TRANSPORT`. Backoff exponencial limitado de 500 ms a 10 s, com 10% de jitter.
- **Modo `always`:** consulta a recuperação a jusante primeiro e depois repete qualquer falha sem limite de tentativas. Só para em sucesso, cancelamento ou dispose.
- Um `Retry-After` válido do provedor substitui o backoff local se couber nos limites da política.
- **Durabilidade:** grava o evento `llm/retry` (id, provider, modo, falha, delay) **antes** da espera, e `llm/retry-started` logo antes da nova tentativa. O passo é re-executado no mesmo turno, sobre o mesmo histórico durável.
- Nada do retry é visível ao modelo.
- Atua no ponto de extensão *failed-step* do agent-loop. Chamadas diretas a `ctx.llm.stream()` são de tentativa única.

### 6. `token-meter`: medição de contexto 🟢
- `measure(session, requestHeader?)` devolve um snapshot imutável `{ totalTokens, surfaceTokens, nodes[] }`. `estimateMessage(msg)` estima uma única mensagem.
- **Heurística:** `ceil(len/4)` + `BLOCK_OVERHEAD` por bloco (`estimate.ts:13`, `CHARS_PER_TOKEN = 4`). O README admite que CJK e schemas JSON ficam subestimados.
- Reaproveita o `usage` real do provedor quando o envelope canônico da última chamada bem-sucedida bate com o envelope atual. Caso contrário, estima tudo.
- Projeções de sessão: `tokenUsage` (uncached/output/cacheRead/cacheWrite), `contextPressure` (`pressureTokens`, `projectedTokens`, `contextWindow`), `contextBreakdown` (system/tools/messages).
- A compactação decide a partir de `measure()`. A ocupação mostrada na UI é só referência.

### 7. Dependências do módulo
`typert` (serviço remoto), `credentials` (`apiKeyEnv`), `settings` (hot-reload), `attachment` e `fs` (projeção de arquivos e imagens), `session` (log durável), `core/agent-loop` (ponto de extensão de retry), `compaction` (offload de imagens).

### 8. Implicações para o port em Rust 🟡
- Um trait `LlmAdapter` com `stream() -> impl Stream<Item = StreamChunk>` e um enum `StreamChunk` reproduzem o contrato 1:1.
- Os três protocolos de wire (OpenAI Chat, OpenAI Responses, Anthropic Messages) já cobrem a grande maioria dos provedores gratuitos e de assinatura.
- O catálogo (`models.generated.js` do pi-ai) pode ser exportado para JSON/TOML e embutido no binário, com atualização via download.
- Os fluxos OAuth (Claude Pro/Max, ChatGPT Plus/Pro, Copilot) precisam ser portados um a um; são os mais sensíveis.

---

## Módulo `core` — agente, loop ReAct, ferramentas, sessão e system prompt

**Pacotes:** `core/agent` (`ctx.agents`), `core/agent-loop` (driver ReAct), `core/tools` (`ctx.tools`), `core/session` (`ctx.sessions`, event sourcing), `core/system-prompt` (`ctx.systemPrompt`), `core/scope` (escopo por agente), `core/agent-default-model`, `core/agent-tool-presentation`. **Complexidade:** muito alta. É o coração do harness.

### 1. Agente (`core/agent`) 🟢
- `ctx.agents.create({ sessionId, agentOptions: { provider, model, reasoningEffort?, maxTokens? }, parentAgent?, setup? })` e `ctx.agents.resume(...)` devolvem um `AgentHandle`. **Só quem tem o handle pode chamar `dispose()`**: o disposer funciona como capability.
- `get(id)`, `list()`, `roots()`, `isOwnedBy(id, parent)`. `parentAgent` cria um filho em runtime (base dos subagentes).
- **Inbox durável** com duas filas: `nextTurn` (prompts) e `nextStep` (steering).
  - `followup()` enfileira um prompt para o próximo turno e acorda o driver.
  - `steer()` enfileira input para o próximo passo e acorda o driver.
  - `inject()` adiciona contexto **sem acordar** o driver.
  - `cancel(cause, keepInbox?)` aborta e, por padrão, limpa as filas.
  - `whenIdle()` resolve quando o agente fica quiescente.
  - Toda mutação gera o evento `agent/inbox/spliced` com coordenadas de splice. Remoções levam `outcome: 'canceled'`.
- **Eventos de extensão:**
  - `agent/pre-step`: rejeita um passo ou substitui as mensagens (`{kind:'reject'} | {kind:'enter', messages, startsRequestSeries?}`).
  - `agent/request-error`: um listener devolve `{kind:'retry'}`. É aqui que o `llm-retry` se encaixa.
  - `agent/turn-stopping`: pode fazer steering para manter o turno aberto. É a base de `goal` e `ralph`.
  - `agent/assistant-stream`: frames ao vivo, só para apresentação.
  - `agent/status` (`idle|running`), `agent/created`, `agent/disposed`, `agent/inbox/{inserted,claimed,discarded}`.
- **Initiator scope:** usa `AsyncLocalStorage` para carregar o agente através do trabalho assíncrono. Serve para atribuição, não para autorização.

### 2. Loop ReAct (`core/agent-loop`) 🟢
- Config: `maxParallelToolCalls` (padrão **10**; `1` = serial) e `agents[]` declarativos (`id`, `provider`, `model`, `reasoningEffort`, `maxTokens`, `cwd`, `sessionId` | `resumeSessionId`). Um id de sessão novo segue o formato `${id}-session-<uuid>`.
- **Turno:** abre um turno durável e reivindica atomicamente o input pendente de `nextStep` mais **um** prompt de `nextTurn`. Entre passos, reivindica só `nextStep`.
- **Passo:**
  1. Monta o prompt e as ferramentas e projeta o contexto de runtime.
  2. Roda `agent/pre-step`.
  3. Grava `step/start`.
  4. Roda o waterfall `agent/request` e depois `ctx.llm.prepareCall()`.
  5. Admite o system prompt e as mensagens do usuário.
  6. Grava `request/header` (só quando há mudança) e `request/context` (só quando há diferença).
  7. Faz o stream e grava `assistant/message` (ou `assistant/attempt`, se a tentativa falhou ou foi cancelada).
  8. Executa as tool calls pelo pipeline com guardas.
  9. Grava `tool/call` e `tool/result`, e repete até `finish != tool-calls`.
- **Concorrência de ferramentas** (`tool-calls.ts`): chamadas *parallel-safe* se sobrepõem até `maxParallelToolCalls`. Chamadas *exclusive* rodam sozinhas e funcionam como barreira de ordem.
- **Admissão do system prompt:** numa rota sem `in-history` ou num início de série, o texto é consolidado no nó 0, com substituições vazias registradas nos nós posteriores. Numa série `in-history`, o prompt alterado é *anexado*, o que preserva o cache do provedor.
- **Falhas:** falhas do adapter viram `finish` e entram em `agent/request-error`. Falhas de plugin, middleware ou ferramenta encerram o **turno**, não o loop. Tool calls não despachadas depois de um cancelamento recebem o par sintético `ABORTED_BEFORE_DISPATCH`.
- **Persistência:** `create` chama `persistence.create(header)`. `resume` abre a sessão em modo write (exclusivo) e anexa `interruptedTurnClosers` se o processo caiu no meio de um turno.

### 3. Ferramentas (`core/tools`) 🟢
- `defineTool({ name, description, parameters, output: { schema, render }, execute(args, exec) })`. A DSL de schema aceita `string|number|integer|boolean|null|array|object|json|oneOf`, e `InferValue` tipa até 16 níveis de aninhamento. O JSON Schema cru (`JsonSchemaNode`) é compartilhado com MCP, subagentes e workflows.
- Os argumentos do modelo são validados antes da execução. Um input inválido vira resultado de erro, sem exceção.
- **Pipeline fixo:**
  1. `tools/pre-execute` decide allow/deny/ask (extensível, reordenável).
  2. Os guardas monotônicos de `ctx.tools.guard()` rodam em seguida: uma negação nunca é revertida.
  3. `tools/execute` envolve o despacho (timeout, retry, métricas).
  4. `tools/post-execute` pode inspecionar ou substituir o resultado e anexar contextos.
  5. `finalizeContent`.
  6. `tools/result` (só observação).
- **Códigos:** `UNKNOWN_TOOL`, `ABORTED_BEFORE_DISPATCH`, `ABORTED`, `TOOL_TIMEOUT`. O cancelamento é cooperativo via `exec.signal`.
- `ctx.tools.restrict(filter)` aplica máscaras allow/deny por agente (as máscaras se intersectam). `schemas(scope)` devolve os schemas visíveis.
- **Modo de apresentação** (`mode`):
  - `native`: todos os schemas.
  - `ptc`: só `run_code` mais um SDK gerado em TS ou Python, e o modelo escreve código que chama as ferramentas.
  - `both`.
  - O `run_code` do runtime Node tem timeout de 120 s por padrão e 600 s no máximo. Os ids das sub-chamadas seguem `<parent>:ptc:<n>`, e `maxParallelSubCalls` vale 10.

### 4. Sessão (`core/session`): event sourcing 🟢
- Uma `Session` é um **log append-only** de `SessionEvent` tipados. Tudo é derivado dele: histórico do modelo, transcrição, telemetria, título e persistência.
- **Tipos de evento centrais:** `turn/start`, `turn/end`, `step/start`, `step/end`, `user/message`, `system/message`, `assistant/message`, `assistant/attempt`, `tool/call`, `tool/result`, `request/header`, `request/context`, `session/end-seed`. Plugins acrescentam outros: `agent/inbox/spliced`, `llm/retry`, `image/offload`, compactação etc.
- **Eventos de superfície** (`system/message`, `user/message`, `assistant/message`, `tool/result`) exigem `surfaceOp`: `append` ou `{ op:'replace', startSeq, endSeq }` (intervalo inclusivo). A compactação é uma *substituição* de um intervalo da superfície.
- **Validação no append:** `snapshotJsonValue()` copia e valida JSON sem perda; BigInt, ciclos, arrays esparsos e `-0` são rejeitados. Um `tool/result` com `data.error` só é aceito se `content[0].isError === true`.
- `deriveMessages()` faz a projeção incremental e cacheada da superfície para `Message[]`. Plugins registram projeções puras com `registerMessageProjection()`.
- `fork(source, boundary?, childId?)` só aceita um prefixo que termine fora de um turno aberto. O filho recebe `isSeeded` e `inheritedEventCount`.
- `flush(session)` é a barreira de durabilidade.
- **Motivos de fim de turno:** `completed`, `aborted{reason}`, `blocked`, `error{LlmFailure}`.
- **Invariantes** (`invariant.ts`): seq contíguo, passos dentro de turnos, pares `tool/call` ↔ `tool/result`.
- `repair.ts` faz o reparo a frio de logs órfãos depois de um crash.

### 5. System prompt (`core/system-prompt`) 🟢
- Seções `{ name, order, text | fn, interpolate?, complete? }` são concatenadas por `order` crescente e, em caso de empate, por nome.
  - O opener fixo "You are an AI agent powered by DeepSeek Harness." fica na ordem −1000 (`includeHarnessIdentity`).
  - `personaPrefix` fica na ordem 0 e `personaSuffix` na ordem 10200.
  - Uma seção `complete: true` substitui o prompt inteiro. Mais de uma seção `complete` gera erro.
- **Variáveis** `{{name}}` são resolvidas a cada montagem (o loop fornece `model` e `cwd`). Uma variável desconhecida ou sem valor faz a montagem **falhar** de propósito.
- `toolOrder` define a ordem das ferramentas, com exatamente uma entrada `'<unlisted-tools>'`.
- O contexto de runtime (hora, tmux etc.) vira um *snapshot user-role* no histórico, não texto do system prompt. `suppressRuntimeContext()` desliga esse contexto.
- O escopo do agente sobrepõe seções e variáveis globais de mesmo nome.

### 6. Escopo (`core/scope`) 🟢
- `createScope(ctx, key)` liga visibilidade e ciclo de vida: o que é registrado pelo escopo some no `dispose()`. A cadeia de pais funciona nos dois sentidos: registros são herdados para baixo e eventos são admitidos para cima.
- O escopo **não é sandbox nem fronteira de autoridade**. Serve para rotear plugins confiáveis do mesmo processo.

### 7. Modelo padrão (`core/agent-default-model`) 🟢
`currentSelection()` e `saveSelection({provider, model, reasoningEffort?})`. A config da composição é a base e a seção de settings é a fonte viva. `reasoningEffort` existe só nos settings.

### 8. Implicações para o Rust 🟡
- O event sourcing com log JSONL é o contrato de persistência mais importante. Portar `SessionEvent` como enum serde com tag `type` mantém compatibilidade com sessões já existentes.
- O loop ReAct pode ser uma task tokio por agente, com inbox em `mpsc` e um semáforo de `maxParallelToolCalls`.
- O pipeline de ferramentas e os pontos de extensão (`pre-step`, `request-error`, `turn-stopping`, `pre/post-execute`) são exatamente onde se encaixam hooks, permissões, plan mode e goal. Precisam virar traits de middleware.

---

## Módulo `session` — formato, persistência, checkpoints e projeções

**Pacotes (19):** `session-format`, `session-format-catalog`, `session-format-v0-to-v1`, `-v1-to-v2`, `-v2-to-v3`, `session-persistence` (contrato), `session-persistence-jsonl` (backend), `session-checkpoint-policy`, `session-projection`, `session-projection-cache`, `session-stats`, `session-turn-outline`, `session-title` (+ `-llm`, `-first-prompt-llm`, `-all-prompts-llm`), `session-telemetry` (+ `-otel`), `session-log-deepseek`. **Complexidade:** alta.

### 1. Formato e migrações 🟢
- Versão atual: **`SESSION_FORMAT_VERSION = 3`** (`core/session/src/types.ts:88`).
- `createSessionFormatCatalog({ currentVersion, codecs, currentEncoder, migrations, restoreCurrent… })`. `readHeader()` classifica o arquivo em `current`, `migration-required`, `unsupported` ou `malformed` sem ler os eventos. A restauração é em *streaming*: `decodeRow()` linha a linha, depois `finish()`.
- As migrações são encadeadas entre versões adjacentes (v0→v1→v2→v3). Um formato mais novo que o build é recusado e o operador é orientado a atualizar.
- **Recuperação:** com `recovery: 'recoverable'`, uma linha malformada ou com gap de sequência é descartada junto com o sufixo não confirmado. Se um `turn/end` aparecer depois da linha ruim, o erro passa a ser fatal.
- Um tipo de evento desconhecido é recusado, a menos que o envelope o marque como `ignorable`. Isso é importante para compatibilidade entre o harness Rust e o TS.

### 2. Persistência (contrato `ctx.sessionPersistence`) 🟢
- Métodos: `create(header)`, `open(id, 'write'|'read')`, `stat(id)`, `list()`, `flush()`. Toda leitura e escrita passa pelo `SessionHandle` (`read(offset?, length?)`, `append`, `flush`, `close`).
- **Writer único por processo.** Erros: `SessionAlreadyOwnedError`, `SessionAlreadyExistsError`, `SessionReadOnlyError`, `SessionHandleClosedError`, `SessionOwnershipLostError`, `SessionPersistenceCorruptionError`.
- **Caminho de escrita:** `session/event` vai para uma janela de batching interna. `session/flush` é a barreira imediata. `session/disposed` faz o drain final. Uma falha em background preserva os eventos, pausa o caminho automático e é reportada no próximo flush.
- **Crash:** a persistência devolve o log fisicamente válido, e o reparo semântico fica com o leitor. No resume, o agent-loop anexa `interruptedTurnClosers`: `tool/result` de erro sintéticos, `step/end` e `turn/end {interrupted}`.

### 3. Backend JSONL 🟢
- Config: `root` (obrigatório, sem default; a composição base usa `dshHomePath('sessions')`, ou seja, `~/.dsh/sessions`) e `compression` (`zstd` com frames checksummed, padrão, ou `none`).
- **Layout em disco:**
  ```
  <root>/--<cwd-normalizado>--/<id-escapado>/session.v3.jsonl.zstd   (ou session.v3.jsonl)
  ```
  - A primeira linha é o header físico, e sua versão bate com o nome do arquivo. As linhas seguintes têm um evento cada.
  - Quando há mais de uma geração, vale a de número mais alto.
  - O id é escapado de forma injetiva, sem traversal nem colisão.
- No Windows, a publicação atômica usa `MoveFileExW` com write-through via koffi.

### 4. Política de checkpoint 🟢
Três barreiras de durabilidade, todas *fail-closed*:
1. **Antes do stream do modelo**, para que um crash não reexecute um request não persistido.
2. **Antes do corpo de cada tool call de topo**, para que a chamada seja durável antes de qualquer efeito externo. Chamadas aninhadas reutilizam esse checkpoint.
3. **Em cada `agent/pre-step`**, para que tudo o que o passo anterior confirmou seja gravado antes do próximo request.

Um cancelamento durante um flush pendente devolve `ABORTED_BEFORE_DISPATCH`.

### 5. Projeções de sessão 🟢
- `ProjectionDefinition { key, stateSchema, stateVersion, init(header, inheritedEventCount), apply(state, event) (síncrono, puro), wire?: { viewSchema, view } }`. Uma unidade sem `wire` fica só no host.
- `snapshot(session) → { asOfSeq, values }`, `onChanged(listener)`, `stateOf(session, key)`. Um evento que carrega estado deve trazer o **estado completo** depois da mudança, nunca um delta.
- Unidades registradas: `title`/`titleInput`, `sessionStats` (`turns`, `steps`, `llmMs`, `toolMs`, `ttftMs`, `decodeMs`, `decodeTokens`…), `turnOutline` (prévia do prompt com até 50 caracteres e da resposta com até 120), `tokenUsage`, `contextPressure`, `contextBreakdown`, `inbox`, `todo`, `goal` etc.

### 6. Título 🟢
- Três fontes, e a mais recente vence: um fallback determinístico (as primeiras palavras da primeira mensagem humana, limitado por `fallbackMaxWords` e `fallbackMaxBytes`), um provedor via LLM (primeiro prompt ou todos os prompts) e `rename()` manual.
- Um título definido pelo usuário *fixa* a sessão. Só `refresh()` desfaz isso.
- O título nunca chega ao modelo.

### 7. Telemetria 🟢
O `session-telemetry-otel` exporta via OTLP/HTTP. Modos: `FEEDBACK_ONLY` (padrão: só envia quando há feedback do usuário) e `DISABLED`.

### 8. ⚠️ `session-log-deepseek`: envio do log da sessão para a DeepSeek 🟢
- **Ligado por padrão** (`enabled: true`) e montado na composição base (`bundle/base/cordis.patch.yml:36`).
- Em todo request à API oficial da DeepSeek que tenha um `sessionId` vivo (agente, compactação, título), anexa o campo `dsh_session_log` com o **sufixo do log de eventos** ainda não confirmado. Isso inclui header, mensagens, tool calls e resultados, ou seja, conteúdo de arquivos e saídas de comandos.
- Depois de um HTTP 2xx, grava `session-log-deepseek/delivery-accepted { throughSeq }`. A entrega é *at-least-once*.
- O modelo não vê esse campo, e o custo em tokens é zero. Só o tamanho do request HTTP aumenta.
- **Para desligar:** overlay com `enabled: false`.
- 🔴 **Para o harness Rust e para o uso corporativo (TOTVS/GP), não reproduzir esse comportamento**, ou mantê-lo desligado por padrão.

---

## Módulo `context` — instruções do projeto e contextos injetados

**Pacotes:** `agent-instructions`, `file-reference` (+ `-local`), `session-reference`, `time-context`, `tmux-context`. **Complexidade:** média.

### 1. `agent-instructions`: equivalente ao CLAUDE.md 🟢
- No primeiro request entra **uma** mensagem-base durável com:
  1. `$DSH_HOME/AGENTS.md` (global do usuário);
  2. a **cadeia do projeto**: todo arquivo candidato existente, da raiz do projeto até o cwd da sessão, do mais amplo para o mais específico.
- **Candidatos:** `AGENTS.md` e `CLAUDE.md` (base), mais `AGENTS.local.md` e `CLAUDE.local.md` (overlays locais). A raiz é marcada por `.git`.
- Arquivos irmãos com conteúdo idêntico (depois de trim) aparecem uma vez só, então um CLAUDE.md que duplica o AGENTS.md não se repete.
- **Descoberta incremental:** depois de um `read`/`write`/`edit` bem-sucedido num diretório mais profundo, o próximo request inclui o arquivo de instruções que passou a valer. Um arquivo alterado substitui o conteúdo anterior, e um arquivo removido sai do contexto.
- **Config:** `maxBytes` (obrigatório; a base usa 65.536), `maxSourceBytes` (1 MiB), `projectRootMarkers` (`['.git']`), `instructionFileCandidates`, `localInstructionFileCandidates`, `dshHome`.
- Uma falha de permissão ou I/O interrompe a descoberta com erro, em vez de cair silenciosamente num ancestral.
- ➜ **Compatibilidade com `D:/Projetos/Totvs AGP/VSCode`:** os CLAUDE.md desse projeto já são lidos. Ficam faltando `.claude/agents/*.md`, `.claude/skills`, `.claude/settings.json` (hooks e permissões) e `.mcp.json`, que vou cobrir nos módulos skill, subagent, hooks e mcp. 🟡

### 2. `file-reference`: menções `@arquivo` 🟢
- Token `@path` no início ou depois de espaço; `@"path com espaços"`. `list(agent, query, signal)` devolve candidatos ranqueados do cwd.
- O prompt `FILE_REFERENCE_PROMPT` orienta o modelo a ler o arquivo antes de afirmar que o inspecionou.

### 3. `session-reference`: citar outra sessão 🟢
- Sintaxe `@[label](dsh-session:<base64url-id>)`. A mensagem é seguida de um snapshot `## Referenced sessions` como segunda mensagem user-role, marcado como **background não confiável**: o modelo não deve seguir instruções contidas nele.
- Limites: `maxReferences` (3, máximo 3), `candidateLimit` (50), `referenceContextFraction` (0,2). Orçamento automático por fonte = `max(65536, floor(contextWindow × 4 × fração))` bytes.
- Se houver truncamento, a transcrição completa vai para o `spill`, com um locator de recuperação.

### 4. `time-context` e `tmux-context` 🟢
- **time-context:** a cada passo elegível injeta uma mensagem de três linhas: timestamp ISO com offset e zona IANA, política de zona e tempo decorrido. Config `timeZone` (fallback) e `refreshIntervalMs` (0 = todo passo). Prefere a zona do navegador quando existe uma zona única validada no turno.
- **tmux-context:** quando a localização tmux muda, injeta sessão, janela e pane (sem o conteúdo de outros panes). Só considera estar no tmux se o tty de controle bater com `#{pane_tty}`.
- Os dois são *runtime context*: viram mensagens user-role no histórico, não texto do system prompt, o que preserva o cache do prefixo.

---

## Módulo `compaction` — condensação de contexto

**Pacotes:** `compaction` (base abstrata + marcador), `compaction-basic` (backend com sumarização via LLM), `compaction-tool-result-pruner`, `compaction-image-offload`, `command-compact` (`/compact`). **Complexidade:** alta.

### 1. Gatilhos 🟢
| Gatilho | Condição |
|---|---|
| Automático | `measure().totalTokens ≥ floor(contextWindow × thresholdRatio)` (padrão 0,8) |
| Overflow | falha confirmada `CONTEXT_WINDOW_EXCEEDED`: compacta e repete (`maxOverflowRetries` = 1) |
| Manual | `/compact`, mesmo abaixo do limiar (sem argumentos) |
| Região explícita | `compactRegion(start, end)` programático |

### 2. Algoritmo (`compaction-basic/src/region.ts`, `summarizer.ts`) 🟢
1. Se o pruner estiver montado, poda primeiro os tool results com mais de `thresholdChars` (8.192 code points). Cada um vira cabeça (4.096) + marcador "middle pruned" + cauda (1.024). Se o resultado já couber no limiar, **não sumariza**.
2. Seleciona o **span balanceado mais antigo**, preservando a cauda recente verbatim (`retainRatio` 0,16 da janela ou `retainTokens`). As fronteiras nunca separam um par tool-call/tool-result (`toolPairingBalancedBefore/After`).
3. **Sumariza com cache quente:** reproduz o system prompt, as ferramentas e as mensagens do último request roteado, e anexa `COMPACTION_INSTRUCTION` no final. A chamada auxiliar vira um *prefixo* do request anterior e reaproveita o KV cache do provedor.
4. O resumo tem estrutura fixa em Markdown: `Primary Request and Intent`, `Key Technical Concepts`, `Files and Code`, `Errors and Fixes`, `Pending Jobs`, `Current Work`, `Next Step`, `Critical Context`. Um checkpoint anterior (`<compacted-summary>`) é consolidado, não copiado.
5. Valida que as fronteiras não mudaram durante a sumarização (`assertStable`). Grava a substituição do intervalo (`surfaceOp: replace`) por uma mensagem user-role com o preâmbulo `CHECKPOINT_PREAMBLE` + `<compacted-summary>…</compacted-summary>`.
6. Se a pressão continuar acima do limiar, faz até `compactionRetries` (1) tentativas extras.

- **Falhas:** um resumo truncado (`MAX_TOKENS`) ou com imagem (`UNSUPPORTED_CONTENT`) falha fechado, e a tentativa fica registrada no log. Se não houver span seguro, nada muda.
- **Modelo do resumo:** `summarizationProvider/Model` (o padrão é o último roteado). `maxTokens` = 8.192.
- `modelPolicies[]` define overrides por `{provider, model}`. `auto: false` deixa só o modo manual.

### 3. Offload de imagens 🟢
Em `IMAGE_OFFLOAD_REQUIRED {offloadImages: n}`, marca as n imagens mais antigas (pulando o assistant) com um evento `image/offload` e repete. Isso não consome o orçamento de retry do provedor.

### 4. Mensagens do `/compact` 🟢
São estáveis: "No compactable history yet.", "Usage: /compact (no arguments)", indisponível (agente ocupado), histórico alterado, resumo inútil, término incompleto, falha ao salvar, "Compaction cancelled.".

### 5. Implicações para o Rust 🟡
O prompt de compactação e o formato `<compacted-summary>` podem ser portados literalmente. Também é preciso manter a regra de não quebrar pares tool-call/result e a técnica de sumarizar reaproveitando o prefixo de cache, que é crucial para o custo em modelos pagos.

---

## Módulo `skill` — registro, provedor filesystem e ferramenta `skill`

**Pacotes:** `skill` (`ctx.skills`), `skill-filesystem`, `tool-skill`, `skill-badge`. **Complexidade:** baixa/média.

### 1. Registro 🟢
- Catálogo mesclado de todos os provedores, ordenado por nome. O corpo da skill é carregado sob demanda e revalidado no carregamento.
- `ctx.skills.register(skill)` registra skills embutidas (provedor `runtime`). `registerProvider(p)` registra provedores externos.
- **Política de invocação:** `modelInvocable` (catálogo e ferramenta do modelo) × `userInvocable` (comando `/nome`).
- Um provedor que falha é logado e pulado, e o catálogo anterior válido é mantido. Uma skill malformada falha cedo. Config: `collectCacheMaxEntries` (128).

### 2. Formato e raízes (`skill-filesystem`) 🟢
- `<nome>/SKILL.md` ou `<nome>.md` no topo de cada raiz. `**/SKILL.md` aninhado **não** é descoberto.
- **Frontmatter YAML:** `name` e `description` (obrigatórios); `whenToUse`, `metadata`, `disable-model-invocation`, `user-invocable` (opcionais). Os booleanos aceitam true/false, yes/no, on/off e 1/0, e um valor inválido descarta a skill.
- **Raízes por prioridade** (rank menor vence):

| Rank | Origem | Caminho |
|---|---|---|
| 100 | project-dsh | `<raiz>/.dsh/skills` |
| 200 | project-agents | `<raiz>/.agents/skills` |
| 300 | custom | `customSkillDirs` |
| 400 | user-dsh | `~/.dsh/skills` (ignora `.system`) |
| 500 | user-agents | `~/.agents/skills` (`$DSH_AGENTS_HOME`) |
| 600 | bundled | `$DSH_BUNDLED_SKILL_DIR` / `bundledSkillDir` |

- As raízes são observadas (watch), então skills novas ou removidas aparecem sem reiniciar. O corpo é relido a cada carregamento.
- 🟡 **Lacuna de compatibilidade:** `.claude/skills` (do projeto e de `~/.claude/skills`) **não está nas raízes padrão**. Dá para incluir via `customSkillDirs`. O harness Rust deve incluí-las por padrão para reaproveitar `D:/Projetos/Totvs AGP/VSCode` e `~/.claude/skills` (onde estão, por exemplo, as skills `reversa-*`).

### 3. Ferramenta e catálogo (`tool-skill`) 🟢
- Antes do primeiro request entra uma mensagem user-role durável com o **catálogo**: nome e descrição limitada a `catalogDescriptionMaxLength` = 500. Ela orienta o modelo a carregar a skill antes de agir.
- Ferramenta `skill(name)` → bloco `<skill_content>` com o corpo e orientação sobre recursos.
- Um `/nome` na entrada do usuário injeta a skill diretamente.
- Mudanças no conjunto de skills anexam um catálogo substituto completo.
- Erros: `Error: invalid skill name "<name>"`, skill desconhecida, skill indisponível para o modelo.

---

## Módulo `subagent` — delegação

**Pacotes:** `subagent` (`ctx.subagents`), `tool-subagent`, `tool-subagent-control`, provedores `subagent-spawn-in-process`, `subagent-fork-in-process`, `subagent-in-process-driver`, `subagent-acp`, `subagent-claude-code`, `subagent-codex`, `subagent-dsh-sdk`. **Complexidade:** alta.

### 1. Contrato 🟢
- Um provedor registra capacidades como `agentOptions`, `persona`, `toolFilter`, `depthLimit`, `prepareContinuable`. Pedir algo que o provedor não suporta **falha no início**, não é ignorado.
- **Filho one-shot:** roda uma vez e devolve o texto final (mais saída estruturada opcional e diagnóstico seguro).
- **Filho contínuo (continuable):** sessão durável e id estável. Aceita `send_message` (steer se estiver ocupado, novo turno se estiver ocioso, retomada a frio se for filho direto) e `interrupt_agent` (só para o turno atual).
- Os filhos são isolados: um crash no filho não corrompe a sessão do pai.

### 2. Ferramenta `subagent` (`tool-subagent`) 🟢
| Campo | Padrão | Significado |
|---|---|---|
| provider | obrigatório | `spawn`, `fork`, `acp`, `claude-code`, `codex`… |
| toolName | `subagent` | um nome distinto por instância |
| enableRunInBackground | true | expõe `run_in_background` |
| backgroundMode | `one-shot` | `continuable` torna background o padrão |
| agentOptions | — | provider, model, reasoningEffort e maxTokens do filho |
| persona | — | persona do filho |
| toolFilter | — | restrição de ferramentas do filho |
| maxDepth | 3 | profundidade máxima de delegação (0 proíbe) |
| modelSelectionSettings | false | — |

- Background one-shot: `started background subagent job <id>`, consultado com `job_output` e encerrado com `job_kill`.
- Controle: `send_message(agent_id, …)`, `interrupt_agent`, `list_agents` (`children` | `descendants`, com status `running|idle|ready`).

### 3. Provedores externos 🟢
- **`subagent-claude-code`:** roda uma sessão real do Claude Code via `@anthropic-ai/claude-agent-sdk` 0.3.263, instalada por `dsh plugin --profile <p> add`.
  - `permissionMode`: `dontAsk` (padrão), `acceptEdits`, `auto`, `plan`, `bypassPermissions`.
  - Variáveis de ambiente com cara de credencial são removidas; a chave vai explicitamente em `env`.
  - Lê as settings normais do Claude Code (usuário, projeto e local).
- **`subagent-codex`** (`@openai/codex`), **`subagent-acp`** (qualquer agente ACP) e **`subagent-dsh-sdk`**.
- ➜ O harness Rust pode **delegar ao Claude Code quando ele estiver disponível** e usar os modelos alternativos quando não estiver. É o inverso do cenário de fallback, mas reaproveita a mesma infraestrutura. 🟡
- 🟡 **Lacuna:** não encontrei leitura de definições `.claude/agents/*.md` (frontmatter name, description, tools, model). As personas vêm de `preset/agent-presets`, a confirmar no módulo `preset`.

---

## Módulo `hooks` — compatibilidade com hooks do Claude Code e do Codex

**Pacotes:** `hook-protocol` (motor compartilhado), `hooks-claude-code`, `hooks-codex`. **Complexidade:** média.

### 1. Semântica 🟢
- Só executa hooks `{ type: 'command', command, timeout? }`. Os tipos `http`, `mcp_tool`, `prompt` e `agent` são lidos e **pulados** com aviso.
- **Exit 2** bloqueia, e o stderr vira o motivo. Qualquer outro código é falha não bloqueante (logada, a execução continua).
- O stdout em JSON pode: anexar contexto, pedir `ask` (só no Claude Code) ou pedir `continue: false` (registrado, mas **sem efeito** no nível da execução).
- Hooks do mesmo evento rodam **em sequência**, na ordem da config. O matcher é uma regex; ausente, vazio ou `*` casa com tudo.
- Um config ilegível gera aviso e nenhum hook roda, mas o agente sobe.
- `updatedInput` é lido mas **não aplicado** (reescrita de input adiada).
- Eventos de sessão `hook/invoked` e `hook/result`, com resumo do stderr limitado a 500 caracteres.

### 2. Eventos do Claude Code suportados 🟢
`SessionStart` (contexto) · `UserPromptSubmit` (bloquear/contexto) · `PreToolUse` (bloquear/ask) · `PostToolUse` (bloquear com feedback/contexto) · `Stop` (forçar mais um passo) · `SubagentStart` (contexto, só in-process) · `SubagentStop` (só observação).
🔴 **Não suportados:** `PreCompact`, `Notification`, `SessionEnd` e outros eventos mais recentes do Claude Code.

### 3. Config 🟢
`configPath` (obrigatório; um `hooks.json` ou settings com chave `hooks`), `pluginRoot` (`${CLAUDE_PLUGIN_ROOT}`), `projectDir` (`${CLAUDE_PROJECT_DIR}`, também exportado como env), `defaultTimeoutMs` (600.000), `stderrSummaryMaxChars` (500). O config é lido **uma vez** no startup, para o processo inteiro.

### 4. Lacunas para o harness Rust 🟡
- O **matcher compara com o nome da ferramenta do harness**. Se as ferramentas tiverem nomes diferentes dos do Claude Code (`bash` × `Bash`, `str_replace_editor` × `Edit`), os matchers do `.claude/settings.json` não casam. 🔴 Confirmar no módulo `fs`/`shell`. O harness Rust deve adotar os nomes do Claude Code (`Bash`, `Read`, `Edit`, `Write`, `Glob`, `Grep`…) ou manter uma tabela de aliases.
- Ler automaticamente `.claude/settings.json` e `settings.local.json` (projeto e usuário), em vez de um `configPath` fixo.

---

## Módulo `mcp` — cliente MCP

**Pacotes:** `mcp-client`, `mcp-resources`. **Complexidade:** média.

### 1. Cliente 🟢
- Um plugin por servidor. Transportes `stdio` (command/args/env/cwd, com env ambiente higienizado) e `streamable-http` (url/headers). SDK oficial `@modelcontextprotocol/client` 2.0.0, protocolo 2026-07-28 com fallback para revisões anteriores.
- **As ferramentas aparecem como `mcp__<serverName>__<tool>`**, o mesmo padrão do Claude Code. 🟢 `serverName`: `[A-Za-z0-9_-]{1,32}`.
- Config: `toolCallTimeoutMs` (60.000), `maxInstructionBytes` (32.768; as instruções do servidor entram no prompt), `failOnStartupError` (false), `reconnect { enabled: true, initialDelayMs: 500, maxDelayMs: 30.000, maxAttempts: 10 }` com backoff dobrando a cada tentativa.
- Uma falha de conexão no início não derruba o harness: as ferramentas do servidor só não aparecem.

### 2. Recursos 🟢
`list_mcp_resources`, `list_mcp_resource_templates`, `read_mcp_resource(server, uri)`, com paginação por cursor. O system prompt lista os servidores visíveis.

### 3. Lacunas 🟡
- 🔴 **Não há leitura de `.mcp.json`** nem de `mcpServers` do Claude Code/Desktop. A config é por linhas YAML do cordis (só o ACP recebe `mcpServers` do cliente). O harness Rust deve importar `.mcp.json` (projeto), `~/.claude.json` e `claude_desktop_config.json`.
- Não suporta transporte SSE legado. 🟡
- ➜ O **REA** entra aqui: um servidor `stdio` pré-registrado (`npx rea-agents` / `rea mcp`), com perfis de schema `REA_MCP_INPUT_SCHEMA_PROFILE=compact` para economizar contexto.

---

## Módulo `sandbox + guard` — confinamento e guardas

**Pacotes:** `sandbox` (contrato `ctx.sandbox`), `sandbox-policy`, `sandbox-local`, `sandbox-windows-acl`, `native/system` (launcher Landlock em C), `guard/timeout-policy`, `guard/repeat-tool-reminder`. **Complexidade:** alta.

### 1. Modos 🟢
| Modo | Efeito |
|---|---|
| `read-only` | Nega escritas, exceto em sinks obrigatórios (`/dev/null`). **É o padrão fail-safe** de `sandbox-policy` |
| `workspace-write` | Permite escritas sob o workspace (o cwd imutável da sessão) e numa área temporária privada |
| `danger-full-access` | Sem confinamento |

- O nível de enforcement é informado **a cada chamada**: `full` ou `partial`. Hoje são `partial` o Windows ACL (Everyone e hard links) e o Landlock com ABI antiga.
- **Backends:** Linux com bubblewrap ou Landlock (`native/system`); macOS com Seatbelt 🟡; Windows com token `WRITE_RESTRICTED` mais ACLs (`AclSandbox` via koffi). O Windows **confina só escrita**, não leitura nem rede.
- **Fail-closed:** se nenhum backend consegue aplicar o modo, a chamada falha com `SANDBOX_UNAVAILABLE`. Nunca roda sem confinamento.
- **Escalonamento:** uma negação devolve `[sandbox: file access denied under <mode> mode]` mais uma dica. O modelo pode repetir **a mesma chamada uma vez** com `sandbox_permissions` (o modo mais estreito que resolve) e `justification`. O usuário vê um único prompt (permitir uma vez, rejeitar ou cancelar), e a permissão vale só para aquela chamada.
- O modo pode ser trocado por sessão em runtime. A troca é registrada no log, sobrevive a reinícios e entra no contexto de runtime do modelo.
- Config `sandbox-local`: `runnerCommand`, `runnerFailureSignatures`, `probeTimeoutMs` (5.000).

### 2. Guardas 🟢
- **timeout-policy:** só atua em ferramentas que declaram um limite (por exemplo `web_fetch` e `web_search`, 30 s). Aborta `exec.signal` e devolve `Error: tool call timed out after <ms>ms`. `bash`, `read`, `write` e `edit` **não têm limite** por esta via; o timeout delas é do executor.
- **repeat-tool-reminder:** detecta chamadas idênticas consecutivas (mesma ferramenta e mesmos argumentos, sem importar a ordem das chaves). Com `thresholds` `[3, 5, 8]`, anexa um lembrete ao resultado. Também aceita `include`, `exclude` e `argumentsPreviewChars` (500). Só lembra, nunca bloqueia.

---

## Módulo `interaction` — aprovações, permissões, comandos e perguntas

**Pacotes:** `user-approval`, `permission-presets`, `commands`, `user-questions`, `tool-ask-user`. **Complexidade:** média.

### 1. Aprovação 🟢
- `ctx.userApproval.request({ agent, tool, callId?, reason?, signal })` exige um turno aberto. Os *answerers* são listeners do waterfall `approval/request`, e a composição tem um answerer terminal (a UI web, o ACP etc.).
- **Sem answerer, resolve `unavailable` e falha fechado.** A política vem da sessão ou do default: `ask` (padrão) ou `never` (rejeita tudo, para CI e headless).
- Toda decisão é auditada no log. Se o append falhar, a decisão é rejeitada.
- `setPolicy(agent, p)` avisa o modelo no próximo passo.

### 2. Presets de permissão 🟢
Cada preset combina um modo de sandbox com uma política de aprovação, por exemplo `workspace-write` = workspace-write + ask e `danger-full-access` = full + never. `defaultPreset` vale para sessões novas. `custom` e `auto` são nomes reservados.

### 3. Comandos (slash) 🟢
- `ctx.commands.register({ name, description, input?: { hint }, handler, definitionId?, recordInput? })`. O nome tem letras minúsculas, dígitos, `_` e `-`.
- A sintaxe é `/` no byte 0, depois o nome, depois espaço ou fim da entrada. O resto é `rawInput`. Um comando desconhecido é rejeitado pelo adapter e não vira prompt.
- O handler devolve `success` ou `error` com texto para a UI e roda **sem** criar mensagem para o modelo.
- Comandos registrados: `/compact`, `/goal`, `/plan`, `/feedback`, `/export`, `/cordis` e skills `user-invocable`.

### 4. `ask_user_question` 🟢
Recebe `questions[]` com `id`, `question`, `header` e `options[]` (`label`, `description`). A opção recomendada vem primeiro, com "(Recommended)". Exige o serviço `ctx.userQuestions` com um answerer; sem ele, a chamada dá erro. Equivale ao AskUserQuestion do Claude Code.

---

## Módulo `shell + subprocess` — execução de comandos

**Pacotes:** `shell` (contrato `ctx.shell`), `bash-local`, `bash-sandbox`, `pwsh-local`, `pwsh-sandbox`, `tool-bash`, `tool-bash-persistent`, `tool-pwsh`, `tool-pwsh-persistent`, `shell-env`; `subprocess` (contrato), `subprocess-local` (node-pty, koffi), `win32-process`; `terminal/*` (PTY interativo). **Complexidade:** alta.

### 1. Contrato `ctx.shell` 🟢
- `resolve(request) → ShellExecSpec`: preenche cwd e timeout e limita os overrides. Não existem defaults ocultos em `run`/`start`.
- `run(spec)`: exit≠0, timeout e abort são **resultados**, não exceções. Só falhas de infraestrutura rejeitam a promise.
- `start(spec)` roda em background sem timeout. `readOutput()` é incremental e não repete saída. `kill()`, `done`.
- A saída guarda a **cauda** (`maxBytes`) em memória e o fluxo completo num *spill* (diretório 0700, arquivos 0600). Uma saída truncada aponta para o spill.

### 2. Ferramentas 🟢
| Ferramenta | Execução | Observações |
|---|---|---|
| `bash` (one-shot) | `bash -c` novo a cada chamada | sem estado: usar `workdir` em vez de `cd`; `[exit code: N]` não é erro; `description` de 5–10 palavras; `timeoutMs`; `run_in_background` → job |
| `bash` (persistente) | um shell PTY por agente | `cd` e `export` persistem; `timeoutMs` 300.000 (o timeout fecha o shell); `maxOutputChars` 16.000; fecha com `[Command finished with exit code N]` |
| `pwsh` / `pwsh` persistente | `pwsh -Command` | Windows; no modo read-only sandbox, roda em ConstrainedLanguage |
| `job_output` / `job_list` / `job_kill` | jobs em background | `job_output(wait?)` |
| `terminal_open/send/read/signal/list/close` | PTY interativo | para programas que leem stdin |

- O `shell-env` injeta `DSH_HOME`, `DSH_SHELL=1` e `DSH_SESSION_ID` em toda chamada. Plugins podem acrescentar outras variáveis `DSH_*`.
- **subprocess-local:** resolve executáveis no PATH higienizado (.COM, .EXE, .BAT, .CMD no Windows) e rejeita caminhos relativos com separador. No Windows, os filhos rodam com janela oculta. No Linux, usa scopes do user-systemd para limpar descendentes. Há um pipe de controle no fd 7.

---

## Módulo `fs` — ferramentas de arquivo e busca

**Pacotes:** `fs` (contrato `ctx.fs`), `fs-local`, `fs-sandbox`, `fs-observation-policy`, `tool-fs`, `tool-fs-search`, `tool-str-replace-editor`, `tool-present`. **Complexidade:** média.

### 1. Ferramentas 🟢
| Ferramenta | Argumentos | Comportamento |
|---|---|---|
| `read` | `file_path`, `offset?` (1-based), `limit?` (≤ `readLimit`) | conteúdo numerado por linha com rodapé de paginação |
| `read_image` | `file_path` | PNG/JPEG/WebP/GIF salvo como anexo; recusado se a rota for só-texto |
| `write` | `file_path`, `content` | cria ou substitui; para sobrescrever, exige um `read` anterior da mesma versão |
| `edit` | `file_path`, `old_string`, `new_string`, `replace_all?` | substituição literal; o match tem de ser único se não houver `replace_all`; exige `read` anterior |
| `glob` | `pattern`, `path?` | ripgrep embutido; inclui arquivos ocultos e ignorados (exceto VCS); ordenado por mtime; padrão sem `/` casa com basenames em qualquer profundidade |
| `grep` | `pattern`, `path?`, `include?` | regex do ripgrep, agrupado por arquivo `Line N: <preview>`; um único glob positivo |
| `str_replace_editor` | `command` (`view`/`create`/`str_replace`/`insert`), `path`… | vocabulário alternativo (estilo Anthropic); `maxOutputChars` 16.000 |
| `present` | `files[{path, description?}]` (≤ 8) | entrega arquivos ao usuário na UI |

- 🟢 Os **argumentos** seguem o Claude Code (`file_path`, `old_string`, `new_string`, `replace_all`), mas os **nomes** são minúsculos (`read` × `Read`). Ver a lacuna de hooks.

### 2. Política de observação (read-before-write) 🟢
- `write` cria livremente, mas só sobrescreve um arquivo que a sessão já leu. `edit` exige leitura prévia.
- Um arquivo alterado depois da leitura gera `FS_STALE_VERSION`. A ausência também é registrada: ler um arquivo inexistente permite um create guardado depois.
- Códigos: `FS_NOT_OBSERVED`, `FS_STALE_VERSION`, `FS_NOT_FOUND`. A mensagem normalizada é `cannot modify "<path>": file has not been read — read the file, then retry`.
- Ao retomar uma sessão, o estado de observação **não** é restaurado, e é preciso reler os arquivos.

### 3. Contrato `ctx.fs` 🟢
Resolver identidade, ler texto inteiro ou em stream, ler bytes até um limite, listar um nível de diretório, criar ou substituir atomicamente e aplicar edição literal atômica com *version guard* opcional. Erros tipados `FsError`. `fs-sandbox` confina as mutações ao workspace e às raízes temporárias.

### 4. Implicações para o Rust 🟡
- Mapa de nomes do Claude Code: `Read`→`read`, `Write`→`write`, `Edit`→`edit`, `Glob`→`glob`, `Grep`→`grep`, `Bash`→`bash`, `PowerShell`→`pwsh`, `TodoWrite`→`todo_write`, `WebFetch`→`web_fetch`, `WebSearch`→`web_search`, `AskUserQuestion`→`ask_user_question`, `Task/Agent`→`subagent`, `Skill`→`skill`. O harness Rust deve adotar os nomes do Claude Code, ou aceitar os dois nos matchers de hooks e permissões.
- Crates prontos: `grep-searcher` + `ignore` (ripgrep), `portable-pty`, `windows-rs` para tokens restritos e ACL, `landlock` (Linux).

---

## Módulo `plan + todo` — plan mode e lista de tarefas

**Pacotes:** `plan/plan-mode`, `todo/tool-todo`. **Complexidade:** baixa.

### 1. Plan mode 🟢
- `/plan` entra no modo; `/plan <mensagem>` entra já com uma instrução; `/plan off` sai. A mensagem pode levar imagens e arquivos.
- Enquanto o modo está ativo, a seção de prompt `plan:policy` (config `section`, **obrigatória**) é renderizada. O agente apresenta o plano chamando **`exit_plan_mode`**, e o usuário revisa.
- 🟢 **Diferença importante em relação ao Claude Code:** o plan mode **não restringe ferramentas**. Todas continuam chamáveis, e os limites reais ficam por conta do sandbox e das aprovações. No Claude Code, o plan mode bloqueia edições. 🟡 O harness Rust deve oferecer as duas opções: um modo consultivo e um modo restritivo (sandbox `read-only` automático).

### 2. `todo_write` 🟢
- O agente envia a **lista inteira** a cada atualização, que substitui a anterior. Cada item tem uma descrição e um status `pending | in_progress | completed`.
- Retorno: `Updated todo list: <p> pending, <i> in progress, <c> completed.`. Falha em descrição vazia ou duplicada.
- `allowParallelInProgress` é **obrigatório**: com `false`, só um item pode estar `in_progress`.
- A lista pertence a uma única sessão de agente; subagentes têm a própria. Estado via projeção `todo`.

---

## Módulo `workflow + goal` — orquestração e execução autônoma

**Pacotes:** `goal/goal`, `goal/goal-round-driver`, `goal/tool-goal`, `goal/command-goal`; `workflow/workflow`, `workflow/workflow-ptc`, `workflow/tool-workflow`, `workflow/tool-ralph`. **Complexidade:** alta.

### 1. Goal (objetivo de longo prazo) 🟢
- No máximo **um goal atual por sessão**. Fases duráveis: `active`, `paused`, `blocked`, `complete`, mais uma flag local de *armado*. `defaultMaxGoalRounds` = 256.
- Ferramentas `get_goal`, `create_goal`, `update_goal` e comando `/goal`. Projeção `goal` (versão 6).
- **Driver de rodadas:** com o agente ocioso, um goal ativo e armado, e rodadas sobrando, enfileira um prompt de rodada (objetivo entre aspas JSON, número da rodada e limite) como mensagem user-sourced do goal, iniciando uma nova série de requests.
  - Para em: conclusão, pausa, bloqueio, `max-tokens`, falha de durabilidade, cancelamento ou unload.
  - **Depois de resume ou fork, o goal fica desarmado** até uma retomada autorizada por humano.
- Equivale ao `/loop` dinâmico e ao "ScheduleWakeup" do Claude Code. 🟡

### 2. Workflow (orquestração multi-agente) 🟢
- Ferramenta `workflow { meta: { name, description, whenToUse?, phases? }, script, args? }` → `{ runId, agentsStarted, result }`. O turno do pai fica bloqueado até o fim, e o modelo vê só o resultado final.
- O script é **JavaScript puro** com top-level `await` e termina em `return <json>`. Hooks: `agent(prompt, opts)` (com schema opcional para saída estruturada), `parallel()`, `pipeline()`, `phase()`. Uma falha comum de um filho resolve `null`.
- Engine `workflow-ptc` roda numa VM Node do runtime PTC. Limites: `provider` (`spawn`), `maxConcurrentAgents` (0 = número de CPUs), `maxTotalAgents` (1000), `maxItemsPerCall` (4096), `syncTimeoutMs` (5000). `maxResultChars` = 50.000.
- **É praticamente o mesmo modelo da ferramenta Workflow do Claude Code** (meta, script, `agent`/`parallel`/`pipeline`/`phase`). 🟢

### 3. Ralph 🟢
- `ralph { objective, maxRounds? }`: a cada rodada um filho **novo**, sem contexto do pai, trabalha no workspace compartilhado (que funciona como memória). Entre rodadas passa só um relatório estruturado `{ status: continue|complete|blocked, summary, evidence, nextSteps, blocker }` (≤ 16.384 caracteres).
- Resultado: `complete | blocked | budget-limited`. **Desabilitado por padrão.**

### 4. Implicações para o Rust 🟡
O workflow exige um runtime de script. Opções: embutir QuickJS ou Boa (JS) no binário Rust para manter a compatibilidade dos scripts, ou usar Rhai/Lua com uma API equivalente.

---

## Módulo `settings + credentials` — configuração e segredos

**Pacotes:** `settings/settings`, `settings/settings-file`, `credentials/credentials`, `credentials/credentials-local`, `credentials/authorization`. **Complexidade:** média.

### 1. Settings 🟢
- `ctx.settings.register(ns, schema, { base })` → `scope.get()` (snapshot deep-frozen), `scope.update(patch)`, `watch((next, prev))`. Namespace em minúsculas, dígitos e hífen.
- **Camadas:** a config da composição cordis é a `base`, e a seção do usuário fica por cima (merge por namespace).
- **`settings-file`:** `~/.dsh/settings.yaml` (ou `.yml` / `.json`), `watch` true, `debounceMs` 100.
  - Hot-reload de edições externas.
  - Um documento inválido **no boot** impede o carregamento. Uma edição inválida **em runtime** gera aviso e mantém a última versão válida.
  - Escrita por diff de folhas YAML, preservando comentários e anchors. O merge com edições concorrentes usa um lock com prazo de 2 s e backoff.
  - O arquivo é criado com permissão 0600 num diretório 0700 e substituído atomicamente (sem seguir symlinks).
- Namespaces vistos até aqui: `llm-pi-ai`, `llm-deepseek`, `agent-default-model`, `agent-loop` (`maxParallelToolCalls`) etc.

### 2. Credenciais 🟢
- `credentialRef('NOME')` (identificador shell POSIX). `resolve(ref) → { value, source }`, `describe(ref) → { configured, source?, writable }` (nunca devolve o valor), `set`, `unset`.
- **Records** `<scope>/<id>` guardam credenciais de fluxos (OAuth), por exemplo `llm-pi-ai/openai-codex`.
- **Store local:** `~/.dsh/.credentials.yaml`, com watch e debounce de 100 ms. A ordem de resolução é store e depois variável de ambiente. 🟡
- O próprio README admite que **as ferramentas do agente rodam como o seu usuário do SO e conseguem ler o arquivo**: a permissão de arquivo não protege as chaves contra o agente. 🟢

### 3. Autorização (OAuth / login interativo) 🟢
`ctx.authorization.registerFlow({ key: '<scope>/<id>', label, methods: [{ id: 'oauth' | 'api-key', label }], run(session) })`. Dentro do fluxo, `session.notify({ message, url })` orienta o humano. A UI ou o CLI disparam as tentativas. Usado pelo `llm-pi-ai` para as assinaturas (ChatGPT, Claude Pro/Max, Copilot…).

### 4. Implicações para o Rust 🟡
- Ler o `~/.dsh/settings.yaml` existente garante a migração do seu ambiente atual.
- Guardar os segredos no **cofre do SO** (Windows Credential Manager via crate `keyring`) em vez de YAML em texto puro. Isso também resolve o problema do `dsh.cmd` com chaves expostas.

---

## Módulo `preset` — presets de agente e persona

**Pacotes:** `preset/agent-presets`, `preset/persona`. **Complexidade:** média.

### 1. Presets 🟢
- Cada sessão aponta para um preset (explícito ou o `default`). O preset é uma pasta com `preset.yml` (nome e descrição) e `agent.cordis.yml` (composição de *agent plane*: ferramentas, seções de prompt, skills, persona).
- Sessões do mesmo preset compartilham uma composição instalada. **Subagentes herdam a composição do pai.**
- **Fontes:** presets embutidos (`standard`, `ptc`, `cordis`, `minimal`), `roots[]` (`path` e `trust: system|user`) e `~/.dsh/.agent-presets` (user). Um preset que não carrega aparece na lista com o motivo.
- Config: `default` (obrigatório), `roots`, `includeShippedRoot` (true), `includeUserRoot` (true).
- O preset `standard` define a persona "You are a coding agent powered by the {{model}} model." com o sufixo "Your working directory is {{cwd}}.".

### 2. Persona 🟢
`prefix` (obrigatório), `suffix` (''), `complete` (false), `includeRuntimeContext` (true). Sobrepõe `deployment:persona-prefix/suffix` só para o escopo do agente.

### 3. Lacuna de compatibilidade 🟢/🟡
- 🟢 **Não há suporte a `.claude/agents/*.md`** (definições de subagentes do Claude Code com frontmatter `name`, `description`, `tools`, `model`). O conceito equivalente é o preset (YAML cordis), um formato diferente.
- 🟡 O harness Rust deve converter cada `.claude/agents/<nome>.md` num "preset leve": persona = corpo do markdown, toolFilter = `tools`, modelo = `model`, exposto como `subagent_type` na ferramenta de delegação. É o requisito central de reaproveitamento do `D:/Projetos/Totvs AGP/VSCode`.

---

## Módulo `extensions + bundle + boot` — composição cordis, perfis e pacotes dinâmicos

**Pacotes:** `boot/app-boot`, `boot/cmdline`, `bundle/{base,web-app,headless,acp-app,sdk-app,sdk-minimal}`, `extensions/{tool-cordis,cordis-host-runner,cordis-client-runner,ui-cordis}`. **Complexidade:** alta.

### 1. Boot (`app-boot`) 🟢
- Entrada mínima: `installFailLoud('dsh')` e depois `boot('dsh', resolveConfigPath(argv[2], DSH_SNAPSHOT))`. Em modo replay sobe o `cordis.snapshot.yml` irmão para reproduzir uma sessão gravada.
- **Camadas de ambiente (`.env`):** ambiente herdado > `.env` do diretório de invocação > `.env` do `DSH_HOME`.
  - Variáveis que decidem o arranque (`PATH`, `DSH_*`, `XDG_*`…) são **rejeitadas** dos arquivos.
  - Proxies (`HTTP(S)_PROXY`, `ALL_PROXY`, `NO_PROXY`) só são aceitos do `.env` do home, nunca do diretório (que pode vir de um clone).
- **Camadas de composição:** bundles do perfil (em ordem de dependência), depois o `cordis.patch.yml` do perfil, depois `$DSH_HOME/cordis.patch.yml` (maior precedência).
  - Um patch pode substituir a config inteira de uma entrada, inserir entradas ou usar expressões `!!js`.
  - Um patch que não casa com nenhuma entrada gera aviso. Um arquivo vazio ou só com comentários **falha o boot**: para desligar, use `[]`.
- **Entradas obrigatórias:** `agent-loop`, `webserver`, `modules`, `connection`, `headless-runner`, `acp`, `sdk-jsonrpc-server`.
  - Falha de entrada obrigatória no arranque: descarta tudo e sai com código ≠ 0.
  - Falha de entrada opcional: aviso rotulado, e o restante continua.
  - Em HMR posterior não há auditoria de obrigatórias. Uma edição inválida é rejeitada e a instância anterior é mantida.
- **`patchReload`:** `live` observa os patches (padrão em perfis custom e no `web`). `startup` não instala watchers (demais perfis embutidos).
- **Preview:** imprime a configuração efetiva como YAML carregável, com comentários dizendo qual arquivo e qual patch alterou cada entrada.

### 2. Linha de comando (`cmdline`) 🟢
- `ctx.cmdlineArgs`: snapshot imutável dos argumentos internos. Exemplo: `dsh --profile tui --resume abc` → `['--resume','abc']`.
- `ctx.appExit(code)` pede saída pelo caminho de shutdown. `ctx.appReady` sinaliza que o arranque deu certo.
- `exitOnStdinEnd`: EOF do stdin → `appExit(0)` (apps stdio).
- Cada app traz o próprio parser commander. **A flag vence o valor de config** escrito ao lado do `!!js` (exemplo: `port: !!js ctx.webStartup.port ?? 3080`). A resolução acontece uma vez, no arranque.

### 3. Perfis e bundles 🟢
| Bundle | Entradas | Papel |
|---|---|---|
| `base` | 87 | Núcleo: llm, session, agent, ferramentas, sandbox, skill, subagent, workflow, compaction, web, telemetria |
| `web-app` | 96 | UI no navegador + webserver + cordis host/client runner + agent-presets |
| `headless` | 4 | Tarefa única pela linha de comando (`headless-runner`) |
| `acp-app` | 4 | Agent Client Protocol via stdio (`acp`) |
| `sdk-app` | 4 | JSON-RPC stdio (`sdk-jsonrpc-server`) |
| `sdk-minimal` | 32 | Árvore completa e autônoma, sem base nem web |

- Perfis ficam em `$DSH_HOME/profiles/<nome>`. `dsh --profile X --from-default-profile <template>` cria um perfil custom, e `dsh plugin --profile X add <pkg>` instala bundles externos.
- 🟢 **Ausentes do `base`:** `hooks` (bridge Claude Code) e `mcp-client` **não vêm montados por padrão**. Só aparece `mcp-resources`. Para usar hooks ou servidores MCP no dsh, é preciso inserir as entradas via patch. 🟡 No harness Rust, ambos devem vir **ligados por padrão**, porque são essenciais para a paridade com o Claude e para o REA e o GBrain.
- Também ficam desligados por padrão: `hmr`, `skill-badge`, `tool-ralph`. `bash-sandbox`/`pwsh-sandbox` se alternam por plataforma (`process.platform`).

### 4. Saída de dados para serviços externos no `base` 🟢 (relevante para privacidade)
| Entrada | Destino | Padrão |
|---|---|---|
| `session-log-deepseek` | API DeepSeek (`dsh_session_log`) | **ligado** |
| `plugin-package-inventory-deepseek` | API DeepSeek (`dsh_plugin_packages`: lista de plugins ativos) | **ligado** |
| `session-telemetry-otel` | `https://harness-telemetry.deepseeksvc.com/v1/logs` (OTLP gzip) | `FEEDBACK_ONLY` |
| `web-search-deepseek` | API DeepSeek (busca web) | ligado (`DEEPSEEK_API_KEY`) |

- Em `FEEDBACK_ONLY`, um feedback explícito do usuário (nota, texto, rating) libera o **prefixo completo do log da sessão**: mensagens, argumentos e resultados de ferramentas, system prompt, `cwd`. Cada exportação leva um user id anônimo (`$DSH_HOME/.anonymous-user-id`).
- Para desligar: `DSH_TELEMETRY_DISABLED` com qualquer valor não vazio.
- 🟡 **Recomendação para o Rust:** nenhuma telemetria nem upload de log. Se existir, deve ser opt-in explícito.

### 5. Pacotes cordis dinâmicos (`extensions`) 🟢
- Sete ferramentas: `cordis_inspect_self`, `cordis_inspect_list`, `cordis_inspect_query`, `cordis_define`, `cordis_run`, `cordis_stop`, `cordis_undefine`. Com elas o agente **inspeciona e estende o runtime vivo** sem editar arquivos.
- Definições ficam só em memória, com versões imutáveis, e somem ao reiniciar.
- A metade host roda em sandbox `node:vm` (`vmTimeoutMs`). A metade browser é JS puro (sem JSX nem imports), recebe `React`, `console`, `styles` e `host`, e sua execução exige **aprovação humana** na página.
- Só existe no perfil `web-app`.
- 🟡 **Para o Rust:** recurso específico da arquitetura cordis/JS. Equivalente possível: plugins WASM (wasmtime) ou scripts no runtime embutido do workflow (QuickJS). **Prioridade baixa**: não há equivalente no Claude Code.

### 6. Implicações para o Rust 🟡
- Trocar a composição cordis por um **registro estático de módulos com feature flags** em TOML/YAML: `[modules] hooks=true, mcp=true, telemetry=false…`.
- Manter o conceito de **perfil** (`tui`, `headless`, `acp`, `sdk`) como subcomandos ou flags do binário único.
- Manter as **camadas de config** (padrão embutido < `~/.<harness>/config` < projeto < flag de CLI) e as regras de segurança do `.env` (sem variáveis de arranque, proxies só do home).
- Saber ler o `$DSH_HOME/cordis.patch.yml` não é necessário. Basta um migrador opcional de `settings.yaml`.

---

## Módulo `acp` — servidor Agent Client Protocol (automação)

**Pacotes:** `acp/acp` (servidor), cliente em `subagent/subagent-acp`. **Complexidade:** média. Arquivos: `src/{index,session,codec,content,mcp,model-control,updates}.ts`.

### 1. Papel 🟢
Servidor **ACP v1** via stdio (`dsh --profile acp`), voltado a automação: subagentes fora do processo, test runners e controladores. O stdout carrega **só tráfego de protocolo**. Várias sessões independentes por conexão.

### 2. Superfície do protocolo 🟢
| Chamada | Comportamento |
|---|---|
| `initialize` | ACP v1 + `session/list`, `session/resume`, `session/close`, MCP Streamable HTTP. Imagens só se houver attachment store durável e rota com suporte a imagem |
| `authenticate` | Sucesso imediato (sem autenticação) |
| `session/new` | Agente persistente. Workspace absoluto e servidores MCP (stdio/HTTP) validados **antes** da publicação. Falha de conexão MCP faz rollback |
| `session/list` | Sessões raiz persistidas, mais novas primeiro, páginas de `sessionListPageSize` (100), filtro opcional por `cwd` |
| `session/resume` | Restaura o log sem reenviar updates antigos, verificando o workspace canônico |
| `session/close` | Cancelamento quiescente, drenagem de updates, descarte dos descendentes, flush da persistência |
| `session/set_config_option` | `model` / `reasoning_effort`, serializado, devolve o estado completo |
| `session/prompt` | Texto, resource links e imagens. Um prompt por vez por sessão. A seleção de modelo fica **fixa durante o turno**, e uma mudança concorrente vale para o próximo |
| `session/cancel`, `$/cancel_request` | Cancela o prompt. Sem prompt ativo, cancela o trabalho autônomo. Id desconhecido = no-op |
| `session/update` | Mensagens e pensamentos do assistente, ciclo de vida de ferramentas, mudanças de config, uso de contexto |
| `session/request_permission` | Permitir ou rejeitar uma vez (o cliente pode responder automaticamente) |

### 3. Fora do escopo 🟢
`session/load` (replay), deleção, fork, diretórios adicionais, MCP via SSE ou transporte ACP, modes, commands, plans, terminals, operações de filesystem do cliente, elicitation, resources/prompts MCP.

### 4. Confiança 🟢
O cliente ACP é um **controlador confiável**: entradas MCP stdio autorizam os comandos e o ambiente informados, e entradas HTTP autorizam URL e headers.

### 5. Implicações para o Rust 🟡
- Crate `agent-client-protocol` (oficial, Rust). Implementar **servidor** e **cliente**:
  - servidor, para o harness ser usado por editores compatíveis com ACP (Zed, JetBrains, extensões VS Code);
  - cliente, para delegar a outros agentes ACP (por exemplo Gemini CLI, Codex, o próprio Claude Code via adaptador) como subagentes.
- Ampliar a superfície em relação ao dsh: `session/load`, modes (incluindo plan), commands (slash commands e skills) e terminals. São recursos que o Claude Code expõe aos editores.

---

## Módulo `jobs + schedule` — trabalho em background e lembretes duráveis

**Pacotes:** `jobs/jobs` (contrato), `jobs/jobs-local`, `jobs/tool-jobs`, `schedule/schedule`. **Complexidade:** média.

### 1. Jobs 🟢
- Contrato `ctx.jobs`: cada job tem id estável `<kind>-N` (exemplos: comando, PTY, subagente). A **posse é da sessão de agente**, e outros agentes não enxergam nem matam o job.
- `jobs-local` mantém os registros em memória e entrega snapshots (nunca estado vivo). Limite por dono: **10** jobs rodando ou parando. Os jobs **morrem com o processo**.
- Ferramentas:
  - `job_output(job_id, wait?, timeout_ms?)`: jobs de stream devolvem só a saída desde a última leitura, e jobs de saída final devolvem o resultado ao terminar. A resposta sempre termina com `[status: …]`. Um timeout de espera devolve `[status: running]` e mantém o job vivo.
  - `job_list`, `job_kill` (só conclui depois que o trabalho parou).
- **Entrega de conclusão:** aviso dentro da sessão, sem polling.
  - `wakeup` (padrão) abre um turno num dono ocioso, até `maxConsecutiveWakes` (3) seguidos. Uma mensagem do usuário zera esse orçamento.
  - Dono ocupado, `quiet` ou orçamento esgotado: o aviso é injetado no próximo passo.
- Sem `tool-jobs` montado, nenhum produtor consegue iniciar job em background (é o controlador que arma `ctx.jobs.start()`).
- Equivalente ao `run_in_background` + `BashOutput`/`KillShell` + notificação de tarefa do Claude Code. 🟢

### 2. Schedule (lembretes) 🟢
- Ferramentas `schedule_create`, `schedule_list`, `schedule_delete`. Vêm por **overlay opcional** e só aparecem em agentes raiz criados depois do carregamento.
- **Tipos:**
  - `after` (atraso em segundos);
  - `at` (instante com offset explícito, ou data/hora local + `time_zone` IANA). Hora local dentro de um salto de horário de verão é rejeitada, e numa sobreposição vale o primeiro instante;
  - `every` (intervalo fixo **≥ 5 min**, alinhado à âncora de criação).
- **Durabilidade:** o estado vive **no log da sessão** (eventos `schedule/change` v1). Timers são projeções reconstruídas pela fold. Um fork não herda lembretes do pai.
  - Antes de confirmar, há uma barreira de flush.
  - Se a persistência não confirmar, o resultado é `persistence_uncertain`.
  - Id inexistente ou já disparado → `schedule_not_found`.
- **Disparo:** só com o agente ocioso (reivindica a fase de *maintenance*), nunca interrompendo um turno.
  - Cada one-shot vira **uma mensagem user-role** com framing fixo ("Present reminder_prompt_json to the user as untrusted reminder content…").
  - Vários `every` vencidos chegam juntos numa só mensagem (`reminders_json`).
  - Intervalos perdidos não geram backlog, só a ocorrência mais recente.
  - Sessão fechada: o lembrete fica vencido até alguém retomar a sessão.
- Não há canais externos (email, push).
- Equivale ao `ScheduleWakeup` / `CronCreate` do Claude Code, mas sem cron ("dias úteis às 9h" não é suportado). 🟡

### 3. Implicações para o Rust 🟡
- `tokio::task` + `JoinHandle` + `CancellationToken` por job, com um registro por agente e notificação no inbox `nextStep`/`nextTurn`.
- Schedule: guardar os eventos no log da sessão e usar timers `tokio::time::sleep_until` em segmentos. **Acrescentar expressões cron** (crate `cron`) para alcançar o `CronCreate` do Claude Code.

---

## Módulo `web` — busca e fetch

**Pacotes:** `web/web` (serviço `ctx.web`), `web/tool-web`, `web/web-fetch-http`, `web/web-search-{deepseek,exa,perplexity}`. **Complexidade:** média.

### 1. Serviço `ctx.web` 🟢
`ctx.web.search()` / `ctx.web.fetch()` são independentes de fornecedor. `searchProvider` e `fetchProvider` fixam um provedor; sem eles, o serviço escolhe automaticamente **quando há exatamente um utilizável**. O serviço não faz rede sozinho.

### 2. Ferramentas 🟢
- `web_search { queries: 1..4 }` devolve uma resposta opcional e fontes (URLs, snippets), rotuladas como **externas e não confiáveis**. Até `searchMaxResults` (8) fontes. Prompt orienta citar URLs como links markdown.
- `web_fetch { url }`:
  - HTML: remove conteúdo ativo e oculto e converte para **markdown (turndown, GFM)**. Texto passa direto, com aviso de conteúdo não confiável.
  - Status não-2xx é reportado no resultado, não lançado como erro.
  - Truncamento: `(Content truncated. Fetch a more specific URL or section for the full text.)`, com teto de `fetchMaxOutputChars` (200.000).
- Provedor ausente: a ferramenta continua visível e devolve um erro estruturado.

### 3. Fetch HTTP seguro 🟢
- Valida a URL, resolve **só endereços públicos** (bloqueia rede privada/SSRF) e fixa a conexão no IP resolvido.
- Redirects **apenas de mesma origem** (`maxRedirects` 5). Não envia credenciais.
- `maxResponseBytes` 5 MB, `maxBodyChars` 100.000, `timeoutMs` 30 s, `User-Agent` `deepseek-harness/…`. Rejeita binários e tipos não suportados.

### 4. Provedores de busca 🟢
| Provedor | Característica |
|---|---|
| `web-search-deepseek` (padrão no base) | Busca nativa via modelo: **custa um turno inteiro** em tokens e latência. Exige `DEEPSEEK_API_KEY` |
| `web-search-exa` | Keyword ou neural, sem resposta gerada. Descarta resultados sem highlight |
| `web-search-perplexity` | Resposta gerada + fontes. Corta as fontes depois da resposta |

### 5. Implicações para o Rust 🟡
- `reqwest` (rustls) + resolvedor com filtro de IP público (`hickory-resolver`) + `html2md`/`htmd` para markdown. Manter os rótulos de conteúdo não confiável.
- Provedores de busca plugáveis. Sugestão de padrão **sem custo de modelo**: SearXNG self-hosted, Brave Search API (plano gratuito), Tavily, Exa ou Perplexity. DeepSeek fica opcional.
- Equivale ao `WebSearch`/`WebFetch` do Claude Code. O `WebFetch` do Claude também resume a página com um prompt. 🟡 O harness pode oferecer o parâmetro `prompt` opcional usando um modelo gratuito barato.

---

## Módulo `lsp + terminal` — navegação semântica e terminal persistente

**Pacotes:** `lsp/{lsp,lsp-stdio,tool-lsp}`, `terminal/{terminal,terminal-bash,tool-terminal}`. **Complexidade:** média.
🟢 **Nenhum deles vem no bundle `base`**: as ferramentas `lsp` e `terminal_*` só existem se forem inseridas por patch. O `web-app` monta apenas o terminal da UI (`terminal-controller`, `ui-sidebar-terminal`).

### 1. LSP 🟢
- Ferramenta única, somente leitura: `lsp { operation, file_path, line, character }`.
  - Operações: `goToDefinition`, `findReferences` (sempre inclui a declaração), `goToImplementation`, `hover`.
  - Coordenadas UTF-16 começando em 1.
  - Resultado: `path:line:character` agrupado por arquivo, limitado a `maxLocations` (100) e `maxResultChars` (16.000). `timeoutMs` 60 s.
- **Excluídos de propósito:** JSON-RPC genérico, rename, formatting, **diagnostics** e listas de símbolos.
- `lsp-stdio`:
  - Configuração explícita por servidor: `command` (sem shell), `extensionToLanguage` (`{'.ts':'typescript'}`), `args`, `initializationOptions`, `configuration`.
  - Um servidor por workspace, iniciado sob demanda. Cada consulta relê o arquivo (sem estado de documento). Consultas serializadas por servidor+workspace.
  - **Ambiente limpo de credenciais:** variáveis com `KEY`/`PASSWORD`/`SECRET`/`TOKEN` e todas as `DSH_*` não são repassadas.
  - Não instala servidores nem aplica sandbox.
- Prompt orienta preferir `search`/`read` e usar `lsp` só quando a busca textual for ambígua.

### 2. Terminal persistente 🟢
- `ctx.terminals` emite ids opacos, com sessões **restritas ao agente que as criou**. Backends registráveis. Sessões locais ao processo.
- Ferramentas:
  - `terminal_open`: abre a sessão e devolve id, nome, tipo, pid, status e a saída inicial.
  - `terminal_send`: escreve texto (Enter opcional) e espera a prontidão, ou devolve um job id com `run_in_background`.
  - `terminal_read`: página da saída retida.
  - `terminal_signal`: sinal permitido ao grupo de processos em primeiro plano.
  - `terminal_close`: espera o fim da árvore de processos.
  - `terminal_list`.
- `terminal-bash`: shell interativo sob o sandbox; `shellDialect` bash ou pwsh (Windows); orientado a linhas (TUIs full-screen ficam fora). `maxReadBytes` 256 KiB, `timeoutMs` 30 s, `disposeGraceMs` 3 s.
- `maxResultBytes` 256 KiB por resultado. O prompt orienta preferir ferramentas one-shot para trabalho limitado.

### 3. Implicações para o Rust 🟡
- LSP: cliente `lsp-types` + `tower-lsp`/`async-lsp` em modo cliente. Acrescentar **diagnostics** depois de editar: o Claude Code reporta diagnósticos do IDE, e isso aumenta a qualidade.
  - Pré-configurar rust-analyzer, typescript-language-server, pyright e gopls, e detectar se estão instalados.
  - 🟡 Para o Totvs AGP (TLPP/AdvPL), verificar se há um servidor LSP disponível. 🔴 LACUNA: confirmar com o usuário quais linguagens predominam no projeto.
- Terminal: `portable-pty` (ConPTY no Windows) com detecção de prompt. Ligar por padrão.

---

## Módulo `spill + storage + session-query` — saídas grandes, estado de app e busca no histórico

**Pacotes:** `spill/{spill,spill-local,spill-policy}`, `storage/{storage,storage-domain,storage-json,storage-sqlite}`, `session-query/{session-query,session-query-sqlite,tool-session-query,session-log-export}`. **Complexidade:** média.

### 1. Spill 🟢
- `ctx.spillStore.save(text)` devolve `{ locator, bytes, guidance }`. Não há API de leitura, busca nem retenção: o modelo lê o arquivo com `read`/`grep`.
- `spill-policy`:
  - Um resultado de texto maior que `maxInlineBytes` (**50.000 no base**) vira um preview head/tail + localizador + instrução de recuperação.
  - Se o spill falhar, o resultado original continua visível. Sem `maxInlineBytes`, a política fica desligada.
  - O mesmo limite vale para cópias de log de subchamadas do `run_code`.
- `spill-local`:
  - Arquivos privados (0700) por sessão, com nomes imprevisíveis e proteção contra symlink plantado.
  - `root` é um diretório temporário privado por padrão.
  - Uma limpeza no arranque apaga arquivos com mais de `cleanupPeriodDays` (30) dias.
- Equivale ao comportamento do Claude Code de gravar saídas longas de Bash/MCP em arquivo e devolver um preview. 🟡

### 2. Storage (estado de app fora do log) 🟢
- `ctx.storageDomain` abre domínios chave-valor validados por schema. Leitura síncrona do estado em memória. Cada escrita fica durável antes de resolver e emite `domain/changed` em ordem.
- **Backends:**
  - `json`: layout `single` (um `<unit>.json`) ou `per-record`, com publicação atômica;
  - `sqlite`: um banco, um documento JSON por linha, indicado para escritas frequentes.
- Invisível ao modelo. Usado em workspaces, sidecars de sessão etc.

### 3. Busca no histórico de sessões 🟢
- `session-query`: listar, filtrar, ler e rastrear sessões e eventos. Prefere a sessão viva à cópia persistida e devolve clones destacados.
- `session-query-sqlite`:
  - Índice derivado **FTS5** separado (não toca na persistência), paginação por cursor, `snippetChars` 240.
  - 🟢 **No base:** `path: ':memory:'` e `openAt: never`, ou seja, **busca full-text desligada**.
- `tool-session-query` (opt-in), cinco ferramentas read-only:
  - `session_search`: sessões que casam com a consulta, ranqueadas (exclui a sessão atual);
  - `session_event_search`;
  - `session_trace`;
  - `session_event_trace`;
  - `session_event_read`.
  - 🟢 Autorização entre sessões só quando o **`cwd` da sessão alvo é idêntico ao do chamador**. Limites: `maxSearchResults` 100, timeout 30 s.
- `session-log-export`: ZIP da árvore da sessão (subsessões e anexos) pela web ou `/export`.

### 4. Implicações para o Rust 🟡
- Spill: ligado por padrão (50 KB), arquivos em `~/.<harness>/spill/<sessão>/`.
- Storage: um `rusqlite` único (WAL) para estado de app.
- Session-query: `rusqlite` com FTS5 (feature `bundled`), **ligado por padrão**, com as ferramentas de busca expostas. Isso complementa o GBrain: o GBrain guarda **memória curada**, e o FTS dá acesso ao **histórico bruto**. Manter a regra de autorização por `cwd`.

---

## Módulo `attachment + ptc-runtime` — anexos e execução de programas do modelo

**Pacotes:** `attachment/{attachment,attachment-local}`, `ptc-runtime/{ptc-runtime,ptc-runtime-node}`, `experimental/ptc-runtime-python`. **Complexidade:** média.

### 1. Anexos 🟢
- Imagens (PNG, JPEG, WebP, GIF) e arquivos genéricos anexados a prompts e comandos. Eles **sobrevivem ao restart** da sessão.
- **Imagens:**
  - Validadas e normalizadas antes de a mensagem ser aceita, com cache por rota.
  - Limites: 20 MiB por imagem, 20 por mensagem, 200 MiB por mensagem, 64 Mpx, 8192 px por lado.
  - Normalização: alvo de 2048×2048 px, lado máximo de 8192 e ~4 MiB, por escada de qualidade. Concorrência 2.
- **Arquivos genéricos:** guardados byte a byte, sem limite. O modelo **lê sob demanda** por um caminho read-only salvo, em vez de receber os bytes.
- **Armazenamento:** abaixo de `DSH_HOME`, endereçado por conteúdo (bytes idênticos são guardados uma vez), com verificação de tamanho e conteúdo na leitura. **Nunca apagado automaticamente.**
- Eventos de sessão não carregam caminhos do navegador, URLs de provedor, caminhos locais nem base64.
- Áudio e vídeo não têm tratamento dedicado.

### 2. PTC runtime (programmatic tool calling) 🟢
- O modelo escreve **um programa** que chama funções do host (as ferramentas) como chamadas async comuns. Só a saída impressa e o valor de retorno voltam ao contexto, o que economiza tokens em cadeias de chamadas.
- É a base do modo `ptc`/`both` do pipeline de ferramentas (`run_code`) e do **workflow**.
- `ptc-runtime-node` (padrão no base):
  - **TypeScript** num processo Node novo a cada execução, **sob o mesmo sandbox do Bash**. Um modo restrito falha se o backend de sandbox não estiver disponível.
  - Resultado: JSON sem perda, logs por canal ou erro estruturado.
  - Limites: `timeoutMs` 120 s (inclui esperas por ferramentas e aprovações), `maxTimeoutMs` 600 s, `maxOutputBytes` 64 MiB, heap V8 de 512 MiB, `maxPendingCalls` 128, `graceMs` 3 s.
- `ptc-runtime-python` (experimental): CPython 3.10+, **sem sandbox de filesystem**, desligado em todos os perfis.

### 3. Implicações para o Rust 🟡
- Anexos: crate `image` para validar e redimensionar, store por hash (`blake3`/`sha2`) em `~/.<harness>/attachments/`. Leitura de PDF (o Claude lê PDFs) via `pdf-extract` / `lopdf`, ou envio nativo ao provedor quando suportado.
- PTC/workflow: o binário único **não deve depender do Node**. Opções:
  - **QuickJS embutido** (`rquickjs`), para compatibilidade com scripts JS de workflow;
  - **Deno core** (`deno_core`), que é mais pesado mas suporta TS;
  - executar no Node do sistema quando ele existir (já é exigido pelo REA).
  - Recomendação: `rquickjs` + bindings async para as ferramentas, rodando sob o sandbox de processo.

---

## Módulo `typert + api + sdk` — reflexão de tipos, RPC host↔cliente e SDK

**Pacotes:** `typert/{generator,loader,protocol,registry}`, `api/{gateway,remotes,session-controller,settings-controller,terminal-controller,workspace-controller,workspace-files}`, `sdk/{protocol,server,client}`, `python/{sdk,sdk-runtime}`. **Complexidade:** alta (infraestrutura).

### 1. Typert 🟢
- O gerador, em tempo de **build**, transforma tipos TypeScript públicos em fábricas de schema Zod e reflexão.
- `@Remote` / `@RemoteScope` marcam métodos do host expostos a clientes remotos.
- O registry guarda reflexão, schemas preguiçosos e descritores de invocação, com registro atômico preso ao ciclo de vida do plugin.
- 🟡 **Para o Rust:** é desnecessário. `serde` + `schemars` geram os JSON Schemas das ferramentas a partir de structs, e uma API web (se houver) usa `axum` + `utoipa`.

### 2. API (BFF da UI web) 🟢
- **Gateway** RPC bidirecional (`ctx.typertGateway` no host, `ctx.remote` no cliente), com correlação de requisições e streams multiplexados.
- **Controllers:**
  - `session`: ciclo de vida, histórico, catálogo de modelos, skills invocáveis pelo usuário, referências de arquivo;
  - `settings`/`credentials`: devolvem metadados **redigidos**, nunca segredos;
  - `workspace`: criar, renomear, ordenar, arquivar;
  - `workspace-files`: preview read-only, que **pode ler fora do workspace**;
  - `terminal`: shell do usuário na sidebar, fora do transcript do agente.
- Específico do perfil `web-app`.

### 3. SDK (JSON-RPC 2.0 por stdio, NDJSON) 🟢
| Direção | Método | Payload |
|---|---|---|
| cliente→servidor | `initialize` | InitializeParams → InitializeResult |
| cliente→servidor | `session/prompt` | SessionPromptParams → recibo durável de enfileiramento |
| cliente→servidor | `shutdown` | → `{}` (descarta o runtime e sai com 0) |
| servidor→cliente | `session.event` | todos os eventos de todas as sessões, sem filtro |
| servidor→cliente | `session.status` | transição `running`/`idle` |
| servidor→cliente | `subagent.started` / `subagent.finished` | ciclo de vida de subagente |

- Uma sessão por `sessionId`. Clientes: `dsh-sdk-client` (TS: `DeepSeekHarness`, `HarnessClient`) e o SDK Python, que espelha os tipos.
- 🟢 Equivale ao modo `claude -p --output-format stream-json` e ao Claude Agent SDK.

### 4. Implicações para o Rust 🟡
- Oferecer um **modo headless/SDK** compatível com a interface do Claude Code: `<bin> -p "<prompt>" --output-format text|json|stream-json`, mais um servidor JSON-RPC stdio. Assim scripts e extensões que hoje chamam `claude -p` podem apontar para o harness Rust como fallback.
- A UI web é opcional e de baixa prioridade. O foco é TUI (`ratatui`) + ACP (editores) + headless.

---

## Módulo `vendor/cordis + util` — framework de plugins e bibliotecas utilitárias

**Diretórios:** `vendor/{cordis,loader,include,group,timer,hmr,logger-console,cosmokit,schemastery}`, `packages/util/*`. **Complexidade:** alta (fundação).

### 1. Cordis (vendored) 🟢
- Framework TypeScript de plugins com DI explícita:
  - `new Context()` é o container raiz;
  - `ctx.plugin()` inicia um plugin e devolve uma `Fiber`;
  - `inject: [...]` declara os serviços exigidos;
  - efeitos, listeners e serviços são removidos quando a fiber dona é descartada.
- **Versões fixadas:** `cordis` 4.0.0-rc.7, `loader` 1.0.0-rc.5, `include` 1.0.4, `group` 1.0.0, `timer` 1.1.2, `hmr` 1.0.15, `cosmokit` 1.8.1, `schemastery` 3.18.0. Todos MIT, renomeados para o escopo `@deepseek-ai`.
- **Modificações locais** (registro exaustivo em `vendor/README.md`):
  - endurecimento do ciclo de vida da fiber (lacunas de descarte reentrante);
  - JSDoc;
  - refresh resiliente do `include`, que reaplica patches;
  - remoção do i18n do HMR.
- **Loader:** monta a árvore a partir do YAML (`cordis.yml`), com `!!js`, grupos, `disabled`, `inject` e mutações **não transacionais** (uma falha de ativação pode deixar a árvore parcialmente aplicada).

### 2. Bibliotecas `util` 🟢
| Lib | Função |
|---|---|
| `home-paths` | Raiz de dados: caminho explícito > `$DSH_HOME` > `~/.dsh`. Valores vazios são ignorados |
| `launch-environment` | Snapshot imutável do ambiente em ordem de confiança fixa (herdado > `./.env` > home `.env`), com camadas excluíveis em buscas sensíveis. Ainda copia tudo para `process.env` (não isola subprocessos) |
| `http-proxy` | Uma política de proxy de saída para o `fetch` (LLM, busca web, MCP HTTP). Loopback direto. Prepara o ambiente dos filhos |
| `native-command` | Executa binários **sem shell**, cancelável, esconde consoles transitórios no Windows. Abre caminhos no desktop (inclusive tradução WSL) |
| `atomic-write` | Substituição atômica (inode novo, sem seguir symlink do temporário), lock entre processos para read-modify-write. **Sem fsync** |
| `output-retention` | `ItemRetainer` (cabeça + contagem omitida), `TextRetainer` (head/tail sem cortar UTF-8), `formatRetentionNotice` |
| `package-manifest` | Tipos de manifesto de perfil e bundle |
| `crypto`, `deque`, `chunked-list`, `time`, `timeout`, `values`, `workspace-path`, `lazy-require`, `brand` | Primitivas de apoio |

### 3. Implicações para o Rust 🟡
- **Não portar o Cordis.** O equivalente idiomático em Rust:
  - **traits** (`Tool`, `LlmProvider`, `SessionStore`, `SandboxBackend`, `Hook`…) registradas num `Registry` montado no arranque;
  - `Arc<dyn Trait>` para DI;
  - `tokio_util::sync::CancellationToken` + `Drop`/`JoinSet` para o ciclo de vida (o papel da fiber);
  - um `tokio::sync::broadcast` como barramento de eventos.
- Extensões dinâmicas ficam por **MCP** (servidores externos), **skills/plugins no formato Claude Code** e, opcionalmente, scripts QuickJS. Isso cobre a extensibilidade sem DI em runtime.
- Mapear as utils:
  - `home-paths` → `directories`/`dirs`;
  - `atomic-write` → `tempfile::NamedTempFile::persist` + `fs4` (lock), **com** fsync;
  - `http-proxy` → proxy nativo do `reqwest` (`HTTPS_PROXY`);
  - `native-command` → `std::process::Command` + `CREATE_NO_WINDOW`;
  - `output-retention` → módulo próprio.

---

## Módulo `apps/cli` — o launcher `dsh`

**Arquivos:** `apps/cli/src/{bin,args,profile-boot,plugin,dump-config,process-shutdown}.ts`, `sdk-source.cordis.patch.yml`, `config/examples/*`, `reference/README.md`. **Complexidade:** média.

### 1. Comandos 🟢
| Comando | Efeito |
|---|---|
| `dsh --profile <nome>` | Sobe `$DSH_HOME/profiles/<nome>` |
| `dsh --profile <nome> --from-default-profile <template>` | Cria um perfil custom a partir de um template embutido |
| `dsh --profile acp` | Servidor ACP stdio |
| `dsh --profile headless "tarefa"` | Uma sessão persistida nova, imprime a resposta final e sai |
| `dsh --profile sdk` / `sdk-minimal` | Servidor JSON-RPC stdio |
| `dsh web` | Alias de `--profile web` |
| `dsh plugin --profile <nome> <args pnpm>` | Gerencia plugins do perfil via pnpm |

- Flags do launcher: `--profile`, `--from-default-profile`, `--patch <path>` (repetível), `--dump-config`, `--dump-default-config`, `-h`.
- O resto vai **intacto** para o app. Exemplo: `--port` pertence ao web, `--resume` ao tui.
- O diretório de invocação é a raiz padrão do workspace.
- Perfis embutidos são inicializados automaticamente no primeiro uso. `desktop` é reservado ao Electron.
- 🟢 **Não há perfil TUI embutido.** O README diz "assuming the tui profile is installed". Hoje a interface interativa padrão é o **navegador** (`dsh web`). 🟡 O usuário roda via `dsh.cmd` (web).

### 2. Camadas e encerramento 🟢
- Composição sobre uma raiz vazia: bundles (ordem de `dsh.profile.bundles`) → `cordis.patch.yml` do perfil → `$DSH_HOME/cordis.patch.yml` → cada `--patch`, na ordem do argv.
  - A camada posterior vence por linha.
  - Um patch **substitui a `config` inteira** da linha (sem deep-merge).
- Bundles são resolvidos primeiro da instalação e depois do `node_modules` do perfil.
- **Encerramento:**
  - até 5 s de drenagem graciosa;
  - `SIGTERM` sai com 0, `SIGINT` com 130;
  - um segundo sinal força a saída;
  - Ctrl+C durante um descarte travado escala na hora.

### 3. Overlays opcionais (`config/examples/`) 🟢
- `github-review` (webhook de revisão), `schedule`, `cordis` (ferramentas runtime).
- **`mcp-memory`**: `engram`, `memorix` 1.3.0, `mcp-reference-memory`.
  - Exemplo: entrada `@deepseek-ai/dsh-mcp-client` com `serverName: memorix`, `transport: stdio`, `command: memorix`, `args: [serve]`.
  - **O DSH não instala o executável** ("does not run a package manager").
  - 🟢 Isso confirma o padrão adotado para o GBrain e o REA: servidor de memória via MCP. A diferença no harness Rust é que **ele mesmo instala e atualiza**.

### 4. Implicações para o Rust 🟡
Um binário único com subcomandos:
- `<bin>`: TUI interativa (padrão, o equivalente ao `claude`);
- `<bin> -p "<prompt>"`: headless, com `--output-format`;
- `<bin> acp`: servidor ACP;
- `<bin> serve`: SDK JSON-RPC;
- `<bin> models`: lista modelos gratuitos e de assinatura;
- `<bin> login <provedor>`: OAuth das assinaturas;
- `<bin> mcp add|list|remove`;
- `<bin> plugin install <marketplace>`;
- `<bin> doctor`: verifica Node, Bun, REA, GBrain e servidores LSP;
- `<bin> --resume <id>` / `--continue`.

Manter os sinais (`SIGINT` 130) e a drenagem graciosa de 5 s.

---

## Fonte externa `morluto/rea` — engenharia reversa via MCP

**Origem:** https://github.com/morluto/rea (MIT; ferramentas de terceiros mantêm as próprias licenças). Pacote npm **`rea-agents`**. **Decisão:** `rea-integration` (aprovada).

### 1. Visão geral 🟢
- Servidor MCP + CLI de engenharia reversa **local**. Alvos: binários nativos, JS/Electron, .NET, APK Android, sites, firmware, pcap, WASM e outros.
- Cada conclusão vem com as evidências e as limitações.
- Clientes citados: Claude Code, Codex, Cursor, Gemini CLI, Grok Build, ou qualquer cliente MCP por registro manual.

### 2. Instalação e ciclo de vida 🟢
| Ação | Comando |
|---|---|
| Setup em agentes | `npx rea-agents setup` (registra o MCP e instala a skill, com backup) |
| CLI global | `npm install --global rea-agents` → `rea --help` |
| Bun | `bunx rea-agents@latest` (atualização global exige Bun ≥ 1.2.17) |
| Atualizar | `rea update` (só a instalação global dona do CLI). Para revisar o registro: `rea setup --client <id> --dry-run` |
| Remover | `rea uninstall [--purge-data]` |

- **Runtime:** Node 22.x (≥ 22.23.4), 24.x (≥ 24.22.1) ou ≥ 26.12.1.
- **Registro Claude Code:** servidor `rea` em `~/.claude.json` → `mcpServers.rea`. Equivalente manual: `<node> <rea>/scripts/rea.mjs mcp`.
- **Skill:** `reverse-engineer-anything`, instalada em `~/.claude/skills/` (ou `$CLAUDE_CONFIG_DIR/skills`, ou `~/.agents/skills`). Skill avulsa: `npx skills add morluto/rea --skill reverse-engineer-anything`.
- **Plugin:** o pacote npm contém `.claude-plugin/plugin.json` e `.codex-plugin/plugin.json`, mais `server.json` (MCP registry) e `AGENTS.md`.

### 3. Superfície MCP (parcial; o catálogo gerado fica em `docs/mcp-contracts.md`) 🟢
| Domínio | Ferramentas (exemplos) |
|---|---|
| Sessão e evidências | binary_session, open_binary, close_binary, reset_investigation, get_evidence_bundle, export_evidence_bundle, inspect_analysis_view, record_unknown / update_unknown / list_unknowns, search_guidance, read_guidance_section |
| Nativo (Ghidra/Hopper/IDA/Kuna) | binary_overview, analyze_function, compare_functions, get_call_graph, trace_call_path, decompile_all, list/search_procedures, list/search_strings, read_bytes, list_segments |
| JS/aplicações | analyze_javascript_application, trace_application_feature, compare_application_versions, compare_source_to_bundle |
| Web e captura | analyze_web_bundle, capture_browser_scenario, capture_process_scenario |
| Artefatos | inspect_artifact, extract_artifact, compare_artifacts, inspect_macho, inspect_wasm_artifact |
| **.NET** | compare_managed_members, verify_managed_native_boundaries, import_managed_reconstruction, project_managed_application_graph |
| Mobile | project_android/apple/harmony_application_graph, inspect_apk_signature, inspect_adb_client |

### 4. Contratos relevantes para o harness 🟢
- **Evidência:** `{ evidence_id, normalized_result, raw_result|null, analysis_profile }`, guardada num ledger de sessão e referenciável por id. `detail: complete|summary`, com paginação via `inspect_analysis_view`.
- **Saída:** `structuredContent` conforme o `outputSchema` + projeção em texto. Erros: `isError: true` + `{error, recovery?}`.
- **Orçamento stdio de 10 MiB**: o excedente vai para evidência ou export.
- **Anotações:** `readOnlyHint` / `destructiveHint` (dicas, não autorização). `_meta.provenance.untrusted_content: true` em conteúdo vindo do alvo.
- **Execução síncrona.** Notificações de progresso (≤ 1 a cada 100 ms) quando o cliente manda progress token. Cancelamento tipado com `partial_observation`. Deadline do provedor (exemplo: Ghidra 330.000 ms) separado do timeout do cliente.
- **Variáveis:** `REA_MCP_INPUT_SCHEMA_PROFILE` (`full`), `REA_MCP_OUTPUT_SCHEMA_PROFILE` (`compact|off`, para clientes que limitam o `tools/list`), `REA_GHIDRA_STARTUP_TIMEOUT_MS`, `REA_HOPPER_STARTUP_TIMEOUT_MS`, `REA_JEB_MCP_URL`.
- **Dependências por alvo** (instaladas pelo usuário): Ghidra/Hopper/IDA, JDK + JADX/Apktool/adb, WABT, TShark, Binwalk/Unblob, navegador Chrome, Docker (AgentEnv). Python não é necessário.

### 5. Requisitos para o harness Rust 🟡
1. **Cliente MCP completo:** `structuredContent`/`outputSchema`, anotações, `_meta`, **progress notifications** (exibir na TUI), **cancelamento** (`notifications/cancelled`) e timeout por servidor **longo e configurável** (≥ 330 s para Ghidra).
2. **Gerenciador `rea`:**
   - `doctor` verifica a versão do Node;
   - instala via `npm i -g rea-agents` e atualiza via `rea update` (agendável);
   - registra `rea` → `node <prefix>/rea-agents/scripts/rea.mjs mcp`.
3. **Respeitar `untrusted_content`:** encapsular o conteúdo como dado não confiável no prompt.
4. **Importar a skill** `reverse-engineer-anything` de `~/.claude/skills` (o mesmo diretório do Claude Code) para não duplicar.
5. Exibir `destructiveHint` na aprovação de ferramentas (pedir confirmação quando for `true`).
6. 🟡 Relevante para o Totvs AGP: as ferramentas **.NET** ajudam a analisar binários C# legados.

---

## Fonte externa `coreos/ai-helpers` — marketplace de plugins Claude Code

**Origem:** https://github.com/coreos/ai-helpers (**Apache-2.0**). **Decisão:** `claude-plugin-format` (aprovada).

### 1. Estrutura do repositório 🟢
```
.claude-plugin/marketplace.json     # catálogo do marketplace
plugins/<nome>/.claude-plugin/plugin.json
plugins/<nome>/commands/<comando>.md
plugins/<nome>/README.md
plugins/utils/                      # comandos sem plugin-pai claro
agents/                             # subagentes
skills/<nome>/SKILL.md              # frontmatter YAML
```

### 2. `marketplace.json` 🟢
```json
{ "name": "ai-helpers", "owner": { "name": "coreos" },
  "plugins": [ { "name": "pipeline-status", "source": "./plugins/pipeline-status",
                 "description": "Summarize the status of the RHCOS pipeline using Slack" }, … ] }
```
Plugins: `pipeline-status`, `customer-cases`, `rhcos-cve-analyzer`, `jenkins-analyzer` (comando `/analyze-failures` + agente `failure-analyzer`). O conteúdo é específico de RHCOS/OpenShift. **O valor para o harness está no formato**, não nos plugins em si.

### 3. Fluxo de instalação (Claude Code) 🟢
`/plugin marketplace add coreos/ai-helpers` → `/plugin install pipeline-status@ai-helpers` → `/pipeline-status`. Para Cursor: clone + symlink em `~/.cursor/commands`.

### 4. Formato de plugin Claude Code (referência para a implementação) 🟡
*(campos documentados pela Anthropic; o repositório só mostra parte deles)*
- **`plugin.json`:**
  - `name` (obrigatório), `version`, `description`, `author`, `homepage`, `repository`, `license`, `keywords`;
  - caminhos opcionais `commands`, `agents`, `hooks`, `mcpServers`.
- **Comandos** `commands/*.md`: frontmatter `description`, `argument-hint`, `allowed-tools`, `model`. Corpo com `$ARGUMENTS` / `$1…`, `!`cmd`` (saída de shell) e `@arquivo`. Namespace `/<plugin>:<comando>`.
- **Agentes** `agents/*.md`: frontmatter `name`, `description`, `tools`, `model`. Corpo = system prompt.
- **Skills** `skills/<nome>/SKILL.md`: `name`, `description` + arquivos de apoio.
- **Hooks** `hooks/hooks.json`: eventos PreToolUse, PostToolUse, UserPromptSubmit, Stop, SubagentStop, SessionStart, SessionEnd, PreCompact, Notification. Tipo `command`, com `matcher` pelos nomes de ferramenta do Claude (`Bash`, `Edit|Write`…).
- **MCP** `.mcp.json` / `mcpServers`.
- Variável `${CLAUDE_PLUGIN_ROOT}` nos caminhos.
- **Instalação:** marketplaces registrados em `~/.claude/plugins/known_marketplaces.json`, plugins em `~/.claude/plugins/` e habilitação em `settings.json` → `enabledPlugins`.

### 5. Requisitos para o harness Rust 🟡
1. **Ler os plugins já instalados no Claude Code** (`~/.claude/plugins/` + `enabledPlugins`) **e** instalar novos (`<bin> plugin marketplace add <github|path>`, `<bin> plugin install <p>@<mkt>`).
2. Carregar comandos como slash commands (com `$ARGUMENTS`, `!cmd` e `@arquivo`), agentes como `subagent_type`, skills, hooks e MCP de cada plugin.
3. **Aliases de nomes de ferramenta:** o harness Rust deve usar **os mesmos nomes do Claude Code** (Read, Write, Edit, Glob, Grep, Bash, WebFetch, WebSearch, Task/Agent, TodoWrite, Skill…). Assim matchers de hooks e `allowed-tools` funcionam sem tradução. Essa é a correção da lacuna encontrada no dsh, que usa nomes em minúsculas.
4. Mesma regra de conteúdo do ai-helpers: nada que identifique pessoas reais nos plugins gerados.

---

## Fonte externa — ferramentas Rust do CoreOS (padrões de engenharia)

**Origem:** coreos-installer, afterburn, zincati, cargo-vendor-filterer (Apache-2.0). Analisado `coreos-installer/Cargo.toml`. Os outros são citados como referência de padrão. 🟡

### 1. Padrões observados 🟢
- Edition 2021, `resolver = 3`, **MSRV explícito** (`rust-version = "1.87.0"`).
- Divisão em **biblioteca + binários finos** (`libcoreinst` + `coreos-installer`, `rdcore` por feature).
- `clap` 4 (derive, sem default-features), `clap_mangen` por feature `docgen` (gera man pages).
- Erros: `anyhow` (aplicação) + `thiserror` 2 (biblioteca).
- `serde`/`serde_json`/`serde_yaml`/`serde_with`. Dependências com faixas de versão amplas (`>=x,<y`) para empacotamento em distros.
- `reqwest` **blocking**, sem tokio: escolha ligada ao domínio. 🟡 Para o harness, que precisa de streaming e paralelismo, use **tokio + reqwest async**.
- Release: commits e tags assinados, hook de pré-release (`make docs`), sem publicação automática.
- **`cargo-vendor-filterer`**: metadado `[package.metadata.vendor-filter]` (alvos `*-unknown-linux-gnu`, tier 2, todas as features). Gera um diretório `vendor/` só com os crates das plataformas-alvo, para builds offline e reprodutíveis.

### 2. Requisitos para o harness Rust 🟡
1. Workspace Cargo: `harness-core` (lib) + crates por domínio (`harness-llm`, `harness-mcp`, `harness-tools`, `harness-sandbox`, `harness-tui`, `harness-acp`) + binário fino `harness`.
2. MSRV fixado, `anyhow` + `thiserror`, clap derive + `clap_mangen`/`clap_complete` (man pages e autocompletar do PowerShell).
3. **Build offline** com `cargo vendor-filterer --platform=x86_64-pc-windows-msvc --platform=x86_64-unknown-linux-gnu`, útil na rede corporativa.
4. `[profile.release]`: `lto = "thin"`, `strip = true`, `codegen-units = 1`. Binário único estático (`+crt-static` no MSVC).
5. Autoatualização no estilo zincati, simplificada: checar releases e trocar o binário com verificação de assinatura (crate `self_update` + `minisign`).

---

## Fonte externa `garrytan/gbrain` — memória persistente para agentes

**Origem:** https://github.com/garrytan/gbrain (MIT). **Não está no npm** (o pacote `gbrain` do npm é outro projeto). **Decisão:** `gbrain-integration` (aprovada).

### 1. Arquitetura 🟢
- Interface `BrainEngine` (`src/core/engine.ts`, mais de 140 métodos), da qual são gerados o CLI e o servidor MCP.
- **Engines:**
  - **PGLite** (Postgres 17 em WASM): padrão, sem configuração, até ~50 mil páginas;
  - **Postgres + pgvector**: compartilhado ou multi-máquina.
- **Recuperação híbrida:** vetor + keyword + RRF + boost por tier de fonte + reranker, com caminhada em largura no grafo.
- **Síntese:** `think` produz respostas citadas e aponta lacunas.
- **Embeddings:** Voyage (`voyage-4`, padrão), OpenAI, OpenRouter, Gemini, Azure, **Ollama, llama.cpp, LM Studio** (locais e gratuitos), Together, LiteLLM etc. Reranker: Voyage `rerank-3-lite` ou Qwen3-Reranker local.
- **Modo sem chave:** `gbrain init --pglite --no-embedding` (só keyword).

### 2. Modelo de memória 🟢
- **Fatos com fonte**, que podem ser corrigidos ou retirados (`takes_add`, `takes_update`, `takes_supersede`). Versões consultáveis (`get_versions`, `known_as_of`, `delta`).
- **Grafo** com arestas tipadas. Escritas locais confiáveis criam links automaticamente, sem LLM. Arestas extraídas contam como evidência, não como prova.
- **Markdown é a fonte da verdade** dos arquivos (`people/`, `meetings/`, `customers/`, `notes/`…). Um delete no git vira soft-delete. Páginas que só existem no banco, revisões e estado operacional exigem **backup do banco** à parte.
- Preferências e fatos duráveis são compartilháveis entre agentes. O estado transitório da tarefa fica local.

### 3. CLI 🟢
- **Setup e instalação:** `gbrain setup <claude-code|codex> [--dry-run|--remove|--no-instructions|--instructions-only]`, `gbrain init`, `gbrain doctor`.
- **Dados:** `gbrain import <dir> [--no-embed]`, `gbrain sources`.
- **Consulta:** `gbrain search`, `gbrain think`, `gbrain whoknows`.
- **Servidor:** `gbrain serve --surface starter|full --advertise verbs`, `gbrain mcp expose`, `gbrain connect <url> --token`.
- **Operação:** `apply-migrations`, `engine status --probe`, `db-repair`, `jobs submit`, `agent run`, `creds`, `config set`, `skillopt`, `onboard`.

### 4. Ferramentas MCP (parcial; referência completa em `docs/mcp/TOOL_REFERENCE.md` ou no recurso `gbrain://capabilities`) 🟢
| Grupo | Ferramentas |
|---|---|
| Leitura | search, query, recall, get_page, assemble_evidence, think, context_pack, search_exact, get_chunks, get_timeline |
| Escrita | remember, put_page, edit_page, add_timeline_entry, add_tag/remove_tag/get_tags |
| Fatos (takes) | takes_add, takes_update, takes_supersede, takes_list, takes_search |
| Cronologia | chronicle_day, chronicle_since, chronicle_on_this_day |
| Ontologia | ontology_propose, ontology_get, ontology_conflicts |
| Versões | get_versions, known_as_of, delta, get_write_request |

- Superfícies: `starter` (verbos de memória, a recomendada) e `full` (todas).
- Há guias de integração para Claude Code, Codex, OpenCode, Grok, Hermes, ChatGPT e Perplexity (`docs/mcp/*.md`).

### 5. Configuração 🟢
- **As 7 camadas valem para a resolução de schema-packs:** flag por chamada > variável de ambiente > chave por fonte no banco > chave do brain no banco > `gbrain.yml` (raiz do repo) > `~/.gbrain/config.json` > padrão `gbrain-base`.
- `gbrain setup` grava um **bloco "brain-first"** no arquivo de instruções do agente e configura hooks. 🔴 LACUNA: nomes e eventos dos hooks não estão documentados no README. É preciso rodar `gbrain setup claude-code --dry-run` para ver.

### 6. Memorable (opcional) 🟢
Guarda procedimentos reproduzíveis de sessões de código concluídas. **O relay envia traces redigidos e o texto das consultas para fora da máquina**, depois de divulgação e consentimento explícitos. → **Desligado no harness Rust** (alinhado à decisão `no-outbound-deepseek`: nenhum dado sai sem opt-in).

### 7. Requisitos para o harness Rust 🟡
1. **Gerenciador `gbrain`:**
   - `doctor` verifica Bun ≥ 1.4.4 e oferece instalar o Bun;
   - instala via `bun install -g github:garrytan/gbrain#latest-stable` e atualiza com o mesmo comando (agendável);
   - na primeira vez roda `gbrain init --pglite`.
   - Embeddings: **Ollama local** quando disponível (gratuito, privado), senão `--no-embedding`. Voyage/OpenAI só com chave do usuário.
2. **Registro MCP:** `gbrain serve --surface starter --advertise verbs` (stdio), pré-registrado como `gbrain`.
3. **Instruções brain-first** injetadas como seção do system prompt (equivalente ao bloco que o `gbrain setup` grava no CLAUDE.md): consultar a memória antes de responder e registrar fatos duráveis com fonte.
4. **Compartilhar o mesmo brain com o Claude Code:** se o usuário também rodar `gbrain setup claude-code`, os dois agentes usam a mesma memória. Isso garante a continuidade quando o Claude estiver indisponível. 🟢 Essa é a peça central do objetivo de fallback.
5. Backup: agendar um export Markdown e um dump do PGLite em `~/.<harness>/backups/`.
6. Integrar com o FTS do histórico de sessões (módulo `session-query`): GBrain = memória curada, FTS = histórico bruto.

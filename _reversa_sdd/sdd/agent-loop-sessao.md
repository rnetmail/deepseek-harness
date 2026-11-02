# Loop do agente e sessão (`harness-core` + `harness-store`)

## Visão Geral
O núcleo do harness conduz um LLM num loop ReAct: recebe uma mensagem, pede uma resposta ao modelo, executa as ferramentas pedidas e repete até o modelo parar. Tudo o que acontece fica num **log de eventos append-only** (a sessão). O histórico enviado ao modelo, a transcrição, as estatísticas e o resume após uma queda são todos derivados desse log. 🟢

O contrato é portado do dsh (`core/agent`, `core/agent-loop`, `core/session`, `session-*`) para Rust, com duas mudanças: traits estáticas em vez de DI cordis 🟡 e nenhum envio do log a terceiros (ADR 101) 🟢. O harness Rust **é o novo dsh** (decisão `dsh-is-harness`): um **executável Rust único** com o mesmo nome e a mesma pasta `~/.dsh`. O dsh TS é aposentado e não roda em paralelo. As sessões já existentes em `~/.dsh/sessions` continuam abríveis, porque o formato v3 é mantido. 🟢

## Responsabilidades
- Manter uma inbox durável por agente, com duas filas (`next_turn`, `next_step`), e acordar o driver quando chega trabalho. 🟢
- Executar turnos e passos (steps) com checkpoints de durabilidade antes de qualquer efeito externo. 🟢
- Despachar tool calls em paralelo até um limite, respeitando ferramentas exclusivas como barreira de ordem. 🟢
- Gravar cada request ao modelo de forma que ele possa ser **reconstruído byte a byte** a partir do log, dos blobs endereçados por conteúdo e da versão do binário. 🟢
- Cancelar turnos de forma explícita e cooperativa, fechando pares de ferramentas pendentes. 🟢
- Persistir a sessão (JSONL comprimido com zstd), retomar depois de uma queda e reparar logs órfãos. 🟢
- Expor pontos de extensão (middleware) para retry, compactação, hooks, permissões, plan mode e goal. 🟢
- Montar o system prompt a partir de seções ordenadas. 🟢

## Interface

### Agente (`AgentHandle`)
| Operação | Entrada | Efeito | Confiança |
|---|---|---|---|
| `create` | `session_id`, `{provider, model, reasoning_effort?, max_tokens?}`, `parent?` | Cria o agente e grava o header da sessão | 🟢 |
| `resume` | `session_id` | Abre a sessão em modo escrita exclusivo; se o processo caiu no meio de um turno, anexa os fechamentos sintéticos | 🟢 |
| `followup(msg)` | mensagem | Enfileira em `next_turn` e acorda o driver | 🟢 |
| `steer(msg)` | mensagem | Enfileira em `next_step` e acorda o driver | 🟢 |
| `inject(ctx)` | contexto | Adiciona em `next_step` **sem acordar** o driver | 🟢 |
| `cancel(cause, keep_inbox=false)` | `user` \| `parent` \| … | Aborta o turno; por padrão limpa as filas | 🟢 |
| `when_idle()` | — | Future que resolve quando o agente fica ocioso | 🟢 |
| `dispose()` | — | Só quem tem o handle pode chamar (o handle funciona como capability) | 🟢 |

Toda mutação da inbox gera `agent/inbox/spliced` com as coordenadas da mudança. Itens removidos por cancelamento levam `outcome: "canceled"`. 🟢

### Configuração do loop
| Campo | Tipo | Padrão | Regra | Confiança |
|---|---|---|---|---|
| `max_parallel_tool_calls` | inteiro ≥ 1 | 10 | `1` = execução serial; um valor inválido é erro de configuração | 🟢 |
| `agents[]` | lista | — | `id`, `provider`, `model`, `reasoning_effort`, `max_tokens`, `cwd`, `session_id` \| `resume_session_id` | 🟢 |
| id de sessão novo | string | `${agent_id}-session-<uuid>` | — | 🟢 |

### Eventos de sessão (enum serde com tag `type`)
| Grupo | Tipos | Confiança |
|---|---|---|
| Estrutura | `turn/start`, `turn/end`, `step/start`, `step/end`, `session/end-seed` | 🟢 |
| Superfície (exigem `surface_op`) | `system/message`, `user/message`, `assistant/message`, `tool/result` | 🟢 |
| Requisição | `request/header` (só quando muda), `request/context` (só quando há diferença), `assistant/attempt` (tentativa falha ou cancelada) | 🟢 |
| Ferramenta | `tool/call`, `tool/result` | 🟢 |
| Extensões | `agent/inbox/spliced`, `llm/retry`, `image/offload`, eventos de compactação, `schedule/*`, `goal/*`… | 🟢 |

- `surface_op` = `append` ou `{op:"replace", start_seq, end_seq}` (intervalo inclusivo). A compactação é um `replace`. 🟢
- Motivos de `turn/end`: `completed`, `aborted{reason}`, `blocked`, `error{LlmFailure}`, `interrupted`. 🟢
- Um tipo de evento desconhecido é recusado, a menos que o envelope o marque como `ignorable`. Eventos ignoráveis vindos de outra implementação são preservados. 🟢

### Pontos de extensão (traits de middleware)
| Ponto | Pode | Usado por | Confiança |
|---|---|---|---|
| `pre_step` | rejeitar o passo ou substituir as mensagens (`Reject` \| `Enter{messages, starts_request_series}`) | compactação, checkpoint | 🟢 |
| `request` (waterfall) | trocar provider/model e ajustar o request | fallback de provedores, plan mode | 🟢 |
| `request_error` | devolver `Retry` | retry com backoff, fallback | 🟢 |
| `turn_stopping` | injetar steering para manter o turno aberto | goal, agent teams | 🟢 |
| `assistant_stream` | só observar frames ao vivo | TUI, ACP, SDK | 🟢 |
| `status` | só observar `idle` \| `running` | superfícies | 🟢 |

### Persistência (`SessionStore`)
`create(header)`, `open(id, Write|Read)`, `stat(id)`, `list()`, `flush()`. Um handle tem `read(offset?, len?)`, `append`, `flush` e `close`. Erros: `AlreadyOwned`, `AlreadyExists`, `ReadOnly`, `HandleClosed`, `OwnershipLost`, `Corruption`. 🟢

Layout em disco 🟢 (mesma raiz do dsh atual, `~/.dsh/sessions`):
```
<root>/--<cwd-normalizado>--/<id-escapado>/session.v3.jsonl.zstd   (ou session.v3.jsonl)
```
- A primeira linha é o header físico, e a versão no header bate com o nome do arquivo.
- Quando há mais de uma geração, vale o maior `N`.
- O id é escapado de forma injetiva, sem traversal nem colisão.

## Regras de Negócio
- **R1. Visível ao modelo ⟺ referenciado de forma durável.** Nada entra num request sem estar no log ou num blob endereçado por hash. 🟢
- **R2.** O append no log em memória é síncrono. A escrita em disco é em lote, com drain obrigatório no `flush`. 🟢
- **R3.** A `assistant/message` é gravada **antes** do despacho das ferramentas. 🟢
- **R4. Três barreiras de durabilidade, todas fail-closed:**
  1. antes do stream do modelo;
  2. antes do corpo de cada tool call de topo (chamadas aninhadas reutilizam o checkpoint do pai);
  3. em cada `pre_step`. 🟢
- **R5.** Um par `tool/call` ↔ `tool/result` nunca é separado, nem por compactação, nem por fronteira de passo. 🟢
- **R6.** `seq` é contíguo, passos só existem dentro de turnos e todo `tool/call` tem um `tool/result`. Isso é validado no append. 🟢
- **R7.** Um JSON não representável sem perda é rejeitado no append: BigInt, ciclos, arrays esparsos, `-0`. 🟢
- **R8.** Um `tool/result` com `data.error` exige `content[0].is_error == true`. 🟢
- **R9.** Falhas do adapter viram `finish` e passam por `request_error`. Falhas de plugin ou ferramenta encerram o **turno**, não o loop. 🟢
- **R10.** Tool calls não despachadas depois de um cancelamento recebem um resultado sintético `ABORTED_BEFORE_DISPATCH`. Um cancelamento durante um flush pendente também gera `ABORTED_BEFORE_DISPATCH`. 🟢
- **R11.** O turno reivindica atomicamente tudo de `next_step` mais **um** item de `next_turn`. Entre passos, reivindica só `next_step`. 🟢
- **R12.** Um único writer por sessão por processo. Uma segunda abertura em escrita falha com `AlreadyOwned`. 🟢
- **R13.** `fork(source, boundary?)` só aceita um prefixo que termine fora de um turno aberto. O filho recebe `is_seeded` e `inherited_event_count`. 🟢
- **R14.** O system prompt é a concatenação das seções por `order` crescente, com empate decidido pelo nome. Uma seção `complete` substitui tudo, e mais de uma seção `complete` é erro. Uma variável `{{x}}` desconhecida faz a montagem falhar. 🟢
- **R15.** O opener de identidade do dsh ("…powered by DeepSeek Harness") é **substituído** por uma identidade neutra do dsh, sem citar a DeepSeek. 🟡
- **R16.** O contexto de runtime (hora, cwd) entra como snapshot user-role no histórico, não como texto do system prompt, para preservar o cache do provedor. 🟢
- **R17.** Numa série `in-history`, um system prompt alterado é anexado (não reescrito), preservando o prefix cache. 🟢
- **R18.** O log **nunca** é anexado a requests de provedores nem enviado a serviço externo. Não há equivalente a `session-log-deepseek` nem a telemetria OTel. 🟢 (decisão `no-outbound-deepseek`)
- **R19.** O par provider/model pode mudar a cada passo, mas fica fixo dentro de um turno quando a superfície é ACP. 🟢
- **R20.** Um formato de sessão mais novo que o binário é recusado com orientação para atualizar. Formatos antigos passam por migração encadeada. 🟢
- **R21. Formato compartilhado:** o dsh Rust lê e grava `session.v3` no mesmo layout do dsh TS. Os tipos de evento novos do Rust (`llm/fallback`, `policy/*`, `catalog/*`, `quota/*`) são gravados com o envelope marcado `ignorable`, que o dsh TS já preserva sem recusar (R5 do domínio). Uma mudança incompatível exige um v4 com migração explícita. 🟡
- **R21b. Migração na primeira execução:** o dsh Rust detecta um `~/.dsh` do dsh TS e (a) indexa as sessões v3 no FTS; (b) importa a seção `llm-pi-ai` do `settings.yaml` para `providers.toml`; (c) move as chaves de `.credentials.yaml` para o cofre do SO e só remove o YAML com confirmação do usuário; (d) ignora `cordis.patch.yml` e `DSH_TELEMETRY_DISABLED`, que deixam de ter efeito porque o executável Rust não tem envio de dados. Nada é apagado sem confirmação. 🟡
- **R22. Trava de escrita entre processos** (duas instâncias do dsh Rust, ou um dsh TS esquecido aberto): a trava de escrita usa o **mesmo** objeto de kernel do dsh TS. No Windows é um semáforo nomeado `Local\dsh-session-lock-<sha256(caminho absoluto em minúsculas de session.lock)>`, com contagem 1. No POSIX é `flock` não bloqueante em `session.lock`, com verificação de inode. Manter o mesmo nome do dsh TS custa nada e protege o histórico durante a migração; a convivência não é um cenário de uso. A instância que chegar depois recebe `AlreadyOwned`. Leitores nunca tocam na trava, e o kernel libera a trava quando o processo morre. 🟢

## Fluxo Principal
1. Uma superfície (TUI, `-p`, ACP, SDK) chama `followup(msg)`. O driver acorda. 🟢
2. O driver abre um turno: grava `turn/start` e reivindica a inbox (R11). 🟢
3. **Passo:**
   1. Monta o system prompt, as ferramentas visíveis (máscara do agente) e o snapshot de runtime.
   2. Roda `pre_step`; aqui a compactação pode substituir mensagens.
   3. Checkpoint 3: flush.
   4. Grava `step/start`.
   5. Roda o waterfall `request`, onde o provedor pode ser trocado por fallback ou política.
   6. Grava `request/header` e `request/context` quando houver mudança.
   7. Checkpoint 1: flush.
   8. Faz o stream do modelo e emite frames em `assistant_stream`.
   9. Grava `assistant/message`. 🟢
4. Se `finish == tool_calls`, cada chamada passa pelo pipeline de ferramentas (spec [ferramentas-claude-code](ferramentas-claude-code.md)), com checkpoint 2 antes do corpo. As chamadas paralelizáveis rodam juntas sob um semáforo de `max_parallel_tool_calls`, e as exclusivas rodam sozinhas. 🟢
5. Grava `tool/call` e `tool/result` e volta ao passo 3. 🟢
6. Quando `finish != tool_calls`, roda `turn_stopping`. Se ninguém fizer steering, grava `step/end` e `turn/end{completed}`, faz flush e muda o status para `idle`. 🟢

## Fluxos Alternativos
- **Erro de LLM:** `request_error` decide `Retry` (backoff de 5 tentativas, 500 ms a 10 s, jitter de 10%) ou passa para o próximo modelo da cadeia de fallback (spec [provedores-modelos](provedores-modelos.md)). Se esgotar, grava `turn/end{error}`. 🟢/🟡
- **Cancelamento pelo usuário (Esc / `session/cancel`):** aborta o stream (grava `assistant/attempt`), sinaliza `cancel` às ferramentas em execução, fecha as pendentes com `ABORTED_BEFORE_DISPATCH` e grava `turn/end{aborted}`. 🟢
- **Queda do processo no meio de um turno:** no `resume`, o leitor repara o log (descarta o sufixo malformado não confirmado) e o loop anexa `tool/result` de erro sintéticos, `step/end` e `turn/end{interrupted}`. 🟢
- **Linha malformada seguida de um `turn/end` válido:** erro fatal de corrupção. A sessão não é aberta em escrita. 🟢
- **Falha de escrita em background:** os eventos ficam em memória, o caminho automático pausa e o erro é reportado no próximo `flush`, que falha fechado. 🟢
- **Ferramenta `run_in_background`:** o resultado imediato é o id do job, e a conclusão chega mais tarde como mensagem em `next_turn` (spec [jobs-schedule-terminal](jobs-schedule-terminal.md)). 🟢

## Dependências
- `harness-llm`: stream do modelo, `LlmFailure`, preparação do request.
- `harness-tools`: pipeline, schemas e máscaras por agente.
- `harness-store`: JSONL zstd, blobs endereçados por conteúdo, índice FTS derivado.
- `harness-claude-compat`: seções de prompt vindas de `CLAUDE.md`, de skills e da memória.
- Compactação (spec [compactacao-contexto](compactacao-contexto.md)): middleware `pre_step`.

## Requisitos Não Funcionais
| Tipo | Requisito inferido | Evidência no código | Confiança |
|---|---|---|---|
| Desempenho | Até 10 ferramentas paralelas por passo | `packages/core/agent-loop/src/index.ts:190` | 🟢 |
| Disponibilidade | Retry de 5 tentativas com backoff de 500 ms a 10 s e jitter de 10% | `packages/llm/llm-retry` | 🟢 |
| Durabilidade | Três barreiras de flush fail-closed | `packages/session/session-checkpoint-policy/src/index.ts` | 🟢 |
| Durabilidade | Publicação atômica no Windows (`MoveFileExW` com write-through) e frames zstd com checksum | `packages/session/session-persistence-jsonl/src/win32.ts`, `zstd.ts` | 🟢 |
| Durabilidade | `fsync` depois da publicação atômica (corrige a dívida 9 do dsh) | `architecture.md` §4 | 🟡 |
| Privacidade | Nenhum upload do log | decisão `no-outbound-deepseek` | 🟢 |

> Inferido a partir do código. Validar com a equipe de operações.

## Critérios de Aceitação
```gherkin
Dado um agente ocioso com uma sessão nova
Quando o usuário envia "liste os arquivos" e o modelo pede Glob e depois responde em texto
Então o log contém turn/start, step/start, request/header, assistant/message, tool/call, tool/result, assistant/message, step/end e turn/end{completed} com seq contíguo
E reconstruir o request do 2º passo a partir do log produz os mesmos bytes enviados ao provedor

Dado um turno com 3 tool calls paralelizáveis em execução
Quando o usuário cancela
Então as chamadas recebem o sinal de cancelamento, as não despachadas recebem ABORTED_BEFORE_DISPATCH
E o log termina em turn/end{aborted} sem nenhum tool/call órfão

Dado um processo encerrado à força depois de tool/call e antes de tool/result
Quando a sessão é retomada
Então o log ganha tool/result de erro sintético, step/end e turn/end{interrupted}, e o agente aceita um novo prompt

Dado uma sessão já aberta em escrita
Quando outra instância tenta abri-la em escrita
Então a abertura falha com AlreadyOwned e nada é gravado

Dado uma sessão aberta em escrita pelo dsh TS
Quando o dsh Rust tenta abri-la em escrita
Então recebe AlreadyOwned, porque as duas versões usam o mesmo semáforo nomeado
E consegue abri-la em leitura e mostrar a transcrição

Dado qualquer configuração
Quando um request é enviado a um provedor
Então o corpo não contém o log da sessão nem identificadores de telemetria
```

## Prioridade
| Requisito | MoSCoW | Justificativa |
|---|---|---|
| Loop turno/passo + pipeline de ferramentas | Must | Caminho crítico de todo uso |
| Event log + persistência + resume | Must | Única fonte da verdade (ADR 001) |
| Checkpoints de durabilidade e cancelamento | Must | Evitam a reexecução de efeitos externos |
| Requests reconstruíveis (R1) | Must | Base de cache, auditoria e replay (ADR 002) |
| Pontos de extensão | Must | Usados por retry, compactação, hooks, goal, fallback |
| Fork de sessão | Should | Usado por subagentes e `/rewind` |
| Leitura e gravação v3 em `~/.dsh/sessions` | Must | O executável Rust substitui o dsh e herda o histórico |
| Migração de `~/.dsh` na primeira execução (sessões, settings, credenciais → cofre) | Must | Uso exclusivo do executável Rust |

## Rastreabilidade de Código
| Arquivo | Função / Classe | Cobertura |
|---|---|---|
| `packages/core/agent/src/index.ts`, `dispatch.ts`, `types.ts` | `ctx.agents`, inbox, `AgentHandle` | 🟢 |
| `packages/core/agent-loop/src/index.ts`, `inbox.ts`, `tool-calls.ts` | driver ReAct, claim, concorrência | 🟢 |
| `packages/core/agent-loop/src/runtime-context.ts` | snapshot de runtime | 🟢 |
| `packages/core/session/src/types.ts`, `surface.ts`, `invariant.ts`, `repair.ts` | `SessionEvent`, `surface_op`, invariantes, reparo | 🟢 |
| `packages/core/session/src/request-header.ts`, `preparation.ts` | `request/header`, reconstrução | 🟢 |
| `packages/core/system-prompt/src/index.ts` | seções e variáveis | 🟢 |
| `packages/session/session-format-catalog` | versões e migrações | 🟢 |
| `packages/session/session-persistence-jsonl/src/*` | layout, geração, lease, zstd, win32 | 🟢 |
| `packages/session/session-persistence-jsonl/src/lease.ts`, `win32.ts:141-160` | trava de escrita entre processos (nome do semáforo) | 🟢 |
| `packages/session/session-checkpoint-policy/src/index.ts` | barreiras de durabilidade | 🟢 |
| `packages/llm/llm-retry` | política de retry | 🟢 |

## Atualização upstream 0.2.1-rc.5 (ciclo 1, 2026-11-01)
> Origem: [ledger](../upstream/ledger.md). Estas regras **prevalecem** sobre as anteriores quando houver conflito.

- **U1. Formato v4:** o dsh Rust **grava v4** (`SESSION_FORMAT_VERSION = 4`) e **lê v3** pela migração v3→v4, sem regravar a geração armazenada: o arquivo v3 continua v3 no disco e é convertido em memória. **Substitui a parte de R21 que manda gravar v3.** Como o dsh TS 0.1.6 não lê v4 e só o executável Rust é usado, isso não afeta a operação. 🟢 (`packages/session/session-format-v3-to-v4`)
- **U2. `tool/result` vira uma mensagem de primeira classe** `role: "tool"` com `toolCallId`, `content` direto e `isError` opcional, validados juntos. O wrapper v3 (`role: "user"` + bloco `tool-result`) só é aceito na migração. Campos extras viram `plugin:message:*` e `plugin:result:*`. Substitui R8 (a regra `content[0].is_error`). 🟢
- **U3. Fontes de mensagem pertencem ao produtor:** `source.kind` identifica o produtor (`system-prompt`, `runtime-context`, `compact-checkpoint`, `ptc-mode`…), e o wrapper `kind: "plugin"` é recusado em v4. 🟢
- **U4. `developer/message`:** adições e remoções de ferramentas entre requests são registradas em ordem e referenciam, por `headerSeq`, o `request/header` que contém o schema completo. Isso preserva a reconstrução dos requests (R1) com ferramentas dinâmicas (MCP, skills, gates). 🟢
- **U5. Validação obrigatória das relações entre eventos** na leitura (ordem de turnos e passos, chamadas e resultados, PTC, cadeias de retry, compactação), antes de expor a sessão. Um final inacabado pode deixar abertos um turno, um passo, uma chamada ou uma compactação. 🟢
- **U6. Política de versão:** uma mudança fica na mesma versão quando os leitores antigos retomam corretamente, ignoram com segurança ou recusam antes de executar. Caso contrário, a versão sobe (nota `session-reader-compatibility-review`). 🟢
- **U7. Sessão em branco reutilizável:** uma sessão sem turno pode ser reutilizada ao abrir uma conversa nova no mesmo diretório, adquirindo o writer **antes** de abrir o histórico. 🟢

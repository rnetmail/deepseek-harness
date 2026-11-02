# Subagentes, skills, hooks e orquestração (`harness-core` + `harness-claude-compat` + `harness-script`)

## Visão Geral
Os mecanismos de extensão do comportamento do agente, com semântica Claude Code:
- **Subagentes** (`Task`/`Agent`): delegação para agentes nomeados (`.claude/agents`, plugins) ou genéricos, com isolamento, filtro de ferramentas e modelo próprio. Podem rodar em segundo plano ou ser contínuos. 🟢
- **Skills** (`Skill` e `/nome`): instruções carregadas sob demanda a partir de um catálogo. 🟢
- **Hooks**: comandos disparados em eventos do ciclo de vida, capazes de bloquear, pedir aprovação, injetar contexto e reescrever argumentos. 🟢/🟡
- **Orquestração**:
  - `Workflow`: script JS que coordena muitos subagentes;
  - `Goal`: objetivo de longo prazo com rodadas automáticas;
  - delegação a **agentes externos** (Claude Code, Codex, Gemini CLI via ACP). 🟢

## Responsabilidades
- Criar subagentes isolados (sessão filha), com máscara de ferramentas, alias de modelo, memória e limite de profundidade. 🟢/🟡
- Rodar subagentes em primeiro plano, em segundo plano (job) ou de forma contínua (mensagens e interrupção). 🟢
- Manter o catálogo de skills e carregar o corpo sob demanda. 🟢
- Executar hooks de todas as fontes (settings, plugins, gerenciados) na ordem e com a semântica do Claude Code. 🟡
- Executar scripts de workflow num runtime JS embutido. 🟡
- Conduzir goals com rodadas limitadas e desarmar o goal depois de resume ou fork. 🟢

## Interface

### `Task` / `Agent` 🟢/🟡
| Campo | Tipo | Significado |
|---|---|---|
| `description` | string (3 a 5 palavras) | Rótulo da tarefa |
| `prompt` | string | Instrução completa (o filho não vê o histórico do pai) |
| `subagent_type` | string? | Agente nomeado (`.claude/agents`, `<plugin>:<agente>`, `general-purpose`, `Explore`, `Plan`) |
| `model` | string? | Alias ou id. Sobrepõe o `model` do agente |
| `run_in_background` | bool? | Devolve o id do job. A conclusão chega como mensagem |

Retorno (primeiro plano): o texto final do filho, mais metadados (tokens, duração, ferramentas usadas). Controle de filhos contínuos: `SendMessage`/`send_message`, `interrupt_agent`, `ListAgents`/`list_agents`. 🟢

**Agentes embutidos:** 🟡
| Nome | Ferramentas | Uso |
|---|---|---|
| `general-purpose` | todas as do pai | Tarefas de várias etapas |
| `Explore` | somente leitura (`Read`, `Glob`, `Grep`, `LSP`, `WebFetch`, `WebSearch`) | Buscas amplas |
| `Plan` | somente leitura | Planos de implementação |

**Provedores de execução:** 🟢
| Provedor | Execução |
|---|---|
| `spawn` (padrão) | Sessão filha nova no mesmo processo |
| `fork` | Filha que herda o histórico do pai (prefixo fora de turno aberto) |
| `acp:<agente>` | Agente externo via ACP (Gemini CLI, Codex…) |
| `claude-code` | Sessão do Claude Code (quando disponível) |

### `Skill` 🟢
`Skill { skill, args? }` → bloco `<skill_content>` com o corpo, mais o caminho da pasta para scripts e recursos. `/nome args` faz o mesmo pela entrada do usuário. O catálogo (nome + descrição de até 500 caracteres) entra como mensagem durável antes do primeiro request e é substituído quando o conjunto muda.

### Hooks (`settings.json → hooks`, `hooks/hooks.json` de plugins)
```json
{ "hooks": { "PreToolUse": [ { "matcher": "Bash|PowerShell",
    "hooks": [ { "type": "command", "command": "python .claude/hooks/check.py", "timeout": 60 } ] } ] } }
```
| Evento | Pode | dsh TS | Alvo |
|---|---|---|---|
| `SessionStart` | injetar contexto | 🟢 | ✔ (+ `source`: startup/resume/clear/compact) |
| `UserPromptSubmit` | bloquear, injetar contexto | 🟢 | ✔ |
| `PreToolUse` | bloquear, `ask`, `allow` (não sobrepõe `deny`), **`updatedInput`** | 🟢 (`updatedInput` não aplicado) | ✔ aplica `updatedInput` 🟡 |
| `PostToolUse` | bloquear com feedback, injetar contexto | 🟢 | ✔ |
| `Stop` / `SubagentStop` | forçar mais um passo | 🟢 / só observação | ✔ ambos podem forçar 🟡 |
| `SubagentStart` | injetar contexto | 🟢 | ✔ |
| `PreCompact` | injetar instruções de compactação | ❌ | ✔ 🟡 |
| `Notification` | observar (aprovação pendente, ociosidade) | ❌ | ✔ 🟡 |
| `SessionEnd` | observar (limpeza) | ❌ | ✔ 🟡 |

| Tipo de hook | dsh TS | Alvo |
|---|---|---|
| `command` | 🟢 | ✔ |
| `http`, `prompt`, `agent`, `mcp_tool` | lidos e pulados com aviso | `http` ✔; `prompt` ✔ (avaliado por modelo barato e permitido); `agent`/`mcp_tool` com aviso 🟡 |

Entrada do hook (stdin, JSON): `session_id`, `transcript_path`, `cwd`, `hook_event_name`, `tool_name` (nome Claude Code), `tool_input`, `tool_response` (Post). Variáveis: `CLAUDE_PROJECT_DIR`, `CLAUDE_PLUGIN_ROOT`, `DSH_SESSION_ID`. 🟢/🟡

### `Workflow` 🟢
`Workflow { meta: {name, description, phases?}, script, args? }` → `{ run_id, agents_started, result }`. O script é JS com top-level `await`, `agent(prompt, opts{schema?, label?, phase?})`, `parallel()`, `pipeline()`, `phase()`, e termina em `return <json>`.

### `Goal` 🟢
`create_goal`, `get_goal`, `update_goal` e `/goal`. Fases `active`, `paused`, `blocked`, `complete`, mais a flag *armado*.

## Regras de Negócio

### Subagentes
- **R1.** O filho roda numa sessão própria (`parentSession` no header). Um crash no filho não corrompe o pai. O filho **não** vê o histórico do pai, exceto no provedor `fork`. 🟢
- **R2.** As ferramentas do filho são a interseção entre a máscara do pai, o `tools` do agente e as permissões. As máscaras só restringem. 🟢
- **R3.** A profundidade padrão é **1** (a sessão principal delega e os subagentes não delegam), compatível com o Claude Code e com a regra da Cláudia. `max_depth` é configurável. No dsh TS o **padrão** é 3 (`packages/subagent/tool-subagent/src/index.ts:129`), não o máximo; o padrão 1 é uma escolha do alvo. 🟡 [Revisão]
- **R4.** As chamadas `Task` são *parallel-safe* (`isConcurrencySafe` em `packages/subagent/tool-subagent/src/index.ts`): várias delegações no mesmo passo rodam juntas, até o limite do loop (10). 🟢
- **R5.** O modelo do filho é resolvido pelo `model` da chamada, depois o `model` do agente, depois o modelo do pai, sempre passando pelo mapa de aliases e pela política (spec 2). 🟡
- **R6.** Pedir ao provedor uma capacidade que ele não suporta (por exemplo, `fork` num `acp:`) falha **no início**. 🟢
- **R7.** Um subagente em segundo plano vira o job `agent-N` (spec 16). A conclusão entra no próximo turno do pai como notificação. 🟢
- **R8. Delegação ao Claude Code quando ele estiver disponível** (provedor `claude-code`): usa as settings normais do Claude, remove do ambiente as variáveis que pareçam credenciais e usa `permission_mode` `dontAsk` por padrão. Permite o caminho inverso do fallback: o dsh orquestra e o Claude executa. 🟢/🟡
- **R9.** Os pedidos de aprovação de um filho sobem para a superfície da sessão raiz, identificados pelo nome do agente. 🟢

### Skills
- **R10.** Formato, raízes e precedência estão na spec 4 (R8). Uma skill malformada é descartada cedo, com aviso. Um provedor com falha é pulado e o último catálogo válido é mantido. 🟢
- **R11.** `disable-model-invocation: true` tira a skill do catálogo do modelo, mantendo `/nome`. `user-invocable: false` faz o contrário. 🟢
- **R12.** `allowed-tools` da skill concede `allow` **temporário** às ferramentas listadas enquanto a skill estiver ativa no turno. Nunca sobrepõe um `deny`. 🟡
- **R13.** O corpo da skill é relido a cada carga. As mudanças no disco valem na próxima invocação. 🟢

### Hooks
- **R14.** Exit **2** bloqueia, e o stderr vira o motivo entregue ao modelo (ou ao usuário, no `UserPromptSubmit`). Qualquer outro código diferente de zero é falha não bloqueante: é registrada e a execução segue. 🟢
- **R15.** O stdout em JSON pode trazer `decision`/`permissionDecision` (`allow|deny|ask`), `reason`, `additionalContext`, `updatedInput`, `continue: false`, `stopReason` e `suppressOutput`. 🟢 (leitura) / 🟡 (`updatedInput` e `continue: false` passam a ter efeito)
- **R16.** Os hooks de um mesmo evento rodam **em sequência**, na ordem: gerenciados, usuário, projeto, local, plugins. O matcher é uma regex sobre o nome Claude Code da ferramenta, com aliases aceitos. Matcher vazio ou `*` casa com tudo. 🟢/🟡
- **R17.** O timeout padrão é de 600 s, configurável por hook. O resumo do stderr gravado no log é limitado a 500 caracteres (`hook/invoked`, `hook/result`). 🟢
- **R18.** A configuração é **observada** (watch). Uma mudança vale na próxima ocorrência do evento. No dsh TS ela era lida uma vez só. Uma config ilegível gera aviso e não roda nenhum hook daquela camada. 🟢/🟡
- **R19.** Hooks só rodam em pastas confiáveis (spec 12, R20) e sob o sandbox da sessão. 🟡
- **R20.** O `updatedInput` de um `PreToolUse` é revalidado contra o schema da ferramenta e passa de novo pelas regras de permissão. Um input que viraria `deny` é negado. 🟡

### Workflow e goal
- **R21.** O workflow roda num **QuickJS embutido** (`rquickjs`), sem Node. Os scripts escritos para o dsh TS e para a ferramenta Workflow do Claude Code devem rodar sem alteração (mesma API). 🟢 (API) / 🟡 (runtime)
- **R22.** Limites do workflow: concorrência padrão igual ao número de CPUs, até 1.000 agentes por execução, 4.096 itens por chamada, resultado de até 50.000 caracteres. Uma falha comum de um filho resolve `null`. O turno do pai fica bloqueado até o fim. 🟢
- **R23.** No máximo um goal por sessão, com até 256 rodadas. O driver só enfileira uma rodada com o agente ocioso, o goal ativo e armado. **Depois de resume ou fork, o goal fica desarmado** até uma retomada humana. 🟢
- **R24.** O `ralph` (worker novo por rodada com conclusão autodeclarada) fica **desligado** por padrão, como no dsh. 🟢

## Fluxo Principal (delegação Totvs)
1. O usuário escreve "Cláudia, levante as regras de recálculo do ponto". 🟡
2. A sessão principal, guiada pelo `CLAUDE.md` e pela skill `claudia`, chama em paralelo `Task(subagent_type="analista-totvs-rm", …)` e `Task(subagent_type="arquiteto-totvs-rm", …)`. 🟢/🟡
3. Para cada filho, o dsh resolve a definição, a máscara de ferramentas, o alias `opus` (via política) e a memória. Roda `SubagentStart` e cria a sessão filha. 🟡
4. Os filhos trabalham. `PreToolUse`/`PostToolUse` valem para eles. As aprovações sobem para a sessão raiz. 🟢
5. Ao terminar, o `SubagentStop` roda e o texto final volta como resultado de cada `Task`. 🟢
6. A sessão principal consolida e responde. O `Stop` roda e pode exigir mais um passo. 🟢

## Fluxos Alternativos
- **`subagent_type` desconhecido:** erro com a lista dos tipos disponíveis. 🟡
- **Hook `PreToolUse` com exit 2:** a ferramenta não roda e o modelo recebe o stderr. 🟢
- **Hook `Stop` força continuação em laço:** no máximo 3 continuações consecutivas forçadas por hook em cada turno. Depois disso o turno encerra, com aviso. 🟡
- **Workflow com erro de script:** o resultado é o erro com linha e coluna. Os agentes já iniciados são cancelados. 🟢/🟡
- **Claude Code indisponível no provedor `claude-code`:** a delegação cai para `spawn` com o alias resolvido localmente, com aviso. 🟡

## Dependências
- Loop e sessões filhas (spec 1); ferramentas e permissões (specs 3 e 12); compatibilidade (spec 4); plugins (spec 6); política e modelos (spec 2); jobs (spec 16).
- `rquickjs` (workflow); Claude Agent SDK ou CLI `claude` (provedor `claude-code`); ACP (spec 15).

## Requisitos Não Funcionais
| Tipo | Requisito inferido | Evidência no código | Confiança |
|---|---|---|---|
| Desempenho | Até 10 delegações paralelas por passo; workflow com concorrência igual ao número de CPUs | `packages/core/agent-loop`, `packages/workflow/workflow-ptc` | 🟢 |
| Disponibilidade | Isolamento de falha entre pai e filho | `packages/subagent/subagent` | 🟢 |
| Desempenho | Timeout de hook de 600 s; resumo de stderr de 500 caracteres | `packages/hooks/hooks-claude-code` | 🟢 |
| Segurança | Ambiente sem credenciais para o Claude Code delegado | `packages/subagent/subagent-claude-code` | 🟢 |
| Disponibilidade | Goal desarmado depois de resume ou fork | `packages/goal/goal` | 🟢 |

> Inferido a partir do código. Validar no piloto com os 7 agentes do Totvs.

## Critérios de Aceitação
```gherkin
Dado os 7 agentes do Totvs carregados
Quando a sessão principal chama dois Task em paralelo
Então os dois filhos rodam ao mesmo tempo, cada um com suas ferramentas e sua memória
E nenhum dos filhos consegue chamar Task (profundidade 1)

Dado um hook PreToolUse que devolve updatedInput trocando "rm -rf build" por "rm -rf ./build"
Quando o modelo chama Bash
Então o comando executado é o reescrito, depois de passar de novo pelas permissões

Dado um hook PreCompact
Quando a compactação começa
Então o additionalContext do hook entra nas instruções do resumo

Dado um script de workflow do Claude Code com agent/parallel/pipeline/phase
Quando ele roda no dsh
Então o resultado tem o mesmo formato { run_id, agents_started, result } e não exige Node

Dado um goal ativo e uma sessão retomada depois de queda
Quando o agente fica ocioso
Então nenhuma rodada é enfileirada até o usuário rearmar o goal

Dado uma skill com disable-model-invocation: true
Quando o catálogo é montado
Então ela não aparece para o modelo e /nome continua funcionando
```

## Prioridade
| Requisito | MoSCoW | Justificativa |
|---|---|---|
| `Task` com agentes nomeados, máscara, alias e profundidade | Must | Requisitos críticos R2 e R10 do Totvs |
| Skills (`Skill`, `/nome`) | Must | Requisito crítico R5 |
| Hooks (`command`) com os eventos atuais | Must | Paridade e plugins |
| Segundo plano e filhos contínuos | Should | Usados por times e jobs |
| `updatedInput` e eventos novos (`PreCompact`, `Notification`, `SessionEnd`) | Should | Paridade completa com o Claude Code |
| Workflow em QuickJS | Should | Orquestração em escala |
| Goal | Should | Autonomia controlada |
| Delegação ao Claude Code e ACP | Could | Uso inverso do fallback |
| Hooks `http`/`prompt` | Could | Pouco usados |
| Ralph | Won't (padrão) | Conclusão autodeclarada |

## Rastreabilidade de Código
| Arquivo | Função / Classe | Cobertura |
|---|---|---|
| `packages/subagent/subagent`, `tool-subagent`, `tool-subagent-control` | contrato, ferramenta, controle | 🟢 |
| `packages/subagent/subagent-spawn-in-process`, `subagent-fork-in-process`, `subagent-acp`, `subagent-claude-code` | provedores | 🟢 |
| `packages/skill/*` | catálogo e ferramenta | 🟢 |
| `packages/hooks/hook-protocol`, `hooks-claude-code` | semântica de hooks | 🟢 |
| `packages/workflow/workflow`, `workflow-ptc`, `tool-workflow`, `tool-ralph` | workflow e ralph | 🟢 |
| `packages/goal/*` | goal e driver de rodadas | 🟢 |
| — | agentes `.claude/agents`, eventos novos, `updatedInput`, QuickJS | — (novo) |

## Atualização upstream 0.2.1-rc.5 (ciclo 1, 2026-11-01)
> Origem: [ledger](../upstream/ledger.md). Estas regras **prevalecem** sobre as anteriores quando houver conflito.

- **U1. Presets declarativos** com registro e revisões retidas; a composição de subagentes pertence ao preset. No dsh Rust, os "presets" correspondem aos agentes `.claude/agents` mais os embutidos, e o registro é o catálogo `subagent_type`. 🟡
- **U2. Ativações unificadas:** spawn, fork e contínuo passam a usar um único caminho de ativação. Isso confirma o desenho de provedores desta spec. 🟢
- **U3.** No upstream, os hooks Claude Code foram movidos para `experimental/` (`hooks-claude-code`, `hook-protocol`), com a mesma semântica. O dsh Rust mantém os hooks como **padrão** (decisão de paridade), sem mudança. 🟢
- **U4. Claude Code mods** (`experimental/claude-code-mods`, alpha): uma ponte que roda mods do Claude Code (`register(on, options)`) dentro das execuções. O dsh Rust **avalia depois** (Could): exigiria um runtime JS (QuickJS) e a API de mods. 🟡

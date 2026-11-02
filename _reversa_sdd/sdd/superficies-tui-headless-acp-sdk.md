# Superfícies: TUI, headless, ACP e SDK (`harness-cli` + `harness-tui` + `harness-acp` + `harness-sdk`)

## Visão Geral
As formas de usar o mesmo núcleo, todas no executável único `dsh`: 🟢 contratos do dsh / 🟡 alvo.
- **TUI** (padrão ao digitar `dsh`): interface interativa no terminal, equivalente ao `claude`. O dsh TS **não** tem TUI embutida e usa a interface web (`dsh web`). 🟢
- **Headless** (`dsh -p`): uma tarefa sem interação, com saída em texto, JSON ou stream JSON, compatível com `claude -p`. Serve para scripts e extensões. 🟡
- **ACP**: servidor Agent Client Protocol v1 para editores (Zed, JetBrains, extensões VS Code compatíveis) e **cliente** ACP para delegar a outros agentes. 🟢
- **SDK**: JSON-RPC 2.0 por stdio (NDJSON), para integração programática. 🟢

A interface web do dsh TS **não** é portada nesta etapa (Won't por ora). 🟡

## Responsabilidades
- Interpretar a linha de comando, carregar a configuração e escolher a superfície. 🟢/🟡
- Renderizar o stream do assistente, as ferramentas, as aprovações, as perguntas, o progresso MCP, a pressão de contexto e as notificações. 🟡
- Converter a entrada do usuário (texto, `/comando`, `@arquivo`, `!comando`, imagem colada) em mensagens da inbox. 🟢/🟡
- Expor o mesmo agente via ACP e SDK com controle de sessão, cancelamento e permissões. 🟢
- Encerrar de forma graciosa: até 5 s de drenagem; `SIGINT` sai com 130 e `SIGTERM` com 0. 🟢

## Interface

### CLI
| Comando | Efeito | Confiança |
|---|---|---|
| `dsh [prompt]` | TUI, opcionalmente já com um prompt inicial | 🟡 |
| `dsh -c` / `--continue` | Retoma a sessão mais recente do cwd | 🟡 |
| `dsh -r [id]` / `--resume [id]` | Retoma uma sessão (seletor se faltar o id) | 🟢/🟡 |
| `dsh -p "<prompt>" [--output-format text\|json\|stream-json] [--input-format text\|stream-json]` | Headless | 🟡 |
| Flags comuns | `--model`, `--permission-mode`, `--allowedTools`, `--disallowedTools`, `--max-turns`, `--append-system-prompt`, `--settings <arq>`, `--mcp-config <arq>`, `--add-dir <dir>`, `--verbose` | 🟡 |
| `dsh acp` | Servidor ACP por stdio | 🟢 |
| `dsh serve` | Servidor SDK JSON-RPC por stdio | 🟢 |
| `dsh models\|login\|mcp\|plugin\|integrations\|sync\|catalog\|compat\|rea\|doctor\|upstream` | Subcomandos de gestão (specs 2, 5–8, 17) | 🟡 |
| `dsh --version`, `dsh update` | Versão e autoatualização assinada (minisign) | 🟡 |

**Códigos de saída do headless:** 0 sucesso; 1 erro do agente ou do modelo; 2 uso inválido; 3 negação de permissão sem respondente; 130 interrompido. 🟡

### TUI
| Elemento | Comportamento | Confiança |
|---|---|---|
| Entrada | Várias linhas (`Shift+Enter`), histórico (`↑`), `@` completa arquivos, `/` completa comandos e skills, `!` executa no shell e insere a saída, imagens coladas viram anexos | 🟡 |
| Stream | Texto, raciocínio (recolhível) e ferramentas com estado (pendente, executando, ok, erro), diffs coloridos para `Edit`/`Write` | 🟡 |
| `Esc` | Cancela o turno (`cancel(user)`) | 🟢 |
| `Esc Esc` | Volta a uma mensagem anterior (fork da sessão) | 🟡 |
| `Shift+Tab` | Alterna os modos de permissão (`default` → `acceptEdits` → `plan`) | 🟡 |
| Aprovação | Diálogo com a prévia (comando, diff, URL, `destructiveHint`): permitir uma vez, nesta sessão, sempre, negar | 🟢/🟡 |
| `AskUserQuestion` | Seleção de opções com "Outro" | 🟢 |
| Barra de status | Modelo e provedor efetivos (com aviso de fallback), modo de permissão, pressão de contexto, jobs ativos, gate de rede | 🟡 |
| Comandos embutidos | `/help`, `/clear`, `/compact`, `/model`, `/permissions`, `/plan`, `/goal`, `/mcp`, `/agents`, `/skills`, `/plugin`, `/memory`, `/resume`, `/export`, `/doctor`, `/cost`, `/status` | 🟢/🟡 |

### Headless `stream-json` 🟡
Uma linha JSON por evento, no formato do Claude Code:
```json
{"type":"system","subtype":"init","session_id":"…","model":"gemini/gemini-2.5-pro","tools":[…],"cwd":"…"}
{"type":"assistant","message":{…}}
{"type":"user","message":{"content":[{"type":"tool_result",…}]}}
{"type":"result","subtype":"success","result":"…","session_id":"…","num_turns":3,"duration_ms":…,"usage":{…}}
```
`--output-format json` emite só o objeto `result`. `--input-format stream-json` aceita mensagens `{"type":"user","message":…}` pela entrada padrão.

### ACP v1 (servidor)
| Chamada | dsh TS | Alvo |
|---|---|---|
| `initialize`, `authenticate` (sem verificação local por stdio) | 🟢 | ✔ |
| `session/new` (workspace absoluto + MCP validados antes; rollback se o MCP falhar) | 🟢 | ✔ |
| `session/list` (100 por página, filtro por cwd), `session/resume`, `session/close` | 🟢 | ✔ |
| `session/prompt` (texto, links, imagens; um por vez; modelo fixo no turno) | 🟢 | ✔ |
| `session/cancel`, `$/cancel_request` | 🟢 | ✔ |
| `session/set_config_option` (`model`, `reasoning_effort`) | 🟢 | ✔ + `permission_mode` 🟡 |
| `session/update`, `session/request_permission` | 🟢 | ✔ |
| `session/load` (replay do histórico), modes, commands (slash e skills), plans, terminals | ❌ | ✔ 🟡 |

### SDK JSON-RPC 🟢
| Direção | Método |
|---|---|
| cliente → servidor | `initialize`, `session/prompt` (recibo durável de enfileiramento), `shutdown` |
| servidor → cliente | `session.event` (todos os eventos), `session.status` (`running`/`idle`), `subagent.started`/`subagent.finished` |

Ampliação 🟡: `session/cancel`, `session/approve` (respondente de aprovação) e `session/list`.

## Regras de Negócio
- **R1.** Todas as superfícies usam o mesmo núcleo e a mesma sessão em disco. Uma sessão criada no headless pode ser retomada na TUI e vice-versa. 🟢/🟡
- **R2.** No ACP e no SDK, o stdout carrega **só** tráfego de protocolo. Logs vão para stderr ou arquivo. 🟢
- **R3.** No ACP, a seleção de modelo fica **fixa durante o turno**. Uma mudança concorrente vale no próximo. O fallback não troca de modelo no meio de um turno ACP (spec 2, R9). 🟢
- **R4.** Clientes ACP e SDK por stdio são controladores confiáveis (autorizam os comandos MCP e o env informados). Exposição por socket ou HTTP exige token (spec 12, R22). 🟢/🟡
- **R5.** No headless, as aprovações seguem `--permission-mode` e as regras. O que pedir `ask` sem respondente é negado (política `never`) e o código de saída é 3 se isso impedir a conclusão. 🟢/🟡
- **R6.** O headless respeita `--max-turns`. Ao atingir o limite, termina com `result.subtype = "error_max_turns"`. 🟡
- **R7.** O formato `stream-json` é **compatível com o do Claude Code**, para que scripts e extensões que chamam `claude -p` possam apontar para `dsh -p` sem mudanças. Campos extras do dsh ficam sob `"dsh": {…}`. 🟡
- **R8.** A TUI mostra **sempre** o provedor e o modelo efetivos de cada resposta. Uma troca por fallback ou alias gera um aviso visível. 🟡
- **R9.** A TUI funciona no Windows Terminal, no terminal integrado do VS Code e no `conhost` (com degradação de cores), e preserva UTF-8 e acentuação. 🟡
- **R10.** Um encerramento por sinal faz o flush da sessão em até 5 s. Um segundo sinal força a saída. `Ctrl+C` com o turno ativo cancela o turno; `Ctrl+C` duas vezes com o agente ocioso sai. 🟢/🟡
- **R11.** `!comando` na TUI roda sob o sandbox e as permissões da sessão, e a saída entra como contexto do usuário, não como resultado de ferramenta. 🟡
- **R12.** `dsh update` só aplica binários com assinatura minisign válida da chave do projeto. Nada é enviado além do pedido de download. 🟡

## Fluxo Principal (TUI)
1. `dsh` no cwd do projeto: carrega a configuração, a compatibilidade e as integrações; roda os gates; mostra a barra de status. 🟡
2. O usuário digita e envia. A mensagem entra em `next_turn` via `followup`. 🟢
3. A TUI assina `assistant_stream` e os eventos de ferramenta e renderiza em tempo real. 🟢/🟡
4. Os pedidos de aprovação e as perguntas abrem diálogos. A resposta volta ao pipeline. 🟢
5. No `turn/end`, a TUI mostra o resumo (tokens, tempo) e volta à entrada. 🟡

## Fluxos Alternativos
- **Terminal sem suporte a cores:** modo monocromático. 🟡
- **Headless sem rede para o provedor:** fallback (spec 2). Se esgotar, sai com 1 e um `result.subtype = "error_during_execution"`. 🟡
- **Cliente ACP envia um prompt com outro ativo:** erro de protocolo "um prompt por vez". 🟢
- **Cliente ACP desconecta:** as sessões fecham com cancelamento quiescente e flush. 🟢

## Dependências
- `clap` (CLI), `ratatui` + `crossterm` (TUI), `agent-client-protocol` (ACP), `serde_json` (SDK/NDJSON), `self_update` + `minisign-verify`.
- Núcleo (spec 1), permissões (spec 12), compatibilidade (spec 4), subagentes (spec 13).

## Requisitos Não Funcionais
| Tipo | Requisito inferido | Evidência no código | Confiança |
|---|---|---|---|
| Disponibilidade | Drenagem de 5 s; `SIGINT` 130, `SIGTERM` 0 | `apps/cli/src/process-shutdown.ts` | 🟢 |
| Escalabilidade | Várias sessões por conexão ACP; `session/list` com 100 por página | `packages/acp/acp/src/session.ts` | 🟢 |
| Compatibilidade | `stream-json` igual ao do Claude Code | equivalência registrada no módulo SDK | 🟡 |
| Segurança | Atualização só com assinatura válida | padrão CoreOS / ADR 101 | 🟡 |
| Usabilidade | Provedor efetivo sempre visível | decisão de fallback (spec 2) | 🟡 |

> Inferido a partir do código. Validar a TUI com o usuário no piloto.

## Critérios de Aceitação
```gherkin
Dado um script que roda claude -p "x" --output-format stream-json
Quando o comando é trocado por dsh -p "x" --output-format stream-json
Então as linhas init, assistant, user e result têm os mesmos campos obrigatórios

Dado uma sessão criada com dsh -p
Quando o usuário roda dsh -r <id>
Então a TUI mostra o histórico e aceita um novo prompt

Dado a TUI com o turno ativo
Quando o usuário pressiona Esc
Então o turno é cancelado e a sessão termina em turn/end{aborted}

Dado um editor ACP conectado
Quando ele envia session/load de uma sessão existente
Então o histórico é reenviado como session/update e um novo prompt é aceito

Dado um fallback do Claude para o Gemini durante a sessão
Quando a resposta aparece na TUI
Então a barra de status e a mensagem indicam gemini/gemini-2.5-pro

Dado dsh -p com uma ação que exige aprovação e nenhum respondente
Quando a execução termina
Então o código de saída é 3 e o result explica a negação
```

## Prioridade
| Requisito | MoSCoW | Justificativa |
|---|---|---|
| TUI | Must | Uso principal como substituto do Claude Code |
| Headless `-p` com `stream-json` compatível | Must | Scripts e extensões que chamam `claude -p` |
| Retomada (`-c`/`-r`) | Must | Continuidade de trabalho |
| ACP servidor (contrato atual do dsh) | Should | Uso dentro de editores |
| ACP `session/load`, modes, commands | Should | Paridade com o que o Claude Code expõe |
| SDK JSON-RPC | Should | Integrações próprias |
| ACP cliente (delegação) | Could | Gemini CLI e outros (spec 13) |
| `dsh update` assinado | Should | Binário único sem gerenciador de pacotes |
| Interface web | Won't (por ora) | Fora do foco atual |

## Rastreabilidade de Código
| Arquivo | Função / Classe | Cobertura |
|---|---|---|
| `apps/cli/src/bin.ts`, `args.ts`, `process-shutdown.ts` | launcher, flags, sinais | 🟢 |
| `packages/acp/acp/src/index.ts`, `session.ts`, `codec.ts`, `updates.ts`, `model-control.ts` | servidor ACP | 🟢 |
| `packages/subagent/subagent-acp` | cliente ACP | 🟢 |
| `packages/sdk/protocol`, `server`, `client` | SDK JSON-RPC | 🟢 |
| `packages/api/*` | BFF da interface web (não portado) | 🟢 |
| — | TUI, `-p`/`stream-json`, `dsh update` | — (novo) |

# Ferramentas compatíveis com o Claude Code (`harness-tools`)

## Visão Geral
O conjunto de ferramentas que o modelo chama: arquivos, busca, shell, web, tarefas, perguntas ao usuário, histórico de sessões e execução programática. No dsh atual os nomes são minúsculos (`read`, `bash`, `pwsh`…). No dsh Rust os nomes são **idênticos aos do Claude Code** (`Read`, `Bash`, `PowerShell`…), para que agentes, skills, `allowed-tools`, permissões e hooks escritos para o Claude funcionem sem alteração (ADR 103). 🟢 contratos do dsh / 🟡 renomeação e novas ferramentas.

## Responsabilidades
- Definir cada ferramenta com nome, descrição, JSON Schema de entrada, schema de saída e `execute`. 🟢
- Validar os argumentos antes da execução. Um input inválido vira resultado de erro, não exceção. 🟢
- Passar toda chamada pelo pipeline fixo de permissões, guardas, hooks e aprovação. 🟢
- Aplicar a política read-before-write e a proteção contra versão obsoleta. 🟢
- Executar comandos com contenção da árvore de processos e sob o sandbox. 🟢
- Fazer spill de saídas grandes para arquivo, com preview head/tail. 🟢
- Rotular conteúdo externo (web, MCP) como não confiável. 🟢
- Aceitar os nomes antigos minúsculos como aliases em matchers e permissões. 🟡

## Interface

### Contrato de ferramenta 🟢
```rust
struct ToolDef {
    name: &'static str,               // nome Claude Code
    aliases: &'static [&'static str], // nomes dsh TS (read, bash, pwsh…)
    description: String,
    input_schema: JsonSchema,         // compartilhado com MCP, subagentes, workflow
    output: OutputSpec,               // schema + render
    concurrency: Concurrency,         // ParallelSafe | Exclusive
}
#[async_trait] trait Tool { async fn execute(&self, args: Value, exec: ExecCtx) -> ToolResult; }
```
`ExecCtx` traz o token de cancelamento cooperativo, o id da chamada, o agente, o cwd e o sandbox efetivo. Códigos genéricos: `UNKNOWN_TOOL`, `INVALID_INPUT`, `ABORTED_BEFORE_DISPATCH`, `ABORTED`, `TOOL_TIMEOUT`. 🟢

### Pipeline (ordem fixa) 🟢
1. `pre_execute`: o motor de permissões decide `allow`, `deny` ou `ask` (allow/deny/ask de `settings.json`, modo de permissão, plan mode).
2. Guardas monotônicos: uma negação nunca é revertida. Inclui o guarda de caminhos negados, como o `.env`.
3. Hooks `PreToolUse`: exit 2 bloqueia e o stderr vai para o modelo.
4. Aprovação humana quando a decisão for `ask`. Sem quem responda, falha fechado.
5. `execute`, com timeout, métricas e checkpoint de durabilidade antes do corpo.
6. `post_execute`: spill, pruner e hooks `PostToolUse`.
7. `finalize_content` e `tool/result`.

### Catálogo
| Ferramenta (Claude Code) | Alias dsh TS | Argumentos | Concorrência | Confiança |
|---|---|---|---|---|
| `Read` | `read`, `read_image` | `file_path`, `offset?` (base 1), `limit?`, `pages?` (PDF) | paralela | 🟢 (imagens e PDF 🟡) |
| `Write` | `write` | `file_path`, `content` | exclusiva por arquivo | 🟢 |
| `Edit` | `edit`, `str_replace_editor` | `file_path`, `old_string`, `new_string`, `replace_all?` | exclusiva por arquivo | 🟢 |
| `NotebookEdit` | — | `notebook_path`, `cell_id?`, `new_source`, `cell_type?`, `edit_mode?` | exclusiva | 🟡 |
| `Glob` | `glob` | `pattern`, `path?` | paralela (no dsh TS: exclusiva) | 🟡 |
| `Grep` | `grep` | `pattern`, `path?`, `glob?`, `type?`, `output_mode?`, `-A/-B/-C?`, `-i?`, `-n?`, `multiline?`, `head_limit?` | paralela (no dsh TS: exclusiva) | 🟡 |
| `Bash` | `bash` | `command`, `description?`, `timeout?` (ms, máx. 600.000), `run_in_background?` | exclusiva | 🟢 |
| `PowerShell` | `pwsh` | idem `Bash` | exclusiva | 🟢 |
| `WebFetch` | `web_fetch` | `url`, `prompt?` | paralela | 🟢 (`prompt` 🟡) |
| `WebSearch` | `web_search` | `query` ou `queries[1..4]` | paralela | 🟢 |
| `TodoWrite` | `todo_write` | `todos[{content, status, activeForm?}]` | exclusiva | 🟢 |
| `AskUserQuestion` | `ask_user_question` | `questions[{question, header, options[{label, description}], multiSelect?}]` | exclusiva | 🟢 |
| `ExitPlanMode` | `exit_plan_mode` | `plan` | exclusiva | 🟢 |
| `Task` / `Agent` | `subagent` | `description`, `prompt`, `subagent_type?`, `model?` | paralela | 🟢 (spec 13) |
| `Skill` | `skill` | `skill`, `args?` | exclusiva | 🟢 (spec 13) |
| `BashOutput` / `KillShell` | `job_output` / `job_kill` | `bash_id` \| `job_id`, `filter?` | paralela | 🟢 (spec 16) |
| `LSP` | `lsp` | `operation`, `file_path`, `line`, `character` | paralela | 🟢 (spec 11) |
| `SessionSearch` (+4) | `session_search`… | ver abaixo | paralela | 🟢 |
| `RunCode` | `run_code` | `code`, `language` | exclusiva | 🟢 |
| `mcp__<srv>__<tool>` | igual | schema do servidor | conforme anotações MCP | 🟢 (spec 7) |

## Regras de Negócio

### Nomes e compatibilidade
- **R1.** O nome exposto ao modelo é o do Claude Code. Os matchers de hooks, `allowed-tools` de skills e agents e as regras de permissão aceitam **também** os aliases minúsculos, normalizados para o nome canônico antes da comparação. 🟡
- **R2.** As regras de permissão seguem a sintaxe do Claude Code: `Tool`, `Tool(padrão)`, `Bash(git log:*)`, `Read(./.claude/.env)`, `mcp__srv`, `mcp__srv__tool`. A ordem de avaliação é `deny` > `ask` > `allow` > modo padrão. 🟡

### Arquivos
- **R3. Read-before-write:** `Write` cria livremente, mas só sobrescreve um arquivo que a sessão já leu. `Edit` sempre exige leitura prévia. Um arquivo alterado depois da leitura gera `FS_STALE_VERSION`. Ler um arquivo inexistente permite um create guardado. Ao retomar a sessão, o estado de observação **não** é restaurado. Códigos: `FS_NOT_OBSERVED`, `FS_STALE_VERSION`, `FS_NOT_FOUND`. 🟢
- **R4.** `Edit` faz substituição literal. Sem `replace_all`, o `old_string` precisa ser único, senão a chamada falha com uma mensagem que diz quantas ocorrências achou. 🟢
- **R5.** `Read` devolve o conteúdo numerado por linha (`cat -n`), com rodapé de paginação e limite padrão de 2.000 linhas (`packages/fs/tool-fs/src/read.ts:15`, `READ_LIMIT = 2000`). Uma imagem (PNG, JPEG, WebP, GIF) vira anexo e é recusada se o modelo da rota for só-texto. Um PDF é lido por páginas (`pages`, no máximo 20 por chamada) com texto extraído. 🟢/🟡
- **R6.** Escrita atômica (arquivo temporário + rename) com `fsync`, preservando o fim de linha (CRLF/LF) original do arquivo editado. 🟢/🟡
- **R7.** `Glob` usa o motor do ripgrep (`ignore`), inclui arquivos ocultos, exclui `.git`, ordena por mtime e casa um padrão sem `/` com nomes de arquivo em qualquer profundidade. 🟢
- **R8.** `Grep` usa a regex do ripgrep (`grep-searcher`), agrupa por arquivo e tem `head_limit` padrão de 250. 🟢/🟡

### Shell
- **R9.** `Bash` e `PowerShell` são one-shot: um processo novo a cada chamada. O cwd do shell persiste entre chamadas na sessão (como no Claude Code), mas variáveis e funções não. Mudar o cwd do shell **não** muda o workspace do sandbox, que continua sendo o cwd inicial da sessão (spec 12, R4). [Revisão] 🟡 (o dsh TS tem as variantes one-shot e persistente 🟢)
- **R10.** Exit ≠ 0, timeout e cancelamento são **resultados**, não exceções. O resultado traz `[exit code: N]`. Só falhas de infraestrutura geram erro. Timeout padrão de 120 s, máximo de 600 s (`packages/shell/bash-local/src/index.ts:107-108`). 🟢
- **R11.** Toda a árvore de processos fica contida (Job Object kill-on-close no Windows; cgroup ou scope no Linux) e morre junto com a chamada ou a sessão. Os filhos rodam com janela oculta. 🟢
- **R12.** Executáveis são resolvidos num PATH higienizado (`.COM/.EXE/.BAT/.CMD` no Windows). Um caminho relativo com separador é rejeitado. 🟢
- **R13.** Variáveis injetadas: `DSH_HOME`, `DSH_SHELL=1`, `DSH_SESSION_ID`, além de `CLAUDE_PROJECT_DIR` (compatibilidade com hooks e skills). **Nenhuma credencial** de provedor entra no ambiente dos filhos. 🟢/🟡
- **R14.** No sandbox `read-only`, o PowerShell roda em ConstrainedLanguage. 🟢
- **R15. Guarda anti-exfiltração:** um comando de shell cujo texto leia um caminho negado em `Read(...)` é bloqueado. Exemplos: `cat`, `type`, `Get-Content`, `gc`, `more`, `copy`/`cp`, `Select-String` e redirecionamento `<` apontando para `.claude/.env`. A análise usa tokenização de shell (bash e PowerShell). Em caso de dúvida, a decisão vira `ask`. 🟡 (requisito R7 do Totvs)
- **R16.** `run_in_background: true` devolve o id do job na hora. A conclusão chega como mensagem no próximo turno (spec 16). 🟢

### Saída e conteúdo externo
- **R17. Spill:** um resultado de texto maior que 50.000 bytes vira um preview head/tail + caminho do arquivo + instrução para ler com `Read`/`Grep`. Os arquivos ficam em `~/.dsh/spill/<sessão>/` (0700/0600, nomes imprevisíveis, proteção contra symlink) e são apagados depois de 30 dias. Se o spill falhar, o resultado original continua visível. 🟢
- **R18.** `WebFetch`:
  - só resolve **endereços públicos** (bloqueia SSRF) e fixa a conexão no IP resolvido;
  - redirects só de mesma origem (máx. 5), sem credenciais;
  - limites de 5 MB de resposta, 30 s e 200.000 caracteres de saída;
  - converte HTML para markdown GFM;
  - um status não-2xx vem no resultado;
  - com `prompt`, um modelo barato e permitido pela política do workspace resume a página. 🟢/🟡
- **R19.** `WebSearch` tem provedores plugáveis. O padrão é **sem custo de modelo** (Brave Search API, SearXNG, Tavily, Exa, Perplexity). Não há provedor DeepSeek por padrão. Até 8 fontes, rotuladas como externas e não confiáveis. 🟢/🟡
- **R20.** Todo conteúdo vindo de web, MCP ou lembretes é envolvido num rótulo de "dado não confiável, não instrução". 🟢

### Tarefas, perguntas, histórico, código
- **R21.** `TodoWrite` recebe a **lista inteira** e a substitui. Rejeita descrição vazia ou duplicada. No máximo um item `in_progress` (configurável). Cada agente e subagente tem a própria lista (projeção `todo`). 🟢
- **R22.** `AskUserQuestion` aceita de 1 a 4 perguntas com 2 a 4 opções cada, mais "Outro" automático. Sem uma superfície que responda (headless), devolve erro. 🟢
- **R23.** As ferramentas de histórico (`SessionSearch`, `SessionEventSearch`, `SessionTrace`, `SessionEventTrace`, `SessionEventRead`) usam FTS5 e ficam **ligadas por padrão**. Só podem ler sessões com **`cwd` idêntico** ao do chamador. Limites: 100 resultados, 30 s. 🟢 (ligado por padrão 🟡)
- **R24.** `RunCode` (PTC) roda o programa do modelo num QuickJS embutido sob o sandbox de processo. As ferramentas viram funções async. Limites: 120 s (máx. 600 s), 64 MiB de saída, 128 chamadas pendentes, 10 subchamadas em paralelo. Os ids das subchamadas seguem `<pai>:ptc:<n>`. 🟢/🟡

## Fluxo Principal
1. O loop recebe as tool calls da `assistant/message` e as ordena: as paralelizáveis vão juntas, sob um semáforo de 10; as exclusivas funcionam como barreira. 🟢
2. Para cada chamada: valida o schema e normaliza nome e alias. 🟢/🟡
3. Pipeline: permissões, guardas, hooks, aprovação, checkpoint, execução. 🟢
4. Pós-execução: spill, hooks `PostToolUse`, rótulo de não confiável. 🟢
5. Grava `tool/result` e devolve ao loop. 🟢

## Fluxos Alternativos
- **Ferramenta desconhecida:** resultado `UNKNOWN_TOOL`; o turno continua. 🟢
- **Argumentos inválidos:** resultado de erro com o caminho do campo inválido. 🟢
- **`Edit` sem leitura prévia:** `cannot modify "<path>": file has not been read — read the file, then retry`. 🟢
- **Negação por regra ou hook:** resultado de erro com o motivo; o modelo pode tentar outra abordagem. 🟢
- **Aprovação sem resposta (headless com política `never`):** rejeitada, falha fechado. 🟢
- **Cancelamento durante a execução:** o token é sinalizado, a árvore de processos é morta e o resultado é `ABORTED`. 🟢
- **Timeout:** `TOOL_TIMEOUT`, com a saída parcial preservada no spill. 🟢

## Dependências
- `harness-sandbox`: Job Objects, AppContainer ou token restrito no Windows; Landlock e seccomp no Linux.
- `harness-store`: spill, anexos, FTS.
- `harness-claude-compat`: regras de permissão, hooks, `allowed-tools`.
- `harness-script` (rquickjs): `RunCode` e workflow.
- Crates: `grep-searcher`, `ignore`, `portable-pty`, `reqwest` (rustls), `hickory-resolver`, `htmd`, `image`, `lopdf`/`pdf-extract`, `windows-rs`.

## Requisitos Não Funcionais
| Tipo | Requisito inferido | Evidência no código | Confiança |
|---|---|---|---|
| Desempenho | No dsh TS, só são concurrency-safe `read`, `read_image`, `web_fetch`, `web_search`, as ferramentas de histórico e `subagent` (`isConcurrencySafe`); as demais são exclusivas. O alvo pode ampliar para `Glob`/`Grep`/`LSP`/MCP `readOnlyHint` | `packages/core/tools/src/index.ts:1284`; `fs/tool-fs/src/read.ts`; `web/tool-web/src/*.ts`; `subagent/tool-subagent/src/index.ts` | 🟢 |
| Desempenho | Até 10 ferramentas paralelas; 10 subchamadas de PTC | `packages/core/agent-loop/src/tool-calls.ts`, `packages/core/tools/src/ptc.ts` | 🟢 |
| Segurança | SSRF: só IP público, redirects de mesma origem | `packages/web/web-fetch-http` | 🟢 |
| Segurança | Read-before-write e version guard | `packages/fs/fs-observation-policy` | 🟢 |
| Segurança | Contenção da árvore de processos | `packages/subprocess/subprocess-local`, `win32-process` | 🟢 |
| Segurança | Bloqueio da leitura de `.env` via shell | requisito R7 do inventário Totvs | 🟡 |
| Desempenho | Spill acima de 50 KB, limpeza depois de 30 dias | `packages/spill/spill-policy`, `spill-local` | 🟢 |
| Desempenho | Timeouts: shell 120/600 s, fetch 30 s, PTC 120/600 s, session-query 30 s | pacotes citados | 🟢 |

> Inferido a partir do código. Validar com a equipe de operações.

## Critérios de Aceitação
```gherkin
Dado um agente do Totvs AGP com tools: Read, Grep, Bash
Quando o dsh carrega o agente
Então as três ferramentas são resolvidas pelos nomes Claude Code, sem aviso de ferramenta desconhecida

Dado a regra deny Read(./.claude/.env)
Quando o modelo chama Bash com "Get-Content .claude/.env"
Então a chamada é bloqueada pelo guarda anti-exfiltração e o resultado explica a regra

Dado um arquivo não lido na sessão
Quando o modelo chama Edit nele
Então o resultado é FS_NOT_OBSERVED e o arquivo não muda

Dado um arquivo lido e depois alterado por outro processo
Quando o modelo chama Edit
Então o resultado é FS_STALE_VERSION

Dado um comando que imprime 2 MB
Quando Bash termina
Então o resultado tem preview head/tail e o caminho do spill, e Read nesse caminho devolve o conteúdo completo

Dado WebFetch para http://192.168.0.10/admin
Quando a URL é resolvida
Então a chamada é recusada por apontar para rede privada

Dado um hook PreToolUse com matcher "Bash" que sai com código 2
Quando o modelo chama Bash (ou o alias bash)
Então a chamada é bloqueada e o stderr do hook vai para o modelo
```

## Prioridade
| Requisito | MoSCoW | Justificativa |
|---|---|---|
| Read/Write/Edit/Glob/Grep/Bash/PowerShell | Must | Caminho crítico de qualquer tarefa |
| Nomes Claude Code + aliases | Must | Compatibilidade com o Totvs AGP e plugins (ADR 103) |
| Pipeline com permissões, hooks e aprovação | Must | Segurança e paridade |
| Guarda anti-exfiltração de `.env` | Must | Requisito crítico R7 |
| Spill | Must | Evita estourar o contexto |
| WebFetch/WebSearch sem custo de modelo | Should | Há alternativa via MCP |
| TodoWrite / AskUserQuestion / ExitPlanMode | Should | Paridade de UX |
| Ferramentas de histórico (FTS) | Should | Complementam o GBrain |
| Leitura de PDF / NotebookEdit | Could | Uso ocasional |
| RunCode (PTC) | Could | Otimização de tokens; workflow depende do mesmo runtime |

## Rastreabilidade de Código
| Arquivo | Função / Classe | Cobertura |
|---|---|---|
| `packages/core/tools/src/index.ts`, `schema.ts`, `json-schema.ts` | `defineTool`, DSL de schema | 🟢 |
| `packages/core/tools/src/ptc.ts`, `presentation.ts` | modos `native`/`ptc`/`both` | 🟢 |
| `packages/core/agent-loop/src/tool-calls.ts` | concorrência | 🟢 |
| `packages/fs/tool-fs`, `tool-fs-search`, `tool-str-replace-editor` | Read/Write/Edit/Glob/Grep | 🟢 |
| `packages/fs/fs-observation-policy` | read-before-write | 🟢 |
| `packages/shell/tool-bash*`, `tool-pwsh*`, `shell-env` | Bash/PowerShell | 🟢 |
| `packages/subprocess/subprocess-local`, `win32-process` | contenção e PATH | 🟢 |
| `packages/web/tool-web`, `web-fetch-http`, `web-search-*` | WebFetch/WebSearch | 🟢 |
| `packages/todo/tool-todo` | TodoWrite | 🟢 |
| `packages/interaction/tool-ask-user`, `user-approval` | AskUserQuestion, aprovação | 🟢 |
| `packages/spill/*` | spill | 🟢 |
| `packages/session-query/tool-session-query`, `session-query-sqlite` | histórico FTS | 🟢 |
| `packages/ptc-runtime/*` | RunCode | 🟢 |
| — | aliases, guarda anti-exfiltração, PDF, NotebookEdit | — (novo) |

## Atualização upstream 0.2.1-rc.5 (ciclo 1, 2026-11-01)
> Origem: [ledger](../upstream/ledger.md). Estas regras **prevalecem** sobre as anteriores quando houver conflito.

- **U1. Apresentação:** cada agente escolhe **`native` ou `ptc`**; `both` é recusado na configuração. Substitui o modo `both` citado em R24. 🟢
- **U2. `RunCode` persistente (opt-in):** `execution_mode = "per-call"` (padrão, processo novo) ou `"persistent-in-memory"` (REPL por agente). Os dois rodam **sob o mesmo sandbox**. Uma mudança de política cancela as chamadas afetadas e invalida os processos retidos antes da próxima execução. 🟢
- **U3. `working_directory({ cd? })`:** ferramenta nova para ler ou trocar o diretório efetivo da sessão (spec 12, U1). O equivalente no Claude Code é o `cd` persistente do Bash, que continua valendo (R9). 🟢/🟡
- **U4. `AskUserQuestion` com timeout:** ao expirar, a pergunta devolve `pending`; uma resposta posterior chega como mensagem durável do usuário (steer). 🟢
- **U5.** O parse incremental dos argumentos durante o stream serve só à apresentação ao vivo e não muda a validação final (R2). 🟢

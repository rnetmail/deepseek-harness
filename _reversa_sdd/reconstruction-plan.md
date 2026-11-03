# Reconstruction Plan — deepseek-harness → dsh (Rust)

**Stack:** Rust (edition 2021, resolver 3), Cargo workspace com crates `harness-*` e binário `dsh`; tokio, reqwest (rustls + certificados do SO), serde, rusqlite (FTS5), zstd, rmcp, agent-client-protocol, ratatui, portable-pty, windows-rs, keyring, rquickjs, tree-sitter. Alvo principal: `x86_64-pc-windows-msvc`.
**Diretório do código:** `D:\dsh\dsh-rs` (repositório novo; o `deepseek-harness` não é alterado).
**Gerado em:** 2026-11-01
**Status:** 25 tarefas | 16 concluídas | 9 pendentes

---

## Alertas de pré-voo

> Revise estes pontos antes de iniciar. Gaps marcados com ⚠️ bloqueiam a tarefa associada. A revisão terminou sem lacunas 🔴; os alertas abaixo são de ambiente e de piloto.

- ⚠️ **Toolchain Rust ausente** (`cargo`/`rustc` não encontrados). O MSVC já existe: VS 2022 Community e Build Tools 2019. Instalar com `winget install Rustlang.Rustup` e depois `rustup default stable-x86_64-pc-windows-msvc`. Bloqueia a Tarefa 00 (Workspace e toolchain).
- ⚠️ **Diretório do código:** proposto `D:\dsh\dsh-rs`. Confirmar antes da Tarefa 00.
- ⚠️ **TLS corporativo:** no Windows o `cargo` usa o schannel, que confia nos certificados do sistema, então deve funcionar sem ajuste. Se a inspeção TLS da empresa falhar na checagem de revogação, configurar `[http] check-revoke = false` só no `cargo` (nunca desligar a verificação de certificado). Validar na Tarefa 00.
- ⚠️ **.NET 10 SDK** (opcional) para o `csharp-ls` atual; com o .NET 7, só a 0.10.0 serve. Afeta a Tarefa 14 (LSP).
- ⚠️ **Piloto:** a cota do Gemini (`max_concurrent`) afeta a Tarefa 05 (Provedores); o `cua-driver-rs` nightly no Windows afeta a Tarefa 19 (Browser/computer/teams). Não bloqueiam a implementação.

---

## Tarefas

### Tarefa 00 — Workspace Cargo, toolchain e CI local
**Status:** done
**Lê:** `_reversa_sdd/sdd/build-release.md`, `_reversa_sdd/c4-containers.md`
**Constrói:** `D:\dsh\dsh-rs\Cargo.toml` (workspace, MSRV), os 13 crates vazios com `lib.rs`, binário `dsh` (`--version`), `deny.toml`, `rustfmt.toml`, `clippy.toml`, `xtask/`, `.gitignore`, `README.md`
**Pronto quando:** `cargo build --locked` e `cargo test` passam; `dsh --version --verbose` imprime versão, commit e MSRV; `cargo deny check` roda
**Resultado (2026-11-02):** workspace em `D:dshdsh-rs` (13 crates + `xtask`); `scripts/check.ps1 -Gnu` passa (fmt, clippy `-D warnings`, 3 testes, `cargo deny`: advisories/bans/licenses/sources ok); `dsh --version --verbose` ok. Compilado com `stable-x86_64-pc-windows-gnu` + winlibs GCC 16.2.0 (MinGW-w64 14.0.0 UCRT, SHA-256 verificado) porque falta o Windows SDK. **Pendente fora desta tarefa:** instalar o Windows SDK na empresa e validar `scripts/check.ps1` com MSVC.

---

### Tarefa 01 — Formato de sessão v4 (+ leitura v3) e persistência JSONL
**Status:** done
**Lê:** `_reversa_sdd/sdd/agent-loop-sessao.md`, `_reversa_sdd/data-dictionary.md`, `_reversa_sdd/database/data-dictionary.md`
**Constrói:** `crates/harness-store`: enum `SessionEvent` (serde, tag `type`), header, `surface_op`, validações de append (R6–R8, U2–U5), codec JSONL + zstd, layout `--<cwd>--/<id>/session.v4.jsonl.zst`, gerações, migração v3→v4 em memória, trava de escrita (semáforo nomeado `Local\dsh-session-lock-<sha256>`), reparo de cauda
**Pronto quando:** fixtures v3 reais de `~/.dsh/sessions` abrem em leitura; um append inválido é recusado; uma segunda abertura em escrita falha com `AlreadyOwned`; uma queda simulada é reparada
**Resultado (2026-11-02):** `crates/harness-store` com os módulos `header`, `event`, `layout`, `codec`, `migrate`, `validate`, `lock` e `store`; 33 testes. As **9 sessões reais v3** de `~/.dsh/sessions` abrem em leitura, passam na validação e migram para v4 em memória (`cargo test -p harness-store -- --ignored real_ts_sessions`). Uma abertura em escrita de v3 publica `session.v4.jsonl.zstd` sem tocar na v3. `zstd` atualizado para 0.14.3 por três advisories (RUSTSEC-2026-0391/0398/0403).
**Lacuna registrada:** a lista completa de tags de conteúdo desconhecidas (renomeadas para `plugin:<tag>` no upstream) e o resto da tabela de produtores da migração v3→v4 não estão na spec; tags e produtores fora da tabela são preservados como estão. Resolver no próximo ciclo de upstream-sync lendo `session-format-v3-to-v4/src`.

---

### Tarefa 02 — Configuração e layout de `~/.dsh`
**Status:** done
**Lê:** `_reversa_sdd/sdd/configuracao-dsh-home.md`
**Constrói:** `crates/harness-core::config`: layout, precedência (CLI > env > workspace > `.claude` > `~/.dsh` > padrões), restrição de política só para restringir, validação com linha e coluna, watch com debounce de 100 ms, escrita com `toml_edit` + lock, `dsh config show|get|set|validate|migrate`
**Pronto quando:** os 4 cenários de aceitação da spec passam em teste
**Resultado (2026-11-02):** `harness-core::config` com os módulos `paths`, `load`, `merge`, `secrets`, `write`, `watch` e `migrate`, mais `dsh config show|get|set|validate|migrate` e as flags globais `--model`, `--permission-mode` e `--settings`; 20 testes. Os 4 cenários foram validados ponta a ponta com um `DSH_HOME` temporário e uma cópia do `settings.yaml` real: 2 provedores migrados, `ui-*` ignorados; `--model openrouter-free/x` recusado pela política; `config get permission_mode --origin`; arquivo inválido mantém o último estado válido. A biblioteca YAML é `yaml-rust2`, porque o `serde_yaml_ng` puxa o `unsafe-libyaml`, sem manutenção (RUSTSEC-2024-0449).
**Pendente para a Tarefa 04:** migrar o `.credentials.yaml` para o cofre do SO.

---

### Tarefa 03 — Entidades de domínio e máquinas de estado
**Status:** done
**Lê:** `_reversa_sdd/domain.md`, `_reversa_sdd/state-machines.md`, `_reversa_sdd/erd-complete.md`
**Constrói:** tipos de domínio compartilhados (Turn, Step, Inbox, ToolCall, Approval, Job, Reminder, McpServerState, WorkspacePolicy, ModelAlias) e as 10 máquinas de estado como enums com transições validadas
**Pronto quando:** toda transição documentada é aceita e as não documentadas são recusadas em teste
**Resultado (2026-11-02):** `harness-core::domain` com o trait `StateMachine` (`ALL`, `initial`, `allows`, `transition`, `is_terminal`), as 10 máquinas (agente, chamada de ferramenta, aprovação, escalonamento, goal, job + aviso, lembrete, MCP, handle de persistência + `HeaderStatus`, pacote cordis) e os tipos `ModelRef`, `ModelAlias`, `DataGate`, `WorkspacePolicy`, `Inbox`, `Turn`, `Step`, `Agent`, `ToolCall`, `Approval`/`ApprovalPolicy`, `WakeupBudget`, `Goal`, `JobId`, `Job`, `Reminder`, `McpServer`/`ReconnectPolicy`. Teste exaustivo: para cada máquina, todo par de estados é aceito se e somente se estiver na lista de setas do documento; 23 testes novos (79 no total) e `check.ps1 -Gnu` verde. Interpretações 🟡 anotadas no código: job que termina sozinho vai de `running` direto a `completed`/`failed` (`killed` só depois de `stopping`); `connecting → failed` (falha inicial) e `reconnecting → reconnecting` (cada tentativa); `writing → closed`; `backgrounded`, `refused` e `overdue` sem saída além das documentadas (um lembrete atrasado não pode ser apagado antes de disparar).

---

### Tarefa 04 — Sandbox, contenção de processos e cofre de credenciais
**Status:** done
**Lê:** `_reversa_sdd/sdd/seguranca-sandbox-permissoes.md`, `_reversa_sdd/permissions.md`
**Constrói:** `crates/harness-sandbox`: Job Object kill-on-close, token restrito + ACL com integridade obrigatória (U3), AppContainer opcional, fail-closed (`SANDBOX_UNAVAILABLE`), escalonamento de uma chamada; `keyring` para credenciais (`dsh/<escopo>/<id>`) com precedência U2
**Pronto quando:** um processo filho morre com o pai; uma escrita fora do workspace é negada com dica; sem backend, nada roda; uma credencial nunca aparece no ambiente do filho
**Resultado (2026-11-02):** `harness-sandbox` com `Sandbox` (sonda de 5 s por backend, fallback em falha de preparo, `SANDBOX_UNAVAILABLE` sem criar processo), backends Windows `windows-appcontainer` (opcional, `full`), `windows-restricted-token` (`partial`) e `unconfined` (só `danger-full-access`); Job Object kill-on-close com o processo criado suspenso e associado antes da primeira instrução, herança só dos handles de stdio; token `WRITE_RESTRICTED` + integridade baixa, e concessão de ACL com ACE + rótulo de integridade numa única `SetNamedSecurityInfoW` (U3), com um SID por workspace; TEMP privado; dica `[sandbox: file access denied under <modo> mode]`; `Escalations` sobre a `EscalationState` do domínio; `EnvScrub` (remove a variável da credencial e qualquer variável que contenha o valor); cofre `dsh/<escopo>/<id>` via `keyring-core` + `windows-native-keyring-store`, `Secret` que não se imprime e zera a memória, precedência U2 e `.env` com `deny` respeitado; migração do `.credentials.yaml` (layout `version: 1` e o plano antigo, BOM aceito) que grava no cofre, registra `state/credentials-migration.json` e só remove o arquivo com confirmação; `dsh credentials set|check|delete|migrate`. 26 testes novos (105 no total) e `check.ps1 -Gnu` verde, incluindo testes contra o Windows real: filho e neto morrem com o job; timeout mata a árvore; escrita/exclusão fora do workspace negada com dica e leitura livre; `read-only` nega escrita no workspace; chamada ampliada uma vez e a seguinte volta ao `workspace-write`; sem backend nada roda; `set` e `Get-ChildItem env:` não mostram a credencial; AppContainer confina a escrita. Teste manual (ignorado no CI) com o Gerenciador de Credenciais real: gravar, ler e apagar. Achado: SIDs da autoridade 15 (`S-1-15-…`) são recusados como SIDs restritivos; o SID do sandbox usa a autoridade 9 (`S-1-9-…`).
**Pendente:** backends Linux/macOS (Landlock/Seatbelt) ainda não existem e falham fechado; `dsh doctor` alertar sobre `dsh.cmd` com chaves (R19) fica para a tarefa do doctor; o sandbox aplica-se aos hooks e scripts de skill quando esses forem implementados (R5).

---

### Tarefa 05 — Provedores e modelos (adapters, aliases, política, gates)
**Status:** done
**Lê:** `_reversa_sdd/sdd/provedores-modelos.md`
**Constrói:** `crates/harness-llm`: trait `LlmAdapter`, protocolo de chunks, adapters `openai-chat`, `openai-responses`, `anthropic-messages`, `gemini-generate-content`; `LlmFailure`; retry; fallback; aliases `opus`/`sonnet`/`haiku`; `data_policy`/`allowed_providers`/`data_gate`; gate de rede; `max_concurrent` + fila; OAuth (PKCE/device) incluindo Google (via 2); `dsh models|login`
**Pronto quando:** os 6 cenários de aceitação da spec passam contra servidores mock
**Alerta:** cota real do Gemini será medida no piloto (`quota_stats`)
**Resultado (2026-11-02):** `harness-llm` com protocolo de chunks + `BlockAssembler` (um `finish` terminal, JSON cru de tool-call), `LlmFailure` com os códigos da spec + `POLICY_DENIED`/`NETWORK_GATE_CLOSED` e a leitura de texto do provedor centralizada em `classify_http` (R2); adapters `openai-chat`, `openai-responses`, `anthropic-messages` e `gemini-generate-content` (AI Studio por `x-goog-api-key` e Code Assist por OAuth, com `thoughtSignature`); `prepare_request` (R3 replay só no mesmo protocolo, R4 placeholders de imagem/arquivo, R5 `reasoning_effort`/`stop`); `LlmRuntime` com registro de um adapter por protocolo e troca atômica de rotas (R1), aliases com glob da versão mais nova (R16), política antes de qualquer I/O (R10/R11), gate de dados `ask` uma vez por sessão com `policy/data-gate` (R12), gate de rede DNS/TCP/tnsping 3 s + guards + seção de prompt (R13), retry 5× 500 ms–10 s com jitter 10% e `Retry-After`, `llm/retry` gravado antes da espera (R7), fallback pela cadeia com `llm/fallback` e erro final citando cada candidato (R8), sem fallback no ACP (R9), `max_concurrent` com fila FIFO e `queue_depth` (R22b), `quota_stats` (R22), estimativa `ceil(chars/4)` (R23); catálogo embutido exportado do pi-ai 0.85.1 (39 provedores, 1354 modelos, 113 gratuitos) com atualização local e regra `models` substitui/herda (R14/R15) e aviso de treinamento (R17); credenciais pelo cofre com precedência U2 e registro no `EnvScrub`; OAuth Google via 2 (PKCE loopback, `loadCodeAssist`/`onboardUser`, grant em `dsh/oauth/<rota>`, refresh sob trava entre processos R19, aviso de risco no primeiro login, client OAuth da configuração ou do Gemini CLI instalado); CLI `dsh models list|add|refresh|doctor` e `dsh login <rota> [--api-key]`. HTTP com rustls + provedor `ring` + verificador de certificados do Windows (TLS corporativo sem desligar verificação). 49 testes novos (154 no total), `check.ps1 -Gnu` verde; os 6 cenários de aceitação + fila + ACP passam contra servidores mock.
**Pendente:** OAuth das demais assinaturas de R20 (Anthropic Pro/Max, ChatGPT/Codex, Copilot com device code, xAI, Kimi, OpenRouter) aguarda saber quais assinaturas o usuário tem; a via 2 só foi validada contra mock (o formato do Code Assist é 🟡 até o primeiro `dsh login gemini` real); a via 3 (`acp:gemini-cli`) é recusada aqui e entra com o `harness-integrations`/ACP; `models refresh` busca só o OpenRouter e sem verificação minisign (não há chave publicada); tokenizer por família (R23) ainda é a estimativa por caracteres; gravar os eventos no log da sessão fica com o loop do agente (o runtime entrega `LlmEvent` com `kind()`/`data()`).

---

### Tarefa 06 — Ferramentas compatíveis com o Claude Code
**Status:** done
**Lê:** `_reversa_sdd/sdd/ferramentas-claude-code.md`
**Constrói:** `crates/harness-tools`: contrato `ToolDef`, pipeline, Read/Write/Edit/NotebookEdit/Glob/Grep/Bash/PowerShell/WebFetch/WebSearch/TodoWrite/AskUserQuestion/ExitPlanMode/SessionSearch/`working_directory`, aliases minúsculos, read-before-write, spill, guarda anti-exfiltração de `.env`
**Pronto quando:** os 7 cenários de aceitação da spec passam
**Resultado (2026-11-02):** `harness-tools` com o contrato `ToolDef`/`Tool`/`ExecCtx`/`SessionState`, nomes Claude Code + aliases do dsh TS em matchers e `allowed-tools` (R1), validação de schema com caminho do campo, e `Pipeline` na ordem fixa: ferramenta → schema → `PermissionEngine` (trait; `AllowAll` até o `harness-claude-compat`) → guardas monotônicos (caminhos `Read(...)` negados e anti-exfiltração dos shells, R15) → hooks `PreToolUse` (`CommandHook`: JSON no stdin, exit 2 bloqueia com o stderr, `block`/`deny`/`ask` no stdout) → aprovação (falha fechado sem interface) → escalonamento de uma chamada (`sandbox_permissions` + `justification`) → execução → hooks `PostToolUse`, rótulo `<untrusted-content>` (R20) e spill (R17). `call_batch` roda as paralelas sob semáforo de 10 com as exclusivas como barreira. Ferramentas: `Read` (`cat -n`, 2.000 linhas, imagens), `Write`/`Edit` (read-before-write, `FS_STALE_VERSION`, escrita atômica com fsync, CRLF preservado), `NotebookEdit`, `Glob`/`Grep` (ignore + grep-searcher; arquivos negados não são lidos), `Bash`/`PowerShell` (one-shot, cwd persistente, `[exit code: N]`, timeout 120/600 s, cancelamento mata a árvore, PATH higienizado, `DSH_*`/`CLAUDE_PROJECT_DIR`, ConstrainedLanguage em read-only), `WebFetch` (só IP público, conexão fixada, redirects de mesma origem, 5 MB/30 s/200k caracteres, HTML→markdown, `prompt` via `Summarizer`), `WebSearch` (Brave, SearXNG), `TodoWrite`, `AskUserQuestion` (pending), `ExitPlanMode` e `working_directory`. As ferramentas de arquivo seguem o modo do sandbox (`SANDBOX_DENIED` com a dica de escalonamento fora do diretório efetivo). O `harness-sandbox` ganhou cancelamento cooperativo (`CommandSpec::cancel_flag`). `check.ps1 -Gnu` verde; 18 testes de aceitação cobrem os 7 cenários e os complementos.
**Decisão (2026-11-02):** o Git Bash (MSYS/Cygwin) não roda sob o token restrito (`NtCreateDirectoryObject`/`CreateFileMapping` negados em integridade baixa e também média). Por escolha do usuário, no Windows o `Bash` roda só com o Job Object e o ambiente sem credenciais, inclusive em `workspace-write`; no `read-only` ele é recusado (`SANDBOX_UNAVAILABLE`) e a dica manda usar o `PowerShell`, que roda confinado.
**Pendente:** `SessionSearch` e as demais ferramentas de histórico (precisam do índice FTS do `harness-store`, Tarefa 08/store); leitura de PDF; `run_in_background`/`BashOutput`/`KillShell` (spec 16); provedores Tavily/Exa/Perplexity; `RunCode` (`harness-script`); motor de permissões completo e carregamento dos hooks de `settings.json`/plugins (`harness-claude-compat`); 🟡 o shell dos hooks no Windows é o `cmd.exe /d /s /c`; 🟡 com proxy do sistema, a fixação de IP do `WebFetch` não se aplica (a checagem de IP público continua).

---

### Tarefa 07 — Loop do agente (turnos, passos, inbox, cancelamento)
**Status:** done
**Lê:** `_reversa_sdd/sdd/agent-loop-sessao.md`
**Constrói:** `crates/harness-core::agent`: driver ReAct em tokio, inbox `next_turn`/`next_step`, checkpoints R4, concorrência com semáforo, pontos de extensão, system prompt por seções, `developer/message` para ferramentas dinâmicas, resume com fechamentos sintéticos
**Pronto quando:** os 6 cenários de aceitação da spec passam com um LLM mock
**Resultado (2026-11-02):** crate novo `crates/harness-agent` (desvio do plano: `harness-core::agent` criaria ciclo, porque `harness-llm` e `harness-tools` já dependem do `harness-core`). Módulos `agent` (driver tokio, `AgentHandle` com `create`/`resume`/`followup`/`steer`/`inject`/`cancel(cause, keep_inbox)`/`when_idle`/`dispose`), `inbox` (claim R11, `agent/inbox/spliced` em toda mutação, `outcome: canceled`), `history` (histórico derivado dos eventos de superfície com `surfaceOp: replace`, `RequestParams` no `request/header`, `rebuild_step_request`, `canonical`, `open_tail`, `transcript`), `prompt` (seções R14, identidade neutra R15, snapshot de runtime user-role R16, system prompt anexado só quando muda R17), `middleware` (`pre_step`, waterfall `request`, `request_error`, `turn_stopping`, `assistant_stream`, `status`) e `model` (`ModelClient`, implementado para o `LlmRuntime`). As três barreiras de flush (R4) são fail-closed; a `assistant/message` é gravada antes do despacho (R3); as chamadas passam pelo pipeline do `harness-tools` com `max_parallel_tool_calls` (novo `call_batch_with`); o cancelamento marca as não despachadas com `ABORTED_BEFORE_DISPATCH` (R10); `developer/message` registra ferramentas adicionadas/removidas com `headerSeq` (U4). 8 testes de aceitação (os 6 cenários + steer e config inválida) e 5 unitários; `check.ps1 -Gnu` verde.
**Pendente:** fork de sessão (R13, Should); `pre_step` com `Enter{messages}` (entra com a compactação); os eventos `llm/retry`/`llm/fallback` do `LlmRuntime` ainda não vão para o log da sessão (falta um `LlmEvents` por sessão); migração de primeira execução do `~/.dsh` (R21b) e sessão em branco reutilizável (U7) entram com o CLI; 🟡 a comparação byte a byte do cenário 1 usa a forma canônica do request (o adapter é função pura dela), não o corpo HTTP; 🟡 imagens em resultados de ferramenta ficam no log, mas o `Content::ToolResult` do `harness-llm` só leva texto ao modelo.

---

### Tarefa 08 — Motor de permissões e aprovações
**Status:** done
**Lê:** `_reversa_sdd/sdd/seguranca-sandbox-permissoes.md`, `_reversa_sdd/sdd/compat-claude-code-totvs.md`
**Constrói:** regras `allow`/`ask`/`deny` no formato Claude Code (globset), modos `default`/`acceptEdits`/`plan`/`bypassPermissions`, monotonicidade, `allow_always` → `settings.local.json`, auditoria no log, confiança de pasta
**Pronto quando:** os cenários de aceitação de permissões das duas specs passam
**Resultado (2026-11-02):** `harness-tools::permissions` com `PermissionRules` (implementa o `PermissionEngine` do pipeline): regras `Tool`/`Tool(espec)` no formato Claude Code (`Bash`/`PowerShell` com `prefixo:*`, curinga e exato, avaliadas por segmento de comando composto; `Read`/`Edit`/`Write` com padrões de caminho `//`, `~/`, `./`; `WebFetch(domain:)`; `mcp__srv[__tool]`; `Task(agente)`; `Skill(nome)`; aliases normalizados), ordem `deny` > plan mode > `ask` > escrita governada > `allow` > modo, e os modos `default`/`acceptEdits`/`plan`/`bypassPermissions` com o mapa para o sandbox. Plan mode restritivo (R11): escrita negada com orientação para o `ExitPlanMode`, shell só com regra `allow`; a aprovação do plano volta para `default`. Aprovações `allow_once`/`allow_session`/`allow_always` (`ApprovalRequest`/`ApprovalDecision`), com `allow_always` gravado no `.claude/settings.local.json` preservando o resto do arquivo (R10). Auditoria `permission/decision`, `approval/asked`, `approval/decided` e `permission/preset` pelo trait `Audit`; falha de gravação vira rejeição (R12) e, no loop, uma decisão fora de turno aberto é inválida (R13). Confiança de pasta (`trust.rs`, `~/.dsh/trusted-folders.json`): hooks do projeto só rodam em pasta confiável (R20). `write_approved_models` (`ApprovedModels::All` por padrão = `"*"`; restrito → `ask` na escrita governada). `AgentHandle::set_permission_mode` grava a troca e avisa o modelo no próximo passo. Correções de integração: o wrapper do PowerShell devolve erros como texto (sem CLIXML) sem marcar falha por stderr de comando nativo, e a detecção de acesso negado reconhece mensagens em português. 10 testes de aceitação de permissões + 1 no loop; `check.ps1 -Gnu` verde.
**Pendente:** carregar e mesclar as regras das settings (`settings.json`, `.local`, `~/.claude`, CLI, política gerenciada) e o `defaultMode` fica com a Tarefa 10 (`harness-claude-compat`), que monta o `RuleSet`; o prompt "Confio nesta pasta" e a recusa de `.mcp.json`/`!cmd`/`env` sem confiança entram com a TUI, o MCP e os comandos; pedidos de subagentes subindo para a sessão raiz (R13) entram com a Tarefa 11; lembrete de repetição (R15, Could); 🟡 a marcação de "escrita governada" usa `governed_paths` (padrões de caminho; vazio = toda escrita), porque a spec não define como skills/agentes são marcados; 🟡 no modo `default`, `WebFetch` e ferramentas MCP pedem aprovação, como no Claude Code.

---

### Tarefa 09 — Compactação de contexto
**Status:** done
**Lê:** `_reversa_sdd/sdd/compactacao-contexto.md`
**Constrói:** medição pelo modelo de destino, pruner, seleção balanceada, resumo em 8 seções com cache quente, offload de imagens, `/compact [instruções]`, gatilho por troca de modelo
**Pronto quando:** os 6 cenários de aceitação da spec passam
**Resultado (2026-11-02):** `harness-agent::compaction` (config com `model_policies`, medição, cauda retida, poda, seleção do trecho, instrução, validação do checkpoint, offload) integrado ao driver. Antes de cada request, a pressão é medida para o modelo de destino do passo (R2), com gatilho `auto` ou `model-switch` quando a janela encolhe; a ordem é poda (`compaction/prune`, aplicada ao montar o histórico, R4) → sumarização do trecho balanceado mais antigo (R5) → passadas extras (`retries`). O resumo usa o último request roteado + a instrução no fim (cache quente, R6), consolida checkpoints anteriores (R7), passa pelo `assert_stable` (R8), é fail-closed em `MAX_TOKENS`, conteúdo não suportado ou seções faltando (R9) e grava `user/message` com `surfaceOp: replace` + `compaction/start|end`; o modelo do resumo passa pela política (`ModelClient::allowed`, R10). Overflow (`CONTEXT_WINDOW_EXCEEDED`) compacta e repete uma vez; `IMAGE_OFFLOAD_REQUIRED` grava `image/offload` e repete sem consumir retry (R13). `AgentHandle::compact(instruções)` com as mensagens estáveis do `/compact`, `context_pressure()` (R15), middleware `pre_compact`/`post_compact` (R11/R12). No `harness-llm`: `Request.estimated_input_tokens` faz o fallback pular candidatos com janela menor gravando `CONTEXT_WINDOW_EXCEEDED`, e `LlmRuntime::check_target` checa a política sem I/O. O `replace` preserva as mensagens de sistema. 9 testes de aceitação + 1 no `harness-llm`; `check.ps1 -Gnu` verde.
**Pendente:** tokenizer por família e uso real do provedor na medição (hoje `ceil(chars/4)`, R1 🟡); o comando `/compact` na TUI e os hooks `PreCompact`/`SessionStart` de verdade (Tarefas 11 e 21, que ligam o middleware); o placeholder do offload ainda não aponta o caminho do anexo (anexos são da Tarefa 20); `compact_region(start, end)` para o SDK.

---

### Tarefa 10 — Compatibilidade Claude Code (`.claude/*`, `CLAUDE.md`, MCP config)
**Status:** done
**Lê:** `_reversa_sdd/sdd/compat-claude-code-totvs.md`, `_reversa_sdd/totvs-agp-inventory.md`
**Constrói:** `crates/harness-claude-compat`: instruções (cadeia + U1), agentes, skills, comandos, settings mescladas, importação de MCP com cofre, aliases, `write_approved_models = "*"`, `dsh compat doctor|import-mcp`
**Pronto quando:** o `compat doctor` no Totvs AGP lista 7 agentes, as skills, 43 allow/3 deny e o gpflow, sem expor segredos
**Resultado (2026-11-02):** `crates/harness-claude-compat` com `frontmatter` (YAML com leitura tolerante `chave: valor`), `instructions` (cadeia `$DSH_HOME/AGENTS.md` → `~/.agents/AGENTS.md` → `~/.claude/CLAUDE.md` → raiz..cwd com `CLAUDE.md`/`CLAUDE.local.md`/`AGENTS.md`/`AGENTS.local.md`, dedupe por conteúdo, 64 KiB/1 MiB, imports `@` até 5 níveis sem sair do workspace e de `~`, `discover_for` para R2, erro explícito de I/O), `agents` (`name`/`description`/`tools`/`model`/`memory`, projeto vence usuário), `skills` (ranking `.dsh` > `.claude` > `.agents` > extras > usuário; pasta-espelho um nível abaixo vira duplicata com aviso e vence o caminho menor; descrição de catálogo ≤ 500), `commands` (`dir:nome`), `settings` (usuário < projeto < local < CLI < gerenciada; arrays concatenados num `RuleSet` com a base de cada camada, `defaultMode`, hooks por evento com matcher normalizado e `from_project`, `env` com detecção de segredo, flags de `.mcp.json`; camada ilegível vira aviso), `mcp` (`.mcp.json` com aprovação, `~/.claude.json` global e por projeto, valores em `Secret`; `import` grava `~/.dsh/mcp-servers.json` com `${secret:mcp/<srv>.<header>}` e o valor no cofre, sem tocar o `~/.claude.json`) e `workspace` (`[compat] write_approved_models`/`governed_paths` do `.dsh/workspace.toml`; relatório do doctor). CLI: `dsh compat doctor [--path]` (com os aliases do `providers.toml`) e `dsh compat import-mcp [--dry-run]`. 8 testes de aceitação numa réplica sintética + execução real autorizada no `D:\Projetos\Totvs AGP\VSCode` (com `DSH_HOME` temporário): 7 agentes, 23 skills do projeto + aviso do espelho `skill-ellevo-db/ellevo-db`, projeto com 19 + 24 = 43 allow e 3 deny, `gpflow` (http, header `Authorization` oculto); nenhum valor de `.env` ou do token exposto. O total de 118 allow inclui 75 regras do `~/.claude/settings.json` do usuário, criadas depois do inventário. `check.ps1 -Gnu` verde.
**Pendente:** watch das fontes `.claude/*` com catálogo substituto no próximo passo (o loader recarrega por chamada; o watcher entra com a integração da sessão); ligar o `Workspace` ao loop (seções de prompt, `RuleSet` no `PermissionRules`, hooks no pipeline, máscaras dos agentes) entra com a Tarefa 11 e a 21; memória `memory: project` (Tarefa 15); plugins (Tarefa 16); confirmação de `env` com cara de segredo na primeira vez e aprovação de `.mcp.json` na TUI; teste de aliases e do gate de rede no doctor depende do `providers.toml` do usuário.

---

### Tarefa 11 — Subagentes, skills, hooks e workflow
**Status:** done
**Lê:** `_reversa_sdd/sdd/subagentes-skills-hooks.md`
**Constrói:** `Task`/`Agent` (spawn, fork, contínuo, profundidade 1), `Skill`, hooks (eventos, `updatedInput`, limite de continuação), `crates/harness-script` (rquickjs) com `Workflow`, `Goal`
**Pronto quando:** os 6 cenários de aceitação da spec passam
**Resultado (2026-11-02):** `harness-agent::subagent` (`SubagentRuntime`, `TaskTool`, `WorkflowTool`, `SubagentDef::from_compat`, embutidos `general-purpose`/`Explore`/`Plan`): cada filho numa sessão própria com `parentSession` (R1), máscara = interseção pai ∩ agente aplicada no próprio registro (`ToolRegistry::retain`, R2), profundidade padrão 1 sem `Task`/`Workflow` nos filhos (R3), `Task` paralela (R4), modelo chamada > agente > pai (R5), memória `MEMORY.md` (≤ 25 KB) no prompt do filho, cancelamento do pai repassado, aprovações pela mesma `Interaction` com o nome do agente (R9). `harness-agent::skills` (`Skill`, catálogo durável sem as `disable-model-invocation`, `/nome` via `invoke_skill`, corpo relido a cada carga, R11/R13) e `harness-agent::goal` (`create_goal`/`get_goal`/`update_goal`, `goal/change`, driver de rodadas só com goal ativo e armado, desarme no resume e `rearm_goal`, R23). Hooks reescritos no `harness-tools` (10 eventos, `HookReply` com `block`/`ask`/`additionalContext`/`updatedInput`/`continue:false`, exit 2 bloqueia, outros códigos não bloqueiam, timeout padrão 600 s, sandbox da sessão, auditoria `hook/invoked`/`hook/result` com stderr ≤ 500): `updatedInput` revalidado no schema e reavaliado nas permissões e guardas (R20); no loop, `SessionStart` (startup/resume/compact), `SubagentStart`, `UserPromptSubmit`, `Stop`/`SubagentStop` (máx. 3 continuações forçadas), `PreCompact` e `SessionEnd`. `crates/harness-script`: QuickJS (`rquickjs` 0.9) com top-level `await`, `agent`/`parallel`/`pipeline`/`phase`, `export const meta` aceito, limites de 1.000 agentes, 4.096 itens e 50.000 caracteres, falha comum vira `null`, erro com linha e coluna e cancelamento dos agentes iniciados. Saída de filhos decodificada em UTF-8 ou, no Windows, no code page OEM (`harness_sandbox::decode_output`), porque o `cmd` sem console escreve em OEM. 7 testes de aceitação no loop, 1 de `updatedInput` no `harness-tools` e 2 no `harness-script`; `check.ps1 -Gnu` verde.
**Pendente:** provedores `fork`, contínuo (`SendMessage`/`interrupt_agent`/`ListAgents`), `acp:` e `claude-code`; `Task` em segundo plano (jobs, Tarefa 12); `allowed-tools` da skill como `allow` temporário (R12); hooks `http`/`prompt`/`agent`/`mcp_tool` e o evento `Notification`; watch da config de hooks (R18); concorrência padrão do workflow = número de CPUs (`std::thread::available_parallelism`); montagem real no CLI (ligar `harness-claude-compat` → `SubagentRuntime`, skills, hooks e goal) entra com a Tarefa 21; memória gravável com permissão implícita e espelho no GBrain (Tarefa 15).

---

### Tarefa 12 — Jobs, agenda e terminais
**Status:** done
**Lê:** `_reversa_sdd/sdd/jobs-schedule-terminal.md`
**Constrói:** registro de jobs (ring de saída, dois cursores, `job/registered`/`settled`), wakeup limitado, agenda em domínio próprio (`after`/`at`/`every`/`cron`, `schedule_update`, framing U5), terminais ConPTY
**Pronto quando:** os 6 cenários de aceitação da spec passam
**Resultado (2026-11-02):** `harness-tools::jobs` (`JobRegistry` no `ToolEnv`, dono = sessão, até 10 ativos, ids `bash-N`/`pwsh-N`/`agent-N`/`terminal-N`, `BashOutput` com aliases `job_output`/`TaskOutput` devolvendo só a saída nova e `[status: …]`, `job_list`, `KillShell`/`job_kill`/`TaskStop` que espera o fim) com `Bash`/`PowerShell` e `Task` em `run_in_background` e `job/registered`; no loop, `job/settled` e aviso `<task-notification>` que abre turno com o agente ocioso até 3 seguidos (zerado por mensagem do usuário) ou entra no próximo passo (R4); no resume, jobs sem `job/settled` fecham como `failed` interrompido (R5/U2); `dispose` mata os jobs e fecha os terminais do dono. `harness-agent::schedule`: domínio `~/.dsh/schedule.json` (U3, ilegível falha por inteiro), `after`/`at` (RFC 3339 ou hora local + fuso, gap de horário de verão recusado e sobreposição no primeiro instante)/`every` (≥ 300 s, alinhado à âncora)/`cron` (croner + chrono-tz, intervalo efetivo ≥ 5 min, fuso padrão do sistema), `schedule_create`/`list`/`delete`/`update` (aliases `CronCreate`/`CronList`/`CronDelete`) só na sessão raiz (R15/U6), entrega só com o agente ocioso, sem backlog, histórico de 30 dias/200 entregas, mensagem com framing U5 e conteúdo `<untrusted-content source="schedule">`; relógio injetável (`Clock`). `harness-tools::terminal`: `terminal_open`/`send`/`read`/`signal`/`close`/`list`, posse pelo agente criador, Job Object, ambiente sem credenciais, recusado em `read-only`. 6 testes de aceitação; `check.ps1 -Gnu` verde.
**Decisão (2026-11-02):** o usuário escolheu tratar o terminal como o `Bash` (sem token restrito; Job Object + ambiente sem credenciais; recusado em `read-only`). 🟡 O ConPTY (`portable-pty` 0.9) não funcionou neste ambiente (nem `cmd /c echo` termina sob o toolchain GNU sem console), então o terminal usa **pipes**: `bash -i` com o marcador no `PS1` (stderr) e `pwsh -Command -` com uma linha marcadora após cada envio; uso orientado a linhas (R19).
**Pendente:** ConPTY (reavaliar com o toolchain MSVC/OpenConsole, U7) e `terminal_signal` com Ctrl+C real (por pipe não chega; hoje orienta `terminal_close`); streaming da saída dos jobs (hoje a saída chega no fim; `job_output` devolve só a parte nova); ring de saída com leitura absoluta para observadores (U1); `dsh schedule install-task` (Agendador do Windows, R17) e toast (R16); modo `quiet` dos avisos.

---

### Tarefa 13 — Cliente MCP e integrações gerenciadas
**Status:** done
**Lê:** `_reversa_sdd/sdd/mcp-cliente-gerenciado.md`
**Constrói:** `crates/harness-mcp` (rmcp stdio/http, reconexão, progresso, cancelamento, anotações, `${secret:}`), `crates/harness-integrations` (detecção de runtimes, install/update/doctor de REA, GBrain, Playwright, Chrome DevTools, cua-driver, csharp-ls, Gemini CLI; `NODE_OPTIONS=--use-system-ca`)
**Pronto quando:** os 6 cenários de aceitação da spec passam
**Resultado (2026-11-02):** `harness-mcp` sobre rmcp 3.8 (stdio com ambiente higienizado + Job Object, Streamable HTTP pelo reqwest/rustls do dsh; `sse` legado lido como http): config Claude Code com `${VAR}`, `${VAR:-padrão}` e `${secret:escopo/id}` (cofre), nomes `[A-Za-z0-9_-]{1,32}`, precedência gerenciadas < `~/.claude.json` < `.mcp.json` < plugins < `~/.dsh/mcp-servers.json`; máquina `connecting → initialized → lost → reconnecting → initialized|failed` com backoff 0,5 s ×2 até 30 s, 10 tentativas (R2); falha não fatal (R1); gate de rede pelos `guards` `mcp:<nome>` do `NetworkGate` (R13); `.mcp.json` não aprovado fica `needs_approval` (R14); timeout por servidor (padrão 60 s, REA 330 s) com `TOOL_TIMEOUT` e `cancelled`; progresso por `progressToken` limitado a 100 ms (R5); cancelamento do turno envia `notifications/cancelled` e preserva a observação parcial (R6); `structuredContent` validado contra `outputSchema` (R7); saída rotulada não confiável, reforçada por `_meta.provenance.untrusted_content` (R8); `readOnlyHint` → paralela, `destructiveHint` → `ask` na 1ª chamada da sessão salvo `allow` explícito, inclusive em `bypassPermissions` (R9, novo `Tool::destructive` e `PermissionEngine::explicitly_allowed`); instruções até 32 KiB (R10); resultado até 10 MiB (R11); `ListMcpResources`/`ReadMcpResource`. `harness-integrations`: catálogo das 7 integrações, detecção de Node/Bun/.NET/Chrome e pacotes npm globais, regra do REA (22.x ≥ 22.23.4, 24.x ≥ 24.22.1 ou ≥ 26.12.1), `missing_runtime` com a instrução winget (R15), instalação pelo canal oficial com saída visível e `NODE_OPTIONS=--use-system-ca` (R16/R16b), estado em `~/.dsh/integrations/state.json`, registro como servidor gerenciado reaproveitando a entrada do Claude Code (R17), REA com perfil compacto (R20), obrigatórias em vermelho (R21), .NET oferece csharp-ls (R22). CLI `dsh mcp list|get|add|remove|test` e `dsh integrations list|install|update|remove|doctor`. 9 testes de aceitação + 5 unitários; `check.ps1 -Gnu` verde.
**Decisão (2026-11-02):** 🟡 computer use (Cua Driver) é instalação manual: o dsh detecta `cua-driver` e registra `cua-driver mcp` (como o dsh TS), sem instalar a extensão AGPL `cua-perception`. Node 25.x (o instalado hoje, 25.9.0) não atende a regra do REA.
**Pendente:** OAuth do MCP remoto (descoberta + PKCE, Should); prompts MCP como `/mcp__srv__prompt` (Could); atualização automática semanal ociosa (R18, Could); registro do plugin do REA (Tarefa 16); `gbrain init --pglite` e instruções brain-first (Tarefa 15); matriz csharp-ls × .NET 7 (Tarefa 14); `/mcp` na TUI, eventos de progresso no ACP/SDK e injeção das ferramentas/instruções na sessão (Tarefa 21).

---

### Tarefa 14 — LSP (C# e índice PL/SQL)
**Status:** done
**Lê:** `_reversa_sdd/sdd/lsp-csharp-plsql.md`
**Constrói:** ferramenta `LSP`, cliente LSP, projeto auxiliar para `.cs` soltos, diagnósticos pós-edição, índice PL/SQL com tree-sitter + fallback
**Pronto quando:** os 5 cenários de aceitação da spec passam
**Alerta:** .NET 10 recomendado para o `csharp-ls` atual
**Resultado (2026-11-02):** `harness-tools::lsp`: ferramenta `LSP` somente leitura com as 10 operações (R1, orientação do R5 na descrição), coordenadas 1-based, saída `caminho:linha:coluna` agrupada por arquivo (≤ 100 locais, ≤ 16.000 caracteres), timeout de 60 s com reinício do servidor; cliente LSP próprio (JSON-RPC `Content-Length`, respostas neutras às requisições do servidor, `publishDiagnostics`); servidores sob demanda por linguagem e raiz, encerrados após 10 min ociosos, consultas em série e `didOpen`/`didChange` a cada consulta (R2/R3); ambiente sem `*KEY*`/`*PASSWORD*`/`*SECRET*`/`*TOKEN*`/`DSH_*` e Job Object (R4); config `~/.dsh/lsp.toml` + `[lsp.*]` do `.dsh/workspace.toml`, com csharp-ls, rust-analyzer, typescript-language-server, pyright e gopls pré-configurados. C#: raiz pelo `.csproj`/`.sln` ou projeto auxiliar `.dsh/lsp/csharp/<hash>/Scratch.csproj` (TargetFramework pelo maior SDK, `reference_dirs` como `<Reference>`) sem tocar nos `.cs` (R9); diagnósticos `RM.*` não resolvidos filtrados sem `reference_dirs`, com aviso de resolução parcial (R10). PL/SQL: índice tree-sitter (`tree-sitter-plsql-sqry` 33.0.1, gramática AndreasMaierDe/tree-sitter-plsql, MIT) de pacotes, bodies, procedures, functions, triggers, types, cursores, variáveis e parâmetros com posição e assinatura, fallback por regex marcado `textual` (R13), SQLite `~/.dsh/index/plsql/<ws-hash>.db` incremental por mtime/tamanho e SHA-256 (R12), resolução `pacote.sub`/`schema.objeto`/nome solto com escopo local → pacote → workspace e todos os candidatos na ambiguidade, body antes da spec (R14), referências e hierarquia de chamadas aproximadas fora de comentários, diagnósticos só de sintaxe (R15), `catalog_enrich = off` responde "not found in the workspace" sem consulta ao banco. Diagnósticos pós-`Edit`/`Write` no pipeline: base antes da edição, só os novos (≤ 20), espera de até 3 s (R6). `harness-integrations`: matriz csharp-ls × .NET (0.10.0/net7, 0.16.0/net8, 0.20.0/net9, 0.28.0/net10), doctor com as duas opções (instalar .NET 10 ou fixar a versão compatível) e `install` fixando a versão com SDK antigo (R8). 5 cenários de aceitação + testes unitários; `check.ps1 -Gnu` verde.
**Decisão (2026-11-02):** 🟡 cliente LSP próprio em vez de `async-lsp`/`tower-lsp` (só o subconjunto usado, sem dependências extras). 🟡 Contornos da gramática de 2022: `CREATE PROCEDURE|FUNCTION` avulso exige um segundo `END <nome>;` e não aceita `END;` (o dsh completa o nome e repete o fechamento, na mesma linha, só para o parse); `CREATE VIEW`, `:new/:old` e o `;` após o `END` de trigger não geram erro (lacunas filtradas, views indexadas pelo fallback); identificador de largura zero é tratado como token faltando.
**Pendente:** validar no piloto com os 181 `.sql` e os `.cs` reais do Totvs AGP (uso dos dados liberado caso a caso) e com o csharp-ls de verdade (hoje SDK 7.0.202 → 0.10.0 ou instalar o .NET 10); enriquecimento pelo catálogo Oracle (R16, Could); XOML (R11, Won't); wiring do `Lsp` na sessão e diagnósticos na TUI (Tarefa 21).

---

### Tarefa 15 — Memória (por agente, automática, GBrain)
**Status:** done
**Lê:** `_reversa_sdd/sdd/memoria-gbrain.md`
**Constrói:** carga de `MEMORY.md` por contexto, gravação com permissão implícita, espelhamento para o GBrain, escopo por workspace, seção brain-first v2, backup
**Pronto quando:** os 5 cenários de aceitação da spec passam
**Resultado (2026-11-02):** `harness-claude-compat::memory`: slug do Claude Code (`d:\dsh` → `d--dsh`, reaproveitando a pasta existente em outra caixa), `MEMORY.md` limitado a 200 linhas/25 KB (R1), seção de prompt com o formato `name`/`description`/`metadata.type`, regras de gravação (R5, R7) e o aviso de verificar o que a memória cita (R3); `AgentDeps.memory` (`MemoryContext`) coloca a memória automática na sessão principal e a do agente no subagente, e nada num subagente sem `memory` (R2). `harness-tools`: `SessionState.memory_dir`/`memory_roots`; `Write`/`Edit` no diretório de memória da sessão passam sem aprovação depois das regras `deny` (inclusive no sandbox `workspace-write`, fora do projeto), um subagente não grava na memória de outro agente e o modo `plan` continua sem edições (R6); detector de segredos reutilizável (`secrets::find`) bloqueia a gravação (R7); `WriteObserver` avisado depois de cada `Write`/`Edit`. `harness-integrations::gbrain`: config `[gbrain]`/`[memory.mirror]` do `~/.dsh/memory.toml` e `[memory]` do workspace; bloco brain-first v2 lido do GBrain instalado (`setup chatgpt --instructions-only --dry-run`, cache por versão, texto oficial embutido como reserva) com a fonte do workspace em `source_ids` (R4); aviso de GBrain indisponível com a memória markdown funcionando (R16); fontes registradas com `gbrain sources add <id> --path <dir>`, as de `scope = "workspace"` com `--no-federated` (fora da busca padrão), registro em `~/.dsh/gbrain/sources.json` (R12/R13); `ScopeFilter` no MCP `gbrain` recusa `source_id(s)` restritos e tira dos resultados o que vier deles (R13); `gbrain sync --source <id> --no-pull` depois de cada gravação (`MirrorObserver`) e no arranque, com `--no-embed` para fonte `local-only` se os embeddings forem remotos (R11/R12); chaves de embeddings remotos fora do ambiente de todo comando e do servidor GBrain (R11); registro `serve --surface starter --advertise verbs` (R9), `gbrain init --pglite` (`--no-embedding` sem Ollama) na instalação (R8); Memorable nunca ligado pelo dsh, `memorable = true` só gera aviso (R10); backup com `gbrain export` + cópia do PGLite, mantendo 14 (R15). 5 cenários de aceitação + testes unitários; `check.ps1 -Gnu` verde.
**Decisão (2026-11-02):** 🟡 o escopo usa o mecanismo nativo do GBrain (`--no-federated`) mais o filtro do dsh, em vez de só filtrar resultados. 🟡 Fontes: `<gbrain_source>` = `.claude/agent-memory` e `<gbrain_source>-auto` = memória automática do cwd; sem `gbrain_source`, `mem-<slug>`.
**Pendente:** validar com um brain real (o GBrain instalado não foi inicializado: `gbrain init` grava em `~/.gbrain` e só roda em `dsh integrations install gbrain`); confirmar se `sources add` aceita pasta sem git com dados; captura automática com opt-in (R4b) e reforço brain-first por turno; agendar o backup diário e a reconciliação no arranque, e ligar `MemoryContext`, `ScopeFilter` e `MirrorObserver` na sessão (Tarefa 21); oferecer `gbrain setup claude-code` no doctor (R14); `takes_supersede` nas contradições é do próprio GBrain.

---

### Tarefa 16 — Plugins e marketplaces
**Status:** pending
**Lê:** `_reversa_sdd/sdd/plugins-marketplace.md`
**Constrói:** leitura de `~/.claude/plugins`, `dsh plugin marketplace add|list|update|remove`, `install|enable|disable|uninstall|update|validate`, cache versionado, confiança para hooks e MCP, namespaces
**Pronto quando:** os 5 cenários de aceitação da spec passam (com `coreos/ai-helpers`)

---

### Tarefa 17 — Sincronização VSCode → dsh e CatalogoSKILLs
**Status:** pending
**Lê:** `_reversa_sdd/sdd/catalog-sync.md`
**Constrói:** `dsh sync add|status|run|log`, watcher + reconciliação, manifesto SHA-256, escopo global com namespace, varredura de segredos, `dsh catalog build` (`.zip` + `.7z` reprodutíveis)
**Pronto quando:** os 5 cenários de aceitação da spec passam

---

### Tarefa 18 — REA
**Status:** pending
**Lê:** `_reversa_sdd/sdd/rea.md`
**Constrói:** `dsh rea <alvo>` e `/rea`, classificação do alvo, pré-requisitos, sessão com skill, relatório + evidências, evento `rea/analysis`
**Pronto quando:** os 5 cenários de aceitação da spec passam

---

### Tarefa 19 — Browser use, computer use e agent teams
**Status:** pending
**Lê:** `_reversa_sdd/sdd/browser-computer-agent-teams.md`
**Constrói:** provedores exclusivos (Playwright, Chrome DevTools, cua-driver), `attach` com confirmação, consentimento por aplicação, as 9 ferramentas de time, mailbox e quadro duráveis, worktree
**Pronto quando:** os 6 cenários de aceitação da spec passam
**Alerta:** o `cua-driver-rs` no Windows só tem builds nightly

---

### Tarefa 20 — Retenção de anexos, spill e sessões
**Status:** pending
**Lê:** `_reversa_sdd/sdd/retencao-anexos-spill.md`
**Constrói:** GC mark-and-sweep de anexos, limpezas por política, `session_max_days`, `dsh storage usage|gc`, `dsh sessions delete`
**Pronto quando:** os 4 cenários de aceitação da spec passam

---

### Tarefa 21 — Superfícies: TUI, headless, ACP e SDK
**Status:** pending
**Lê:** `_reversa_sdd/sdd/superficies-tui-headless-acp-sdk.md`
**Constrói:** `crates/harness-tui` (ratatui), `dsh -p` com `stream-json` compatível, `crates/harness-acp` (servidor + `session/load`, modos e comandos), `crates/harness-sdk` (JSON-RPC), sinais e códigos de saída
**Pronto quando:** os 6 cenários de aceitação da spec passam

---

### Tarefa 22 — Upstream-sync (`dsh upstream`)
**Status:** pending
**Lê:** `_reversa_sdd/sdd/upstream-sync.md`, `_reversa_sdd/upstream/ledger.md`
**Constrói:** `dsh upstream status|fetch|diff|triage|mark|baseline`, ledger JSON, rejeição automática por padrões de envio de dados
**Pronto quando:** os 5 cenários de aceitação da spec passam; o ledger do ciclo 1 é importado

---

### Tarefa 23 — Release, assinatura minisign e `dsh update`
**Status:** pending
**Lê:** `_reversa_sdd/sdd/build-release.md`
**Constrói:** `xtask dist|sign`, vendoring filtrado, `SHA256SUMS` + `.minisig`, `dsh-verify.ps1`, `dsh update` sem Mark-of-the-Web, `THIRD_PARTY_NOTICES` (MIT DeepSeek)
**Pronto quando:** os 4 cenários de aceitação da spec passam

---

### Tarefa 24 — Fluxos de usuário (end-to-end)
**Status:** pending
**Lê:** `_reversa_sdd/user-stories/fallback-quando-claude-indisponivel.md`, `_reversa_sdd/user-stories/trabalhar-no-totvs-agp.md`, `_reversa_sdd/user-stories/nova-skill-replicada.md`, `_reversa_sdd/user-stories/analise-com-rea.md`
**Constrói:** testes end-to-end e ajustes de integração dos 4 fluxos
**Pronto quando:** todos os critérios de pronto das 4 histórias estão marcados

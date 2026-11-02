# Reconstruction Plan — deepseek-harness → dsh (Rust)

**Stack:** Rust (edition 2021, resolver 3), Cargo workspace com crates `harness-*` e binário `dsh`; tokio, reqwest (rustls + certificados do SO), serde, rusqlite (FTS5), zstd, rmcp, agent-client-protocol, ratatui, portable-pty, windows-rs, keyring, rquickjs, tree-sitter. Alvo principal: `x86_64-pc-windows-msvc`.
**Diretório do código:** `D:\dsh\dsh-rs` (repositório novo; o `deepseek-harness` não é alterado).
**Gerado em:** 2026-11-01
**Status:** 25 tarefas | 6 concluídas | 19 pendentes

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
**Status:** in_progress
**Lê:** `_reversa_sdd/sdd/ferramentas-claude-code.md`
**Constrói:** `crates/harness-tools`: contrato `ToolDef`, pipeline, Read/Write/Edit/NotebookEdit/Glob/Grep/Bash/PowerShell/WebFetch/WebSearch/TodoWrite/AskUserQuestion/ExitPlanMode/SessionSearch/`working_directory`, aliases minúsculos, read-before-write, spill, guarda anti-exfiltração de `.env`
**Pronto quando:** os 7 cenários de aceitação da spec passam

---

### Tarefa 07 — Loop do agente (turnos, passos, inbox, cancelamento)
**Status:** pending
**Lê:** `_reversa_sdd/sdd/agent-loop-sessao.md`
**Constrói:** `crates/harness-core::agent`: driver ReAct em tokio, inbox `next_turn`/`next_step`, checkpoints R4, concorrência com semáforo, pontos de extensão, system prompt por seções, `developer/message` para ferramentas dinâmicas, resume com fechamentos sintéticos
**Pronto quando:** os 6 cenários de aceitação da spec passam com um LLM mock

---

### Tarefa 08 — Motor de permissões e aprovações
**Status:** pending
**Lê:** `_reversa_sdd/sdd/seguranca-sandbox-permissoes.md`, `_reversa_sdd/sdd/compat-claude-code-totvs.md`
**Constrói:** regras `allow`/`ask`/`deny` no formato Claude Code (globset), modos `default`/`acceptEdits`/`plan`/`bypassPermissions`, monotonicidade, `allow_always` → `settings.local.json`, auditoria no log, confiança de pasta
**Pronto quando:** os cenários de aceitação de permissões das duas specs passam

---

### Tarefa 09 — Compactação de contexto
**Status:** pending
**Lê:** `_reversa_sdd/sdd/compactacao-contexto.md`
**Constrói:** medição pelo modelo de destino, pruner, seleção balanceada, resumo em 8 seções com cache quente, offload de imagens, `/compact [instruções]`, gatilho por troca de modelo
**Pronto quando:** os 6 cenários de aceitação da spec passam

---

### Tarefa 10 — Compatibilidade Claude Code (`.claude/*`, `CLAUDE.md`, MCP config)
**Status:** pending
**Lê:** `_reversa_sdd/sdd/compat-claude-code-totvs.md`, `_reversa_sdd/totvs-agp-inventory.md`
**Constrói:** `crates/harness-claude-compat`: instruções (cadeia + U1), agentes, skills, comandos, settings mescladas, importação de MCP com cofre, aliases, `write_approved_models = "*"`, `dsh compat doctor|import-mcp`
**Pronto quando:** o `compat doctor` no Totvs AGP lista 7 agentes, as skills, 43 allow/3 deny e o gpflow, sem expor segredos

---

### Tarefa 11 — Subagentes, skills, hooks e workflow
**Status:** pending
**Lê:** `_reversa_sdd/sdd/subagentes-skills-hooks.md`
**Constrói:** `Task`/`Agent` (spawn, fork, contínuo, profundidade 1), `Skill`, hooks (eventos, `updatedInput`, limite de continuação), `crates/harness-script` (rquickjs) com `Workflow`, `Goal`
**Pronto quando:** os 6 cenários de aceitação da spec passam

---

### Tarefa 12 — Jobs, agenda e terminais
**Status:** pending
**Lê:** `_reversa_sdd/sdd/jobs-schedule-terminal.md`
**Constrói:** registro de jobs (ring de saída, dois cursores, `job/registered`/`settled`), wakeup limitado, agenda em domínio próprio (`after`/`at`/`every`/`cron`, `schedule_update`, framing U5), terminais ConPTY
**Pronto quando:** os 6 cenários de aceitação da spec passam

---

### Tarefa 13 — Cliente MCP e integrações gerenciadas
**Status:** pending
**Lê:** `_reversa_sdd/sdd/mcp-cliente-gerenciado.md`
**Constrói:** `crates/harness-mcp` (rmcp stdio/http, reconexão, progresso, cancelamento, anotações, `${secret:}`), `crates/harness-integrations` (detecção de runtimes, install/update/doctor de REA, GBrain, Playwright, Chrome DevTools, cua-driver, csharp-ls, Gemini CLI; `NODE_OPTIONS=--use-system-ca`)
**Pronto quando:** os 6 cenários de aceitação da spec passam

---

### Tarefa 14 — LSP (C# e índice PL/SQL)
**Status:** pending
**Lê:** `_reversa_sdd/sdd/lsp-csharp-plsql.md`
**Constrói:** ferramenta `LSP`, cliente LSP, projeto auxiliar para `.cs` soltos, diagnósticos pós-edição, índice PL/SQL com tree-sitter + fallback
**Pronto quando:** os 5 cenários de aceitação da spec passam
**Alerta:** .NET 10 recomendado para o `csharp-ls` atual

---

### Tarefa 15 — Memória (por agente, automática, GBrain)
**Status:** pending
**Lê:** `_reversa_sdd/sdd/memoria-gbrain.md`
**Constrói:** carga de `MEMORY.md` por contexto, gravação com permissão implícita, espelhamento para o GBrain, escopo por workspace, seção brain-first v2, backup
**Pronto quando:** os 5 cenários de aceitação da spec passam

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

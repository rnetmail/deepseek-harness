# Ledger upstream-sync — ciclo 1: 0.1.6-alpha.1 → dsh-v0.2.1-rc.5

> Gerado em 2026-11-01 a partir de um clone raso **só para leitura** da tag `dsh-v0.2.1-rc.5`, no scratchpad, sem nenhuma execução. Processo definido em [`sdd/upstream-sync.md`](../sdd/upstream-sync.md).
> Estado: **confirmado pelo usuário em 2026-11-01**. A base passou para `dsh-v0.2.1-rc.5`. As specs dos itens `port`/`adapt` já foram atualizadas, cada uma com uma seção "Atualização upstream 0.2.1-rc.5". A base só avança com a confirmação do usuário (R9).

## Resumo
| Medida | Valor |
|---|---|
| Versão do upstream | `0.2.1-rc.5` (`package.json`); alpha paralela `0.2.2-alpha.2` só acompanhada |
| Formato de sessão | **v3 → v4** (`SESSION_FORMAT_VERSION = 4`; novo pacote `session-format-v3-to-v4`) |
| Pacotes | 284 → 343 (76 novos, 17 removidos ou movidos) |
| Notas de design novas (sem `zh`) | 272 (104 arquivadas no próprio ciclo; **168** implementadas ou propostas, triadas abaixo) |
| Decisões sugeridas (168) | **port 38 · adapt 25 · skip 94 · reject 11** |

## Mudanças de maior impacto
| # | Mudança upstream | Decisão | Spec atualizada |
|---|---|---|---|
| 1 | Sessão **v4**: resultado de ferramenta como mensagem `role: 'tool'`; fontes de mensagem do produtor; `developer/message` para mudanças de ferramentas; validação obrigatória de relações; política de versão por compatibilidade do leitor | port | `agent-loop-sessao.md` |
| 2 | `working_directory`: o cwd efetivo por sessão é registrado no log, e o projeto original continua sendo a raiz | port | `seguranca-sandbox-permissoes.md`, `ferramentas-claude-code.md` |
| 3 | Jobs: um ring de saída por job, dois cursores, `progress`; fatos `job/registered`/`job/settled` persistidos com `JobId` UUID | port | `jobs-schedule-terminal.md` |
| 4 | Agenda: armazenamento do Host (não mais no log), status ativo e inativo, remoção definitiva, histórico limitado (30 dias / 200), framing "This is a scheduled message from the user", ferramentas por preset (+`schedule_update`) | port | `jobs-schedule-terminal.md` |
| 5 | Apresentação **native ou ptc** por agente (`both` removido); PTC persistente opt-in (`persistent-in-memory`) sob o mesmo sandbox | port | `ferramentas-claude-code.md` |
| 6 | Credencial armazenada vence a variável de ambiente; o headless usa env/`.env` | port | `seguranca-sandbox-permissoes.md` |
| 7 | Sandbox Windows: confinamento de **delete** com integridade obrigatória (mandatory integrity) | port | `seguranca-sandbox-permissoes.md` |
| 8 | Instruções globais: `~/.dsh/AGENTS.md` e depois `~/.agents/AGENTS.md` | port | `compat-claude-code-totvs.md` |
| 9 | Times: mensagens na inbox do destinatário; identidade do colega no contexto de runtime; **worktree** explícito | adapt | `browser-computer-agent-teams.md` |
| 10 | CUA nativo (objetos, consentimento por aplicação, helper compartilhado) | adapt | `browser-computer-agent-teams.md` |
| 11 | Presets declarativos com registro; ativações de subagente unificadas; hooks Claude Code movidos para `experimental/` | adapt | `subagentes-skills-hooks.md` |
| 12 | Mercado de plugins próprio do dsh (índice + Discover); ponte para **Claude Code mods** (alpha) | adapt | `plugins-marketplace.md` |
| 13 | Upload limitado do log de sessão para a DeepSeek; log via OTel; telemetria de produto; login de conta DeepSeek; catálogo, busca e fetch DeepSeek | **reject** | — (ADR 101) |
| 14 | Perguntas ao usuário com timeout e duas resoluções | port | `ferramentas-claude-code.md` |
| 15 | Terminais: tolerância para a cauda do prompt do pwsh; ConPTY com o OpenConsole embarcado | port | `jobs-schedule-terminal.md` |

## Pacotes novos relevantes
`session/session-format-v3-to-v4`, `session/working-directory`, `session/tool-working-directory`, `experimental/worktree` + `tool-worktree`, `experimental/claude-code-mods`, `experimental/cua` (+`cua-macos`), `boot/plugin-market` + `util/plugin-market-index`, `preset/agent-preset-registry`, `schedule/tool-schedule`, `skill/tool-workspace-dependencies`, `deliverables/workspace-changes`.
**Rejeitados:** `telemetry/otel`, `host/product-telemetry-otel`, `client/product-analytics`, `credentials/deepseek-account*`, `llm/llm-deepseek-account`, `web/web-deepseek-api`.
**Ignorados:** UI web e desktop, Office/LibreOffice, voz e tradução de raciocínio.

## Itens triados
| Decisão | Grupo | Nota | Spec | Motivo |
|---|---|---|---|---|
| port | implemented/architecture | `2026-09-03-jobs-seam-consolidation` — One output ring, one projection, one event stream for background jobs | `sdd/jobs-schedule-terminal.md` | jobs, agenda, terminal |
| skip | implemented/architecture | `2026-09-08-desktop-welcome-window-material` — Desktop welcome window material | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| port | implemented/architecture | `2026-09-09-producer-owned-message-sources` — Producer-owned message sources at the V4 edge | `sdd/agent-loop-sessao.md` | formato de sessão v4 / persistência |
| skip | implemented/architecture | `2026-09-10-component-factories-and-local-slots` — Reusable component factories with local slots | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| skip | implemented/architecture | `2026-09-10-desktop-web-wrapper` — Run Desktop through the shared Web application | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| port | implemented/architecture | `2026-09-13-session-working-directory` — Session working directory | `sdd/seguranca-sandbox-permissoes.md` | segurança, credenciais, cwd |
| adapt | implemented/architecture | `2026-09-14-current-profile-plugin-management` — Current-profile plugin management shares CLI transactions | `sdd/plugins-marketplace.md` | mercado de plugins do dsh (formato próprio) × formato Claude Code |
| reject | implemented/architecture | `2026-09-14-deepseek-account-login` — DeepSeek account browser login | — | envio de dados ou conta/serviço DeepSeek (ADR 101); fetch/search via DeepSeek substituído por provedores sem custo |
| skip | implemented/architecture | `2026-09-14-independent-libreoffice-kit` — Independent LibreOffice kit ownership | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| skip | implemented/architecture | `2026-09-14-web-document-relative-app-routes` — Web app-owned routes are document-relative | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| skip | implemented/architecture | `2026-09-15-bounded-office-conversion` — Bounded shared Office conversion | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| skip | implemented/architecture | `2026-09-15-client-session-references` — Client Session references, reference sources, and UI status | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| skip | implemented/architecture | `2026-09-15-desktop-native-fatal-recovery` — Native Desktop fatal recovery | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| port | implemented/architecture | `2026-09-15-first-class-tool-role-messages` — First-class tool-role messages | `sdd/agent-loop-sessao.md` | formato de sessão v4 / persistência |
| adapt | implemented/architecture | `2026-09-15-guided-plugin-installation` — Guided plugin installation | `sdd/plugins-marketplace.md` | mercado de plugins do dsh (formato próprio) × formato Claude Code |
| adapt | implemented/architecture | `2026-09-16-creator-persistent-plugin-management` — Creator mode installs persistent plugin bundles | `sdd/plugins-marketplace.md` | mercado de plugins do dsh (formato próprio) × formato Claude Code |
| skip | implemented/architecture | `2026-09-16-experimental-voice-input` — Experimental voice input with explicit recognition providers | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| port | implemented/architecture | `2026-09-16-host-schedule-storage` — Host-owned scheduled messages | `sdd/jobs-schedule-terminal.md` | jobs, agenda, terminal |
| port | implemented/architecture | `2026-09-16-user-terminal-permissions` — User-terminal permissions | `sdd/seguranca-sandbox-permissoes.md` | segurança, credenciais, cwd |
| reject | implemented/architecture | `2026-09-17-account-provider-signout` — Account provider and active request cancellation | — | envio de dados ou conta/serviço DeepSeek (ADR 101); fetch/search via DeepSeek substituído por provedores sem custo |
| port | implemented/architecture | `2026-09-17-developer-session-changes` — Developer messages for incremental session changes | `sdd/agent-loop-sessao.md` | formato de sessão v4 / persistência |
| adapt | implemented/architecture | `2026-09-17-experimental-cua-object-api` — Native CUA objects independent of evaluator ownership | `sdd/browser-computer-agent-teams.md` | times, worktree e computer use |
| port | implemented/architecture | `2026-09-17-native-v4-read-validation` — Mandatory native V4 relationship validation | `sdd/agent-loop-sessao.md` | formato de sessão v4 / persistência |
| port | implemented/architecture | `2026-09-17-persistence-attribution-policy` — Recorded user-source attribution compatibility | `sdd/agent-loop-sessao.md` | formato de sessão v4 / persistência |
| port | implemented/architecture | `2026-09-17-process-local-blank-sessions` — Writer acquisition for blank Session reuse | `sdd/agent-loop-sessao.md` | formato de sessão v4 / persistência |
| skip | implemented/architecture | `2026-09-17-workspace-file-binary-transfer` — Binary workspace file transfer | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| adapt | implemented/architecture | `2026-09-18-declarative-agent-presets` — Declarative Agent presets and retained revisions | `sdd/subagentes-skills-hooks.md` | presets e subagentes (mapear para .claude/agents) |
| adapt | implemented/architecture | `2026-09-18-plugin-install-registries` — Plugin install registries | `sdd/plugins-marketplace.md` | mercado de plugins do dsh (formato próprio) × formato Claude Code |
| skip | implemented/architecture | `2026-09-18-volatile-config-references` — Volatile configuration references | — | perfis/cordis/HMR (substituídos por config TOML) |
| skip | implemented/architecture | `2026-09-19-concrete-web-bind-address` — Bind one concrete Web address | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| skip | implemented/architecture | `2026-09-19-profile-owned-live-configuration` — Keep live configuration in the owning profile | — | perfis/cordis/HMR (substituídos por config TOML) |
| skip | implemented/architecture | `2026-09-19-profile-resolution-lookup-order` — Resolve profile modules along the ancestor chain with an interception layer | — | perfis/cordis/HMR (substituídos por config TOML) |
| skip | implemented/architecture | `2026-09-19-projection-cache-listing-identity-and-cached-rows` — Projection cache read-only face matches the lifecycle identity; the client store separates cached and sequenced rows | — | perfis/cordis/HMR (substituídos por config TOML) |
| skip | implemented/architecture | `2026-09-19-remote-duplex-stream` — Remote duplex stream: one kind of stream, an uplink channel, and an invocation context | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| port | implemented/architecture | `2026-09-19-timed-user-question-two-settlements` — Timed user questions as two settlements | `sdd/ferramentas-claude-code.md` | pipeline e ferramentas |
| skip | implemented/architecture | `2026-09-19-window-drag-coverage-contract` — Window drag coverage as a contract | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| skip | implemented/architecture | `2026-09-20-device-local-shortcut-preferences` — Keep shortcut preferences device-local and commit before activation | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| port | implemented/architecture | `2026-09-20-dynamic-tool-updates` — Active tool headers and stateful request projection | `sdd/ferramentas-claude-code.md` | pipeline e ferramentas |
| skip | implemented/architecture | `2026-09-20-sidebar-retained-tab-layout` — Stable Sidebar tab mounting and Session retention | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| skip | implemented/architecture | `2026-09-21-conversation-build-groups` — Definition-owned grouping with generic Node/Group assembly | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| skip | implemented/architecture | `2026-09-21-desktop-page-close-shortcuts` — Desktop close shortcuts follow page focus | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| adapt | implemented/architecture | `2026-09-21-experimental-capabilities-as-optional-bundles` — Ship experimental capabilities as optional bundles | `sdd/plugins-marketplace.md` | mercado de plugins do dsh (formato próprio) × formato Claude Code |
| adapt | implemented/architecture | `2026-09-22-fatal-diagnostics-and-crash-reports` — Fatal diagnostics and Desktop crash reports | `sdd/build-release.md` | diagnóstico de falhas fatais (sem crash report remoto) |
| reject | implemented/architecture | `2026-09-22-platform-browser-storage` — Account-scoped Platform browser storage | — | envio de dados ou conta/serviço DeepSeek (ADR 101); fetch/search via DeepSeek substituído por provedores sem custo |
| port | implemented/architecture | `2026-09-22-schedule-hard-delete-and-bounded-history` — Schedule deletes task rows and bounds delivery history | `sdd/jobs-schedule-terminal.md` | jobs, agenda, terminal |
| port | implemented/architecture | `2026-09-22-tool-call-three-phases` — Tool calls own preparation, dispatch, and results by callId | `sdd/ferramentas-claude-code.md` | pipeline e ferramentas |
| port | implemented/architecture | `2026-09-23-best-effort-persistent-ptc` — Node PTC execution modes and in-memory state | `sdd/ferramentas-claude-code.md` | pipeline e ferramentas |
| skip | implemented/architecture | `2026-09-23-desktop-close-to-background-and-quit-confirmation` — Hide the Desktop window on close and confirm interruptible quits | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| skip | implemented/architecture | `2026-09-23-native-https-web-listener` — Native HTTPS on the Web listener | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| reject | implemented/architecture | `2026-09-24-bounded-session-log-upload` — Bounded DeepSeek Session-log upload | — | envio de dados ou conta/serviço DeepSeek (ADR 101); fetch/search via DeepSeek substituído por provedores sem custo |
| reject | implemented/architecture | `2026-09-24-deepseek-account-remote-model-catalog` — DeepSeek account models come from the account listing and gate by client version | — | envio de dados ou conta/serviço DeepSeek (ADR 101); fetch/search via DeepSeek substituído por provedores sem custo |
| port | implemented/architecture | `2026-09-24-preparing-tool-arguments` — Incremental tool-argument scanning and argument order during preparation | `sdd/ferramentas-claude-code.md` | pipeline e ferramentas |
| port | implemented/architecture | `2026-09-24-preset-scoped-schedule-tools` — Preset-scoped Schedule tools | `sdd/jobs-schedule-terminal.md` | jobs, agenda, terminal |
| adapt | implemented/architecture | `2026-09-24-preset-scoped-time-context` — Preset-scoped time context | `sdd/subagentes-skills-hooks.md` | presets e subagentes (mapear para .claude/agents) |
| port | implemented/architecture | `2026-09-24-scheduled-message-framing` — Delivered reminders are framed as scheduled messages from the user | `sdd/jobs-schedule-terminal.md` | jobs, agenda, terminal |
| reject | implemented/architecture | `2026-09-25-session-log-otel-byte-limits` — Session-log OTLP strings and bounded requests | — | envio de dados ou conta/serviço DeepSeek (ADR 101); fetch/search via DeepSeek substituído por provedores sem custo |
| adapt | implemented/architecture | `2026-09-26-cua-application-consent` — Application consent for native Computer Use | `sdd/browser-computer-agent-teams.md` | times, worktree e computer use |
| skip | implemented/architecture | `2026-09-27-progressive-office-skills` — Progressive Office skills with ordinary shell execution | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| port | implemented/architecture | `2026-09-29-explicit-session-history-preparation` — Open the Session when migration is required | `sdd/agent-loop-sessao.md` | formato de sessão v4 / persistência |
| skip | implemented/architecture | `2026-09-30-profile-package-refresh-and-manifest-invalidation` — Refresh runtime resolution after package operations and expire HMR package configuration | — | perfis/cordis/HMR (substituídos por config TOML) |
| skip | implemented/architecture | `2026-09-30-structured-draft-initialization` — Initial draft content and structured reference restoration | — | simplificação/teste interno do TS, sem efeito no contrato |
| skip | implemented/architecture | `2026-10-01-replace-changed-package-modules-on-recomposition` — Replace plugins whose loaded package files changed during profile recomposition | — | perfis/cordis/HMR (substituídos por config TOML) |
| reject | implemented/architecture | `2026-10-05-anonymous-reasoning-translation` — Anonymous translation remains separate from model history | — | envio de dados ou conta/serviço DeepSeek (ADR 101); fetch/search via DeepSeek substituído por provedores sem custo |
| adapt | implemented/architecture | `2026-10-05-official-on-demand-bundles` — Discover Official bundles without installing their runtimes | `sdd/plugins-marketplace.md` | mercado de plugins do dsh (formato próprio) × formato Claude Code |
| reject | implemented/architecture | `2026-10-05-paid-native-flash-reasoning-translation` — Paid native Flash translation uses independent queries | — | envio de dados ou conta/serviço DeepSeek (ADR 101); fetch/search via DeepSeek substituído por provedores sem custo |
| reject | implemented/architecture | `2026-10-05-persistent-machine-translation` — Machine translation results use shared Session records | — | envio de dados ou conta/serviço DeepSeek (ADR 101); fetch/search via DeepSeek substituído por provedores sem custo |
| port | implemented/architecture | `2026-10-05-shared-agents-root-instructions` — Shared agents root joins the user-global instruction chain | `sdd/compat-claude-code-totvs.md` | instruções e contexto de runtime |
| skip | implemented/architecture | `2026-10-06-web-desktop-plugin-identity-compatibility` — Preserve Web and Desktop plugin identities across experimental consolidation | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| skip | implemented/architecture | `2026-10-08-desktop-shell-configuration` — Desktop shell configuration independent of the Host | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| port | implemented/architecture | `2026-10-08-persistent-ptc-sandbox` — Retained PTC processes follow file policy | `sdd/ferramentas-claude-code.md` | pipeline e ferramentas |
| skip | implemented/architecture | `2026-10-09-composer-command-chip` — Claimed commands as atomic composer chips | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| adapt | implemented/architecture | `2026-10-09-cua-shared-native-helper` — Shared native CUA process with Agent-scoped state | `sdd/browser-computer-agent-teams.md` | times, worktree e computer use |
| skip | implemented/architecture | `2026-10-09-message-node-factories-and-delegates` — Domain image Slots and a shared gallery Factory | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| skip | implemented/architecture | `2026-10-09-operator-scoped-connect-handshake` — Peer-scoped connect handshake | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| port | implemented/architecture | `2026-10-09-ptc-owned-policy-cleanup` — PTC owns policy-change cleanup waits | `sdd/ferramentas-claude-code.md` | pipeline e ferramentas |
| skip | implemented/architecture | `2026-10-09-reasoning-content-slot-factory` — Standard reasoning Body Slot and reusable Content Factory | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| port | implemented/architecture | `2026-10-09-stored-credential-precedence` — Stored credentials take precedence over the launch environment | `sdd/seguranca-sandbox-permissoes.md` | segurança, credenciais, cwd |
| skip | implemented/architecture | `2026-10-10-user-text-json-nodes` — User-text candidate scheduling and Facts-driven refresh | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| skip | implemented/architecture | `2026-10-11-chat-group-tree-exchanges` — Chat group-tree positions and exchanges | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| skip | implemented/architecture | `2026-10-12-exclusive-conversation-matches` — Exclusive Conversation matches | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| port | implemented/architecture | `2026-10-12-headless-environment-auth` — Headless jobs omit the interactive credential store | `sdd/seguranca-sandbox-permissoes.md` | segurança, credenciais, cwd |
| skip | implemented/architecture | `2026-10-12-peer-principal-and-remote-access` — Peer principals and method-level Remote access control | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| skip | implemented/architecture | `2026-10-13-chat-single-fold-motion` — One fold motion for collapsing on the next message | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| port | implemented/architecture | `2026-10-13-context-breakdown-system-prompt-state` — Context breakdown retains only system prompts | `sdd/compat-claude-code-totvs.md` | instruções e contexto de runtime |
| adapt | implemented/architecture | `2026-10-13-cua-continuous-hover` — Continuous CUA hover without cross-call ownership | `sdd/browser-computer-agent-teams.md` | times, worktree e computer use |
| adapt | implemented/architecture | `2026-10-13-cua-hover-routes-and-coordinates` — Background hover and screenshot coordinates | `sdd/browser-computer-agent-teams.md` | times, worktree e computer use |
| adapt | implemented/architecture | `2026-10-13-team-identity-runtime-context` — Team identity in required runtime context | `sdd/browser-computer-agent-teams.md` | times, worktree e computer use |
| skip | implemented/architecture | `2026-10-14-plugin-entry-customization` — Plugin entry customization: region-owned pins and More menus | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| skip | implemented/architecture | `2026-10-14-webworker-preview-profile-parity` — Boot the Preview Worker from the shipped profile and fetch lazy chunks on demand | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| port | implemented/architecture | `2026-10-20-new-session-blank-reuse-by-preset` — New Session reuses a blank Session, matching a requested preset | `sdd/agent-loop-sessao.md` | formato de sessão v4 / persistência |
| reject | implemented/architecture | `2026-10-22-deepseek-official-model-listing` — The official DeepSeek route follows the official model listing until its catalog is customized | — | envio de dados ou conta/serviço DeepSeek (ADR 101); fetch/search via DeepSeek substituído por provedores sem custo |
| skip | implemented/bug-fix | `2026-09-06-packed-install-private-npm-cache` — Packed-install verification owns its npm cache | `sdd/build-release.md` | processo do repositório TS; ideias úteis só para build/CI |
| skip | implemented/bug-fix | `2026-09-15-authenticated-browser-host-preferences` — Share Host preferences with every authenticated browser | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| skip | implemented/bug-fix | `2026-09-15-client-session-blank-reconciliation` — Keep accepted Session presentation across list refreshes | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| port | implemented/bug-fix | `2026-09-20-session-open-read-coordination` — Session opening cache identity and read ordering | `sdd/agent-loop-sessao.md` | formato de sessão v4 / persistência |
| port | implemented/bug-fix | `2026-09-27-pwsh-prompt-tail-grace` — A seen prompt marker keeps its own tail grace | `sdd/jobs-schedule-terminal.md` | jobs, agenda, terminal |
| port | implemented/bug-fix | `2026-09-28-pty-bundled-console-host` — Windows persistent PTYs use the console host node-pty ships | `sdd/jobs-schedule-terminal.md` | jobs, agenda, terminal |
| port | implemented/bug-fix | `2026-09-29-reused-blank-runs-requested-preset` — A reused blank Session runs the requested preset | `sdd/agent-loop-sessao.md` | formato de sessão v4 / persistência |
| port | implemented/bug-fix | `2026-09-30-job-session-lifecycle` — Persist owned job lifecycle facts | `sdd/jobs-schedule-terminal.md` | jobs, agenda, terminal |
| skip | implemented/bug-fix | `2026-09-30-session-list-host-fairness` — Time-sliced Session-list work | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| port | implemented/bug-fix | `2026-10-10-background-shell-command-display` — Separate background Shell command display from job state | `sdd/jobs-schedule-terminal.md` | jobs, agenda, terminal |
| port | implemented/bug-fix | `2026-10-11-process-specific-prompt-values-in-runtime-context` — Process-specific checkout path and Web URL move to runtime context | `sdd/compat-claude-code-totvs.md` | instruções e contexto de runtime |
| adapt | implemented/feature | `2026-09-13-explicit-worktree-creation` — Explicit worktree creation | `sdd/browser-computer-agent-teams.md` | times, worktree e computer use |
| skip | implemented/feature | `2026-09-14-desktop-primary-runtime` — Desktop primary runtime | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| skip | implemented/feature | `2026-09-14-unattended-browser-terminal-reclamation` — Two-hour reclamation of unattended browser terminals | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| skip | implemented/feature | `2026-09-16-browser-excel-preview` — Browser Excel preview with saved formula results | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| skip | implemented/feature | `2026-09-16-sidebar-browser` — Sandboxed Sidebar browser | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| skip | implemented/feature | `2026-09-17-sidebar-file-and-directory-auto-refresh` — On-demand watching and automatic refresh for Sidebar previews and file trees | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| port | implemented/feature | `2026-09-19-windows-acl-mandatory-integrity-confinement` — Windows sandbox delete confinement: mandatory integrity plus an ambient-delete deny | `sdd/seguranca-sandbox-permissoes.md` | segurança, credenciais, cwd |
| skip | implemented/feature | `2026-09-20-config-schema-dump-validation-policy` — Config schema dump validation policy | — | perfis/cordis/HMR (substituídos por config TOML) |
| skip | implemented/feature | `2026-09-20-desktop-browser-webview` — Retained Desktop browser webviews | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| adapt | implemented/feature | `2026-09-20-spawn-agent-preset-selection` — Independent spawn preset selection and restoration | `sdd/subagentes-skills-hooks.md` | presets e subagentes (mapear para .claude/agents) |
| skip | implemented/feature | `2026-09-21-archive-stops-running-session-work` — Archiving a Session with running work | — | fora do escopo atual (candidatos futuros) |
| skip | implemented/feature | `2026-09-22-archive-admission-reads-stored-tasks` — Archive admission reads the stored tasks | — | fora do escopo atual (candidatos futuros) |
| skip | implemented/feature | `2026-09-24-auto-review-user-approval-fallback` — Auto review asks the user after a denial | — | fora do escopo atual (candidatos futuros) |
| skip | implemented/feature | `2026-09-24-session-inspector` — Session Inspector data ownership and navigation | — | fora do escopo atual (candidatos futuros) |
| adapt | implemented/feature | `2026-10-02-plugin-market-discover` — Plugin market Discover page and index reader | `sdd/plugins-marketplace.md` | mercado de plugins do dsh (formato próprio) × formato Claude Code |
| adapt | implemented/feature | `2026-10-02-plugin-market-index-format` — Plugin market index format | `sdd/plugins-marketplace.md` | mercado de plugins do dsh (formato próprio) × formato Claude Code |
| skip | implemented/feature | `2026-10-14-built-in-plugin-feature-previews` — Built-in plugin feature previews | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| skip | implemented/feature | `2026-10-14-host-release-change-refresh-prompt` — Prompt to reload when the Host release changes instead of hot-replacing across releases | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| reject | implemented/feature | `2026-10-15-deepseek-search-fetch-api` — Direct DeepSeek Search and Fetch APIs | — | envio de dados ou conta/serviço DeepSeek (ADR 101); fetch/search via DeepSeek substituído por provedores sem custo |
| skip | implemented/process | `2026-09-14-generated-cli-help-reference` — Generate the CLI reference from command help | `sdd/build-release.md` | processo do repositório TS; ideias úteis só para build/CI |
| adapt | implemented/process | `2026-09-15-shipped-optional-bundles` — Ship optional bundles with the installation | `sdd/plugins-marketplace.md` | mercado de plugins do dsh (formato próprio) × formato Claude Code |
| port | implemented/process | `2026-09-17-persistence-schema-review` — Persistence schema policy and review | `sdd/agent-loop-sessao.md` | formato de sessão v4 / persistência |
| skip | implemented/process | `2026-09-17-windows-runtime-signature-cache` — Windows runtime signature cache | `sdd/build-release.md` | processo do repositório TS; ideias úteis só para build/CI |
| skip | implemented/process | `2026-09-17-windows-signature-completion` — Separate Windows hardware signing from timestamp completion | `sdd/build-release.md` | processo do repositório TS; ideias úteis só para build/CI |
| skip | implemented/process | `2026-09-19-no-unknown-casts` — Prohibit new assertions to unknown | `sdd/build-release.md` | processo do repositório TS; ideias úteis só para build/CI |
| skip | implemented/process | `2026-09-28-dual-release-install-layout-work-budget` — Dual-release install layout work budget | `sdd/build-release.md` | processo do repositório TS; ideias úteis só para build/CI |
| skip | implemented/process | `2026-10-05-pr-artifact-typechecks` — PR artifact builds and aggregate TypeScript checks | `sdd/build-release.md` | processo do repositório TS; ideias úteis só para build/CI |
| skip | implemented/process | `2026-10-05-product-package-classification` — Require product use or an explicit package classification | `sdd/build-release.md` | processo do repositório TS; ideias úteis só para build/CI |
| port | implemented/process | `2026-10-08-session-reader-compatibility-review` — Session versions follow reader compatibility | `sdd/agent-loop-sessao.md` | formato de sessão v4 / persistência |
| skip | implemented/process | `2026-10-11-build-artifacts-check-split` — Split build:artifacts and build:check, with build running both concurrently | `sdd/build-release.md` | processo do repositório TS; ideias úteis só para build/CI |
| skip | implemented/process | `2026-10-11-lossless-build-speedups` — Build speed-ups that keep outputs unchanged | `sdd/build-release.md` | processo do repositório TS; ideias úteis só para build/CI |
| skip | implemented/process | `2026-10-12-local-gate-speedups` — Local gate, hook, and test speed-ups | `sdd/build-release.md` | processo do repositório TS; ideias úteis só para build/CI |
| skip | implemented/process | `2026-10-14-experimental-dependency-policy` — Experimental packages follow the same dependency declaration rules | `sdd/build-release.md` | processo do repositório TS; ideias úteis só para build/CI |
| skip | implemented/process | `2026-10-14-no-top-level-await-in-packages` — Package sources must not use top-level await | `sdd/build-release.md` | processo do repositório TS; ideias úteis só para build/CI |
| adapt | implemented/simplification | `2026-09-17-unified-subagent-activations` — Unified subagent activations | `sdd/subagentes-skills-hooks.md` | presets e subagentes (mapear para .claude/agents) |
| skip | implemented/simplification | `2026-09-19-remove-desktop-profile-core-cleanup` — Remove the Desktop production profile core-package cleanup | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| adapt | implemented/simplification | `2026-09-26-team-direct-inbox` — Team messages use the target inbox | `sdd/browser-computer-agent-teams.md` | times, worktree e computer use |
| port | implemented/simplification | `2026-10-05-native-or-ptc-presentation` — One tool invocation form per agent | `sdd/ferramentas-claude-code.md` | pipeline e ferramentas |
| adapt | implemented/simplification | `2026-10-09-preset-owned-subagent-composition` — Preset-owned subagent composition | `sdd/subagentes-skills-hooks.md` | presets e subagentes (mapear para .claude/agents) |
| skip | implemented/testing | `2026-09-10-desktop-local-updater-qualification` — Qualify Desktop update downloads with an isolated local server | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| skip | implemented/testing | `2026-09-14-desktop-installed-update-journal` — Preserve installed Desktop update evidence across restarts | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| skip | implemented/testing | `2026-09-14-desktop-installed-update-materials` — Isolate installed-update materials from releases and user data | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| skip | implemented/testing | `2026-09-27-observation-waits-on-observed-state` — Windows-lane observations wait on the state they read | — | simplificação/teste interno do TS, sem efeito no contrato |
| skip | implemented/testing | `2026-09-27-plugin-manager-git-child-environment` — Plugin-manager Git children isolate the host's command-line configuration | — | simplificação/teste interno do TS, sem efeito no contrato |
| skip | implemented/testing | `2026-09-27-spreadsheet-selection-by-keyboard-before-reflow` — Web lane moves spreadsheet selections by keyboard before reflow assertions | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| skip | implemented/testing | `2026-09-28-session-corpus-performance` — Performance baselines for large Session corpora | `sdd/build-release.md` | processo do repositório TS; ideias úteis só para build/CI |
| skip | implemented/testing | `2026-10-07-memory-limited-benchmarks` — Memory-limited benchmark reruns | `sdd/build-release.md` | processo do repositório TS; ideias úteis só para build/CI |
| skip | implemented/testing | `2026-10-07-scaled-benchmark-report` — Scaled machine and memory-pressure benchmark report | `sdd/build-release.md` | processo do repositório TS; ideias úteis só para build/CI |
| skip | implemented/testing | `2026-10-11-user-text-credential-matching` — Linear credential matching and performance gates for user text | `sdd/build-release.md` | processo do repositório TS; ideias úteis só para build/CI |
| skip | proposed/architecture | `2026-09-20-unified-typert-remote-metadata` — Unified Typert/Remote metadata | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| port | proposed/architecture | `2026-09-29-session-migration-follow-redirect` — Present single-Session migration through follow redirection | `sdd/agent-loop-sessao.md` | formato de sessão v4 / persistência |
| skip | proposed/architecture | `2026-10-09-cordis-cleanup-dependency-lifetimes` — Keep cleanup dependencies usable until consumers finish | — | perfis/cordis/HMR (substituídos por config TOML) |
| skip | proposed/feature | `2026-09-08-desktop-mandatory-update-api` — Desktop mandatory-update API | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| skip | proposed/feature | `2026-09-08-desktop-update-extensions` — Deferred Desktop update extensions | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| skip | proposed/feature | `2026-09-08-desktop-update-policy-and-installation` — Desktop update policy and installation | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| adapt | proposed/feature | `2026-10-03-pi-extensions-bridge` — Pi extensions bridge | `sdd/plugins-marketplace.md` | mercado de plugins do dsh (formato próprio) × formato Claude Code |
| adapt | proposed/feature | `2026-10-03-pi-plugins-compatibility` — Pi plugins compatibility — levels of support | `sdd/plugins-marketplace.md` | mercado de plugins do dsh (formato próprio) × formato Claude Code |
| skip | proposed/simplification | `2026-09-17-defer-office-background-scheduling` — Defer Office background scheduling until a consumer needs it | — | interface web/desktop, Office ou infraestrutura cordis/typert não portadas |
| skip | proposed/simplification | `2026-09-19-config-only-hmr` — Limit DSH HMR to live profile configuration | — | perfis/cordis/HMR (substituídos por config TOML) |
| skip | proposed/simplification | `2026-09-19-provider-only-instruction-reads` — Read instruction files only through a filesystem provider | — | simplificação/teste interno do TS, sem efeito no contrato |
| skip | proposed/simplification | `2026-09-19-prune-stream-visibility-helpers` — Prune unused stream visibility helpers | — | simplificação/teste interno do TS, sem efeito no contrato |
| skip | proposed/simplification | `2026-09-19-python-sdk-directional-client` — Remove the Python SDK's unused reverse RPC direction | — | simplificação/teste interno do TS, sem efeito no contrato |
| skip | proposed/simplification | `2026-09-19-retire-app-bin-boot-helpers` — Retire app-boot helpers used only by fixtures | — | simplificação/teste interno do TS, sem efeito no contrato |
| skip | proposed/simplification | `2026-09-19-retire-prompt-registry-change-event` — Retire the prompt registry change event | — | simplificação/teste interno do TS, sem efeito no contrato |
| skip | proposed/simplification | `2026-09-19-shell-env-key-declarations` — Keep shell-environment declarations focused on key ownership | — | simplificação/teste interno do TS, sem efeito no contrato |
| skip | proposed/simplification | `2026-09-19-trim-session-query-convenience-api` — Trim unused SessionQuery convenience methods | — | simplificação/teste interno do TS, sem efeito no contrato |
# Code/Spec Matrix — deepseek-harness (snapshot 0.1.6-alpha.1) → specs do dsh Rust

Leitura: cada pacote do dsh TS (e as fontes externas) com a spec que cobre o seu contrato no dsh Rust.
- 🟢 contrato coberto por regra explícita na spec.
- 🟡 coberto parcialmente ou de forma indireta.
- **—** sem spec: fora do escopo por decisão (UI web, cordis, telemetria rejeitada, testes) ou candidato à análise adicional.

## Resumo
| Medida | Valor |
|---|---|
| Pacotes em `packages/` | 284 |
| Cobertos 🟢 | 141 (50%) |
| Parciais 🟡 | 26 (9%) |
| Sem spec — | 117 (41%) |
| Fora do escopo por decisão (UI web 53 + API 7 + bundles 6 + cordis 4 + host 8 + web experimental 4 + feedback/identidade 3 + typert 4 + test-support 7) | 96 |
| **Cobertura do escopo do dsh Rust** (🟢+🟡 ÷ pacotes no escopo = 167 ÷ 188) | **≈ 89%** |
| Lacunas no escopo (candidatos à análise adicional) | 21: `ssh/*` (4), `webhook/*` (2), `experimental/auto-review`, `experimental/inspector`, `runtime-diagnostics/invariants`, utilitários `util/*` (12, absorvidos pelos crates Rust) |

> [Revisão 2026-11-01] Specs complementares `configuracao-dsh-home`, `build-release` e `retencao-anexos-spill` acrescentadas. `settings/*` e `attachment/*` passaram de 🟡 a 🟢, e `util/home-paths` e `util/output-retention` ganharam spec.

> Base: snapshot local 0.1.6-alpha.1. O upstream já está em `dsh-v0.2.1-rc.5`; a matriz deve ser refeita no primeiro ciclo de [upstream-sync](../sdd/upstream-sync.md).

## Specs × quantidade de pacotes
| Spec | Pacotes 🟢/🟡 |
|---|---|
| `sdd/ferramentas-claude-code.md` | 39 |
| `sdd/agent-loop-sessao.md` | 33 |
| `sdd/subagentes-skills-hooks.md` | 24 |
| `sdd/seguranca-sandbox-permissoes.md` | 17 |
| `sdd/superficies-tui-headless-acp-sdk.md` | 11 |
| `sdd/browser-computer-agent-teams.md` | 11 |
| `sdd/jobs-schedule-terminal.md` | 7 |
| `sdd/provedores-modelos.md` | 7 |
| `sdd/compactacao-contexto.md` | 5 |
| `sdd/compat-claude-code-totvs.md` | 5 |
| `sdd/lsp-csharp-plsql.md` | 3 |
| `sdd/mcp-cliente-gerenciado.md` | 3 |

## Matriz por arquivo / pacote
| Arquivo | Spec correspondente | Cobertura | Observação |
|---|---|---|---|
| `packages/acp/acp` | `sdd/superficies-tui-headless-acp-sdk.md` | 🟢 |  |
| `packages/api/gateway` | — | — | BFF da UI web (não portada) |
| `packages/api/remotes` | — | — | BFF da UI web (não portada) |
| `packages/api/session-controller` | — | — | BFF da UI web (não portada) |
| `packages/api/settings-controller` | — | — | BFF da UI web (não portada) |
| `packages/api/terminal-controller` | — | — | BFF da UI web (não portada) |
| `packages/api/workspace-controller` | — | — | BFF da UI web (não portada) |
| `packages/api/workspace-files` | — | — | BFF da UI web (não portada) |
| `packages/attachment/attachment` | `sdd/retencao-anexos-spill.md` | 🟢 | armazenamento por hash + GC |
| `packages/attachment/attachment-local` | `sdd/retencao-anexos-spill.md` | 🟢 | armazenamento por hash + GC |
| `packages/boot/app-boot` | `sdd/superficies-tui-headless-acp-sdk.md` | 🟡 | arranque; substituído pelo launcher Rust |
| `packages/boot/cmdline` | `sdd/superficies-tui-headless-acp-sdk.md` | 🟡 | arranque; substituído pelo launcher Rust |
| `packages/browser-use/browser-use` | `sdd/browser-computer-agent-teams.md` | 🟢 |  |
| `packages/bundle/acp-app` | — | — | composição cordis (substituída por traits; ADR 012) |
| `packages/bundle/base` | — | — | composição cordis (substituída por traits; ADR 012) |
| `packages/bundle/headless` | — | — | composição cordis (substituída por traits; ADR 012) |
| `packages/bundle/sdk-app` | — | — | composição cordis (substituída por traits; ADR 012) |
| `packages/bundle/sdk-minimal` | — | — | composição cordis (substituída por traits; ADR 012) |
| `packages/bundle/web-app` | — | — | composição cordis (substituída por traits; ADR 012) |
| `packages/client/connection` | — | — | UI web React (não portada) |
| `packages/client/file-upload` | — | — | UI web React (não portada) |
| `packages/client/hmr` | — | — | UI web React (não portada) |
| `packages/client/locale` | — | — | UI web React (não portada) |
| `packages/client/modules` | — | — | UI web React (não portada) |
| `packages/client/resources` | — | — | UI web React (não portada) |
| `packages/client/store` | — | — | UI web React (não portada) |
| `packages/client/ui-agent-preset` | — | — | UI web React (não portada) |
| `packages/client/ui-approval` | — | — | UI web React (não portada) |
| `packages/client/ui-attachment` | — | — | UI web React (não portada) |
| `packages/client/ui-brand-official` | — | — | UI web React (não portada) |
| `packages/client/ui-chat` | — | — | UI web React (não portada) |
| `packages/client/ui-commands` | — | — | UI web React (não portada) |
| `packages/client/ui-conversation` | — | — | UI web React (não portada) |
| `packages/client/ui-deliverables` | — | — | UI web React (não portada) |
| `packages/client/ui-directory-picker-browse` | — | — | UI web React (não portada) |
| `packages/client/ui-directory-picker-native` | — | — | UI web React (não portada) |
| `packages/client/ui-dockkit` | — | — | UI web React (não portada) |
| `packages/client/ui-goal` | — | — | UI web React (não portada) |
| `packages/client/ui-input-trigger` | — | — | UI web React (não portada) |
| `packages/client/ui-jobs` | — | — | UI web React (não portada) |
| `packages/client/ui-layout` | — | — | UI web React (não portada) |
| `packages/client/ui-message-feedback` | — | — | UI web React (não portada) |
| `packages/client/ui-model-selection` | — | — | UI web React (não portada) |
| `packages/client/ui-open-in-app` | — | — | UI web React (não portada) |
| `packages/client/ui-permission-presets` | — | — | UI web React (não portada) |
| `packages/client/ui-plan` | — | — | UI web React (não portada) |
| `packages/client/ui-primitives` | — | — | UI web React (não portada) |
| `packages/client/ui-reference` | — | — | UI web React (não portada) |
| `packages/client/ui-renderer` | — | — | UI web React (não portada) |
| `packages/client/ui-schedule` | — | — | UI web React (não portada) |
| `packages/client/ui-session` | — | — | UI web React (não portada) |
| `packages/client/ui-settings` | — | — | UI web React (não portada) |
| `packages/client/ui-settings-general` | — | — | UI web React (não portada) |
| `packages/client/ui-settings-models` | — | — | UI web React (não portada) |
| `packages/client/ui-settings-plugin-inventory` | — | — | UI web React (não portada) |
| `packages/client/ui-settings-plugins` | — | — | UI web React (não portada) |
| `packages/client/ui-settings-unarchive-sessions` | — | — | UI web React (não portada) |
| `packages/client/ui-sidebar` | — | — | UI web React (não portada) |
| `packages/client/ui-sidebar-documentpreview` | — | — | UI web React (não portada) |
| `packages/client/ui-sidebar-files` | — | — | UI web React (não portada) |
| `packages/client/ui-sidebar-right` | — | — | UI web React (não portada) |
| `packages/client/ui-sidebar-terminal` | — | — | UI web React (não portada) |
| `packages/client/ui-skill` | — | — | UI web React (não portada) |
| `packages/client/ui-slots` | — | — | UI web React (não portada) |
| `packages/client/ui-subagent` | — | — | UI web React (não portada) |
| `packages/client/ui-theme` | — | — | UI web React (não portada) |
| `packages/client/ui-tool` | — | — | UI web React (não portada) |
| `packages/client/ui-trajectory` | — | — | UI web React (não portada) |
| `packages/client/ui-user-questions` | — | — | UI web React (não portada) |
| `packages/client/ui-workflow-run` | — | — | UI web React (não portada) |
| `packages/client/ui-workspace` | — | — | UI web React (não portada) |
| `packages/client/web` | — | — | UI web React (não portada) |
| `packages/compaction/command-compact` | `sdd/compactacao-contexto.md` | 🟢 |  |
| `packages/compaction/compaction` | `sdd/compactacao-contexto.md` | 🟢 |  |
| `packages/compaction/compaction-basic` | `sdd/compactacao-contexto.md` | 🟢 |  |
| `packages/compaction/compaction-image-offload` | `sdd/compactacao-contexto.md` | 🟢 |  |
| `packages/compaction/compaction-tool-result-pruner` | `sdd/compactacao-contexto.md` | 🟢 |  |
| `packages/computer-use/computer-use` | `sdd/browser-computer-agent-teams.md` | 🟢 |  |
| `packages/context/agent-instructions` | `sdd/compat-claude-code-totvs.md` | 🟢 |  |
| `packages/context/file-reference` | `sdd/superficies-tui-headless-acp-sdk.md` | 🟡 | menções @arquivo |
| `packages/context/file-reference-local` | `sdd/superficies-tui-headless-acp-sdk.md` | 🟡 | menções @arquivo |
| `packages/context/session-reference` | `sdd/agent-loop-sessao.md` | 🟡 |  |
| `packages/context/time-context` | `sdd/agent-loop-sessao.md` | 🟢 | snapshot de runtime |
| `packages/context/tmux-context` | `sdd/agent-loop-sessao.md` | 🟡 |  |
| `packages/core/agent` | `sdd/agent-loop-sessao.md` | 🟢 |  |
| `packages/core/agent-default-model` | `sdd/agent-loop-sessao.md` | 🟢 |  |
| `packages/core/agent-loop` | `sdd/agent-loop-sessao.md` | 🟢 |  |
| `packages/core/agent-tool-presentation` | `sdd/agent-loop-sessao.md` | 🟢 |  |
| `packages/core/scope` | `sdd/agent-loop-sessao.md` | 🟢 |  |
| `packages/core/session` | `sdd/agent-loop-sessao.md` | 🟢 |  |
| `packages/core/system-prompt` | `sdd/agent-loop-sessao.md` | 🟢 |  |
| `packages/core/tools` | `sdd/ferramentas-claude-code.md` | 🟢 |  |
| `packages/credentials/authorization` | `sdd/seguranca-sandbox-permissoes.md` | 🟢 |  |
| `packages/credentials/credentials` | `sdd/seguranca-sandbox-permissoes.md` | 🟢 |  |
| `packages/credentials/credentials-local` | `sdd/seguranca-sandbox-permissoes.md` | 🟢 |  |
| `packages/experimental/agent-team` | `sdd/browser-computer-agent-teams.md` | 🟢 |  |
| `packages/experimental/agent-team-profile` | `sdd/browser-computer-agent-teams.md` | 🟡 |  |
| `packages/experimental/agent-team-web-profile` | — | — | web |
| `packages/experimental/auto-review` | — | — | candidato à análise adicional |
| `packages/experimental/browser-use-chrome-devtools-mcp` | `sdd/browser-computer-agent-teams.md` | 🟢 |  |
| `packages/experimental/browser-use-playwright-mcp` | `sdd/browser-computer-agent-teams.md` | 🟢 |  |
| `packages/experimental/browser-use-runtime` | `sdd/browser-computer-agent-teams.md` | 🟢 |  |
| `packages/experimental/browser-use-stagehand-native` | `sdd/browser-computer-agent-teams.md` | 🟢 |  |
| `packages/experimental/client-ui-agent-team` | — | — | web |
| `packages/experimental/computer-use-cua-driver-mcp` | `sdd/browser-computer-agent-teams.md` | 🟢 |  |
| `packages/experimental/computer-use-cua-driver-native` | `sdd/browser-computer-agent-teams.md` | 🟢 |  |
| `packages/experimental/inspector` | — | — | ferramenta de desenvolvimento |
| `packages/experimental/ptc-runtime-python` | `sdd/ferramentas-claude-code.md` | 🟢 | Won't (sem sandbox) |
| `packages/experimental/tool-agent-team` | `sdd/browser-computer-agent-teams.md` | 🟢 |  |
| `packages/experimental/webworker-packer` | — | — | web |
| `packages/experimental/webworker-runtime` | — | — | web |
| `packages/extensions/cordis-client-runner` | — | — | cordis (não portado) |
| `packages/extensions/cordis-host-runner` | — | — | cordis (não portado) |
| `packages/extensions/tool-cordis` | — | — | cordis (não portado) |
| `packages/extensions/ui-cordis` | — | — | cordis (não portado) |
| `packages/feedback/command-feedback` | — | — | feedback → telemetria (rejeitado, ADR 101) |
| `packages/feedback/message-feedback` | — | — | feedback → telemetria (rejeitado, ADR 101) |
| `packages/fs/fs` | `sdd/ferramentas-claude-code.md` | 🟢 |  |
| `packages/fs/fs-local` | `sdd/ferramentas-claude-code.md` | 🟢 |  |
| `packages/fs/fs-observation-policy` | `sdd/ferramentas-claude-code.md` | 🟢 |  |
| `packages/fs/fs-sandbox` | `sdd/ferramentas-claude-code.md` | 🟢 |  |
| `packages/fs/tool-fs` | `sdd/ferramentas-claude-code.md` | 🟢 |  |
| `packages/fs/tool-fs-search` | `sdd/ferramentas-claude-code.md` | 🟢 |  |
| `packages/fs/tool-present` | `sdd/superficies-tui-headless-acp-sdk.md` | 🟡 |  |
| `packages/fs/tool-str-replace-editor` | `sdd/ferramentas-claude-code.md` | 🟢 |  |
| `packages/goal/command-goal` | `sdd/subagentes-skills-hooks.md` | 🟢 |  |
| `packages/goal/goal` | `sdd/subagentes-skills-hooks.md` | 🟢 |  |
| `packages/goal/goal-round-driver` | `sdd/subagentes-skills-hooks.md` | 🟢 |  |
| `packages/goal/tool-goal` | `sdd/subagentes-skills-hooks.md` | 🟢 |  |
| `packages/guard/repeat-tool-reminder` | `sdd/seguranca-sandbox-permissoes.md` | 🟢 |  |
| `packages/guard/timeout-policy` | `sdd/seguranca-sandbox-permissoes.md` | 🟢 |  |
| `packages/hooks/hook-protocol` | `sdd/subagentes-skills-hooks.md` | 🟢 |  |
| `packages/hooks/hooks-claude-code` | `sdd/subagentes-skills-hooks.md` | 🟢 |  |
| `packages/hooks/hooks-codex` | `sdd/subagentes-skills-hooks.md` | 🟡 | formato Codex (opcional) |
| `packages/host/directory-picker` | — | — | web/desktop |
| `packages/host/directory-picker-auto` | — | — | web/desktop |
| `packages/host/directory-picker-browse` | — | — | web/desktop |
| `packages/host/directory-picker-native` | — | — | web/desktop |
| `packages/host/frontend-static` | — | — | web |
| `packages/host/open-in-app` | — | — | desktop |
| `packages/host/plugin-inventory` | — | — | inventário (rejeitado, ADR 101) |
| `packages/host/webserver` | — | — | web |
| `packages/identity/anonymous-user-id` | — | — | telemetria (rejeitado, ADR 101) |
| `packages/interaction/commands` | `sdd/superficies-tui-headless-acp-sdk.md` | 🟢 |  |
| `packages/interaction/permission-presets` | `sdd/seguranca-sandbox-permissoes.md` | 🟢 |  |
| `packages/interaction/tool-ask-user` | `sdd/ferramentas-claude-code.md` | 🟢 |  |
| `packages/interaction/user-approval` | `sdd/seguranca-sandbox-permissoes.md` | 🟢 |  |
| `packages/interaction/user-questions` | `sdd/ferramentas-claude-code.md` | 🟢 |  |
| `packages/jobs/jobs` | `sdd/jobs-schedule-terminal.md` | 🟢 |  |
| `packages/jobs/jobs-local` | `sdd/jobs-schedule-terminal.md` | 🟢 |  |
| `packages/jobs/tool-jobs` | `sdd/jobs-schedule-terminal.md` | 🟢 |  |
| `packages/llm/deepseek-llm-api-extensions` | `sdd/provedores-modelos.md` | 🟢 | não portado (ADR 101) |
| `packages/llm/llm` | `sdd/provedores-modelos.md` | 🟢 |  |
| `packages/llm/llm-deepseek` | `sdd/provedores-modelos.md` | 🟢 |  |
| `packages/llm/llm-pi-ai` | `sdd/provedores-modelos.md` | 🟢 |  |
| `packages/llm/llm-retry` | `sdd/provedores-modelos.md` | 🟢 |  |
| `packages/llm/plugin-package-inventory-deepseek` | `sdd/provedores-modelos.md` | 🟢 | não portado (ADR 101) |
| `packages/llm/token-meter` | `sdd/provedores-modelos.md` | 🟢 |  |
| `packages/lsp/lsp` | `sdd/lsp-csharp-plsql.md` | 🟢 |  |
| `packages/lsp/lsp-stdio` | `sdd/lsp-csharp-plsql.md` | 🟢 |  |
| `packages/lsp/tool-lsp` | `sdd/lsp-csharp-plsql.md` | 🟢 |  |
| `packages/mcp/mcp-client` | `sdd/mcp-cliente-gerenciado.md` | 🟢 |  |
| `packages/mcp/mcp-resources` | `sdd/mcp-cliente-gerenciado.md` | 🟢 |  |
| `packages/plan/plan-mode` | `sdd/seguranca-sandbox-permissoes.md` | 🟢 | plan restritivo |
| `packages/preset/agent-presets` | `sdd/compat-claude-code-totvs.md` | 🟡 | personas/presets → agentes .claude |
| `packages/preset/persona` | `sdd/compat-claude-code-totvs.md` | 🟡 | personas/presets → agentes .claude |
| `packages/ptc-runtime/ptc-runtime` | `sdd/ferramentas-claude-code.md` | 🟢 | RunCode |
| `packages/ptc-runtime/ptc-runtime-node` | `sdd/ferramentas-claude-code.md` | 🟢 | RunCode |
| `packages/runtime-diagnostics/invariants` | — | — | diagnóstico interno |
| `packages/sandbox/sandbox` | `sdd/seguranca-sandbox-permissoes.md` | 🟢 |  |
| `packages/sandbox/sandbox-local` | `sdd/seguranca-sandbox-permissoes.md` | 🟢 |  |
| `packages/sandbox/sandbox-policy` | `sdd/seguranca-sandbox-permissoes.md` | 🟢 |  |
| `packages/sandbox/sandbox-windows-acl` | `sdd/seguranca-sandbox-permissoes.md` | 🟢 |  |
| `packages/schedule/schedule` | `sdd/jobs-schedule-terminal.md` | 🟢 |  |
| `packages/sdk/client` | `sdd/superficies-tui-headless-acp-sdk.md` | 🟢 |  |
| `packages/sdk/protocol` | `sdd/superficies-tui-headless-acp-sdk.md` | 🟢 |  |
| `packages/sdk/server` | `sdd/superficies-tui-headless-acp-sdk.md` | 🟢 |  |
| `packages/session/session-checkpoint-policy` | `sdd/agent-loop-sessao.md` | 🟢 |  |
| `packages/session/session-format` | `sdd/agent-loop-sessao.md` | 🟢 |  |
| `packages/session/session-format-catalog` | `sdd/agent-loop-sessao.md` | 🟢 |  |
| `packages/session/session-format-v0-to-v1` | `sdd/agent-loop-sessao.md` | 🟢 |  |
| `packages/session/session-format-v1-to-v2` | `sdd/agent-loop-sessao.md` | 🟢 |  |
| `packages/session/session-format-v2-to-v3` | `sdd/agent-loop-sessao.md` | 🟢 |  |
| `packages/session/session-log-deepseek` | `sdd/agent-loop-sessao.md` | 🟢 | não portado (ADR 101) |
| `packages/session/session-persistence` | `sdd/agent-loop-sessao.md` | 🟢 |  |
| `packages/session/session-persistence-jsonl` | `sdd/agent-loop-sessao.md` | 🟢 |  |
| `packages/session/session-projection` | `sdd/agent-loop-sessao.md` | 🟢 |  |
| `packages/session/session-projection-cache` | `sdd/agent-loop-sessao.md` | 🟢 |  |
| `packages/session/session-stats` | `sdd/agent-loop-sessao.md` | 🟡 |  |
| `packages/session/session-telemetry` | `sdd/agent-loop-sessao.md` | 🟢 | não portado (ADR 101) |
| `packages/session/session-telemetry-otel` | `sdd/agent-loop-sessao.md` | 🟢 | não portado (ADR 101) |
| `packages/session/session-title` | `sdd/agent-loop-sessao.md` | 🟡 | título (sem regra dedicada) |
| `packages/session/session-title-all-prompts-llm` | `sdd/agent-loop-sessao.md` | 🟡 | título (sem regra dedicada) |
| `packages/session/session-title-first-prompt-llm` | `sdd/agent-loop-sessao.md` | 🟡 | título (sem regra dedicada) |
| `packages/session/session-title-llm` | `sdd/agent-loop-sessao.md` | 🟡 | título (sem regra dedicada) |
| `packages/session/session-turn-outline` | `sdd/agent-loop-sessao.md` | 🟡 |  |
| `packages/session-query/session-log-export` | `sdd/superficies-tui-headless-acp-sdk.md` | 🟡 | /export |
| `packages/session-query/session-query` | `sdd/ferramentas-claude-code.md` | 🟢 |  |
| `packages/session-query/session-query-sqlite` | `sdd/ferramentas-claude-code.md` | 🟢 |  |
| `packages/session-query/tool-session-query` | `sdd/ferramentas-claude-code.md` | 🟢 |  |
| `packages/settings/settings` | `sdd/configuracao-dsh-home.md` | 🟢 | camadas, watch, escrita atômica |
| `packages/settings/settings-file` | `sdd/configuracao-dsh-home.md` | 🟢 | camadas, watch, escrita atômica |
| `packages/shell/bash-local` | `sdd/ferramentas-claude-code.md` | 🟢 |  |
| `packages/shell/bash-sandbox` | `sdd/ferramentas-claude-code.md` | 🟢 |  |
| `packages/shell/pwsh-local` | `sdd/ferramentas-claude-code.md` | 🟢 |  |
| `packages/shell/pwsh-sandbox` | `sdd/ferramentas-claude-code.md` | 🟢 |  |
| `packages/shell/shell` | `sdd/ferramentas-claude-code.md` | 🟢 |  |
| `packages/shell/shell-env` | `sdd/ferramentas-claude-code.md` | 🟢 |  |
| `packages/shell/tool-bash` | `sdd/ferramentas-claude-code.md` | 🟢 |  |
| `packages/shell/tool-bash-persistent` | `sdd/ferramentas-claude-code.md` | 🟢 |  |
| `packages/shell/tool-pwsh` | `sdd/ferramentas-claude-code.md` | 🟢 |  |
| `packages/shell/tool-pwsh-persistent` | `sdd/ferramentas-claude-code.md` | 🟢 |  |
| `packages/skill/skill` | `sdd/subagentes-skills-hooks.md` | 🟢 |  |
| `packages/skill/skill-badge` | `sdd/subagentes-skills-hooks.md` | 🟢 |  |
| `packages/skill/skill-filesystem` | `sdd/compat-claude-code-totvs.md` | 🟢 |  |
| `packages/skill/tool-skill` | `sdd/subagentes-skills-hooks.md` | 🟢 |  |
| `packages/spill/spill` | `sdd/ferramentas-claude-code.md` | 🟢 |  |
| `packages/spill/spill-local` | `sdd/ferramentas-claude-code.md` | 🟢 |  |
| `packages/spill/spill-policy` | `sdd/ferramentas-claude-code.md` | 🟢 |  |
| `packages/ssh/fs-ssh` | — | — | execução remota SSH (prioridade C) |
| `packages/ssh/sandbox-ssh` | — | — | execução remota SSH (prioridade C) |
| `packages/ssh/ssh` | — | — | execução remota SSH (prioridade C) |
| `packages/ssh/subprocess-ssh` | — | — | execução remota SSH (prioridade C) |
| `packages/storage/storage` | `sdd/agent-loop-sessao.md` | 🟡 | estado de app em SQLite (harness-store) |
| `packages/storage/storage-domain` | `sdd/agent-loop-sessao.md` | 🟡 | estado de app em SQLite (harness-store) |
| `packages/storage/storage-json` | `sdd/agent-loop-sessao.md` | 🟡 | estado de app em SQLite (harness-store) |
| `packages/storage/storage-sqlite` | `sdd/agent-loop-sessao.md` | 🟡 | estado de app em SQLite (harness-store) |
| `packages/subagent/subagent` | `sdd/subagentes-skills-hooks.md` | 🟢 |  |
| `packages/subagent/subagent-acp` | `sdd/subagentes-skills-hooks.md` | 🟢 |  |
| `packages/subagent/subagent-claude-code` | `sdd/subagentes-skills-hooks.md` | 🟢 |  |
| `packages/subagent/subagent-codex` | `sdd/subagentes-skills-hooks.md` | 🟢 |  |
| `packages/subagent/subagent-dsh-sdk` | `sdd/subagentes-skills-hooks.md` | 🟢 |  |
| `packages/subagent/subagent-fork-in-process` | `sdd/subagentes-skills-hooks.md` | 🟢 |  |
| `packages/subagent/subagent-in-process-driver` | `sdd/subagentes-skills-hooks.md` | 🟢 |  |
| `packages/subagent/subagent-spawn-in-process` | `sdd/subagentes-skills-hooks.md` | 🟢 |  |
| `packages/subagent/tool-subagent` | `sdd/subagentes-skills-hooks.md` | 🟢 |  |
| `packages/subagent/tool-subagent-control` | `sdd/subagentes-skills-hooks.md` | 🟢 |  |
| `packages/subprocess/subprocess` | `sdd/seguranca-sandbox-permissoes.md` | 🟢 | contenção |
| `packages/subprocess/subprocess-local` | `sdd/seguranca-sandbox-permissoes.md` | 🟢 | contenção |
| `packages/subprocess/win32-process` | `sdd/seguranca-sandbox-permissoes.md` | 🟢 | contenção |
| `packages/terminal/terminal` | `sdd/jobs-schedule-terminal.md` | 🟢 |  |
| `packages/terminal/terminal-bash` | `sdd/jobs-schedule-terminal.md` | 🟢 |  |
| `packages/terminal/tool-terminal` | `sdd/jobs-schedule-terminal.md` | 🟢 |  |
| `packages/test-support/agent-loop-testkit` | — | — | infra de testes |
| `packages/test-support/client-runtime` | — | — | infra de testes |
| `packages/test-support/llm-mock-server` | — | — | infra de testes |
| `packages/test-support/llm-replay` | — | — | infra de testes |
| `packages/test-support/loader-smoke` | — | — | infra de testes |
| `packages/test-support/remote-mock` | — | — | infra de testes |
| `packages/test-support/session-snapshot` | — | — | infra de testes |
| `packages/todo/tool-todo` | `sdd/ferramentas-claude-code.md` | 🟢 |  |
| `packages/typert/generator` | — | — | substituído por serde/schemars |
| `packages/typert/loader` | — | — | substituído por serde/schemars |
| `packages/typert/protocol` | — | — | substituído por serde/schemars |
| `packages/typert/registry` | — | — | substituído por serde/schemars |
| `packages/util/atomic-write` | `sdd/ferramentas-claude-code.md` | 🟡 |  |
| `packages/util/brand` | — | — | utilitários |
| `packages/util/chunked-list` | — | — | utilitários |
| `packages/util/crypto` | — | — | utilitários |
| `packages/util/deque` | — | — | utilitários |
| `packages/util/home-paths` | `sdd/configuracao-dsh-home.md` | 🟢 | DSH_HOME |
| `packages/util/http-proxy` | `sdd/mcp-cliente-gerenciado.md` | 🟡 | proxy/TLS corporativo |
| `packages/util/launch-environment` | — | — | utilitários |
| `packages/util/lazy-require` | — | — | utilitários |
| `packages/util/native-command` | — | — | utilitários |
| `packages/util/output-retention` | `sdd/ferramentas-claude-code.md` | 🟡 | corte cabeça/cauda de saídas |
| `packages/util/package-manifest` | — | — | utilitários |
| `packages/util/time` | — | — | utilitários |
| `packages/util/timeout` | — | — | utilitários |
| `packages/util/values` | — | — | utilitários |
| `packages/util/workspace-path` | — | — | utilitários |
| `packages/web/tool-web` | `sdd/ferramentas-claude-code.md` | 🟢 |  |
| `packages/web/web` | `sdd/ferramentas-claude-code.md` | 🟢 |  |
| `packages/web/web-fetch-http` | `sdd/ferramentas-claude-code.md` | 🟢 |  |
| `packages/web/web-search-deepseek` | `sdd/ferramentas-claude-code.md` | 🟢 | não é padrão (custo de turno) |
| `packages/web/web-search-exa` | `sdd/ferramentas-claude-code.md` | 🟢 |  |
| `packages/web/web-search-perplexity` | `sdd/ferramentas-claude-code.md` | 🟢 |  |
| `packages/webhook/webhook` | — | — | GitHub review (futuro) |
| `packages/webhook/webhook-github` | — | — | GitHub review (futuro) |
| `packages/workflow/tool-ralph` | `sdd/subagentes-skills-hooks.md` | 🟢 |  |
| `packages/workflow/tool-workflow` | `sdd/subagentes-skills-hooks.md` | 🟢 |  |
| `packages/workflow/workflow` | `sdd/subagentes-skills-hooks.md` | 🟢 |  |
| `packages/workflow/workflow-ptc` | `sdd/subagentes-skills-hooks.md` | 🟢 |  |
| `packages/workspace/workspace` | `sdd/compat-claude-code-totvs.md` | 🟡 | workspace = cwd/raiz |
| `apps/cli` | `sdd/superficies-tui-headless-acp-sdk.md` | 🟢 |  |
| `apps/web` | — | — | UI web |
| `apps/desktop` | — | — | Electron |
| `apps/desktop-host` | — | — | Electron |
| `native/system` | `sdd/seguranca-sandbox-permissoes.md` | 🟢 | launcher Landlock |
| `vendor/cordis (+ loader, hmr, schemastery…)` | — | — | DI cordis (ADR 003/012; substituído) |
| `python/sdk, python/sdk-runtime` | `sdd/superficies-tui-headless-acp-sdk.md` | 🟡 | SDK Python espelha o protocolo |
| `.agents/notes/**` | `sdd/upstream-sync.md` | 🟢 | fonte do upstream-sync |
| `D:/Projetos/Totvs AGP/VSCode/.claude/**` | `sdd/compat-claude-code-totvs.md` | 🟢 | inventário |
| `D:/Projetos/Totvs AGP/VSCode/CatalogoSKILLs/` | `sdd/catalog-sync.md` | 🟢 |  |
| `morluto/rea (npm rea-agents)` | `sdd/rea.md` | 🟢 | externo |
| `coreos/ai-helpers` | `sdd/plugins-marketplace.md` | 🟢 | externo |
| `garrytan/gbrain` | `sdd/memoria-gbrain.md` | 🟢 | externo |
## Specs sem código de origem no dsh TS (componentes novos)
| Spec | Origem |
|---|---|
| `sdd/catalog-sync.md` | Requisito do usuário (`catalog-sync`) + inventário Totvs |
| `sdd/plugins-marketplace.md` | Formato Claude Code (coreos/ai-helpers, REA) |
| `sdd/rea.md` | morluto/rea |
| `sdd/memoria-gbrain.md` (GBrain, memória por agente) | garrytan/gbrain + formato Claude Code |
| `sdd/upstream-sync.md` | Decisão `upstream-tracking` |
| `sdd/configuracao-dsh-home.md` | Revisão (lacuna G1) |
| `sdd/build-release.md` | Revisão (lacuna G2) + padrões CoreOS |
| `sdd/retencao-anexos-spill.md` | Revisão (lacuna G3) |
| Gate de rede, `data_policy`, adapter Gemini (`sdd/provedores-modelos.md`) | Decisões `gemini-subscription`, `totvs-data-gate` |
| TUI e `-p`/`stream-json` (`sdd/superficies-tui-headless-acp-sdk.md`) | Paridade com o Claude Code |

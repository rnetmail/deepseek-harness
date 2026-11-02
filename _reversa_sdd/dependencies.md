# Dependências — deepseek-harness

> Gerado pelo **Reversa Scout** em 2026-11-01. As versões são as declaradas nos `package.json` (faixas semver), não as resolvidas no lockfile. Fonte: `pnpm-lock.yaml`, `pnpm-workspace.yaml`.

## Runtime e toolchain

| Item | Versão | Onde |
|---|---|---|
| Node.js | `^22.19.0 \|\| >=24.0.0` | `package.json#engines` |
| pnpm | 11.7.0 | `package.json#packageManager` |
| TypeScript | ^6.0.3 | raiz |
| tsdown | ^0.22.2 | raiz (bundler) |
| tsx | ^4.22.4 | raiz (scripts) |
| @yao-pkg/pkg | 6.21.0 | raiz (exe único) |
| Vitest / coverage-v8 | ^4.1.8 | raiz |
| oxlint / oxlint-tsgolint | 1.76.0 / 7.0.2001 | raiz |
| jscpd, lefthook, publint | ^5.0.12, ^2.1.9, ^0.3.21 | raiz |
| fast-check | ^4.8.0 | raiz (testes de propriedade) |

## Núcleo da plataforma (vendorizado)

| Pacote | Papel |
|---|---|
| `vendor/cordis` | Container de plugins e injeção de dependência: a espinha dorsal do produto |
| `vendor/schemastery` | Schemas de configuração dos plugins (override no workspace) |
| `vendor/cosmokit` | Utilitários (override no workspace) |
| `vendor/loader`, `vendor/include`, `vendor/hmr` | Carregamento de `cordis.yml` e patches YAML, includes, hot-reload |
| `vendor/logger-console`, `vendor/timer`, `vendor/group` | Infraestrutura |

## LLM e protocolos de agente (críticos para o port)

| Dependência | Versão | Pacote consumidor | Uso |
|---|---|---|---|
| `@earendil-works/pi-ai` | ^0.85.1 | `llm/llm-pi-ai` | Catálogo multi-provedor de modelos e protocolos (OpenAI, Anthropic, Azure, Google...). Isento de `minimumReleaseAge` para acompanhar atualizações do catálogo |
| `eventsource-parser` | ^3.1.0 | `llm/llm-deepseek` | Streaming SSE |
| `@modelcontextprotocol/client` / `server` / `node` | 2.0.0 | `mcp/mcp-client` | Cliente MCP (servidores `everything` e `filesystem` usados nos testes) |
| `@modelcontextprotocol/sdk` | ^1.29.0 | `subagent/subagent-claude-code` | MCP no driver Claude Code |
| `@agentclientprotocol/sdk` | 1.4.0 | `acp/acp`, `subagent/subagent-acp`, `apps/cli` | Agent Client Protocol |
| `@anthropic-ai/claude-agent-sdk` | 0.3.263 | `subagent/subagent-claude-code` | Claude Code como subagente |
| `@anthropic-ai/sdk` | 0.93.0 | `subagent/subagent-claude-code` | API Anthropic |
| `@openai/codex` | 0.153.4 | `subagent/subagent-codex` | Codex como subagente |
| `zod` | ^4.4.3 | ~60 pacotes | Validação de schemas e contratos |

## Sistema, processos e arquivos

| Dependência | Versão | Uso |
|---|---|---|
| `node-pty` | 1.2.0-beta.15 | PTY persistente (ConPTY no Windows) |
| `koffi` | ^3.1.0 | FFI: escrita atômica (MoveFileExW), ACL no Windows, directory picker nativo, Win32 process |
| `@vscode/ripgrep` | ^1.18.0 | Ferramenta de busca |
| `@xterm/headless` / `addon-serialize` | ^6.0.0 | Terminal headless |
| `chokidar` | 4.0.3 / ^5.0.0 | Watch de settings, credenciais e skills |
| `yaml` / `js-yaml` | ^2.9.0 / ^4.2.0 | Settings, presets, skills, patches cordis |
| `diff` | ^9.0.0 | Diffs de edição |
| `sharp` | ^0.35.3 | Processamento de imagens anexadas |
| `undici` | ^8.10.0 | HTTP, proxy, web-fetch |
| `ipaddr.js` | — | Proteção SSRF em web-fetch 🟡 |
| `turndown` + `@joplin/turndown-plugin-gfm` | ^7.2.4 | HTML → Markdown (web-fetch) |
| `fflate` | ^0.8.2 | Exportação de sessões compactadas |
| `ws` | 8.21.0 | Gateway WebSocket (cliente ↔ host) |
| `commander` | ^15.0.0 | CLI |
| `@opentelemetry/*` | ^1.9.1 | Telemetria de sessão (OTLP) |
| `@octokit/webhooks` | — | Webhooks do GitHub |
| SQLite | `node:sqlite` (built-in) 🟡 | `storage-sqlite`, `session-query-sqlite` |

## Interface gráfica

| Dependência | Versão | Uso |
|---|---|---|
| React / React DOM | ^18.2.0 | `apps/web`, `packages/client/*` |
| Vite | ^6.0.0 | Build do frontend |
| Electron | ^44.0.0 | `apps/desktop` |
| electron-builder / electron-updater | ^26.15.3 / — | Empacotamento (NSIS, mac) e auto-update |
| mermaid | 11.16.0 | Diagramas na UI/docs |

## Experimentais

| Dependência | Versão | Uso |
|---|---|---|
| `@playwright/mcp` | 0.0.80 | Browser-use via MCP |
| `chrome-devtools-mcp` | — | Browser-use via CDP |
| `@browserbasehq/stagehand` + `@puppeteer/browsers` | 4.1.0 | Browser-use nativo |
| `@trycua/cua-driver` | — | Computer-use |
| `acorn`, `@yarnpkg/parsers`, `@noble/hashes` | — | Webworker runtime (Node no browser) |

## Python

| Projeto | Arquivo |
|---|---|
| `python/sdk` | `pyproject.toml`, `uv.lock` — SDK cliente |
| `python/sdk-runtime` | `pyproject.toml`, `hatch_build.py`: empacota o exe/runtime Node por plataforma (`platforms.json`) |

## Política de dependências (pnpm)

- `allowBuilds` em modo *deny by default*. Liberados: esbuild, lefthook, node-pty, koffi e o postinstall de `subprocess-local`.
- `minimumReleaseAgeExclude` libera pi-ai 0.85.1 para receber atualizações do catálogo de modelos.
- Dependabot configurado (`.github/dependabot.yml`).

## Equivalentes prováveis em Rust (🟡 sugestão para o Architect / Reconstructor)

| Função | Node atual | Candidato em Rust |
|---|---|---|
| Async runtime / HTTP / SSE | undici, eventsource-parser | `tokio`, `reqwest`, `eventsource-stream` |
| Catálogo multi-provedor | pi-ai | `genai` / `async-openai` + catálogo próprio em TOML/YAML |
| MCP | @modelcontextprotocol/* | `rmcp` (SDK oficial em Rust) |
| ACP | @agentclientprotocol/sdk | `agent-client-protocol` (crate) |
| Validação de schemas | zod | `serde` + `schemars` |
| CLI / TUI | commander / React | `clap` + `ratatui` |
| PTY / processos | node-pty, koffi | `portable-pty`, `windows-rs` |
| Busca | @vscode/ripgrep | `grep-searcher` / `ignore` (crates do ripgrep) |
| Watch de arquivos | chokidar | `notify` |
| YAML / JSONL / SQLite | yaml, node:sqlite | `serde_yaml`, `serde_json`, `rusqlite` |
| Telemetria | OpenTelemetry JS | `opentelemetry` + `tracing` |

# Inventário — deepseek-harness

> Gerado pelo **Reversa Scout** em 2026-11-01. Fonte estruturada: `.reversa/context/surface.json`.
> Escala de confiança: 🟢 CONFIRMADO · 🟡 INFERIDO · 🔴 LACUNA

## 1. Visão geral

| Item | Valor |
|---|---|
| Nome | `@deepseek-ai/dsh-root` — **DeepSeek Harness (dsh)** 🟢 |
| Versão | `0.1.6-alpha.1` 🟢 |
| Licença | MIT 🟢 |
| Tipo | Monorepo pnpm de um *agent harness* de código (CLI + Web + Desktop + SDKs) 🟢 |
| Linguagem principal | TypeScript (ESM, `"type": "module"`) 🟢 |
| Runtime | Node.js `^22.19.0 \|\| >=24.0.0` 🟢 |
| Gerenciador | pnpm 11.7.0 (workspaces) 🟢 |
| Histórico | ~17.177 commits no Git 🟢 |
| Arquivos (sem node_modules/lib/dist) | ~11.245 🟢 |
| Pacotes no workspace | ~300 (`packages/*/*`, `apps/*`, `vendor/*`, `native/*`, `python/*`) 🟢 |
| Código Rust existente | **nenhum** (0 arquivos `.rs`) 🟢 |

### Arquitetura em uma frase
Todo o produto é uma **composição de plugins** sobre o container de injeção de dependência **cordis** (vendorizado em `vendor/cordis`). Cada pacote é um plugin que publica ou consome serviços, e os "perfis" do produto (web, headless, acp, sdk) são arquivos `cordis.patch.yml` em `packages/bundle/*` que declaram quais plugins montar. 🟢 (confirmado em `apps/cli/composition.md` e `packages/bundle/base/cordis.patch.yml`)

## 2. Linguagens (contagem por extensão)

| Linguagem | Extensões | Arquivos |
|---|---|---|
| TypeScript | .ts / .tsx / .mts | 4.027 (3.634 + 385 + 8) |
| Markdown | .md | 3.443 (docs, README em en/zh, notas de agentes) |
| YAML | .yaml / .yml | 1.986 (i18n, patches cordis, CI) |
| JSON / JSONL | .json / .jsonl | 1.332 (JSONL = snapshots de sessão) |
| CSS | .css | 149 |
| JavaScript | .mjs / .js / .cjs | 102 |
| Python | .py | 22 (SDK Python) |
| C / C++ | .c / .cpp | 5 (launcher Landlock em `native/system`) |
| Shell / Batch | .sh / .cmd | 10 |

## 3. Estrutura de pastas (nível 1–2)

```
deepseek-harness/
├── apps/                 # Produtos montados
│   ├── cli/              # bin `dsh` (commander) — entry principal
│   ├── desktop/          # App Electron 44
│   ├── desktop-host/     # Host do processo desktop
│   └── web/              # Frontend React 18 + Vite 6
├── packages/             # ~59 famílias, ~280 pacotes-plugin (ver §5)
├── vendor/               # cordis, cosmokit, schemastery, loader, hmr, logger, timer...
├── native/system/        # Launcher de sandbox Landlock (C) + addons por plataforma
├── python/               # sdk (cliente Python) e sdk-runtime (distribuição do exe)
├── scripts/              # build, lint, docs, release (tsx)
├── docs/                 # Documentação do produto (en/zh)
├── snapshots/            # Snapshots esperados (acp, sdk, session, web)
├── benchmarks/, website/, patches/
├── .agents/              # skills e notes para agentes de desenvolvimento do repo
├── .claude/skills/       # skills para Claude Code (dev do repo)
├── .github/workflows/    # 19 workflows de CI/CD
├── dsh.cmd               # launcher local do usuário (⚠ contém segredos)
└── dsh_inicializacao.*   # rascunho de launcher integrado ao Obsidian
```

## 4. Pontos de entrada

| Caminho | Tipo | Conf. |
|---|---|---|
| `apps/cli/src/bin.ts` → `lib/bin.js` | bin `dsh`; carrega env em camadas (`loadLayeredEnv`) e despacha modo via `parseDshArgs` | 🟢 |
| `apps/cli/src/args.ts` | parsing de modos (`web`, headless, acp, sdk...) | 🟢 |
| `packages/bundle/base/cordis.patch.yml` | composição base compartilhada (llm, session, agent, settings, credentials, llm-pi-ai, jsonl, sqlite, storage...) | 🟢 |
| `packages/bundle/{web-app,headless,acp-app,sdk-app,sdk-minimal}/cordis.patch.yml` | perfis de execução | 🟢 |
| `apps/desktop/src` | processo main do Electron | 🟢 |
| `apps/web/index.html` + `src/` | SPA servida pelo modo `dsh web` (porta padrão 3080, segundo `dsh.cmd`) | 🟡 |
| `python/sdk/src` | SDK Python | 🟢 |
| `dsh.cmd` | launcher Windows do usuário: define env e executa `pnpm dsh web` | 🟢 |

### Configuração em tempo de execução
- Diretório do usuário: `~/.dsh`, que pode ser sobrescrito pela variável `DSH_HOME` 🟢 (`packages/util/home-paths`)
- Settings em YAML com hot-reload via chokidar (`settings/settings-file`) 🟢
- Credenciais em YAML local (`credentials/credentials-local`); provedores referenciam chaves por `apiKeyEnv` 🟢
- Exemplos de configuração: `apps/cli/config/examples/{cordis,github-review,mcp-memory,schedule}` 🟢
- Workspace de trabalho: variável `DSH_WORKSPACE` (usada em `dsh.cmd`) 🟡

## 5. Módulos (famílias de pacotes)

Prioridade em relação ao objetivo de **reconstruir o harness em Rust**:
- **A** = núcleo do agente, obrigatório para paridade com o Claude Code
- **B** = suporte
- **C** = interface gráfica ou experimental

| Módulo | Pacotes | Arq. src | Arq. teste | Prior. | Papel (🟡 inferido pelo nome/README) |
|---|---|---|---|---|---|
| `llm` | 7 | 84 | 56 | A | Abstração LLM; adaptadores `llm-deepseek` e **`llm-pi-ai`** (multi-provedor: OpenAI-compatível, Anthropic, Azure, Gemini, self-hosted, OAuth); retry; token-meter |
| `core` | 8 | 51 | 67 | A | `agent`, `agent-loop`, `tools`, `session`, `system-prompt`, `scope` |
| `session` | 19 | 94 | 64 | A | Formato de sessão (migrações v0→v3), persistência JSONL, título, stats, telemetria OTel |
| `context` | 6 | 29 | 10 | A | `agent-instructions` (equivalente a CLAUDE.md/AGENTS.md), referências de arquivo/sessão, contexto de tempo/tmux |
| `compaction` | 5 | 20 | 13 | A | Compactação de contexto, pruning de tool results, offload de imagens, `/compact` |
| `skill` | 4 | 4 | 5 | A | Skills do filesystem (YAML/frontmatter) e ferramenta `skill` |
| `subagent` | 10 | 77 | 35 | A | Subagentes in-process, fork e spawn; drivers ACP, **Claude Code**, Codex e dsh-sdk |
| `hooks` | 3 | 15 | 18 | A | Protocolo de hooks, compatível com **Claude Code** e Codex |
| `mcp` | 2 | 13 | 10 | A | Cliente MCP e recursos MCP |
| `sandbox` | 4 | 26 | 22 | A | Políticas de sandbox; ACL no Windows; Landlock no Linux |
| `guard` | 2 | 2 | 2 | A | Timeout e lembrete contra repetição de ferramenta |
| `interaction` | 5 | 15 | 9 | A | Aprovação do usuário, presets de permissão, comandos, `ask-user` |
| `shell` | 10 | 26 | 23 | A | Ferramentas Bash/PowerShell (local, sandbox, persistente) |
| `subprocess` | 3 | 32 | 22 | A | Processos, PTY (node-pty), Win32 |
| `fs` | 8 | 34 | 22 | A | Ferramentas de arquivo, busca com ripgrep, `str-replace-editor` |
| `plan` | 1 | 4 | 4 | A | Plan mode |
| `todo` | 1 | 4 | 5 | A | Ferramenta de todo |
| `workflow` | 4 | 21 | 14 | A | Workflows/orquestração (PTC), `ralph` |
| `goal` | 4 | 18 | 7 | A | Metas e rodadas autônomas (`/goal`) |
| `settings` | 2 | 7 | 8 | A | Settings em arquivo |
| `credentials` | 3 | 9 | 10 | A | Credenciais e autorização |
| `preset` | 2 | 13 | 13 | A | Presets de agente e persona (YAML) |
| `extensions` | 4 | 43 | 15 | B | Runners de extensões cordis (cliente/host) |
| `acp` | 1 | 8 | 11 | B | Agent Client Protocol (integração com IDEs) |
| `jobs` / `schedule` | 3 / 1 | 6 / 11 | 5 / 8 | B | Jobs em background e agendamento |
| `web` | 6 | 23 | 15 | B | `web-fetch` e `web-search` (DeepSeek, Exa, Perplexity) |
| `lsp` | 3 | 17 | 12 | B | Cliente LSP via stdio e ferramenta LSP |
| `terminal` | 3 | 8 | 10 | B | Terminal headless (xterm) |
| `spill` | 3 | 8 | 5 | B | Despejo em disco de saídas grandes |
| `storage` / `session-query` | 4 / 4 | 20 / 32 | 5 / 16 | B | Storage JSON/SQLite; consulta e exportação de sessões |
| `attachment` | 2 | 15 | 13 | B | Anexos (imagens via sharp) |
| `ptc-runtime` | 2 | 18 | 14 | B | Runtime de *programmatic tool calling* (Node) |
| `boot` / `bundle` | 2 / 6 | 9 / 12 | 11 / 13 | B | Boot da aplicação, cmdline, perfis de composição |
| `typert` | 4 | 18 | 11 | B | Gerador e registry de tipos/schemas em runtime |
| `api` / `sdk` | 7 / 3 | 93 / 14 | 69 / 7 | B | Gateway WebSocket, controllers, protocolo do SDK |
| `util` / `vendor` | 16 / 9 | 23 / 34 | 18 / 0 | B | Utilitários; cordis (DI/plugins), schemastery |
| `app:cli` | 1 | 56 | 10 | B | Bin `dsh` |
| `client` | 53 | 766 | 377 | C | UI React (chat, settings, model selection, sidebar...) |
| `app:web` / `app:desktop` / `host` | — | 124 / 40 / 28 | — | C | Frontend, Electron, servidor estático e directory picker |
| `experimental` | 16 | 303 | 89 | C | agent-team, inspector, browser-use, computer-use, webworker-runtime |
| `ssh`, `webhook`, `feedback`, `identity`, `browser-use`, `computer-use`, `workspace`, `runtime-diagnostics`, `test-support`, `native`, `python` | — | — | — | C | Diversos |

## 6. Persistência (superficial — análise detalhada fica com o Data Master)

| Caminho | Tipo |
|---|---|
| `packages/session/session-persistence-jsonl` | Log de sessão em JSONL (escrita atômica via koffi/MoveFileExW no Windows) |
| `packages/session/session-format*` | Schema de sessão com migrações v0→v1→v2→v3 |
| `packages/storage/storage-sqlite`, `session-query/session-query-sqlite` | SQLite (índice e consulta de sessões) |
| `packages/storage/storage-json` | Store JSON |
| `settings/settings-file`, `credentials/credentials-local` | YAML no `~/.dsh` |

## 7. CI/CD e build

- **GitHub Actions** (19 workflows): `ci`, `ci-master`, `e2e` (API real da DeepSeek), `pi-ai-provider-e2e` (Azure OpenAI e Anthropic), `sandbox`, `release*`, `python-release`, `node-addon-system*`, `docs-pages`, gestão de issues
- **GitLab CI**: publicação do wheel Python a partir de tags `python-v*`
- **Build**: `tsc -b` + `tsdown` (faces `host` e `client`); exe único via `@yao-pkg/pkg` (distribuído pelo `python/sdk-runtime`); Electron via electron-builder (NSIS no Windows)
- **Qualidade**: oxlint, jscpd, lefthook, publint
- Sem Dockerfile ou docker-compose 🟢

## 8. Testes

| Framework | Escopo |
|---|---|
| Vitest 4 | unit (`vitest.config.ts`), e2e, expected, snapshot, web, web-stress, web-perf, bench |
| pytest | SDK Python (6 arquivos `test_*.py`) |

- **1.119** arquivos `*.test.ts(x)` / `*.spec.ts` em TS, além de diretórios `tests/` por pacote 🟢
- Cobertura via `@vitest/coverage-v8`, particionada (`scripts/run-coverage-partitions.ts`) 🟢
- Testkits próprios: `llm-mock-server`, `llm-replay`, `agent-loop-testkit`, `session-snapshot` 🟢. Úteis como **oráculo de comportamento** para o port em Rust 🟡

## 9. Observações relevantes para o objetivo (harness em Rust)

1. 🟢 **Já existe compatibilidade com o ecossistema Claude Code**: `hooks-claude-code`, `subagent-claude-code` (usa `@anthropic-ai/claude-agent-sdk`) e `context/agent-instructions`. A spec de compatibilidade com `D:/Projetos/Totvs AGP/VSCode` pode partir desses pacotes.
2. 🟢 **Multi-provedor já resolvido conceitualmente**: `llm-pi-ai` lê catálogos de modelos (pi-ai) e aceita gateways declarados à mão (OpenAI-compatível, self-hosted) sem mudar código. É o modelo de referência para "modelos gratuitos listados ou cadastrados manualmente" e para assinaturas via OAuth.
3. 🟡 A arquitetura de plugins (cordis + YAML) pede uma decisão de projeto no Rust: traits com registro estático, ou plugins dinâmicos. Fica para o Architect.
4. 🟡 A UI (`client`, 766 arquivos) é a maior parte do código, mas tem prioridade baixa para um harness CLI.
5. 🔴 **Segurança**: `dsh.cmd` contém chaves de API em texto puro e desativa a verificação TLS (`NODE_TLS_REJECT_UNAUTHORIZED=0`). Os valores foram omitidos deste inventário de propósito.

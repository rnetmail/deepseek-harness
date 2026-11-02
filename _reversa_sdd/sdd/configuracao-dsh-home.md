# Configuração e layout de `~/.dsh` (`harness-core::config`)

## Visão Geral
Regras comuns para todos os arquivos de configuração e de estado do dsh Rust: onde ficam, qual vence quando dois dizem coisas diferentes, como são validados e recarregados e como o `~/.dsh` do dsh TS é migrado. Esta spec existe porque as outras specs criaram arquivos sem uma regra única (lacuna G1 da revisão). 🟡

## Responsabilidades
- Definir o layout de `~/.dsh` e de `.dsh/` no workspace. 🟡
- Definir a precedência entre o padrão embutido, o usuário, o workspace, a CLI e as configurações do Claude Code. 🟡
- Validar os arquivos no arranque e recarregá-los a quente sem perder o último estado válido. 🟢 (semântica do `settings-file` do dsh TS)
- Migrar o `settings.yaml` e o `.credentials.yaml` do dsh TS. 🟡
- Separar **configuração** (editável pelo usuário) de **estado** (gerido pelo dsh). 🟡

## Interface

### Layout
```
~/.dsh/                         (DSH_HOME; Windows: %USERPROFILE%\.dsh)
├─ config.toml                  configuração geral (permissões padrão, TUI, compactação, web, sandbox)
├─ providers.toml               provedores, modelos, aliases, fallback (spec 2)
├─ integrations.toml            browser/computer use, atualização automática (specs 7, 10)
├─ lsp.toml                     servidores de linguagem (spec 11)
├─ memory.toml                  GBrain e memória (spec 9)
├─ sync.toml                    fontes do catalog-sync (spec 5)
├─ AGENTS.md                    instruções globais do usuário
├─ skills/  agents/  commands/  conteúdo do usuário no formato Claude Code
├─ state/                       ESTADO (não editar à mão)
│  ├─ integrations.json  plugins.json  upstream.json  migration.json
├─ sessions/                    sessões v4, lendo também v3 (spec 1, U1)
├─ index/                       SQLite: FTS, PL/SQL, storage (derivados, reconstruíveis)
├─ attachments/  spill/         blobs (spec retencao-anexos-spill)
├─ plugins/  synced/  rea/  backups/  upstream/  logs/
└─ (legado dsh TS: settings.yaml, profiles/, storages/, cordis.patch.yml — só lidos na migração)

<workspace>/.dsh/
├─ workspace.toml               política, gates, aliases e LSP do workspace (specs 2, 4, 9, 11)
└─ lsp/                         projetos auxiliares (spec 11)
```
Todos os arquivos `.toml` são opcionais. A ausência de um arquivo significa "usar os padrões". 🟡

### Precedência (a maior vence)
1. Flags da CLI (`--model`, `--permission-mode`, `--settings`, `--mcp-config`…).
2. Variáveis de ambiente `DSH_*` (por exemplo, `DSH_HOME`, `DSH_MODEL`, `DSH_LOG`).
3. `<workspace>/.dsh/workspace.toml`.
4. Configurações do Claude Code no projeto: `.claude/settings.local.json` > `.claude/settings.json` (spec 4).
5. `~/.dsh/*.toml`.
6. `~/.claude/settings.json`.
7. Padrões embutidos.

**Exceções:**
- **Política de dados e gates** (`allowed_providers`, `data_gate`, `network_gate`): o workspace só pode **restringir** em relação ao usuário, nunca ampliar. Uma CLI que tente ampliar além do `workspace.toml` é recusada.
- **Regras `deny`** de qualquer camada sempre valem (spec 12, R7).
- **Listas** (`permissions.allow`, `hooks`, `mcpServers`) são concatenadas entre camadas, como no Claude Code. **Escalares e tabelas** seguem a precedência. 🟡

### Comandos
| Comando | Efeito |
|---|---|
| `dsh config show [--effective] [--origin]` | Mostra a configuração efetiva e, com `--origin`, de qual camada veio cada valor |
| `dsh config get <chave>` / `set <chave> <valor> [--workspace]` | Lê e grava preservando comentários |
| `dsh config validate` | Valida todos os arquivos e aponta linha e coluna dos erros |
| `dsh config migrate [--dry-run]` | Repete a migração do dsh TS |

## Regras de Negócio
- **R1.** Um arquivo inválido **no arranque** impede o carregamento daquele arquivo e mostra linha e coluna. O dsh sobe com os padrões e avisa, exceto para `providers.toml` sem nenhum provedor válido, que gera erro fatal com instrução. 🟢 (semântica do `settings-file`) / 🟡
- **R2.** Uma edição inválida **em runtime** gera aviso e mantém a última versão válida. O watch usa debounce de 100 ms. 🟢
- **R3.** A escrita pelo dsh (`config set`, `allow_always`, `login`) preserva comentários e ordem (`toml_edit`), é atômica (temporário + rename + fsync) e usa um lock entre processos com prazo de 2 s. 🟢 (padrão do dsh TS) / 🟡
- **R4.** Os arquivos de configuração **nunca** guardam segredos. Um valor que pareça segredo (padrões de chave e token) num `.toml` gera aviso, e `dsh config validate` recomenda movê-lo para o cofre com `dsh login`/`${secret:…}`. 🟡
- **R5.** Permissões: `~/.dsh` é 0700 no POSIX; no Windows a ACL fica restrita ao usuário. 🟢 (padrão do dsh TS) / 🟡
- **R6.** A pasta `state/` é do dsh. Uma edição manual que torne um arquivo inválido faz o dsh reconstruí-lo a partir das fontes (por exemplo, rescanear os plugins) e guardar o inválido como `.bak`. 🟡
- **R7.** As pastas `index/` são **derivadas** e podem ser apagadas: o dsh as reconstrói a partir das sessões e dos arquivos. 🟢 (FTS derivado do dsh) / 🟡
- **R8. Migração do dsh TS** (spec 1, R21b), feita uma vez e registrada em `state/migration.json`:
  - `settings.yaml`: `llm-pi-ai.providers` → `providers.toml`; `agent-default-model` → `providers.toml [default]`; `agent-loop.maxParallelToolCalls` → `config.toml`;
  - `.credentials.yaml` → cofre do SO (o arquivo só é removido com confirmação);
  - `profiles/`, `cordis.patch.yml` e `storages/` são ignorados, com uma nota no relatório;
  - `sessions/` é usado no lugar. 🟡
- **R9.** `DSH_HOME` muda a raiz inteira (testes, perfis isolados). Não existe "perfil" cordis no alvo; perfis viram diretórios `DSH_HOME` diferentes. 🟢 (variável do dsh) / 🟡

## Fluxo Principal
1. No arranque, o dsh resolve `DSH_HOME` e carrega os padrões e depois `~/.dsh/*.toml`. 🟡
2. Ao abrir a sessão, localiza o workspace e aplica as camadas 6 → 4 → 3, depois as variáveis de ambiente e por último a CLI. 🟡
3. Valida as restrições (exceções da precedência). Se algo tentar ampliar a política, o arranque falha com a explicação. 🟡
4. Inicia o watch dos arquivos. Uma mudança válida é publicada para os componentes, e uma inválida mantém o último estado válido (R2). 🟢/🟡

## Fluxos Alternativos
- **Primeiro uso, sem `~/.dsh`:** o dsh cria o layout mínimo e roda um assistente na TUI (login de um provedor, política padrão). 🟡
- **`~/.dsh` do dsh TS presente:** a migração R8 é oferecida no primeiro arranque. 🟡
- **Dois processos gravando ao mesmo tempo:** o lock com prazo de 2 s e o merge por diff. Se o prazo esgotar, a escrita falha com uma mensagem. 🟢/🟡

## Dependências
- `toml_edit`, `serde`, `notify`, `fs4` (locks), `keyring`, `directories`.
- Todas as specs que definem arquivos de configuração (1, 2, 4, 5, 7, 9, 10, 11, 12).

## Requisitos Não Funcionais
| Tipo | Requisito inferido | Evidência no código | Confiança |
|---|---|---|---|
| Disponibilidade | Último estado válido mantido em edição inválida | `packages/settings/settings-file` | 🟢 |
| Consistência | Escrita atômica com lock de 2 s | `packages/settings/settings-file` | 🟢 |
| Segurança | 0600/0700 e nenhum segredo em arquivo | `packages/settings/settings-file`; ADR 007 | 🟢/🟡 |
| Usabilidade | `config show --origin` | — (novo) | 🟡 |

## Critérios de Aceitação
```gherkin
Dado allowed_providers = [gemini, ollama] no workspace.toml do Totvs
Quando o usuário roda dsh --model openrouter-free/x nesse workspace
Então o arranque recusa com a explicação de que o workspace restringe os provedores

Dado um providers.toml editado com erro de sintaxe durante a sessão
Quando o arquivo é salvo
Então aparece um aviso com linha e coluna e a configuração anterior continua valendo

Dado um ~/.dsh do dsh TS com settings.yaml e .credentials.yaml
Quando o dsh Rust inicia pela primeira vez
Então os provedores aparecem em providers.toml, as chaves vão para o cofre e o YAML só é apagado se o usuário confirmar

Dado dsh config show --origin
Quando o usuário consulta permission_mode
Então vê o valor e a camada de onde veio
```

## Prioridade
| Requisito | MoSCoW | Justificativa |
|---|---|---|
| Layout + precedência + restrição de política | Must | Todas as specs dependem disso |
| Validação e hot-reload | Must | Herança do dsh, evita quebrar a sessão |
| Migração do dsh TS | Must | Uso exclusivo do executável Rust |
| `config show --origin` | Should | Diagnóstico |
| Assistente de primeiro uso | Could | Conveniência |

## Rastreabilidade de Código
| Arquivo | Função / Classe | Cobertura |
|---|---|---|
| `packages/settings/settings`, `settings-file` | camadas, watch, escrita | 🟢 |
| `packages/util/home-paths` | `DSH_HOME` | 🟢 |
| `apps/cli/src/profile-boot.ts` | perfis (não portados) | 🟢 |
| — | layout, precedência, migração | — (novo) |

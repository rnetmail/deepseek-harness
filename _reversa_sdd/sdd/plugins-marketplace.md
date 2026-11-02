# Plugins e marketplaces no formato Claude Code (`harness-claude-compat::plugins`)

## Visão Geral
O dsh Rust lê, instala, atualiza e executa plugins no **formato Claude Code**. Um plugin é uma pasta com:
- `.claude-plugin/plugin.json`;
- os componentes `commands/`, `agents/`, `skills/`, `hooks/hooks.json` e `.mcp.json`.

Plugins são distribuídos por **marketplaces**, que são repositórios ou pastas com `.claude-plugin/marketplace.json`.

Casos de uso confirmados:
- o marketplace **coreos/ai-helpers** (Apache-2.0);
- o plugin do **REA**, publicado no pacote npm `rea-agents`;
- os plugins que o usuário já tiver instalado no Claude Code.

Decisão `claude-plugin-format`, ADR 103. 🟢 formato observado / 🟡 campos documentados pela Anthropic e não observados nos repositórios.

O dsh TS **não** tem esse suporte: as extensões dele são pacotes cordis. Este componente é novo.

## Responsabilidades
- Registrar marketplaces (GitHub `owner/repo`, URL git, caminho local) e listar os plugins de cada um. 🟡
- Instalar, atualizar, habilitar, desabilitar e remover plugins. 🟡
- Ler os plugins **já instalados pelo Claude Code** (`~/.claude/plugins`) sem duplicá-los. 🟡
- Carregar os componentes de cada plugin habilitado nos registros do dsh: comandos, agentes, skills, hooks e MCP. 🟡
- Aplicar namespace `<plugin>:<item>` e resolver `${CLAUDE_PLUGIN_ROOT}`. 🟡
- Pedir confiança explícita antes de habilitar um plugin que traga hooks ou MCP, porque ambos executam código. 🟡

## Interface

### Formatos de arquivo
**`marketplace.json`** 🟢 (ai-helpers)
```json
{ "name": "ai-helpers", "owner": { "name": "coreos" },
  "plugins": [ { "name": "jenkins-analyzer", "source": "./plugins/jenkins-analyzer",
                 "description": "…", "version?": "…" } ] }
```
`source` pode ser um caminho relativo 🟢, `{ "source": "github", "repo": "o/r" }` ou `{ "source": "url", "url": "…git" }` 🟡.

**`plugin.json`** 🟡
| Campo | Obrig. | Uso |
|---|---|---|
| `name` | sim | Namespace (`/<name>:<cmd>`) |
| `version`, `description`, `author`, `homepage`, `repository`, `license`, `keywords` | não | Exibição e checagem de atualização |
| `commands`, `agents`, `hooks`, `mcpServers` | não | Caminhos alternativos. Sem eles, valem as pastas padrão |

**Componentes** 🟢/🟡
| Pasta | Frontmatter | Vira |
|---|---|---|
| `commands/*.md` | `description`, `argument-hint`, `allowed-tools`, `model` | `/<plugin>:<comando>` |
| `agents/*.md` | `name`, `description`, `tools`, `model` | `subagent_type` `<plugin>:<agente>` |
| `skills/<n>/SKILL.md` | `name`, `description`, … | skill `<plugin>:<n>` |
| `hooks/hooks.json` | `{ "hooks": { "<Evento>": [ { "matcher", "hooks": [ {type, command, timeout} ] } ] } }` | hooks (spec 13) |
| `.mcp.json` | `{ "mcpServers": { … } }` | servidores `mcp__<plugin>_<srv>__*` |

### Corpo de comandos 🟡
| Construção | Efeito |
|---|---|
| `$ARGUMENTS` | Texto inteiro após o comando |
| `$1`, `$2`… | Argumentos posicionais (separação estilo shell) |
| `` !`cmd` `` | Executa `cmd` **antes** do envio (sujeito a `allowed-tools`/permissões) e insere a saída |
| `@caminho` | Insere o conteúdo do arquivo (sujeito às regras de leitura) |
| `${CLAUDE_PLUGIN_ROOT}` | Caminho absoluto da pasta do plugin |

### Armazenamento
| Caminho | Conteúdo |
|---|---|
| `~/.dsh/plugins/marketplaces/<mkt>/` | Clone ou cópia do marketplace |
| `~/.dsh/plugins/cache/<mkt>/<plugin>/<versão>/` | Plugin instalado (imutável por versão) |
| `~/.dsh/plugins/installed.json` | `{plugin@mkt: {version, source_commit, sha256, enabled, trusted_at}}` |
| `~/.claude/plugins/` + `settings.json:enabledPlugins` | **Somente leitura**: plugins do Claude Code |

### CLI e slash commands
| Comando | Efeito |
|---|---|
| `dsh plugin marketplace add <owner/repo \| url \| caminho>` | Registra e baixa o marketplace |
| `dsh plugin marketplace list \| update [mkt] \| remove <mkt>` | Gerencia os marketplaces |
| `dsh plugin install <plugin>@<mkt>` | Instala (pedindo confiança se houver hooks ou MCP) |
| `dsh plugin list \| enable \| disable \| uninstall <plugin>@<mkt>` | Gerencia os plugins |
| `dsh plugin update [--all]` | Atualiza para a versão mais nova do marketplace |
| `dsh plugin validate <pasta>` | Valida um plugin ou marketplace em desenvolvimento |
| `/plugin …` na TUI | Mesmas ações, interativas |

## Regras de Negócio

### Descoberta e precedência
- **R1.** Fontes de plugins, em ordem: os instalados pelo dsh (`~/.dsh/plugins`), depois os do Claude Code habilitados em `enabledPlugins` (`~/.claude/plugins`). Um mesmo `plugin@mkt` presente nos dois locais é carregado **uma vez**, e vence a versão mais nova. 🟡
- **R2.** Itens de plugin são sempre namespaced (`<plugin>:<nome>`). Um item de projeto ou de usuário com o mesmo nome sem namespace tem precedência na invocação curta (`/nome`). O de plugin continua acessível pelo nome completo. 🟡
- **R3.** Um plugin habilitado num projeto via `.claude/settings.json → enabledPlugins` só vale nesse projeto. Os habilitados no usuário valem em todos. 🟡

### Instalação e confiança
- **R4.** A instalação baixa a versão do marketplace (commit fixado), grava o `sha256` da árvore e copia para o cache imutável. Uma atualização cria uma pasta de versão nova e troca o ponteiro; a versão anterior fica para rollback (`dsh plugin install p@m --version X`). 🟡
- **R5.** Um plugin com `hooks/` ou `.mcp.json` exige **confirmação explícita** na instalação e de novo em cada atualização que mude esses arquivos. A confirmação mostra os comandos que serão executados. 🟡
- **R6.** Comandos, agentes e skills de plugin respeitam as permissões normais. `allowed-tools` de um comando só **restringe** o que pode rodar sem pedir; nunca amplia além das regras `deny`. 🟡
- **R7.** `` !`cmd` `` num comando passa pelo motor de permissões como uma chamada `Bash`. Se for negado, o comando não é enviado e o motivo é mostrado. 🟡
- **R8.** Um plugin com `plugin.json` inválido ou componente malformado é desabilitado com o erro em `dsh plugin list`. Os outros plugins continuam funcionando. 🟡

### Integração com o restante
- **R9.** Os hooks de plugins são concatenados aos das settings, depois deles, na ordem de instalação. `${CLAUDE_PLUGIN_ROOT}` é exportado como variável de ambiente para o processo do hook. 🟡
- **R10.** Os servidores MCP de plugin entram no registro com o prefixo do plugin. Valem as mesmas regras do cliente MCP (spec 7), inclusive o gate de rede e o cofre de segredos. 🟡
- **R11.** As políticas do workspace (`allowed_providers`, `data_gate`) valem também para agentes e comandos de plugin com `model:`. O alias é resolvido pelo mapa (spec 2). 🟡
- **R12. Plugin do REA:** quando o `harness-integrations` instala o `rea-agents` (spec 8), o `.claude-plugin/plugin.json` do pacote é registrado como plugin local (`source` = pasta do pacote npm). Assim a skill `reverse-engineer-anything` e o servidor `rea` vêm de uma única fonte. Se a skill já existir em `~/.claude/skills`, ela não é duplicada. 🟡
- **R13. ai-helpers:** `dsh plugin marketplace add coreos/ai-helpers` deve listar `pipeline-status`, `customer-cases`, `rhcos-cve-analyzer` e `jenkins-analyzer`. O conteúdo é específico de RHCOS e o uso é opcional. O valor está no formato, que serve de teste de conformidade. 🟢
- **R14.** O dsh não envia inventário de plugins a nenhum serviço. Não há equivalente a `plugin-package-inventory-deepseek` (ADR 101). 🟢

## Fluxo Principal
1. O usuário executa `dsh plugin marketplace add coreos/ai-helpers`. 🟡
2. O dsh clona (git via `gix`, sem git instalado) para `~/.dsh/plugins/marketplaces/ai-helpers/` e valida o `marketplace.json`. 🟡
3. O usuário executa `dsh plugin install jenkins-analyzer@ai-helpers`. O dsh resolve o `source`, valida o `plugin.json`, verifica se há hooks ou MCP (R5), copia para o cache e grava em `installed.json`. 🟡
4. Na próxima sessão, ou imediatamente com a TUI aberta, o dsh registra `/jenkins-analyzer:analyze-failures` e o agente `jenkins-analyzer:failure-analyzer`. 🟡
5. O usuário digita `/jenkins-analyzer:analyze-failures <url>`. O dsh substitui `$ARGUMENTS`, executa os `` !`cmd` `` permitidos e envia o prompt. 🟡

## Fluxos Alternativos
- **Marketplace inacessível (sem rede, repositório privado):** erro com a causa. Os plugins já instalados continuam funcionando a partir do cache. 🟡
- **Atualização muda hooks:** a atualização fica pendente até a confirmação (R5). 🟡
- **Conflito de nome entre dois plugins:** sem conflito, porque o namespace separa. A forma curta `/nome` só é aceita quando ela for única. 🟡
- **Plugin do Claude Code habilitado, mas com a pasta ausente:** aviso em `dsh plugin list` e item ignorado. 🟡
- **Hook de plugin que usa um evento sem suporte:** aviso e hook ignorado (spec 13). 🟢 (semântica do dsh)

## Dependências
- Compatibilidade com o Claude Code (spec 4): mesmos carregadores de agents, skills e commands.
- Hooks e subagentes (spec 13), cliente MCP (spec 7), integrações (spec 8, REA).
- Crates: `gix` (git puro Rust), `serde_json`, `sha2`, `reqwest` (tarball de release como alternativa ao git).

## Requisitos Não Funcionais
| Tipo | Requisito inferido | Evidência no código | Confiança |
|---|---|---|---|
| Segurança | Confirmação explícita para plugins com hooks ou MCP | ADR 103; hooks executam comandos arbitrários | 🟡 |
| Integridade | Versões imutáveis no cache + `sha256` + rollback | padrão CoreOS (vendoring reproduzível) | 🟡 |
| Privacidade | Nenhum inventário enviado | ADR 101 | 🟢 |
| Disponibilidade | Plugins instalados funcionam offline | — (novo) | 🟡 |

> Novo componente. Validar com a documentação oficial de plugins do Claude Code antes de implementar.

## Critérios de Aceitação
```gherkin
Dado nenhum marketplace registrado
Quando o usuário roda dsh plugin marketplace add coreos/ai-helpers
Então dsh plugin list --available mostra pipeline-status, customer-cases, rhcos-cve-analyzer e jenkins-analyzer

Dado jenkins-analyzer instalado
Quando o usuário digita /jenkins-analyzer:analyze-failures https://jenkins/x
Então o prompt enviado contém a URL no lugar de $ARGUMENTS
E o agente jenkins-analyzer:failure-analyzer aparece no catálogo de Task

Dado um plugin com hooks/hooks.json
Quando o usuário o instala
Então o dsh mostra os comandos dos hooks e só habilita o plugin após confirmação

Dado um comando de plugin com !`git log -5` e uma regra deny Bash(git:*)
Quando o comando é invocado
Então ele não é enviado e o usuário vê a regra que bloqueou

Dado o REA instalado pelo harness-integrations
Quando o dsh inicia
Então existe exatamente uma skill reverse-engineer-anything e um servidor MCP rea
```

## Prioridade
| Requisito | MoSCoW | Justificativa |
|---|---|---|
| Leitura de plugins já instalados no Claude Code | Must | Reaproveitamento automático do ambiente do usuário |
| Carregar comandos, agentes, skills, hooks e MCP de plugin | Must | Decisão `claude-plugin-format` |
| Confiança explícita para hooks e MCP | Must | Segurança |
| `marketplace add` / `install` / `update` | Should | Instalação de plugins novos (ai-helpers) |
| Rollback por versão | Could | Uso raro |
| `plugin validate` | Could | Útil para desenvolver plugins próprios do time |

## Rastreabilidade de Código
| Arquivo | Função / Classe | Cobertura |
|---|---|---|
| https://github.com/coreos/ai-helpers `.claude-plugin/marketplace.json` | formato do marketplace | 🟢 |
| `plugins/*/.claude-plugin/plugin.json`, `commands/*.md` (ai-helpers) | formato de plugin e comando | 🟢 |
| pacote npm `rea-agents`: `.claude-plugin/plugin.json` | plugin do REA | 🟢 |
| `packages/hooks/hooks-claude-code` (config `pluginRoot`) | `${CLAUDE_PLUGIN_ROOT}` | 🟢 |
| `packages/extensions/*` (cordis) | modelo de extensão do dsh TS, **não** portado | 🟢 |
| — | marketplace, instalação, cache, namespaces | — (novo) |

## Atualização upstream 0.2.1-rc.5 (ciclo 1, 2026-11-01)
> Origem: [ledger](../upstream/ledger.md). Estas regras **prevalecem** sobre as anteriores quando houver conflito.

- **U1.** O upstream criou um **mercado de plugins próprio** (índice `plugin-market-index`, aba Discover, ferramenta `plugin_market`, instalação de bundles cordis). O dsh Rust **não** adota esse formato, porque os bundles são cordis, e mantém o formato Claude Code (decisão `claude-plugin-format`). A ideia de **índice com status de compatibilidade e argumentos exatos de instalação** é aproveitada em `dsh plugin list --available`. 🟡
- **U2.** Há uma proposta no upstream de compatibilidade com plugins e extensões **Pi** (`proposed/feature/2026-10-03-pi-*`). Fica só acompanhada. 🟡

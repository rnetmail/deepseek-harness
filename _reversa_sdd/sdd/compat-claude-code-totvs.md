# Compatibilidade com o Claude Code e mapeamento do Totvs AGP (`harness-claude-compat`)

## Visão Geral
Camada que faz o dsh Rust ler, **sem conversão nem cópia manual**, tudo o que um workspace do Claude Code já tem:
- `CLAUDE.md`;
- `.claude/agents`, `.claude/skills`, `.claude/commands`;
- `.claude/settings*.json` (permissões, hooks, env);
- `.mcp.json` e `~/.claude.json`;
- plugins e `.claude/agent-memory`.

Com isso, um projeto como `D:\Projetos\Totvs AGP\VSCode` (orquestradora Cláudia + 7 agentes + 24 skills) funciona no dsh quando o Claude estiver indisponível (ADR 103). 🟢 inventário confirmado / 🟡 comportamento do alvo.

## Responsabilidades
- Descobrir e carregar as fontes `.claude/*` do projeto e do usuário, com observação de mudanças (watch). 🟡
- Montar as seções de prompt a partir de `CLAUDE.md`, `AGENTS.md` e `*.local.md`. 🟢
- Registrar os agentes de `.claude/agents/*.md` como `subagent_type`, com filtro de ferramentas, alias de modelo e memória. 🟡
- Registrar as skills de `.claude/skills` e `~/.claude/skills` e os comandos de `.claude/commands`. 🟡
- Mesclar as permissões e os hooks na ordem do Claude Code. 🟡
- Importar os servidores MCP de `.mcp.json` e `~/.claude.json`, movendo segredos para o cofre do SO. 🟡
- Aplicar o mapa de aliases de modelo e a política de dados do workspace. 🟡
- Nunca ler o **conteúdo** de arquivos negados (como `.claude/.env`). 🟢

## Interface

### Fontes e precedência
| Fonte | Projeto | Usuário | Destino no dsh | Confiança |
|---|---|---|---|---|
| Instruções | `CLAUDE.md`, `CLAUDE.local.md`, `AGENTS.md`, `AGENTS.local.md` (da raiz até o cwd) | `~/.claude/CLAUDE.md`, `~/.dsh/AGENTS.md` | seção de prompt durável | 🟢 |
| Agentes | `.claude/agents/*.md` | `~/.claude/agents/*.md` | catálogo `subagent_type` | 🟡 |
| Skills | `.claude/skills/<n>/SKILL.md`, `.dsh/skills`, `.agents/skills` | `~/.claude/skills`, `~/.dsh/skills`, `~/.agents/skills` | catálogo de skills | 🟢 formato / 🟡 raízes `.claude` |
| Comandos | `.claude/commands/**/*.md` | `~/.claude/commands` | slash commands | 🟡 |
| Configurações | `.claude/settings.json`, `.claude/settings.local.json` | `~/.claude/settings.json` | permissões, hooks, env | 🟡 |
| MCP | `.mcp.json` | `~/.claude.json` (`mcpServers`, `projects[<cwd>].mcpServers`) | registro MCP | 🟡 |
| Plugins | — | `~/.claude/plugins` (marketplaces instalados) | agents, skills, commands, hooks, MCP | 🟡 (spec 6) |
| Memória de agente | `.claude/agent-memory/<agente>/` | `~/.claude/agent-memory/<agente>/` | memória por agente + GBrain | 🟡 (spec 9) |
| Política dsh | `.dsh/workspace.toml` | `~/.dsh/providers.toml` | política, aliases, gate | 🟡 (spec 2) |

**Precedência** (igual ao Claude Code), da maior para a menor:
1. política gerenciada;
2. CLI (`--settings`, `--allowedTools`);
3. `settings.local.json`;
4. `settings.json` do projeto;
5. `~/.claude/settings.json`.

Para skills, agents e commands de mesmo nome, vence **projeto > usuário > plugin > embutido**. Para skills, mantém-se o ranking do dsh: `.dsh` > `.claude` > `.agents` > custom > usuário > embutido. 🟡

### Frontmatter de agente (`.claude/agents/<nome>.md`)
| Campo | Tipo | Efeito | Confiança |
|---|---|---|---|
| `name` | string | `subagent_type` | 🟢 inventário |
| `description` | string | Entra no catálogo da ferramenta `Task`; orienta a delegação automática | 🟢 |
| `tools` | lista separada por vírgula | Máscara de ferramentas (nomes Claude Code + `mcp__*`); ausente = herda todas | 🟢 |
| `model` | `opus` \| `sonnet` \| `haiku` \| `inherit` \| id | Resolvido pelo mapa de aliases (spec 2, R16) | 🟢 |
| `memory` | `project` \| `user` \| `local` | Diretório de memória injetado e gravável (R9) | 🟢 |
| `color`, outros | — | Ignorados com aviso em nível debug | 🟡 |
| Corpo markdown | — | System prompt do subagente | 🟢 |

### Comandos
| Comando | Efeito |
|---|---|
| `dsh compat doctor [--path <ws>]` | Lista o que foi detectado (agentes, skills, comandos, permissões, MCP, hooks), aponta itens ignorados e por quê, e testa os aliases de modelo e o gate de rede |
| `dsh compat import-mcp` | Copia os MCP de `~/.claude.json` para o registro do dsh e move os headers secretos para o cofre do SO, deixando apenas uma referência |

## Regras de Negócio

### Instruções
- **R1.** A mensagem-base durável traz `~/.dsh/AGENTS.md`, `~/.claude/CLAUDE.md` e a cadeia do projeto, da raiz (marcada por `.git` ou `.claude/`) até o cwd, do mais amplo para o mais específico. Arquivos irmãos idênticos aparecem uma vez. O limite total é de 64 KiB e cada arquivo tem no máximo 1 MiB. 🟢 (raiz `.claude/` e `~/.claude/CLAUDE.md` 🟡)
- **R2.** Descoberta incremental: depois de um `Read`/`Write`/`Edit` num subdiretório com o próprio `CLAUDE.md`, ele entra no próximo request. Um arquivo alterado substitui o anterior; um removido sai. 🟢
- **R3.** Imports `@caminho` dentro do `CLAUDE.md` são expandidos (até 5 níveis, sem sair do workspace nem de `~`). 🟡
- **R4.** Uma falha de permissão ou de I/O na descoberta é erro explícito. A descoberta não cai em silêncio para um ancestral. 🟢

### Agentes e skills
- **R5.** Todo arquivo válido de `.claude/agents` vira um `subagent_type`. Um frontmatter inválido descarta só aquele agente, com aviso no `compat doctor`. 🟡
- **R6.** O `tools` do agente é intersectado com a máscara do agente pai e com as permissões (as máscaras só restringem). 🟢 (`ctx.tools.restrict`)
- **R7.** Um subagente **não** delega a outro subagente por padrão (profundidade 1), o que bate com a regra da Cláudia: "a sessão principal orquestra". As delegações em paralelo são permitidas (`subagent` é concurrency-safe no dsh 🟢). O padrão do dsh TS é `maxDepth = 3` (`packages/subagent/tool-subagent/src/index.ts:129`); o padrão 1 é uma escolha do alvo. 🟡 [Revisão: era 🟢]
- **R8.** Skills seguem o formato `<nome>/SKILL.md` com frontmatter `name` + `description` (mais `allowed-tools`, `disable-model-invocation`, `user-invocable`, `argument-hint`, `model`). O catálogo limita a descrição a 500 caracteres. `/nome` injeta a skill. `scripts/` e recursos ficam acessíveis pelo caminho da pasta. 🟢
- **R9. Memória `memory: project`:**
  - no início do subagente, o conteúdo de `.claude/agent-memory/<agente>/MEMORY.md` (e arquivos indicados nele) entra como seção do prompt, com limite de 25 KB;
  - o subagente pode gravar nesse diretório (permissão implícita `Write`/`Edit` restrita a ele);
  - cada gravação é espelhada no GBrain quando ele estiver ativo (spec 9). 🟡

### Permissões e segredos
- **R10.** Os arrays `permissions.allow`, `ask` e `deny` são concatenados entre as camadas. A avaliação é `deny` > `ask` > `allow` > modo padrão (`defaultMode`: `default` \| `acceptEdits` \| `plan` \| `bypassPermissions`). 🟡
- **R11.** A sintaxe suportada inclui:
  - `Bash(prefixo:*)` e `Bash(comando exato)`;
  - `PowerShell(...)`;
  - `Read|Edit|Write(caminho ou glob gitignore)`, com `./` relativo ao arquivo de settings e `//` absoluto;
  - `WebFetch(domain:x)`;
  - `mcp__srv`, `mcp__srv__tool`;
  - `Task(agente)`, `Skill(nome)`. 🟡
- **R12. `.claude/.env`:** um `deny` em `Read/Edit/Write(./.claude/.env)` vale para as ferramentas de arquivo **e** para o guarda anti-exfiltração do shell (spec 3, R15). A camada de compatibilidade só lê os **nomes** das variáveis, quando um script declarar que precisa delas, e nunca os valores. 🟢/🟡
- **R13. Variáveis do `.env` para scripts de skill:**
  - o dsh não injeta o `.env` no ambiente das ferramentas;
  - os scripts do Totvs que leem o `.env` diretamente (Python/Bash) continuam funcionando, porque é o processo do script, não o modelo, quem lê o arquivo;
  - um `Bash(python3 .claude/skills/...:*)` permitido executa normalmente. 🟡
- **R14.** O header `Authorization` de um servidor MCP vindo de `~/.claude.json` é movido para o cofre do SO na importação. O registro do dsh guarda só `secret_ref`. O `~/.claude.json` **não** é alterado. 🟡
- **R15.** `env` de `settings.json` é aplicado às ferramentas e aos hooks, exceto chaves que pareçam segredo (`*_KEY`, `*_TOKEN`, `*_SECRET`, `*PASSWORD*`), que exigem confirmação na primeira vez. 🟡

### Hooks e MCP
- **R16.** Os hooks de todas as camadas são concatenados por evento. Os matchers usam os nomes Claude Code, com aliases aceitos. A semântica de execução está na spec 13. 🟡
- **R17.** Servidores MCP de `.mcp.json` pedem aprovação na primeira vez em cada workspace (`enableAllProjectMcpServers`, `enabledMcpjsonServers` e `disabledMcpjsonServers` respeitados). Os de `~/.claude.json` são confiáveis. 🟡
- **R18.** Servidores listados em `network_gate.guards` só conectam com o gate aberto (spec 2, R13). 🟡

### Aderência e escrita governada (Totvs)
- **R19. Lista de modelos aprovados para escrita:**
  - skills ou agentes marcados como escrita governada (no Totvs, `rm-escrita-nativa` e os agentes com `Write`/`Edit` que gravam em HML) só operam com modelos de `write_approved_models` do workspace;
  - com outro modelo, as ferramentas de escrita do agente passam a `ask` em toda chamada.
  - **Decisão do usuário (2026-11-01):** o padrão do Totvs AGP é `write_approved_models = "*"`, ou seja, **todos os modelos disponíveis** (já filtrados pela política de dados, spec 2) podem escrever, sem o `ask` extra por modelo. O mecanismo continua configurável para restringir no futuro. As aprovações normais das permissões e dos modos (spec 12) continuam valendo. 🟢 [Revisão: era 🟡]
- **R20.** O dsh não reinterpreta regras de negócio do `CLAUDE.md` (datas de governança, liberações). Elas chegam ao modelo como texto. Com `write_approved_models = "*"` (R19), a bateria de conformidade `dsh compat eval` deixa de ser pré-requisito e vira um **diagnóstico opcional**: ela mede a aderência de cada modelo às regras do `CLAUDE.md` e informa o resultado, sem bloquear. 🟢 [Revisão: era 🔴; decisão do usuário]

### Mapeamento do Totvs AGP
| # | Recurso | Atendido por | Regra |
|---|---|---|---|
| R1 inv. | `CLAUDE.md` | Instruções | R1–R4 |
| R2 inv. | 7 agentes `.claude/agents` | Catálogo `subagent_type` | R5–R7 |
| R3 inv. | `memory: project` (6 diretórios) | Memória por agente + GBrain | R9 |
| R4 inv. | `model: opus` | Aliases → `gemini-2.5-pro` | spec 2 R16 |
| R5 inv. | 24 skills + `scripts/` | Raízes `.claude/skills` | R8 |
| R6/R8 inv. | `settings.json` + `.local` | Motor de permissões | R10–R11 |
| R7 inv. | deny `.env` | Guarda de arquivo + shell | R12 |
| R9 inv. | MCP `gpflow` (http + Authorization) | Importação + cofre + gate de rede | R14, R18 |
| R10 inv. | Orquestração em profundidade 1 | Subagentes | R7 |
| R11 inv. | Navegador Playwright/CDP com perfil dedicado | Browser use (spec 10) + `Bash(python…)` | spec 10 |
| R12 inv. | Git Bash | `Bash` resolve `bash.exe` do Git for Windows | spec 3 |
| R13 inv. | "Cláudia, …" | `CLAUDE.md` + skill `claudia` | R1, R8 |
| R14 inv. | WebSearch/WebFetch por domínio | Provedores sem custo | spec 3 R18–R19 |
| R15 inv. | Governança rígida | Aprovados para escrita + `compat eval` | R19–R20 |
| CatalogoSKILLs | Pacote gerado | Sincronização | spec 5 |

## Fluxo Principal
1. Ao abrir uma sessão com cwd X, o dsh localiza a raiz do projeto (`.git` ou `.claude/`) e lê `.dsh/workspace.toml`. 🟡
2. Roda o gate de rede e o de dados (spec 2). 🟡
3. Carrega as instruções (R1), as skills (R8), os agentes (R5), os comandos, as settings mescladas (R10) e os MCP (R17), filtrando o que depende da rede fechada. 🟡
4. Monta o catálogo de `Task` (agentes) e de `Skill` e as seções de prompt. 🟢/🟡
5. Inicia o watch dessas fontes. Uma mudança gera um catálogo substituto anexado no próximo passo, sem reiniciar. 🟢
6. O usuário escreve "Cláudia, …". A skill `claudia` e o `CLAUDE.md` orientam a delegação, e `Task(subagent_type="arquiteto-totvs-rm")` cria o subagente com tools, alias e memória dele. 🟡

## Fluxos Alternativos
- **Agente com `model: opus` e nenhum candidato permitido:** o subagente não inicia e o erro cita o alias e as políticas (spec 2). 🟡
- **Skill com o mesmo nome em `.claude/skills` e em `CatalogoSKILLs/`:** só `.claude/skills` é raiz. `CatalogoSKILLs/` é saída gerada (spec 5) e nunca é lida como fonte. 🟢 (diff mostrou cópias idênticas)
- **Fora da rede da empresa:** `gpflow` e `skill:gpflow-*` aparecem como indisponíveis no catálogo, com o motivo. 🟡
- **`settings.json` ilegível:** aviso; vale a camada anterior; hooks daquela camada não rodam; a sessão sobe. 🟢 (semântica do dsh)
- **Pasta-espelho (`skill-ellevo-db`):** duas skills com o mesmo `name` vindas do mesmo nível geram aviso, e vence a de caminho lexicograficamente menor. 🟡

## Dependências
- `harness-core`: seções de prompt, catálogo de ferramentas e máscaras.
- `harness-tools`: motor de permissões e guarda do shell.
- `harness-mcp`: registro dos servidores importados.
- `harness-llm`: aliases e política.
- Memória (spec 9), plugins (spec 6), sincronização (spec 5), hooks e subagentes (spec 13).
- Crates: `notify` (watch), `serde_yaml`/`gray_matter` (frontmatter), `globset` (padrões de permissão), `keyring`.

## Requisitos Não Funcionais
| Tipo | Requisito inferido | Evidência no código | Confiança |
|---|---|---|---|
| Desempenho | Instruções limitadas a 64 KiB no total e 1 MiB por arquivo | `packages/context/agent-instructions` (config base) | 🟢 |
| Desempenho | Descrição de skill no catálogo limitada a 500 caracteres; cache de 128 entradas | `packages/skill/tool-skill`, `packages/skill/skill` | 🟢 |
| Disponibilidade | Provedor de skill com falha é pulado e o último catálogo válido é mantido | `packages/skill/skill` | 🟢 |
| Segurança | Segredos de MCP no cofre; `.env` nunca lido pelo dsh | inventário Totvs, riscos 3 e 4 | 🟡 |
| Usabilidade | Mudanças em `.claude/*` aparecem sem reiniciar | `packages/skill/skill-filesystem` (watch) | 🟢 |

> Inferido a partir do código. Validar com a equipe de operações.

## Critérios de Aceitação
```gherkin
Dado o workspace D:\Projetos\Totvs AGP\VSCode
Quando o usuário roda dsh compat doctor
Então são listados 7 agentes, 23 skills distintas (+1 espelho com aviso), 43 regras allow, 3 deny e o MCP gpflow
E nenhum valor de .claude/.env ou do header Authorization aparece na saída

Dado o agente especialista-metadados com tools Read, Grep, Glob, WebSearch, WebFetch, Bash, Write, Edit e memory: project
Quando a Cláudia o delega
Então o subagente só enxerga essas ferramentas e recebe o MEMORY.md do seu diretório no prompt
E uma gravação em .claude/agent-memory/especialista-metadados/ é permitida sem pedir aprovação

Dado uma skill nova criada em .claude/skills/nova-skill/SKILL.md com a sessão aberta
Quando o próximo passo começa
Então o catálogo substituto inclui nova-skill e /nova-skill funciona

Dado write_approved_models = "*" no Totvs AGP
Quando qualquer modelo permitido pela política chama Edit num arquivo de entrega de HML
Então só as regras normais de permissão se aplicam, sem aprovação extra por modelo

Dado write_approved_models restrito a [gemini/gemini-2.5-pro] e outro modelo ativo
Quando o agente chama Edit num arquivo de entrega de HML
Então a chamada pede aprovação humana

Dado ~/.claude.json com gpflow e header Authorization
Quando dsh compat import-mcp roda
Então o token vai para o Gerenciador de Credenciais e o registro do dsh guarda só secret_ref
E ~/.claude.json fica inalterado
```

## Prioridade
| Requisito | MoSCoW | Justificativa |
|---|---|---|
| Agentes `.claude/agents` + aliases + máscara de tools | Must | Requisitos críticos R2/R4 do Totvs |
| Skills de `.claude/skills` e `~/.claude/skills` | Must | Requisito crítico R5; inclui as skills Reversa |
| Permissões mescladas + deny `.env` | Must | Requisito crítico R7 |
| Memória por agente | Must | Requisito crítico R3 |
| `CLAUDE.md` + descoberta incremental | Must | Já existe no dsh; base de toda sessão |
| Importação de MCP com cofre | Should | O `gpflow` depende disso, mas só na rede da empresa |
| Comandos `.claude/commands` | Should | Não usados hoje no Totvs; usados por plugins |
| Aprovados para escrita (padrão "*") + `compat eval` opcional | Could | Decisão do usuário: todos os modelos; o mecanismo fica para restrição futura. Mitiga o risco de escrita por modelo mais fraco |
| Imports `@` no `CLAUDE.md` | Could | Não usados no Totvs hoje |

## Rastreabilidade de Código
| Arquivo | Função / Classe | Cobertura |
|---|---|---|
| `packages/context/agent-instructions` | cadeia `CLAUDE.md`/`AGENTS.md`, descoberta incremental | 🟢 |
| `packages/skill/skill`, `skill-filesystem`, `tool-skill` | registro, raízes, catálogo | 🟢 |
| `packages/subagent/*` | delegação e profundidade | 🟢 |
| `packages/hooks/hooks-claude-code`, `hook-protocol` | leitura de `hooks` | 🟢 |
| `packages/mcp/*` | cliente MCP | 🟢 |
| `packages/interaction/permission-presets`, `user-approval` | presets e aprovação | 🟢 |
| `_reversa_sdd/totvs-agp-inventory.md` | inventário R1–R15 | 🟢 |
| — | `.claude/agents`, `agent-memory`, `settings*.json`, `~/.claude.json`, aprovados para escrita | — (novo) |

## Atualização upstream 0.2.1-rc.5 (ciclo 1, 2026-11-01)
> Origem: [ledger](../upstream/ledger.md). Estas regras **prevalecem** sobre as anteriores quando houver conflito.

- **U1. Instruções globais:** a cadeia do usuário tem duas raízes, nesta ordem: `$DSH_HOME/AGENTS.md` e depois `$DSH_AGENTS_HOME/AGENTS.md` (`~/.agents`), deduplicadas por conteúdo. No dsh Rust, `~/.claude/CLAUDE.md` entra logo depois (R1). 🟢/🟡
- **U2.** Valores específicos do processo (caminho do checkout, URL) vão para o **contexto de runtime**, não para o system prompt, o que preserva o cache. 🟢

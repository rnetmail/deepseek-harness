# Memória persistente: GBrain + memória por agente + memória automática (`harness-claude-compat::memory` + integração GBrain)

## Visão Geral
O dsh oferece **três camadas de memória**, todas compatíveis com o Claude Code, para que a continuidade se mantenha quando o Claude estiver indisponível:
1. **Memória por agente** (`.claude/agent-memory/<agente>/`): arquivos markdown que os subagentes do Totvs AGP já usam (`memory: project`). 🟢
2. **Memória automática do usuário** (`~/.claude/projects/<slug-do-cwd>/memory/`): `MEMORY.md` como índice mais um arquivo por fato, com frontmatter `name`/`description`/`metadata.type`. É o mesmo formato que o Claude Code usa hoje. 🟢
3. **GBrain** (garrytan/gbrain, MIT): memória curada com fatos citados, correção e retirada, busca híbrida, grafo e `think`, servida por MCP. Os markdowns das camadas 1 e 2 são espelhados nele. 🟢 (decisão `gbrain-integration`)

O histórico bruto das sessões fica com o FTS (spec 3, R23). O GBrain guarda o que foi **curado**. 🟡

## Responsabilidades
- Carregar a memória certa em cada contexto: a memória do projeto na sessão principal e a memória do agente no subagente. 🟡
- Permitir que o agente grave memórias no formato Claude Code, sem pedir aprovação, dentro dos diretórios de memória. 🟡
- Instalar, inicializar e registrar o GBrain com engine local (PGLite) e embeddings locais. 🟡
- Injetar a seção de instruções **brain-first** no prompt. 🟡
- Espelhar os markdowns de memória no GBrain, por fonte, respeitando a política de dados de cada workspace. 🟡
- Compartilhar o **mesmo brain** com o Claude Code, quando o usuário também o configurar lá. 🟡
- Fazer backup do brain. 🟡

## Interface

### Formato de memória (camadas 1 e 2) 🟢
```markdown
---
name: ambiente-homologacao-oracle
description: Ambiente RM de homologação (TOTVSRMHOMOL) - Oracle, schema RM, …
metadata:
  type: reference          # user | feedback | project | reference
---
<fato; para feedback/project: **Why:** e **How to apply:**; links [[outro-nome]]>
```
O `MEMORY.md` é o índice, com uma linha por memória: `- [Título](arquivo.md) — gancho`. 🟢 (observado em `.claude/agent-memory/analista-totvs-rm/` e em `~/.claude/projects/*/memory/`)

### Locais
| Camada | Caminho | Quem lê | Quem grava |
|---|---|---|---|
| Agente (`memory: project`) | `<ws>/.claude/agent-memory/<agente>/` | o subagente | o subagente |
| Agente (`memory: user`) | `~/.claude/agent-memory/<agente>/` | o subagente | o subagente |
| Automática | `~/.claude/projects/<slug>/memory/` (slug = cwd com `:\/` trocados por `-`, ex.: `d--dsh`) | sessão principal | sessão principal |
| GBrain | `~/.gbrain/` (PGLite) + repositório markdown do brain | MCP `gbrain` | MCP `gbrain` + espelhamento |

### Configuração (`~/.dsh/memory.toml` e `.dsh/workspace.toml`)
```toml
[gbrain]
enabled      = true
surface      = "starter"            # verbos de memória
engine       = "pglite"
embeddings   = "ollama:nomic-embed-text"   # ou "none" (só keyword); Voyage/OpenAI só com chave do usuário
memorable    = false                # relay externo: sempre desligado por padrão
backup       = { every = "daily", keep = 14, dir = "~/.dsh/backups/gbrain" }

[memory.mirror]                     # espelhamento markdown → GBrain
auto_memory  = true
agent_memory = true
```
```toml
# .dsh/workspace.toml do Totvs AGP
[memory]
gbrain_source = "totvs-agp"         # fonte própria dentro do brain
scope         = "workspace"         # consultas fora do workspace não veem esta fonte
embeddings    = "local-only"        # proíbe embeddings remotos para esta fonte
```

### Ferramentas visíveis ao modelo
- `mcp__gbrain__*` da superfície `starter`: `search`, `query`, `recall`, `get_page`, `think`, `remember`, `put_page`, `takes_*`, `get_versions`… 🟢
- `Read`/`Write`/`Edit` nos diretórios de memória, com permissão implícita (R6). 🟡

## Regras de Negócio

### Carga no prompt
- **R1.** A sessão principal recebe o `MEMORY.md` da memória automática do cwd como seção de prompt, até 200 linhas ou 25 KB. Os arquivos individuais são lidos **sob demanda** pelo modelo. 🟡
- **R2.** Um subagente com `memory: project|user|local` recebe o `MEMORY.md` do seu diretório no lugar da memória automática, com o mesmo limite. Um subagente sem `memory` não recebe nenhuma memória. 🟡
- **R3.** A seção de memória vem com a instrução do Claude Code: as memórias refletem o que era verdade quando foram escritas, e nomes de arquivos, funções ou flags citados devem ser verificados antes de usados. 🟡
- **R4.** A seção **brain-first** entra quando o GBrain está `initialized`. Ela usa o **texto oficial** do GBrain, obtido com `gbrain setup chatgpt --instructions-only --dry-run` (gbrain 0.60.544.0, 2026-11-01). O bloco vai entre os marcadores `<!-- gbrain:brain-first:begin -->` e `<!-- gbrain:brain-first:end -->`, é versionado (v2) e é substituído em cada upgrade. Ele manda:
  - checar o GBrain **antes** de responder cada turno: `search` para nomes e frases, `query` para perguntas e `get_page` no melhor resultado;
  - preencher as lacunas com outras ferramentas e, quando vier pouco, tentar sinônimos, o nome formal, o dono ou termos combinados com OR;
  - restringir com `source_ids` ao que o solicitante pode ver e aplicar as próprias regras de acesso;
  - tratar os resultados como complemento do que o usuário disse, nunca como substituto, e terminar a tarefa inteira.

  No dsh, o bloco entra como **seção de prompt** (não é gravado no `CLAUDE.md`) e o `source_ids` é preenchido automaticamente pelo escopo do workspace (R13). 🟢 [Revisão: era 🔴]
- **R4b. Hooks do GBrain no Claude Code** (de `gbrain setup --help`):
  - por padrão, um `UserPromptSubmit` reforça a regra brain-first a cada turno;
  - com `--capture` (opt-in), os hooks de captura automática são `Stop` e `SessionEnd`;
  - o padrão é `--no-capture` e `--no-providers` (sem embeddings pagos).

  O dsh reproduz o reforço por turno internamente (sem hook) e só liga a captura com opt-in. 🟢

### Gravação
- **R5.** O agente grava memórias no formato do Claude Code: um arquivo por fato, mais uma linha no `MEMORY.md`. Antes de criar, verifica se já existe um arquivo que cubra o mesmo fato e atualiza esse arquivo. Memórias que se mostraram erradas são apagadas. 🟢 (regra do Claude Code) / 🟡 (aplicação no dsh)
- **R6.** `Write` e `Edit` dentro do diretório de memória do próprio agente (ou da memória automática, na sessão principal) são **permitidos sem aprovação**. Fora dele valem as regras normais. Um subagente não grava na memória de outro agente. Uma regra `deny` explícita continua prevalecendo sobre essa permissão implícita (spec 12, R7). 🟡 [Revisão]
- **R7.** Não se grava em memória o que o repositório já registra (estrutura de código, histórico git, `CLAUDE.md`) nem segredos. Um padrão de segredo detectado na gravação bloqueia a escrita (mesmo verificador da spec 5, R12). 🟡

### GBrain
- **R8.** Instalação: Bun ≥ 1.4.4, depois `bun install -g github:garrytan/gbrain#latest-stable`, depois `gbrain init --pglite` (com `--no-embedding` se não houver Ollama). A atualização usa o mesmo comando (spec 7). 🟢
- **R9.** O registro MCP é `gbrain serve --surface starter --advertise verbs`, por stdio. 🟢
- **R10.** **Memorable fica desligado** por padrão, porque envia traces e consultas a um relay externo. Ligá-lo exige um opt-in explícito com o aviso do que sai da máquina. 🟢
- **R11.** Embeddings locais (Ollama, llama.cpp, LM Studio) são preferidos. Na falta deles, `--no-embedding` (só busca por palavras-chave). Embeddings remotos só com a chave do usuário e **nunca** para fontes marcadas `embeddings = "local-only"`. 🟢/🟡
- **R12. Espelhamento:**
  - cada arquivo de memória (camadas 1 e 2) é importado no GBrain como página da fonte do workspace (`gbrain_source`), com `source_path` apontando para o markdown;
  - o markdown continua sendo a fonte da verdade, e uma remoção vira soft-delete no brain;
  - o espelhamento roda depois de cada gravação de memória e no arranque (reconciliação por hash). 🟡
- **R13. Escopo:**
  - uma fonte com `scope = "workspace"` (Totvs AGP) só é consultável em sessões dentro daquele workspace;
  - o dsh passa o filtro de fonte nas chamadas ao GBrain e rejeita resultados de fontes fora do escopo antes de entregá-los ao modelo. 🟡
- **R14.** O dsh e o Claude Code podem usar **o mesmo brain** (`~/.gbrain`). Quando o Claude voltar a estar disponível, o que foi aprendido no dsh já está lá, e vice-versa. O dsh não roda `gbrain setup claude-code` sozinho: oferece o comando para o usuário. 🟡
- **R15.** Backup diário com 14 cópias: export markdown + dump do PGLite em `~/.dsh/backups/gbrain/`. As páginas que só existem no banco dependem desse backup. 🟢 (README do GBrain) / 🟡 (agenda)
- **R16.** Sem Bun ou com o GBrain falhando, as camadas 1 e 2 continuam funcionando normalmente. Só a busca semântica e o `think` deixam de existir, e o modelo é avisado. 🟡

## Fluxo Principal
1. A sessão abre no Totvs AGP. O dsh carrega o `MEMORY.md` da memória automática do cwd e a seção brain-first. 🟡
2. A Cláudia delega ao `arquiteto-totvs-rm`. O subagente recebe o `MEMORY.md` de `.claude/agent-memory/arquiteto-totvs-rm/` (R2). 🟡
3. O subagente consulta `mcp__gbrain__search` (fonte `totvs-agp`) sobre "recálculo de ponto" e recebe as páginas espelhadas com as citações. 🟡
4. Ao concluir, ele grava `ponto-recalculo-estruturas.md` atualizado (R5, R6). 🟡
5. O espelhamento importa a mudança no GBrain (R12). 🟡

## Fluxos Alternativos
- **Duas memórias contraditórias:** o agente usa `takes_supersede` no GBrain e corrige o markdown. A versão anterior fica consultável (`get_versions`). 🟢/🟡
- **Arquivo de memória editado à mão no VSCode:** a reconciliação do arranque ou o watcher reimporta. 🟡
- **Claude Code e dsh gravando a mesma memória ao mesmo tempo:** a gravação é atômica (rename) e quem gravar por último vence. O GBrain mantém o histórico das versões. 🟡
- **Consulta fora do Totvs AGP sobre um assunto do Totvs:** a fonte `totvs-agp` não aparece (R13). 🟡

## Dependências
- Integrações gerenciadas (spec 7): instalação e registro do GBrain.
- Compatibilidade com o Claude Code (spec 4): frontmatter `memory:` dos agentes.
- Política de dados (spec 2): escopo e embeddings.
- Runtimes: Bun ≥ 1.4.4; Ollama opcional.

## Requisitos Não Funcionais
| Tipo | Requisito inferido | Evidência no código | Confiança |
|---|---|---|---|
| Privacidade | Memorable desligado; embeddings locais por padrão | README do GBrain §Memorable; decisão `gbrain-integration` | 🟢 |
| Privacidade | Fonte do Totvs restrita ao workspace | decisão `totvs-data-gate` | 🟡 |
| Desempenho | PGLite até cerca de 50 mil páginas | README do GBrain | 🟢 |
| Desempenho | Índice de memória no prompt limitado a 200 linhas ou 25 KB | regra do Claude Code (MEMORY.md truncado) | 🟡 |
| Durabilidade | Backup diário, 14 cópias | README do GBrain (backup do banco à parte) | 🟢/🟡 |
| Disponibilidade | Camadas 1 e 2 funcionam sem o GBrain | — (novo) | 🟡 |

> Inferido a partir do código e da documentação. Validar no piloto.

## Critérios de Aceitação
```gherkin
Dado .claude/agent-memory/analista-totvs-rm/MEMORY.md com 4 entradas
Quando a Cláudia delega ao analista-totvs-rm
Então o prompt do subagente contém as 4 linhas do índice e não contém a memória dos outros agentes

Dado uma sessão no cwd d:\dsh
Quando o agente aprende uma preferência do usuário
Então grava um arquivo em ~/.claude/projects/d--dsh/memory/ com frontmatter name/description/metadata.type
E adiciona uma linha ao MEMORY.md, sem pedir aprovação

Dado o GBrain inicializado e a fonte totvs-agp com scope workspace
Quando uma sessão em outro projeto pesquisa "TOTVSRMHOMOL"
Então nenhum resultado da fonte totvs-agp é devolvido

Dado o GBrain sem Bun instalado
Quando a sessão abre
Então as memórias markdown são carregadas normalmente e o modelo é avisado de que a busca semântica está indisponível

Dado qualquer configuração padrão
Quando o GBrain é registrado
Então Memorable está desligado e nenhum embedding é calculado por serviço remoto para a fonte totvs-agp
```

## Prioridade
| Requisito | MoSCoW | Justificativa |
|---|---|---|
| Memória por agente (`memory: project`) | Must | Requisito crítico R3 do Totvs |
| Memória automática no formato Claude Code | Must | Continuidade com o Claude Code do usuário |
| GBrain instalado e registrado (PGLite, local) | Must | Decisão `gbrain-integration` |
| Escopo por workspace + embeddings locais | Must | LGPD |
| Espelhamento markdown → GBrain | Should | O markdown sozinho já funciona |
| Seção brain-first (texto oficial v2) | Should | Melhora o uso do GBrain |
| Backup agendado | Should | Proteção das páginas que só existem no banco |
| Brain compartilhado com o Claude Code | Could | Depende da configuração do usuário no Claude |

## Rastreabilidade de Código
| Arquivo | Função / Classe | Cobertura |
|---|---|---|
| `D:/Projetos/Totvs AGP/VSCode/.claude/agent-memory/*/MEMORY.md` e arquivos | formato observado (6 agentes) | 🟢 |
| `~/.claude/projects/<slug>/memory/` | formato da memória automática do Claude Code | 🟢 |
| https://github.com/garrytan/gbrain `src/core/engine.ts`, `docs/mcp/*` | BrainEngine, ferramentas MCP | 🟢 |
| `_reversa_sdd/code-analysis.md` §GBrain | requisitos levantados | 🟢 |
| `packages/session-query/*` | FTS do histórico (complemento) | 🟢 |
| — | carga, gravação, espelhamento e escopo | — (novo) |

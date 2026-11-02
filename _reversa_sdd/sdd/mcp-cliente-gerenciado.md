# Cliente MCP e integrações gerenciadas (`harness-mcp` + `harness-integrations`)

## Visão Geral
Dois componentes que trabalham juntos:
1. **Cliente MCP completo:** conecta o dsh a servidores MCP por stdio e por Streamable HTTP. Expõe as ferramentas como `mcp__<servidor>__<ferramenta>` (o mesmo padrão do Claude Code), além de recursos, prompts, progresso, cancelamento e reconexão. 🟢 portado do dsh, ampliado.
2. **Integrações gerenciadas:** o dsh **instala, atualiza, diagnostica e registra** sozinho os servidores e as ferramentas que são obrigatórios ou aprovados:
   - REA, GBrain, Playwright MCP, Chrome DevTools MCP e computer use;
   - csharp-ls (servidor de linguagem C#);
   - Gemini CLI, como agente ACP.

   Com isso o binário Rust continua único, e os runtimes Node, Bun e .NET viram dependências **detectadas**, não embutidas (ADR 102). 🟡

## Responsabilidades

### Cliente MCP
- Ler a configuração de servidores das fontes Claude Code (`.mcp.json`, `~/.claude.json`), de plugins e das integrações gerenciadas. 🟡
- Conectar, negociar a versão do protocolo, listar ferramentas, recursos e prompts e reconectar com backoff. 🟢
- Aplicar timeouts por servidor, mostrar o progresso e propagar o cancelamento. 🟢/🟡
- Rotular a saída como não confiável e respeitar as anotações (`readOnlyHint`, `destructiveHint`, `_meta.provenance`). 🟡
- Injetar as instruções do servidor no prompt, com limite de tamanho. 🟢

### Integrações gerenciadas
- Detectar runtimes e versões (Node, Bun, .NET SDK, Chrome) e apontar o que falta. 🟡
- Instalar e atualizar cada integração pelo canal oficial dela, verificando a versão mínima. 🟡
- Registrar cada uma como servidor MCP, LSP ou agente ACP já configurado. 🟡
- Manter um agendamento de atualização e um `doctor`. 🟡

## Interface

### Configuração de servidor (forma Claude Code) 🟢/🟡
```json
{ "mcpServers": {
    "gpflow": { "type": "http", "url": "https://gpflow.grupopereira.link/mcp",
                "headers": { "Authorization": "${secret:gpflow-auth}" } },
    "rea":    { "type": "stdio", "command": "node", "args": ["<prefix>/rea-agents/scripts/rea.mjs", "mcp"],
                "env": { "REA_MCP_OUTPUT_SCHEMA_PROFILE": "compact" },
                "timeout_ms": 330000 } } }
```
Os valores aceitam `${VAR}`, `${VAR:-padrão}` e `${secret:<nome>}`, este último resolvido no cofre do SO. Os nomes de servidor seguem `[A-Za-z0-9_-]{1,32}`. 🟢/🟡

| Transporte | Suporte | Confiança |
|---|---|---|
| `stdio` | command, args, env (ambiente higienizado), cwd | 🟢 |
| `http` (Streamable HTTP) | url, headers, OAuth do MCP (descoberta + PKCE) | 🟢 (OAuth 🟡) |
| `sse` legado | só leitura de config antiga, convertida para `http` quando possível | 🟡 |

### Ferramentas expostas ao modelo
| Ferramenta | Origem | Confiança |
|---|---|---|
| `mcp__<srv>__<tool>` | `tools/list` de cada servidor | 🟢 |
| `ListMcpResources`, `ReadMcpResource` | `resources/list`, `resources/read`, templates, paginação | 🟢 |
| prompts MCP | expostos como slash commands `/mcp__<srv>__<prompt>` | 🟡 |

### Catálogo de integrações gerenciadas
| Integração | Tipo | Pré-requisito detectado | Instalar / Atualizar | Registro | Confiança |
|---|---|---|---|---|---|
| **REA** | MCP stdio + plugin | Node ≥ 22.23.4 (24.x ≥ 24.22.1, ≥ 26.12.1) | `npm i -g rea-agents` / `rea update` | `rea` → `node …/scripts/rea.mjs mcp` | 🟢 |
| **GBrain** | MCP stdio | Bun ≥ 1.4.4 | `bun install -g github:garrytan/gbrain#latest-stable` (a mesma linha atualiza) | `gbrain serve --surface starter` | 🟢 |
| **Playwright MCP** | MCP stdio | Node ≥ 18 | `npm i -g @playwright/mcp` + `npx playwright install chromium` | `playwright` → `mcp-server-playwright` | 🟡 |
| **Chrome DevTools MCP** | MCP stdio | Node + Chrome | `npm i -g chrome-devtools-mcp` | `chrome-devtools` | 🟡 |
| **Computer use** | MCP stdio | conforme o driver (Cua Driver ou equivalente para Windows) | canal oficial do driver | `computer` | 🟡 (Windows: builds nightly 0.38.1; sem a extensão AGPL `cua-perception`) |
| **csharp-ls** | LSP | .NET SDK (`dotnet --list-sdks`) | `dotnet tool install -g csharp-ls` / `dotnet tool update -g csharp-ls` | LSP de `*.cs`/`*.csproj` | 🟢 |
| **Gemini CLI** | agente ACP | Node ≥ 20 | `npm i -g @google/gemini-cli` | `gemini --experimental-acp` | 🟡 |

### CLI
| Comando | Efeito |
|---|---|
| `dsh mcp list \| get <srv> \| add <nome> -- <cmd…> \| add --transport http <nome> <url> \| remove <nome>` | Gerencia servidores (mesma sintaxe do `claude mcp`) |
| `dsh mcp test <srv>` | Conecta, lista as ferramentas e mede a latência |
| `dsh integrations list` | Estado de cada integração: instalada, versão, runtime, registrada |
| `dsh integrations install <nome> \| update [<nome>\|--all] \| remove <nome>` | Ciclo de vida |
| `dsh integrations doctor` | Verifica runtimes, versões mínimas, caminhos e conectividade |
| `/mcp` na TUI | Lista os servidores com estado, ferramentas e erros; permite reconectar |

## Regras de Negócio

### Cliente
- **R1.** Cada servidor é independente. Uma falha de conexão no arranque **não** derruba o dsh: as ferramentas daquele servidor só não aparecem, e `/mcp` mostra o erro. 🟢
- **R2. Máquina de estados:** `connecting → initialized → lost → reconnecting → initialized | failed`. O backoff começa em 500 ms, dobra a cada tentativa, tem teto de 30 s e no máximo 10 tentativas. 🟢
- **R3.** O protocolo usado é o mais novo suportado pelo servidor, com fallback para revisões anteriores. 🟢
- **R4.** O timeout padrão de chamada é de 60 s. Ele pode ser configurado por servidor (`timeout_ms`), e as integrações gerenciadas trazem os próprios valores: REA 330 s, por causa do Ghidra. O timeout do provedor de análise e o do cliente são separados. 🟢/🟡
- **R5.** As notificações de progresso (quando o servidor as envia em resposta a um `progressToken`) aparecem na TUI e nos eventos ACP e SDK, com no máximo uma atualização a cada 100 ms. 🟡
- **R6.** O cancelamento do turno envia `notifications/cancelled` ao servidor. Uma resposta parcial (`partial_observation`) é preservada no resultado. 🟡
- **R7.** `structuredContent` e `outputSchema` são validados. Para o modelo vai a projeção em texto do servidor. O conteúdo estruturado fica no evento e no spill (spec 3, R17). 🟡
- **R8.** Toda saída MCP é rotulada como não confiável. Com `_meta.provenance.untrusted_content: true`, o rótulo é reforçado. 🟢/🟡
- **R9.** `destructiveHint: true` força `ask` na primeira chamada de cada ferramenta por sessão, a menos que uma regra `allow` explícita cubra a ferramenta. `readOnlyHint: true` permite paralelismo. São dicas, não autorização. 🟡
- **R10.** As instruções do servidor entram no prompt com limite de 32 KiB por servidor. 🟢
- **R11.** Um resultado acima de 10 MiB via stdio é truncado com aviso. O REA já respeita esse orçamento. 🟢
- **R12.** Segredos de headers e env vêm do cofre (`${secret:…}`) e nunca são gravados no log de sessão. Os eventos registram só o nome do servidor e da ferramenta e os argumentos. 🟡
- **R13.** Os servidores sujeitos ao gate de rede (spec 2, R13) ficam em `blocked_by_network` e não tentam conectar enquanto o gate estiver fechado. 🟡
- **R14.** Os servidores de `.mcp.json` do projeto pedem aprovação na primeira vez (spec 4, R17). 🟡

### Integrações gerenciadas
- **R15.** O dsh **não embute** Node, Bun ou .NET. Um runtime ausente ou abaixo do mínimo deixa a integração como `missing_runtime`, com a instrução de instalação (por exemplo, `winget install OpenJS.NodeJS.LTS`). O dsh não instala runtimes sem confirmação. 🟡
- **R16.** A instalação e a atualização usam **o canal oficial de cada projeto**, o que mantém as licenças e a procedência. O dsh registra a versão instalada em `~/.dsh/integrations/state.json`. 🟡
- **R16b. TLS corporativo:**
  - na rede da empresa há inspeção TLS, e o `npm`/`npx` falha com `SELF_SIGNED_CERT_IN_CHAIN`, porque a CA corporativa está no repositório do Windows mas não no bundle do Node (observado em 2026-11-01, Node v25.9.0);
  - o dsh executa os instaladores Node com `NODE_OPTIONS=--use-system-ca` e, no próprio HTTP (`reqwest`), usa o repositório de certificados do SO (`rustls-platform-verifier`);
  - **nunca** desliga a verificação TLS. 🟢
- **R17.** O registro feito pelo dsh fica no **registro próprio do dsh**. O `~/.claude.json` só é alterado se o usuário pedir (`--also-claude`). Se o servidor já estiver registrado no Claude Code (por exemplo, pelo `rea setup`), o dsh reutiliza a entrada em vez de duplicá-la. 🟡
- **R18.** A atualização automática é **opcional** e semanal (padrão desligado, `integrations.auto_update = "weekly"`). Ela nunca roda durante uma sessão com chamadas MCP em curso: espera o servidor ficar ocioso, atualiza e reconecta. 🟡
- **R19.** O GBrain é registrado com o módulo **Memorable desligado**, porque ele compartilha dados com um relay externo (decisão `gbrain-integration`). 🟢
- **R20.** O REA recebe `REA_MCP_OUTPUT_SCHEMA_PROFILE=compact` por padrão, para economizar contexto. 🟢/🟡
- **R21.** As integrações **obrigatórias** são REA, GBrain, Playwright MCP, Chrome DevTools MCP e computer use (decisões `rea-integration`, `gbrain-integration`, `experimental-mandatory`). `dsh integrations doctor` as marca em vermelho quando faltam. O dsh funciona sem elas, mas avisa no arranque. 🟢/🟡
- **R22.** A detecção do .NET (`dotnet --list-sdks`) dispara a oferta de instalar o `csharp-ls` (decisão `lsp-totvs`). 🟢

## Fluxo Principal

### Conexão
1. No arranque, o dsh reúne as configs: integrações gerenciadas, `~/.claude.json`, `.mcp.json`, plugins e CLI, nessa ordem de precedência inversa. 🟡
2. Filtra pelo gate de rede e pelas aprovações. 🟡
3. Conecta em paralelo, com até 8 conexões simultâneas. Cada servidor que responde `initialize` tem as ferramentas registradas. 🟢/🟡
4. O catálogo de ferramentas da próxima requisição inclui `mcp__*`. 🟢

### Instalação (exemplo: REA)
1. Ao rodar `dsh integrations install rea`, o dsh executa `node --version` e verifica se é ≥ 22.23.4. 🟡
2. Executa `npm i -g rea-agents` com a saída visível e localiza o prefixo com `npm root -g`. 🟡
3. Registra o servidor `rea` (R17) e o plugin do pacote (spec 6, R12). 🟡
4. Executa `dsh mcp test rea`, que conecta e lista as ferramentas, e grava o estado. 🟡

## Fluxos Alternativos
- **Servidor cai no meio de uma chamada:** a chamada falha com `TRANSPORT`, e a reconexão começa (R2). O modelo recebe um erro estruturado. 🟢
- **Timeout:** `TOOL_TIMEOUT` com o que tiver chegado de progresso. Envia `cancelled` ao servidor. 🟢/🟡
- **Node abaixo do mínimo:** `missing_runtime` com a instrução de atualização. O REA fica indisponível. 🟡
- **`npm` sem rede ou atrás de proxy:** a instalação falha com a saída do npm. A versão anterior continua registrada. 🟡
- **Gate de rede fechado:** `gpflow` fica `blocked_by_network`, sem tentativas de conexão. 🟡
- **Servidor com mais de 100 ferramentas:** o dsh registra todas, mas o catálogo pode ser filtrado por `allowed-tools` do agente. 🟡

## Dependências
- `rmcp` (SDK Rust oficial do MCP): cliente stdio e Streamable HTTP.
- `harness-tools`: pipeline e permissões. `harness-store`: spill e eventos.
- Cofre do SO (`keyring`). Política e gate (spec 2).
- Processos externos: `node`/`npm`, `bun`, `dotnet`, `gemini`, Chrome.

## Requisitos Não Funcionais
| Tipo | Requisito inferido | Evidência no código | Confiança |
|---|---|---|---|
| Disponibilidade | Reconexão de 500 ms a 30 s, até 10 tentativas; falha no arranque não fatal | `packages/mcp/mcp-client` (config `reconnect`, `failOnStartupError=false`) | 🟢 |
| Desempenho | Timeout de chamada de 60 s (REA 330 s) | `packages/mcp/mcp-client` `toolCallTimeoutMs`; docs do REA | 🟢 |
| Desempenho | Instruções de servidor até 32 KiB; resultado stdio até 10 MiB | `packages/mcp/mcp-client` `maxInstructionBytes`; REA | 🟢 |
| Segurança | Ambiente stdio higienizado; segredos no cofre | `packages/mcp/mcp-client`; ADR 007 | 🟢/🟡 |
| Manutenibilidade | Integrações atualizadas pelo canal oficial; versões mínimas verificadas | ADR 102 | 🟡 |

> Inferido a partir do código. Validar com a equipe de operações.

## Critérios de Aceitação
```gherkin
Dado Node 22.23.4 instalado e o REA ausente
Quando o usuário roda dsh integrations install rea
Então rea-agents é instalado via npm, o servidor rea fica initialized e dsh mcp test rea lista as ferramentas

Dado Node 20 instalado
Quando dsh integrations doctor roda
Então REA aparece como missing_runtime com a versão mínima exigida
E o dsh continua funcional sem o REA

Dado uma chamada mcp__rea__decompile_all em andamento com progresso
Quando o usuário cancela o turno
Então o servidor recebe notifications/cancelled e o resultado traz a observação parcial

Dado um servidor MCP que cai
Quando o dsh tenta reconectar
Então as tentativas seguem 0,5 s, 1 s, 2 s… até 30 s, no máximo 10, e o estado final é failed

Dado uma ferramenta com destructiveHint true e nenhuma regra allow
Quando o modelo a chama pela primeira vez na sessão
Então o usuário é consultado

Dado o .NET SDK detectado
Quando dsh integrations doctor roda
Então csharp-ls aparece como instalável com o comando dotnet tool install -g csharp-ls
```

## Prioridade
| Requisito | MoSCoW | Justificativa |
|---|---|---|
| Cliente stdio + http com reconexão | Must | Base de REA, GBrain, browser e `gpflow` |
| Leitura das configs do Claude Code | Must | Reaproveitamento do Totvs (`gpflow`) |
| Progresso, cancelamento e timeout por servidor | Must | REA com Ghidra leva minutos |
| Instalação e registro de REA, GBrain, Playwright, Chrome DevTools e computer use | Must | Decisões aprovadas como obrigatórias |
| csharp-ls via .NET detectado | Must | Decisão `lsp-totvs` |
| Anotações `destructive`/`readOnly` | Should | Segurança adicional |
| OAuth de MCP remoto | Should | Nenhum servidor atual exige |
| Prompts MCP como comandos | Could | Uso raro |
| Atualização automática semanal | Could | Há o `update` manual |
| Gemini CLI via ACP | Could | Via 3 do Gemini (spec 2) |

## Rastreabilidade de Código
| Arquivo | Função / Classe | Cobertura |
|---|---|---|
| `packages/mcp/mcp-client` | cliente, nomes `mcp__`, reconexão, timeouts | 🟢 |
| `packages/mcp/mcp-resources` | recursos e templates | 🟢 |
| `_reversa_sdd/code-analysis.md` §REA, §GBrain | requisitos das integrações | 🟢 |
| `_reversa_sdd/adrs/102-*.md` | decisão de integrações gerenciadas | 🟢 |
| — | `.mcp.json`/`~/.claude.json`, `${secret:}`, `harness-integrations` | — (novo) |

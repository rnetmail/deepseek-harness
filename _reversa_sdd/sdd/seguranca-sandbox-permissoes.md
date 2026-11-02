# Segurança: sandbox, permissões, aprovações e credenciais (`harness-sandbox` + motor de permissões)

## Visão Geral
O dsh é **monousuário** e não tem papéis (RBAC). A segurança vem da combinação de cinco camadas, todas **fail-closed**: 🟢 dsh / 🟡 alvo.
1. **Sandbox de processo:** o que comandos, scripts e PTC podem escrever (e, quando o SO permitir, ler e acessar pela rede).
2. **Permissões declarativas:** regras `allow`/`ask`/`deny` no formato Claude Code e modos de permissão.
3. **Aprovação humana:** pedidos `ask` e escalonamento de sandbox de uma única chamada.
4. **Credenciais no cofre do SO:** fora do alcance das ferramentas do agente.
5. **Confiança por canal e por workspace:** o que vem do usuário, de clientes ACP/SDK, de conteúdo externo e de pastas ainda não confiáveis.

As políticas de dados e o gate de rede ficam na spec 2. O guarda anti-exfiltração do `.env` fica na spec 3.

## Responsabilidades
- Aplicar o modo de sandbox a toda execução de processo (Bash, PowerShell, `RunCode`, hooks, scripts de skill). 🟢
- Falhar fechado quando o backend de sandbox não estiver disponível. 🟢
- Avaliar as regras de permissão e o modo de permissão antes de cada ferramenta. 🟡
- Conduzir as aprovações e o escalonamento de uma única chamada, com auditoria no log. 🟢
- Guardar e entregar credenciais sem expô-las ao modelo, ao log ou ao ambiente dos filhos. 🟡
- Conter árvores de processo e encerrá-las em cancelamentos e timeouts. 🟢
- Pedir confiança na primeira abertura de um workspace. 🟡

## Interface

### Modos de sandbox 🟢
| Modo | Escrita | Leitura | Rede de subprocessos |
|---|---|---|---|
| `read-only` | só sinks obrigatórios (`NUL`, `/dev/null`) | livre | negada onde o backend permitir |
| `workspace-write` | workspace (cwd da sessão) + área temporária privada | livre | conforme a política (padrão: permitida) 🟡 |
| `danger-full-access` | livre | livre | livre |

### Modos de permissão (Claude Code) mapeados para sandbox 🟡
| Modo de permissão | Sandbox | Ferramentas de edição | Bash/PowerShell |
|---|---|---|---|
| `default` | `workspace-write` | `ask` (exceto regras `allow`) | `ask` (exceto `allow`) |
| `acceptEdits` | `workspace-write` | `allow` dentro do workspace | `ask` (exceto `allow`) |
| `plan` | `read-only` **forçado** | `deny` (exceto o arquivo de plano) | só regras `allow` de leitura |
| `bypassPermissions` | `danger-full-access` | `allow` | `allow` (as regras `deny` continuam valendo) |

A troca é feita por `/permissions`, por `Shift+Tab` na TUI, por `--permission-mode` ou por `defaultMode` em `settings.json`. Toda troca é gravada no log e informada ao modelo no próximo passo. 🟢 (registro) / 🟡 (modos)

### Backends de sandbox
| SO | Backend | Escrita | Leitura | Rede | Confiança |
|---|---|---|---|---|---|
| Windows | token `WRITE_RESTRICTED` + ACLs (como no dsh) | ✔ (`partial`: Everyone, hard links) | ✖ | ✖ | 🟢 |
| Windows | **AppContainer** (alvo) | ✔ | ✔ por capacidades | ✔ (sem `internetClient`) | 🟡 |
| Linux | Landlock + seccomp (ou bubblewrap) | ✔ | ✔ | ✔ (seccomp/netns) | 🟢/🟡 |
| macOS | Seatbelt | ✔ | ✔ | ✔ | 🟡 |

Cada chamada informa o nível de aplicação, `full` ou `partial`, no resultado e no log. 🟢

### Aprovação 🟢
`request {agent, tool, call_id, reason, preview}` → `allow_once` | `allow_session` | `allow_always` (grava uma regra `allow` no `settings.local.json`) | `deny` | `cancel`. Sem respondente → `unavailable`, ou seja, rejeitado.

## Regras de Negócio

### Sandbox
- **R1.** O modo padrão de sessões novas é `workspace-write` com permissão `default`. 🟡 (o dsh TS usa `read-only` como padrão de fallback da política 🟢)
- **R2.** Fail-closed: se nenhum backend consegue aplicar o modo pedido, a execução falha com `SANDBOX_UNAVAILABLE`. **Nunca** roda sem confinamento por omissão. 🟢
- **R3. Escalonamento:**
  - uma negação devolve `[sandbox: file access denied under <modo> mode]` mais uma dica;
  - o modelo pode repetir **a mesma chamada uma vez** com `sandbox_permissions` (o modo mais estreito que resolve) e uma `justification`;
  - o usuário vê um único prompt, e a permissão vale **só para aquela chamada**. 🟢
- **R4.** O workspace é o cwd imutável da sessão. Symlinks e junctions que apontam para fora são resolvidos antes da checagem. 🟢/🟡
- **R5.** Hooks e scripts de skill rodam sob o mesmo sandbox do Bash da sessão. 🟡
- **R6.** Toda árvore de processos fica num Job Object kill-on-close (Windows) ou num cgroup/scope (Linux). Cancelamento, timeout ou fim da sessão matam a árvore inteira. 🟢

### Permissões
- **R7.** A avaliação é `deny` > `ask` > `allow` > modo de permissão. Um `deny` nunca é revertido por hook, por modo ou por `allow` de outra camada (os guardas são monotônicos). 🟢 (monotonicidade) / 🟡 (formato Claude)
- **R8.** Um hook `PreToolUse` pode endurecer a decisão (`deny` ou `ask`), nunca afrouxar um `deny`. 🟢/🟡
- **R9.** As regras de caminho valem para as ferramentas de arquivo **e** para os caminhos detectados em comandos de shell (spec 3, R15). 🟡
- **R10.** `allow_always` grava a regra no `settings.local.json` do projeto, nunca no `settings.json` versionado. 🟡
- **R11.** No `plan`, as ferramentas de escrita são de fato bloqueadas, ao contrário do dsh TS, onde o plan mode é só consultivo (dívida 5). `ExitPlanMode` apresenta o plano, e só a aprovação do usuário sai do modo. 🟢 (dívida) / 🟡 (regra)

### Aprovação e auditoria
- **R12.** Toda decisão (de regra, hook, usuário ou modo) é gravada no log (`approval/*`, `permission/decision`). Se a gravação falhar, a decisão vira **rejeição**. 🟢
- **R13.** Uma aprovação pedida sem turno aberto é inválida. Pedidos de subagentes sobem para a superfície da sessão raiz com o nome do agente. 🟢/🟡
- **R14.** A política `never` (CI, headless sem respondente) rejeita de forma determinística tudo o que pedir aprovação. 🟢
- **R15.** O lembrete de repetição avisa em chamadas idênticas consecutivas (3, 5 e 8), sem bloquear. 🟢

### Credenciais
- **R16.** Credenciais de provedores, de MCP e de plugins ficam **no cofre do SO** (Windows Credential Manager via `keyring`), com o nome `dsh/<escopo>/<id>`. O dsh não grava credencial em arquivo legível. 🟡
- **R17.** O processo do dsh resolve a credencial só no momento da requisição. Ela nunca entra no ambiente dos filhos, no log, no spill nem no prompt. 🟢 (env e modelo) / 🟡 (cofre)
- **R18.** Na migração (spec 1, R21b), `~/.dsh/.credentials.yaml` é lido uma vez, as chaves vão para o cofre e o arquivo só é removido com confirmação. 🟡
- **R19.** As chaves que estavam em texto puro no `dsh.cmd` do usuário devem ser **rotacionadas**. O `dsh doctor` alerta enquanto detectar um `dsh.cmd` com padrões de chave no PATH. 🟡 (dívida 2)

### Confiança
- **R20.** Na primeira abertura de uma pasta, o dsh pede confiança: "Confio nesta pasta". Sem ela, não executa hooks, servidores de `.mcp.json`, `!cmd` de comandos nem `env` das settings do projeto. 🟡
- **R21.** Conteúdo de web, MCP, páginas, lembretes, alvos do REA e mensagens de colegas de time é rotulado como **não confiável**. 🟢
- **R22.** Clientes ACP e SDK por stdio são controladores confiáveis (como no dsh). Uma exposição por socket ou HTTP exige um token local gerado por instalação. 🟢/🟡

## Fluxo Principal
1. O modelo pede `Bash {command: "npm test"}`. 🟢
2. Regras: nenhum `deny`; `ask` porque não há `allow`. Hooks `PreToolUse`: nada. 🟡
3. Aprovação: o usuário escolhe `allow_session`. Isso é gravado no log. 🟢
4. O processo roda sob `workspace-write` num Job Object. O resultado informa `sandbox: full|partial`. 🟢
5. O teste tenta escrever em `C:\Users\…\AppData`: o acesso é negado e o resultado traz a dica. 🟢
6. O modelo repete com `sandbox_permissions: danger-full-access` e uma justificativa. O usuário recusa. A recusa é registrada e o modelo segue por outro caminho. 🟢

## Fluxos Alternativos
- **Backend indisponível** (por exemplo, AppContainer bloqueado por política corporativa): cai para o token restrito + ACLs com `partial`. Se nenhum funcionar, `SANDBOX_UNAVAILABLE`. 🟢/🟡
- **Headless sem respondente:** `unavailable`, rejeitado. 🟢
- **Hook manda `allow` sobre um `deny`:** ignorado, e o `deny` vale. 🟡
- **Cofre do SO indisponível** (sessão de serviço sem perfil): o dsh recusa gravar credenciais e orienta a usar uma variável de ambiente só naquela execução. 🟡

## Dependências
- `windows-rs` (Job Objects, AppContainer, tokens, ACLs), `landlock`, `seccompiler`, `nix`; `keyring`.
- Ferramentas (spec 3), compatibilidade (spec 4: regras e settings), hooks (spec 13), superfícies (spec 15: prompts de aprovação).

## Requisitos Não Funcionais
| Tipo | Requisito inferido | Evidência no código | Confiança |
|---|---|---|---|
| Segurança | Fail-closed sem backend | `packages/sandbox/sandbox-policy` | 🟢 |
| Segurança | Escalonamento de uma única chamada | `packages/sandbox/sandbox-policy`, `interaction/user-approval` | 🟢 |
| Segurança | Auditoria obrigatória (falha de append = rejeição) | `packages/interaction/user-approval` | 🟢 |
| Segurança | Contenção de árvore de processos | `packages/subprocess/win32-process`, ADR 006 | 🟢 |
| Segurança | Credenciais fora do alcance do agente | ADR 007 (limitação admitida) → cofre do SO | 🟡 |
| Desempenho | Sonda de backend de sandbox com timeout de 5 s | `packages/sandbox/sandbox-local` (`probeTimeoutMs`) | 🟢 |

> Inferido a partir do código. Validar com a equipe de segurança da empresa (AppContainer e políticas corporativas).

## Critérios de Aceitação
```gherkin
Dado o modo plan
Quando o modelo chama Edit num arquivo do projeto
Então a chamada é negada e o resultado orienta a apresentar o plano com ExitPlanMode

Dado nenhum backend de sandbox utilizável
Quando o modelo chama Bash
Então o resultado é SANDBOX_UNAVAILABLE e nenhum processo é criado

Dado workspace-write e um comando que escreve fora do workspace
Quando o modelo repete a chamada com sandbox_permissions e o usuário permite uma vez
Então só aquela chamada roda ampliada e a próxima volta ao workspace-write

Dado uma regra deny Bash(rm:*) e um hook PreToolUse que responde allow
Quando o modelo chama Bash "rm -rf build"
Então a chamada é negada

Dado o usuário escolhendo "sempre permitir" para Bash(npm test)
Quando a decisão é gravada
Então a regra aparece em .claude/settings.local.json e não em .claude/settings.json

Dado uma credencial de provedor no cofre
Quando o modelo roda "set" ou "Get-ChildItem env:" via Bash
Então a credencial não aparece na saída
```

## Prioridade
| Requisito | MoSCoW | Justificativa |
|---|---|---|
| Sandbox fail-closed + contenção de processos | Must | Base de toda execução |
| Regras `allow`/`ask`/`deny` no formato Claude + monotonicidade | Must | Compatibilidade e governança Totvs |
| Aprovação com auditoria | Must | Rastreabilidade |
| Cofre do SO para credenciais | Must | Corrige a dívida 2 / ADR 007 |
| Plan mode restritivo | Must | Paridade com o Claude Code |
| Escalonamento de uma única chamada | Should | Evita ampliar a sessão inteira |
| AppContainer (leitura e rede no Windows) | Should | Hoje o Windows só confina escrita |
| Confiança de pasta | Should | Evita a execução de hooks e MCP de repositórios desconhecidos |
| Lembrete de repetição | Could | Ajuda, não protege |

## Rastreabilidade de Código
| Arquivo | Função / Classe | Cobertura |
|---|---|---|
| `packages/sandbox/sandbox`, `sandbox-policy`, `sandbox-local`, `sandbox-windows-acl` | modos, backends, escalonamento | 🟢 |
| `native/system` | launcher Landlock | 🟢 |
| `packages/guard/timeout-policy`, `repeat-tool-reminder` | guardas | 🟢 |
| `packages/interaction/user-approval`, `permission-presets` | aprovação e presets | 🟢 |
| `packages/credentials/*`, `packages/settings/*` | credenciais e settings | 🟢 |
| `packages/subprocess/win32-process` | Job Objects | 🟢 |
| `_reversa_sdd/permissions.md`, `adrs/006`, `adrs/007` | matriz e decisões | 🟢 |
| — | modos Claude, cofre, AppContainer, confiança de pasta | — (novo) |

## Atualização upstream 0.2.1-rc.5 (ciclo 1, 2026-11-01)
> Origem: [ledger](../upstream/ledger.md). Estas regras **prevalecem** sobre as anteriores quando houver conflito.

- **U1. Diretório de trabalho da sessão:** o diretório efetivo é registrado no log (`working-directory/change`) e sobrevive a replay. Ele é validado pelo provedor de filesystem e começa no projeto original. Os processos já em execução mantêm o diretório antigo. **R4 passa a ser:** a raiz do sandbox `workspace-write` é o diretório efetivo atual; mudar para fora do projeto original exige aprovação. 🟢 (diretório) / 🟡 (regra de aprovação)
- **U2. Precedência de credenciais:** uma credencial **armazenada** vence a variável de ambiente do processo. No modo headless sem cofre, a ordem é ambiente do processo, depois `.env` do diretório de invocação, depois `.env` de `~/.dsh`. Complementa R16–R17. 🟢 (o `.env` do projeto **nunca** é lido quando houver `deny` sobre ele: R12 da spec 4 prevalece 🟡)
- **U3. Windows: confinamento de exclusão:** cada concessão de ACL aplica, numa única chamada `SetNamedSecurityInfoW`, a ACE de escrita mais o rótulo de **integridade obrigatória**, e o token restrito é rebaixado para combinar. Isso impede exclusões fora das áreas concedidas. 🟢 (nota `windows-acl-mandatory-integrity-confinement`)

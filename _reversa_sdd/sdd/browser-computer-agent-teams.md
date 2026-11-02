# Browser use, computer use e agent teams (`harness-integrations` + `harness-core::teams`)

## Visão Geral
Três capacidades que no dsh TS são **experimentais e opt-in**, e no dsh Rust passam a ser **obrigatórias** (decisão `experimental-mandatory`):
1. **Browser use:** o modelo inspeciona e opera páginas web por **Playwright MCP** ou **Chrome DevTools MCP**, seja abrindo um navegador próprio, seja se conectando a um Chrome já aberto. 🟢
2. **Computer use:** o modelo observa e opera a área de trabalho por um driver MCP (no dsh, o Cua Driver). 🟢
3. **Agent teams:** a sessão vira um time pequeno. O agente principal (Lead) cria colegas com nome, troca mensagens duráveis com eles e coordena um quadro de tarefas compartilhado. 🟢

O dsh Rust reproduz os contratos do dsh (ADR 011) e instala os componentes pela spec 7. 🟢 contratos / 🟡 obrigatoriedade e Windows.

## Responsabilidades
- Registrar **um** provedor de browser e **um** de computer use por sessão, cada um exclusivo. 🟢
- Manter a conexão do provedor viva entre os turnos da sessão e liberá-la ao final. 🟢
- Conectar o navegador a um Chrome existente (via CDP) quando pedido, com proteções extras. 🟢/🟡
- Oferecer ao modelo as 9 ferramentas de time, com mailbox durável, quadro de tarefas com controle de revisão e limites. 🟢
- Reconstruir o estado do time a partir do log da sessão do Lead depois de uma queda. 🟢

## Interface

### Browser use 🟢
| Provedor | Pacote upstream | Modos | Ferramentas expostas |
|---|---|---|---|
| `playwright` (padrão) | `@playwright/mcp` | navegador próprio (perfil temporário) ou `attach` a um Chrome via CDP | as do servidor (`browser_navigate`, `browser_click`, `browser_snapshot`, `browser_take_screenshot`…) como `mcp__playwright__*` |
| `chrome-devtools` | `chrome-devtools-mcp` | idem, mais performance, rede e console | `mcp__chrome-devtools__*` |

O Stagehand nativo do dsh TS **não** é portado, porque exige Node e LLM próprios. 🟡

Configuração (`~/.dsh/integrations.toml`):
```toml
[browser]
provider = "playwright"            # | "chrome-devtools"
mode     = "isolated"              # | "attach"
attach   = { cdp_url = "http://127.0.0.1:9222", allow = ["ask"] }
headless = false
```

### Computer use 🟢
| Provedor | Estado | Observações |
|---|---|---|
| `cua-driver` (MCP) | dsh TS: `computer-use-cua-driver-mcp`, versão de referência `cua-driver-rs v0.28.0` | O driver cuida da instalação e das permissões de desktop. As ferramentas são as do próprio driver (`check_permissions`, captura de tela, mouse, teclado…) |
| Suporte a Windows | 🟡 há builds `cua-driver-rs` para **windows-x86_64** e **windows-arm64** (nightly 0.38.1, 01/11/2026) e um runner de testes E2E Windows no repositório. Ainda não há release estável Windows identificada [Revisão: era 🔴] | Validar no piloto. Driver MIT. **Não** instalar a extensão `cua-perception`, que é AGPL-3.0 |

### Agent teams: 9 ferramentas 🟢
| Ferramenta | Quem pode | Efeito |
|---|---|---|
| `spawn_teammate` | Lead | Cria um colega com `name`, `description` e `task`. A tarefa inicial ganha o prefixo `<system-reminder>You are teammate "<name>".</system-reminder>` |
| `send_message` | todos | Entrega no próximo passo do destinatário (`accepted`) ou enfileira (`queued`). Inicia um colega ocioso ou retoma um inativo |
| `list_agents` | todos | Lista os membros com status ao vivo |
| `wait_agent` | todos | Espera a próxima mudança no time (de 10 s a 1 h). Devolve `noProgress` se ninguém estiver rodando |
| `interrupt_agent` | Lead | Cancela o turno atual de um colega, mantendo a inbox |
| `team_task_create`, `team_task_list`, `team_task_get`, `team_task_update` | todos | Quadro de tarefas com dependências (DAG), dono e revisão (CAS) |

Alias de compatibilidade: `SendMessage` → `send_message` e `ListAgents` → `list_agents`, para skills e hooks escritos para os times do Claude Code. 🟡

Limites (configuráveis e validados no arranque): `max_members` 16, `max_tasks` 256, `max_pending_messages_per_member` 64, `max_message_bytes` 65.536, `disposal_timeout_ms` 5.000. 🟢

## Regras de Negócio

### Browser e computer use
- **R1.** Um único provedor de browser e um único de computer use por processo. O registro é exclusivo, e um segundo provedor é erro de configuração. 🟢
- **R2.** A conexão MCP do provedor é inicializada **antes** de a criação ou o resume da sessão concluir e é mantida entre os turnos. Um navegador externo fica reservado àquela sessão até ela fechar. 🟢
- **R3.** O modo padrão é `isolated`: um perfil temporário, apagado ao fim da sessão. 🟡
- **R4. `attach`** (conectar a um Chrome com login e abas abertas) exige confirmação na primeira vez em cada sessão. A confirmação mostra a URL CDP e avisa que o modelo verá e poderá agir nas sessões logadas daquele navegador. 🟡
- **R5. Totvs AGP:** o perfil dedicado `.claude/navegador-perfis/gpflow` (Chrome logado no GPFlow) é usado pelas skills Python do time via CDP. O dsh **não** abre esse perfil por conta própria. O `attach` a ele segue R4 e o gate de rede (spec 2, R13). As skills Python continuam funcionando, porque são processos chamados por `Bash`. 🟢 (inventário) / 🟡
- **R6.** Todo conteúdo de página (snapshot, texto, console, rede) é **não confiável**. Instruções vindas da página nunca são seguidas. 🟢
- **R7.** As ações de computer use são, por padrão, `ask` por ação, com possibilidade de "permitir nesta sessão". Captura de tela e leitura de estado podem ser `allow`. 🟡
- **R8.** O computer use não age enquanto a tela estiver bloqueada nem sem a permissão de acessibilidade concedida. `check_permissions` é chamado sem prompt no arranque, e a falta de permissão é exibida. 🟢
- **R9.** O provedor de computer use fica reservado até sua conexão e ferramentas fecharem. Sessões concorrentes não compartilham o desktop: a segunda recebe "computer use em uso por <sessão>". 🟢
- **R10.** Screenshots entram como imagens e seguem os limites de anexo e o offload de imagens (spec 14). 🟢

### Agent teams
- **R11.** O time **só é criado quando o usuário pede explicitamente** ("crie um time…", "crie um colega…"). Tarefas comuns nunca disparam delegação para o time. 🟢
- **R12.** O log da sessão do Lead é a única fonte da verdade. O roster, a mailbox e as tarefas são reconstruídos a partir dele. Os registros do time ficam só no log e **não** entram no histórico do modelo. 🟢
- **R13.** Um envio é bem-sucedido quando a mensagem está gravada de forma durável. Uma mensagem `queued` **não** deve ser reenviada (regra de não repetir). 🟢
- **R14.** Cada mensagem entregue é uma mensagem user-role, com um primeiro bloco curto contendo o id estável e o remetente, seguido do conteúdo original. 🟢
- **R15.** Uma edição de tarefa baseada numa revisão desatualizada é rejeitada (`conflict`), sem sobrescrever. Reivindicar uma tarefa cujas dependências não terminaram é rejeitado. 🟢
- **R16.** Esgotar um limite gera um erro tipado. Ids e nomes nunca são reutilizados. 🟢
- **R17.** Todos os membros compartilham o mesmo cwd e o mesmo checkout. A política do time orienta sobre versão obsoleta de arquivo (`FS_STALE_VERSION`), riscos de formatador e codegen e coordenação do escopo de escrita. O Lead deve esperar o time antes de responder ao usuário. 🟢
- **R18.** As ferramentas do time substituem as ferramentas legadas de subagente com os mesmos nomes **só para membros do time**. A ferramenta `Task`/`Agent` (spec 13) continua disponível para a sessão principal fora do time. 🟢/🟡
- **R19.** Cada colega é um agente com a própria sessão filha. O modelo dele é resolvido pelo mapa de aliases e pela política (spec 2), e a máscara de ferramentas é herdada do Lead. 🟡
- **R20.** O encerramento fecha a admissão, aborta as criações em curso, libera os colegas vivos e seus descendentes e falha visivelmente se passar de `disposal_timeout_ms`. 🟢
- **R21.** Times aninhados, liberação automática de tarefas e worktrees por colega **não** fazem parte do escopo inicial (trabalho adiado no dsh). 🟢

## Fluxo Principal

### Browser
1. O usuário pede "abra o painel X e confira o valor Y". 🟡
2. Com o provedor `playwright` em `isolated`, o dsh garante a conexão (R2) e o modelo chama `mcp__playwright__browser_navigate` e depois `browser_snapshot`. 🟢
3. O conteúdo volta rotulado como não confiável (R6), e o modelo responde. 🟢

### Time
1. O usuário pede "crie um time: um revisor e um testador para esta mudança". 🟢
2. O Lead chama `spawn_teammate` duas vezes, cria tarefas no quadro e envia o resumo da mudança. 🟢
3. Os colegas trabalham, atualizam tarefas (com CAS) e respondem por `send_message`. 🟢
4. O Lead faz `wait_agent` até as tarefas fecharem e então responde ao usuário (R17). 🟢

## Fluxos Alternativos
- **Chrome não instalado:** o `doctor` aponta a falta. O Playwright pode baixar o Chromium (`npx playwright install chromium`) com confirmação. 🟡
- **`attach` recusado pelo usuário:** cai para `isolated`. 🟡
- **Driver de computer use ausente ou sem permissão:** as ferramentas não aparecem e o modelo é avisado. O resto da sessão continua. 🟢
- **Queda do processo com o time ativo:** no resume, o roster e a mailbox são reconstruídos do log. Os colegas offline recebem as mensagens enfileiradas quando forem retomados. 🟢
- **Mensagem para um nome inexistente:** erro específico, sem efeito colateral. 🟢

## Dependências
- Cliente MCP e integrações gerenciadas (spec 7). Loop e sessões filhas (spec 1). Subagentes (spec 13). Política e gate (spec 2). Anexos e imagens (spec 14).
- Externos: Node (Playwright e Chrome DevTools MCP), Chrome/Chromium, Cua Driver ou equivalente.

## Requisitos Não Funcionais
| Tipo | Requisito inferido | Evidência no código | Confiança |
|---|---|---|---|
| Escalabilidade | Até 16 membros, 256 tarefas, 64 mensagens pendentes por membro, 64 KiB por mensagem | `packages/experimental/agent-team/README.md` (limites) | 🟢 |
| Durabilidade | Mailbox e tarefas sobrevivem a quedas (log do Lead) | `packages/experimental/agent-team/src/mailbox.ts`, `task-board.ts` | 🟢 |
| Disponibilidade | Encerramento limitado a 5 s | `agent-team` (`disposalTimeoutMs`) | 🟢 |
| Segurança | `attach` e ações de desktop com confirmação | ADR 011; inventário Totvs R11 | 🟡 |
| Desempenho | `wait_agent` de 10 s a 1 h | `agent-team` (`waitForChange`) | 🟢 |

> Inferido a partir do código. Validar no piloto, sobretudo o computer use no Windows.

## Critérios de Aceitação
```gherkin
Dado o Playwright MCP instalado e o modo isolated
Quando o modelo navega para uma página e tira um snapshot
Então o conteúdo vem rotulado como não confiável e o perfil temporário é apagado ao fim da sessão

Dado mode = attach com um Chrome aberto em 9222
Quando a sessão tenta se conectar
Então o usuário confirma antes, vendo a URL CDP e o aviso sobre as sessões logadas

Dado o pedido "crie um colega chamado revisor para checar o diff"
Quando o Lead chama spawn_teammate e send_message
Então list_agents mostra revisor e a mensagem retorna accepted ou queued
E derrubar o processo e retomar a sessão preserva o roster e as mensagens pendentes

Dado uma tarefa na revisão 3
Quando um colega tenta atualizá-la com base na revisão 2
Então a atualização é rejeitada como conflito

Dado uma tarefa comum sem pedido de time
Quando o modelo trabalha
Então nenhum colega é criado

Dado o Cua Driver ausente
Quando a sessão inicia
Então as ferramentas de computer use não aparecem e o doctor indica o que falta
```

## Prioridade
| Requisito | MoSCoW | Justificativa |
|---|---|---|
| Browser use via Playwright MCP | Must | Obrigatório; usado pelo time Totvs |
| Browser use via Chrome DevTools MCP | Must | Obrigatório |
| Agent teams (9 ferramentas, durável) | Must | Obrigatório |
| Computer use | Must | Obrigatório; o driver para Windows existe em nightly (validar no piloto) |
| `attach` a Chrome existente | Should | Necessário para fluxos logados como o GPFlow |
| Aliases `SendMessage`/`ListAgents` | Could | Compatibilidade com conteúdo escrito para o Claude |
| Stagehand nativo | Won't | Exige Node e LLM próprios; coberto pelos MCP |

## Rastreabilidade de Código
| Arquivo | Função / Classe | Cobertura |
|---|---|---|
| `packages/browser-use/browser-use` | registro exclusivo de provedor | 🟢 |
| `packages/experimental/browser-use-playwright-mcp`, `browser-use-chrome-devtools-mcp`, `browser-use-runtime` | provedores de browser | 🟢 |
| `packages/computer-use/computer-use` | registro exclusivo de provedor | 🟢 |
| `packages/experimental/computer-use-cua-driver-mcp`, `computer-use-cua-driver-native` | provedores de desktop | 🟢 |
| `packages/experimental/agent-team/src/*` (`mailbox.ts`, `task-board.ts`) | time, mailbox, quadro | 🟢 |
| `packages/experimental/tool-agent-team` | as 9 ferramentas | 🟢 |
| `_reversa_sdd/adrs/011-*.md` | decisão de provedores | 🟢 |

## Atualização upstream 0.2.1-rc.5 (ciclo 1, 2026-11-01)
> Origem: [ledger](../upstream/ledger.md). Estas regras **prevalecem** sobre as anteriores quando houver conflito.

- **U1. Times:** as mensagens entre membros entram **direto na inbox do destinatário**, e a identidade do colega fica no contexto de runtime obrigatório (não só no prefixo da tarefa). 🟢
- **U2. Worktree explícito** (`experimental/worktree`): cria um checkout isolado a partir de um commit local, numa **branch nova com nome**, e continua a mesma sessão nele, sem tocar no checkout de origem. O sandbox governa as escritas. **Muda R21:** worktrees por colega viram uma opção suportada, via `git worktree` com `gix`/`git`. 🟢/🟡
- **U3. CUA nativo do upstream** (`experimental/cua`, `cua-macos`): objetos independentes do avaliador, **consentimento por aplicação**, helper nativo compartilhado com estado por agente, hover contínuo. No dsh Rust, o **consentimento por aplicação** é adotado: a primeira ação numa janela de cada aplicação pede aprovação. O driver continua sendo o `cua-driver-rs` (Windows). 🟡

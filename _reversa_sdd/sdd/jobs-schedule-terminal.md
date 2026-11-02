# Jobs em segundo plano, agendamentos e terminais persistentes (`harness-core::jobs` + `schedule` + `terminal`)

## Visão Geral
Três capacidades para trabalho que não cabe numa única chamada síncrona:
- **Jobs**: comandos, subagentes e envios de terminal rodando em segundo plano. A conclusão é avisada na sessão sem polling, como `run_in_background` e as notificações de tarefa do Claude Code. 🟢
- **Agendamentos**: lembretes duráveis (`after`, `at`, `every`) que disparam um turno quando o agente está ocioso. No alvo ganham **expressões cron**, para alcançar o `CronCreate` do Claude Code. 🟢/🟡
- **Terminais persistentes**: sessões de shell interativo (PTY/ConPTY) para programas que leem stdin ou mantêm estado. 🟢

No dsh TS, `schedule` vem por overlay opcional e os terminais ficam fora do `base`. No alvo, os três vêm **ligados por padrão**. 🟢/🟡

## Responsabilidades
- Iniciar, listar, ler e encerrar jobs, com posse por sessão de agente e limites. 🟢
- Entregar a conclusão do job por *wakeup* do agente ocioso ou por injeção no próximo passo. 🟢
- Criar, listar e apagar agendamentos com estado durável no log da sessão. 🟢
- Disparar os lembretes só com o agente ocioso, sem backlog de intervalos perdidos. 🟢
- Abrir e operar terminais persistentes restritos ao agente criador, sob o sandbox. 🟢

## Interface

### Ferramentas de jobs 🟢
| Ferramenta (alias Claude) | Argumentos | Retorno |
|---|---|---|
| `job_output` (`TaskOutput`, `BashOutput`) | `job_id`, `wait?`, `timeout_ms?`, `filter?` (regex 🟡) | Saída nova desde a última leitura (stream) ou resultado final, terminando com `[status: running\|completed\|failed\|killed]` |
| `job_list` | — | Jobs do agente com id, tipo, estado e início |
| `job_kill` (`TaskStop`, `KillShell`) | `job_id` | Conclui só depois que o trabalho parou |

Ids estáveis no formato `<kind>-N`: `bash-1`, `agent-2`, `terminal-3`.

### Ferramentas de agendamento 🟢/🟡
| Ferramenta (alias Claude) | Argumentos | Retorno |
|---|---|---|
| `schedule_create` (`CronCreate`) | `prompt`, e um entre: `after` (segundos) · `at` (instante com offset, ou local + `time_zone` IANA) · `every` (≥ 5 min) · **`cron`** (expressão de 5 campos + `time_zone`) 🟡 | `id`, próxima ocorrência; ou `persistence_uncertain` |
| `schedule_list` (`CronList`) | — | Agendamentos ativos com a próxima ocorrência |
| `schedule_delete` (`CronDelete`) | `id` | ok ou `schedule_not_found` |

### Ferramentas de terminal 🟢
| Ferramenta | Efeito |
|---|---|
| `terminal_open` | Abre um shell (`bash` do Git for Windows ou `pwsh`) e devolve id, nome, pid, status e a saída inicial |
| `terminal_send` | Escreve um texto (Enter opcional) e espera a prontidão, ou vira job com `run_in_background` |
| `terminal_read` | Lê uma página da saída retida |
| `terminal_signal` | Envia um sinal permitido (Ctrl+C) ao grupo em primeiro plano |
| `terminal_close` | Encerra e espera o fim da árvore de processos |
| `terminal_list` | Terminais do agente |

## Regras de Negócio

### Jobs
- **R1.** Um job pertence à **sessão de agente** que o criou. Outros agentes, inclusive pai e filhos, não o veem nem o matam. 🟢
- **R2.** No máximo **10** jobs rodando ou parando por dono. Acima disso, a criação falha com um erro tipado. 🟢
- **R3.** `job_output` de um job de stream devolve **só a saída nova** desde a última leitura. Um timeout de espera devolve `[status: running]` e mantém o job vivo. 🟢
- **R4. Conclusão:**
  - com o dono ocioso, o aviso abre um turno (*wakeup*), no máximo **3** seguidos;
  - uma mensagem do usuário zera esse orçamento;
  - com o dono ocupado, no modo `quiet` ou com o orçamento esgotado, o aviso entra no próximo passo. 🟢
- **R5.** Os jobs **morrem com o processo** do dsh. No resume, um job que estava ativo aparece como `failed` (interrompido) no histórico, e não é retomado. 🟢/🟡
- **R6.** A saída do job segue o spill (50 KB inline), com o arquivo completo legível por `Read`. 🟢
- **R7.** `job_kill` mata a árvore de processos do job (Job Object). 🟢

### Agendamentos
- **R8.** O estado vive **no log da sessão** (`schedule/change`). Os timers são projeções reconstruídas na abertura. Antes de confirmar, há uma barreira de flush; sem confirmação, o resultado é `persistence_uncertain`. 🟢
- **R9.** Um lembrete só dispara com o agente **ocioso** (fase de *maintenance*). Nunca interrompe um turno. 🟢
- **R10.** Cada disparo vira **uma mensagem user-role** com framing fixo que trata o texto do lembrete como conteúdo **não confiável**. Vários recorrentes vencidos chegam juntos numa mensagem só. 🟢
- **R11.** Ocorrências perdidas não geram backlog; vale só a mais recente. Com a sessão fechada, o lembrete fica vencido até a sessão ser retomada. 🟢
- **R12.** `every` exige ≥ 5 min e se alinha à âncora de criação. 🟢
- **R13. `at` com hora local:**
  - um horário que cai no salto do horário de verão é rejeitado;
  - um horário que existe duas vezes (sobreposição) usa o primeiro instante. 🟢
- **R14.** **Cron** (novo): 5 campos (`min hora dia mês dia-da-semana`), fuso IANA obrigatório (padrão: o do sistema, por exemplo `America/Sao_Paulo`) e intervalo mínimo efetivo de 5 min. "Dias úteis às 9h" = `0 9 * * 1-5`. 🟡
- **R15.** Um fork **não** herda os agendamentos do pai. Os subagentes não criam agendamentos (só a sessão raiz). 🟢
- **R16.** Não há canais externos (e-mail, push) por padrão. Uma notificação do SO (toast do Windows) quando a TUI está minimizada é opcional. 🟢/🟡
- **R17.** O dsh **não** roda em segundo plano depois de fechado. Agendamentos só disparam com o dsh aberto ou ao retomar a sessão. Um agendamento que precise rodar com o dsh fechado exige uma tarefa do Agendador do Windows criada sob confirmação (`dsh schedule install-task`), que chama `dsh -p --resume <id>`. 🟡

### Terminais
- **R18.** Um terminal é restrito ao **agente criador** e vive só no processo atual. 🟢
- **R19.** O shell roda sob o sandbox da sessão. O dialeto é `bash` (Git for Windows) ou `pwsh` no Windows, e o uso é orientado a linhas (TUIs de tela cheia não são suportadas). 🟢
- **R20.** Limites: `max_read_bytes` 256 KiB, `max_result_bytes` 256 KiB, timeout de prontidão de 30 s e encerramento gracioso de 3 s. 🟢
- **R21.** O prompt orienta o modelo a preferir `Bash`/`PowerShell` one-shot para trabalho limitado e usar terminal só para programas interativos ou de estado longo (REPL, `sqlplus`, servidor de desenvolvimento). 🟢
- **R22.** A detecção de prontidão usa o retorno do prompt do shell (marcador injetado no `PS1`/`prompt`) e o silêncio da saída. 🟡

## Fluxo Principal

### Job
1. O modelo chama `Bash {command: "npm run build", run_in_background: true}` e recebe `bash-1` na hora. 🟢
2. Segue trabalhando e, quando precisar, consulta `job_output bash-1`. 🟢
3. Ao terminar o build, com o agente ocioso, o aviso abre um turno ("job bash-1 concluído, exit 0"). 🟢

### Agendamento
1. O usuário pede "me lembre todo dia útil às 9h de verificar os jobs de batida". 🟡
2. O modelo chama `schedule_create {cron: "0 9 * * 1-5", time_zone: "America/Sao_Paulo", prompt: "Verificar jobs de importação de batidas"}`. 🟡
3. Às 9h, com o dsh aberto e o agente ocioso, o lembrete entra como mensagem não confiável e o agente executa a verificação. 🟡

## Fluxos Alternativos
- **11º job:** erro de limite e nenhum processo criado. 🟢
- **Dsh fechado no horário:** o lembrete fica vencido e dispara na retomada (só a ocorrência mais recente). 🟢
- **Fuso inválido ou cron inválido:** erro na criação, sem gravar nada. 🟢/🟡
- **Terminal travado:** `terminal_signal` com Ctrl+C. Se não resolver, `terminal_close` mata a árvore. 🟢

## Dependências
- Loop e inbox (spec 1), ferramentas e spill (spec 3), sandbox e contenção (spec 12), subagentes (spec 13), TUI (spec 15).
- Crates: `tokio` (tasks, `sleep_until`), `tokio-util` (`CancellationToken`), `croner` ou `cron` + `chrono-tz`, `portable-pty` (ConPTY), `windows-rs` (Agendador de Tarefas, toast).

## Requisitos Não Funcionais
| Tipo | Requisito inferido | Evidência no código | Confiança |
|---|---|---|---|
| Escalabilidade | 10 jobs por dono; 3 wakeups seguidos | `packages/jobs/jobs-local`, `jobs` | 🟢 |
| Durabilidade | Agendamentos no log com flush antes de confirmar | `packages/schedule/schedule` | 🟢 |
| Desempenho | Terminal: 256 KiB por leitura, 30 s de prontidão, 3 s de encerramento | `packages/terminal/terminal-bash`, `tool-terminal` | 🟢 |
| Segurança | Lembretes e saídas como conteúdo não confiável; posse por agente | `schedule`, `terminal` | 🟢 |
| Disponibilidade | `every` ≥ 5 min; sem backlog | `packages/schedule/schedule` | 🟢 |

> Inferido a partir do código. Validar o uso de cron e do Agendador do Windows com o usuário.

## Critérios de Aceitação
```gherkin
Dado um Bash com run_in_background
Quando o comando termina com o agente ocioso
Então um turno é aberto com o aviso de conclusão, sem o modelo ter feito polling

Dado 3 wakeups seguidos sem mensagem do usuário
Quando um 4º job termina
Então o aviso é injetado no próximo passo, sem abrir turno

Dado schedule_create com cron "0 9 * * 1-5" e fuso America/Sao_Paulo
Quando chega segunda-feira às 9h com o agente ocioso
Então o lembrete entra como mensagem user-role rotulada como não confiável

Dado um lembrete every 10 min e o dsh fechado por 1 h
Quando a sessão é retomada
Então dispara uma única ocorrência, não seis

Dado um subagente
Quando ele tenta chamar schedule_create
Então a ferramenta não está disponível para ele

Dado um terminal aberto pelo agente A
Quando o agente B tenta ler esse terminal
Então recebe erro de terminal desconhecido
```

## Prioridade
| Requisito | MoSCoW | Justificativa |
|---|---|---|
| Jobs + `run_in_background` + aviso de conclusão | Must | Paridade com o Claude Code; builds e testes longos |
| Agendamentos `after`/`at`/`every` | Should | Lembretes e verificações recorrentes |
| Cron | Should | Paridade com `CronCreate` |
| Terminais persistentes | Should | `sqlplus` e REPLs no Totvs |
| Aliases Claude (`TaskOutput`, `CronCreate`…) | Should | Skills escritas para o Claude |
| Tarefa do Agendador do Windows | Could | Execução com o dsh fechado |
| Toast do Windows | Could | Conveniência |

## Rastreabilidade de Código
| Arquivo | Função / Classe | Cobertura |
|---|---|---|
| `packages/jobs/jobs`, `jobs-local`, `tool-jobs` | contrato, registro, ferramentas | 🟢 |
| `packages/schedule/schedule` | agendamentos duráveis | 🟢 |
| `packages/terminal/terminal`, `terminal-bash`, `tool-terminal` | terminais | 🟢 |
| `_reversa_sdd/state-machines.md` §6, §7 | máquinas de estado | 🟢 |
| — | cron, aliases Claude, Agendador do Windows | — (novo) |

## Atualização upstream 0.2.1-rc.5 (ciclo 1, 2026-11-01)
> Origem: [ledger](../upstream/ledger.md). Estas regras **prevalecem** sobre as anteriores quando houver conflito.

- **U1. Jobs:** um **ring de saída por job**, com duas leituras: o cursor do modelo (`job_output`, sem os chunks `log`) e a leitura absoluta para observadores. `progress` é a linha viva, limpa no fim; `detail` é o motivo terminal. 🟢
- **U2. Fatos persistidos:** `job/registered` e `job/settled` (dono, tipo, rótulo, chamada criadora, `producerReported`) são gravados na sessão, com `JobId` UUID estável entre reinícios. **R5 fica assim:** o job continua morrendo com o processo, mas o histórico mostra o registro e o fechamento. 🟢
- **U3. Armazenamento da agenda:** a agenda passa a ser um **domínio de armazenamento do Host** (`schedule.json`), com sessão de origem, status `active`/`inactive`, último recibo e histórico de entregas. Ela **não** é mais estado no log da sessão. **R8 muda:** a fonte da verdade é o domínio, e um arquivo ilegível falha por inteiro, nunca vira exclusão. 🟢
- **U4.** A exclusão é **definitiva**, sem lixeira. O histórico de entregas é limitado a 30 dias e 200 registros (configurável). Há uma ferramenta nova, `schedule_update`. 🟢
- **U5. Framing:** a segunda linha fixa do bloco é `This is a scheduled message from the user`. O conteúdo continua rotulado como agendado e não como digitado agora (R10). 🟢
- **U6.** As ferramentas de agenda são registradas **por preset**: só os presets que as montam as expõem. Isso combina com R15 (subagentes sem agenda). 🟢
- **U7. Terminal:** `prompt_tail_grace_ms` (padrão 0) tolera a cauda do prompt do pwsh. No Windows, o PTY usa o **OpenConsole embarcado** em vez do conhost do sistema. No Rust: `portable-pty` com ConPTY e um OpenConsole distribuído junto, a avaliar. 🟢/🟡

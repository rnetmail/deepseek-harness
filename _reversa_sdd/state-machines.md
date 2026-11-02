# Máquinas de estado

## 1. Agente / turno 🟢
```mermaid
stateDiagram-v2
  [*] --> idle
  idle --> running: mensagem em nextTurn (usuário, goal, lembrete, wakeup)
  running --> running: step (request → tools) enquanto houver tool calls
  running --> cancelling: cancel() {user|parent}
  cancelling --> idle: turn/end {aborted} + flush
  running --> idle: turn/end {completed | max-tokens | failed} + flush
  idle --> maintenance: runMaintenance() (lembrete vencido)
  maintenance --> idle
  idle --> [*]: dispose
```
Regras: a cancelação antes do claim limpa as filas. Um observer de notificação não pode vetar a parada. Depois de `turn/end`, a cancelação não afeta o trabalho já concluído.

## 2. Chamada de ferramenta 🟢
```mermaid
stateDiagram-v2
  [*] --> proposed: tool-call no stream
  proposed --> pre_execute: checkpoint (flush)
  pre_execute --> rejected: guard/hook exit 2/política
  pre_execute --> awaiting_approval: decisão ask
  awaiting_approval --> executing: allow
  awaiting_approval --> rejected: deny | cancelled | unavailable (fail-closed)
  pre_execute --> executing: allow
  executing --> backgrounded: run_in_background → job
  executing --> post_execute: resultado | erro | timeout | cancelado
  post_execute --> result: spill/pruner/hooks PostToolUse
  result --> [*]
  rejected --> result
```

## 3. Aprovação 🟢
`pending → allow | deny | cancelled | unavailable`. Com policy `never`, a rejeição é determinística. Uma falha de append da auditoria vira rejeição.

## 4. Escalonamento de sandbox 🟢
```mermaid
stateDiagram-v2
  [*] --> confined: modo da sessão (read-only/workspace-write)
  confined --> denied: acesso negado + dica
  denied --> escalation_request: modelo repete 1× com sandbox_permissions + justification
  escalation_request --> one_shot_wide: usuário permite uma vez
  escalation_request --> refused: rejeita/cancela
  one_shot_wide --> confined: próxima chamada volta ao modo da sessão
```

## 5. Goal 🟢
```mermaid
stateDiagram-v2
  [*] --> active: create_goal
  active --> paused: update_goal/usuário
  paused --> active: retomada
  active --> blocked: relatório blocked
  blocked --> active: retomada humana
  active --> complete: relatório complete
  note right of active: armado/desarmado é flag local; resume/fork ⇒ desarmado
```
O driver de rodadas atua só em `active` + armado + agente ocioso + rodadas restantes (≤ 256). Para em max-tokens, falha de durabilidade, cancelamento ou unload.

## 6. Job 🟢
`running → stopping → settled {completed|failed|killed}`. Aviso de conclusão: `pending → delivered (wakeup | injeção no próximo step)`. Avisos pendentes morrem com o dono.

## 7. Lembrete (schedule) 🟢
```mermaid
stateDiagram-v2
  [*] --> scheduled: schedule_create (após flush)
  scheduled --> overdue: horário passou e agente indisponível/ocupado
  overdue --> dispatched: agente ocioso reivindica maintenance
  scheduled --> dispatched
  dispatched --> [*]: one-shot
  dispatched --> scheduled: every (próxima ocorrência alinhada)
  scheduled --> deleted: schedule_delete
```

## 8. Servidor MCP 🟢
`connecting → initialized (tools/list registradas) → lost → reconnecting (500 ms ×2 até 30 s, máx. 10) → initialized | failed`. A falha inicial não derruba o harness.

## 9. Handle de persistência 🟢
`open(write) → writing → flushing → writing … → closed`. Possíveis desvios: `ownership-lost` e `corruption` (terminais). Leitura do header: `current | migration-required | unsupported | malformed`.

## 10. Execução de pacote cordis dinâmico 🟢
`defined → host-running → awaiting-browser-approval → loaded | declined | failed(stage)` → `stopped` → `undefined`.

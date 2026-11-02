# Domínio — deepseek-harness (dsh)

> Fontes: código (Archaeologist), 355 notas de design em `.agents/notes/implemented/` (128 de arquitetura, 88 de feature, 56 de processo, 39 de bug-fix, 22 de simplificação, 22 de testes). Não há histórico Git: as notas de design substituem a arqueologia por commits.

## 1. Glossário

| Termo | Definição | Confiança |
|---|---|---|
| **Harness** | Runtime que conduz um LLM em loop ReAct com ferramentas, memória de sessão e políticas de segurança | 🟢 |
| **Agent** | Instância viva que executa turnos sobre uma Session. Tem preset, provider/model e inbox | 🟢 |
| **Session** | Log append-only de `SessionEvent` tipados, **única fonte da verdade**. O histórico de mensagens é derivado dele | 🟢 |
| **Turn** | Unidade iniciada por uma mensagem (usuário, goal, lembrete, wakeup de job). Contém 1..N steps e termina em `turn/end` | 🟢 |
| **Step** | Uma requisição ao modelo + o despacho das ferramentas que ela pediu | 🟢 |
| **Inbox** | Duas filas: `nextTurn` (abre um turno novo) e `nextStep` (steering ou aviso injetado no próximo passo) | 🟢 |
| **Provider / Model** | Par obrigatório de roteamento. O provider é a chave de registro do adapter | 🟢 |
| **Adapter** | Implementação de wire protocol (openai-completions, openai-responses, anthropic-messages) | 🟢 |
| **Projection** | Estado derivado por fold puro dos eventos (todo, goal, schedule, stats…) | 🟢 |
| **Compaction / checkpoint** | Substituição do prefixo do histórico por um `<compacted-summary>` de 8 seções | 🟢 |
| **Spill** | Gravação de saída grande em arquivo privado, com preview head/tail no contexto | 🟢 |
| **Preset** | Composição do *agent plane* (ferramentas, seções de prompt, persona) escolhida por sessão | 🟢 |
| **Profile / Bundle / Patch** | Perfil = pilha de bundles + patches que define a superfície do app (web, headless, acp, sdk) | 🟢 |
| **Capability seam** | Tripé Service Definition / Provider / Consumer para capacidades trocáveis (shell, web, LLM, sandbox) | 🟢 |
| **Sandbox mode** | `read-only` (padrão), `workspace-write`, `danger-full-access`. Fail-closed | 🟢 |
| **Escalation** | Uma única nova tentativa com permissão maior, aprovada pelo humano, só para aquela chamada | 🟢 |
| **Job** | Trabalho em background (`<kind>-N`) que pertence a uma sessão de agente | 🟢 |
| **Goal** | Objetivo de longo prazo, conduzido por rodadas automáticas enquanto armado | 🟢 |
| **Workflow** | Script JS que orquestra subagentes (`agent`/`parallel`/`pipeline`/`phase`) | 🟢 |
| **PTC** | *Programmatic tool calling*: o modelo escreve um programa que chama ferramentas como funções | 🟢 |
| **Skill** | Pasta com `SKILL.md` (frontmatter + instruções) carregada sob demanda | 🟢 |
| **Untrusted content** | Conteúdo externo (web, MCP, lembretes) rotulado como dado, nunca instrução | 🟢 |

## 2. Regras de domínio

### 2.1 Sessão e reprodutibilidade
1. **Visível ao modelo ⟺ referenciado de forma durável.** Toda requisição ao modelo pode ser reconstruída byte a byte a partir do log + objetos endereçados por conteúdo + versão do código. Daí vêm a estabilidade do prefix cache, a auditoria e o replay. 🟢 (`reconstructable-requests`)
2. O append no log é **síncrono**. A persistência faz write-behind e drena no checkpoint `session/flush` (fim de turno e barreiras). 🟢
3. A `assistant/message` montada é gravada **antes** do despacho das ferramentas, e o log registra exatamente a mensagem que as ferramentas seguem. 🟢
4. Um par tool-call/tool-result nunca é separado (compactação e fronteiras de passo). 🟢
5. Evento desconhecido de versão alpha histórica é **recusado**. Eventos externos ignoráveis são retidos. 🟢 (`alpha-historical-unknown-event-refusal`, `retain-ignorable-external-session-events`)
6. Formatos lançados têm **migração explícita**, com versão no nome do arquivo. 🟢

### 2.2 Roteamento de modelos
7. Um provider tem **exatamente um** adapter dono. Dois donos são erro de configuração, não fallback. 🟢
8. O par provider/model pode mudar a cada step (`agent/request`), mas fica **fixo dentro do turno** no ACP. 🟢
9. Toda requisição HTTP a provedores leva `User-Agent: deepseek-harness/…` (atribuição obrigatória e não secreta). Headers específicos do OpenRouter **não** são enviados. 🟢
10. Falhas de LLM são fatos serializáveis (`LlmFailure`). A política de retry é separada (5 tentativas, backoff de 500 ms a 10 s, jitter de 10%). 🟢

### 2.3 Segurança e confiança
11. Sandbox **fail-closed**: sem backend capaz, a execução é negada (`SANDBOX_UNAVAILABLE`). 🟢
12. Subprocessos ficam em **contenção nativa**: Job Object kill-on-close no Windows, scope systemd no Linux. A árvore inteira morre junto. 🟢
13. **A credencial armazenada não tem fronteira contra o modelo.** O harness só garante não colocá-la no `process.env` nem entregar o caminho ao modelo. O cofre do SO é registrado como "a resposta real". 🟢 (`credential-boundaries`)
14. Edição exige leitura prévia e versão não obsoleta (`FS_NOT_OBSERVED`, `FS_STALE_VERSION`). 🟢
15. Conteúdo externo (web, MCP, lembretes, resultados de busca) é sempre rotulado como não confiável. 🟢
16. Hooks com exit 2 bloqueiam. Outros erros são registrados e a execução segue. 🟢

### 2.4 Envio de dados (política original × decisão do usuário)
17. Original: `session-log-deepseek.enabled` = **true por padrão em todo processo**. A nota de 2026-09-14 inverteu o opt-in anterior. 🟢
18. Original: telemetria OTel `FEEDBACK_ONLY`, com upload do prefixo da sessão após feedback e user id anônimo. 🟢
19. **Decisão do usuário (2026-11-01):** o harness Rust **não envia nenhum dado** à DeepSeek e só **recebe** melhorias e correções do upstream. O dsh atual foi neutralizado (`DSH_TELEMETRY_DISABLED=1` + patch `enabled: false`). 🟢

### 2.5 Autonomia
20. O goal fica **desarmado depois de resume ou fork** e só volta com autorização humana. 🟢
21. Wakeups automáticos de jobs têm limite de 3 seguidos. Uma mensagem do usuário zera o contador. 🟢
22. O lembrete só dispara com o agente ocioso e nunca interrompe um turno. 🟢
23. O `ralph` vem desligado porque a conclusão é autodeclarada pelo worker, não avaliada de forma independente. 🟢

## 3. Capacidades experimentais encontradas (relevantes para a paridade com o Claude) 🟢
| Capacidade | Pacotes | Equivalente no Claude |
|---|---|---|
| **Computer use** | `computer-use` + Cua Driver (MCP ou nativo) | Computer use |
| **Browser use** | `browser-use` + **Playwright MCP**, **Chrome DevTools MCP**, Stagehand nativo | Claude in Chrome / navegação |
| **Agent teams** | `experimental/agent-team*` | Times de agentes |
| **SSH runtime** | `posix-ssh-runtime` (execução remota) | Execução remota |
| **GitHub review** | overlay `github-review` (webhook) | Code review em PR |

Todas são opt-in. 🟡 No harness Rust: browser use via Playwright/Chrome DevTools MCP, pré-registráveis da mesma forma que REA e GBrain.

## 4. Lacunas 🔴
- Hooks do GBrain (eventos e arquivos), que exigem `gbrain setup claude-code --dry-run`.
- Inventário real do `D:/Projetos/Totvs AGP/VSCode`: agentes, skills, comandos e MCPs a reaproveitar. Ainda não foi feito o levantamento.
- Assinaturas exatas que o usuário possui (Claude Pro/Max? ChatGPT Plus? Copilot? outras?), para priorizar os fluxos OAuth.

# Processo upstream-sync: trazer melhorias do deepseek-harness para o dsh Rust (`dsh upstream`)

## Visão Geral
O dsh Rust **substitui** o dsh TS (decisão `dsh-is-harness`), mas continua aproveitando a evolução do projeto original **deepseek-ai/deepseek-harness** (MIT, © 2026 DeepSeek). O fluxo é **de mão única**: o dsh lê releases, notas de design e contratos do upstream e porta seletivamente o que interessa. **Nenhum dado** do usuário, das sessões ou do inventário vai para a DeepSeek (decisões `upstream-tracking` e `no-outbound-deepseek`, ADR 101). 🟢

Situação verificada em 2026-11-01:
- a análise do Reversa foi feita sobre o snapshot local **0.1.6-alpha.1**;
- o upstream já publicou `dsh-v0.2.1-rc.5` (30/10/2026), `dsh-v0.2.2-alpha.2` (28/10/2026) e `dsh-v0.2.1-rc.4`;
- os dist-tags do npm `@deepseek-ai/dsh` são `latest = 0.2.1-rc.5`, `next = 0.2.1-rc.5` e `alpha = 0.2.2-alpha.2`.

O primeiro ciclo de sincronização deve, portanto, cobrir 0.1.6-alpha.1 → 0.2.1-rc.5. 🟢

## Responsabilidades
- Acompanhar as releases e as tags do upstream, só com leitura. 🟡
- Extrair o que mudou entre a base registrada e a versão nova: notas de design, contratos de pacote, formato de sessão, schemas de ferramentas e correções de bug. 🟡
- Classificar cada mudança e manter um backlog de porte rastreável. 🟡
- Atualizar as specs SDD afetadas e a matriz de impacto antes de mexer no código Rust. 🟡
- Registrar a base sincronizada e cada decisão num ledger. 🟡
- Preservar a atribuição e a licença MIT do código ou das ideias portadas. 🟢

## Interface

### Fontes do upstream (só leitura) 🟢
| Fonte | O que traz | Como é lida |
|---|---|---|
| GitHub Releases `deepseek-ai/deepseek-harness` | tags `dsh-vX.Y.Z[-pre]`, notas de release | API pública sem token (ou com token do usuário só para limite de taxa) |
| Repositório git (tag) | `.agents/notes/{implemented,proposed,rejected,archived}/{architecture,feature,bug-fix,process,simplification,testing}/*.md`, `packages/**/README.md`, `docs/**` | `gix` fetch raso da tag, num diretório espelho |
| npm `@deepseek-ai/dsh` | dist-tags e versões | `GET registry.npmjs.org/@deepseek-ai%2Fdsh` |

As **notas de design** são a fonte principal. No snapshot atual são 1.076 arquivos de notas implementadas (cerca de 355 por idioma), classificadas por tipo e com data no nome. O dsh nunca lê os arquivos `*.zh.md` nem `*.i18n.yaml`. 🟢

### Comandos 🟡
| Comando | Efeito |
|---|---|
| `dsh upstream status` | Base sincronizada × releases disponíveis (estável, rc, alpha) |
| `dsh upstream fetch [--tag X]` | Atualiza o espelho local na tag (padrão: a última rc ou estável) |
| `dsh upstream diff [--from base] [--to tag]` | Relatório de mudanças agrupado por tipo e por componente do dsh Rust |
| `dsh upstream triage` | Abre o backlog com a classificação sugerida para revisão |
| `dsh upstream mark <item> port\|adapt\|skip\|reject [--reason]` | Registra a decisão |
| `dsh upstream baseline <tag>` | Avança a base depois de concluir o ciclo |

### Ledger (`_reversa_sdd/upstream/ledger.md` + `ledger.json`) 🟡
```json
{ "baseline": "0.1.6-alpha.1", "target": "dsh-v0.2.1-rc.5",
  "items": [ { "id": "notes/implemented/bug-fix/2026-09-14-web-diff-context",
               "kind": "bug-fix", "components": ["tui"], "decision": "skip",
               "reason": "interface web não portada", "spec": null, "decided_at": "…" } ] }
```

### Classificação
| Decisão | Quando | Saída |
|---|---|---|
| `port` | Contrato ou correção aplicável como está | Atualizar a spec + tarefa de implementação Rust |
| `adapt` | Ideia válida que precisa de mudança (traits no lugar de cordis, cofre no lugar de YAML, nomes Claude Code) | Spec com uma seção "Adaptação" + tarefa |
| `skip` | Fora do escopo (interface web, Electron, cordis HMR, i18n zh) | Só o registro |
| `reject` | Conflita com uma decisão do usuário (qualquer envio de dados, telemetria, inventário, upload de log) | Registro com a decisão citada |

## Regras de Negócio
- **R1. Mão única:** o processo só faz leituras HTTP e git do upstream. Ele não abre issues nem PRs, não envia telemetria e não registra uso. Contribuir de volta ao upstream é uma decisão manual do usuário, fora do processo. 🟢 (decisão)
- **R2. Rejeição automática:** uma mudança que introduza ou altere envio de dados (padrões: `session-log-deepseek`, `plugin-package-inventory-deepseek`, `telemetry`, `otel`, `upload`, `feedback`, `anonymous-user-id`, `dsh_session_log`, `dsh_plugin_packages`) já vem com a sugestão `reject`, citando ADR 101. 🟡
- **R3.** O porte segue **contratos e notas**, não o código TS linha a linha. Isso vem da dívida 10: a versão alpha muda APIs com frequência. 🟢
- **R4.** As notas `bug-fix` têm **prioridade de triagem**. Cada uma é checada contra o comportamento correspondente nas specs: se o bug existe na spec, a spec é corrigida antes. 🟡
- **R5.** Mudanças no **formato de sessão** (`SESSION_FORMAT_VERSION`, migrações, tipos de evento) são sempre `port` ou `adapt`. Elas exigem uma atualização do importador e leitor v3+ (spec 1, R20 a R21b), para o dsh Rust continuar abrindo sessões criadas pelo dsh TS mais novo. 🟡
- **R6.** Mudanças nos **schemas de ferramentas** são traduzidas para os nomes Claude Code (spec 3), mantendo os aliases. 🟡
- **R7.** Por padrão, as releases `alpha` são acompanhadas só para leitura. A base avança apenas para tags `rc` ou estáveis. 🟡
- **R8.** Cada item portado registra no ledger a origem (tag, caminho da nota ou pacote) e a spec alterada. O código Rust portado leva um comentário de atribuição quando reproduz uma lógica não trivial, e o `THIRD_PARTY_NOTICES` do dsh mantém o aviso MIT da DeepSeek. 🟢/🟡
- **R9.** Um ciclo termina quando todos os itens têm decisão, as specs afetadas estão atualizadas e a matriz de impacto foi revisada. Só então a base avança. 🟡
- **R10.** A cadência sugerida é verificar a cada nova rc ou estável, ou mensalmente. `dsh upstream status` pode ser agendado (spec 16). 🟡
- **R11.** O espelho do upstream fica em `~/.dsh/upstream/mirror/` e nunca é executado. Não há `pnpm install` nem scripts, só leitura de arquivos. 🟡
- **R12. Reversa:** para ciclos grandes (como o primeiro, de 0.1.6 para 0.2.1), o usuário pode rodar o Reversa sobre o espelho para regenerar a análise e comparar com estas specs. 🟡

## Fluxo Principal
1. `dsh upstream status` mostra que a base 0.1.6-alpha.1 está atrás de `dsh-v0.2.1-rc.5`. 🟢
2. `dsh upstream fetch --tag dsh-v0.2.1-rc.5` faz um fetch raso para o espelho. 🟡
3. `dsh upstream diff` lista as notas novas e alteradas por tipo, os pacotes com README alterado, a mudança de formato de sessão e os schemas de ferramentas. Cada item é ligado ao componente do dsh Rust pela matriz de impacto. 🟡
4. `dsh upstream triage` sugere uma decisão por item (R2, R4–R6). O usuário revisa e confirma. 🟡
5. Para cada `port`/`adapt`: atualizar a spec SDD, criar a tarefa no plano do Reconstructor e implementar com testes. 🟡
6. `dsh upstream baseline dsh-v0.2.1-rc.5` grava a nova base no ledger. 🟡

## Fluxos Alternativos
- **Limite de taxa do GitHub:** o dsh usa o token do usuário, do cofre, se houver. Sem token, espera a janela. 🟡
- **Sem rede ou bloqueio corporativo:** o processo aceita um tarball baixado manualmente (`--from-archive`). 🟡
- **Nota que altera uma decisão anterior** (por exemplo, inverte um padrão de envio de dados): vira `reject` com a referência e um aviso destacado no relatório. 🟡
- **Upstream muda de licença:** o processo para e exige revisão manual antes de portar qualquer coisa da nova versão. 🟡

## Dependências
- `gix` (git em Rust puro), `reqwest` (rustls + certificados do SO, spec 7 R16b), parser markdown/frontmatter.
- Specs e matriz de impacto (`_reversa_sdd/traceability/spec-impact-matrix.md`), Reconstructor (plano de implementação).

## Requisitos Não Funcionais
| Tipo | Requisito inferido | Evidência | Confiança |
|---|---|---|---|
| Privacidade | Só leituras; nenhum envio | decisões `upstream-tracking`, `no-outbound-deepseek` | 🟢 |
| Segurança | O espelho nunca é executado | — (novo) | 🟡 |
| Licença | Atribuição MIT preservada | `LICENSE` (MIT, © 2026 DeepSeek) | 🟢 |
| Rastreabilidade | Ledger por item, com a origem e a spec alterada | — (novo) | 🟡 |

> Processo novo. Validar a cadência e o fluxo de triagem com o usuário.

## Critérios de Aceitação
```gherkin
Dado a base 0.1.6-alpha.1
Quando dsh upstream status roda
Então mostra dsh-v0.2.1-rc.5 como alvo sugerido e dsh-v0.2.2-alpha.2 como alpha só para leitura

Dado uma nota nova que acrescenta upload de dados à DeepSeek
Quando a triagem roda
Então o item vem com a sugestão reject citando o ADR 101

Dado uma nota bug-fix sobre pares tool-call/result na compactação
Quando a triagem roda
Então o item é ligado a sdd/compactacao-contexto.md e aparece no topo do backlog

Dado um ciclo com itens sem decisão
Quando o usuário tenta dsh upstream baseline <tag>
Então o comando recusa e lista os itens pendentes

Dado qualquer execução de dsh upstream
Quando o tráfego de rede é inspecionado
Então só há GETs ao GitHub, ao npm e o git fetch
```

## Prioridade
| Requisito | MoSCoW | Justificativa |
|---|---|---|
| Mão única e rejeição de envio de dados | Must | Decisões do usuário |
| `status`/`fetch`/`diff` | Must | Base do processo |
| Triagem + ledger | Should | Rastreabilidade |
| Atualização obrigatória das specs antes do código | Should | Mantém as specs como contrato |
| Primeiro ciclo de 0.1.6-alpha.1 para 0.2.1-rc.5 | Must | A análise já está duas minor versions atrás |
| Agendamento da verificação | Could | Conveniência |

## Rastreabilidade de Código
| Arquivo | Função / Classe | Cobertura |
|---|---|---|
| `.agents/notes/implemented/**` (snapshot 0.1.6-alpha.1) | fonte de decisões | 🟢 |
| `.agents/notes/proposed/**`, `rejected/**` | direções futuras e descartadas | 🟢 |
| `package.json` (`@deepseek-ai/dsh-root` 0.1.6-alpha.1), `LICENSE` (MIT) | base e licença | 🟢 |
| GitHub Releases `deepseek-ai/deepseek-harness`; npm `@deepseek-ai/dsh` | releases novas | 🟢 (consultado em 2026-11-01) |
| `_reversa_sdd/adrs/101-*.md` | fluxo de mão única | 🟢 |
| — | `dsh upstream`, ledger | — (novo) |

## Atualização upstream 0.2.1-rc.5 (ciclo 1, 2026-11-01)
> Origem: [ledger](../upstream/ledger.md). Estas regras **prevalecem** sobre as anteriores quando houver conflito.

- **U1. Ciclo 1 executado:** o [ledger](../upstream/ledger.md) tem 168 notas triadas (port 38 · adapt 25 · skip 94 · reject 11), 76 pacotes novos e 17 removidos, e o formato de sessão passou para v4. As specs dos itens `port`/`adapt` foram atualizadas. Base avançada para `dsh-v0.2.1-rc.5` com a confirmação do usuário (R9). 🟢

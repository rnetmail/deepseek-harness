# Requirements: Backend Linux (sandbox de processos e credenciais fora do Windows)

> Identificador: `001-backend-linux`
> Data: `2026-11-06`
> Pasta da extração reversa: `_reversa_sdd/`
> Confidência: 🟢 CONFIRMADO, 🟡 INFERIDO, 🔴 LACUNA / DÚVIDA

## 1. Resumo executivo

Hoje o dsh só executa comandos e guarda chaves no Windows. No Linux, como numa VPS Ubuntu ou num contêiner, toda execução de shell falha com `SANDBOX_UNAVAILABLE` e não há onde guardar a chave de API. Esta feature faz o dsh funcionar no Linux. Os comandos do agente passam a rodar confinados nos mesmos três modos de sandbox do Windows, e as credenciais ficam num cofre do Linux. Onde o cofre não existir, a origem é declarada pelo usuário. Esta feature é a base das features 002 a 005 da fila (web, contêiner, instalador e `dsh setup`).

## 2. Contexto a partir do legado

| Fonte | Trecho relevante | Confidência |
|-------|------------------|-------------|
| `_reversa_sdd/sdd/seguranca-sandbox-permissoes.md#Backends de sandbox` | Linux: Landlock + seccomp (ou bubblewrap), confina escrita, leitura e rede | 🟢/🟡 |
| `_reversa_sdd/sdd/seguranca-sandbox-permissoes.md#Sandbox` (R2) | Fail-closed: sem backend capaz, `SANDBOX_UNAVAILABLE` e nenhum processo criado | 🟢 |
| `_reversa_sdd/sdd/seguranca-sandbox-permissoes.md#Sandbox` (R6) | Árvore de processos num cgroup/scope no Linux; cancelamento e timeout matam a árvore inteira | 🟢 |
| `_reversa_sdd/domain.md#2.3 Segurança e confiança` (regra 12) | Contenção nativa: scope systemd no Linux | 🟢 |
| `_reversa_sdd/code-analysis.md#sandbox` | Backends do dsh TS: bubblewrap ou Landlock (`native/system`); nível `full`/`partial` por chamada; Landlock com ABI antiga é `partial` | 🟢 |
| `_reversa_sdd/sdd/seguranca-sandbox-permissoes.md#Credenciais` (R16, R17) | Credenciais no cofre do SO com nome `dsh/<escopo>/<id>`; nunca no ambiente dos filhos, no log ou no prompt | 🟡 |
| `_reversa_sdd/sdd/seguranca-sandbox-permissoes.md#Atualização upstream` (U2) | Sem cofre: ambiente do processo, depois `.env` do diretório de invocação, depois `.env` de `~/.dsh` | 🟢 |
| `_reversa_sdd/sdd/seguranca-sandbox-permissoes.md#Fluxos Alternativos` | Cofre indisponível: recusa gravar credencial e orienta usar variável de ambiente | 🟡 |
| `_reversa_sdd/sdd/build-release.md#Artefatos` | Linux é plataforma secundária de release (`x86_64-unknown-linux-gnu`) | 🟡 |

## 3. Personas e cenários de uso

| Persona | Objetivo | Cenário-chave |
|---------|----------|---------------|
| Dono da VPS (Robson) | Usar o dsh numa VPS Ubuntu por SSH ou pela web | Instala o dsh e o agente roda `cargo test` no projeto, confinado ao workspace |
| Operador de contêiner | Rodar o dsh num contêiner sem systemd e sem cofre gráfico | Sobe o contêiner com a chave em variável de ambiente; o agente executa comandos normalmente |
| Usuário Windows atual | Não perder nada | Nada muda no Windows: mesmos backends, mesmo cofre |

## 4. Regras de negócio novas ou alteradas

1. **RN-01:** No Linux, os três modos de sandbox (`read-only`, `workspace-write`, `danger-full-access`) têm backend real com as mesmas semânticas da tabela de modos. 🟢
   - Origem no legado: `_reversa_sdd/sdd/seguranca-sandbox-permissoes.md#Modos de sandbox`
   - Tipo: alterada (antes, no Linux, nenhum modo tinha backend)
2. **RN-02:** A ordem de escolha no Linux vai do confinamento mais forte para o mais fraco. O primeiro backend cuja sonda passa é usado, e cada chamada informa `full` ou `partial`. 🟢
   - Origem no legado: `_reversa_sdd/code-analysis.md#sandbox`
   - Tipo: nova (para o Linux)
3. **RN-03:** Fail-closed continua valendo. Se nenhum backend de confinamento funcionar (kernel sem Landlock e sem namespaces de usuário), os modos `read-only` e `workspace-write` falham com `SANDBOX_UNAVAILABLE` e a mensagem diz o que instalar ou habilitar. Só o `danger-full-access` roda sem confinamento. 🟢
   - Origem no legado: `_reversa_sdd/sdd/seguranca-sandbox-permissoes.md#Sandbox` (R2)
   - Tipo: alterada
4. **RN-04:** No Linux, toda árvore de processos de uma execução morre junto em cancelamento, timeout ou fim da sessão, com ou sem systemd. 🟢
   - Origem no legado: `_reversa_sdd/sdd/seguranca-sandbox-permissoes.md#Sandbox` (R6)
   - Tipo: alterada (o scope systemd vira uma opção; sem systemd, a contenção continua garantida)
5. **RN-05:** As credenciais no Linux vão para o cofre do usuário quando ele estiver disponível, com o mesmo nome `dsh/<escopo>/<id>` do Windows. 🟡
   - Origem no legado: `_reversa_sdd/sdd/seguranca-sandbox-permissoes.md#Credenciais` (R16)
   - Tipo: alterada
6. **RN-06:** Sem cofre (VPS sem sessão gráfica, contêiner), o dsh lê as credenciais na ordem U2: ambiente do processo, `.env` do diretório de invocação, `.env` de `~/.dsh`. Nesse caso `dsh login` não grava nada e diz como declarar a chave. 🟢
   - Origem no legado: `_reversa_sdd/sdd/seguranca-sandbox-permissoes.md#Atualização upstream` (U2) e `#Fluxos Alternativos`
   - Tipo: alterada
7. **RN-07:** A credencial nunca entra no ambiente dos filhos, no log, no spill nem no prompt, venha ela do cofre ou do `.env`. 🟢
   - Origem no legado: `_reversa_sdd/sdd/seguranca-sandbox-permissoes.md#Credenciais` (R17)
   - Tipo: nova (para o Linux)
8. **RN-08:** O dsh compila, passa nos testes e gera o binário `x86_64-unknown-linux-gnu`, sem nenhum código Windows ativo nesse alvo. 🟡
   - Origem no legado: `_reversa_sdd/sdd/build-release.md#Artefatos`
   - Tipo: nova

## 5. Requisitos Funcionais

| ID | Requisito | Prioridade | Critério de aceite | Confidência |
|----|-----------|------------|--------------------|-------------|
| RF-01 | Backend Linux para `workspace-write`: escrita só no workspace, na área temporária privada e nos sinks obrigatórios | Must | Um comando que escreve em `$HOME/fora.txt` falha com a dica de negação; escrever em `./dentro.txt` funciona | 🟢 |
| RF-02 | Backend Linux para `read-only`: nenhuma escrita além dos sinks | Must | `touch ./x` falha com a dica; `cat ./arquivo` funciona | 🟢 |
| RF-03 | `danger-full-access` no Linux roda sem confinamento e informa isso | Must | O comando roda e o resultado traz o backend usado | 🟢 |
| RF-04 | Nível `full` ou `partial` informado em cada chamada no Linux | Must | O resultado e o log trazem o nível e o nome do backend | 🟢 |
| RF-05 | Fail-closed com mensagem acionável quando falta suporte do kernel | Must | Sem Landlock e sem namespaces, `workspace-write` devolve `SANDBOX_UNAVAILABLE` com o motivo de cada backend | 🟢 |
| RF-06 | Contenção da árvore de processos no Linux | Must | Um comando que deixa um neto em background é cancelado, e nenhum processo da árvore sobra | 🟢 |
| RF-07 | Rede de subprocessos negada no `read-only` quando o backend permitir | Should | Com o backend que isola rede, `curl` falha no `read-only`; com o que não isola, o nível é `partial` | 🟡 |
| RF-08 | Cofre de credenciais do Linux quando disponível | Should | Com um cofre de sessão ativo, `dsh login` grava e a próxima execução lê a chave sem variável de ambiente | 🟡 |
| RF-09 | Credenciais por ambiente/`.env` quando não houver cofre | Must | No contêiner, com `GEMINI_API_KEY` no ambiente, o dsh conversa com o provedor e o `dsh doctor` mostra a origem "ambiente" sem o valor | 🟢 |
| RF-10 | `dsh doctor` mostra no Linux qual backend de sandbox está ativo e qual cofre está em uso | Should | A saída lista os backends com ok/falha e o motivo, e a origem das credenciais | 🟡 |
| RF-11 | Build e testes do workspace no alvo Linux | Must | `cargo test --workspace` passa num Linux x86_64 | 🟡 |

## 6. Requisitos Não Funcionais

| Tipo | Requisito | Evidência ou justificativa | Confidência |
|------|-----------|----------------------------|-------------|
| Desempenho | Sonda de backend com timeout de 5 s, uma vez por processo | `_reversa_sdd/sdd/seguranca-sandbox-permissoes.md#Requisitos Não Funcionais` | 🟢 |
| Desempenho | O confinamento não adiciona mais de 50 ms por comando | Comandos do agente são frequentes e curtos | 🟡 |
| Segurança | Fail-closed; nada roda sem confinamento por omissão | R2 | 🟢 |
| Segurança | O ambiente do filho passa pela mesma higienização do Windows | R17 | 🟢 |
| Compatibilidade | Ubuntu 22.04 e 24.04, kernel ≥ 5.13; contêiner Docker padrão sem privilégios | Alvo do usuário: VPS Ubuntu e contêiner | 🟡 |
| Observabilidade | Backend e nível gravados no log de cada execução | R4 de `#Backends de sandbox` | 🟢 |

## 7. Critérios de Aceitação

```gherkin
Cenário: comando confinado ao workspace no Linux
  Dado o dsh rodando no Ubuntu com sandbox workspace-write
  Quando o modelo chama Bash "echo ok > ./dentro.txt && echo x > $HOME/fora.txt"
  Então dentro.txt é criado
  E a escrita em fora.txt é negada com a dica de escalonamento
  E o resultado informa o backend e o nível

Cenário: read-only no Linux
  Dado o modo plan
  Quando o modelo chama Bash "touch ./novo"
  Então a escrita é negada

Cenário: kernel sem suporte
  Dado um Linux sem Landlock e sem namespaces de usuário
  Quando o modelo chama Bash em workspace-write
  Então o resultado é SANDBOX_UNAVAILABLE com o motivo de cada backend
  E nenhum processo é criado

Cenário: cancelamento mata a árvore
  Dado um comando que deixa "sleep 600" em background
  Quando o usuário cancela o turno
  Então nenhum processo da árvore continua vivo

Cenário: chave por variável de ambiente no contêiner
  Dado um contêiner sem cofre com GEMINI_API_KEY no ambiente
  Quando o usuário roda dsh -p "oi"
  Então a resposta chega
  E "env" rodado pelo agente não mostra a chave

Cenário: login sem cofre
  Dado um Linux sem cofre disponível
  Quando o usuário roda dsh login gemini
  Então nada é gravado
  E a mensagem explica como declarar a chave no ambiente ou no .env de ~/.dsh
```

## 8. Prioridade MoSCoW

| Item | MoSCoW | Justificativa |
|------|--------|---------------|
| RF-01, RF-02, RF-03, RF-05, RF-06 | Must | Sem isso o agente não executa nada no Linux |
| RF-04 | Must | Contrato de auditoria já existente |
| RF-09, RF-11 | Must | VPS e contêiner precisam de chave e de build |
| RF-07, RF-08, RF-10 | Should | Melhoram a segurança e o diagnóstico, mas não bloqueiam o uso |
| RNF de desempenho | Should | Perceptível, não bloqueante |

## 9. Esclarecimentos

> Nenhuma sessão de dúvidas registrada ainda. Rode `/reversa-clarify` quando houver `[DÚVIDA]` pendente.

## 10. Lacunas

- 🔴 [DÚVIDA] Numa VPS sem cofre, o `dsh setup` (feature 005) pode **gravar** a chave no `.env` de `~/.dsh` com permissão só do dono, ou só orienta o usuário a fazer isso? R16 proíbe arquivo legível, mas U2 já lê esse `.env`.
- 🔴 [DÚVIDA] Quando o único backend disponível é o Landlock sem isolamento de rede, o `read-only` aceita rodar como `partial` ou falha fechado?

## 11. Histórico de alterações

| Data | Alteração | Autor |
|------|-----------|-------|
| 2026-11-06 | Versão inicial gerada por `/reversa-requirements` | reversa |

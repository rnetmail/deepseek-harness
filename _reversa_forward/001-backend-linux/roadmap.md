# Roadmap: Backend Linux (sandbox de processos e credenciais fora do Windows)

> Identificador: `001-backend-linux`
> Data: `2026-11-06`
> Requirements: `_reversa_forward/001-backend-linux/requirements.md`
> Confidência: 🟢 CONFIRMADO, 🟡 INFERIDO, 🔴 LACUNA

## 1. Resumo da abordagem

A mudança fica concentrada no crate `harness-sandbox` do `dsh-rs`, o equivalente Rust de `packages/sandbox/*` (`_reversa_sdd/code-analysis.md#sandbox`). O módulo novo `linux` implementa o trait `Backend` já existente com três backends, em ordem de força:

1. `bubblewrap`: processo externo `bwrap` com namespaces. Confina escrita e rede, cria um PID namespace e usa `--die-with-parent`.
2. `landlock`: aplicado no próprio filho, antes do `exec`. Confina a escrita; a rede só é confinada com ABI ≥ 4.
3. `unconfined`: só para `danger-full-access`.

A escolha, a sonda com prazo de 5 s e o fail-closed continuam em `Sandbox::run`, sem mudança. A contenção usa um grupo de processos próprio mais `PR_SET_PDEATHSIG`, e a árvore é morta com `kill(-pgid)` em timeout, cancelamento ou fim da execução.

As credenciais ganham um cofre Secret Service (`os_store` no Linux). Quando não há barramento de sessão (VPS por SSH, contêiner), ele responde `Unavailable`, e o `Resolver` já existente segue a ordem U2: ambiente do processo, `.env` do diretório de invocação, `.env` de `~/.dsh`.

O `dsh doctor` passa a listar os backends e o cofre. A validação é feita com `cargo test --workspace` dentro de um contêiner Linux.

## 2. Princípios aplicados

`.reversa/principles.md` não existe neste projeto. Valem os princípios da extração (`_reversa_sdd/architecture.md#2. Estilo arquitetural`):

| Princípio | Como a feature se relaciona | Status |
|-----------|------------------------------|--------|
| Fail-closed | Sem backend capaz, `SANDBOX_UNAVAILABLE`; o `unconfined` nunca atende `read-only` nem `workspace-write` | respeita |
| Microkernel + traits | Os backends novos implementam o trait `Backend`; nada muda no motor | respeita |
| Binário único | O cofre usa D-Bus em Rust puro, sem libdbus em C; o `bwrap` é opcional, e o Landlock vem do kernel | respeita |
| Credenciais fora do alcance do agente | O `.env` de `~/.dsh` é lido só pelo processo do dsh; o ambiente dos filhos é higienizado | respeita (ver P1) |

## 3. Decisões técnicas

| ID | Decisão | Justificativa | Alternativas descartadas | Confidência |
|----|---------|----------------|--------------------------|-------------|
| D-01 | Ordem no Linux: `bubblewrap` → `landlock` → `unconfined` (só `danger-full-access`) | Mesma família de backends do dsh TS (`code-analysis.md#sandbox`). O `bwrap` dá rede e PID namespace; o Landlock funciona em contêiner, onde o `bwrap` costuma falhar por falta de user namespace | seccomp próprio (complexo, frágil); só Landlock (sem isolamento de rede em kernel < 6.7) | 🟢 |
| D-02 | Nível: `bubblewrap` = `full`; `landlock` = `full` com ABI ≥ 4 (rede TCP), senão `partial` | Contrato de `Enforcement` existente: `Full` = escrita + rede, `Partial` = só escrita | — | 🟢 |
| D-03 | `bwrap` montado com `--ro-bind / /`, `--dev /dev`, `--proc /proc`, `--bind` do workspace e da área temporária (só em `workspace-write`), `--unshare-net` em `read-only`, `--unshare-pid` e `--die-with-parent` | Leitura livre e escrita só nos locais concedidos (tabela de modos); rede negada no `read-only` | `--unshare-all` (quebra a rede do `workspace-write`, que é permitida por padrão) | 🟢 |
| D-04 | Landlock: o ruleset é criado no pai; no filho (`pre_exec`), só `restrict_self` + `no_new_privs`. Escrita concedida ao workspace, à área temporária e a `/dev/null`, `/dev/tty`, `/dev/zero` | Entre o `fork` e o `exec`, nada que aloque; o crate `landlock` expõe essa separação | Launcher externo em C (o dsh TS usa `native/system`; aqui é desnecessário) | 🟡 |
| D-05 | Contenção: `process_group(0)` + `PR_SET_PDEATHSIG(SIGKILL)` em todo backend Linux; `kill(-pgid, SIGKILL)` em timeout e cancelamento; com o `bwrap`, o PID namespace mata a árvore inteira | Funciona sem systemd (contêiner). O scope systemd do dsh TS fica de fora | scope do user-systemd (não existe em contêiner); cgroup v2 próprio (exige delegação) | 🟡 |
| D-06 | Limitação registrada: com o `landlock`, um neto que chama `setsid` escapa do `kill(-pgid)` | Sem PID namespace não há contenção total sem cgroup; o `doctor` avisa quando o `bwrap` está ausente | — | 🟡 |
| D-07 | Cofre Linux: `zbus-secret-service-keyring-store` atrás de `keyring-core`, que já é dependência | Rust puro, sem libdbus; mesma API do `windows-native-keyring-store`; nome `dsh/<escopo>/<id>` preservado | `dbus-secret-service` (libdbus em C); `linux-keyutils` (não persiste depois do reboot) | 🟡 |
| D-08 | Sem `DBUS_SESSION_BUS_ADDRESS`, ou com o serviço ausente, o cofre responde `Unavailable` e o `Resolver` aplica a ordem U2, que já está implementada | `credentials/resolve.rs` já trata `Unavailable` | Arquivo de credenciais próprio (viola R16) | 🟢 |
| D-09 | Shell do Bash no Linux: `bash -c` quando existir, senão `/bin/sh -c` | Ubuntu tem bash; imagens mínimas podem não ter | — | 🟢 |
| D-10 | Validação no Linux: `cargo test --workspace` num contêiner `rust` (Docker Desktop local) | Não há distro WSL instalada; cross-compile no Windows não roda os testes | CI remoto (fora do escopo desta feature) | 🟡 |
| D-11 | `dsh doctor` ganha a seção "sandbox": cada backend com ok/falha e motivo, e o cofre em uso ("Secret Service" ou "ambiente/.env") | RF-10 | — | 🟡 |

## 4. Premissas

| Premissa | Origem (`requirements.md` seção) | Risco se errada |
|----------|----------------------------------|-----------------|
| P1: Sem cofre, o `dsh setup` (feature 005) pode gravar a chave em `~/.dsh/.env` com modo `0600`, só depois de uma confirmação explícita. Esta feature apenas lê esse `.env` (U2) | §10, 1ª dúvida | Se o usuário não quiser nenhum arquivo, a feature 005 passa a só orientar; aqui nada muda |
| P2: Com só o Landlock e sem ABI de rede, o `read-only` roda como `partial`, do mesmo jeito que o token restrito no Windows | §10, 2ª dúvida | Se o certo for falhar fechado, basta mudar o `enforcement()` do backend para `None` nesse caso |

## 5. Delta arquitetural

| Componente | Arquivo de origem no legado | Tipo de mudança | Resumo |
|------------|------------------------------|-----------------|--------|
| `harness-sandbox` (backends) | `_reversa_sdd/code-analysis.md#sandbox` | componente-novo | Módulo `linux` com `BubblewrapBackend`, `LandlockBackend` e `UnconfinedBackend` |
| `harness-sandbox` (`Sandbox::platform`) | `_reversa_sdd/sdd/seguranca-sandbox-permissoes.md#Backends de sandbox` | regra-alterada | No Linux, devolve os três backends em vez da lista vazia |
| `harness-sandbox` (cofre) | `_reversa_sdd/sdd/seguranca-sandbox-permissoes.md#Credenciais` | regra-alterada | `os_store()` no Linux = Secret Service |
| `harness-tools` (Bash) | `_reversa_sdd/sdd/ferramentas-claude-code.md` | regra-alterada | Escolha de shell no Linux (D-09) |
| `harness-cli` (`doctor`) | `_reversa_sdd/sdd/superficies-tui-headless-acp-sdk.md` | regra-alterada | Seção de sandbox e cofre |

## 6. Delta no modelo de dados

- Resumo das mudanças: nenhum dado persistido muda. O campo `backend` do log de execução ganha os valores `bubblewrap`, `landlock` e `unconfined` no Linux.
- Detalhe completo em: `_reversa_forward/001-backend-linux/data-delta.md`

## 7. Delta de contratos externos

Nenhum contrato externo muda (não há HTTP, fila ou RPC novos). O diretório `interfaces/` é omitido.

## 8. Plano de migração

n/a: no Linux não existia instalação funcional, e no Windows nada muda.

## 9. Riscos e mitigações

| Risco | Impacto | Probabilidade | Mitigação |
|-------|---------|---------------|-----------|
| O perfil seccomp padrão do Docker bloqueia user namespaces, e o `bwrap` falha | médio | alto | Cai para o `landlock`; o `doctor` explica |
| O Docker bloqueia as syscalls do Landlock (versões antigas) | alto | baixo | Fail-closed com motivo; o `doctor` sugere atualizar o Docker ou usar `danger-full-access` conscientemente |
| O Ubuntu 24.04 restringe user namespaces não privilegiados via AppArmor, e o `bwrap` falha | médio | médio | Cai para o `landlock`; o `doctor` cita o perfil AppArmor do `bwrap` |
| Escape de neto com `setsid` sob o `landlock` (D-06) | baixo | baixo | Aviso no `doctor`; o `bwrap` é recomendado |
| A compilação no Linux revela código que só compila no Windows | médio | médio | Validação no contêiner (D-10) antes de fechar |

## 10. Critério de pronto

- [ ] Todas as ações do `actions.md` marcadas `[X]`
- [ ] `cargo test --workspace` verde no Linux (contêiner) e `scripts/check.ps1 -Gnu` verde no Windows
- [ ] `cross-check.md` (se executado) sem CRITICAL nem HIGH
- [ ] `regression-watch.md` gerado

## 11. Histórico de alterações

| Data | Alteração | Autor |
|------|-----------|-------|
| 2026-11-06 | Versão inicial gerada por `/reversa-plan` (modo autônomo, dúvidas viraram premissas P1 e P2) | reversa |

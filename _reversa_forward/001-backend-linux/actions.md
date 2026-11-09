# Actions: Backend Linux (sandbox de processos e credenciais fora do Windows)

> Identificador: `001-backend-linux`
> Data: `2026-11-06`
> Roadmap: `_reversa_forward/001-backend-linux/roadmap.md`
> Repositório do código: `D:/dsh/dsh-rs`

## Resumo

| Métrica | Valor |
|---------|-------|
| Total de ações | 12 |
| Paralelizáveis (`[//]`) | 4 |
| Maior cadeia de dependência | 6 (T001 → T003 → T004 → T006 → T010 → T012) |

## Fase 1, Preparação

| ID | Descrição | Dependências | Paralelismo | Arquivo alvo | Confidência | Status |
|----|-----------|--------------|-------------|--------------|-------------|--------|
| T001 | Dependências do alvo Linux no crate de sandbox: `landlock`, `rustix` (process, thread) e `zbus-secret-service-keyring-store` | - | - | `crates/harness-sandbox/Cargo.toml` | 🟢 | [ ] |
| T002 | Ambiente de validação Linux: script que roda `cargo test --workspace` num contêiner `rust` montando o repo | - | `[//]` | `scripts/check-linux.ps1` | 🟡 | [ ] |

## Fase 2, Testes

| ID | Descrição | Dependências | Paralelismo | Arquivo alvo | Confidência | Status |
|----|-----------|--------------|-------------|--------------|-------------|--------|
| T003 | Testes de aceitação Linux (`cfg(target_os = "linux")`): escrita dentro e fora do workspace, read-only, danger-full-access, árvore morta no cancelamento e no timeout, nível e backend informados | T001 | - | `crates/harness-sandbox/tests/linux_sandbox.rs` | 🟢 | [ ] |

## Fase 3, Núcleo

| ID | Descrição | Dependências | Paralelismo | Arquivo alvo | Confidência | Status |
|----|-----------|--------------|-------------|--------------|-------------|--------|
| T004 | Execução de processo no Linux: spawn com grupo próprio + `PR_SET_PDEATHSIG`, stdin, coleta de saída, timeout e cancelamento com `kill(-pgid)` | T001 | - | `crates/harness-sandbox/src/linux/process.rs` | 🟡 | [ ] |
| T005 | `BubblewrapBackend`: sonda (`bwrap` no PATH e execução de teste), montagem por modo (D-03), nível `full` | T004 | - | `crates/harness-sandbox/src/linux/bwrap.rs` | 🟢 | [ ] |
| T006 | `LandlockBackend`: sonda da ABI, ruleset no pai, `restrict_self` no `pre_exec`, nível `full`/`partial` pela ABI de rede (D-02, D-04, P2) | T004 | - | `crates/harness-sandbox/src/linux/landlock.rs` | 🟡 | [ ] |
| T007 | `UnconfinedBackend` do Linux e `Sandbox::platform` montando `bubblewrap` → `landlock` → `unconfined` | T005, T006 | - | `crates/harness-sandbox/src/linux/mod.rs` | 🟢 | [ ] |
| T008 | Cofre Secret Service no `os_store()` do Linux, com `Unavailable` sem barramento de sessão (D-07, D-08) | T001 | `[//]` | `crates/harness-sandbox/src/credentials/mod.rs` | 🟡 | [ ] |

## Fase 4, Integração

| ID | Descrição | Dependências | Paralelismo | Arquivo alvo | Confidência | Status |
|----|-----------|--------------|-------------|--------------|-------------|--------|
| T009 | Shell do Bash no Linux: `bash -c` se existir, senão `/bin/sh -c` (D-09) | - | `[//]` | `crates/harness-tools/src` (ferramenta Bash) | 🟢 | [ ] |
| T010 | Compilação e testes do workspace inteiro no Linux, corrigindo o código que só compila no Windows | T007, T008, T009 | - | (vários, conforme os erros) | 🟡 | [ ] |

## Fase 5, Polimento

| ID | Descrição | Dependências | Paralelismo | Arquivo alvo | Confidência | Status |
|----|-----------|--------------|-------------|--------------|-------------|--------|
| T011 | `dsh doctor`: seção "sandbox" com cada backend, ok/falha e motivo, e o cofre em uso (D-11) | T007 | `[//]` | `crates/harness-cli/src` (doctor) | 🟡 | [ ] |
| T012 | `login` sem cofre no Linux: mensagem orientando ambiente ou `~/.dsh/.env`, sem gravar nada | T010 | - | `crates/harness-cli/src` (login) | 🟢 | [ ] |

## Notas de execução

## Histórico de alterações

| Data | Alteração | Autor |
|------|-----------|-------|
| 2026-11-06 | Versão inicial gerada por `/reversa-to-do` | reversa |

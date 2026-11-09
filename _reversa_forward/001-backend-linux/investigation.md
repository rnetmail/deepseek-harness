# Investigation: 001-backend-linux

## Estado atual (dsh-rs)
- `crates/harness-sandbox/src/sandbox.rs`: `Sandbox::platform` devolve uma lista vazia fora do Windows. Com isso, todo comando falha com `SANDBOX_UNAVAILABLE`.
- `crates/harness-sandbox/src/credentials/mod.rs`: `os_store()` fora do Windows responde `Unavailable("sem cofre suportado neste SO")`.
- `crates/harness-sandbox/src/credentials/resolve.rs`: já faz o fallback U2 (ambiente → `.env` de invocação → `.env` de `~/.dsh`) quando o cofre está `Unavailable`.
- `crates/harness-store/src/lock.rs`: já tem o caminho `cfg(not(windows))` com `fs4`.
- `crates/harness-tools/src/hooks.rs`: já usa `/bin/sh -c` fora do Windows.

## Alternativas avaliadas
| Opção | Prós | Contras |
|---|---|---|
| bubblewrap (`bwrap`) | namespaces de montagem, rede e PID; `--die-with-parent`; usado pelo dsh TS e pelo Flatpak | precisa de user namespaces (bloqueados no Docker padrão e, às vezes, pelo AppArmor do Ubuntu 24.04) |
| Landlock (crate `landlock` 0.4) | sem processo externo; funciona em contêiner; kernel ≥ 5.13 | rede só com ABI 4 (kernel 6.7); sem PID namespace |
| seccomp próprio | controle fino | alto risco de quebrar programas; não confina caminhos |
| scope do user-systemd | igual ao dsh TS | não existe em contêiner nem em sessão SSH sem lingering |

## Cofre
- `zbus-secret-service-keyring-store` 1.0: Secret Service via zbus, em Rust puro. Exige um barramento de sessão (gnome-keyring, KeePassXC).
- Numa VPS por SSH normalmente não há barramento de sessão, e aí vale o fallback U2.

## Fontes
- Landlock: https://docs.kernel.org/userspace-api/landlock.html
- bubblewrap: https://github.com/containers/bubblewrap
- Restrição de user namespaces no Ubuntu 24.04: https://ubuntu.com/blog/ubuntu-23-10-restricted-unprivileged-user-namespaces

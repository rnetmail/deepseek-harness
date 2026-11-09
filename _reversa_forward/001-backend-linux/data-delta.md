# Data delta: 001-backend-linux

Nenhuma entidade, tabela ou formato persistido muda.

| Item | Antes | Depois |
|---|---|---|
| `backend` no resultado de execução e no log | `appcontainer`, `restricted-token`, `unconfined` (Windows) | + `bubblewrap`, `landlock`, `unconfined` (Linux) |
| `enforcement` | `full` / `partial` / `none` | sem mudança |
| Origem da credencial (`Source`) | `Vault`, `ProcessEnv`, `InvocationDotenv`, `HomeDotenv` | sem mudança; `Vault` no Linux = Secret Service |

Migração: n/a.

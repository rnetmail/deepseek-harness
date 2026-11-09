# Dúvidas registradas (modo autônomo, sem clarify)

| # | Dúvida | Premissa adotada | Onde |
|---|---|---|---|
| 1 | Sem cofre, o `dsh setup` pode gravar a chave em `~/.dsh/.env` (0600)? | Sim, só com confirmação explícita; a 001 só lê | roadmap.md §4 P1 |
| 2 | Landlock sem ABI de rede: o `read-only` roda `partial` ou falha fechado? | Roda `partial` (igual ao token restrito do Windows) | roadmap.md §4 P2 |

Revise e, se discordar, rode `/reversa-clarify` nesta feature.

# Permissões e fronteiras de confiança

O dsh é **monousuário**: não há RBAC por papéis. O controle de acesso combina modo de sandbox, política de aprovação, posse por agente e confiança por canal.

## 1. Matriz do modo de sandbox 🟢
| Ação | read-only (padrão) | workspace-write | danger-full-access |
|---|---|---|---|
| Ler arquivos | ✅ | ✅ | ✅ |
| Escrever no workspace | ❌ (escalável 1×) | ✅ | ✅ |
| Escrever fora do workspace | ❌ (escalável 1×) | ❌ (escalável 1×) | ✅ |
| Rede em subprocessos | ❌ (escalável) | 🟡 conforme política | ✅ |
| Executar shell | ✅ confinado | ✅ confinado | ✅ sem confinamento |
| PTC (run_code) | confinado igual ao Bash | idem | idem |

Backend indisponível → `SANDBOX_UNAVAILABLE` (nunca degrada em silêncio).

## 2. Política de aprovação 🟢
| Policy | Comportamento |
|---|---|
| `ask` | Pergunta ao respondente do terminal. Sem respondente → `unavailable` (rejeita) |
| `never` | Rejeita deterministicamente o que pedir aprovação |
| Hook `PreToolUse` | Pode bloquear (exit 2), pedir aprovação (`decision: ask`) ou acrescentar contexto |

## 3. Posse e escopo 🟢
| Recurso | Quem pode operar |
|---|---|
| Job | Só a sessão de agente dona |
| Terminal persistente | Só o agente criador |
| Todo list | A sessão. Subagentes têm a própria |
| Busca em sessões anteriores | Só sessões com `cwd` idêntico. Sem `cwd`, só a própria |
| Lembretes | A sessão raiz. Forks não herdam |
| Browser (browser-use) | O agente e a sessão vivos. Um browser externo anexado fica reservado a uma sessão |

## 4. Confiança por canal 🟢
| Canal | Nível |
|---|---|
| Usuário no terminal/web | Humano autoritativo (aprova escalonamentos) |
| Cliente ACP / SDK | **Controlador confiável**: autoriza comandos MCP, env e URLs. `authenticate` não verifica nada |
| UI web | Autenticada por token de navegador (`browser-token-authentication`) |
| Conteúdo web, MCP e lembretes | **Não confiável**: rotulado como dado |
| Metade browser de pacote cordis | Exige aprovação humana na página |
| Credenciais armazenadas | Protegidas contra outros usuários do SO, **não** contra as ferramentas do agente |

## 5. Recomendações para o harness Rust 🟡
1. Modos de permissão equivalentes aos do Claude Code: `default` (pergunta), `acceptEdits`, `plan` (restritivo de verdade), `bypassPermissions`, mapeados sobre os modos de sandbox.
2. Regras `allow`/`deny`/`ask` por ferramenta e padrão, no formato `settings.json` do Claude (`Bash(git:*)`, `Read(./**)`), lidas também de `.claude/settings.json` do projeto.
3. Credenciais no cofre do SO (inacessível às ferramentas do agente).
4. Mostrar o `destructiveHint` de ferramentas MCP (REA) na aprovação.
5. ACP/SDK: exigir token local opcional quando houver exposição além do stdio.

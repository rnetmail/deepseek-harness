# Lacunas — revisão de 2026-11-01

## Lacunas que permanecem sem resposta
| # | Lacuna | Spec | Quem resolve | Pergunta |
|---|---|---|---|---|
| L5 | Cota real do Gemini Pro com 7 agentes (o plano B já foi decidido) | `provedores-modelos.md` R22 (🟡) | Piloto (`quota_stats`) | — |
| L6 | Driver de computer use no Windows: só builds nightly | `browser-computer-agent-teams.md` (🟡) | Piloto | — |
| L7 | Specs baseadas na 0.1.6-alpha.1; upstream na 0.2.1-rc.5 | `upstream-sync.md` | Próximo ciclo (decidido: depois das specs, antes do código) | — |

## Lacunas resolvidas na revisão
| # | Lacuna | Resolução |
|---|---|---|
| L1 | `write_approved_models` e `compat eval` | Todos os modelos ("*"); eval opcional (Q4) |
| L2 | Uso legal do REA em binários TOTVS | Sem restrição (Q5) |
| L3 | Authenticode | Solução de contorno (Q8) |
| L4 | Retenção de sessões | Parâmetro `session_max_days`, padrão 0 (Q9) |
| G1 | Sem spec de configuração em `~/.dsh` | `sdd/configuracao-dsh-home.md` |
| G2 | Sem spec de build e release | `sdd/build-release.md` |
| G3 | Sem retenção de anexos | `sdd/retencao-anexos-spill.md` |
| — | Matriz csharp-ls × .NET | Pesquisa no NuGet (spec 11 R8) |
| — | Computer use no Windows | Builds `cua-driver-rs` windows-x86_64/arm64 encontradas (spec 10) |
| — | Texto do bloco brain-first e hooks do GBrain | `gbrain` 0.60.544.0, instalado sem gravar em `~/.claude` (spec 9 R4/R4b) |
| — | OAuth Google (via 2) | Decisão do usuário: implementar (spec 2 R21) |
| — | Plano B da cota | Decisão do usuário: limitar o paralelismo (spec 2 R22b) |

## Inconsistências corrigidas na revisão
| Spec | Problema | Correção |
|---|---|---|
| `compat-claude-code-totvs.md` R7 | Profundidade 1 marcada como 🟢, mas o padrão do dsh é `maxDepth = 3` | 🟡, com a evidência `tool-subagent/src/index.ts:129` |
| `subagentes-skills-hooks.md` R3 | "máximo 3" quando 3 é o **padrão** | Corrigido |
| `ferramentas-claude-code.md` (catálogo) | `Glob`/`Grep` marcadas como paralelas 🟢; no dsh são exclusivas | 🟡, com uma linha de RNF (`isConcurrencySafe`) |
| `ferramentas-claude-code.md` R9 × `seguranca` R4 | Mudança de cwd do shell × workspace imutável | Esclarecido |
| `memoria-gbrain.md` R6 | A permissão implícita não dizia se prevalece sobre `deny` | O `deny` prevalece |
| `traceability/spec-impact-matrix.md` | Faltavam as linhas das specs 5, 13, 14 e 16 | Acrescentadas |
| `traceability/code-spec-matrix.md` | `settings/*`, `attachment/*` e `util/home-paths` sem spec dedicada | Ligados às specs complementares |

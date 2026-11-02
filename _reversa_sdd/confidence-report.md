# Relatório de Confiança — deepseek-harness → dsh Rust

> Gerado pelo Reviewer em 2026-11-01 e atualizado depois do ciclo 1 de upstream-sync e das respostas Q4, Q5, Q8 e Q9. 20 specs revisadas (17 do Writer + 3 complementares da revisão). Não houve revisão cruzada por outra engine (o plugin Codex não está ativo).

**Como foi contado:**
- cada marcador 🟢/🟡/🔴 nas specs conta como uma afirmação, e marcadores compostos (`🟢/🟡`) contam nos dois níveis;
- menções históricas `[Revisão: era 🔴]` **não** contam como lacuna (5 descontadas).

---

## Resumo Geral

| Nível | Quantidade | Percentual |
|-------|-----------|------------|
| 🟢 CONFIRMADO | 761 | 56,5% |
| 🟡 INFERIDO   | 586 | 43,5% |
| 🔴 LACUNA     | 0 | 0,0% |
| **Total**     | 1.347 | 100% |

**Confiança geral:** **78%** (🟢 + metade dos 🟡)

> Interpretação: a alta proporção de 🟡 é esperada. Uma parte grande das specs descreve **componentes novos** do dsh Rust (Gemini, compatibilidade com o Claude Code, sincronização, plugins, REA, GBrain, upstream-sync, configuração, build e retenção), que não têm código de origem e por isso são inferências de projeto. O que foi extraído do dsh TS está majoritariamente 🟢.

---

## Por Spec

| Spec | 🟢 | 🟡 | 🔴 | Confiança |
|------|----|----|-----|-----------|
| `sdd/agent-loop-sessao.md` | 94 | 6 | 0 | 97% |
| `sdd/browser-computer-agent-teams.md` | 52 | 17 | 0 | 88% |
| `sdd/build-release.md` | 13 | 23 | 0 | 68% |
| `sdd/catalog-sync.md` | 9 | 42 | 0 | 59% |
| `sdd/compactacao-contexto.md` | 41 | 15 | 0 | 87% |
| `sdd/compat-claude-code-totvs.md` | 38 | 42 | 0 | 74% |
| `sdd/configuracao-dsh-home.md` | 15 | 24 | 0 | 69% |
| `sdd/ferramentas-claude-code.md` | 86 | 22 | 0 | 90% |
| `sdd/jobs-schedule-terminal.md` | 54 | 15 | 0 | 89% |
| `sdd/lsp-csharp-plsql.md` | 25 | 31 | 0 | 72% |
| `sdd/mcp-cliente-gerenciado.md` | 37 | 46 | 0 | 72% |
| `sdd/memoria-gbrain.md` | 23 | 34 | 0 | 70% |
| `sdd/plugins-marketplace.md` | 13 | 37 | 0 | 63% |
| `sdd/provedores-modelos.md` | 50 | 35 | 0 | 79% |
| `sdd/rea.md` | 32 | 33 | 0 | 75% |
| `sdd/retencao-anexos-spill.md` | 13 | 29 | 0 | 65% |
| `sdd/seguranca-sandbox-permissoes.md` | 48 | 32 | 0 | 80% |
| `sdd/subagentes-skills-hooks.md` | 60 | 32 | 0 | 83% |
| `sdd/superficies-tui-headless-acp-sdk.md` | 41 | 43 | 0 | 74% |
| `sdd/upstream-sync.md` | 17 | 28 | 0 | 69% |

---

## Lacunas Pendentes 🔴

Nenhuma. As 9 perguntas de `questions.md` foram respondidas (Q1–Q9, 2026-11-01). Ficam só as verificações de piloto (🟡): cota do Gemini (`quota_stats`/`max_concurrent`) e o `cua-driver-rs` nightly no Windows.

---

## Recomendações

- [x] Ciclo 1 de upstream-sync (0.1.6-alpha.1 → 0.2.1-rc.5) triado e aplicado às specs ([ledger](upstream/ledger.md)). Falta confirmar o avanço da base e refazer a code-spec matrix contra os 343 pacotes.
- [ ] **Piloto:**
  - medir a cota do Gemini (`quota_stats`) e definir `max_concurrent`;
  - validar o `cua-driver-rs` nightly no Windows;
  - testar os aliases com os 7 agentes.
- [ ] `catalog-sync.md` e `plugins-marketplace.md` têm a menor confiança (59% e 64%). São componentes novos; validar contra a documentação oficial de plugins do Claude Code e com um protótipo de sincronização.
- [ ] Instalar o .NET 10 SDK para usar o `csharp-ls` atual (spec 11, R8).
- [ ] Rotacionar as chaves que estavam em texto puro no `dsh.cmd` (spec 12, R19).

---

## Histórico de Reclassificações

| De | Para | Afirmação | Evidência |
|----|------|-----------|-----------|
| 🔴 | 🟢 | Comportamento com dsh TS e dsh Rust na mesma sessão (spec 1) | Decisão `dsh-is-harness` + `session-persistence-jsonl/src/win32.ts:153` (mesmo semáforo) |
| 🔴 | 🟢 | `.rar` do CatalogoSKILLs (spec 5 R17) | Decisão do usuário: `.7z` |
| 🔴 | 🟢 | Matriz `csharp-ls` × .NET (spec 11 R8) | NuGet: ≤0.10 net7; 0.11–0.16 net8; 0.17–0.20 net9; ≥0.21 net10 |
| 🔴 | 🟡 | Computer use no Windows (spec 10) | GitHub Releases `cua-driver-rs` 0.38.1 nightly windows-x86_64/arm64 |
| 🔴 | 🟢 | Texto do bloco brain-first e hooks (spec 9 R4/R4b) | `gbrain setup … --instructions-only --dry-run` e `gbrain setup --help` (0.60.544.0) |
| 🔴 | 🟢 | OAuth Google, via 2 (spec 2 R21) | Decisão do usuário |
| 🔴 | 🟡 | Cota do Gemini (spec 2 R22) + plano B 🟢 (R22b) | Decisão do usuário: limitar o paralelismo |
| 🟢 | 🟡 | Profundidade 1 de subagentes (spec 4 R7) | `packages/subagent/tool-subagent/src/index.ts:129` (`maxDepth` padrão 3) |
| 🟢 | 🟡 | `Glob`/`Grep` paralelas (spec 3) | `packages/core/tools/src/index.ts:1284`: só ferramentas `isConcurrencySafe` são paralelas, e `tool-fs-search` não é |
| 🟡 | 🟢 | `Read` com limite de 2.000 linhas (spec 3 R5) | `packages/fs/tool-fs/src/read.ts:15` |
| 🟡 | 🟢 | Timeout de Bash de 120 s / 600 s (spec 3 R10) | `packages/shell/bash-local/src/index.ts:107-108` |
| 🔴 | 🟢 | `write_approved_models` / `compat eval` (spec 4 R19–R20) | Q4: todos os modelos; eval opcional |
| 🔴 | 🟢 | Uso do REA em binários de terceiros (spec 8 R7) | Q5: sem restrição |
| 🔴 | 🟢 | Authenticode (build-release R8) | Q8: solução de contorno |
| 🔴 | 🟢 | Retenção de sessões (retenção R4) | Q9: parâmetro `session_max_days`, padrão 0 |
| 🟡 | 🟢 | `Task` parallel-safe (spec 13 R4) | `isConcurrencySafe` em `packages/subagent/tool-subagent/src/index.ts` |

# Perguntas para Validação — deepseek-harness → dsh Rust

> Gerado pelo Reviewer em 2026-11-01 (answer_mode: chat; as respostas são registradas aqui).
> Duas lacunas foram resolvidas por pesquisa durante a revisão e não viraram pergunta:
> - matriz csharp-ls × .NET (NuGet);
> - builds Windows do Cua Driver (GitHub Releases).

---

## Pergunta 1

**Contexto:** Via 2 do Gemini Pro: login com a conta Google no estilo do Gemini CLI, usando o client OAuth dele, para consumir a cota da assinatura Google AI Pro.
**Spec afetada:** [`sdd/provedores-modelos.md`](sdd/provedores-modelos.md) R21
**Pergunta:** Implementamos a via 2 (OAuth Google próprio do dsh) mesmo sem garantia dos termos de uso, ficamos só com as vias 1 (chave do AI Studio) e 3 (delegar ao Gemini CLI via ACP), ou adiamos até ler os termos?
**Impacto:** Define se a via 2 entra no MVP, fica como Could bloqueada ou sai do escopo.

**Resposta:** ✅ Respondida — Implementar a via 2 (OAuth Google), aceitando o risco dos termos. Spec 2 R21 atualizada (🔴→🟢).

---

## Pergunta 2

**Contexto:** 7 agentes com `model: opus` mapeados para `gemini-2.5-pro`.
**Spec afetada:** [`sdd/provedores-modelos.md`](sdd/provedores-modelos.md) R22
**Pergunta:** Se o piloto mostrar que a cota do Gemini Pro não sustenta os 7 agentes, qual deve ser o segundo modelo para o alias `opus` no Totvs AGP: um modelo local (Ollama), um provedor pago com DPA ou a redução de paralelismo (no máximo N agentes simultâneos)?
**Impacto:** Define a cadeia `aliases.opus` e um limite de concorrência por provedor.

**Resposta:** ✅ Respondida — Limitar o paralelismo por provedor (max_concurrent + fila). Spec 2 R22b. A cota real segue sendo medida no piloto (R22 🟡).

---

## Pergunta 3

**Contexto:** O texto exato do bloco "brain-first" e os hooks que o GBrain instala só aparecem com `gbrain setup claude-code --dry-run`, que exige Bun ≥ 1.4.4 e o GBrain instalados.
**Spec afetada:** [`sdd/memoria-gbrain.md`](sdd/memoria-gbrain.md) R4
**Pergunta:** Posso instalar o Bun e o GBrain agora, só para rodar o `--dry-run` (sem alterar o `CLAUDE.md` nem o `~/.claude`)?
**Impacto:** Com o texto real, R4 passa de 🔴 a 🟢 e os hooks do GBrain entram na spec 13.

**Resposta:** ✅ Respondida — Sim. Bun 1.4.4 (winget) e GBrain 0.60.544.0 instalados. O bloco brain-first v2 foi obtido com `gbrain setup chatgpt --instructions-only --dry-run`, sem gravar nada em ~/.claude. Spec 9 R4 (🔴→🟢) + R4b.

---

## Pergunta 4

**Contexto:** Escrita governada em HML com modelo de fallback.
**Spec afetada:** [`sdd/compat-claude-code-totvs.md`](sdd/compat-claude-code-totvs.md) R19–R20
**Pergunta:** Quais modelos o time aceita em `write_approved_models` e quem define a bateria de conformidade (`dsh compat eval`) com as regras do `CLAUDE.md`?
**Impacto:** Sem isso, toda escrita em HML pelo dsh pede aprovação manual.

**Resposta:** ✅ Respondida — Todos os modelos disponíveis (`write_approved_models = "*"`). O `compat eval` vira diagnóstico opcional. Spec 4 R19–R20 (🔴→🟢).

---

## Pergunta 5

**Contexto:** Análise de DLLs proprietárias do TOTVS RM com o REA.
**Spec afetada:** [`sdd/rea.md`](sdd/rea.md) R7
**Pergunta:** O jurídico ou o contrato com a TOTVS permite engenharia reversa das bibliotecas do RM para fins de manutenção interna?
**Impacto:** Se não permitir, o dsh bloqueia (e não só avisa) a análise de binários assinados pela TOTVS.

**Resposta:** ✅ Respondida — Sem restrição: o REA será usado para fins diversos, talvez nem para isso. Spec 8 R7 (🔴→🟢): sem bloqueio; aviso informativo opcional, desligado.

---

## Pergunta 6

**Contexto:** A análise usou o snapshot 0.1.6-alpha.1; o upstream está em `dsh-v0.2.1-rc.5`.
**Spec afetada:** [`sdd/upstream-sync.md`](sdd/upstream-sync.md)
**Pergunta:** Antes do Reconstructor, rodamos o primeiro ciclo de upstream-sync (0.1.6 → 0.2.1-rc.5) para atualizar as specs, ou começamos a implementação com as specs atuais e sincronizamos depois?
**Impacto:** Ordem do plano de reconstrução.

**Resposta:** ✅ Respondida — Gerar as 3 specs agora e fazer o ciclo de upstream-sync (0.1.6 → 0.2.1-rc.5) depois, antes do código.

---

## Pergunta 7

**Contexto:** A revisão cruzada encontrou três temas sem spec própria (ver [`gaps.md`](gaps.md)):
- layout e precedência dos arquivos de configuração em `~/.dsh`;
- build, empacotamento e release do binário;
- retenção de anexos e spill.

**Pergunta:** Gero essas três specs complementares agora?
**Impacto:** Sem elas, o Reconstructor decide esses pontos sozinho.

**Resposta:** ✅ Respondida — Sim. Geradas `sdd/configuracao-dsh-home.md`, `sdd/build-release.md` e `sdd/retencao-anexos-spill.md`.

---

## Pergunta 8

**Contexto:** Spec complementar de build e release, gerada na revisão.
**Spec afetada:** [`sdd/build-release.md`](sdd/build-release.md) R8
**Pergunta:** Haverá certificado de assinatura de código (Authenticode) para o `dsh.exe`, da empresa ou pessoal, ou o binário será distribuído sem assinatura, com desbloqueio manual?
**Impacto:** Sem certificado, o SmartScreen e as políticas corporativas podem bloquear o executável.

**Resposta:** ✅ Respondida — Usar uma solução de contorno: verificação SHA-256/minisign + `Unblock-File` na primeira instalação, `dsh update` sem Mark-of-the-Web e liberação por hash se houver AppLocker/WDAC. Spec build-release R8 (🔴→🟢).

---

## Pergunta 9

**Contexto:** Retenção opcional de sessões por workspace (LGPD).
**Spec afetada:** [`sdd/retencao-anexos-spill.md`](sdd/retencao-anexos-spill.md) R4
**Pergunta:** As sessões do dsh no Totvs AGP (que podem conter dados de RH) devem ser apagadas automaticamente depois de N dias? Se sim, qual N?
**Impacto:** Define `session_max_days` no `workspace.toml` do Totvs AGP.

**Resposta:** ✅ Respondida — Incluir um parâmetro: `retention.session_max_days` (workspace e global), padrão 0 = desligado. Spec retenção R4 (🔴→🟢).

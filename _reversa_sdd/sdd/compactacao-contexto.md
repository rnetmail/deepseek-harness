# Compactação de contexto, poda de resultados, offload de imagens e spill (`harness-core::compaction`)

## Visão Geral
Sessões longas estouram a janela de contexto do modelo. O dsh resolve isso com quatro mecanismos em camadas, todos registrados no log e sem perder pares `tool/call` ↔ `tool/result` (ADR 009): 🟢
1. **Spill:** saídas grandes vão para arquivo antes de entrar no contexto (spec 3, R17).
2. **Poda:** resultados antigos e muito longos ficam com cabeça e cauda.
3. **Sumarização:** o trecho mais antigo do histórico é trocado por um `<compacted-summary>` estruturado em 8 seções.
4. **Offload de imagens:** as imagens mais antigas saem quando o orçamento de imagens do modelo estoura.

No dsh Rust isso ganha importância, porque o **fallback** pode trocar um modelo de 1M de tokens por um de 128k ou menos no meio da sessão (spec 2). 🟡

## Responsabilidades
- Medir a pressão de contexto a cada passo, **para o modelo de destino** daquele passo. 🟢/🟡
- Disparar a compactação automática no limiar, depois de um overflow, por `/compact` ou programaticamente. 🟢
- Escolher o trecho seguro (o mais antigo, balanceado e preservando a cauda recente) e sumarizá-lo reaproveitando o cache do provedor. 🟢
- Gravar a substituição como `surface_op: replace`, mantendo a reprodutibilidade dos requests. 🟢
- Rodar o hook `PreCompact` e aceitar instruções do usuário. 🟡
- Fazer o offload de imagens sob demanda do adapter. 🟢

## Interface

### Gatilhos 🟢
| Gatilho | Condição |
|---|---|
| Automático | `tokens_medidos ≥ floor(janela_do_modelo_destino × threshold_ratio)` (padrão 0,8) |
| Troca de modelo | **novo:** antes do request, quando o modelo resolvido (fallback ou alias) tem janela menor e a pressão passa do limiar dele 🟡 |
| Overflow | falha `CONTEXT_WINDOW_EXCEEDED`: compacta e repete uma vez (`max_overflow_retries` = 1) |
| Manual | `/compact [instruções]`: no dsh TS não aceita argumentos 🟢; no alvo aceita instruções livres, como no Claude Code 🟡 |
| Região | `compact_region(start, end)` (uso interno e SDK) |

### Configuração
```toml
[compaction]
auto              = true
threshold_ratio   = 0.80
retain_ratio      = 0.16        # cauda verbatim (ou retain_tokens)
summary_max_tokens = 8192
retries           = 1
summarizer        = "same"      # | "<provider>/<model>" (sujeito à política do workspace)

[compaction.pruner]
threshold_chars = 8192          # code points
head_chars      = 4096
tail_chars      = 1024

[[compaction.model_policies]]   # overrides por modelo
match = "openrouter-free/*"
threshold_ratio = 0.70
```

### Formato do checkpoint 🟢
Uma mensagem user-role com `CHECKPOINT_PREAMBLE` + `<compacted-summary>…</compacted-summary>`, contendo exatamente estas seções:
`Primary Request and Intent` · `Key Technical Concepts` · `Files and Code` · `Errors and Fixes` · `Pending Jobs` · `Current Work` · `Next Step` · `Critical Context`.

### Mensagens do `/compact` (estáveis) 🟢
"No compactable history yet." · indisponível (agente ocupado) · histórico alterado · resumo inútil · término incompleto · falha ao salvar · "Compaction cancelled.".

## Regras de Negócio
- **R1. A medição usa:**
  - o `usage` real do provedor quando o envelope do último request bate com o atual;
  - senão, o tokenizer da família do modelo;
  - por último, `ceil(len/4)` mais o overhead de cada bloco (spec 2, R23). 🟢/🟡
- **R2.** A janela usada é a **do modelo de destino** do passo, depois de resolver o alias, a política e o fallback. 🟡
- **R3. Ordem de alívio:** primeiro a poda (se ela já resolver, não há sumarização), depois a sumarização do trecho mais antigo e depois as tentativas extras (`retries`). 🟢
- **R4. Poda:**
  - um resultado de ferramenta com mais de 8.192 code points vira cabeça (4.096) + marcador "middle pruned" + cauda (1.024);
  - a poda só atinge resultados **fora** da cauda retida;
  - o conteúdo completo continua no log e, se houver, no spill. 🟢
- **R5. Seleção do trecho:**
  - é o **span balanceado mais antigo**;
  - a cauda recente (16% da janela ou `retain_tokens`) é preservada verbatim;
  - as fronteiras nunca separam `tool/call` de `tool/result` nem cortam um turno de subagente em andamento. 🟢
- **R6. Sumarização com cache quente:**
  - a chamada auxiliar reproduz o system prompt, as ferramentas e as mensagens do último request roteado e anexa a instrução de compactação no final;
  - assim ela vira um prefixo do request anterior e reaproveita o cache do provedor. 🟢
- **R7.** Um checkpoint anterior (`<compacted-summary>`) dentro do trecho é **consolidado** no novo resumo, nunca copiado literalmente. 🟢
- **R8.** Antes de gravar, o dsh verifica que as fronteiras não mudaram durante a sumarização (`assert_stable`). A gravação é um `replace` do intervalo, o log original continua íntegro e o request continua reconstruível. 🟢
- **R9. Fail-closed:** um resumo truncado (`MAX_TOKENS`) ou com conteúdo não suportado falha, a tentativa fica no log e o histórico não muda. Se não houver um trecho seguro, nada muda. 🟢
- **R10.** O modelo do resumo é o último roteado, ou o configurado em `summarizer`. Ele passa pela **política do workspace**: no Totvs AGP, o resumo nunca vai a um provedor `may-train`. 🟢 (padrão) / 🟡 (política)
- **R11.** O hook `PreCompact` roda antes da sumarização (`trigger: auto|manual`). O `additionalContext` dele e as instruções de `/compact` são anexados à instrução de compactação. 🟡
- **R12.** Depois de uma compactação, o `SessionStart` com `source: compact` roda. As seções de prompt (`CLAUDE.md`, memória, skills) são reinjetadas de forma normal, porque não fazem parte do trecho compactado. 🟡
- **R13. Offload de imagens:** em `IMAGE_OFFLOAD_REQUIRED{n}`, as n imagens mais antigas (pulando as do assistant) são marcadas com `image/offload` e trocadas por um placeholder com o caminho do anexo. O passo é repetido, e isso **não** consome o orçamento de retry do provedor. 🟢
- **R14.** Com `auto = false` (globalmente ou por modelo), só a compactação manual e a de overflow continuam ativas. 🟢
- **R15.** A projeção `context_pressure` (`pressure_tokens`, `projected_tokens`, `context_window`) alimenta o indicador da TUI. A decisão de compactar usa a medição, nunca o indicador. 🟢

## Fluxo Principal
1. No `pre_step`, o dsh resolve o modelo de destino e mede a pressão (R1, R2). 🟢/🟡
2. Se ela passa do limiar: roda o pruner. Se resolveu, segue o passo. 🟢
3. Se não resolveu: roda o `PreCompact`, seleciona o trecho (R5), chama a sumarização com cache quente (R6) e valida a estabilidade (R8). 🟢/🟡
4. Grava o `replace` com o checkpoint e mede de novo. Se ainda estiver acima do limiar, faz uma tentativa extra. 🟢
5. Roda o `SessionStart(compact)` e segue com o request normal. 🟢/🟡

## Fluxos Alternativos
- **Fallback de um modelo de 1M para um de 128k com 300k de contexto:** antes do request, o gatilho "troca de modelo" compacta até caber em 80% de 128k. Se uma passada não bastar, faz as tentativas extras. Se ainda assim não couber, o fallback pula esse candidato e tenta o próximo da cadeia, gravando o motivo `CONTEXT_WINDOW_EXCEEDED`. 🟡
- **Overflow inesperado (a estimativa errou):** compacta e repete uma vez. Se falhar de novo, `turn/end{error}`. 🟢
- **`/compact` com o agente ocupado:** a mensagem "indisponível". 🟢
- **Cancelamento durante a sumarização:** "Compaction cancelled.", sem nenhuma alteração. 🟢

## Dependências
- Loop e sessão (spec 1: `pre_step`, `surface_op`). Modelos e política (spec 2). Hooks (spec 13). Ferramentas e spill (spec 3). Anexos (`harness-store`).

## Requisitos Não Funcionais
| Tipo | Requisito inferido | Evidência no código | Confiança |
|---|---|---|---|
| Custo | Sumarização como prefixo do request anterior (cache do provedor) | `packages/compaction/compaction-basic/src/summarizer.ts` | 🟢 |
| Consistência | Pares de ferramenta preservados; checagem de estabilidade | `compaction-basic/src/region.ts` | 🟢 |
| Disponibilidade | Uma nova tentativa depois de overflow; offload sem consumir retry | `compaction-basic`, `compaction-image-offload` | 🟢 |
| Desempenho | Limiar de 80%, cauda de 16%, resumo de até 8.192 tokens | config do `compaction-basic` | 🟢 |
| Privacidade | Resumo sujeito à política do workspace | decisão `totvs-data-gate` | 🟡 |

> Inferido a partir do código. Validar o comportamento com modelos de janela pequena no piloto.

## Critérios de Aceitação
```gherkin
Dado uma sessão com 85% da janela ocupada e resultados de ferramenta longos antigos
Quando o próximo passo começa
Então o pruner reduz esses resultados e, se ficar abaixo de 80%, nenhuma sumarização acontece

Dado uma sessão acima do limiar mesmo depois da poda
Quando a compactação roda
Então o trecho mais antigo vira um <compacted-summary> com as 8 seções
E nenhum tool/call fica sem o seu tool/result no histórico derivado

Dado o fallback trocando gemini-2.5-pro (1M) por um modelo de 128k com 300k de contexto
Quando o request vai ser montado
Então a compactação roda antes, para caber em 80% de 128k, ou o candidato é pulado com o motivo registrado

Dado /compact "preserve as decisões sobre o S-2240"
Quando a compactação roda
Então a instrução do usuário entra na instrução de compactação

Dado o workspace Totvs e summarizer = "openrouter-free/x"
Quando a compactação roda
Então o resumo é feito pelo modelo atual permitido e o motivo é registrado

Dado um resumo que estoura max_tokens
Quando a compactação termina
Então o histórico não muda e a falha fica no log
```

## Prioridade
| Requisito | MoSCoW | Justificativa |
|---|---|---|
| Compactação automática e de overflow | Must | Sessões longas e fallback |
| Pares preservados e falha fechada | Must | Integridade do histórico |
| Medição pelo modelo de destino (troca de modelo) | Must | Fallback para janelas menores |
| Poda de resultados | Should | Evita a sumarização na maior parte dos casos |
| Resumo com cache quente | Should | Custo em modelos pagos |
| `/compact` com instruções + `PreCompact` | Should | Paridade com o Claude Code |
| Offload de imagens | Should | Sessões com screenshots (browser e computer use) |
| Overrides por modelo | Could | Ajuste fino |

## Rastreabilidade de Código
| Arquivo | Função / Classe | Cobertura |
|---|---|---|
| `packages/compaction/compaction` | base e marcador | 🟢 |
| `packages/compaction/compaction-basic/src/region.ts`, `summarizer.ts` | seleção e resumo | 🟢 |
| `packages/compaction/compaction-tool-result-pruner` | poda | 🟢 |
| `packages/compaction/compaction-image-offload` | offload de imagens | 🟢 |
| `packages/compaction/command-compact` | `/compact` | 🟢 |
| `packages/llm/token-meter` | medição | 🟢 |
| `_reversa_sdd/adrs/009-*.md` | decisão | 🟢 |
| — | gatilho por troca de modelo, `PreCompact`, instruções no `/compact` | — (novo) |

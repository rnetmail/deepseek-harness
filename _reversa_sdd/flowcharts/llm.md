# Fluxogramas — módulo `llm`

## 1. Chamada de modelo (`ctx.llm.stream`) 🟢

```mermaid
flowchart TD
  A[GenerateOptions<br/>provider, model, messages, tools] --> B{Rota registrada?}
  B -- não --> F1[finish error NO_ADAPTER]
  B -- sim --> C[resolveModelInfo<br/>contexto, efforts, modalidades]
  C --> D{reasoningEffort válido?}
  D -- não --> F2[falha antes de I/O]
  D -- sim --> E[materializa defaults do adapter<br/>maxTokens etc. + deep-freeze]
  E --> G[prepareCall: liga à geração exata do adapter]
  G --> H[projeção de conteúdo]
  H --> H1[FileBlock → texto com caminho read-only]
  H --> H2{modelo aceita imagem?}
  H2 -- não --> H3[placeholder determinístico]
  H2 -- sim --> H4{dentro do orçamento de imagem?}
  H4 -- não --> F3[finish error IMAGE_OFFLOAD_REQUIRED n]
  H4 -- sim --> I
  H3 --> I[waterfall llm/stream → adapter]
  H1 --> I
  I --> J[chunks: block-start / deltas / block-end]
  J --> K[usage]
  K --> L[finish stop | tool-calls | max-tokens | error | aborted]
```

## 2. Retry (`llm-retry`, ponto de extensão failed-step) 🟢

```mermaid
flowchart TD
  A[Passo falhou com LlmFailure] --> B{adapter final selecionado?}
  B -- não --> Z[delega a jusante]
  B -- sim --> C{modo}
  C -- normal --> D{código elegível e orçamento > 0?}
  D -- não --> Z
  D -- sim --> E[delay = Retry-After se dentro dos limites<br/>senão backoff exp 500ms..10s ±10%]
  C -- always --> C1[consulta recuperação a jusante]
  C1 --> E2[delay = Retry-After ou backoff local]
  E --> F[append evento llm/retry]
  E2 --> F
  F --> G[aguarda delay — abortável]
  G --> H[append llm/retry-started]
  H --> I[re-executa o passo no mesmo turno]
```

## 3. Resolução de catálogo e descoberta (`llm-pi-ai`) 🟢

```mermaid
flowchart TD
  A[Perfil da rota: base ⊕ settings usuário] --> B{rota existe no catálogo pi-ai?}
  B -- sim --> C{models definido?}
  C -- sim --> C1[substitui catálogo; cada entrada herda campos do modelo instalado]
  C -- não --> C2[catálogo instalado + modelOverrides]
  B -- não --> D{api + baseURL + models não vazio?}
  D -- não --> X[recusa: settings-rejected / INVALID_CONFIG]
  D -- sim --> E[rota declarada à mão]
  F[Descoberta pedida pela UI] --> G{rota do catálogo?}
  G -- sim --> G1[responde do catálogo, sem rede]
  G -- não --> G2{api}
  G2 -- openai-* --> H1[GET baseURL/models Bearer]
  G2 -- anthropic-messages --> H2[GET /v1/models?limit=1000 x-api-key]
  H1 --> I[normaliza id, nome, contextWindow, maxTokens<br/>nada é gravado]
  H2 --> I
```

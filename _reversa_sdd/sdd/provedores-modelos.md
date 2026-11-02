# Provedores e modelos (`harness-llm`)

## Visão Geral
Camada única entre o harness e qualquer modelo de linguagem. Ela oferece:
- adapters de protocolo (OpenAI Chat, OpenAI Responses, Anthropic Messages e **Gemini generateContent**);
- um catálogo de modelos **gratuitos** listado automaticamente, além de cadastro manual;
- login das **assinaturas** do usuário, com prioridade para o Google Gemini Pro;
- uma **cadeia de fallback**;
- um **mapa de aliases** (`opus`/`sonnet`/`haiku`) para rodar agentes escritos para o Claude.

Também aplica a política de dados por workspace (`data_policy`, `allowed_providers`) e o **gate de rede corporativa** exigido pelo Totvs AGP. 🟢 contrato do dsh / 🟡 extensões do alvo.

## Responsabilidades
- Expor um único `LlmRuntime::stream()` com protocolo de chunks neutro em relação ao provedor. 🟢
- Registrar adapters por rota, com exatamente um dono por provider. 🟢
- Manter o catálogo de provedores e modelos (embutido + atualizável + manual) e descobrir modelos de endpoints compatíveis. 🟢/🟡
- Autenticar por API key (cofre do SO) ou por OAuth de assinatura, com renovação sob lock entre processos. 🟢/🟡
- Classificar falhas por código (`LlmFailure`) e aplicar retry limitado, depois fallback. 🟢/🟡
- Resolver aliases de modelo vindos de `.claude/agents/*.md` (`model: opus`). 🟡
- Bloquear provedores fora da política do workspace **antes** de qualquer I/O. 🟡
- Medir o contexto (tokens) para alimentar a compactação. 🟢

## Interface

### Contrato de stream 🟢
```
block-start → (text-delta | reasoning-delta | tool-call-delta)* → block-end → usage → finish
```
- Há sempre **exatamente um** `finish` terminal, e nada vem depois dele.
- Os argumentos de tool-call trafegam como string JSON crua e são montados pelo `BlockAssembler`.
- `PreparedCall` amarra os fatos do modelo, o contexto e a política de retry à geração atual do adapter, e pode ser despachado **uma única vez**.

### Trait de adapter 🟢→🟡
```rust
trait LlmAdapter: Send + Sync {
    fn routes(&self) -> &[ProviderId];
    async fn stream(&self, call: PreparedCall, cancel: CancellationToken)
        -> Result<BoxStream<'static, StreamChunk>, LlmFailure>;
    async fn discover_models(&self, req: DiscoverReq) -> Result<Vec<ModelInfo>, LlmFailure>;
}
```
Protocolos do alvo:

| Protocolo | Estado | Confiança |
|---|---|---|
| `openai-chat` | Portado | 🟢 |
| `openai-responses` | Portado | 🟢 |
| `anthropic-messages` | Portado | 🟢 |
| `gemini-generate-content` | **Novo** (SSE `streamGenerateContent?alt=sse`) | 🟡 |

### Configuração (`~/.dsh/providers.toml` + `.dsh/workspace.toml`) 🟡
```toml
[providers.gemini]
protocol    = "gemini-generate-content"
auth        = "google-oauth"            # | "api-key" (AI Studio) | "acp:gemini-cli"
data_policy = "no-training"             # no-training | may-train | local | unknown
models      = ["gemini-2.5-pro", "gemini-2.5-flash"]
max_concurrent = 4                      # R22b: fila por provedor (valor do piloto)

[providers.openrouter-free]
protocol    = "openai-chat"
base_url    = "https://openrouter.ai/api/v1"
auth        = "api-key"                 # chave no cofre do SO: harness/openrouter-free
data_policy = "may-train"
models      = "catalog:free"            # listagem automática

[providers.ollama]
protocol    = "openai-chat"
base_url    = "http://localhost:11434/v1"
auth        = "none"
data_policy = "local"

[aliases]                               # agentes .claude com model: opus/sonnet/haiku
opus   = ["gemini/gemini-2.5-pro", "anthropic/claude-opus-*"]
sonnet = ["gemini/gemini-2.5-flash", "openrouter-free/qwen3-coder:free"]
haiku  = ["gemini/gemini-2.5-flash-lite", "ollama/qwen3:8b"]

[fallback]
chain = ["anthropic", "gemini", "openrouter-free", "ollama"]
```

```toml
# D:/Projetos/Totvs AGP/VSCode/.dsh/workspace.toml
[policy]
allowed_data_policies = ["no-training", "local"]
allowed_providers     = ["gemini", "ollama", "anthropic"]
data_gate             = "ask"           # ask | allow | deny  (decisão totvs-data-gate)

[network_gate]
probe   = [{ dns = "gpflow.grupopereira.link" }]   # ou { tcp = "host:1521" }, { tnsping = "ALIAS" }
guards  = ["mcp:gpflow", "skill:gpflow-*"]          # o que só liga com a rede da empresa
```

### CLI 🟡
| Comando | Efeito |
|---|---|
| `dsh models list [--free] [--provider X] [--policy no-training]` | Lista o catálogo, marcando `free`, `data_policy`, janela, imagens e raciocínio |
| `dsh models add <provider>/<model> [--context N --images --reasoning]` | Cadastra um modelo à mão |
| `dsh models refresh` | Atualiza o catálogo de modelos gratuitos (models.dev / OpenRouter `/models`), com assinatura minisign do índice |
| `dsh login <provider>` | OAuth (PKCE ou device code) ou pedido de API key gravada no cofre do SO |
| `dsh models doctor` | Testa as credenciais, o gate de rede e a política de cada provedor |

### Códigos de falha 🟢
`NO_ADAPTER`, `DUPLICATE_ADAPTER`, `MISSING_CREDENTIAL`, `INVALID_CREDENTIAL`, `AUTH`, `RATE_LIMIT`, `QUOTA`, `SERVER`, `TIMEOUT`, `TRANSPORT`, `EMPTY_RESPONSE`, `CONTEXT_WINDOW_EXCEEDED`, `IMAGE_OFFLOAD_REQUIRED`, `UNKNOWN_MODEL`, `INVALID_CONFIG`, `UNSUPPORTED_OPTION`.

Novos: `POLICY_DENIED`, `NETWORK_GATE_CLOSED`. 🟡

## Regras de Negócio

### Roteamento
- **R1.** Um provider tem exatamente um adapter dono. Dois donos são erro de configuração (`DUPLICATE_ADAPTER`), não fallback. A troca do conjunto de rotas é atômica: se for recusada, o estado anterior continua valendo. 🟢
- **R2.** Consumidores roteiam pelo **código** da falha, nunca pelo texto. A detecção de estouro de contexto e de cota por texto do provedor fica centralizada num único lugar. 🟢
- **R3.** O replay state (assinaturas de thinking etc.) só viaja quando o mesmo adapter é dono da rota de origem e da rota de destino. Numa troca por fallback, ele é descartado. 🟢
- **R4.** Um modelo só-texto recebe um placeholder determinístico por imagem, sem reescrever o histórico. Um `FileBlock` vira uma linha de texto com o caminho, e o modelo lê o arquivo com ferramentas. 🟢
- **R5.** `reasoning_effort` é validado contra o modelo exato antes de qualquer I/O. `stop` é rejeitado com `UNSUPPORTED_OPTION`. 🟢
- **R6.** Toda requisição leva `User-Agent: dsh/<versão>`. Nenhum campo extra de log ou inventário é anexado. O `dsh_session_log` e o `dsh_plugin_packages` não existem no alvo. 🟢/🟡

### Retry e fallback
- **R7. Retry no modo `normal`:** até 5 tentativas para `EMPTY_RESPONSE`, `RATE_LIMIT`, `SERVER`, `TIMEOUT` e `TRANSPORT`, com backoff de 500 ms a 10 s e jitter de 10%. Um `Retry-After` válido substitui o backoff se couber nos limites. O evento `llm/retry` é gravado **antes** da espera. 🟢
- **R8. Fallback:** depois de esgotar o retry, ou imediatamente em `QUOTA`, `AUTH`, `INVALID_CREDENTIAL`, `MISSING_CREDENTIAL` ou `NETWORK_GATE_CLOSED`, o waterfall `request` tenta o próximo provedor da `fallback.chain` que passe em R10 a R12. Cada troca grava `llm/fallback {from, to, failure}` e é avisada na TUI. 🟡
- **R9.** O fallback nunca troca de modelo dentro de um turno ACP (o par fica fixo no ACP). Nesse caso a falha encerra o turno com `error`. 🟢 (regra do dsh) / 🟡 (aplicação ao fallback)

### Política de dados
- **R10.** Um provedor cujo `data_policy` não esteja em `allowed_data_policies`, ou que não esteja em `allowed_providers` do workspace, é **filtrado antes da chamada** com `POLICY_DENIED`. Ele nunca recebe nem o system prompt. Um provedor `unknown` é tratado como `may-train`. 🟡
- **R11.** O workspace Totvs AGP (dados de RH sob LGPD) vem configurado com `allowed_data_policies = ["no-training","local"]`. Modelos gratuitos `may-train` ficam indisponíveis nele, mesmo como fallback. 🟡
- **R12. Gate de dados (`totvs-data-gate`):**
  - com `data_gate = "ask"`, a primeira chamada da sessão que envolva conteúdo do workspace pede autorização ao usuário (uma vez por sessão, com opção de recusar);
  - a decisão é gravada no log (`policy/data-gate {decision, by:"user"}`);
  - `deny` impede qualquer provedor remoto (só `local` é aceito). 🟡
- **R13. Gate de rede:**
  - as sondas de `network_gate.probe` (resolução DNS, TCP ou `tnsping`) rodam no início da sessão e a cada 5 min, com timeout de 3 s;
  - com o gate fechado, os itens em `guards` (MCP `gpflow`, skills `gpflow-*`) ficam desativados e são marcados como "requer rede da empresa", e o agente é avisado por uma seção de prompt;
  - o resto do harness continua funcionando. 🟡

  A sessão em que esta spec foi escrita mostrou exatamente esse caso: `getaddrinfo ENOTFOUND gpflow.grupopereira.link` fora da rede. 🟢

### Catálogo e aliases
- **R14.** O catálogo embutido é exportado do pi-ai 0.85 (39 provedores, 113 modelos de custo zero) e atualizado por `models refresh`. Uma rota do catálogo responde **sem rede**. Uma rota manual descobre modelos via `GET {base_url}/models` (OpenAI) ou `/v1/models?limit=1000` (Anthropic). A descoberta só sugere: nada é gravado sem confirmação. 🟢/🟡
- **R15.** `models` numa rota **substitui** o catálogo daquela rota, e cada entrada herda os campos do modelo de mesmo id. `model_overrides` é recusado junto com `models`, em rota manual ou para um id inexistente. 🟢
- **R16.** Um alias (`opus`, `sonnet`, `haiku` e qualquer nome definido pelo usuário) resolve para o **primeiro** candidato da lista que tenha credencial válida e passe em R10 a R13. Um glob (`claude-opus-*`) escolhe a versão mais nova do catálogo. Se nenhum candidato servir, o agente não inicia e a falha `UNKNOWN_MODEL` cita o alias. 🟡
- **R17.** Um modelo marcado `free` em `may-train` aparece na listagem com o aviso "pode usar seus prompts para treinamento". 🟡

### Credenciais e assinaturas
- **R18.** As credenciais ficam no **cofre do SO** (Windows Credential Manager via `keyring`), nunca em YAML/TOML legível pelas ferramentas do agente, nunca em variável de ambiente de processos filhos e nunca no log. Isso corrige a dívida 2 e o ADR 007 do dsh. 🟡
- **R19.** O refresh de OAuth acontece sob um lock entre processos (arquivo de lock + cofre), para que duas instâncias não invalidem o token uma da outra. 🟢
- **R20. OAuth portado do dsh/pi-ai:** Anthropic (Claude Pro/Max), OpenAI (ChatGPT Plus/Pro via Codex), GitHub Copilot, xAI, Kimi Code, OpenRouter. 🟢
- **R21. Gemini Pro (decisão `gemini-subscription`), três vias, nesta ordem:**
  1. `api-key`: chave do AI Studio, com protocolo nativo; inclui a cota gratuita. 🟡
  2. `google-oauth`: login com a conta Google (estilo Gemini CLI, loopback PKCE) para usar a cota da assinatura Google AI Pro. 🟢 **Decisão do usuário (2026-11-01): implementar**, mesmo sem confirmação dos termos de uso do client OAuth. O risco aceito é o bloqueio ou a revogação do acesso pelo Google. O dsh mostra esse aviso no primeiro `dsh login gemini --oauth`, e as vias 1 e 3 continuam disponíveis como alternativa. [Revisão: era 🔴]
  3. `acp:gemini-cli`: o harness delega a sessão ao `gemini --experimental-acp` instalado pelo `harness-integrations`. O Gemini CLI usa o próprio login dele. 🟡
- **R22. Cota do Gemini Pro para 7 agentes `opus` simultâneos:** desconhecida. O harness deve registrar `RATE_LIMIT`/`QUOTA` por provedor numa projeção `quota_stats` para medir no piloto. 🟡
- **R22b. Plano B de cota (decisão do usuário, 2026-11-01):** se a cota não sustentar os agentes, o dsh **limita o paralelismo** por provedor (`[providers.gemini] max_concurrent = N`, padrão a definir no piloto). Chamadas acima do limite esperam numa fila FIFO por provedor, sem trocar de modelo. A fila e a espera aparecem na TUI. O fallback por `QUOTA` (R8) continua valendo quando a cota se esgota de vez. 🟢 [Revisão]

### Medição
- **R23.** Uso real do provedor quando o envelope da última chamada bate com o atual. Caso contrário, estimativa por tokenizer da família (`tiktoken-rs` para OpenAI/Anthropic aproximado, contagem da API do Gemini) e, por último, `ceil(len/4)`. 🟢/🟡

## Fluxo Principal
1. O loop chama o waterfall `request` com `{provider, model}` do agente. Se o modelo for um alias, ele é resolvido aqui (R16). 🟡
2. Filtro de política (R10 a R12) e gate de rede (R13). Uma reprovação passa para o fallback (R8) ou devolve `POLICY_DENIED`. 🟡
3. `prepare_call()` valida as opções (R5), injeta a credencial a partir do cofre (renovando o OAuth se necessário) e congela o request. 🟢
4. O adapter abre o stream HTTP (SSE) e traduz o wire para `StreamChunk`. O idle timeout do stream é de 300 s. 🟢
5. O `BlockAssembler` monta a `assistant/message`. O `usage` alimenta `token_usage` e `quota_stats`. 🟢

## Fluxos Alternativos
- **429 / 5xx / timeout:** retry (R7). Se esgotar, fallback (R8). 🟢/🟡
- **Cota da assinatura esgotada (`QUOTA`):** fallback imediato para o próximo provedor permitido. Se não houver nenhum, `turn/end{error}` com uma mensagem que diz qual política bloqueou cada candidato. 🟡
- **Fora da rede da empresa:** o gate fecha, o `gpflow` e as skills dependentes ficam desativados, e o agente é informado; os modelos remotos permitidos continuam funcionando. 🟡
- **Workspace Totvs com o Claude indisponível:** o alias `opus` resolve para `gemini-2.5-pro` (`no-training`). Um modelo gratuito `may-train` nunca é escolhido. 🟡
- **`CONTEXT_WINDOW_EXCEEDED`:** não há retry. A falha é entregue à compactação, que reduz o contexto e repete o passo uma vez. 🟢
- **`IMAGE_OFFLOAD_REQUIRED`:** a compactação faz o offload das N imagens mais antigas e repete. 🟢

## Dependências
- `harness-core`: pontos de extensão `request` e `request_error`.
- `harness-store`: eventos `llm/retry`, `llm/fallback`, `policy/*` e projeções `token_usage`/`quota_stats`.
- Cofre do SO (`keyring`): credenciais e tokens OAuth.
- `harness-integrations`: instalação do Gemini CLI (via 3) e do Ollama (opcional).
- `harness-claude-compat`: aliases vindos de `.claude/agents`.

## Requisitos Não Funcionais
| Tipo | Requisito inferido | Evidência no código | Confiança |
|---|---|---|---|
| Disponibilidade | 5 retries, 500 ms a 10 s, jitter de 10%, `Retry-After` respeitado | `packages/llm/llm-retry` | 🟢 |
| Desempenho | Idle timeout de stream de 300 s, janela padrão de 262.144, saída padrão de 32.768 | `packages/llm/llm-pi-ai` (defaults) | 🟢 |
| Segurança | Credencial só no provider e nunca no env de processos filhos; cofre do SO | `adrs/007`, `packages/llm/llm-pi-ai/src/auth.ts` | 🟢/🟡 |
| Privacidade | Filtro de política antes de qualquer I/O; nenhum campo extra no request | decisões `no-outbound-deepseek`, `totvs-data-gate` | 🟡 |
| Disponibilidade | Sondas do gate de rede com timeout de 3 s, a cada 5 min | — (novo) | 🟡 |

> Inferido a partir do código. Validar com a equipe de operações.

## Critérios de Aceitação
```gherkin
Dado o workspace Totvs AGP com allowed_data_policies = [no-training, local]
Quando um agente com model: opus inicia e a Anthropic está sem credencial
Então o alias resolve para gemini/gemini-2.5-pro e o evento request/header registra esse par

Dado o mesmo workspace e a cadeia [anthropic, gemini, openrouter-free]
Quando o Gemini devolve QUOTA
Então openrouter-free é ignorado com POLICY_DENIED e o turno termina em error citando as políticas

Dado um workspace sem restrição de política
Quando o provedor principal devolve 503 seis vezes seguidas
Então há 5 eventos llm/retry, um llm/fallback para o próximo provedor e o turno conclui

Dado a máquina fora da rede da empresa
Quando a sessão inicia no Totvs AGP
Então a sonda DNS de gpflow.grupopereira.link falha, mcp:gpflow e skill:gpflow-* ficam desativados com o motivo "requer rede da empresa"
E as demais ferramentas e modelos permitidos continuam disponíveis

Dado data_gate = ask
Quando a primeira chamada da sessão inclui conteúdo do workspace
Então o usuário é consultado e a decisão fica gravada em policy/data-gate

Dado dsh models list --free
Quando o catálogo é exibido
Então cada modelo mostra data_policy, e os may-train trazem o aviso de treinamento
```

## Prioridade
| Requisito | MoSCoW | Justificativa |
|---|---|---|
| Contrato de stream + 3 protocolos portados | Must | Caminho crítico de todo request |
| Adapter Gemini nativo (via 1) | Must | Assinatura do usuário e principal substituto do Claude |
| Cofre do SO para credenciais | Must | Corrige o risco de credenciais legíveis |
| `data_policy` / `allowed_providers` / gate de dados | Must | LGPD no Totvs AGP |
| Mapa de aliases `opus`/`sonnet`/`haiku` | Must | Os 7 agentes do Totvs usam `model: opus` |
| Retry + fallback | Must | Razão de ser do harness (alternativa ao Claude) |
| Catálogo gratuito + `models refresh` | Should | Há cadastro manual como alternativa |
| Gate de rede corporativa | Should | Só afeta o `gpflow` e skills dependentes |
| OAuth Google (via 2) | Should | Decisão do usuário: implementar, com aviso de risco |
| Limite de concorrência por provedor | Must | Plano B da cota Gemini (R22b) |
| Gemini CLI via ACP (via 3) | Could | Alternativa à via 2 |
| `quota_stats` | Should | Responde à lacuna da cota no piloto |

## Rastreabilidade de Código
| Arquivo | Função / Classe | Cobertura |
|---|---|---|
| `packages/llm/llm/src/index.ts` | `LlmRuntime`, `registerAdapter`, `prepareCall` | 🟢 |
| `packages/llm/llm/src/types.ts`, `assembler.ts`, `assistant-stream.ts` | `StreamChunk`, `BlockAssembler` | 🟢 |
| `packages/llm/llm/src/content.ts` | projeção de arquivos e imagens, offload | 🟢 |
| `packages/llm/llm/src/error.ts`, `call-config.ts` | `LlmFailure`, validação de opções | 🟢 |
| `packages/llm/llm-pi-ai/src/provider.ts` | `PROTOCOLS`, catálogo, descoberta | 🟢 |
| `packages/llm/llm-pi-ai/src/login.ts`, `auth.ts` | OAuth e lock de refresh | 🟢 |
| `packages/llm/llm-deepseek` | rota `deepseek-official` (não portada como padrão) | 🟢 |
| `packages/llm/llm-retry` | política de retry | 🟢 |
| `packages/llm/token-meter/src/estimate.ts` | heurística chars/4 | 🟢 |
| `packages/llm/plugin-package-inventory-deepseek` | **não portado** (ADR 101) | 🟢 |
| — | adapter Gemini, aliases, fallback, política, gate de rede | — (novo) |

## Atualização upstream 0.2.1-rc.5 (ciclo 1, 2026-11-01)
> Origem: [ledger](../upstream/ledger.md). Estas regras **prevalecem** sobre as anteriores quando houver conflito.

- **U1. Rejeitados (ADR 101):** login com conta DeepSeek e catálogo remoto pela conta, a rota oficial DeepSeek seguindo a listagem oficial, as APIs diretas de busca e fetch da DeepSeek e a tradução de raciocínio paga. Nada disso é portado. 🟢
- **U2.** A precedência de credenciais segue a spec 12, U2. 🟢

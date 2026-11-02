# User Story: continuar trabalhando quando o Claude estiver indisponível

**Persona:** Robson, analista do time TOTVS RM (RH e SST). Usa VSCode + Claude Code no dia a dia e assina o Google AI Pro (Gemini).
**Specs relacionadas:** [provedores-modelos](../sdd/provedores-modelos.md) · [agent-loop-sessao](../sdd/agent-loop-sessao.md) · [compactacao-contexto](../sdd/compactacao-contexto.md) · [superficies-tui-headless-acp-sdk](../sdd/superficies-tui-headless-acp-sdk.md) · [memoria-gbrain](../sdd/memoria-gbrain.md)

## História
> **Como** usuário que depende do Claude Code para o trabalho diário,
> **quero** que o `dsh` assuma a mesma tarefa com outro modelo quando o Claude estiver fora do ar, sem cota ou bloqueado,
> **para** não interromper minhas entregas nem perder o contexto do que já foi feito.

## Contexto
- O Claude pode ficar indisponível por vários motivos: falha do serviço, limite de uso da assinatura, bloqueio de rede ou credencial expirada. 🟡
- O `dsh` lê o mesmo projeto: `CLAUDE.md`, agentes, skills, permissões, MCP e memória. 🟡
- Os modelos alternativos são, em ordem:
  - a assinatura Gemini;
  - modelos locais (Ollama);
  - modelos gratuitos, só onde a política do workspace permitir. 🟡

## Cenários

### 1. Troca automática durante a sessão
```gherkin
Dado que estou numa sessão do dsh usando anthropic/claude-opus
E a cadeia de fallback é [anthropic, gemini, ollama]
Quando a Anthropic responde 429 de cota esgotada
Então o dsh troca para gemini/gemini-2.5-pro sem eu precisar fazer nada
E a barra de status mostra o novo modelo com um aviso de fallback
E o evento llm/fallback fica registrado no histórico com o motivo
```

### 2. Começar direto pelo dsh quando o Claude Code nem abre
```gherkin
Dado que o Claude Code está fora do ar
Quando abro o terminal na pasta do projeto e digito dsh
Então a TUI abre com CLAUDE.md, agentes, skills e memória do projeto carregados
E posso retomar com dsh -c a última sessão do dsh nesta pasta
```

### 3. Continuidade da memória
```gherkin
Dado que ontem o Claude Code gravou uma memória em ~/.claude/projects/<pasta>/memory/
Quando hoje abro o dsh na mesma pasta
Então o índice MEMORY.md está no contexto
E o que o dsh aprender hoje fica disponível para o Claude Code quando ele voltar
```

### 4. Modelo menor sem estourar o contexto
```gherkin
Dado uma sessão longa com 300 mil tokens de contexto
Quando o fallback escolhe um modelo com janela de 128 mil tokens
Então o dsh compacta a conversa antes de enviar
E o resumo preserva o pedido, os arquivos tocados, os erros e o próximo passo
```

### 5. Nada a usar
```gherkin
Dado que nenhum provedor da cadeia está disponível ou permitido
Quando envio uma mensagem
Então o turno termina com erro explicando, provedor a provedor, por que cada um foi descartado
E nada do meu histórico se perde
```

### 6. Scripts que chamavam o Claude
```gherkin
Dado um script que roda claude -p "gerar relatório" --output-format stream-json
Quando troco claude por dsh
Então o script continua funcionando com o mesmo formato de saída
```

## Critérios de pronto
- [ ] Fallback automático com aviso visível e evento no log.
- [ ] Mesmo projeto carregado sem nenhuma configuração extra.
- [ ] Memória compartilhada com o Claude Code nos dois sentidos.
- [ ] Compactação antes da troca para janela menor.
- [ ] `dsh -p` compatível com `claude -p`.
- [ ] Nenhum dado enviado à DeepSeek.

## Fora do escopo
- Garantir que um modelo alternativo tenha a mesma qualidade do Claude. A aderência é medida pelo `dsh compat eval` (spec 4, R20).
- Cota do Gemini para 7 agentes simultâneos: será medida no piloto (`quota_stats`). 🔴

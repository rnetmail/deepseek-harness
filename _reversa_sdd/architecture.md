# Arquitetura — deepseek-harness (como é hoje) → harness Rust (alvo)

Documentos relacionados: [c4-context](c4-context.md) · [c4-containers](c4-containers.md) · [c4-components](c4-components.md) · [erd-complete](erd-complete.md) · [spec-impact-matrix](traceability/spec-impact-matrix.md) · [adrs/](adrs/) · [domain](domain.md) · [state-machines](state-machines.md) · [permissions](permissions.md) · [database/](database/) · [totvs-agp-inventory](totvs-agp-inventory.md)

## 1. Resumo
O **dsh** (v0.1.6-alpha.1) é um harness de agentes em TypeScript: cerca de 300 pacotes-plugin sobre um container de DI vendored (cordis), compostos por perfis YAML. A arquitetura é **microkernel + event sourcing**:
- o kernel (agent loop) só orquestra eventos;
- toda capacidade é um plugin trocável (capability seams);
- a sessão é um log append-only do qual cada requisição ao modelo pode ser reconstruída.

O **harness Rust** mantém essas ideias, que estão entre as melhores do projeto. Ele troca a DI dinâmica por traits estáticas, entrega tudo num **binário único** e acrescenta o que o dsh não tem para ser a alternativa ao Claude:
- compatibilidade nativa com `.claude/*`;
- Gemini e o login da assinatura;
- integrações MCP gerenciadas (REA, GBrain, browser e computer use);
- política de dados por workspace;
- TUI.

## 2. Estilo arquitetural 🟢
| Princípio | Hoje | Alvo |
|---|---|---|
| Microkernel + plugins | cordis (DI em runtime, HMR) | Traits + registro no arranque. Extensão por MCP, plugins Claude e QuickJS |
| Event sourcing | Session JSONL v3 zstd | Igual (formato próprio v1 + importador do v3) |
| Requisições reconstruíveis | Sim | Sim |
| Fail-closed | Sandbox, aprovação, auditoria | Igual + gate de rede + `allowed_providers` |
| Superfícies | web, headless, ACP, SDK | **TUI**, `-p`/stream-json, ACP, SDK (web opcional/futura) |

## 3. Integrações externas
| Integração | Protocolo / formato | Direção | Hoje | Alvo |
|---|---|---|---|---|
| Provedores LLM | HTTPS + SSE (openai-chat, openai-responses, anthropic-messages) | saída | 🟢 | + Gemini generateContent |
| OAuth de assinaturas | OAuth2 PKCE / device code | saída | 🟢 (6 provedores) | + Google (Gemini Pro) |
| API DeepSeek (log, inventário) | campos extras na requisição | saída | 🟢 ligado → neutralizado | ❌ removido |
| Telemetria OTLP | HTTPS gzip | saída | 🟢 FEEDBACK_ONLY → desligado | ❌ removido |
| MCP | JSON-RPC stdio / Streamable HTTP | saída | 🟢 opt-in | ● padrão + gerenciados |
| LSP | JSON-RPC stdio | saída | 🟢 opt-in | ● padrão (csharp-ls etc.) |
| ACP | JSON-RPC stdio | entrada (servidor) + saída (cliente) | 🟢 | ● + modes/commands/load |
| SDK | JSON-RPC 2.0 NDJSON | entrada | 🟢 | ● + `-p --output-format` |
| Web fetch/search | HTTPS | saída | 🟢 | ● provedores sem custo |
| Upstream deepseek-harness | releases GitHub/npm | entrada (processo) | — | ● upstream-sync |
| GitHub review webhook | HTTP | entrada | 🟢 overlay | 🟡 futuro |

## 4. Dívidas técnicas e riscos do dsh 🟢
1. **Envio de dados por padrão** (session log, inventário, telemetria): risco de privacidade. Removido no alvo.
2. **Credenciais em YAML legível** pelas ferramentas do agente (admitido no ADR 007). Chaves em texto puro no `dsh.cmd` do usuário, que devem ser **rotacionadas**.
3. **Hooks, MCP, LSP, terminal e FTS fora do `base`**: a experiência padrão fica abaixo da do Claude.
4. **Nomes de ferramenta em minúsculas**, diferentes do Claude Code: matchers de hooks e `allowed-tools` não casam.
5. **Plan mode só consultivo**: não restringe ferramentas.
6. **Sem `.claude/agents`, `.mcp.json` nem `.claude/skills` por padrão**: bloqueia o reaproveitamento do Totvs AGP.
7. **Dependência pesada de runtime**: Node ≥ 22.19, pnpm, cerca de 300 pacotes, TS compilado. O alvo é um binário único.
8. **Loader cordis não transacional**: um patch com falha pode deixar a árvore parcialmente aplicada.
9. **`atomic-write` sem fsync** e **attachments sem retenção** (crescimento ilimitado).
10. **Versão alpha** com APIs em mudança (notas de deprecação frequentes). O porte deve seguir contratos, não código.
11. **Busca web padrão via DeepSeek**: custa um turno inteiro de modelo e depende da chave DeepSeek.

## 5. Decisões-chave do alvo
ADRs 101 (dados só de entrada), 102 (MCP gerenciado), 103 (compat Claude Code nativa), 104 (LSP C#/PL/SQL), mais as decisões registradas em `state.json`: experimental-mandatory, gemini-subscription, totvs-data-gate e lsp-totvs.

## 6. Lacunas 🔴
- Cota real do Gemini Pro para uso multiagente intenso: medir no piloto.
- Hooks do GBrain: rodar `gbrain setup claude-code --dry-run`.
- Termos de uso do OAuth Google fora do Gemini CLI: validar antes de implementar a opção 2 do `gemini-subscription`.

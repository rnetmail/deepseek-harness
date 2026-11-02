# Inventário de reaproveitamento — `D:\Projetos\Totvs AGP\VSCode`

> Levantamento **somente leitura** em 2026-11-01. Nenhum valor de credencial foi lido ou copiado: de `.claude/.env` foram listados só os **nomes** das variáveis, e o header `Authorization` do MCP `gpflow` foi omitido.

## 1. Visão geral 🟢
Workspace do **Time TOTVS RM (RH e SST)**, operado no VSCode + Claude Code. Uma orquestradora (**Cláudia**) coordena 7 subagentes especialistas sobre o TOTVS RM (Oracle), a Ellevo (SQL Server) e o GPFlow (n8n), com **governança rígida** de leitura e escrita.

| Item | Local | Qtde |
|---|---|---|
| Instruções do projeto | `CLAUDE.md` (6,8 KB) | 1 |
| Subagentes | `.claude/agents/*.md` | 7 |
| Memória por agente | `.claude/agent-memory/<agente>/` | 6 |
| Skills do projeto | `.claude/skills/*/SKILL.md` | 24 (uma é pasta-espelho: `skill-ellevo-db`) |
| Catálogo/fontes de skills | `CatalogoSKILLs/` | 8 |
| Permissões | `.claude/settings.json` (19 allow, 3 deny), `.claude/settings.local.json` (24 allow) | — |
| Segredos | `.claude/.env` (RM_DB_*, RM_HML_*, ELLEVO_DB_*) | 10 variáveis |
| MCP | `~/.claude.json` → `gpflow` (**http**, `https://gpflow.grupopereira.link`, header Authorization) | 1 |
| Perfil de navegador | `.claude/navegador-perfis/gpflow` (Chrome dedicado logado) | 1 |
| Base de conhecimento | `conhecimento/{casos,temas,objetos,consultas}` | — |
| Entregas | `entregas/<data>-<demanda>/` + `log-alteracoes.md` | 9+ |

Nível do usuário (`~/.claude`): skills Reversa e `motion-html`. Sem hooks e sem plugins habilitados (`settings.json` só tem permissions, autoUpdatesChannel e theme).

## 2. Subagentes 🟢
| Agente | Papel | tools | model | memory |
|---|---|---|---|---|
| analista-totvs-rm | Negócio, RH/SST, eSocial (leitura) | Read, Grep, Glob, WebSearch, WebFetch, Bash | opus | project |
| analista-rh-senior | Auditoria de RH (leitura) | + Write, Edit | opus | project |
| arquiteto-totvs-rm | Técnico, Oracle, APIs, Jobs (leitura) | Read, Grep, Glob, WebSearch, WebFetch, Bash | opus | project |
| especialista-consultas-relatorios | Sentenças SQL, Cubos, Reports | + Write, Edit | opus | project |
| especialista-formula-visual | Fórmulas Visuais (XOML, C#, SQL) | + Write, Edit | opus | project |
| especialista-metadados | Metadados, telas, tabelas Z | + Write, Edit | opus | project |
| historiador-totvs | Curadoria da base de conhecimento | Read, Grep, Glob, Bash, Write, Edit | opus | project |

## 3. Skills (24) 🟢
| Grupo | Skills | Scripts |
|---|---|---|
| Orquestração | `claudia` (gatilho "Cláudia, …") | — |
| Oracle/RM (leitura) | rm-oracle-consulta (SQL*Plus, TNS `TOTVSRMHOMOL`), rm-sistema, rm-api (~11,5 mil operações REST), rm-pacotes (.TotvsWF/.TotvsMD/.TotvsCube), rh-indicadores-auditoria, checagem-batida-ponto, dw-ponto-consulta | sh, js, sql |
| Conhecimento RM | rm-consultas-relatorios, rm-formula-visual, rm-metadados | — |
| Escrita governada | rm-escrita-nativa (DataServer SaveRecord, só em HML) | sh |
| Ellevo (SQL Server) | ellevo-db, consulta-fila-chamados, kb-chamados | py, sql |
| Base de conhecimento | kb-consulta, kb-coleta, kb-curadoria | py |
| Navegador e GPFlow | navegador (Playwright/CDP em Chrome dedicado), gpflow-validacao, gpflow-credencial, criar-painel-gpflow, painel-tv | py, js, sh |

**Runtimes exigidos pelos scripts:** Bash (Git Bash no Windows), Python 3, Node, SQL*Plus/tnsping (Oracle client), Chrome.

## 4. Recursos do Claude Code em uso → requisitos do harness Rust
| # | Recurso usado | Requisito | Situação no dsh | Prioridade |
|---|---|---|---|---|
| R1 | `CLAUDE.md` na raiz | Carregar como instruções do projeto | ✅ lê | Alta |
| R2 | `.claude/agents/*.md` (frontmatter `name`, `description`, `tools`, `model`, **`memory: project`**) | Subagentes nomeados delegáveis (`subagent_type`), com filtro de ferramentas | ❌ não suporta | **Crítica** |
| R3 | `memory: project` → `.claude/agent-memory/<agente>/` | Memória persistente por agente (ler no início, gravar aprendizados). Pode ser espelhada no GBrain | ❌ | **Crítica** |
| R4 | `model: opus` | **Mapa de modelos** (`opus`/`sonnet`/`haiku` → modelo disponível: Gemini 2.5 Pro, modelo gratuito forte…) configurável | ❌ | **Crítica** |
| R5 | `.claude/skills/*/SKILL.md` + `scripts/` | Descobrir skills em `.claude/skills` (projeto e usuário). Gatilho por descrição e por `/nome` | ⚠️ raízes padrão excluem `.claude` | **Crítica** |
| R6 | `settings.json` → `permissions.allow` com padrões `Bash(cmd:*)`, `WebFetch(domain:x)` | Motor de permissões compatível com a sintaxe do Claude Code | ❌ | Alta |
| R7 | `permissions.deny` → `Read/Edit/Write(./.claude/.env)` | **Bloqueio obrigatório** de leitura e escrita do `.env` (governança + LGPD), aplicado também a Bash (`cat .env`) quando possível | ❌ | **Crítica** |
| R8 | `settings.local.json` | Mesclar local > projeto > usuário | ❌ | Alta |
| R9 | MCP **http** com header `Authorization` (gpflow), escopo usuário | Ler `~/.claude.json` (`mcpServers` e `projects[*].mcpServers`) e `.mcp.json`. Header vindo do cofre do SO | ⚠️ cliente existe, mas não lê `.claude.json` | Alta |
| R10 | Orquestração "a sessão principal orquestra, subagentes não chamam subagentes" + chamadas paralelas | Subagentes com profundidade 1 por padrão e delegação paralela | ✅ (maxDepth configurável) | Alta |
| R11 | Navegador Playwright/CDP com perfil dedicado | Browser use obrigatório (Playwright/Chrome DevTools MCP) **e** permissão para as skills Python usarem o CDP | ⚠️ experimental | Alta |
| R12 | Bash (Git Bash) no Windows | Ferramenta `Bash` usando Git Bash, mais `PowerShell` | ✅ bash e pwsh | Alta |
| R13 | Gatilho de persona por prefixo ("Cláudia, …") | Funciona via CLAUDE.md + skill `claudia` (sem recurso especial) | ✅ | Média |
| R14 | WebSearch/WebFetch por domínio | Provedor de busca sem custo (ver módulo `web`) | ⚠️ busca via DeepSeek | Média |
| R15 | Datas de governança e liberações no CLAUDE.md | O modelo de fallback precisa **seguir as regras com rigor**. Avaliar a aderência de cada modelo gratuito antes de liberar a escrita em HML | — | **Crítica** |

## 5. Riscos específicos do fallback 🟡
1. **Modelos gratuitos e dados sensíveis:** o time lida com dados de RH (LGPD). Provedores gratuitos podem **usar prompts para treinamento**. → O harness deve marcar provedores com `data_policy` e **restringir** este workspace aos provedores sem retenção ou treinamento (assinatura Gemini, modelos locais via Ollama, provedores pagos com DPA). Regra configurável por workspace: `allowed_providers`.
2. **Escrita em HML** por um modelo mais fraco: manter as aprovações. Sugestão: no fallback, a liberação permanente da regra 4 só vale para modelos de uma lista permitida.
3. **`.env` com credenciais de banco:** o `deny` de R7 é inegociável. O harness deve impedir também a exfiltração via Bash (`cat`, `type`, `Get-Content` do caminho negado).
4. **MCP gpflow com token no `~/.claude.json`:** ao importar, mover o token para o cofre do SO.

## 6. Lacunas 🔴
- O Gemini Pro (assinatura do usuário) cobre a cota necessária para 7 agentes em `opus`? Validar limites do plano.
- O conteúdo de `CatalogoSKILLs/` é a fonte das skills de `.claude/skills/` ou uma versão diferente? Definir qual é a origem canônica.

# User Story: trabalhar no Totvs AGP pelo dsh com governança e LGPD

**Persona:** Robson, do Time TOTVS RM (RH e SST). Trabalha com a orquestradora Cláudia e 7 agentes especialistas sobre o TOTVS RM (Oracle), a Ellevo (SQL Server) e o GPFlow (n8n), com dados de RH sujeitos à LGPD.
**Specs relacionadas:** [compat-claude-code-totvs](../sdd/compat-claude-code-totvs.md) · [provedores-modelos](../sdd/provedores-modelos.md) · [seguranca-sandbox-permissoes](../sdd/seguranca-sandbox-permissoes.md) · [subagentes-skills-hooks](../sdd/subagentes-skills-hooks.md) · [memoria-gbrain](../sdd/memoria-gbrain.md) · [lsp-csharp-plsql](../sdd/lsp-csharp-plsql.md) · [browser-computer-agent-teams](../sdd/browser-computer-agent-teams.md)

## História
> **Como** analista que usa a Cláudia e os agentes do Totvs AGP,
> **quero** abrir `D:\Projetos\Totvs AGP\VSCode` no dsh e trabalhar exatamente como no Claude Code,
> **para** manter as entregas quando o Claude estiver indisponível, sem violar a governança do time nem expor dados de RH.

## Regras que não podem ser quebradas
1. O `.claude/.env` (credenciais de banco) nunca é lido, nem por ferramenta de arquivo, nem por comando de shell. 🟢 (deny existente)
2. Os dados do workspace só vão para provedores sem treinamento e sem retenção, ou para modelos locais. Eu libero o uso caso a caso. 🟢 (decisão `totvs-data-gate`)
3. Recursos que dependem da rede da empresa (`gpflow` e afins) só funcionam com o gate de rede aberto. 🟢
4. Escritas em HML com um modelo fora da lista aprovada pedem minha aprovação em toda chamada. 🟡
5. A memória dos agentes e o perfil logado do Chrome não saem do workspace. 🟡

## Cenários

### 1. Abrir o workspace
```gherkin
Dado que abro o dsh em D:\Projetos\Totvs AGP\VSCode
Quando a sessão inicia
Então CLAUDE.md, 7 agentes, as skills de .claude/skills, as permissões mescladas e o MCP gpflow são carregados
E a barra de status mostra a política do workspace (no-training/local) e o estado do gate de rede
```

### 2. Liberação caso a caso
```gherkin
Dado data_gate = ask no workspace
Quando envio a primeira mensagem da sessão
Então o dsh pergunta se pode usar o conteúdo do workspace com o modelo escolhido
E, se eu recusar, só modelos locais podem ser usados nesta sessão
E a decisão fica registrada no histórico
```

### 3. Delegação pela Cláudia
```gherkin
Dado que escrevo "Cláudia, quais eventos do eSocial são afetados pela mudança de lotação?"
Quando a Cláudia delega em paralelo ao analista-totvs-rm e ao arquiteto-totvs-rm
Então cada agente roda com as suas ferramentas, a sua memória e o alias opus resolvido para um modelo permitido
E nenhum deles consegue delegar a outro agente
```

### 4. Consulta ao Oracle de homologação
```gherkin
Dado que estou na rede da empresa
Quando o agente usa a skill rm-oracle-consulta (sqlplus em TOTVSRMHOMOL)
Então o comando roda conforme as regras allow de Bash do projeto
E o script lê as credenciais do .env por conta própria, sem que os valores passem pelo modelo
```

### 5. Fora da rede da empresa
```gherkin
Dado que estou em casa, sem VPN
Quando abro o workspace
Então o gpflow e as skills gpflow-* aparecem como "requer rede da empresa"
E consigo trabalhar normalmente em análise de documentação, SQL local e base de conhecimento
```

### 6. Tentativa de ler o .env
```gherkin
Dado qualquer modelo
Quando ele tenta "Get-Content .claude/.env" ou Read(.claude/.env)
Então a chamada é bloqueada e eu vejo qual regra bloqueou
```

### 7. Edição de Fórmula Visual com verificação
```gherkin
Dado o csharp-ls instalado
Quando o especialista-formula-visual edita um codXxx.cs da entrega
Então os erros de compilação aparecem no resultado da edição antes da entrega
E o gpflow/HML não é tocado sem minha aprovação se o modelo não estiver aprovado para escrita
```

### 8. Navegador do GPFlow
```gherkin
Dado a skill navegador com o Chrome dedicado do perfil gpflow
Quando o agente precisa validar um painel
Então o script Python da skill funciona como no Claude Code
E qualquer conexão do dsh a esse Chrome pede minha confirmação
```

## Critérios de pronto
- [ ] `dsh compat doctor` no workspace lista tudo sem expor segredos.
- [ ] Gate de dados e gate de rede funcionando, com as decisões no log.
- [ ] Os 7 agentes delegáveis, com aliases e memória.
- [ ] Bloqueio do `.env` por arquivo e por shell.
- [ ] Aprovação obrigatória para escrita com modelo não aprovado.
- [ ] Diagnósticos de C# e PL/SQL nas edições.

## Pendências 🔴
- Definir com o time a bateria do `dsh compat eval` e a lista `write_approved_models`.
- Medir no piloto se a cota do Gemini Pro sustenta os 7 agentes.
- Confirmar com o jurídico o uso do REA em binários do TOTVS RM.

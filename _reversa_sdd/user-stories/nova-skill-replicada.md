# User Story: criar uma skill no VSCode e tê-la no dsh e no CatalogoSKILLs sem trabalho manual

**Persona:** Robson, que cria e ajusta skills e agentes no VSCode com o Claude Code, no workspace `D:\Projetos\Totvs AGP\VSCode`.
**Specs relacionadas:** [catalog-sync](../sdd/catalog-sync.md) · [compat-claude-code-totvs](../sdd/compat-claude-code-totvs.md) · [provedores-modelos](../sdd/provedores-modelos.md)

## História
> **Como** autor das skills e dos agentes do time,
> **quero** que tudo o que eu criar ou alterar em `.claude/` no VSCode apareça no dsh automaticamente, e que o `CatalogoSKILLs` (pastas, `.zip` e `.7z`) seja gerado sozinho,
> **para** não manter cópias à mão nem correr o risco de o catálogo ficar desatualizado ou levar segredos.

## Cenários

### 1. Skill nova dentro do workspace
```gherkin
Dado o dsh fechado
Quando crio .claude/skills/conferencia-ferias/SKILL.md no VSCode
E depois abro o dsh no Totvs AGP
Então /conferencia-ferias já está disponível, sem cópia e sem comando extra
```

### 2. Alteração com o dsh aberto
```gherkin
Dado uma sessão do dsh aberta no Totvs AGP
Quando edito o SKILL.md de rm-api no VSCode
Então em poucos segundos o próximo passo do agente já usa a versão nova
```

### 3. Skill liberada para outros projetos
```gherkin
Dado rm-api na lista global de ~/.dsh/sync.toml
Quando a sincronização roda
Então totvs-agp:rm-api fica disponível no dsh em qualquer pasta
E a política de dados do workspace em que eu estiver continua valendo
```

### 4. Catálogo gerado
```gherkin
Dado auto_build = true e rm-api incluída no catálogo
Quando altero rm-api
Então CatalogoSKILLs/rm-api, CatalogoSKILLs.zip e CatalogoSKILLs.7z são regenerados
E gerar de novo sem mudanças produz arquivos com o mesmo hash
E o CatalogoSKILLs.rar antigo não é apagado, só listado como "não gerenciado"
```

### 5. Segredo esquecido num script
```gherkin
Dado uma skill do catálogo cujo script ficou com uma senha escrita
Quando a sincronização roda
Então a skill fica bloqueada, não entra no catálogo nem nos pacotes
E dsh sync status mostra arquivo e linha, sem mostrar a senha
```

### 6. O que nunca sai do workspace
```gherkin
Dado qualquer sincronização ou geração de catálogo
Quando ela termina
Então .claude/.env, navegador-perfis e agent-memory não existem em ~/.dsh/synced nem no CatalogoSKILLs
```

### 7. Remoção
```gherkin
Dado uma skill apagada no VSCode
Quando a sincronização roda
Então ela some do dsh e vai para a lixeira por 30 dias
```

## Critérios de pronto
- [ ] Leitura ao vivo de `.claude/*` no workspace.
- [ ] Reconciliação no arranque e watcher com o dsh aberto.
- [ ] `dsh catalog build` reprodutível, com `.zip` e `.7z`.
- [ ] Varredura de segredos e exclusões obrigatórias.
- [ ] `dsh sync status` e `dsh sync log` para conferência.

## Fora do escopo
- Levar mudanças do dsh de volta ao VSCode: a sincronização é de mão única.
- Gerar `.rar`, que foi substituído por `.7z`.

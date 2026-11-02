# Sincronização VSCode → dsh e CatalogoSKILLs gerado (`harness-claude-compat::sync`)

## Visão Geral
Hoje o `CatalogoSKILLs/` do Totvs AGP é mantido à mão. Ele contém 8 das 24 skills de `.claude/skills/` (cópias idênticas, conferidas por diff) mais os pacotes `CatalogoSKILLs.zip`, `CatalogoSKILLs.rar`, `painel-tv.zip` e `painel-tv-skill.zip`. 🟢

Esta spec define dois mecanismos, ambos de **sentido único**: o VSCode é a fonte e o dsh e o catálogo são derivados. 🟡 (decisão `catalog-sync`)
1. **Replicação automática:** tudo o que for criado ou alterado no VSCode (skills, agents, commands, `CLAUDE.md`) chega ao dsh sem nenhuma ação manual.
2. **Catálogo gerado:** `CatalogoSKILLs/` e seus `.zip` deixam de ser editados à mão e passam a ser produzidos a partir de `.claude/skills`.

## Responsabilidades
- Manter um registro de **fontes** (workspaces VSCode/Claude Code) a acompanhar. 🟡
- Detectar mudanças por watcher (com o dsh aberto) e por reconciliação no arranque (com o dsh fechado). 🟡
- Comparar por manifesto de hash SHA-256 e aplicar apenas as diferenças. 🟡
- Publicar os itens no dsh com o escopo correto (só no workspace ou em qualquer lugar), respeitando o gate de dados. 🟡
- Gerar `CatalogoSKILLs/` e os pacotes `.zip` de forma reprodutível. 🟡
- Bloquear segredos, perfis de navegador e dados de RH. 🟡
- Registrar cada sincronização num log auditável. 🟡

## Interface

### Configuração (`~/.dsh/sync.toml`)
```toml
[[source]]
id       = "totvs-agp"
path     = "D:/Projetos/Totvs AGP/VSCode"
kinds    = ["skills", "agents", "commands", "claude_md"]   # o que replicar
scope    = "workspace"        # padrão para itens desta fonte
data_gate = true              # itens herdam o totvs-data-gate (spec 2 R12)

  [source.global]             # itens liberados para uso fora do workspace
  skills = ["claudia", "navegador", "rm-api", "rm-consultas-relatorios",
            "rm-formula-visual", "rm-metadados", "rm-pacotes", "rm-sistema"]

  [source.catalog]            # geração do CatalogoSKILLs
  enabled    = true
  output     = "CatalogoSKILLs"
  skills     = "@global"      # mesma lista de [source.global].skills; ou lista própria
  zip        = "CatalogoSKILLs.zip"
  sevenzip   = "CatalogoSKILLs.7z"    # substitui o antigo .rar
  per_skill_zip = { "painel-tv" = "painel-tv.zip" }
  auto_build = true           # recria quando uma skill incluída mudar
```
A lista inicial de `global.skills` e de `catalog.skills` reproduz as 8 skills que hoje estão no catálogo. 🟢 (diff de 2026-11-01)

### Escopos
| Escopo | Onde o item fica disponível | Como o dsh o carrega |
|---|---|---|
| `workspace` | Só em sessões cujo cwd está dentro de `source.path` | Leitura direta de `.claude/*` (spec 4), **sem cópia** |
| `global` | Em qualquer workspace | Cópia em `~/.dsh/synced/<source-id>/{skills,agents,commands}/` com o namespace `<source-id>:` |

### Comandos
| Comando | Efeito |
|---|---|
| `dsh sync add <caminho> [--id X]` | Registra uma fonte e faz a primeira reconciliação (simulada, pedindo confirmação) |
| `dsh sync status` | Mostra, por fonte, os itens novos, alterados, removidos, bloqueados e em conflito |
| `dsh sync run [--dry-run] [--source X]` | Reconciliação manual |
| `dsh catalog build [--source X] [--check]` | Gera o catálogo; `--check` só diz se está desatualizado (útil em CI) |
| `dsh sync log` | Histórico das sincronizações |

### Manifesto (`~/.dsh/synced/<source-id>/manifest.json`)
```json
{ "source": "totvs-agp", "schema": 1, "synced_at": "2026-11-01T12:00:00Z",
  "items": { "skills/rm-api": { "kind": "skill", "scope": "global",
             "files": { "SKILL.md": "sha256:…", "scripts/x.js": "sha256:…" },
             "tree_hash": "sha256:…", "state": "active" } } }
```
`state` pode ser `active`, `disabled` (removido na fonte), `blocked` (bloqueado por segredo) ou `conflict`.

## Regras de Negócio

### Direção e fonte
- **R1.** A **fonte canônica** é o `.claude/*` do workspace VSCode. Nada flui do dsh para o VSCode, com uma exceção: a pasta de saída do catálogo (R14). 🟡
- **R2.** Itens de escopo `workspace` não são copiados: o dsh os lê ao vivo (spec 4). A replicação só copia itens `global`. Isso evita duplicar dados de RH fora do workspace. 🟡
- **R3.** `CatalogoSKILLs/` **nunca** é lido como fonte de skills. Se alguém editar uma cópia dentro do catálogo, o próximo `catalog build` sobrescreve a edição e avisa. 🟢 (catálogo = cópia) / 🟡 (regra)

### Detecção e aplicação
- **R4.** Com o dsh em execução, um watcher (`notify`) observa as pastas da fonte, com *debounce* de 2 s. Com o dsh fechado, a reconciliação roda a cada arranque e quando `dsh sync run` é chamado. 🟡
- **R5.** A comparação é por `tree_hash` (SHA-256 de cada arquivo, em ordem, mais os caminhos relativos). O mtime é só um atalho para evitar recalcular hashes. 🟡
- **R6.** A aplicação é atômica por item: o dsh escreve em `…/.staging/<item>` e troca por rename. Um item nunca fica meio copiado. 🟡
- **R7.** Um item **removido** na fonte vira `disabled`: sai do catálogo de skills e agents do dsh, e os arquivos vão para `~/.dsh/synced/.trash/` por 30 dias. 🟡
- **R8.** Um item **renomeado** é tratado como remoção mais criação. A ligação pelo `tree_hash` igual é registrada no log para auditoria. 🟡
- **R9.** Se a cópia global no dsh foi alterada à mão (o hash difere do manifesto), o item vira `conflict` e não é sobrescrito. `dsh sync status` mostra o diff, e `dsh sync run --accept-source` resolve a favor do VSCode. 🟡
- **R10.** Uma mudança aplicada durante uma sessão aberta gera um catálogo substituto no próximo passo (mesma regra das skills, spec 4). 🟢

### Proteção de dados
- **R11. Nunca replicados nem empacotados:**
  - `.claude/.env`, `*.env`, `.claude/settings.local.json`;
  - `.claude/navegador-perfis/**` (perfil do Chrome com cookies e sessão);
  - `.claude/agent-memory/**` (pode conter dados de RH);
  - `__pycache__/`, `*.pyc`, `node_modules/`;
  - arquivos acima de 10 MB. 🟡
- **R12. Varredura de segredos:**
  - antes de publicar um item `global` ou empacotá-lo, o dsh procura padrões de segredo: chaves de API, `password=`, strings de conexão Oracle/SQL Server com senha, tokens `Bearer`, chaves privadas PEM;
  - se achar, o item vira `blocked` e o relatório mostra o arquivo e a linha, **sem o valor**. 🟡
- **R13.** Itens de uma fonte com `data_gate = true` levam o atributo `data_gate` para o dsh. Uma skill `global` vinda do Totvs, usada fora do workspace, continua sujeita à política do workspace em que está rodando e não carrega consigo nenhum dado. 🟡

### Catálogo gerado
- **R14.** `dsh catalog build`:
  1. copia as skills listadas para `CatalogoSKILLs/<nome>/`, aplicando R11 e R12;
  2. remove dali as skills que saíram da lista;
  3. gera `CatalogoSKILLs.zip`, `CatalogoSKILLs.7z` e os zips por skill.

  Tudo é feito em staging e trocado por rename. É a **única escrita** do dsh dentro do workspace da fonte, e só acontece dentro de `catalog.output`. 🟡
- **R15.** Os zips são **reprodutíveis**: entradas em ordem lexicográfica, mtime fixo (o do commit ou o do arquivo mais novo), sem metadados do SO e com separadores `/`. O mesmo conteúdo produz o mesmo hash. 🟡
- **R16.** Com `auto_build = true`, a reconstrução acontece depois de cada sincronização que alterar uma skill incluída. Sem isso, `dsh catalog build --check` informa se o catálogo ficou desatualizado. 🟡
- **R17.** O antigo `CatalogoSKILLs.rar` é **substituído por `CatalogoSKILLs.7z`**, gerado pelo próprio dsh com o crate `sevenz-rust2` (LZMA2), sem depender de ferramenta externa. A ordem das entradas e os mtimes seguem R15, e o hash é estável para o mesmo conteúdo. O `.rar` existente vira "não gerenciado" (R18) e só é apagado pelo usuário. 🟢 (decisão do usuário, 2026-11-01)
- **R18.** Um arquivo que existe em `CatalogoSKILLs/` sem correspondência em `.claude/skills` (por exemplo, um zip antigo) **não é apagado**. Ele é listado como "não gerenciado" em `catalog build`. 🟡

### Auditoria
- **R19.** Cada execução grava uma linha em `~/.dsh/synced/sync-log.jsonl`: `{at, source, trigger: watch|startup|manual, added[], updated[], disabled[], blocked[], conflicts[], catalog_built}`. O log não contém conteúdo de arquivos. 🟡

## Fluxo Principal
1. O usuário cria `.claude/skills/nova-skill/SKILL.md` no VSCode (pelo Claude Code ou à mão). 🟡
2. **Com o dsh aberto:** o watcher dispara depois de 2 s. **Com o dsh fechado:** a próxima abertura reconcilia. 🟡
3. O dsh calcula o `tree_hash`, compara com o manifesto e classifica o item como `added`. 🟡
4. Escopo `workspace` (padrão): nada é copiado, e a skill já aparece em sessões dentro do Totvs AGP (spec 4). 🟡
5. Se `nova-skill` estiver em `global.skills`: varredura (R11, R12), cópia atômica para `~/.dsh/synced/totvs-agp/skills/nova-skill/` e registro como `totvs-agp:nova-skill`. 🟡
6. Se estiver em `catalog.skills` e `auto_build = true`: o dsh regenera `CatalogoSKILLs/` e os zips. 🟡
7. Grava a linha do log. A TUI mostra uma notificação discreta ("1 skill sincronizada de totvs-agp"). 🟡

## Fluxos Alternativos
- **Segredo detectado:** o item fica `blocked`, o catálogo não o inclui e `dsh sync status` aponta o arquivo e a linha. 🟡
- **Pasta da fonte inacessível** (disco desconectado, VPN): a fonte é marcada como `offline`, os itens já sincronizados continuam valendo e não há erro fatal. 🟡
- **Edição manual de uma cópia global:** `conflict` (R9). 🟡
- **Skill removida da lista `global`:** a cópia global fica `disabled`; ela continua disponível dentro do workspace. 🟡
- **Dois itens com o mesmo nome em fontes diferentes:** não colidem, porque o namespace `<source-id>:` os separa. 🟡

## Dependências
- Compatibilidade com o Claude Code (spec 4): leitura ao vivo do escopo `workspace` e formato de skills e agents.
- Política de dados (spec 2): atributo `data_gate`.
- Crates: `notify` + `notify-debouncer-full`, `sha2`, `walkdir`, `globset`, `zip` (escrita determinística), `sevenz-rust2` (.7z), `regex` (varredura de segredos).

## Requisitos Não Funcionais
| Tipo | Requisito inferido | Evidência no código | Confiança |
|---|---|---|---|
| Consistência | Aplicação atômica por item (staging + rename) | padrão `atomic-write` do dsh (`packages/util`) | 🟡 |
| Reprodutibilidade | Zips determinísticos | padrão de build reproduzível das ferramentas CoreOS | 🟡 |
| Segurança | Varredura de segredos e exclusão de perfis e memória | inventário Totvs, riscos 3 e 4 | 🟡 |
| Desempenho | Debounce de 2 s; hash só quando o mtime mudar | — (novo) | 🟡 |
| Disponibilidade | Fonte offline não interrompe o dsh | — (novo) | 🟡 |

> Novo componente. Não há código de origem no dsh TS. Validar com o usuário.

## Critérios de Aceitação
```gherkin
Dado a fonte totvs-agp registrada e o dsh fechado
Quando o usuário cria .claude/skills/nova-skill no VSCode e depois abre o dsh no Totvs AGP
Então /nova-skill está disponível sem nenhuma ação manual

Dado rm-api em global.skills e catalog.skills e auto_build = true
Quando o SKILL.md de rm-api é alterado no VSCode com o dsh aberto
Então em até 5 s ~/.dsh/synced/totvs-agp/skills/rm-api reflete a mudança
E CatalogoSKILLs/rm-api, CatalogoSKILLs.zip e CatalogoSKILLs.7z são regenerados
E rodar catalog build de novo produz um zip com o mesmo SHA-256

Dado uma skill global cujo script contém "password=Abc123"
Quando a sincronização roda
Então o item fica blocked, não entra no catálogo e o relatório mostra arquivo e linha sem o valor

Dado uma skill removida de .claude/skills
Quando a sincronização roda
Então ela some do catálogo de skills do dsh e seus arquivos vão para .trash por 30 dias

Dado qualquer sincronização
Quando ela termina
Então .claude/.env, navegador-perfis e agent-memory não existem em ~/.dsh/synced nem no CatalogoSKILLs
```

## Prioridade
| Requisito | MoSCoW | Justificativa |
|---|---|---|
| Leitura ao vivo do escopo `workspace` | Must | Atende "criou no VSCode, está no dsh" sem copiar dados |
| Reconciliação no arranque + watcher | Must | Garante a automação pedida pelo usuário |
| Exclusões e varredura de segredos | Must | LGPD e governança |
| `catalog build` reprodutível | Must | Automação do CatalogoSKILLs pedida pelo usuário |
| Replicação `global` com namespace | Should | Uso das skills fora do workspace |
| Conflitos, lixeira e log | Should | Segurança operacional |
| Zips por skill | Could | Hoje só o `painel-tv` tem |
| `.7z` no lugar do `.rar` | Should | Decisão do usuário; mesmo conteúdo do `.zip` |

## Rastreabilidade de Código
| Arquivo | Função / Classe | Cobertura |
|---|---|---|
| `D:/Projetos/Totvs AGP/VSCode/CatalogoSKILLs/` | 8 skills + 4 pacotes (estado atual) | 🟢 |
| `D:/Projetos/Totvs AGP/VSCode/.claude/skills/` | 24 skills (fonte canônica) | 🟢 |
| `packages/skill/skill-filesystem` | watch de raízes de skill (base do watcher) | 🟢 |
| `_reversa_sdd/totvs-agp-inventory.md` | riscos e lacunas | 🟢 |
| — | sync, manifesto, catalog build | — (novo) |

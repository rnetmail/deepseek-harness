# Navegação semântica: LSP para C# e índice de símbolos para PL/SQL (`harness-tools::lsp` + `harness-integrations`)

## Visão Geral
O modelo ganha navegação de código além da busca textual: ir para a definição, achar referências, ver a assinatura e receber os erros de compilação logo depois de editar. O Totvs AGP usa **PL/SQL** e **C#** e não tem servidor de linguagem configurado. 🟢

Levantamento de 2026-11-01:
- 181 arquivos `.sql`, 29 deles com `CREATE OR REPLACE`;
- 10 arquivos `.cs`, que são trechos de `CodeActivity` extraídos de Fórmulas Visuais;
- 4 `.xoml` (workflow do RM 12.1.2602, com 5 `CodeActivity` em C#).

A solução segue a decisão `lsp-totvs` (ADR 104) e tem dois caminhos:
- **C#:** o `csharp-ls`, instalado pelo dsh a partir do .NET SDK detectado na máquina. 🟡
- **PL/SQL:** não há servidor de linguagem maduro. O dsh usa um **índice de símbolos próprio** (tree-sitter + busca textual), exposto pela mesma ferramenta `LSP`. 🟡

O dsh TS tem o cliente LSP (`lsp`, `lsp-stdio`, `tool-lsp`), mas ele vem fora do `base`, sem servidores instalados e sem diagnósticos. 🟢

## Responsabilidades
- Expor ao modelo uma única ferramenta `LSP`, com as mesmas operações do Claude Code. 🟡
- Detectar o .NET e oferecer, instalar e atualizar o `csharp-ls`. 🟢 (decisão)
- Iniciar servidores de linguagem sob demanda, um por linguagem e workspace, com ambiente sem credenciais. 🟢
- Montar um projeto auxiliar para que trechos `.cs` soltos tenham contexto de compilação. 🟡
- Indexar objetos PL/SQL e responder definição, referências, símbolos e erros de sintaxe. 🟡
- Entregar diagnósticos automaticamente depois de `Edit`/`Write` em arquivos cobertos. 🟡

## Interface

### Ferramenta `LSP` 🟢/🟡
```
LSP { operation, file_path, line, character, query? }
```
| Operação | dsh TS | dsh Rust: C# (csharp-ls) | dsh Rust: PL/SQL (índice) |
|---|---|---|---|
| `goToDefinition` | 🟢 | ✔ | ✔ (pacote, procedure, function, trigger, view, type, cursor, variável local) |
| `findReferences` | 🟢 | ✔ | ✔ aproximado (por nome qualificado e escopo) |
| `goToImplementation` | 🟢 | ✔ | ✔ spec de pacote → body |
| `hover` | 🟢 | ✔ | ✔ assinatura e parâmetros |
| `documentSymbol` | ❌ excluído | ✔ | ✔ |
| `workspaceSymbol` | ❌ excluído | ✔ | ✔ |
| `prepareCallHierarchy` / `incomingCalls` / `outgoingCalls` | ❌ | ✔ | ✔ aproximado |
| `diagnostics` | ❌ excluído | ✔ erros de compilação | ✔ só erros de sintaxe |

- Coordenadas começam em 1 (UTF-16).
- O resultado é `caminho:linha:coluna`, agrupado por arquivo, com até 100 locais e 16.000 caracteres.
- Timeout de 60 s. 🟢

### Configuração (`~/.dsh/lsp.toml` + `.dsh/workspace.toml`)
```toml
[servers.csharp]
command   = "csharp-ls"                 # instalado via dotnet tool
extensions = { ".cs" = "csharp", ".csx" = "csharp" }
min_dotnet = "auto"                     # derivado da matriz csharp-ls × .NET (R8)

[servers.rust]       # e typescript, python, go: pré-configurados e ativados se o binário existir
command = "rust-analyzer"

[plsql]
extensions = [".sql", ".pks", ".pkb", ".pck", ".prc", ".fnc", ".trg", ".vw"]
dialect    = "oracle"
```
```toml
# .dsh/workspace.toml do Totvs AGP
[lsp.csharp]
reference_dirs = []          # opcional: pasta com as DLLs do RM (RM.Lib.*.dll) para resolver tipos
loose_files    = "scratch-project"   # monta projeto auxiliar para .cs soltos

[lsp.plsql]
catalog_enrich = "off"       # "on" consulta ALL_SOURCE/ALL_ARGUMENTS pela skill rm-oracle-consulta (exige gate de rede)
```

## Regras de Negócio

### Geral
- **R1.** A ferramenta `LSP` é somente leitura. Rename, formatting e code actions ficam fora do escopo (o dsh TS também os exclui). 🟢
- **R2.** Os servidores sobem **sob demanda**, na primeira consulta por linguagem e workspace, e encerram depois de 10 min ociosos. 🟢/🟡
- **R3.** Cada consulta sincroniza o conteúdo atual do arquivo (`didOpen`/`didChange`). Em cada servidor e workspace, as consultas rodam em série. 🟢
- **R4.** O ambiente dos servidores **não** recebe variáveis com `KEY`, `PASSWORD`, `SECRET` ou `TOKEN`, nem `DSH_*`. Os servidores não leem o `.claude/.env`. 🟢
- **R5.** O prompt orienta o modelo a preferir `Grep`/`Read` e usar `LSP` quando a busca textual for ambígua ou quando precisar de tipos e assinaturas. 🟢
- **R6. Diagnósticos automáticos:**
  - depois de um `Edit`/`Write` bem-sucedido num arquivo coberto, o dsh pede os diagnósticos (com espera de até 3 s);
  - os erros e avisos novos são anexados ao resultado da ferramenta, com até 20 itens;
  - os anteriores à edição não são repetidos. 🟡 (equivale aos diagnósticos que o Claude Code recebe do IDE)

### C#
- **R7. Detecção:** `dotnet --list-sdks`. Na máquina atual foi encontrado o **.NET SDK 7.0.202** (`C:\Program Files\dotnet\sdk`). 🟢
- **R8. Instalação:** `dotnet tool install -g csharp-ls`, atualizada por `dotnet tool update -g csharp-ls` (spec 7). As versões recentes do `csharp-ls` exigem um SDK .NET mais novo que o 7. Se o SDK detectado estiver abaixo do mínimo, o dsh:
  - oferece instalar o .NET 8 LTS ou superior (`winget install Microsoft.DotNet.SDK.8`), que convive com o 7; **ou**
  - instala a última versão do `csharp-ls` compatível com o SDK presente (`--version X`). 🟢

  Matriz verificada nos pacotes NuGet (pasta `tools/netX`) em 2026-11-01 [Revisão: era 🔴]:

  | `csharp-ls` | .NET exigido |
  |---|---|
  | ≤ 0.10.0 | net7.0 |
  | 0.11.0 – 0.16.0 | net8.0 |
  | 0.17.0 – 0.20.0 | net9.0 |
  | 0.21.0 – 0.28.0 (atual) | net10.0 |

  Com o SDK 7.0.202 da máquina, a versão compatível é a **0.10.0**, antiga. A recomendação é instalar o **.NET 10** (`winget install Microsoft.DotNet.SDK.10`), que convive com o 7, e usar o `csharp-ls` atual. O dsh deriva o mínimo da tabela, e não de um valor fixo.
- **R9. Arquivos `.cs` soltos** (trechos de `CodeActivity` da Fórmula Visual, sem `.csproj`):
  - o dsh gera um projeto auxiliar em `.dsh/lsp/csharp/<pasta-hash>/Scratch.csproj`, que inclui os `.cs` da pasta e, se configuradas, referências às DLLs do RM (`reference_dirs`);
  - o projeto fica **fora** das pastas de entrega e não altera nenhum arquivo do projeto. 🟡
- **R10.** Sem `reference_dirs`, os tipos `RM.Lib.*` não são resolvidos. Os diagnósticos de "tipo ou namespace não encontrado" para `RM.*` são filtrados, para não poluir o contexto, e o resultado avisa que a resolução é parcial. 🟡
- **R11.** O C# embutido em `.xoml` (`CodeActivity`) **não** é analisado diretamente. Ele só é coberto quando extraído para `.cs`, como já é feito no fluxo de entrega do time. A extração automática de XOML para documento virtual fica como evolução. 🟡

### PL/SQL
- **R12. Índice:**
  - os arquivos com as extensões de `[plsql]` são analisados com tree-sitter (gramática SQL com extensões Oracle/PL-SQL);
  - o índice grava os objetos (`PACKAGE`, `PACKAGE BODY`, `PROCEDURE`, `FUNCTION`, `TRIGGER`, `VIEW`, `TYPE`), os subprogramas internos, os parâmetros, os cursores e as variáveis de pacote, com posição;
  - armazenamento em `rusqlite` (`~/.dsh/index/plsql/<ws-hash>.db`), atualizado por hash de arquivo. 🟡
- **R13.** Quando o tree-sitter não consegue interpretar um trecho (sintaxe Oracle exótica, SQL dinâmico), o índice usa um analisador de fallback por expressões regulares para cabeçalhos `CREATE [OR REPLACE] … <tipo> <nome>` e `PROCEDURE|FUNCTION <nome>`, com a confiança marcada como "textual" no resultado. 🟡
- **R14. Resolução de nomes:** `pacote.subprograma`, `schema.objeto` e nomes não qualificados (procurados primeiro no mesmo pacote, depois no workspace). Uma ambiguidade devolve todos os candidatos. 🟡
- **R15. Diagnósticos PL/SQL:** só erros de sintaxe do parser (nós `ERROR`/`MISSING`). Não há validação semântica contra o banco. 🟡
- **R16. Enriquecimento pelo catálogo Oracle** (opcional, padrão desligado):
  - com `catalog_enrich = "on"`, o gate de rede aberto e a liberação do gate de dados, o dsh pode consultar `ALL_SOURCE` e `ALL_ARGUMENTS` pela skill `rm-oracle-consulta` (somente leitura, HML) para completar assinaturas de objetos que não estão em arquivo;
  - nunca grava no banco;
  - o resultado é marcado como "catálogo HML". 🟡

## Fluxo Principal

### C#
1. O modelo edita `fonte/codPreparar.cs` com `Edit`. 🟡
2. O dsh vê que `.cs` está mapeado para `csharp`. Se o servidor não está rodando, ele gera o projeto auxiliar (R9) e inicia o `csharp-ls`. 🟡
3. Sincroniza o arquivo e pede os diagnósticos. Os erros novos (por exemplo, `CS1002: ; esperado`) são anexados ao resultado do `Edit`. 🟡
4. O modelo corrige antes de entregar. 🟡

### PL/SQL
1. O modelo chama `LSP {operation: "goToDefinition", file_path: "sql/jobs.sql", line: 42, character: 15}` sobre `PKG_PONTO.RECALCULA`. 🟡
2. O índice resolve `PKG_PONTO` → `PACKAGE BODY` em outro arquivo → subprograma `RECALCULA`, e devolve `caminho:linha:coluna`. 🟡

## Fluxos Alternativos
- **.NET ausente:** o C# fica sem LSP, o `doctor` mostra como instalar e o PL/SQL continua funcionando. 🟡
- **`csharp-ls` incompatível com o SDK:** R8. 🟡
- **Servidor trava:** a consulta expira em 60 s, o servidor é reiniciado na próxima e o modelo recebe `TOOL_TIMEOUT`. 🟢/🟡
- **Arquivo `.sql` só com DML ou consultas:** é indexado sem objetos; `documentSymbol` lista as instruções principais. 🟡
- **Objeto PL/SQL fora dos arquivos:** "não encontrado no workspace". Se R16 estiver ativo, o catálogo é consultado. 🟡

## Dependências
- Cliente LSP: `async-lsp` ou `tower-lsp` em modo cliente, mais `lsp-types`.
- `tree-sitter` + gramática SQL/PL-SQL; `rusqlite` para o índice.
- Integrações gerenciadas (spec 7): instalação do `csharp-ls`. Ferramentas (spec 3): ganchos pós-`Edit`/`Write`.
- Política e gate (spec 2): enriquecimento pelo catálogo.

## Requisitos Não Funcionais
| Tipo | Requisito inferido | Evidência no código | Confiança |
|---|---|---|---|
| Desempenho | Timeout de 60 s, até 100 locais, 16.000 caracteres | `packages/lsp/tool-lsp` | 🟢 |
| Segurança | Ambiente sem credenciais para os servidores | `packages/lsp/lsp-stdio` | 🟢 |
| Desempenho | Espera de até 3 s pelos diagnósticos depois da edição | — (novo) | 🟡 |
| Desempenho | Indexação incremental do PL/SQL por hash | — (novo) | 🟡 |
| Segurança | Projeto auxiliar fora das entregas; nenhuma alteração em arquivos do projeto | regra de governança do Totvs | 🟡 |

> Novo em grande parte. Validar no piloto com os `.sql` e `.cs` reais do time.

## Critérios de Aceitação
```gherkin
Dado o .NET SDK 7.0.202 instalado e o csharp-ls 0.28 exigindo net10.0
Quando dsh integrations doctor roda
Então o csharp-ls aparece com as duas opções: instalar o .NET 10 ou fixar a 0.10.0 (net7.0)

Dado o csharp-ls ativo e codPreparar.cs sem .csproj
Quando o modelo edita o arquivo introduzindo um erro de sintaxe
Então o resultado do Edit traz o diagnóstico CS com linha e coluna
E os erros de tipos RM.* não resolvidos não aparecem

Dado um PACKAGE BODY PKG_X com a procedure P definida em pkg_x.pkb
Quando o modelo pede goToDefinition em PKG_X.P a partir de outro .sql
Então o resultado aponta para a linha da procedure P no body

Dado um .sql com erro de sintaxe PL/SQL
Quando o modelo o edita
Então o resultado traz o erro de sintaxe do parser e não traz validação semântica

Dado catalog_enrich = off
Quando um objeto não está em nenhum arquivo
Então a resposta é "não encontrado no workspace" e nenhuma consulta ao banco é feita
```

## Prioridade
| Requisito | MoSCoW | Justificativa |
|---|---|---|
| Ferramenta `LSP` com as operações básicas | Must | Paridade com o Claude Code |
| Índice PL/SQL (definição, referências, símbolos) | Must | Linguagem predominante no Totvs AGP |
| `csharp-ls` com detecção do .NET | Must | Decisão `lsp-totvs` |
| Diagnósticos depois da edição | Should | Ganho grande de qualidade nas entregas |
| Projeto auxiliar para `.cs` soltos | Should | Formato real dos arquivos C# do time |
| Servidores pré-configurados (Rust, TS, Python, Go) | Should | Uso fora do Totvs, inclusive no próprio dsh |
| Enriquecimento pelo catálogo Oracle | Could | Depende de rede e do gate de dados |
| C# direto do XOML | Won't (por ora) | Evolução futura (R11) |

## Rastreabilidade de Código
| Arquivo | Função / Classe | Cobertura |
|---|---|---|
| `packages/lsp/lsp`, `lsp-stdio`, `tool-lsp` | cliente, ambiente limpo, ferramenta | 🟢 |
| `_reversa_sdd/adrs/104-*.md` | decisão | 🟢 |
| `D:/Projetos/Totvs AGP/VSCode/entregas/2026-10-28-copia-avrec-riscos/v3-fv-codigo/fonte/*.cs` | trechos C# soltos | 🟢 |
| `D:/Projetos/Totvs AGP/VSCode/**/*.sql` (181) | corpus PL/SQL | 🟢 |
| `…/XOMLDATA-v3.3.xoml` | `RMSWorkflowActivity` com `CodeActivity` | 🟢 |
| — | diagnósticos, índice PL/SQL, projeto auxiliar | — (novo) |

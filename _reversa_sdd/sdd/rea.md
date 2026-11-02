# REA: engenharia reversa em qualquer projeto (`harness-integrations::rea`)

## Visão Geral
O **REA** (morluto/rea, MIT; pacote npm `rea-agents`) é um servidor MCP com CLI de engenharia reversa local. Ele analisa binários nativos, aplicações JS/Electron, **.NET**, APK, sites, capturas HAR/pcap, firmware, WASM e outros formatos, e entrega cada conclusão com as evidências e as limitações. 🟢

No dsh, o REA é uma **ferramenta transversal**: o usuário a usa em qualquer projeto informando um **alvo**, que é uma **URL web** ou um **caminho local** (arquivo, pasta, binário, APK, pcap…). O dsh classifica o alvo, prepara a sessão de análise, encaminha às ferramentas certas do REA e guarda o pacote de evidências. Decisão `rea-integration`; o código TS do REA **não** é portado (ADR 102). 🟡

## Responsabilidades
- Garantir que o REA esteja instalado, atualizado e registrado (spec 7). 🟢
- Oferecer um ponto de entrada único para o alvo: `dsh rea <alvo>`, `/rea <alvo>` e a skill `reverse-engineer-anything`. 🟡
- Classificar o alvo (URL × caminho; tipo de artefato) e sugerir o fluxo de análise. 🟡
- Verificar pré-requisitos por tipo de alvo (Ghidra, JDK/JADX, TShark, Chrome…). 🟡
- Aplicar a política de dados e o gate do workspace ao conteúdo extraído do alvo. 🟡
- Guardar e exportar as evidências por análise, ligadas à sessão. 🟡

## Interface

### Entradas
| Forma | Exemplo | Confiança |
|---|---|---|
| CLI | `dsh rea https://app.exemplo.com` · `dsh rea "D:\bin\RM.Lib.dll" --goal "como calcula férias"` | 🟡 |
| Slash command | `/rea <alvo> [objetivo]` | 🟡 |
| Linguagem natural | "analise o executável X e diga…" → o modelo carrega a skill `reverse-engineer-anything` | 🟢 (skill do REA) |
| Ferramentas diretas | `mcp__rea__*` (ex.: `open_binary`, `analyze_web_bundle`, `get_evidence_bundle`) | 🟢 |

Opções da CLI: `--goal <texto>`, `--profile quick|deep`, `--out <pasta>`, `--no-capture` (proíbe execução e captura dinâmica), `--project <ws>` (aplica a política de outro workspace). 🟡

### Classificação do alvo → fluxo do REA
| Alvo detectado | Como o dsh detecta | Ferramentas iniciais do REA | Pré-requisitos | Confiança |
|---|---|---|---|---|
| URL `http(s)://` | esquema | `analyze_web_bundle`, `capture_browser_scenario` | Chrome | 🟢 ferramentas / 🟡 roteamento |
| HAR / pcap / pcapng | extensão + magic | `inspect_artifact`, captura de rede | TShark (pcap) | 🟡 |
| PE com cabeçalho CLR (`.dll`/`.exe` .NET) | PE + diretório CLI não vazio | `open_binary` + `compare_managed_members`, `project_managed_application_graph` | ILSpy/decompilador conforme o REA | 🟢/🟡 |
| PE/ELF/Mach-O nativo | magic | `open_binary`, `binary_overview`, `analyze_function`, `decompile_all` | Ghidra (ou Hopper/IDA) | 🟢 |
| APK / AAB | ZIP + `AndroidManifest.xml` | `project_android_application_graph`, `inspect_apk_signature` | JDK + JADX/Apktool | 🟢 |
| Pasta ou `.asar` com `package.json` | arquivos | `analyze_javascript_application`, `trace_application_feature` | Node | 🟢 |
| WASM | magic `\0asm` | `inspect_wasm_artifact` | WABT | 🟢 |
| Imagem de firmware / binário desconhecido | heurística | `inspect_artifact`, `extract_artifact` | Binwalk/Unblob | 🟢 |
| Duas versões (`--compare A B`) | dois alvos | `compare_artifacts`, `compare_application_versions` | conforme o tipo | 🟢 |

### Saídas
| Saída | Local | Confiança |
|---|---|---|
| Resposta ao objetivo, com cada afirmação citando `evidence_id` | conversa | 🟢 (modelo de evidência do REA) |
| Pacote de evidências exportado | `<out>` ou `~/.dsh/rea/<data>-<slug>/evidence/` | 🟢/🟡 |
| Relatório markdown (objetivo, alvo, hash, achados, incógnitas, limitações) | `…/report.md` | 🟡 |
| Registro da análise | evento `rea/analysis {target_kind, target_hash, workspace, evidence_dir}` no log de sessão | 🟡 |

## Regras de Negócio
- **R1.** Qualquer cwd pode usar o REA. O alvo **não precisa** estar dentro do workspace atual. O REA trabalha só com leitura local; o alvo nunca é enviado a um serviço externo pelo dsh. 🟢/🟡
- **R2.** Para um alvo local, o dsh calcula o SHA-256 (de arquivos até 2 GB; de pastas, o hash da árvore) e o registra. Um alvo que muda durante a análise gera um aviso. 🟡
- **R3.** Para um alvo URL, a **captura dinâmica** (abrir no navegador, executar o cenário) exige confirmação, a menos que `--no-capture` esteja ativo. Ela usa um perfil de navegador temporário e isolado, nunca o perfil logado do usuário (por exemplo, `.claude/navegador-perfis/gpflow`). 🟡
- **R4.** Executar o alvo (`capture_process_scenario`, ambientes RL ou Docker) é sempre `ask`, porque é um efeito externo e roda código não confiável. Dentro de sandbox, quando o REA oferecer. 🟡
- **R5.** O conteúdo extraído do alvo (strings, código decompilado, respostas HTTP) é **dado não confiável** (`_meta.provenance.untrusted_content`). Instruções encontradas no alvo nunca são seguidas. 🟢
- **R6.** A política de dados vale para o conteúdo extraído:
  - se o alvo estiver dentro de um workspace com política (ex.: Totvs AGP), ou se `--project` for usado, valem as regras `allowed_providers`, `data_policy` e `data_gate` daquele workspace;
  - um alvo fora de qualquer workspace usa a política do workspace atual. 🟡
- **R7.** O REA é usado para fins diversos, e a análise de binários do TOTVS RM pode nem acontecer. **Decisão do usuário (2026-11-01): não há restrição.** O dsh não bloqueia nem exige confirmação por licença de terceiros. A responsabilidade pelo uso é do usuário. Um aviso informativo opcional (`rea.license_notice = false` por padrão) pode ser ligado. 🟢 [Revisão: era 🔴]
- **R8.** Toda conclusão apresentada deve citar evidências. Uma afirmação sem evidência é marcada como hipótese. O dsh instrui isso pela skill do REA e por uma seção de prompt ativa enquanto houver uma análise aberta. 🟢/🟡
- **R9.** As incógnitas (`record_unknown`/`list_unknowns`) entram no relatório numa seção própria. 🟢
- **R10.** Timeouts: 330 s por chamada (Ghidra), com progresso visível e cancelamento propagado (spec 7, R4–R6). 🟢
- **R11.** A versão do REA, as ferramentas usadas e os perfis (`analysis_profile`) ficam no relatório, para que a análise seja reproduzível. 🟢/🟡
- **R12.** Com o REA indisponível (Node ausente ou desatualizado), `dsh rea` diz o que falta e como instalar. O modelo é informado de que a capacidade não existe nesta sessão. 🟡

## Fluxo Principal
1. O usuário digita `dsh rea "D:\Totvs\bin\RM.Lib.Folha.dll" --goal "regra de cálculo do adicional noturno"`. 🟡
2. O dsh verifica a integração (spec 7) e classifica o alvo como PE .NET (cabeçalho CLR presente). 🟡
3. Checa os pré-requisitos do fluxo .NET. Calcula o hash. Aplica a política (R6) e, se houver, o aviso de licença (R7). 🟡
4. Abre (ou retoma) uma sessão do dsh com o objetivo, a skill `reverse-engineer-anything` carregada e uma seção de prompt com o alvo, o tipo e o fluxo sugerido. 🟡
5. O modelo chama `mcp__rea__open_binary` e depois as ferramentas de análise managed. O progresso aparece na TUI. 🟢
6. O modelo responde citando `evidence_id`s. O dsh chama `export_evidence_bundle` e grava `report.md`. 🟢/🟡
7. O evento `rea/analysis` é gravado. A análise pode ser retomada depois pelo mesmo hash. 🟡

## Fluxos Alternativos
- **URL atrás de login:** o REA captura só o que for público. Usar uma sessão autenticada exige que o usuário forneça um HAR exportado, analisado como arquivo local. 🟡
- **Ghidra ausente para um binário nativo:** o dsh avisa e oferece a análise superficial (`inspect_artifact`, strings, imports), que não depende dele. 🟡
- **Alvo grande (> 2 GB):** o hash é calculado por amostragem (início, meio e fim mais o tamanho), com aviso. 🟡
- **Comparação de versões (`--compare`):** gera um relatório de diferenças com evidências dos dois lados. 🟢
- **Cancelamento:** a observação parcial é preservada e o relatório é marcado como incompleto. 🟢

## Dependências
- Cliente MCP e integrações gerenciadas (spec 7). Plugins (spec 6, R12): a skill e o plugin do REA.
- Política de dados (spec 2). Browser use (spec 10) para captura de URL, quando o REA delega ao navegador.
- Ferramentas externas por tipo de alvo, instaladas pelo usuário: Ghidra, JDK/JADX, TShark, Binwalk, Chrome.

## Requisitos Não Funcionais
| Tipo | Requisito inferido | Evidência no código | Confiança |
|---|---|---|---|
| Desempenho | Timeout de 330 s por chamada (Ghidra) | docs do REA (`REA_GHIDRA_STARTUP_TIMEOUT_MS`) | 🟢 |
| Desempenho | Resultado stdio até 10 MiB; excedente em evidência ou export | docs do REA | 🟢 |
| Segurança | Conteúdo do alvo rotulado como não confiável | `_meta.provenance.untrusted_content` (REA) | 🟢 |
| Segurança | Execução do alvo sempre com confirmação | — (novo) | 🟡 |
| Rastreabilidade | Evidência por afirmação; versão e perfil no relatório | modelo de evidência do REA | 🟢 |

> Inferido a partir da documentação do REA. Validar com uma análise piloto.

## Critérios de Aceitação
```gherkin
Dado o REA instalado e um cwd qualquer
Quando o usuário roda dsh rea https://exemplo.com --no-capture
Então o alvo é classificado como URL, só a análise estática do bundle web é feita
E o relatório fica em ~/.dsh/rea/<data>-exemplo-com/report.md

Dado um .dll .NET
Quando dsh rea o recebe como alvo
Então o fluxo managed é sugerido e o hash SHA-256 aparece no relatório

Dado uma análise que pede capture_process_scenario
Quando o modelo chama a ferramenta
Então o usuário precisa aprovar a execução do alvo

Dado um alvo dentro de D:\Projetos\Totvs AGP\VSCode
Quando a análise envia conteúdo ao modelo
Então só provedores permitidos pela política do Totvs são usados

Dado Node ausente
Quando o usuário roda dsh rea x.exe
Então a mensagem informa a versão mínima do Node e o comando de instalação
```

## Prioridade
| Requisito | MoSCoW | Justificativa |
|---|---|---|
| Registro gerenciado + skill | Must | Decisão `rea-integration` |
| Alvo URL ou caminho local, em qualquer projeto | Must | Requisito explícito do usuário |
| Classificação do alvo e fluxo sugerido | Should | Acelera o uso; o modelo pode decidir sozinho |
| Política de dados no conteúdo extraído | Must | LGPD quando o alvo vem do Totvs |
| Confirmação para captura e execução | Must | Segurança |
| Relatório + evidências exportadas | Should | Reprodutibilidade |
| Comparação de versões | Could | Uso ocasional |

## Rastreabilidade de Código
| Arquivo | Função / Classe | Cobertura |
|---|---|---|
| https://github.com/morluto/rea `docs/mcp-contracts.md` | catálogo de ferramentas | 🟢 |
| pacote npm `rea-agents`: `scripts/rea.mjs`, `.claude-plugin/plugin.json`, skill `reverse-engineer-anything` | servidor, plugin e skill | 🟢 |
| `_reversa_sdd/code-analysis.md` §REA | requisitos levantados | 🟢 |
| — | `dsh rea`, classificação do alvo, relatório | — (novo) |

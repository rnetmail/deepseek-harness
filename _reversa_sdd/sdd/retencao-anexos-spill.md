# Retenção de anexos, spill, sessões e artefatos (`harness-store::retention`)

## Visão Geral
O dsh guarda muitos dados no disco: sessões, anexos endereçados por conteúdo, arquivos de spill, evidências do REA, cópias sincronizadas, backups e índices. No dsh TS, o spill é limpo depois de 30 dias, mas os **anexos nunca são apagados** (dívida 9). Esta spec define a retenção de cada tipo, a coleta de lixo de blobs sem referência e uma retenção opcional por workspace para dados sensíveis (LGPD) (lacuna G3 da revisão). 🟢 (estado atual) / 🟡 (alvo)

## Responsabilidades
- Definir a política de retenção por tipo de artefato. 🟡
- Remover anexos que nenhuma sessão referencia, com segurança e sem quebrar a reconstrução de requests. 🟡
- Permitir a remoção explícita de sessões e a retenção máxima por workspace. 🟡
- Mostrar o uso de disco e o que será removido antes de remover. 🟡

## Interface

### Política por tipo
| Artefato | Local | Padrão | Confiança |
|---|---|---|---|
| Sessões | `~/.dsh/sessions/` | **mantidas indefinidamente**; remoção só explícita ou por política do workspace | 🟡 |
| Anexos (blobs por hash) | `~/.dsh/attachments/` | removidos quando **nenhuma sessão** os referencia há mais de 7 dias (GC) | 🟡 (dsh TS: nunca 🟢) |
| Spill | `~/.dsh/spill/<sessão>/` | 30 dias | 🟢 (`cleanupPeriodDays`) |
| Lixeira do sync | `~/.dsh/synced/.trash/` | 30 dias | 🟡 (spec 5) |
| Evidências do REA | `~/.dsh/rea/` | mantidas; `dsh rea prune --older-than` | 🟡 |
| Backups do GBrain | `~/.dsh/backups/gbrain/` | 14 cópias | 🟡 (spec 9) |
| Índices (`index/`) | `~/.dsh/index/` | derivados; reconstruídos sob demanda | 🟢/🟡 |
| Logs do dsh | `~/.dsh/logs/` | 14 dias ou 100 MB | 🟡 |
| Cache de plugins (versões antigas) | `~/.dsh/plugins/cache/` | última versão + 1 anterior | 🟡 (spec 6) |

### Configuração
```toml
# ~/.dsh/config.toml
[retention]
attachments_gc_grace_days = 7
spill_days = 30
logs_days = 14

# <workspace>/.dsh/workspace.toml (opcional, ex.: Totvs AGP)
[retention]
session_max_days = 0          # 0 = sem limite; N = apaga sessões deste workspace com mais de N dias
purge_spill_on_close = false  # true = apaga o spill da sessão ao fechá-la
```

### Comandos
| Comando | Efeito |
|---|---|
| `dsh storage usage` | Uso por tipo e por workspace |
| `dsh storage gc [--dry-run]` | Coleta de lixo de anexos e limpeza por política |
| `dsh sessions delete <id> \| --workspace <dir> --older-than <dias>` | Remove sessões, incluindo subsessões, spill e anexos que ficarem órfãos |

## Regras de Negócio
- **R1.** O **anexo** é endereçado por conteúdo (hash), deduplicado e verificado na leitura. Um anexo **referenciado** por qualquer evento de qualquer sessão existente **nunca** é removido. Isso preserva a regra de que todo request é reconstruível (spec 1, R1). 🟢 (endereçamento) / 🟡 (GC)
- **R2. GC de anexos (mark-and-sweep):**
  - o dsh percorre as sessões, marca os hashes referenciados e remove os blobs não marcados cujo mtime passou do período de carência;
  - o GC roda no arranque, no máximo uma vez a cada 24 h, e em `dsh storage gc`;
  - uma sessão aberta em escrita durante o GC protege os blobs criados depois do início da varredura (pelo mtime). 🟡
- **R3.** A **remoção de sessão** é sempre explícita (comando ou política do workspace) e apaga as subsessões, o spill da sessão e a entrada no índice FTS. Os anexos que ficarem órfãos saem no próximo GC. 🟡
- **R4. `session_max_days` do workspace:**
  - quando definido, as sessões com `cwd` dentro do workspace e o último evento mais antigo que N dias são removidas pelo GC;
  - antes da primeira remoção automática, o dsh pede confirmação uma única vez e registra a decisão em `state/`.
  - **Decisão do usuário (2026-11-01):** o dsh oferece o **parâmetro** `session_max_days` em `[retention]` do `workspace.toml` (e um padrão global em `~/.dsh/config.toml`). O valor padrão é `0` (desligado); o time escolhe N quando quiser. O parâmetro também pode ser definido com `dsh config set retention.session_max_days <N> --workspace`. 🟢 [Revisão: era 🔴]
- **R5.** O spill segue a limpeza do dsh: no arranque, os arquivos com mais de `spill_days` são removidos, com proteção contra symlinks. Com `purge_spill_on_close`, o spill da sessão é apagado ao fechá-la. 🟢/🟡
- **R6.** A memória (arquivos markdown e GBrain) **não** é afetada pela retenção de sessões. Ela é curada e removida só pelo agente ou pelo usuário (spec 9). 🟡
- **R7.** Toda remoção automática gera uma linha em `~/.dsh/logs/retention.jsonl` (tipo, quantidade, bytes, política), sem conteúdo. 🟡
- **R8.** `--dry-run` mostra exatamente o que seria removido e quanto espaço seria liberado. 🟡

## Fluxo Principal
1. No arranque, se o último GC tiver mais de 24 h, o dsh roda o GC em segundo plano com prioridade baixa. 🟡
2. Ele limpa o spill vencido, a lixeira do sync, os logs e as versões antigas de plugins. 🟢/🟡
3. Aplica o `session_max_days` dos workspaces que o definem (com a confirmação prévia de R4). 🟡
4. Faz o mark-and-sweep dos anexos (R2) e grava o log (R7). 🟡

## Fluxos Alternativos
- **Sessão v3 corrompida durante a marcação:** a sessão é tratada como se referenciasse **todos** os blobs (fail-safe) e um aviso é registrado. 🟡
- **Disco cheio:** `dsh storage usage` sugere o GC e os itens maiores. O dsh nunca apaga sessões sem política ou comando. 🟡

## Dependências
- Sessão e eventos (spec 1), spill (spec 3), sync (spec 5), plugins (spec 6), memória (spec 9), REA (spec 8), configuração (`configuracao-dsh-home.md`).

## Requisitos Não Funcionais
| Tipo | Requisito inferido | Evidência no código | Confiança |
|---|---|---|---|
| Integridade | Anexo referenciado nunca removido | `packages/attachment/attachment-local` (endereçamento por conteúdo) | 🟢/🟡 |
| Disponibilidade | Spill limpo depois de 30 dias no arranque | `packages/spill/spill-local/src/index.ts`, `cleanup.ts` | 🟢 |
| Privacidade | Retenção máxima opcional por workspace | decisão `totvs-data-gate` (LGPD) | 🟡 |
| Desempenho | GC no máximo 1 vez por dia, em segundo plano | — (novo) | 🟡 |

## Critérios de Aceitação
```gherkin
Dado um anexo referenciado por uma sessão de 200 dias
Quando o GC roda
Então o anexo é mantido

Dado uma sessão removida com dsh sessions delete
Quando o GC roda 7 dias depois
Então os anexos que só ela referenciava são removidos e o log registra a quantidade e os bytes

Dado session_max_days = 90 no workspace do Totvs
Quando o GC roda pela primeira vez
Então o usuário confirma antes, e depois as sessões com mais de 90 dias daquele workspace são removidas

Dado dsh storage gc --dry-run
Quando executado
Então lista o que seria removido sem remover nada
```

## Prioridade
| Requisito | MoSCoW | Justificativa |
|---|---|---|
| GC de anexos sem quebrar a reconstrução | Should | Corrige a dívida 9 |
| Limpeza do spill (30 dias) | Must | Herança do dsh |
| Remoção explícita de sessões | Should | Controle do usuário |
| `session_max_days` por workspace | Should | LGPD (valor a definir) |
| `storage usage` | Could | Diagnóstico |

## Rastreabilidade de Código
| Arquivo | Função / Classe | Cobertura |
|---|---|---|
| `packages/attachment/attachment`, `attachment-local` | armazenamento por hash | 🟢 |
| `packages/spill/spill-local/src/index.ts`, `cleanup.ts` | limpeza do spill | 🟢 |
| `_reversa_sdd/architecture.md` §4 (dívida 9) | anexos sem retenção | 🟢 |
| — | GC, retenção por workspace | — (novo) |

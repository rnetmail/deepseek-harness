# Build, empacotamento e release do executável `dsh` (processo + `harness-cli::update`)

## Visão Geral
Como o dsh Rust é construído, verificado, assinado, distribuído e atualizado como **binário único**. O processo segue os padrões das ferramentas Rust do CoreOS (coreos-installer, afterburn, zincati), com build reprodutível e vendoring filtrado, e o dsh **não envia nada** além do download (lacuna G2 da revisão). 🟡

## Responsabilidades
- Organizar o código num Cargo workspace com crates de biblioteca e um binário fino. 🟡
- Garantir builds reprodutíveis e offline com dependências vendorizadas e filtradas por plataforma. 🟢 (padrão CoreOS) / 🟡
- Rodar testes, lint, auditoria de licenças e de vulnerabilidades no CI. 🟡
- Produzir artefatos para Windows (x86_64, arm64) e Linux e macOS (secundários), com assinatura. 🟡
- Atualizar o binário instalado com verificação de assinatura (`dsh update`). 🟡

## Interface

### Workspace Cargo
```
dsh/
├─ Cargo.toml            [workspace] resolver = "3", edition 2021, rust-version (MSRV) fixo
├─ crates/
│  ├─ harness-core  harness-llm  harness-tools  harness-sandbox  harness-mcp
│  ├─ harness-claude-compat  harness-integrations  harness-script  harness-store
│  ├─ harness-tui  harness-acp  harness-sdk
│  └─ harness-cli        → binário `dsh` (fino: clap + wiring)
├─ vendor/               gerado por cargo-vendor-filterer (só plataformas-alvo)
├─ deny.toml             cargo-deny: licenças, advisories, fontes
└─ xtask/                tarefas de release (empacotar, assinar, gerar notas)
```
Padrões CoreOS adotados: `clap` 4 derive, `anyhow` + `thiserror`, crates de biblioteca com bins finos e metadados `[package.metadata.vendor-filter]`. 🟢 (code-analysis §CoreOS)

### Artefatos de release
| Artefato | Conteúdo |
|---|---|
| `dsh-<versão>-x86_64-pc-windows-msvc.zip` | `dsh.exe` + `LICENSE` + `THIRD_PARTY_NOTICES.md` |
| `dsh-<versão>-aarch64-pc-windows-msvc.zip` | idem |
| `dsh-<versão>-x86_64-unknown-linux-gnu.tar.gz`, `…-apple-darwin.tar.gz` | secundários |
| `SHA256SUMS` + `SHA256SUMS.minisig` | hashes e assinatura minisign |
| `dsh-<versão>-vendor.tar.zst` | código vendorizado, para build offline |
| `CHANGELOG.md` | inclui a seção "Portado do upstream" (spec 17) |

### Comandos
| Comando | Efeito |
|---|---|
| `cargo xtask dist --target <triple>` | Build release + empacotamento |
| `cargo xtask sign` | Gera `SHA256SUMS` e assina com minisign; assina o `.exe` com Authenticode quando houver certificado |
| `dsh update [--check] [--channel stable\|rc]` | Verifica e aplica a atualização |
| `dsh --version --verbose` | Versão, commit, data do build, MSRV, features e base do upstream sincronizada |

## Regras de Negócio
- **R1.** O build é **offline e reprodutível**: `cargo build --locked --offline` com `vendor/`. O mesmo commit e a mesma toolchain geram o mesmo binário (`SOURCE_DATE_EPOCH`, `--remap-path-prefix`, sem timestamps). 🟡
- **R2.** O `vendor/` é filtrado para as plataformas-alvo (`cargo-vendor-filterer --platform x86_64-pc-windows-msvc …`), o que reduz o tamanho e a superfície. 🟢 (CoreOS) / 🟡
- **R3.** O MSRV é declarado (`rust-version`) e testado no CI. Ele só sobe numa minor, com nota no CHANGELOG. 🟢 (CoreOS: MSRV 1.87 no coreos-installer 0.27) / 🟡
- **R4.** O CI roda `cargo fmt --check`, `clippy -D warnings`, os testes (unidade, integração e conformidade dos formatos de sessão v3 e v4 contra fixtures do upstream), `cargo deny check` (licenças permitidas: MIT, Apache-2.0, BSD, ISC, Zlib, MPL-2.0; **AGPL e GPL proibidas** no binário) e `cargo audit`. 🟡
- **R5.** O `THIRD_PARTY_NOTICES.md` é gerado (`cargo about`) e inclui o aviso MIT da DeepSeek pelo código e pelas ideias portadas (spec 17, R8). 🟢 (licença MIT do upstream) / 🟡
- **R6. Atualização:**
  - `dsh update` baixa só do canal oficial do projeto (GitHub Releases do repositório do dsh Rust);
  - verifica a assinatura minisign com a **chave pública embutida** no binário e o SHA-256, e troca o executável de forma atômica (no Windows, renomeia o em uso para `.old` e o remove na próxima execução);
  - uma assinatura inválida aborta sem tocar no binário. 🟡
- **R7.** A atualização **não** envia identificadores, telemetria nem inventário. O pedido HTTP leva só o `User-Agent: dsh/<versão>`. 🟢 (ADR 101)
- **R8. Sem certificado Authenticode: solução de contorno** (decisão do usuário, 2026-11-01). 🟢 [Revisão: era 🔴]
  1. **Primeira instalação:** o usuário baixa o `.zip`, confere o SHA-256 contra o `SHA256SUMS` assinado (`dsh-verify.ps1`, que vem no release e também checa o `.minisig`) e remove a marca da web com `Unblock-File .\dsh.exe`. O README do release traz os três comandos.
  2. **SmartScreen:** se ainda aparecer, usar "Mais informações → Executar assim mesmo" uma única vez. O hash conferido garante a integridade.
  3. **Atualizações:** `dsh update` baixa pelo próprio dsh, sem navegador, e grava o binário **sem** o Mark-of-the-Web (Zone.Identifier). Por isso as atualizações não passam pelo SmartScreen. A verificação minisign (R6) substitui a assinatura de código.
  4. **Política corporativa** (AppLocker/WDAC): se bloquear executáveis não assinados, instalar em `%LOCALAPPDATA%\Programs\dsh` e pedir à TI a liberação por **hash** ou por **caminho**. O `dsh doctor` detecta o bloqueio e mostra o hash a liberar.
  5. O Authenticode continua como evolução opcional (`xtask sign` assina se houver um certificado configurado).
- **R9.** As features de compilação são opcionais por componente (por exemplo, `--no-default-features --features tui,acp`), mas o artefato oficial inclui todas as obrigatórias: browser, computer use, agent teams e LSP. 🟡
- **R10.** Dependências com código nativo (`rusqlite` bundled, `tree-sitter`, `zstd`) compilam estaticamente. O binário não exige DLLs extras além das do sistema. 🟡

## Fluxo Principal
1. Uma tag `vX.Y.Z` no repositório dispara o CI: build por alvo, testes, `deny` e `audit`. 🟡
2. `xtask dist` empacota e `xtask sign` assina (minisign, e Authenticode se houver). 🟡
3. Publica o release com os artefatos, `SHA256SUMS`, `.minisig`, o vendor e o CHANGELOG. 🟡
4. No usuário, `dsh update --check` vê a versão nova e `dsh update` aplica (R6). 🟡

## Fluxos Alternativos
- **Máquina sem acesso ao GitHub (rede corporativa):** `dsh update --from <zip>` aplica um pacote baixado manualmente, com a mesma verificação. 🟡
- **TLS corporativo:** o HTTP usa os certificados do SO (spec 7, R16b). 🟢
- **Falha no meio da troca do binário:** o `.old` é restaurado. 🟡

## Dependências
- `cargo-vendor-filterer`, `cargo-deny`, `cargo-audit`, `cargo-about`, `minisign`, `signtool` (Windows SDK).
- Crates: `self_update` (ou implementação própria) + `minisign-verify`, `reqwest` + `rustls-platform-verifier`.

## Requisitos Não Funcionais
| Tipo | Requisito inferido | Evidência | Confiança |
|---|---|---|---|
| Reprodutibilidade | Build offline com vendor filtrado | padrões CoreOS (`cargo-vendor-filterer`) | 🟢 |
| Segurança | Atualização só com assinatura válida | — (novo) | 🟡 |
| Licença | AGPL/GPL proibidas no binário (por exemplo, a extensão `cua-perception`) | spec 10; `cargo deny` | 🟡 |
| Privacidade | Nenhum dado enviado na atualização | ADR 101 | 🟢 |
| Portabilidade | Binário estático sem DLLs extras | — (novo) | 🟡 |

## Critérios de Aceitação
```gherkin
Dado o mesmo commit e a mesma toolchain
Quando o binário é gerado duas vezes
Então os dois dsh.exe têm o mesmo SHA-256

Dado uma dependência nova com licença AGPL
Quando o CI roda cargo deny
Então o build falha

Dado um release cujo .minisig não confere
Quando o usuário roda dsh update
Então a atualização é abortada e o binário atual continua intacto

Dado dsh --version --verbose
Quando executado
Então mostra versão, commit, MSRV e a base do upstream sincronizada
```

## Prioridade
| Requisito | MoSCoW | Justificativa |
|---|---|---|
| Workspace + build release Windows x86_64 | Must | Uso principal |
| Vendoring + build reprodutível | Should | Padrão CoreOS; builds offline |
| `cargo deny` / `audit` / NOTICES | Must | Licenças (MIT do upstream, proibição de AGPL) |
| `dsh update` com minisign | Should | Binário único sem gerenciador de pacotes |
| Contorno sem Authenticode (verificação, `Unblock-File`, update sem MOTW) | Must | Decisão do usuário (R8) |
| Authenticode | Could | Só se houver certificado |
| Linux e macOS | Could | Secundários |

## Rastreabilidade de Código
| Arquivo | Função / Classe | Cobertura |
|---|---|---|
| `_reversa_sdd/code-analysis.md` §CoreOS | padrões de crates, vendoring, MSRV | 🟢 |
| `_reversa_sdd/c4-containers.md` | crates do workspace | 🟢 |
| `LICENSE` (MIT, DeepSeek) | atribuição | 🟢 |
| — | xtask, update, assinatura | — (novo) |

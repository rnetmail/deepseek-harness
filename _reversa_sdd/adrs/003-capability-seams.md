# ADR 003 — capability seams

- **Status:** Retroativo
- **Data:** 2026-06-13
- **Fonte:** `.agents/notes/implemented/architecture/2026-06-13-capability-seams.md`
- **Confiança:** 🟢 nota de design implementada

## Contexto
Capacidades (shell, web, sandbox, LLM) precisam de implementações trocáveis (local, remota, sandboxed).

## Decisão
Tripé Service Definition / Provider / Consumer. O consumer injeta a chave do serviço e nunca importa tipos do provider.

## Consequências
Permite SSH runtime, sandbox por SO e provedores de busca plugáveis.

## Implicação para o harness Rust
No Rust: traits (`ShellExecutor`, `WebSearch`, `SandboxBackend`, `LlmAdapter`) + `Arc<dyn Trait>` registradas no arranque.

# ADR 007 — fronteiras de credenciais

- **Status:** Retroativo
- **Data:** 2026-07-30
- **Fonte:** `.agents/notes/implemented/architecture/2026-07-30-credential-boundaries-and-atomic-registration.md`
- **Confiança:** 🟢 nota de design implementada

## Contexto
Chaves de API ficam em disco, e as ferramentas do agente rodam como o mesmo usuário.

## Decisão
O documento de credenciais pertence só ao provider. Nunca é posto no `process.env` nem entregue ao modelo. A limitação é declarada abertamente, e o cofre do SO é registrado como a solução real.

## Consequências
Risco residual aceito no dsh.

## Implicação para o harness Rust
**Superado no Rust:** credenciais no Windows Credential Manager (`keyring`). Rotacionar as chaves expostas no `dsh.cmd`.

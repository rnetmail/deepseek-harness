# ADR 006 — contencao nativa de subprocessos

- **Status:** Retroativo
- **Data:** 2026-08-28
- **Fonte:** `.agents/notes/implemented/architecture/2026-08-28-subprocess-native-containment.md`
- **Confiança:** 🟢 nota de design implementada

## Contexto
Processos filhos órfãos sobrevivem a cancelamentos e timeouts.

## Decisão
Windows: Job Object kill-on-close. Linux: scope systemd transitório. Fallback mais fraco com aviso.

## Consequências
Cancelamento confiável de árvores de processo.

## Implicação para o harness Rust
No Rust: crate `windows` (JobObject) e `nix`/cgroups. Requisito para Bash, PowerShell e PTC.

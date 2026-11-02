# ADR 104 — lsp para csharp e plsql

- **Status:** Aceito (novo)
- **Data:** 2026-11-01
- **Fonte:** `decisão lsp-totvs`
- **Confiança:** 🟢 decisão registrada com o usuário

## Contexto
O Totvs AGP usa principalmente PL/SQL e C#, sem servidor LSP hoje.

## Decisão
C#: detectar o .NET já instalado e instalar/atualizar `csharp-ls`. PL/SQL: busca textual + índice de símbolos tree-sitter (packages, procedures, functions, triggers). LSP genérico opcional.

## Consequências
Navegação semântica e diagnostics em C#. PL/SQL com navegação aproximada.

## Implicação para o harness Rust
—

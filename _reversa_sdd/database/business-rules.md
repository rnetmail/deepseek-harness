# Regras de negócio nos dados persistidos

Não há triggers nem procedures. As regras abaixo ficam na camada de persistência.

## Log de sessão 🟢
1. **Append-only e dono único:** um handle de escrita por sessão (`SessionAlreadyOwnedError`). Uma perda de posse invalida a escrita (`SessionOwnershipLostError`).
2. **Versão do formato no nome do arquivo** (`session.v3`). Header de versão diferente → `migration-required` ou `unsupported`, nunca leitura silenciosa.
3. **Barreiras de checkpoint:** flush antes do stream do modelo, antes de cada chamada de ferramenta e no pre-step. Ferramentas que mudam estado durável (schedule, aprovações) só confirmam depois do flush. Se não conseguem, devolvem `persistence_uncertain`.
4. **Fork herda o prefixo** e não herda estado de domínio próprio (exemplo: lembretes).
5. **Compactação não apaga eventos:** insere um checkpoint `<compacted-summary>` e as projeções passam a usar o sufixo.
6. **Falha de append em auditoria** de aprovação = a ação é rejeitada (fail-closed).

## Settings e credenciais 🟢
7. Documento inválido no boot impede o arranque. Em runtime, mantém a última versão válida.
8. Escrita atômica 0600 / diretório 0700, sem seguir symlinks, com lock de 2 s e backoff.
9. Os metadados de credencial expostos à UI **nunca** incluem o valor.

## Storage e índice 🟢
10. `user_version` é gravado **por último**: um layout incompleto nunca fica marcado.
11. Versão do unit no banco ≠ versão do descritor → erro (sem migração implícita).
12. O índice FTS é **derivado**: pode ser apagado e reconstruído e não altera a persistência.
13. Autorização de busca entre sessões: só quando o `cwd` é idêntico.

## Blobs 🟢
14. Attachments deduplicados por conteúdo, verificados na leitura e **nunca apagados automaticamente** (🟡 crescimento ilimitado).
15. Spill: nomes imprevisíveis, diretório por sessão, limpeza no arranque após 30 dias.

## Implicações para o harness Rust 🟡
- **Manter:** log JSONL por eventos (zstd) com versão no nome, barreiras de flush, índice FTS derivado, store por hash.
- **Mudar:**
  - credenciais no **cofre do SO** (`keyring`);
  - um único `state.db` (rusqlite, WAL) para storage + FTS;
  - política de retenção para attachments;
  - `fsync` nas escritas atômicas.
- **Migração:** importador opcional de `~/.dsh/sessions/*.jsonl.zstd` (v3) e `settings.yaml`, para continuar sessões antigas do dsh.

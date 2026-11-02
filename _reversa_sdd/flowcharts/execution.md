# Fluxogramas — `sandbox`, `interaction`, `shell`, `fs`

## 1. Chamada de shell confinada com escalonamento 🟢

```mermaid
flowchart TD
  A[bash/pwsh command] --> B[resolve spec: cwd, timeout, caps]
  B --> C{modo efetivo da sessão}
  C -- danger-full-access --> R[executa argv original]
  C -- read-only / workspace-write --> D{backend consegue aplicar?}
  D -- não --> X[SANDBOX_UNAVAILABLE fail-closed]
  D -- sim --> E[executa confinado; enforcement full/partial]
  E --> F{acesso negado?}
  F -- não --> OUT[resultado: exit code, cauda + spill]
  F -- sim --> G[sandbox: file access denied + dica]
  G --> H[modelo repete UMA vez com sandbox_permissions + justification]
  H --> I{aprovação do usuário}
  I -- permitir uma vez --> R2[executa só esta chamada no modo mais amplo]
  I -- rejeitar/cancelar --> Y[negado]
```

## 2. Aprovação (`user-approval`) 🟢

```mermaid
flowchart LR
  A[pre-execute decide ask] --> B{turno aberto?}
  B -- não --> X[throw]
  B -- sim --> C{policy da sessão}
  C -- never --> D[rejeita deterministicamente]
  C -- ask --> E[waterfall approval/request]
  E --> F{answerer terminal?}
  F -- não --> G[unavailable → fail-closed]
  F -- sim --> H[allow / deny / cancelled]
  H --> I[auditoria no log; falha de append = rejeita]
```

## 3. Edição guardada por observação (`fs-observation-policy`) 🟢

```mermaid
flowchart TD
  A[edit ou write em arquivo existente] --> B{sessão leu este arquivo?}
  B -- não --> X1[FS_NOT_OBSERVED: leia antes]
  B -- sim --> C{versão mudou desde a leitura?}
  C -- sim --> X2[FS_STALE_VERSION: releia]
  C -- não --> D{edit: old_string único ou replace_all?}
  D -- não --> X3[erro de match]
  D -- sim --> E[substituição atômica + diff card]
  W[write em arquivo inexistente] --> E2[create permitido]
```

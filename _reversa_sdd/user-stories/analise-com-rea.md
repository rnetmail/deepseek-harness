# User Story: analisar qualquer alvo com o REA a partir de qualquer pasta

**Persona:** Robson, que precisa entender sistemas sem código-fonte disponível: um site, uma DLL .NET, um executável, uma captura de rede ou um aplicativo.
**Specs relacionadas:** [rea](../sdd/rea.md) · [mcp-cliente-gerenciado](../sdd/mcp-cliente-gerenciado.md) · [provedores-modelos](../sdd/provedores-modelos.md) · [browser-computer-agent-teams](../sdd/browser-computer-agent-teams.md)

## História
> **Como** analista que investiga sistemas de terceiros e legados,
> **quero** apontar o REA para uma URL ou um caminho local, de qualquer projeto, e receber conclusões com provas,
> **para** entender como algo funciona sem depender do código-fonte e sem enviar o alvo para fora da minha máquina.

## Cenários

### 1. Instalação assistida
```gherkin
Dado o Node 22.23+ instalado e o REA ausente
Quando rodo dsh integrations install rea
Então o REA é instalado pelo npm (com os certificados do Windows na rede da empresa)
E o servidor rea e a skill reverse-engineer-anything ficam disponíveis, sem duplicar o que já existir no Claude Code
```

### 2. Site
```gherkin
Dado que estou em qualquer pasta
Quando rodo dsh rea https://portal.exemplo.com --goal "como a tela de login valida o usuário" --no-capture
Então o alvo é classificado como URL e só o bundle web é analisado, sem executar a página
E o relatório cita evidências para cada conclusão e lista o que ficou desconhecido
```

### 3. DLL .NET
```gherkin
Dado um arquivo D:\temp\Componente.dll com cabeçalho CLR
Quando rodo dsh rea D:\temp\Componente.dll --goal "regra de arredondamento"
Então o fluxo managed é usado, o hash SHA-256 entra no relatório
E o progresso da análise aparece na tela, podendo ser cancelado com Esc sem perder o parcial
```

### 4. Binário de terceiros
```gherkin
Dado um binário assinado por outra empresa
Quando inicio a análise
Então vejo um aviso sobre licença e autorização antes de prosseguir
```

### 5. Execução do alvo
```gherkin
Dado uma análise que precisa executar o programa ou abrir a página
Quando o modelo pede a captura dinâmica
Então eu preciso aprovar
E o navegador usado é temporário e isolado, nunca o meu Chrome logado
```

### 6. Alvo dentro do Totvs AGP
```gherkin
Dado um alvo dentro de D:\Projetos\Totvs AGP\VSCode
Quando a análise envia trechos ao modelo
Então só provedores permitidos pela política do Totvs são usados
```

### 7. Ferramenta externa ausente
```gherkin
Dado um executável nativo e o Ghidra não instalado
Quando rodo dsh rea programa.exe
Então o dsh avisa o que falta e oferece a análise superficial (strings, imports, seções)
```

### 8. Comparar versões
```gherkin
Dado duas versões de um mesmo componente
Quando rodo dsh rea --compare v1\App.dll v2\App.dll
Então recebo um relatório das diferenças com evidências dos dois lados
```

## Critérios de pronto
- [ ] Instalação, atualização e `doctor` do REA pelo dsh.
- [ ] Alvo URL ou caminho local, aceito em qualquer cwd.
- [ ] Classificação automática do alvo e fluxo sugerido.
- [ ] Relatório com evidências, incógnitas, hash e versão do REA.
- [ ] Aprovação para captura e execução; navegador isolado.
- [ ] Política de dados aplicada ao conteúdo extraído.

## Pendências 🔴
- Validar com o jurídico a análise de binários do TOTVS RM.

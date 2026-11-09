# Onboarding: testar o backend Linux

1. Num Ubuntu 22.04/24.04 (ou num contêiner `rust`), compile: `cargo build --release -p harness-cli`.
2. Rode `./target/release/dsh doctor` e confira a seção "sandbox": quais backends passaram (`bubblewrap`, `landlock`) e qual cofre está em uso.
3. Exporte uma chave: `export GEMINI_API_KEY=...`.
4. `./target/release/dsh -p "rode: echo ok > dentro.txt; echo x > ~/fora.txt"`.
   - Esperado: `dentro.txt` criado; a escrita em `~/fora.txt` é negada com a dica de escalonamento.
5. Peça ao agente para rodar `env`. A chave não pode aparecer na saída.
6. Peça um `sleep 600 &` e cancele o turno (Ctrl+C na TUI). `pgrep sleep` não pode mostrar nada.
7. Sem `bwrap` instalado: `sudo apt remove bubblewrap` e repita o passo 4. O backend passa a ser `landlock`.

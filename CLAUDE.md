# Fluxo de trabalho com agentes

Este projeto usa um loop de agentes controlado por `loop.sh`. Quando receber uma volta do loop, siga exatamente esta ordem, usando os subagentes:

1. **analista**: somente se `.loop/contexto.md` não existir.
2. **developer**: implementa a TAREFA ou corrige o que está em `.loop/feedback.md`.
3. **tester**: escreve e roda os testes e mantém `.loop/gate.sh`.
4. **reviewer**: faz a revisão crítica e escreve `.loop/review.md`.
5. **finalizador**: escreve `.loop/veredito.md` (e `.loop/RELATORIO.md` se for ENTREGUE).

Regras gerais:
- `TAREFA.md` é a fonte da verdade sobre o que foi pedido e os critérios de aceite.
- Nenhum agente edita `.loop/gate.frozen.sh`.
- Nada de commit, push, deploy, publicação ou requisições a sistemas externos. Quem faz o commit é o `loop.sh`.
- Se algo for ambíguo, não invente: o finalizador marca BLOQUEADO e explica o motivo.

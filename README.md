# Loop de agentes para o Claude Code

Um kit para o Claude Code implementar uma tarefa sozinho, em voltas, até que **três sinais independentes concordem** que ela está pronta:

1. o **gate** (um script de testes congelado) passa,
2. o **reviewer** aprova,
3. o **finalizador** declara `ENTREGUE`.

Quem decide se acabou é o `loop.sh`, não os agentes. Assim, nenhum agente consegue "se aprovar" sozinho.

Foi usado para migrar o CorteCerto do web Angular para Flutter (3 fases, 502 testes no fim).

## Como funciona

```
                 ┌───────────── uma vez ─────────────┐
                 │  analista → .loop/contexto.md     │
                 └───────────────────────────────────┘
                                  │
        ┌─────────────────────────▼──────────────────────────┐
        │ developer → tester → reviewer → finalizador         │  ← uma volta
        └─────────────────────────┬──────────────────────────┘
                                  │
                loop.sh roda o gate congelado e lê:
                gate = 0?  review = APROVADO?  veredito = ENTREGUE?
                                  │
          os 3 sim → commit e fim │ BLOQUEADO → para e chama você
          senão → monta .loop/feedback.md e começa outra volta
```

Cada agente lê os arquivos do anterior em `.loop/` e escreve o seu. Os formatos com primeira linha fixa (`APROVADO`/`REPROVADO`, `STATUS: ...`) são o que o `loop.sh` confere.

## Os agentes

Ficam em `.claude/agents/`. As instruções foram refinadas com base nas skills de [rubenmarcus/agent-skills](https://github.com/rubenmarcus/agent-skills) (MIT; veja os créditos no fim).

### analista — entende o projeto e torna o pedido verificável
Roda **uma vez**, antes de tudo. Não altera nada.
- Registra o ponto de partida: branch, HEAD e o resultado dos testes **antes** de qualquer mudança (baseline).
- Descobre o tipo de app, stack, comandos que funcionam de verdade, testes existentes (e o que eles **não** cobrem), arquitetura e contratos.
- Lê o `TAREFA.md` separando requisito de sugestão. Para cada critério de aceite, escreve o resultado observável e **como verificar**.
- Lista as "Decisões em aberto" que mudam escopo ou comportamento. O finalizador usa essa lista para pedir sua decisão em vez de inventar.
- Saída: `.loop/contexto.md`. *Base: `spec-to-plan`.*

### developer — implementa em passos pequenos e validados
- Lê o pedido, o contexto e o feedback da volta anterior (que tem prioridade). Lê o histórico para não repetir o que já falhou.
- Confere branch e `git status` antes de editar e preserva o que não é seu.
- Faz **uma mudança focada por vez**: hipótese → mudança → menor verificação útil → código de saída → manter, revisar ou descartar.
- Em bug: segue o caminho exato do problema, **reproduz antes de corrigir** e corrige a causa. Se não conseguiu reproduzir, diz que a correção é só proposta.
- Não altera teste para passar e não mexe no gate. Se o mesmo erro se repete sem ideia nova, para.
- Saída: `.loop/dev.md`. *Base: `validated-coding-loop` + `evidence-driven-debugging`.*

### tester — prova que os testes pegam defeitos de verdade
- Escolhe as ferramentas pela stack (JUnit/Spring, Jest/Vitest, pytest, Playwright, flutter test/integration_test, auditoria de dependências).
- Escreve testes para **cada** critério de aceite e sabe dizer **que defeito** cada um pega. Testa o efeito em execução, não a declaração.
- **Controle negativo:** aplica uma quebra proposital no código, confirma que o teste falha e desfaz a quebra, conferindo com `git diff`. Teste que não falha com o defeito não prova nada.
- Mantém o `.loop/gate.sh`, que sai com 0 só se tudo passar. **Teste pulado, timeout ou etapa que não rodou fazem o gate falhar**, nunca viram um "passou".
- Saída: `.loop/testes.md` e `.loop/gate.sh`. *Base: `quality-gates`.*

### reviewer — tenta derrubar a entrega
Não edita código.
- Lê o **diff real antes** do resumo do developer, para formar opinião sem influência.
- Procura modos de falha concretos: chamador esquecido, autorização, entrada não confiável, concorrência, estado desatualizado, teste que mede a coisa errada, teste enfraquecido, mudança fora do escopo.
- Cada achado traz local, gatilho, esperado x obtido, impacto, menor correção e se foi **DEMONSTRADO** ou é só **SUSPEITA**. Não inventa achado para preencher a lista.
- Saída: `.loop/review.md`, com a primeira linha `APROVADO` ou `REPROVADO`. *Base: `critical-adversarial-review`.*

### finalizador — dá o veredito com base em evidência
Não edita código.
- Trata o que os outros agentes escreveram como **hipótese** e confere sozinho as afirmações que decidem o veredito, sempre no HEAD atual.
- Teste pulado, timeout ou artefato ausente = **DESCONHECIDO**, nunca OK. Dois agentes concordando não conta como medição.
- Decide `ENTREGUE`, `CONTINUAR` ou `BLOQUEADO` (quando precisa de você).
- Em `ENTREGUE`, escreve o `RELATORIO.md`, incluindo **o que ainda precisa de aprovação visual ou humana** e as decisões que os agentes tomaram sem você.
- Saída: `.loop/veredito.md`, `.loop/RELATORIO.md` e uma linha em `.loop/historico.md`. *Base: `multi-agent-collaboration`.*

## Os outros arquivos

| Arquivo | Para que serve |
|---|---|
| `loop.sh` | Orquestra as voltas, congela o gate na primeira vez, roda o gate, confere os três sinais, detecta loop travado (mesmo erro 3x) e faz o commit no fim |
| `CLAUDE.md` | Diz ao Claude a ordem dos agentes e as regras (nada de push/deploy, não editar o gate congelado, BLOQUEADO se for ambíguo) |
| `TAREFA.md` | Modelo do pedido: o que quero, critérios de aceite, fora do escopo, restrições. **É a parte que mais importa** |
| `COMO-USAR.md` | Instalação, permissões, códigos de saída e notificação no celular |

## Uso rápido

1. Copie `.claude/agents/`, `CLAUDE.md`, `TAREFA.md` e `loop.sh` para a raiz do seu projeto.
2. Escreva o pedido e os critérios de aceite no `TAREFA.md` e faça um commit.
3. Rode `./loop.sh 8` (8 = máximo de voltas). No Windows, use WSL ou Git Bash.
4. Na primeira volta, **revise `.loop/gate.frozen.sh`**: é ele que define o que conta como "deu certo".

Sem o `loop.sh`, dá para pedir dentro do chat do Claude Code: *"entre no loop e siga o fluxo do CLAUDE.md, no máximo 6 voltas"*.

Detalhes, códigos de saída e permissões estão em [`COMO-USAR.md`](COMO-USAR.md).

## Créditos

Partes das instruções dos agentes foram traduzidas e adaptadas de [rubenmarcus/agent-skills](https://github.com/rubenmarcus/agent-skills) (commit `e1aa118`), sob a licença MIT, Copyright (c) 2026 Ruben Marcus. O texto completo da licença está em [`COMO-USAR.md`](COMO-USAR.md#créditos).

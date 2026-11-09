# Tarefa

## O que quero
<!-- Descreva o pedido. Exemplo: -->
Criar um endpoint `POST /api/favoritos` que permite ao usuário logado favoritar um produto.

## Critérios de aceite
<!-- Cada item deve ser verificável por um teste. -->
- [ ] Usuário autenticado consegue favoritar um produto existente (retorna 201).
- [ ] Favoritar o mesmo produto duas vezes não duplica o registro (retorna 200).
- [ ] Produto inexistente retorna 404.
- [ ] Sem autenticação retorna 401.
- [ ] Um usuário não consegue ver nem alterar os favoritos de outro usuário.

## Fora do escopo
- Tela no frontend.

## Restrições
- Seguir o padrão de controllers/services já existente.

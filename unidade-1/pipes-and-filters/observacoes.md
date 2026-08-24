# Observações — Experimento 2: Pipes and Filters

## Condição alterada

No `main.py`, o critério `salario_maximo` da `Vaga` foi alterado de `18_000.0`
para `25_000.0`.

```diff
- salario_maximo=18_000.0,
+ salario_maximo=25_000.0,  # [MODIFICADO] era 18_000.0
```

## O que a saída revelou

- **Antes:** o pipeline aprovava 3 candidatos (Ana Lima, Elena Souza, Diego
  Faria). Clara Mendes era **reprovada** pelo filtro `FiltroPorPretensaoSalarial`
  com a mensagem `pretensão R$22,000 > máximo R$18,000`.
- **Depois:** com o orçamento elevado para R$25.000, Clara Mendes deixou de ser
  descartada nesse filtro e passou a compor o relatório final, elevando o total
  para **4 candidatos aprovados**. Ela entrou no ranking na 3ª posição, com
  score de 75% (3 de 4 habilidades compatíveis) — o mesmo score de Diego
  Faria, que foi reordenado para a 4ª posição.
- As mensagens `[DESCARTADO]` (currículo id=3, nome ausente) e `[REPROVADO]
  Bruno Rocha` (experiência insuficiente) permaneceram idênticas, pois nenhum
  desses dois critérios foi alterado.

## Responsabilidade arquitetural

O experimento evidencia que cada filtro (*tester*) do pipeline é responsável
por **um único critério de rejeição**, de forma independente e sem estado
compartilhado — `FiltroPorPretensaoSalarial` só conhece o limite salarial da
vaga, não sabe nada sobre experiência ou habilidades. Alterar apenas o
parâmetro `salario_maximo` (passado a esse filtro via seu construtor) mudou o
resultado de exatamente um estágio do pipeline, sem qualquer efeito sobre os
demais filtros (`ValidadorDeCurriculo`, `FiltroPorExperienciaMinima`) nem sobre
o cálculo de score, que só passa a atuar sobre quem sobrevive aos testers
anteriores. Isso demonstra a composição do estilo Pipes and Filters: estágios
desacoplados, encadeados pelo `framework.Pipeline`, onde o resultado de um
filtro é puramente o input transformado/filtrado do anterior.

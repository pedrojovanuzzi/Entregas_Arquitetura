# Nota comparativa — três perspectivas sobre a mesma API

A oficina usa três ferramentas que parecem fazer a mesma coisa ("verificar a API"),
mas cada uma olha para um objeto diferente. A tabela abaixo separa o que cada uma
examina e, principalmente, o que cada uma **não** consegue enxergar.

| Ferramenta | O que ela examina | Objeto de estudo | O que ela **não** vê |
|---|---|---|---|
| **Spectral** | O documento `openapi.yaml` | O contrato, como texto | Se a aplicação implementa o que o documento promete |
| **pytest + TestClient** | A aplicação em execução, dentro do processo | A implementação | Se o consumidor real consegue usar (rede, cliente HTTP, cabeçalhos) |
| **Bruno** | O tráfego HTTP real contra o servidor | A experiência do consumidor | Se o documento publicado descreve o que aconteceu |

## As três evidências desta entrega

- **Spectral** (`spectral-valido.txt`): `No results with a severity of 'error' found!`,
  código de saída 0. O documento é internamente coerente — os exemplos casam com os
  schemas, toda operação tem `operationId`, `description` e `tags`.
- **pytest** (`testes-contrato.txt`): `7 passed`. A implementação responde 202 com
  `Location`, devolve 422 estruturado, recusa propriedade não contratada e o
  contrato explícito concorda com o gerado nos pontos verificados.
- **Bruno** (`bruno-execucao.txt`): 5 requisições, **12/12 asserções**, contra o
  servidor Uvicorn de verdade, pela rede local. O consumidor seguiu o `Location`
  devolvido em vez de montar a URL por conta própria.

## A falha deliberada não é um defeito

Em `openapi-experimento.yaml` o exemplo `pedidoValido` teve o `cpf` trocado de
`'12345678901'` para `'123'`. O schema `PedidoElegibilidade` **não** foi alterado —
ele continua exigindo `^\d{11}$`. O Spectral então acusa:

```
35:24  error  oas3-valid-media-example  "cpf" property must match pattern "^\d{11}$"
✖ 1 problem (1 error, 0 warnings, 0 infos, 0 hints)   → código de saída 1
```

O ponto do experimento não é "quebrar" o contrato, e sim mostrar **onde** cada
ferramenta captura o mesmo erro e **a que custo**:

- O **Spectral** rejeitou em tempo de documento, sem subir servidor, sem rede,
  em milissegundos, e devolveu o caminho YAML exato do problema.
- O **servidor** rejeita o mesmo dado em tempo de execução — confirmado no
  Experimento "EXTRA A" de `respostas-http.txt` e na requisição `04` da coleção
  Bruno: `422`, `tipo: string_pattern_mismatch`.

Ou seja: o dado inválido seria pego de qualquer jeito, mas o linter pega antes de
existir consumidor, e o servidor só pega depois que alguém já tentou usar. Um
exemplo errado publicado na documentação é pior que um bug de servidor, porque
ensina o consumidor a construir a requisição errada — e o consumidor confia no
exemplo, não no schema.

## O que só a combinação das três revela

Nenhuma ferramenta sozinha detecta uma divergência entre documento e aplicação.
Rodando o Spectral sobre o contrato **gerado** pela própria aplicação
(`spectral-gerado.txt`), aparecem 4 erros que o contrato escrito à mão não tem:

```
error  operation-description  paths./elegibilidades.post
error  operation-tags         paths./elegibilidades.post
error  operation-description  paths./elegibilidades/{protocolo}.get
error  operation-tags         paths./elegibilidades/{protocolo}.get
```

O documento publicado promete `tags: [Elegibilidades]` e uma `description` por
operação; a aplicação não declara nenhum dos dois. O Spectral sozinho aprovou
(olhou só o documento escrito à mão). O pytest sozinho aprovou (não compara esses
campos). O Bruno sozinho aprovou (tags não afetam o tráfego HTTP). A divergência
só apareceu quando se cruzou uma ferramenta com o artefato da outra.

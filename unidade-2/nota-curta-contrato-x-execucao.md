# Nota curta — contrato explícito, contrato gerado e execução

Três descrições da mesma API, produzidas por caminhos diferentes.

| | **Contrato explícito** | **Contrato gerado** | **Execução** |
|---|---|---|---|
| Origem | `contratos/openapi.yaml`, escrito à mão | `app.openapi()`, derivado do código | Servidor Uvicorn respondendo HTTP |
| Evidência | `spectral-valido.txt` | `openapi-gerado.yaml`, `spectral-gerado.txt` | `respostas-http.txt`, `bruno-execucao.txt` |
| Veredito | 0 erros de lint | **4 erros** de lint | 202 / 200 / 422 / 404 conforme prometido |

## Onde os três concordam

Caminhos, verbos, `required` de `PedidoElegibilidade`, schema do `422` (`ErroAPI`),
`description` e header `Location` do `202`. A execução confirma tudo isso:
`POST` devolveu `202` com `Location`, e o `GET` naquele caminho devolveu `200` com
corpo idêntico ao do `POST`.

Essa concordância não é acidente: é o que
`test_application_and_explicit_contract_agree_on_operations_and_models` exige.

## Onde divergem

**O gerado não declara `tags` nem `description`.** O explícito declara as duas.
Como as regras do projeto promovem `operation-tags` e `operation-description` a
`error`, o contrato publicado passa no Spectral e o contrato que a aplicação
realmente descreve reprova com 4 erros. É a única divergência que importa: se um
dia o time servir `/openapi.json` no lugar do arquivo escrito à mão, o portal
perde o agrupamento e o lint quebra. Correção: `tags=["Elegibilidades"]` e
`description=` nos decoradores.

**O gerado documenta um `422` a mais no `GET` e dois schemas órfãos**
(`HTTPValidationError`, `ValidationError`). São resíduo do FastAPI: o parâmetro
`protocolo` é `str` sem restrição, então esse `422` nunca dispara — a execução
confirma, `GET /elegibilidades/protocolo-inexistente` devolveu `404`. E o handler
customizado faz toda resposta de erro sair como `ErroAPI`, nunca nesses dois
schemas. São diferenças de descrição, não incompatibilidade.

## O que só a execução mostrou

Rodar a **coleção recém-importada** do contrato (`bruno-importada-execucao.txt`)
deu:

```
Aceita uma consulta de elegibilidade  (202 Accepted)
Consulta uma elegibilidade aceita     (404 Not Found)
```

O `POST` funciona de primeira, porque o exemplo do contrato é diretamente
utilizável. O `GET` falha porque `:protocolo` chegou vazio — nenhum documento pode
carregar um valor que só existe em tempo de execução. É exatamente por isso que a
resposta traz `Location`: o consumidor pega dali, em vez de montar a URL. A
coleção estendida (`bruno/`) faz esse encadeamento e fecha 12/12 asserções.

## A falha deliberada

`openapi-experimento.yaml` é uma cópia de estudo com o exemplo `pedidoValido`
alterado para `cpf: '123'`, mantendo o schema intacto. O Spectral acusa
`oas3-valid-media-example` e sai com código 1 — antes de existir servidor,
consumidor ou rede. O mesmo dado também é recusado em execução (`422`,
`string_pattern_mismatch`, requisição `04` do Bruno).

Não é defeito pendente: é o instrumento de medida da oficina. Serve para mostrar
que um exemplo errado publicado é pior que um bug de servidor, porque ensina o
consumidor a montar a requisição errada — e o consumidor lê o exemplo, não o
schema. O contrato de produção (`contratos/openapi.yaml`) permanece íntegro e com
lint limpo.

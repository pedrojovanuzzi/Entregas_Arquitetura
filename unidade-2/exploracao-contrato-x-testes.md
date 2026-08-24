# Exploração em dupla — o que o contrato promete x o que os testes comprovam

Exercício da Fase 1: uma leitura parte só do `openapi.yaml` e prevê as respostas;
a outra parte só de `tests/test_api_contract.py` e lista o que é de fato
comprovado. Abaixo, as duas listas e a interseção.

## Lendo só o OpenAPI — o que ele promete

| Promessa | Onde está no documento |
|---|---|
| `POST /elegibilidades` devolve `202` | `paths./elegibilidades.post.responses.202` |
| A resposta `202` traz `Location` obrigatório | `responses.202.headers.Location.required: true` |
| `situacao` é sempre o literal `recebida` | `ElegibilidadeAceita.situacao.const` |
| `cpf` tem exatamente 11 dígitos | `PedidoElegibilidade.cpf.pattern: ^\d{11}$` |
| Nenhuma propriedade extra é aceita | `additionalProperties: false` |
| Erro de entrada devolve `422` com `ErroAPI` | `responses.422` |
| Protocolo desconhecido devolve `404` | `paths./elegibilidades/{protocolo}.get.responses.404` |
| As operações pertencem à tag `Elegibilidades` | `tags: [Elegibilidades]` |
| O servidor da oficina é `http://127.0.0.1:8000` | `servers[0].url` |

## Lendo só os testes — o que eles comprovam

| Comprovação | Teste |
|---|---|
| `202` + `situacao: recebida` + `criado_em` parseável | `test_post_accepts_request_and_get_recovers_it` |
| `Location` == `/elegibilidades/{protocolo}` e o GET nele devolve o mesmo corpo | idem |
| `cpf` ausente → `422`, `codigo: dados_invalidos`, detalhe `body.cpf` | `test_missing_cpf_returns_structured_422_error` |
| Propriedade não contratada → `422` com detalhe apontando o campo | `test_request_rejects_property_forbidden_by_explicit_contract` |
| Protocolo desconhecido → `404` com corpo exato | `test_unknown_protocol_returns_structured_404_error` |
| `/health/live` e `/health/ready` se distinguem | `test_health_endpoints_distinguish_...` |
| O contrato declara os caminhos, schemas e exemplos válidos | `test_explicit_openapi_contract_defines_paths_schemas_and_valid_examples` |
| Contrato explícito e gerado concordam em `required`, `description` e `Location` do 202 | `test_application_and_explicit_contract_agree_on_operations_and_models` |

## Registro pedido pela oficina

### Uma promessa documentada que **não** é verificada

> **`tags: [Elegibilidades]`.**

O `openapi.yaml` declara essa tag nas duas operações. Nenhum teste a compara com a
aplicação — e a aplicação **não** a declara. Confirmado em
`contrato-explicito-x-gerado.txt`:

```
POST /elegibilidades
   explícito : ['Elegibilidades']
   gerado    : (nenhuma)
```

Escolhi esta e não outra porque ela não é apenas "não testada": ela está de fato
**quebrada**, e ninguém percebe. A consequência aparece em `spectral-gerado.txt`,
onde o contrato gerado pela aplicação reprova em `operation-tags`.

Uma segunda promessa não verificada, mais inofensiva: `servers[0].url`. Nenhum
teste confirma que a API atende naquele endereço — o `TestClient` fala com o app
em memória, sem passar por rede. Só o Bruno exercitou o endereço real.

### Uma asserção que depende do contrato

> Em `test_explicit_openapi_contract_defines_paths_schemas_and_valid_examples`:
>
> ```python
> example = request_schema["examples"][0]
> response = TestClient(app).post("/elegibilidades", json=example)
> assert response.status_code == 202
> ```

Essa asserção não tem sentido isolada da documentação: ela pega **o exemplo escrito
no `openapi.yaml`** e o envia para a aplicação real, exigindo `202`. É o único
ponto da suíte em que um erro de documentação derruba um teste de implementação —
se alguém editar o exemplo do schema para algo inválido, este teste quebra.

Vale notar o limite: o teste lê `components.schemas.PedidoElegibilidade.examples`,
**não** o exemplo de `requestBody...examples.pedidoValido`. Foi exatamente por isso
que a oficina mandou alterar o segundo e não o primeiro na falha deliberada — o
Spectral pega esse, o pytest não.

## Comparação

Há sobreposição, mas não equivalência:

- **A descrição explica a intenção.** O `openapi.yaml` diz *por que* existe um
  `Location` e o que `recebida` significa. Nenhum teste carrega essa intenção.
- **O teste escolhe amostras.** Ele prova `cpf` ausente e `cpf` com 3 dígitos, não
  os infinitos outros formatos errados possíveis.
- **O código executa muitos casos.** A regex `^\d{11}$` roda contra qualquer
  entrada real, inclusive as que ninguém escreveu como amostra.

Por isso as três frentes não se substituem: o documento é a promessa, o teste é a
amostra que a confere, e o código é o que efetivamente atende o consumidor.

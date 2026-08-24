# Extensão — contrato explícito x contrato gerado

Comparação entre `contratos/openapi.yaml` (escrito à mão, publicado) e
`app.openapi()` (gerado em tempo de execução pelo FastAPI). Nada da API pública
foi alterado: a comparação roda sobre uma cópia de estudo exportada em
`openapi-gerado.yaml`. Saída bruta em `contrato-explicito-x-gerado.txt`.

## Onde os dois concordam

| Aspecto | Resultado |
|---|---|
| Versão OpenAPI | `3.1.0` nos dois |
| Caminhos | idênticos — nenhum sobrando de nenhum lado |
| Status do `POST /elegibilidades` | `202`, `422` nos dois |
| Schema do `422` | `#/components/schemas/ErroAPI` nos dois |
| `required` de `PedidoElegibilidade` | idêntico |
| Header `Location` do `202` | idêntico |
| `description` do `202` | idêntica |

Os quatro últimos são justamente os que
`test_application_and_explicit_contract_agree_on_operations_and_models` verifica.
Concordam porque alguém escreveu um teste para exigir que concordassem.

## Onde os dois divergem

### 1. `tags` — **incompatibilidade**

```
POST /elegibilidades
   explícito : ['Elegibilidades']
   gerado    : (nenhuma)
GET /elegibilidades/{protocolo}
   explícito : ['Elegibilidades']
   gerado    : (nenhuma)
```

O documento publicado promete a tag; o código nunca a declara — nem no decorador
`@app.post(...)`, nem em `@app.get(...)`.

Classifico como incompatibilidade, e não como diferença de descrição, por uma razão
verificável: as duas versões **não passam nas mesmas regras de governança**. O
contrato explícito passa no Spectral do projeto; o gerado reprova:

```
error  operation-tags         paths./elegibilidades.post
error  operation-tags         paths./elegibilidades/{protocolo}.get
```

Se amanhã o time decidir publicar `/openapi.json` (o contrato gerado) em vez do
arquivo escrito à mão — algo comum, justamente para evitar documentação
desatualizada — o portal perde o agrupamento e o pipeline de lint quebra. A
correção é de uma linha por operação: `tags=["Elegibilidades"]` no decorador.

O mesmo vale para `operation-description`: o `openapi.yaml` traz `description` nas
duas operações, o código só traz `summary`. Reprova pela mesma regra.

### 2. `422` extra no `GET` — **diferença de descrição**

```
GET /elegibilidades/{protocolo}
   explícito : ['200', '404']
   gerado    : ['200', '404', '422']
```

O FastAPI adiciona `422` automaticamente a qualquer operação com parâmetro
validado. Aqui o parâmetro é `protocolo: str`, sem `pattern`, sem tamanho — ou
seja, **qualquer** string de caminho é aceita e esse `422` nunca dispara. Está
comprovado em `respostas-http.txt`: `GET /elegibilidades/protocolo-inexistente`
devolveu `404`, não `422`.

É diferença de descrição: o gerado documenta uma resposta que o servidor não
produz. Documentar a mais não quebra consumidor, mas polui — um cliente gerado
passa a ter um ramo de tratamento morto.

### 3. `HTTPValidationError` e `ValidationError` — **diferença de descrição**

```
Somente no gerado : ['HTTPValidationError', 'ValidationError']
```

São os schemas padrão de erro do FastAPI. Ficam órfãos porque `main.py` instala um
handler próprio:

```python
@app.exception_handler(RequestValidationError)
async def tratar_erro_de_validacao(...) -> JSONResponse:
    ...  # devolve o formato ErroAPI
```

Toda resposta de validação real sai no formato `ErroAPI` — confirmado nos três
casos de 422 em `respostas-http.txt` e nas asserções `03` e `04` do Bruno. Os dois
schemas descrevem um formato que a aplicação **nunca** emite.

## Conclusão da extensão

Das três divergências, apenas a primeira precisa de correção de código; as outras
duas são ruído do gerador e se resolvem com anotação (`responses=`,
`response_model=`) ou simplesmente aceitando que o contrato publicado é o escrito
à mão.

O ponto arquitetural: manter **dois** contratos — um escrito, um gerado — cria uma
superfície de divergência silenciosa. As três encontradas aqui passaram por sete
testes verdes, um Spectral limpo e doze asserções Bruno. Só apareceram quando os
dois documentos foram comparados campo a campo. Ou se gera o contrato a partir do
código e se aplica o lint no gerado, ou se escreve o contrato à mão e se testa o
código contra ele — manter os dois sem um teste que os cruze integralmente é
manter uma mentira em potencial.

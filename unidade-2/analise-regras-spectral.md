# Extensão — análise das regras do `.spectral.yaml`

## As regras do projeto

`.spectral.yaml` na raiz do laboratório apenas encadeia o arquivo de contratos:

```yaml
extends:
  - ./contratos/.spectral.yaml
```

E `contratos/.spectral.yaml` traz o conjunto base da OpenAPI mais três regras
promovidas a `error`:

```yaml
extends: spectral:oas

rules:
  operation-operationId: error
  operation-description: error
  operation-tags: error
```

O detalhe importante é a **promoção de severidade**: no `spectral:oas` essas três
regras existem como `warning`. Aqui elas viram `error`, o que muda o código de
saída do processo — e portanto derruba um pipeline de CI, em vez de apenas
imprimir um aviso que todo mundo ignora.

## O que cada regra entrega, e para quem

### `operation-operationId: error`

**Quem ganha:** automação.

O `operationId` é o único identificador estável de uma operação. Geradores de
cliente (`openapi-generator`, `swagger-codegen`) o usam para nomear métodos: com
`operationId: criarElegibilidade`, o SDK gerado expõe `criarElegibilidade()`; sem
ele, o gerador inventa um nome a partir do caminho e do verbo — algo como
`postElegibilidades` — que **muda sozinho** se o caminho mudar. O resultado é que
uma mudança de rota, invisível para o contrato, quebra a compilação de todo cliente
gerado.

Também é o que permite rastrear a mesma operação entre versões do documento: se o
caminho vira `/v2/elegibilidades`, o `operationId` idêntico diz que é a mesma
operação, não uma nova.

### `operation-description: error`

**Quem ganha:** o consumidor humano.

`summary` cabe numa linha de lista; `description` é onde entra o que o consumidor
não consegue deduzir do schema. No contrato desta oficina:

```yaml
description: >-
  Valida o pedido, cria um protocolo efêmero e informa onde consultar o
  recurso aceito.
```

A palavra **efêmero** é a informação crítica, e ela não está em lugar nenhum do
schema. `ElegibilidadeAceita` não tem como dizer "isto some quando o processo
reinicia". Sem essa frase, o consumidor razoavelmente assume durabilidade e
constrói em cima de uma garantia que não existe.

### `operation-tags: error`

**Quem ganha:** os dois — navegação humana e organização de portal.

As tags agrupam operações no Swagger UI, no Bruno e em portais de API. Com duas
operações, parece burocracia. Com quarenta, é a diferença entre um portal
navegável e uma lista plana.

Esta regra é também a que rendeu o achado mais concreto da oficina: rodando o
Spectral sobre o contrato **gerado pela aplicação** (`spectral-gerado.txt`), ela
reprova, porque o código FastAPI nunca declara `tags`. O contrato escrito à mão
passa; o que a aplicação realmente descreve, não. A regra funcionou exatamente
como deveria — só estava sendo aplicada no artefato errado.

## Uma regra semântica que o linter não conseguiria sozinho

> **"O `Location` devolvido no `202` precisa apontar para um recurso que o `GET`
> desta mesma API consiga recuperar."**

O Spectral verifica que `Location` está declarado, que é obrigatório, que é
`type: string` e que o exemplo é uma string. Ele **não** tem como verificar que o
valor produzido em tempo de execução casa com o caminho
`/elegibilidades/{protocolo}` — isso exige executar a aplicação, capturar a
resposta e fazer a segunda chamada. É precisamente o que o
`test_post_accepts_request_and_get_recovers_it` faz:

```python
assert created.headers["location"] == f"/elegibilidades/{body['protocolo']}"
recovered = client.get(created.headers["location"])
assert recovered.status_code == 200
```

Outras regras da mesma natureza, todas fora do alcance de um linter de documento:

- **"`situacao: recebida` é o estado inicial e nunca o final."** O `const: recebida`
  garante o valor, não a semântica de que existe uma transição posterior.
- **"O mesmo pedido enviado duas vezes deve gerar dois protocolos distintos"** —
  ou, se um dia houver idempotência, **o mesmo**. Nenhuma das duas alternativas é
  expressável em OpenAPI 3.1.
- **"`codigo` do `ErroAPI` é estável entre versões."** O schema diz que é uma
  string; que `dados_invalidos` não pode virar `invalid_data` na v1.1 é uma
  política de compatibilidade, não uma restrição de tipo.

A divisão de trabalho, em uma frase: **o linter garante que o documento é
coerente consigo mesmo; só a execução garante que a aplicação é coerente com o
documento.**

# Questões exploratórias

## 1. O que `202` permite ao provedor mudar sem quebrar o consumidor?

O `202 Accepted` promete apenas que **o pedido foi aceito**, não que ele foi
processado. Isso deixa livre tudo que acontece depois do aceite:

- processar de forma síncrona hoje e enfileirar amanhã;
- mudar quanto tempo o processamento leva;
- trocar o worker, o banco, a operadora consultada;
- reprocessar em caso de falha.

Se a operação respondesse `200` com o resultado da elegibilidade no corpo, o
provedor estaria preso a produzir a resposta dentro da requisição — qualquer
mudança para processamento assíncrono viraria quebra de contrato. Com `202`, o
consumidor foi treinado desde o início a **não** esperar o resultado inline: ele
recebe um protocolo e sabe onde perguntar depois.

O que o `202` **não** permite mudar: o formato do `ElegibilidadeAceita`, a
existência do `Location`, e o fato de o protocolo ser consultável.

## 2. Por que `Location` é melhor que pedir ao consumidor montar a URL por convenção?

Porque move a regra de formação da URL para o lado de quem pode mudá-la.

Se o consumidor monta `"/elegibilidades/" + protocolo`, ele passa a depender de um
detalhe que nunca foi contratado: o formato do caminho. O provedor então não pode
mais versionar (`/v2/elegibilidades/...`), mover para outro prefixo, mudar para um
identificador opaco, nem apontar para outro host — sem quebrar todo consumidor que
concatenou string.

Com `Location`, o único contrato é "siga o que eu te devolvi". Na coleção Bruno
isso está explícito: a requisição `01` captura `res.headers.location` numa variável
e a `02` usa essa variável. O consumidor funciona sem nunca saber como a URL é
formada.

Evidência em `respostas-http.txt`:

```
LOCATION : /elegibilidades/098de1ae-6dec-46a4-bf95-7a2833b946a3
→ GET /elegibilidades/098de1ae-6dec-46a4-bf95-7a2833b946a3 → 200 OK
Corpo idêntico ao do POST? True
```

## 3. Qual divergência entre OpenAPI e aplicação os testes atuais ainda não detectam?

Três, todas comprovadas em `contrato-explicito-x-gerado.txt`:

**a) `tags` — a mais grave.** O `openapi.yaml` declara `tags: [Elegibilidades]` nas
duas operações. A aplicação não declara nenhuma. Nenhum teste compara esse campo,
então a divergência passa. A consequência é concreta: rodando o Spectral sobre o
contrato gerado pela aplicação (`spectral-gerado.txt`), ele **reprova** com
`operation-tags` e `operation-description` — 4 erros. O documento publicado passa
nas regras do projeto; o que a aplicação realmente descreve, não.

**b) Um `422` a mais no GET.** A aplicação gera `422` para
`GET /elegibilidades/{protocolo}` (comportamento padrão do FastAPI para validação
de parâmetro de caminho) que o contrato explícito não documenta. Como o parâmetro
é `str` sem restrição, esse `422` na prática nunca dispara — é ruído do gerador,
não incompatibilidade real.

**c) Schemas órfãos.** O contrato gerado carrega `HTTPValidationError` e
`ValidationError` em `components.schemas`, que não existem no contrato explícito.
Também são resíduo do FastAPI: o handler de erro customizado faz a resposta real
usar `ErroAPI`, então esses schemas descrevem algo que o servidor nunca devolve.

**Quais são incompatibilidade e quais são diferença de descrição:** só (a) é
incompatibilidade de verdade, porque afeta a governança — o contrato publicado e o
contrato real reprovariam em regras diferentes. (b) e (c) são diferenças de
descrição: descrevem respostas e tipos que o servidor, na configuração atual, não
produz. Corrigi-las é higiene de documentação; corrigir (a) é alinhar promessa e
implementação.

## 4. Quando uma chave de idempotência passaria a ser necessária?

No momento em que o consumidor puder **repetir** o mesmo `POST` sem saber se o
primeiro chegou. Hoje cada `POST` cria um `uuid4()` novo e incondicional:

```python
aceita = ElegibilidadeAceita(protocolo=str(uuid4()), ...)
```

Então duas chamadas idênticas produzem dois protocolos e duas elegibilidades. Isso
é inofensivo enquanto a operação não custa nada e não persiste. Passa a doer quando:

- houver **timeout de rede**: o cliente não recebeu o `202`, mas o servidor
  processou — reenviar duplica;
- houver **retry automático** em gateway, proxy ou biblioteca cliente;
- o aceite passar a ter **efeito externo** (cobrar da operadora, abrir um caso,
  disparar evento a jusante) — aí a duplicata vira problema de negócio, não de dado.

A forma usual seria um cabeçalho `Idempotency-Key` fornecido pelo consumidor, com o
servidor guardando a associação chave → protocolo e devolvendo **o mesmo** `202` e
o mesmo `Location` na repetição.

## 5. Que parte do experimento deixaria de funcionar com duas instâncias e memória separada?

A recuperação pelo `GET` — ou seja, exatamente o passo `02` da coleção Bruno e a
segunda metade do Experimento 3.

O estado vive num dicionário de processo:

```python
_elegibilidades: dict[str, ElegibilidadeAceita] = {}
```

Com duas instâncias atrás de um balanceador, o `POST` grava na memória da instância
A e o `GET` seguinte pode cair na instância B, que não conhece aquele protocolo e
responde `404 elegibilidade_nao_encontrada` — o mesmo corpo do "EXTRA B" em
`respostas-http.txt`, só que agora como falso negativo.

O que **continuaria** funcionando sem alteração: o `POST` (202 e `Location`), toda
a validação de entrada (422 por `cpf` ausente, por padrão inválido e por
propriedade extra), o Spectral e os testes com `TestClient` — porque nenhum deles
depende de estado compartilhado entre requisições.

A correção não é arquitetural-milagrosa: é tirar o estado do processo (banco,
cache distribuído) ou grudar a sessão na instância — sendo a primeira a única que
sobrevive à perda de uma instância.

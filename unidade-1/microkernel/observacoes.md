# Observações — Experimento 3: MicroKernel

## Condição alterada

No `main.py`, o estado do cliente da Fatura #1002 (Distribuidora Rio) foi
alterado de `"RJ"` para `"SP"` — o nome e os demais dados do cliente
permaneceram os mesmos (apenas a composição/dado de entrada mudou).

```diff
- estado="RJ", tipo="pj", email="nf@distribrj.com",
+ estado="SP", tipo="pj", email="nf@distribrj.com",
```

## O que a saída revelou

- **Antes:** o núcleo aplicava o plugin `ICMS-RJ` à fatura #1002 (o único
  plugin de imposto cujo `processar()` reconhece `estado == "RJ"`), gerando
  `ICMS-RJ: R$1.680,00` e total `R$10.080,00`.
- **Depois:** com o cliente agora em `SP`, o plugin `ICMS-RJ` não reconheceu
  mais a fatura (retornou o resultado inalterado) e, em vez dele, o plugin
  `ICMS-SP` passou a processá-la, gerando `ICMS-SP: R$1.152,00` — um valor e
  uma alíquota diferentes — e total `R$9.552,00`. O plugin `ISS-SP` continuou
  sem incidir, pois nenhum item da fatura tem categoria `"servico"`. O valor
  do frete não mudou (R$0,00) porque o valor bruto (R$8.400,00) segue acima do
  teto de isenção de R$5.000,00, então a regra de isenção do
  `FreteCorrespondenciaPlugin` prevaleceu independentemente do estado.
- As faturas #1001, #1003 e #1004 permaneceram idênticas, pois não foram
  tocadas.

## Responsabilidade arquitetural

O experimento demonstra a essência do estilo MicroKernel: o **núcleo**
(`CoreFaturamento.emitir`) não contém nenhuma regra fiscal — ele apenas
percorre os plugins registrados na categoria `"impostos"`, na ordem definida
por `ORDEM_CATEGORIAS`, e cada plugin decide por si mesmo, olhando o dado de
entrada (`fatura.cliente.estado`), se deve ou não atuar. Trocar o estado do
cliente foi suficiente para ativar um plugin diferente (`ICMS-SP` no lugar de
`ICMS-RJ`) sem qualquer alteração no núcleo, no registro de plugins ou na
ordem de execução — apenas o dado de entrada mudou qual "peça plugável" da
composição respondeu à requisição. Isso confirma que a extensibilidade do
MicroKernel depende inteiramente de cada plugin reconhecer (ou ignorar) o caso
que lhe compete, sem que o núcleo precise saber quantos ou quais plugins de
imposto existem.

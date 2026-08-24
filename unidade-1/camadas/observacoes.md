# Observações — Experimento 1: Estilo em Camadas

## Condição alterada

No `main.py`, cenário 2 ("Tentando agendar com CONFLITO de horário"), o horário da
nova solicitação de consulta para a paciente Lúcia Ferreira (id 3) com a Dra. Ana
Silva (médico id 1) foi alterado de `09:15–09:45` (que se sobrepõe à consulta já
existente das `09:00–09:30`) para `11:00–11:30`, um horário livre na agenda da
médica naquele dia.

```diff
- "inicio": "2025-06-10T09:15:00", "fim": "2025-06-10T09:45:00",
+ "inicio": "2025-06-10T11:00:00", "fim": "2025-06-10T11:30:00",
```

## O que a saída revelou

- **Antes:** a requisição retornava `HTTP 409 CONFLICT`, com a mensagem de erro
  informando o horário já ocupado pela Dra. Ana Silva.
- **Depois:** a mesma requisição, agora em horário livre, retornou `HTTP 201
  CREATED`, criando a consulta `id=4`. Essa nova consulta passou a aparecer
  corretamente nas listagens seguintes: na "Agenda do dia" (item 3) e, após o
  cancelamento da consulta `id=2`, também na "Agenda após alterações" (item 7),
  já que era a única consulta ainda `agendada` para aquele médico/dia.
- Nenhum outro trecho da saída mudou — os cenários de realizar, cancelar e
  histórico do paciente continuaram idênticos, pois dependem de outras
  consultas (`id=1` e `id=2`), não afetadas pela alteração.

## Responsabilidade arquitetural

A alteração demonstra que a **regra de negócio de conflito de agenda** (o
invariante "um médico não pode ter dois horários sobrepostos") está
centralizada exclusivamente na **camada de negócios** (`AgendamentoServico.agendar`,
em `servicos.py`, através de `Horario.conflita_com`). A camada de apresentação
(`AgendaController`) apenas traduz o resultado dessa regra em códigos HTTP
(`409` para `ConflitodeAgendaError`, `201` para sucesso) — ela não decide se há
conflito, apenas formata a resposta. Isso confirma a separação de
responsabilidades do estilo em camadas: mudar um dado de entrada (o horário)
foi suficiente para observar a regra de negócio em ação, sem tocar em nenhuma
linha de código de validação.

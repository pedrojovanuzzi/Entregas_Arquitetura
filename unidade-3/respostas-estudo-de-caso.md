# Estudo de caso: quando a fragmentação supera a autonomia

**Aluno:** Pedro Artur Jovanuzzi
**Disciplina:** Arquitetura de Software, Módulo 3: Serviços
**Caso:** plataforma hospitalar, os onze processos de Elegibilidade e Autorização

## 1. Qual forma de acoplamento mais atrasava a equipe?

O acoplamento de implantação. O sintoma é direto: uma nova regra de autorização obrigava a alterar contratos de quatro processos e implantar seis deles na mesma janela.

Escolho essa porque ela cobra pedágio em toda mudança. Não dava para entregar uma regra sem montar janela, alinhar a ordem de publicação de seis unidades e ter plano de volta para todas. Coordenar a entrega passou a custar mais do que escrever a regra. Além disso, é o acoplamento que nega a promessa de dividir em processos: a equipe pagava rede, falha parcial e onze pipelines, mas continuava com uma entrega só.

As outras três formas estão ligadas a essa. O acoplamento de contrato é a origem, porque cada processo expunha uma fatia do mesmo assunto. O temporal aparecia na tela de atendimento, com oito chamadas em sequência e erro genérico quando qualquer uma atrasava. O organizacional era o menos custoso, já que era uma equipe só negociando consigo mesma.

## 2. Qual critério revelou que quatro processos eram um único bounded context?

O critério das regras, o que pergunta se elas mudam sob a mesma política. A frase do caso que sustenta a escolha está no diagnóstico:

"Vínculo, vigência, categoria de plano e regra contratual sempre participavam da mesma decisão, 'este beneficiário está elegível?', e mudavam sob as mesmas políticas: esse conjunto formava, na prática, um único bounded context."

A segunda metade da frase aplica o critério quase com as mesmas palavras da definição. Quando a operadora muda a política de vigência ou de categoria de plano, as quatro partes se mexem, porque existem para responder à mesma pergunta. Separadas, cada uma guardava um pedaço de uma decisão que nunca foi de nenhuma delas sozinha. O critério de mudanças conjuntas confirma o mesmo pelo histórico, e é o mais fácil de observar sem discutir modelo.

## 3. Por que uma consolidação preserva as fronteiras e a outra não?

A escolhida junta quatro partes que já pertenciam ao mesmo bounded context, então ela apenas faz a unidade implantável coincidir com a fronteira lógica que o mapa de capacidades encontrou, com módulos internos e testes de arquitetura impedindo um módulo de ler as tabelas do outro.

A descartada juntaria Elegibilidade, Autorização e Auditoria, que são três contextos diferentes, e misturaria a autoridade de decisão de cada um: Elegibilidade passaria a decidir por Autorização sem contrato nenhum, e a auditoria, que só deveria receber fatos, entraria no caminho síncrono do atendimento.

## 4. Por que definir as três regras antes de ligar a mensageria?

Escolhi o comportamento de repetição, a garantia de que a mesma mensagem entregue duas vezes não produz o efeito duas vezes.

Sem essa regra, a trilha de auditoria mostraria o mesmo fato registrado mais de uma vez. Reentrega acontece por motivos banais, como o consumidor cair antes de confirmar ou o serviço reiniciar durante uma implantação. A trilha ficaria com duas linhas para uma única autorização, com identificadores e horários diferentes, e ninguém saberia dizer se foram duas entregas da mesma coisa ou dois eventos de verdade.

O estrago é grande porque a auditoria é o lugar onde se busca prova. Qualquer contagem tirada dali fica suspeita, a conferência com o que Autorização registrou nunca fecha, e num incidente a equipe gasta tempo decidindo em que linha acreditar. Como reentrega é o comportamento normal de mensageria com entrega ao menos uma vez, tratar repetição é requisito, e não precaução exagerada.

## 5. Que dado tornaria visível um sinal de revisão?

Peguei o segundo sinal, o de uma regra passar a precisar de um ciclo de implantação isolado.

O dado é a origem de cada implantação do macrosserviço, ou seja, quais módulos foram tocados na mudança que gerou aquela publicação. Com isso dá para calcular quantas publicações do período existiram por causa de um módulo só. Se a regra contratual começar a responder sozinha por uma parcela crescente delas, é sinal de que tem ritmo próprio e está sendo carregada junto com o resto sem precisar.

Vale acompanhar junto o tempo entre a mudança ficar pronta e entrar no ar, separado por módulo. Se os dois números sobem para o mesmo módulo, o sinal apareceu. Um ou dois meses de acompanhamento já separam uma temporada de mudanças contratuais de uma tendência real.

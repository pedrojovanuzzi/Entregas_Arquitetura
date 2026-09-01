# Estudo de caso: quando a fragmentação supera a autonomia

**Aluno:** Pedro Artur Jovanuzzi
**Disciplina:** Arquitetura de Software, Módulo 3: Serviços
**Caso:** plataforma hospitalar, os onze processos de Elegibilidade e Autorização

## 1. Qual forma de acoplamento mais atrasava a equipe?

As quatro formas apareciam ao mesmo tempo, mas a que mais pesava no dia a dia era o acoplamento de implantação. O sintoma está na primeira frase da seção: adicionar uma nova regra de autorização exigia alterar contratos de quatro processos e implantar seis deles na mesma janela.

Escolho essa porque ela cobra pedágio em toda mudança, por menor que seja. Não existia como entregar uma regra nova sem montar uma janela, alinhar a ordem de publicação de seis unidades e ter plano de volta para todas elas. O trabalho de coordenar a entrega passou a custar mais do que escrever a regra em si, e é exatamente isso que consome o dia da equipe.

Tem também o fato de que esse acoplamento nega a principal promessa de dividir em processos. Se a equipe está pagando o preço da distribuição, ou seja, rede, latência, falha parcial e mais pipelines para manter, o mínimo que deveria receber em troca é poder publicar cada unidade quando quiser. Não recebia. Eram onze repositórios e onze pipelines, mas uma entrega só.

As outras três formas continuam sendo problema, e vale reconhecer que estão ligadas. O acoplamento de contrato é a origem: como cada processo expunha uma fatia do mesmo assunto, uma regra nova mexia no contrato de quatro deles, e é isso que obriga as seis implantações. O temporal aparece do lado do usuário, com a tela de atendimento fazendo oito chamadas em sequência e mostrando erro genérico quando qualquer uma atrasava. O organizacional era o menos custoso dos quatro, justamente porque era uma equipe só; o desenho pedia negociação entre times que não existiam, e ela virava reunião interna e retrabalho. Mesmo assim, o que aparecia no calendário toda semana era a janela de implantação.

## 2. Qual critério revelou que quatro processos eram um único bounded context?

O critério das regras, aquele que pergunta se as regras mudam sob a mesma política. A frase do caso que sustenta a escolha está no diagnóstico:

"Vínculo, vigência, categoria de plano e regra contratual sempre participavam da mesma decisão, 'este beneficiário está elegível?', e mudavam sob as mesmas políticas: esse conjunto formava, na prática, um único bounded context."

A segunda metade da frase é o critério aplicado quase com as mesmas palavras da definição. Quando a operadora muda a política de vigência ou de categoria de plano, as quatro partes se mexem, porque as quatro existem para responder à mesma pergunta de negócio. Um bounded context é o limite dentro do qual o modelo tem significado consistente, e aqui o significado de "elegível" só fecha com os quatro juntos. Separados, cada processo guardava um pedaço de uma decisão que nunca foi de nenhum deles sozinho.

O critério de mudanças conjuntas confirma o mesmo diagnóstico por outro caminho, e é o mais fácil de observar sem discutir modelo: bastava olhar o histórico e ver que as alterações sempre chegavam em bloco, o que é a razão de precisar implantar seis processos por causa de uma regra. Linguagem e propriedade dos dados também apontavam para o mesmo lugar, mas de forma menos direta, porque termo repetido em serviços vizinhos é comum e nem sempre significa fronteira errada.

## 3. Por que uma consolidação preserva as fronteiras e a outra não?

A consolidação escolhida junta vínculo, vigência, categoria de plano e regra contratual, que já eram quatro partes do mesmo bounded context, então ela só faz a unidade implantável coincidir com a fronteira lógica que o mapa de capacidades tinha encontrado. Nada que era separado em significado foi misturado; o que era um só voltou a ser um só, agora em módulos internos, com testes de arquitetura impedindo um módulo de ler as tabelas do outro.

A consolidação descartada juntaria Elegibilidade, Autorização e Auditoria, que são três contextos diferentes, e misturaria a autoridade de decisão de cada um. Elegibilidade passaria a poder decidir por Autorização, sem passar por contrato nenhum, e o registro de auditoria, que tem carga e ritmo bem diferentes e deveria só receber fatos, entraria no caminho síncrono do atendimento. Ou seja, ganharia menos rede em troca de perder justamente a separação que o diagnóstico tinha achado.

## 4. Por que definir as três regras antes de ligar a mensageria?

Escolhi o comportamento de repetição, a garantia de que a mesma mensagem entregue duas vezes não produz o efeito duas vezes.

Sem essa regra, a trilha de auditoria passaria a mostrar o mesmo fato registrado mais de uma vez. Uma reentrega acontece por motivos banais: o consumidor processou a mensagem e caiu antes de confirmar, o broker não recebeu o reconhecimento e reenviou, ou o serviço foi reiniciado durante uma implantação. Em qualquer um desses casos a trilha ficaria com duas linhas para uma única autorização, com identificadores e horários diferentes, e ninguém conseguiria dizer olhando para ela se foram duas entregas da mesma coisa ou dois eventos de verdade.

O estrago é maior do que parece porque a auditoria é justamente o lugar onde se vai buscar prova. Qualquer contagem tirada dali passa a ser suspeita, a conferência entre o que Autorização registrou e o que Auditoria recebeu nunca fecha, e num incidente a equipe gasta tempo decidindo em quais linhas acreditar em vez de investigar o problema. Uma trilha que pode duplicar registro deixa de servir para o que ela existe.

Vale notar também que reentrega não é caso raro. É o comportamento normal de mensageria com entrega ao menos uma vez, então tratar repetição não é precaução exagerada, é requisito. Foi por isso que a equipe definiu a regra antes de ligar a mensageria, e não depois de o primeiro relatório sair errado.

## 5. Que dado tornaria visível um sinal de revisão?

Peguei o segundo sinal: uma das regras passa a precisar de um ciclo de implantação isolado.

O dado a acompanhar é a origem de cada implantação do macrosserviço, ou seja, quais módulos foram tocados na mudança que gerou aquela publicação. Com isso, a equipe consegue calcular quantas das implantações do período existiram por causa de um módulo só. Se a regra contratual, por exemplo, começar a responder sozinha por uma parcela crescente das publicações, é sinal de que ela tem um ritmo próprio e está sendo carregada junto com o resto sem precisar.

Vale medir junto o tempo entre a mudança ficar pronta e ela entrar no ar, separado por módulo. Se o número de implantações puxadas por um módulo sobe e o tempo de espera dele também, os dois dados juntos mostram o mesmo sinal por ângulos diferentes: aquele pedaço está esperando uma janela que não é dele. Um mês ou dois de acompanhamento já dão para diferenciar uma temporada de mudanças contratuais de uma tendência real.

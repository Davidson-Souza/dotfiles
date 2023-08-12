---
layout: post
title: Client-side validation(PT-BR)
date: 2023-07-24
categories: bitcoin
---
# O que é Client-side validation e, por que isso é útil?

## Motivação
Durante os últimos meses, projetos utilizando a ideia de Client Side Validation (Validação no Lado do Cliente, CSV). Todavia, enquanto lendo os comentários, é aparente que muitos ainda não compreendem a diferença entre um sistema com estado global, como o Bitcoin, e estado segregado como em um cliente CSV. Por isso, esse post é uma tentativa de clarificar algumas dúvidas sobre o tema.

## O que é estado?

Em sistemas distribuídos como o Bitcoin, é necessária uma maneira de garantir que os participantes do sistema concordem com uma série de fatores. Quando Alice envia uma transação para Bob; Bob irá validar o conteúdo da transação, verificando se Alice realmente possui aqueles fundos, e se a transação foi devidamente autorizada. Sem uma maneira de validar por si próprio a informação que Bob recebe, e assumindo que não exista um terceiro de confiança. Bob estaria vulnerável a todo tipo de ataque por atores maliciosos.

Na Ciência da Computação, estado é um conjunto de atributos armazenados em um sistema. Imagine um jogo de xadrez, ele começa no estado inicial - sendo fixo, definido pelas regras do jogo. Após o movimento inicial, o tabuleiro irá transitar em diferentes estados, cada vez que um dos jogadores efetuar um movimento. Nesse caso, o estado é a posição de cada peça em um dado momento.
Digamos que eu te diga: Dama E4 era o melhor movimento no estado E10? Como você saberia, sem saber o estado E10? Seria virtualmente impossível. Para tanto, é comum necessitarmos armazenar um ou mais estados do sistema, se quisermos verificar certas afirmações sobre o mesmo.

## O que é um sistema com estado global?

No Bitcoin, e em múltiplas altcoins, utiliza-se o sistema de Unspent Transaction Output (UTXO). Para entender o que isso significa, precisamos observar como a transferência de fundos ocorre no protocolo. Uma transação, isto é, uma transferência de fundos na rede, precisa indicar de onde os fundos sendo gasto originam-se. Pode parecer estranho, afinal o seu banco sabe de onde o seu dinheiro vem. Mas nesse caso, o protocolo Bitcoin sabe quem é você, você não tem uma conta no Banco Bitcoin ou algo do tipo. Precisamos de uma maneira mais engenhosa para armazenar os fundos.

Em um sistema de UTXO, os fundos vivem em pequenas Moedas, que possuem um valor e uma chave pública. Quando você cria uma transação, essa deve dizer qual(quais) UTXO(s) ela está gastando. Então, você deve realocar os fundos em novas moedas, assinalando os valores e chaves respectivamente. As moedas gastas por uma transação precisam, obrigatoriamente, ter sido criada em uma transação que tenha ocorrido anteriormente a essa. Além disso, as moedas gastas nessa transação são marcadas como gastas, e não poderão ser utilizadas em outras transações. Em termos técnicos, as moedas criadas são chamadas Saídas(Output) e as moedas sendo gastas são chamadas Entradas(Input). Uma saída que ainda não foi utilizada como entrada é conhecida como Saída não Gasta (Unspent Transaction Output - UTXO).


![Como uma transação funciona](/assets/Bitcoin-Transaction.png)

Com isso em mente, retomemos o caso onde Alice envia fundos para Bob. Como Bob estaria certo de que ele realmente recebeu seus fundos, e não um cheque sem fundo? Bom, Bob precisa buscar pelas saídas que Alice diz possuir (as entradas), se elas não existem, Alice está mentindo. Se elas existem, prossiga da seguinte maneira:
Verifique se a soma das é maior ou igual a soma das saídas (caso contrário, Alice estaria criando dinheiro do nada).

Verifique se Alice realmente é dona das UTXOs. Isso é efetuado verificando-se a assinatura digital que vem junto à transação.
Se tudo ocorrer bem, então a transação é valida.

Para que Bob consiga averiguar se as UTXOs são válidas, ele deve guardar o registro de todas as saídas criadas no passado, que nunca foram gastas. Uma vez que ela seja gasta. Bob pode removê-las do seu registro. Isso faz com que Bob precise guardar um número elevado de UTXOs. Na data em que escrevo esse post, existem cerca de 100 milhões de UTXO, consumindo pouco mais de 4GB para ser armazenada no Bitcoin Core.


Como todos os participantes devem guardar o estado de maneira redundante, tal sistema é conhecido como Sistema com Estado Global ou Estado Compartilhado. Existe um termo técnico pra isso, Máquina de Estados Auto Replicável, mas não é necessário se aprofundar nisso.

## Por que isso é um problema?

Que Bob precisa guardar todas as UTXOs no sistema é um sério problema para a escalabilidade do sistema, pois, você sempre estará limitado pelo poder computacional do “Bob mais fraco”. Além disso, existem um gigantesco problema de coordenação. Imagine a quantidade de dados que deve ser trocado para atualizar cada transação. Alice envia sua transação para a rede, que nada mais é que um amontoado de máquinas executando um cliente compatível. Todos os clientes ao redor do mundo precisam seguir os mesmos passos; validar a transação, remover as UTXOs sendo gastas e adicionar quais foram criadas. Isso é incrivelmente ineficiente, e praticamente impossível de escalar verticalmente.

Imagine por um instante que Alice consiga provar para Bob a existência das suas UTXOs, sem que Bob precise ter uma lista extensiva de UTXOs. Imagine também que os participantes nesse protocolo só precisariam manter suas próprias UTXOs sem comprometer na segurança e descentralização do sistema. Parece impossível, certo. Mas é exatamente isso que CSV resolve!

## Como funciona?

A ideia é antiga, proposta por meados de 2016 por Peter Todd, mas aplicações utilizando esse conceito são relativamente recentes. Os exemplos mais proeminentementes são Taproot Assets (Taro) e RGB. Eles combinam CSV com outras ferramentas como Single Use Seal para criar protocolos escaláveis e privados por natureza. Esse post não entrará nos detalhes de cada um, ou na ideia de Single Use Seal. Se alguém me pedir (e lembrar) posso escrever sobre cada um em detalhe. Mas vejamos a ideia geral de CSV.

O estado é quebrado em “micro estados”, compostos por apenas informações referentes a você. Esse estado pode ser generalizado para algo maior que uma UTXO, como um valor imobiliário com múltiplos atributos, um domínio DNS, etc. O sistema inicia-se com exatamente um estado, chamado de Estado Gênesis. O conteúdo desse estado é definido pelo protocolo, mas não é importante. O único requerimento é que possamos identificá-lo unicamente. Além disso, existem as Mudanças de Estado(State Transitions) que funcionam como as transações no Bitcoin, e contém, além da lista dos estados, as informações extra para validar, como assinaturas digitais.

Quando uma mudança de estado ocorrer na rede, um ou mais estados serão criados, e o primeiro será invalidado, isto é, não poderá mais ser utilizado. Os novos estados devem apontar para o seu antecessor, formando uma cadeia de forma que se você seguir essa cadeia até o fim, chegará ao estado gênesis.

Imagine agora um protocolo idêntico ao Bitcoin, todavia utilizando essa nova noção. Quando Alice enviar a sua transação para Bob, ela irá enviar todos os estados até o gênesis para Bob, e isso será o suficiente para Bob saber que elas existiam. Isso é verdade, pois se todos os estados anteriores e suas respectivas transições são válidas, deduz-se por absurdo que o estado mais recente também é válido (A composição de elementos válidos também é válida).

![Uma demonstração do conceito de cadeia de estados](/assets/states.png)

Os estados são **owned**, ou seja, tem um **dono**. O dono do estado é aquele que consegue invalidá-lo e utilizar ele em uma transição. Para uma UTXO, isso significa aquele que pode gastá-la. Mas lembre-se que esse conceito é muito mais poderoso que uma simples emulação do UTXO set.

No exemplo acima, Satoshi é o dono do estado gênesis, e ele pode utilizá-lo para criar os estados #1 e #2. O estado #1 pertence à Alice, já o estado #2 ainda pertence à Satoshi. Imagine que esse seria análogo à UTXO de troco no Bitcoin. Alice então pode utilizar o estado #1 para criar os estados #3 e #4. Os estados que já foram invalidados uma vez estão com um "x" vermelho. Satoshi não pode mais utilizar o estado #1, pois ele já foi invalidado. Alice não pode utilizar o estado #2, pois ele não é dela. Alice pode utilizar o estado #3 para criar o estado #5.

Como os estados e mudanças de estado não são compartilhadas em uma rede, a privacidade também é substancialmente melhorada. A melhor maneira de anonimizar algo é nunca o publicando na internet.

Mas os leitores mais atentos devem ter uma dúvida: um dos argumentos para ter todas as UTXOs é para permitir auditar a base monetária por inflação e verificar se não há gastos não autorizado de moedas, esse protocolo quebra essa premissa, não? Surpreendentemente, não! Vamos por partes:

 - Verificar a oferta de moedas: Assume-se um token ou contrato que não permite re-emissão, isto é, criar moedas depois do estado gênesis. Só existem duas maneiras para Alice de criar inflação: utilizar uma UTXO inexistente e tentar criar outro gênesis.
    - O primeiro é impossível, pois se ela enviar uma transação com uma UTXO que não existe seria facilmente detectável por Bob. “Mas Bob poderia fazer um conluio com Alice e tentar passar a UTXO inexistente para alguém, certo?”. Não! Isso seria exatamente como uma nota falsa, mas uma bem tosca, que dê para saber logo de cara. Se Bob aceitar essa UTXO, ele dificilmente conseguirá passá-la para outra pessoa. Ele teria tomado um prejuízo certamente.
    - O segundo também não faria sentido, pois nenhum cliente iria aceitar esse bloco como gênesis.
 - Verificar se não existem gastos inválidos: O argumento aqui é idêntico ao anterior. Imagine que Alice faça uma transação inválida para Bob, digamos que as assinaturas dela são inválidas. Quando Bob for enviar moedas para alguém, esse alguém irá rejeitar. E que ninguém aceitaria isso e seria imediatamente reconhecido como fraude, garante que isso não ocorrerá. Não é economicamente viável.

![Alice tenta passar uma UTXO inexistente para Bob](/assets/Estado%20invalido.png)

De fato. Bitcoin percê poderia operar em um sistema similar, todavia, seria uma mudança invasiva e praticamente impossível de ser ativada. Uma analogia com o Bitcoin seria cada usuário guardar apenas as suas UTXOs, e o caminho até uma transação coinbase. Contudo, o fato de existirem múltiplas coinbase criariam exponencialmente mais caminhos possíveis, então o ideal seria todos os Satoshis terem sido criado no bloco gênesis.

Esse sistema é escalável, pois o tamanho da prova para o estado (o caminho até o estado gênesis) cresce baseado no número de transações individuais, e não do número de transações o número total. Em outras palavras, o número de estados na prova seria análogo ao número de vezes que uma nota física troca de mãos. E quando uma nota de outra pessoa troca de mãos, você não precisa fazer nada. Além disso, para você gerar a sua prova, basta incluir a transição que criou aquele estado (Alice pagando para Bob) junto aos que já estavam na transação anterior.

O número global de transações sendo executadas, não mais depende do processamento individual de cada participante. O limite global é a soma dos limites locais, e os limites locais dependem do quanto de poder computacional você tem. Para o usuário casual, que faz apenas algumas dúzias de transações diárias, um smartphone seria o suficiente, sem limitar quantas transações outros usuários podem fazer. Com o emprego de Provas de Conhecimento Zero, podemos reduzir ainda mais o custo computacional de tal protocolo para apenas alguns Kb que nunca crescem. Com uma Provas de Conhecimento Zero Recursiva, você verifica a prova quando receber, e gera uma prova recursiva quando for pagar. A demanda de hardware para isso é ínfima, se comparada ao modelo de Estado Global Compartilho.

Como dito acima, o conteúdo do estado pode ser qualquer coisa que possa ser representada computacionalmente. Com essa ideia, podemos implementar contratos inteligentes escaláveis e privados. Isso é um dos pontos mais importante e bullish no Taro e RGB.

Outra coisa importante de ser clarificada é que contratos diferentes possuem gênesis diferentes, e interagem utilizando-se protocolos acima do protocolo base, como Trocas Atômicas. Isso é importante porque um usuário que não se importe com, digamos, contrato de gatos virtuais, não precisa se importar com isso. Apenas guarde o estado gênesis dos contratos que você usa, bem como as transições que você fez. Isso é análogo a um usuário do Bitcoin que não se importa com inscriptions, mas no Bitcoin, você precisa guardar todas as inscriptions, mesmo que você não se importe com elas.

## O tendão de Aquiles

Existe, porém, um problema. E é o mesmo problema que Satoshi teve que resolver quando criou o Bitcoin: O problema do gasto duplo. Como saber se um estado foi invalidado anteriormente? Nesse caso seria ainda pior, pois quando existe o broadcast das transações, eventualmente você irá aprender sobre o fato de uma UTXO ter sido gasta, mas e aqui? Bom, isso já fica para uma próxima oportunidade.

## Conclusão

Sistemas distribuídos comumente empregam diferentes máquinas de estado para atingir seus objetivos. Todavia, se o estado for compartilhado entre todos os participantes, a escalabilidade e privacidade do sistema é fortemente degradada. Client Side Validation é uma importante alternativa para a solução de tais problemas.

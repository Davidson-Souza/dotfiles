---
layout: post
---

# Resolvendo o problema do gasto duplo em Client Side Validation

## Introdução

No [post anterior](/2013/08/02/client-side-validation.html) nós vimos como client side validation funciona, e como ele pode aumentar a privacidade e escalabilidade de um sistema descentralizado. Nós também vimos que ele tem um problema: o problema do gasto duplo. Neste post nós vamos ver como resolver este problema de forma simples e eficiente.

## O problema do gasto duplo

O problema do gasto duplo é um problema que ocorre quando um usuário tenta gastar uma moeda mais de uma vez. Por exemplo, se o usuário Alice tentar gastar a moeda `C` duas vezes, ela pode enviar duas transações para a rede:

```
T1: Alice -> Bob, C
T2: Alice -> Carol, C
```

Esse problema foi o que fez com que os antecessores do Bitcoin não funcionassem, pois precisavam de um servidor central para evitar o gasto duplo, o que tornava o sistema centralizado e vulnerável a ataques. Tal ente centralizado é chamado de `mint` (cunhador), e é o que o Bitcoin tenta evitar. Satoshi resolveu o problema através do uso de uma simples estrutura de dados que ele chamou de `timechain` (cadeia de tempo), que é o que nós chamamos de `blockchain`.

A ideia é que cada transação seja incluída em um bloco, e que cada bloco seja ligado ao bloco anterior através de uma função de hash. Isso garante que todos os participantes da rede tenham uma visão única e consistente do estado atual do sistema, e que não haja gasto duplo.

O problema é que, para que isso funcione, é necessário que todos os participantes da rede tenham uma cópia completa da blockchain, o que é inviável em um sistema descentralizado. Por isso, nós precisamos de uma forma de garantir que todos os participantes da rede tenham uma visão única e consistente do estado atual do sistema, sem que seja necessário que todos tenham uma cópia completa da blockchain.

## Commitment

Em criptografia, `commitment` é um conceito que permite que uma pessoa se comprometa com um valor sem revelar esse valor. Por exemplo, se Alice quer se comprometer com o valor `x`, ela pode enviar para Bob o valor `y = hash(x)`. Bob não sabe qual é o valor `x`, mas ele sabe que Alice se comprometeu com um valor `x` tal que `y = hash(x)`. Se ela quiser provar que ela se comprometeu com o valor `x`, ela pode simplesmente enviar o valor `x` para Bob, e ele pode verificar que `y = hash(x)`. Aqui está um commitment de uma previsão minha que pode ou não virar realidade. Se for verdade, eu posso provar que eu fiz esse commitment:

> 4b32ab70804016ff10b43e87b732e9a4070d95a53ca0ad146681e51d04577a1e

## Commitment em Client Side Validation

Lembra dos estados parciais? Cada carteira possui multiplos estados parciais, e para enviar fundos, ou realizar qualquer outra operação, a carteira precisa invalidar o estado anterior, e criar uma `State Transition` que leva do estado anterior para um novo estado. O problema é que, se a carteira criar duas state transitions para dois estados diferentes, e enviar as duas para a rede, não há como saber qual delas é a correta. Nesse caso seria, pior, pois em Client Side Validation, as transições não são publicadas em uma rede pública, e sim enviadas diretamente para o destinatário. Isso significa que, se a carteira enviar duas transições para dois estados diferentes, o destinatário não tem como sequer saber que isso aconteceu.

## Single Use Seals

A sacada que precisamos para resolver o nosso problema, é uma abstração matemática conhecida como `Single Use Seals` (selos de uso único). A ideia é que, para cada estado parcial, a carteira cria um selo de uso único, que contém um commitment do estado parcial. Esse selo é enviado para o destinatário, e o destinatário pode verificar que o selo é válido. A mágica é que, como o nome sugere, os selos só podem ser utilizados uma vez. Quando o selo é utilizado, ele é invalidado, e não pode mais ser utilizado. Isso significa que, se a carteira tentar enviar dois selos para dois estados diferentes, o destinatário vai saber que isso aconteceu, pois o segundo selo não vai ser válido.

Cada estado parcial tem um selo associado, enquanto o selo estiver fechado, o estado será valido. Uma state transition só pode ser aceita se ela contiver uma prova de que o selo foi aberto. Como o selo só pode ser aberto uma vez, isso garante que cada estado parcial só pode ser utilizado uma vez, portanto, o problema do gasto duplo é resolvido.

A forma mais trival de se implementar Single Use Seals é utilizando as UTXOs do Bitcoin. UTXOs comportam-se como selos de uso único, e são utilizados para garantir que cada moeda só possa ser gasta uma vez. A ideia é que cada estado parcial seja associado a uma UTXO. Abrir o selo significa gastar a UTXO, uma state transition só pode ser aceita se ela contiver uma prova de que a UTXO foi gasta.

Mas como associar um estado a uma UTXO? Simples, basta utilizar o hash do estado, fazendo algum commitment para ela, o mais comum e conhecido como `tweak`, que é simplesmente somar a chave pública pela hash do estado (P + H(S)*G). Para gastar a UTXO, você precisá somar a Hash na chave privada também. Esta forma é interessante pois não adiciona nenhuma informação extra. No ponto de vista da rede, é uma simples chave pública.

Para provar que você sabe a hash, basta enviar `P` e o estado (ou a hash) para alguém, e essa pessoa pode verificar que `P + H(S)*G = P'`, onde `P'` é a chave pública que foi usada na transação. Quando construindo uma state transition, basta gastar essa UTXO, e assinalar o próximo estado parcial na próxima UTXO.

## Yay?

Mas agora você deve estar pensando: "Mas isso não resolve o problema de escalabilidade, pois ainda é necessário que cada state transition seja incluída em um bloco. Você ainda está limitado pelo número de transações por segundo que a rede consegue processar". E você está certo. Mas essa é a forma mais fácil de se implementar Client Side Validation, existem formas mais convexas de se fazer isso. Aqui vai um exeplo:

Imagine que temos um grupo de oito participanetes que querem enviar transações. Podemos utilizar uma única UTXO para as oito pessoas, e contruir uma ávore com as oito hashes.

```
14
|---------------\
12              13
|-------\       |-------\
08      09      10      11
|---\   |---\   |---\   |---\
00  01  02  03  04  05  06  07
```

Apenas 14 seria publicado, e teríamos apenas uma UTXO. Então definimos que esta UTXO só pode ser gasta se a transação gastando ela criar uma outra UTXO com **exatamente a mesma chave pública**. Cada participante publica a prova que vimos anteriormente para 14, mas também uma `Merkle Proof` do seu próprio `commitment`. Exemplo: digamos que eu queira provar `00`. Então eu preciso enviar `01`, `09`, `13`. Assim a parte verificando pode recomputar a raiz da árvore, e verificar que `00` está na árvore.

Para cada abertura, uma transação é feita publicando a prova para um estado parcial. Se alguém abrir o selo antes de você, você irá salvar a transação e qual selo foi aberto. Quando você abrir o seu, basta enviar todas as aberturas anteriores, para provar que o seu não está aberto ainda.

Claro, isso é só um exemplo simples, e requer interação. Mas é possível criar sistemas mais otimizados e com mais participantes. Mas isso é assunto para outro dia.

## Conclusão

Com o uso de Single Use Seal podemos resolver o problema do gasto duplo e criar sistemas de Client Side Validation super escaláveis. Essa ideia está no coração de projetos como Taro e RGB, e é uma das ideias mais interessantes que eu já vi. Espero que você tenha gostado, e até a próxima!
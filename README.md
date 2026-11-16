TDE - Jantar dos Filósofos, Semáforos e Deadlock
Vídeo no YouTube: https://youtu.be/QKqzeF4SxLo

Grupo 5
Clara Amazonas Pereira da Costa
Geovanna Karolyne Javé Soto
Guilherme Reis Carvalho
Vinicius Meier Trevisan

Fizemos as três partes em Java e testamos com o JDK 21. Não usamos nenhuma biblioteca externa, só o que já vem no Java (Thread, Semaphore e ReentrantLock). Os logs das execuções que aparecem aqui estão na pasta logs/.


Como compilar e executar

Em cada pasta, compile com:

    javac *.java

Depois é só rodar a classe que quiser:

    java FilosofosIngenuo        (parte 1, trava de propósito, sai com Ctrl+C)
    java FilosofosHierarquia     (parte 1, roda por 10 segundos e termina sozinho)
    java ContadorConcorrente     (parte 2, roda cada versão 3 vezes, leva alguns segundos)
    java DeadlockTravando        (parte 3, trava de propósito, sai com Ctrl+C)
    java DeadlockCorrigido       (parte 3, termina normalmente)


Parte 1 - Jantar dos Filósofos

O problema
São cinco filósofos numa mesa redonda, e cada um fica alternando entre pensar, sentir fome e comer. Entre cada dois filósofos tem um garfo, então são 5 garfos no total. Pra comer, o filósofo precisa dos dois garfos que estão do lado dele, e cada um desses garfos também é usado pelo vizinho. No código, o garfo i fica entre o filósofo i e o filósofo (i+1) mod 5, e cada garfo é um semáforo com 1 permissão.

Por que trava
Na versão ingênua (FilosofosIngenuo.java), todo mundo pega primeiro o garfo da esquerda e depois o da direita. Se os cinco pegarem o da esquerda ao mesmo tempo, cada um fica com um garfo na mão esperando o da direita, que está com o vizinho. Como ninguém solta o que já pegou, ninguém come. Colocamos um sleep entre pegar o primeiro e o segundo garfo pra garantir que isso acontecesse em toda execução.

Log da execução (logs/p1-ingenuo.txt). Depois da última linha o programa fica parado:

    Filosofo 0: pensando
    Filosofo 3: pensando
    Filosofo 1: pensando
    Filosofo 4: pensando
    Filosofo 2: pensando
    Filosofo 3: com fome
    Filosofo 1: com fome
    Filosofo 2: com fome
    Filosofo 4: com fome
    Filosofo 0: com fome
    Filosofo 1: pegou o garfo 1 (esquerda)
    Filosofo 3: pegou o garfo 3 (esquerda)
    Filosofo 0: pegou o garfo 0 (esquerda)
    Filosofo 2: pegou o garfo 2 (esquerda)
    Filosofo 4: pegou o garfo 4 (esquerda)

Nenhum filósofo chega a imprimir "comendo".

Olhando as condições de Coffman, todas aparecem nessa versão. Tem exclusão mútua, porque um garfo só fica com um filósofo de cada vez. Tem manter e esperar, já que o filósofo continua segurando o garfo da esquerda enquanto espera o da direita. Não tem preempção, ou seja, ninguém consegue tirar o garfo da mão de outro. E tem espera circular: o 0 espera o 1, o 1 espera o 2, e assim vai até o 4, que espera o 0.

A solução: hierarquia de recursos
Numeramos os garfos de 0 a 4 e todo filósofo pega primeiro o garfo de número menor e depois o de número maior. Na prática, isso só muda alguma coisa pro filósofo 4, que tem os garfos 4 e 0 e por isso passa a pegar o 0 primeiro.

Com isso a espera circular deixa de existir. Pra fechar um ciclo, alguém teria que estar segurando um garfo de número maior e esperando um de número menor, e a regra não deixa isso acontecer. Então sempre tem pelo menos um filósofo que consegue pegar os dois garfos, comer e devolver. As outras três condições continuam lá, mas sem a espera circular não dá deadlock.

Pseudocódigo:

    N = 5
    garfos[0..N-1] = semáforos justos com 1 permissão

    Para cada filósofo p:
        esquerda = p
        direita = (p + 1) mod N
        menor = min(esquerda, direita)
        maior = max(esquerda, direita)

        loop:
            estado[p] = "pensando"
            pensar()
            estado[p] = "com fome"
            adquirir(garfos[menor])
            adquirir(garfos[maior])
            estado[p] = "comendo"
            comer()
            liberar(garfos[maior])
            liberar(garfos[menor])

Progresso e justiça
Como sempre tem alguém que consegue comer, o sistema nunca fica parado. Faltava evitar a inanição, que é um filósofo ficar esperando pra sempre enquanto o vizinho pega o mesmo garfo de novo e de novo. Pra isso criamos os semáforos no modo justo (new Semaphore(1, true)), que entrega o garfo pra quem está esperando há mais tempo, numa fila FIFO.

Numa execução de 10 segundos (logs/p1-hierarquia.txt), o filósofo 0 comeu 19 vezes, o 1 comeu 25, o 2 comeu 28, o 3 comeu 29 e o 4 comeu 18. Os números não ficaram iguais, mas todos comeram várias vezes e nenhum ficou sem comer.


Parte 2 - Semáforos

O ContadorConcorrente.java cria 8 threads (T) e cada uma soma 1 num contador compartilhado 200.000 vezes (M). No final o contador deveria valer T x M = 1.600.000. Fizemos uma versão sem nenhuma sincronização e outra com um semáforo binário justo (new Semaphore(1, true)), e rodamos cada uma 3 vezes (logs/p2-contador.txt).

Uma coisa que tivemos que fazer foi declarar o contador como volatile. Sem isso, em algumas execuções o JIT otimizava o laço e a versão sem semáforo acabava dando o valor certo por sorte, e aí não dava pra ver a condição de corrida.

Resultados
Sem semáforo, as três execuções deram 1.134.581 (45 ms), 1.251.137 (15 ms) e 807.696 (23 ms). Nenhuma chegou perto de 1.600.000, e cada uma deu um valor diferente.
Com semáforo, as três deram exatamente 1.600.000, mas levaram 20.722 ms, 17.706 ms e 20.063 ms.

Por que a versão sem sincronização perde incrementos
O count = count + 1 parece uma coisa só, mas na verdade são três passos: ler o valor, somar 1 e escrever de volta. Se duas threads leem o mesmo valor antes de qualquer uma escrever, as duas escrevem o mesmo número e um dos incrementos some. É uma condição de corrida, e é por isso que o resultado muda toda vez.

Por que a versão com semáforo dá certo
O semáforo binário só deixa uma thread por vez entrar no trecho do count = count + 1. As outras ficam paradas no acquire() até a que está lá dentro chamar release(). Assim ninguém se mete no meio da leitura, da soma e da escrita. Colocamos o release() dentro de um finally pra permissão ser devolvida mesmo se der alguma exceção.

O custo no desempenho
A diferença de tempo foi enorme: entre 17 e 20 segundos com semáforo, contra algumas dezenas de milissegundos sem. A versão sem semáforo é rápida porque nenhuma thread espera ninguém, só que o resultado sai errado. Com o semáforo justo, a cada incremento a thread entra na fila, pode ser bloqueada e depois precisa ser acordada pelo sistema operacional, e isso demora muito mais do que a soma em si. O modo justo garante que as threads sejam atendidas na ordem em que chegaram, mas em troca o throughput cai bastante.

Visibilidade e ordenação (happens-before)
No modelo de memória do Java, o release() de uma thread acontece antes (happens-before) do próximo acquire() de outra thread no mesmo semáforo. Isso quer dizer que tudo o que a primeira thread escreveu em count antes do release() vai estar visível pra segunda depois do acquire(), e que as instruções não podem ser reordenadas passando por esse ponto. O volatile sozinho só resolve a visibilidade. Ele não torna o incremento atômico, e por isso a versão sem semáforo continuou errada mesmo com o contador volatile. O semáforo resolve as duas coisas.


Parte 3 - Deadlock

O cenário
Temos dois locks, LOCK_A e LOCK_B, e duas threads. A Thread 1 pega o A, dorme 50 ms e tenta pegar o B. A Thread 2 faz o contrário: pega o B, dorme 50 ms e tenta pegar o A. O sleep serve pra garantir que, quando cada uma for atrás do segundo lock, a outra já esteja segurando ele.

Pseudocódigo da versão que trava:

    Thread 1:                      Thread 2:
      adquirir(LOCK_A)               adquirir(LOCK_B)
      dormir(50ms)                   dormir(50ms)
      adquirir(LOCK_B)               adquirir(LOCK_A)
      imprimir "T1 concluiu"         imprimir "T2 concluiu"
      liberar(LOCK_B)                liberar(LOCK_A)
      liberar(LOCK_A)                liberar(LOCK_B)

Saída do java DeadlockTravando (logs/p3-deadlock.txt). O programa não termina, e "T1 concluiu" e "T2 concluiu" nunca aparecem:

    T2 tentando pegar LOCK_B
    T2 pegou LOCK_B
    T1 tentando pegar LOCK_A
    T1 pegou LOCK_A
    T2 tentando pegar LOCK_A
    T1 tentando pegar LOCK_B

Com o programa travado, rodamos o jstack, e ele mesmo detectou o deadlock (a saída completa está em logs/p3-jstack.txt):

    Found one Java-level deadlock:
    =============================
    "Thread-0":
      waiting for ownable synchronizer 0x000000008cd62ce0, (a java.util.concurrent.locks.ReentrantLock$NonfairSync),
      which is held by "Thread-1"

    "Thread-1":
      waiting for ownable synchronizer 0x000000008cd62cb0, (a java.util.concurrent.locks.ReentrantLock$NonfairSync),
      which is held by "Thread-0"

A Thread-0 é a T1 e a Thread-1 é a T2, pela ordem em que foram criadas.

As condições de Coffman aparecem todas aqui também. Cada lock só pode estar com uma thread de cada vez (exclusão mútua). A T1 segura o A enquanto espera o B, e a T2 segura o B enquanto espera o A (manter e esperar). Um lock não pode ser tirado de uma thread, só ela mesma pode liberar (não preempção). E a T1 espera a T2, que espera a T1 (espera circular). Como as quatro acontecem ao mesmo tempo, o programa trava.

A correção: ordem global de aquisição
No DeadlockCorrigido.java, as duas threads sempre pegam o LOCK_A antes do LOCK_B:

    Thread 1 e Thread 2:
      adquirir(LOCK_A)
        adquirir(LOCK_B)
          // seção crítica
        liberar(LOCK_B)
      liberar(LOCK_A)

Isso quebra a espera circular. A thread que consegue o LOCK_A primeiro também consegue o LOCK_B, porque a outra fica esperando lá no LOCK_A e nem chega a tentar o B. Então nunca acontece de uma thread estar com o B esperando o A, e o ciclo não fecha.

Saída (logs/p3-corrigido.txt):

    T1 tentando pegar LOCK_A
    T2 tentando pegar LOCK_A
    T1 pegou LOCK_A
    T1 tentando pegar LOCK_B
    T1 pegou LOCK_B
    T1 concluiu
    T2 pegou LOCK_A
    T2 tentando pegar LOCK_B
    T2 pegou LOCK_B
    T2 concluiu
    Terminou sem deadlock

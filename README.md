TDE - Jantar dos Filósofos, Semáforos e Deadlock
Vídeo no YouTube:

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

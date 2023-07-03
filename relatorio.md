## Introdução

Metaheurísticas de *Ant Colony Optimization* são bem eficientes para soluções aproximativas de problemas np-difíceis em grafos, realizando uma espécie de heurística gulosa estocástica, repetidamente buscando soluções golusas rápidas e aproveitando  partes boas de soluções passadas para encontrar soluções melhores futuras.

O problema da clique-máxima consiste em encontrar a clique maximal máxima do grafo, com a clique sendo um _____. Podemos encontrar a clique tanto pelos vértices que a compõe tanto pelas arestas. Dada o speedup do algoritmo ao utilizar a abordagem de vértices, preferimos considerar essa abordagem.

Solucionaremos isso construindo cliques maximais aleatórias baseando-se no movimento de cada formiga, com a formiga que produziu a melhor clique maximal de cada iteração depositando "feromônio" nos vértices que comporam essa clique, e o feromônio dos demais "evaporando" com a passagem do tempo. O movimento das formigas se baseia na quantidade de feromônio em cada vértice, o seja, se ela está no vértice v0, cada vértice ligado a v0 tem uma probabilidade de deslocamento. A formiga prefere caminhos que outras formigas já percorreram, escolhendo com maior probabilidade o vértice com mais feromônio. Isso garante *exploitation*, já que ela utiliza soluções para tentar contruir uma melhor, e o componente probabilístico dos demais vértices garante *exploitation*, já que elas sempre começam de vértices aleatórios e, mesmo que um vértice v1 tenha muito feromônio, ela ainda pode acabar indo para v2, evitando máximos locais.

Conforme o tempo passa, o feromônio inicial evaporará quase que por completo, com as formigas então compondo soluções passadas para criar soluções futuras, priorizando o refinamento do espaço de soluções ao invés de explorar soluções novas no espaço de busca.

## Implementação

Modelamos o problema da clique máxima como encontrar o maior conjunto de vértices que compõe uma clique ao longo de várias repetições. Cada clique será maximal, já que construímos a solução até esgotar vértices que poderiam entrar na clique, mas **não necessariamente é máxima**

### Otimização do tempo de execução

Um dos gargalos do execução completa do algoritmo é a criação das cliques maximais, que é interpretado numa metáfora biológica como o *caminho de uma formiga*, operação que é repetida para cada formiga em cada iteração, ou seja, acelerar sua execução oferece speedup quase linear para o algoritmo

Para resolver esse problema

Criamos três funções principais

### `construct_clique(G, tau, n, alpha)`

Essa função é responsável por criar a clique estocásticamente. 
1. Escolhemos um vértice $$v_0$$ aleatório para começar e criamos uma lista de candidatos, inicialmente todos os vizinhos de $$v_0$$.

2. Baseado nos feromônios de cada vértice, registrados no vetor `tau`, e o fator `alpha`, criamos um vetor de probabilidades para cada vértice candidato, de forma que vértices com mais feromônio tenham maior probabilidade.
3. Escolher vértice $$v_c$$ para entrar na clique e registrar seus vizinhos em outra lista de candidatos
4. Atualizar os candidatos, fazendo a interseção entre os candidatos do início da interação com os de $$v_c$$, já que todo candidato futuro a entrar na clique também deve ser vizinho dos dois
5. Verificar se o vértice é de fato vizinho de todos os vértices presentes na clique, e somente se for, o adicionamos a lista de vértices. Isso geralmente seria feito 
6. Verificar a lista de candidatos. Se tivermos algum, repetimos os passos 3-5. Quando a lista estiver vazia, terminamos nossa clique maximal

### `ant_clique(G, ants_per_v, maxCycles, tauMin, tauMax, alpha=1.05, evaporation_rate=0.9, plot=False, parallel=False)`

Nessa função fazemos de fato o algoritmo de ACO, de acordo com os parâmetros especificados. Basicamente, criamos estruturas de dados para contagem de métricas. Estamos acompanhando a iteração do algoritmo que atingiu o melhor valor, assim como o número de cliques repetidas. Fazemos a conta de cliques repetidas com um dicionário tuplas x ocorrência, em que se uma clique for composta pelos mesmos vértices mas em ordens diferentes, consideramos a mesma.

O algoritmo segue esses passos
1. Determinar a quantidade de formigas, `total_ants`, de acordo com o parâmetro `ants_per_V`
2. Criar loop para iterar `maxCycles` vezes
3. Montar  `total_ants` cliques estocásticas com a função de construção
4. Escolher a maior clique desse conjunto
5. Registrar a ocorrência dela no dicionário de ocorrência de cliques
6. Comparar a melhor clique encontrada anteriormente com essa. Se for melhor, a trocamos
7. Evaporar o feromônio, multiplicando por `evaporation_rate`
8. Calcular a adição de feromônio, baseando-se na distância entre essa solução local e a melhor encontrada até o momento
9. Incrementar os vértices da solução local no vetor `tau`
10. Arrumar os limites min/max de feromônio, controlados por `tauMin, tauMax`, de forma que o vetor `tau` respeite os limites
11. Repetir o passo 2, e quando ele terminar, retornamos a melhor clique encontrada e as métricas desejadas

O algoritmo executa razoavelmente rápido, já que o gargalo é a função de construção de cliques, que devido a implementação com *Numba* é super rápida. Será que podemos melhorar?
#### Paralelismo

Um ponto importante desse algoritmo é paralelismo. A função parece um bom candidato de paralelismo, vide a criação independente de cliques estocásticas, então teoricamente se criamos 50 cliques, usar 5 processos paralelos diminuiria a carga de trabalho para o tempo de execução de ~10 cliques. 
Experimentalmente isso foi uma abordagem horrível, já que o tempo de execução aumentou bastante. Imaginamos que a causa disso seja o tempo de serialização das estruturas de dados. Como o fluxo de um sistema paralelo é o *fork* dos dados, execução e *join*, observamos que a maior parte do tempo de execução é justamente o *fork*, que demora bastante, já que o grafo inteiro e outras variáveis precisam ser copiados para a thread que executará, algo que demora com grafos grandes. Em linguagens de nível menor, poderíamos definir o grafo como variável global imutável atômica, de forma que seu compartilhamento seria feito em $$O(1)$$, algo que não consegui fazer com sucesso em *python*.

Logo, como boa parte do tempo de execução é gasto no *fork*, não foi encontrado *speedup* de execução paralelo, sendo então melhor fazer as cliques aleatórias sequencialmente.

### `experiment`

Nessa função fazemos o intervalo de confiança do ACO, variando a semente aleatória de acordo com o número de variações aleatórias desejadas, combinando ao final as melhores soluções para escolher a melhor clique.

Resumidamente, recebemos o número de `random_runs` e, para cada índice, fixamos a semente aleatória, rodamos a função `ant_clique`, registramos os resultados e comparamos com a melhor clique vista até o momento, trocando-a se melhor.


## Experimentos: análise dos parâmetros/decisões no resultado do ACO e
discussões de resultados;

### Geração de métricas

Definimos um espaço grande de experimentação, variando os seguintes parâmetros

```
ants_per_vertex = [0.02, 0.05, 0.06, 0.07, 0.1, 0.2, 0.5 ]
max_cycles = [10, 50, 100, 500, 1000]
tau_min = [0.01, 0.1, 0.5]
tau_max = [2, 3, 6, 8]
alpha = [0, 0.75, 0.98, 1, 1.1, 2]
evaporation_rate = [0.8, 0.9, 0.95, 0.98, 0.99]
```

Em seguida, combinamos todos eles em um produto, criando todas as combinações únicas de parâmetros.

Como o tamanho do vetor de combinações é 12600, quantidade bem grande, criamos uma lógica paralela de experimentação, seguindo uma lógica de memorização de resultados

1. Dividir as combinações em batches, de acordo com o número de CPUs
2. Para cada batch, redistribuir a execução do experimento nas CPUs desejadas
3. Em cada experimento, verificamos se os parâmetros estão na base de cache. Se estiver, pulamos este experimento. Senão, rodamos a função de experimentação e retornamos os resultados
4. Ao final da execução paralela, agregamos os resultados de cada CPU e salvamos na base de cache

Isso proveu speedup substancial, já que a execução demoraria mais de 40 horas para o caso *easy* em minha máquina, reduzindo a execução total bastante.

Como discutimos anteriormente, grande parte do gargalo é o *fork*, em que o grafo é serializado e copiado para cada thread. Usando primitivas de sincronização de baixo nível, poderíamos melhorar ainda mais o tempo de execução, porém para o escopo desse trabalho essa melhora já foi suficiente

### Análise de métricas

As três métricas mais valiosas para nossa análise foram
1. Tamanho da clique maximal, queremos aproximar o tamanho da clique máxima
2. Número de iterações em que 80% das execuções atingiu a melhor clique maximal. Calculamos isso ao longo de todas as execuções variando semente aleatória, na intuição de que parâmetros que convergem mais rápido a solução para uma clique boa em um percentil bom dos resultados, são experimentalmente bons. Vale lembrar que esse parâmetro depende de encontrar a melhor clique maximal, pois execuções com parâmetros muito ruins vão convergir para uma clique maximal local em poucas iterações, porém isso não significa que os parâmetros são bons, é necessário analisar a métrica juntamente do tamanho da clique maximal. Isso é uma métrica de *velocidade*
   
3. Quantas execuções não atingiram o máximo. Calculamos a proporção de execuções que não atingiu o melhor valor encontrado ao final de todos os ciclos, uma métrica auxiliar do número de iterações e sujeita às mesmas ressalvas. Um detalhe que as diferencia é que essa proporção é ao final, ou seja, se temos um número muito grande de iterações, como 1000, é esperado que grande parte consiga atingir o valor máximo. Isso é uma métrica de *consistência*, ou seja, quanto maior for essa proporção, mais esses parâmetros chegam perto do melhor valor independente da semente aleatória


Hierarquicamente, escolhemos os parâmetros que atingiram as melhores cliques, para só então analisar as métricas 2 e 3. Queremos soluções que cheguem rapidamente em valores bons de clique em um número aceitável de variações de semente aleatória.


Podemos observar que a melhor estratégia d execução do algoritmo de ACO não é aumentar o número de ciclos, algo que tem pouca melhora ao longo do tempo, ficando preso em ótimos locais, e sim executar várias vezes com sementes diferentes um número menor de ciclos, escolhendo a melhor ao longo do tempo. Como podemos ver no gráfico, uma porcentagem boa de soluções atingiu o melhor valor em menos de 300 ciclos. Tendo mais variações aleatórias e menos iterações, conseguimos chegar em soluções de qualidade



## Conclusões

analisamos mais de 10 000 permutações de parâmetros do algoritmo para o problema da clique máxima em dois grafos, concluindo que os resultados experimentais do artigo referência estão bons, podendo melhorar apenas no número de formigas, que depende bastante da natureza do grafo estudado, sendo então interessante ir além do que foi sugerido de 30 formigas.

## Bibliografia.
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

Essa função 

## Experimentos: análise dos parâmetros/decisões no resultado do ACO e
discussões de resultados;

## Conclusões;

## Bibliografia.
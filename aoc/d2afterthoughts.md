# [AOC 2016 - Day 2](http://adventofcode.com/2016/day/2) Afterthoughts

After I first finished coding the [Advent of Code Day 2 challenge](https://github.com/zakaluka/fsaoc2016/blob/master/2/TwoMain.fsx) and then again after I wrote the [corresponding blog entry](http://znprojects.blogspot.com/2017/03/aoc-2016-day-2.html), I felt dissatisfied with my solution.

The reasons for this are numerous:

1. The keypad was never formally modeled as a type.
1. The `move` function basically encoded the constraints of the keypad, with rather brittle assumptions about what that keypad looks like.
1. The `move` function operated on `string`s, as opposed to a more 'proper' data type.

For a toy problem such as the AOC challenge, the poor choice I made there is barely acceptable. However, in the 'real world', this type of decision can mean the difference between gibberish and maintainable code.

Let's take this problem one step further. If you look closely, you will see that the keypad is really just a specialized version of a general 2-dimensional (2D) board.

## How Can A 2D Board Be Represented As A Functional Data Structure?

In general, 2D boards are often represented as some form of graph. We have some choices when it comes to representing a 2D board in F#. We are aided by the fact that, for the AOC problem, we don't 'need' to (though we may want to) change this data structure once it is created. This means that even if we use an inherently mutable data structure like an array, our logic can be written so that we do not to 'lose' immutability in this scenario.

If you were implementing an actual game where the state of the board changes (e.g. the locations of the players, threats, obstacles, etc.), your decision would require different analysis.

Here are some data structures we can use to represent a 2D board in F#:

- Adjacency matrix
- Edge list
- Adjacency list
- Inductive graphs

## Adjacency matrix

References: [Wikipedia](https://en.wikipedia.org/wiki/Adjacency_matrix), [NIST](https://xlinux.nist.gov/dads/HTML/adjacencyMatrixRep.html), [Khan Academy](https://www.khanacademy.org/computing/computer-science/algorithms/graph-representation/a/representing-graphs)

Per [Wikipedia](https://en.wikipedia.org/wiki/Adjacency_matrix), an adjacency matrix is "a square matrix used to represent a finite graph".

F# comes with a convenient data structure called `Array2D` that we can use for this purpose. You can also represent a fixed-width 2D array as a 1D `array`/`list`/`seq` using [row- or column-major order](https://en.wikipedia.org/wiki/Row-_and_column-major_order) to store the information.

In general, adjacency matrices have the following characteristics:

- Space:
    - $\mathcal{O}(V^2)$ space usage, where $V$ is the number of vertices.
- Speed:
    - $\mathcal{O}(1)$ constant time operations to:
        - Check if an edge exists.
        - Add an edge.
        - Remove an edge.
    - $\mathcal{O}(n)$ time operations to:
        - Get all the edges for a given vertex.

## Edge List

References: [Khan Academy](https://www.khanacademy.org/computing/computer-science/algorithms/graph-representation/a/representing-graphs)

An edge list is a way to represent a graph by simply keeping an exhaustive list of all the edges. For undirected graphs, a 2-tuple is sufficient. For directed graphs, a 3-tuple (or more) may be required to hold additional information about the edge.

Though these data structures are quite simple, they work very well for smaller graphs and are easy to reason about. You can also optimize these structures (to a certain degree) by creatively reducing the number of edges that need to be tracked.

In general, edge lists have the following characteristics:

- Space:
    - $\mathcal{O}(E)$ space usage, where $E$ is the number of edges.
- Speed:
    - $\mathcal{O}(n)$ time operations to:
        - Get all the edges for a given vertex (if using a list to store the edge list).
    - $\mathcal{O}(1)$ constant time operations to:
        - Get all the edges for a given vertex (if using a hash-based structure to store the edge list).
    - $\mathcal{O}(d)$ speed to check if a particular edge exists, where $d$ is the degree of the vertex in question.

## Adjacency List

References: [Wikipedia](https://en.wikipedia.org/wiki/Adjacency_list), [Open Data Structures](http://opendatastructures.org/ods-java/12_2_AdjacencyLists_Graph_a.html), [NIST](https://xlinux.nist.gov/dads/HTML/adjacencyListRep.html), [Khan Academy](https://www.khanacademy.org/computing/computer-science/algorithms/graph-representation/a/representing-graphs)

Per [Wikipedia](https://en.wikipedia.org/wiki/Adjacency_list), an adjacency list is "a collection of unordered lists used to represent a finite graph". It is a combination of an adjacency matrix and an edge list.

In F#, an adjacency list can use a number of data structures, including `list`, `map`, `dict`, etc. Hash-based structures (such as `dict`) should provide good performance due to faster access to the mapping from vertex to edges.

In general, adjacency lists have the following characteristics:

- Space:
    - $2|E|$ space usage for an undirected graph, where $E$ is the number of edges.
    - $|E|$ space usage for a directed graph, where $E$ is the number of edges.
- Speed:
    - $\mathcal{O}(d)$ time operations to:
        - Check if a particular edge exists, where $d$ is the degree of the vertex in question.
    - $\mathcal{O}(1)$ constant time operations to:
        - Get all the edges for a given vertex.

## Inductive Graph

References: [Inductive Graphs and Functional Graph Algorithms](https://web.engr.oregonstate.edu/~erwig/papers/InductiveGraphs_JFP01.pdf), [Martin Erwig's Functional Graph Library](https://hackage.haskell.org/package/fgl), [Hekate - Graphs for F#](https://xyncro.tech/hekate/), [Wikipedia](https://en.wikipedia.org/wiki/Inductive_data_type)

Per Martin Erwig's paper, there is a gap in how graph structures are represented and manipulated in functional languages, especially when it comes to performance as compared to imperative implementations (I completely, 100% agree). His paper lays out a strategy for creating a graph as an inductively defined data type. Inductive data types are algebraic data types or sum types (i.e. F# discriminated unions) which can also be recursive (e.g. the classic functional representation of a tree or list structure that is based on discriminated unions).

I actually found this paper when I was looking for information on how to represent graphs in functional languages. I strongly recommend that anyone who is interested in such data structures should read the [original paper](https://web.engr.oregonstate.edu/~erwig/papers/InductiveGraphs_JFP01.pdf).

Erwig's goal was to create a fully functional data structure whose performance is as close to imperative data structures (e.g. adjacency lists) as possible. Erwig performed some analysis based on his recommended implementation of inductive graphs using binary search trees, and I'll provide those characteristics here:

- Speed:
    - $\mathcal{O}(n \log n)$ time operations to:
        - Insert new vertices into the graph
    - $\mathcal{O}(n^2 \log n)$ or $\mathcal{O}(n \log^2 n)$ time operations to:
        - Delete vertices from the graph

## So, What Comes Next?

Returning to our problem, we see that we have a number of implementation choices. Naively, we have up to 9 vertices and 36 edges in this example (Day 2, Part 1) or 13 vertices and 52 edges (Day 2, Part 2). This means that even a poorly selected data structure will most likely have minimum impact on the run speed.

When I first started the exercise of blogging about my AOC solutions, I had no intention of rewriting solutions. However, I did start this as a personal learning exercise, so I feel that this is a good use of my time (and will allow me to play with data structures and libraries that I've never used in F#).

I am currently going through the exercise of re-writing an implementation for Day 2 using the 4 choices presented above. My goal is to finish and write about those implementations before writing about the solution for AOC Day 3.

See you next time!
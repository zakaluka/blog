# AOC 2016 - Day 2 Alternate Solutions

_[Problem on AOC website](http://adventofcode.com/2016/day/2)._

_Code for this post can be found on [GitHub](https://github.com/zakaluka/fsaoc2016/blob/master/2/)._

This post is a follow-up to my [last post](https://znprojects.blogspot.com/2017/03/aoc-2016-day-2-afterthoughts.html) on my solution for AOC Day 2 and how I was unhappy with the design (though my initial results were still correct).

Since that time, I have implemented a set of five graph data structures to make the solution more robust than simply encoding the minimum solution with a function.

These data structures have a few advantages:

- The data types are much more robust - for the board and the graphs. Instead of talking in terms of raw numbers / strings, I can now speak in terms of Vertices and Edges.
- The solution's design allows for the underlying data structure to be swapped out as long as we can provide implementations of two key functions - `create` and `getNext`.
- Different graph structures can be compared in terms of (basic) performance. I will provide my numbers at the end of this blog.

## Modeling a board

_Code on [GitHub](https://github.com/zakaluka/fsaoc2016/blob/master/2/TwoCommon.fsx)_

### Storing a board in a text file

First, I changed the way that boards are stored. Rather than hard-coding the board or worse, encoding the representation directly into the algorithm with no easy way to change it, I picked a very simple text-based format.

Blank spaces are represented by dots / periods (`.`). All other characters represent a unique `Vertex`.

For day 2, part 1, the following board is stored in a [text file](https://github.com/zakaluka/fsaoc2016/blob/master/2/d2p1_map).

```
123
456
789
```

Similarly, here is the [day 2, part 2 board](https://github.com/zakaluka/fsaoc2016/blob/master/2/d2p2_map):

```
..1..
.234.
56789
.ABC.
..D..
```

Boards are represented internally as a set of vertices and edges. Let's look at those in more detail.

### Vertices

A `Vertex` is a tuple consisting of an integer accompanied by a string label. Here are the type and module signatures for `Vertex`.

```fsharp
type Vertex =
  | V of int * string
  with
    override ToString : unit -> string
  end
module Vertex = begin
  val create : idx:int -> label:string -> Vertex
  val toInt : _arg1:Vertex -> int
  val toString : x:Vertex -> string
  val strToV : lbl:string -> (seq<Vertex> -> Vertex)
  val intToV : idx:int -> (seq<Vertex> -> Vertex)
end
```

Here are the expected properties of a `Vertex`.

- Each `Vertex` must be represented by a unique integer.
- Each `Vertex` must have a unique string label (for `strToV` to work correctly).
- While not strictly required, the on-screen display for certain graph data structures is much prettier if each string label has a length of 1 (aka a single character).

All the properties listed above are not enforced by the data type. However, some properties are enforced by the `Board` creation algorithm.

### Edges

The AOC Day 2 problem requires directed graphs. Thus, an edge is stored as a pair of vertices and a direction.

We have four directions in the AOC problem - `R`ight, `L`eft, `U`p, and `D`own.

`Direction` also has both a type and an accompanying module:

```fsharp
type Direction =
  | R
  | L
  | U
  | D
  with
    override ToString : unit -> string
  end
module Direction = begin
  val opposite : _arg1:Direction -> Direction
  val toDirection : _arg1:string -> Direction
end
```

_As I was working on the alternate solutions for Day 2, I added certain functions to various modules as I was writing the graph structures. However, in certain cases, I did not require those functions in the final solution but still left them in the module. A good example of this is the `opposite` function in the `Direction` module._

The `Edge` type and module are as follows:

```fsharp
type Edge =
  | E of Vertex * Direction * Vertex
  with
    override ToString : unit -> string
  end
module Edge = begin
  val create : dir:Direction -> v1:Vertex * v2:Vertex -> Edge
  val toString : e:Edge -> string
  val fromV : Edge -> Vertex
  val toV : Edge -> Vertex
  val direction : Edge -> Direction
end
```

`fromV`, `direction`, and `toV` represent the three individual components of an `Edge`.

Some properties of an `Edge`:

- Edges are defined solely by the two vertices they connect and the direction of the connection. They have no other unique identifiers (internal or external).

### Building the board

Now that all the building blocks are in place, we can build the `Board`. The type and module for `Board` are as follows.

```fsharp
type Board =
  {vertices: seq<Vertex>;
    edges: seq<Edge>;}
  with
    override ToString : unit -> string
  end
module Board = begin
  val transpose : rows:seq<#seq<'b>> -> seq<seq<'b>>
  val validEdge : l1:string * l2:string -> bool
  val potentialEdgesByRow :
    verts:seq<Vertex> -> rows:seq<#seq<string>> -> seq<Vertex * Vertex>
  val reverse : x:'a * y:'b -> 'b * 'a
  val squareup : strs:seq<seq<string>> -> seq<seq<string>>
  val parseBoard : strB:string -> Board
end
```

Most of the functions for the `Board` module are internal functions and should not be invoked by an external client. The most important function in the module is `parseBoard`, which takes in a string representing the board (e.g. the text files I mentioned earlier) and returns a constructed `Board` that meets a number of requirements.

- Boards are two-dimensional.
- Each character in the string representation that is not a blank space is turned into a unique vertex.
- Each vertex is given a unique integer ID.
- Boards do not contain diagonal edges.
- Each pair of neighboring vertices has two edges between them.
- Given two neighboring vertices, the two edges between them have opposite directions.

## Generic solution framework

The solution performs the following steps to get to the final answer for the problem.

1. Create a `Board` from the `boardFile`.
1. Use that `Board` to create a graph using `bCreateF`.
1. Translate the instructions from the `instrFile` into `Direction`s.
1. Take each line of instructions and run them through the graph. The start `Vertex` for each line of instructions is the result from the previous line of instructions, except for the first line of instructions which uses `Vertex` "5" as its start `Vertex`.
1. Turn the result from each line of instructions into a `string`, then concatenate them.

The solution framework has two functions, `moveAll` and `day2part1`.  The former uses `bGetNextF` to move through a line of instructions. The latter is the connector that converts a board and lines of instructions into a final answer.

The solution framework's signature is as follows.

```fsharp
module Solution = begin
  val moveAll :
    func:(Vertex -> 'a -> Vertex) ->
      startv:Vertex -> instrs:seq<'a> -> Vertex
  val day2part1 :
    boardFile:string ->
      instrFile:string ->
        bCreateF:(Board -> 'a) ->
          bGetNextF:('a -> Vertex -> Direction -> Vertex) -> string
end
```

## Graph data structures

The graph data structures below are admittedly naively implemented, except inductive graphs which make use of the [Hekate](https://xyncro.tech/hekate/) library. However, they all follow certain rules:

- Each data structure's type is hidden within its respective module.
- Once a graph is created, it is not modified in any way.
- If the `getNext` function cannot find a move in the given direction from the given source vertex, it will return the source vertex to the caller.

### Adjacency Matrix

_Code on [GitHub](https://github.com/zakaluka/fsaoc2016/blob/master/2/TwoAdjMat.fsx)._

Per [Wikipedia](https://en.wikipedia.org/wiki/Adjacency_matrix), an adjacency matrix is "a square matrix used to represent a finite graph".

Since the AOC Day 2 implementation requires a directed graph, the adjacency matrix cannot be a symmetric matrix. For my implementation, the label for each row represents the "from `Vertex`", the label for each column represents the "to `Vertex`" and the value at their intersection represents the `Direction` of movement.

For example, when looking at the Day 2, Part 1 board below, you can read the first entry as "When you start from Vertex 1 and go Right, you arrive at Vertex 2".

Here is the text representation of the Day 2, Part 1 board as an adjacency matrix.

```
  123456789
1 .R.D.....
2 L.R.D....
3 .L...D...
4 U...R.D..
5 .U.L.R.D.
6 ..U.L...D
7 ...U...R.
8 ....U.L.R
9 .....U.L.
```

And here is the text representation of the Day 2, Part 2 board.

```
  123456789ABCD
1 ..D..........
2 ..R..D.......
3 UL.R..D......
4 ..L....D.....
5 .....R.......
6 .U..L.R..D...
7 ..U..L.R..D..
8 ...U..L.R..D.
9 .......L.....
A .....U....R..
B ......U..L.RD
C .......U..L..
D ..........U..
```

Internally, an adjacency matrix is stored as an `Array2D [,]`.  The scheme used to create this data structure is somewhat fragile because it assumes that the first Vertex has a unique integer ID of `0` and that subsequent vertices' IDs are incremented deterministically without gaps. Since we control `Board` and `Vertex` creation, we can rely on this assumption for now but it would be a dangerous assumption to make in production code.

The `getNext` function takes the following steps to move through the board:

1. Use the 2D array to find the row representing the `from` Vertex.
1. Once found, go through the row and looks for the desired `Direction`.
1. If found, use that column index to go from an integer to a `Vertex` using `Board.vertices`.
1. If not found, return the `from` Vertex to the caller.

```fsharp
let getNext am from dir =
  match am with
  | AM(x, b) ->
    let fromInt = Vertex.toInt from
    let colBase = Array2D.base2 x
    x.[fromInt..fromInt,colBase..]
    |> Seq.cast<Direction option>
    |> Seq.tryFindIndex (( = ) (Some dir))
    |> function
      | Some idx -> Seq.item idx b.vertices
      | None -> from
```

### Edge List

_Code on [GitHub](https://github.com/zakaluka/fsaoc2016/blob/master/2/TwoEdgeList.fsx)._

An edge list is an extremely simple graph implementation which does not transform the original `Board`'s internal structures in any way.  It is, as the name implies, an exhaustive list of all edges in the graph.

Here is a visual representation of the Day 2, Part 1 board.

```
Edges: ["4->U->1"; "7->U->4"; "5->U->2"; "8->U->5"; "6->U->3"; "9->U->6"; "1->D->4";
 "4->D->7"; "2->D->5"; "5->D->8"; "3->D->6"; "6->D->9"; "1->R->2"; "2->R->3";
 "4->R->5"; "5->R->6"; "7->R->8"; "8->R->9"; "2->L->1"; "3->L->2"; "5->L->4";
 "6->L->5"; "8->L->7"; "9->L->8"]
```

And the Day 2, Part 2 board.

```
Edges: ["6->U->2"; "A->U->6"; "3->U->1"; "7->U->3"; "B->U->7"; "D->U->B"; "8->U->4";
 "C->U->8"; "2->D->6"; "6->D->A"; "1->D->3"; "3->D->7"; "7->D->B"; "B->D->D";
 "4->D->8"; "8->D->C"; "2->R->3"; "3->R->4"; "5->R->6"; "6->R->7"; "7->R->8";
 "8->R->9"; "A->R->B"; "B->R->C"; "3->L->2"; "4->L->3"; "6->L->5"; "7->L->6";
 "8->L->7"; "9->L->8"; "B->L->A"; "C->L->B"]
```

The `getNext` function for an edge list is quite simple as well.

```fsharp
let getNext (EL(b)) from dir =
  let destv =
    Seq.tryFind
      (fun (E(v1,d,v2)) -> v1 = from && d = dir)
      b.edges
  match destv with
  | Some(E(_,_,v2)) -> v2
  | None -> from
```

### Adjacency List

_Code on [GitHub](https://github.com/zakaluka/fsaoc2016/blob/master/2/TwoAdjList.fsx)_

As opposed to an edge list, an adjacency list takes a `Vertex`-centric approach to representing graphs.  Many implementations of adjacency lists internally use a data structure like a hashmap or hashtable, where the key is a `Vertex` and the value is a list of vertices (possibly with additional data) that the key can connect to.

For my implementation, I used two separate data structures to implement the adjacency lists - `dict` and `Map`.  The keys are `Vertex` instances and the values are lists of `Edge`s.

Here is a visual representation of the Day 2, Part 1 board.

```fsharp
1==>1->D->4, 1->R->2,
2==>2->D->5, 2->R->3, 2->L->1,
3==>3->D->6, 3->L->2,
4==>4->U->1, 4->D->7, 4->R->5,
5==>5->U->2, 5->D->8, 5->R->6, 5->L->4,
6==>6->U->3, 6->D->9, 6->L->5,
7==>7->U->4, 7->R->8,
8==>8->U->5, 8->R->9, 8->L->7,
9==>9->U->6, 9->L->8,
```

Here is the Day 2, Part 2 board.

```fsharp
1==>1->D->3,
2==>2->D->6, 2->R->3,
3==>3->U->1, 3->D->7, 3->R->4, 3->L->2,
4==>4->D->8, 4->L->3,
5==>5->R->6,
6==>6->U->2, 6->D->A, 6->R->7, 6->L->5,
7==>7->U->3, 7->D->B, 7->R->8, 7->L->6,
8==>8->U->4, 8->D->C, 8->R->9, 8->L->7,
9==>9->L->8,
A==>A->U->6, A->R->B,
B==>B->U->7, B->D->D, B->R->C, B->L->A,
C==>C->U->8, C->L->B,
D==>D->U->B,
```

The `getNext` implementations for both the `dict` and `Map` versions are very similar, only differing in how the various values are accessed.

First, here is the `getNext` implementation for the `dict`-version of adjacency lists.

```fsharp
let getNext (AL(d,_)) from dir : Vertex =
  let es = d.Item from
  match Seq.tryFind (fun (E(_,d,_)) -> dir = d) es with
  | Some (E(_,_,v2)) -> v2
  | None -> from
```

And, similarly, here is the `Map`-version of `getNext`.

```fsharp
let getNext (AL(d,_)) from dir =
  let es = Map.find from d
  match Seq.tryFind (fun (E(_,d,_)) -> dir = d) es with
  | Some (E(_,_,v2)) -> v2
  | None -> from
```

### Inductive Graph

_Code on [GitHub](https://github.com/zakaluka/fsaoc2016/blob/master/2/TwoIndGraph.fsx)._

Finally, I used an inductive graph library, [Hekate](https://github.com/xyncro/hekate), as my final representation of a graph in F#.  As I noted in my [last blog post](https://znprojects.blogspot.com/2017/03/aoc-2016-day-2-afterthoughts.html), inductive graphs were introduced by Erwig in his 2001 paper and then implemented (first) as the Haskell [fgl](https://hackage.haskell.org/package/fgl) library.

Inductive graphs are functional data structures that allow similar operations to other inductive data structures such as lists and trees.

Despite using a library here instead of implementing the data structure myself, I actually enjoyed writing code with this library.  The authors have done a great job of using familiar syntax and patterns with Hekate.  The only downside I could find is that the Hekate library is almost completely undocumented.  I learned how to use the library from the unit tests I found on [GitHub](https://github.com/xyncro/hekate/blob/master/tests/Hekate.Tests/Hekate.Tests.fs) and from reading Erwig's original paper.

Here is the Day 2, Part 1 board represented visually as an inductive graph.

```fsharp
map
  [(V (0,"1"),
    (map [(V (1,"2"), L); (V (3,"4"), U)], "1",
     map [(V (1,"2"), R); (V (3,"4"), D)]));
   (V (1,"2"),
    (map [(V (0,"1"), R); (V (2,"3"), L); (V (4,"5"), U)], "2",
     map [(V (0,"1"), L); (V (2,"3"), R); (V (4,"5"), D)]));
   (V (2,"3"),
    (map [(V (1,"2"), R); (V (5,"6"), U)], "3",
     map [(V (1,"2"), L); (V (5,"6"), D)]));
   (V (3,"4"),
    (map [(V (0,"1"), D); (V (4,"5"), L); (V (6,"7"), U)], "4",
     map [(V (0,"1"), U); (V (4,"5"), R); (V (6,"7"), D)]));
   (V (4,"5"),
    (map [(V (1,"2"), D); (V (3,"4"), R); (V (5,"6"), L); (V (7,"8"), U)], "5",
     map [(V (1,"2"), U); (V (3,"4"), L); (V (5,"6"), R); (V (7,"8"), D)]));
   (V (5,"6"),
    (map [(V (2,"3"), D); (V (4,"5"), R); (V (8,"9"), U)], "6",
     map [(V (2,"3"), U); (V (4,"5"), L); (V (8,"9"), D)]));
   (V (6,"7"),
    (map [(V (3,"4"), D); (V (7,"8"), L)], "7",
     map [(V (3,"4"), U); (V (7,"8"), R)]));
   (V (7,"8"),
    (map [(V (4,"5"), D); (V (6,"7"), R); (V (8,"9"), L)], "8",
     map [(V (4,"5"), U); (V (6,"7"), L); (V (8,"9"), R)]));
   (V (8,"9"),
    (map [(V (5,"6"), D); (V (7,"8"), R)], "9",
     map [(V (5,"6"), U); (V (7,"8"), L)]))]
```

And here is the Day 2, Part 2 board as an inductive graph.  Please note that F# Interactive actually truncated the print out.  I am assuming it did this due to the size of the map.

```fsharp
map
  [(V (0,"1"), (map [(V (2,"3"), U)], "1", map [(V (2,"3"), D)]));
   (V (1,"2"),
    (map [(V (2,"3"), L); (V (5,"6"), U)], "2",
     map [(V (2,"3"), R); (V (5,"6"), D)]));
   (V (2,"3"),
    (map [(V (0,"1"), D); (V (1,"2"), R); (V (3,"4"), L); (V (6,"7"), U)], "3",
     map [(V (0,"1"), U); (V (1,"2"), L); (V (3,"4"), R); (V (6,"7"), D)]));
   (V (3,"4"),
    (map [(V (2,"3"), R); (V (7,"8"), U)], "4",
     map [(V (2,"3"), L); (V (7,"8"), D)]));
   (V (4,"5"), (map [(V (5,"6"), L)], "5", map [(V (5,"6"), R)]));
   (V (5,"6"),
    (map [(V (1,"2"), D); (V (4,"5"), R); (V (6,"7"), L); (V (9,"A"), U)], "6",
     map [(V (1,"2"), U); (V (4,"5"), L); (V (6,"7"), R); (V (9,"A"), D)]));
   (V (6,"7"),
    (map [(V (2,"3"), D); (V (5,"6"), R); (V (7,"8"), L); (V (10,"B"), U)], "7",
     map [(V (2,"3"), U); (V (5,"6"), L); (V (7,"8"), R); (V (10,"B"), D)]));
   (V (7,"8"),
    (map [(V (3,"4"), D); (V (6,"7"), R); (V (8,"9"), L); (V (11,"C"), U)], "8",
     map [(V (3,"4"), U); (V (6,"7"), L); (V (8,"9"), R); (V (11,"C"), D)]));
   (V (8,"9"), (map [(V (7,"8"), R)], "9", map [(V (7,"8"), L)])); ...]
```

The `getNext` algorithm is relatively simple, considering that each `Vertex` stores information about both its predecessors and successors, i.e. what points to the `Vertex` and what the `Vertex` points to, respectively.

```fsharp
let getNext ig from dir =
  match Graph.Nodes.successors from ig with
  | None -> from
  | Some succs ->
    succs
    |> List.tryFind (fun (_,d) -> d = dir)
    |> function
      | None -> from
      | Some (dest,_) -> dest
```

## Testing

_Code on [GitHub](https://github.com/zakaluka/fsaoc2016/blob/master/2/TwoCommonTest.fsx)._

In the past, I have struggled quite a bit with properly implementing [FsCheck](https://github.com/fscheck/FsCheck) tests.  However, this time around, the combination of using FsCheck directly (as opposed to using it in combination with a testing library like [Fuchu](https://github.com/mausch/Fuchu) or [Xunit](https://xunit.github.io/)) and using it from F# script files (as opposed to compiled programs) made the experience much more pleasant and was a great learning experience.

I wrote tests using [FsCheck](https://github.com/fscheck/FsCheck) to ensure that my implementations were correct - and I'm glad I did.  I found a number of elementary mistakes by having automated tests that I could easily run after each change to validate my results.  For each data structure, I wrote 6 tests:

1. A printout of the Day 2, Part 1 board (to be verified manually).
1. A printout of the Day 2, Part 2 board (to be verified manually).
1. The Day 2, Part 1 test provided in the AOC problem description.
1. The Day 2, Part 2 test provided in the AOC problem description.
1. The Day 2, Part 1 problem stated in the AOC problem description.
1. The Day 2, Part 2 problem stated in the AOC problem description.

I could not automate the first two tests because each data structure's printout was quite different from the others.

I was able to implement the last two tests because I had already solved Day 2's problems and knew the correct answers.

## Performance

_Raw data on [GitHub](https://github.com/zakaluka/blog/blob/master/aoc/d2runtimes.xlsx) - requires a program that can read XLSX files._

After writing the five graph data structure implementations, I decided to run some basic performance tests on them.  Having a generalized solution framework allowed me to attribute all differences in runtimes to the specific data structure being used.

To perform the tests in the most "unbiased" way possible, I ran each test 5 times and took the average of the results.  For each run, I executed the last 4 tests listed in the [Testing](#testing) section.

Without further ado, here are the results from using F# interactive's `#time` directive.

Algorithm|Real average|CPU average|Gen0 average|Gen1 average|Gen2 average
:--------|-----------:|----------:|-----------:|-----------:|-----------:
Adjacency List Dict|0.8452|0.8326|104.2|25.2|0
Adjacency List Map|0.8684|0.8576|105.8|21.6|0
Inductive Graph|0.9894|0.9852|136.2|33|0
Adjacency Matrix|4.4932|4.4798|530.6|78.8|0.6
Edge List|359.2024|359.1448|68896.6|61.4|3.6

Table: _Average run-time and garbage collection performance of the algorithms, sorted by the `Real average` column in ascending order._

I then took the Adjacency List Dict implementation as the baseline (since it was the fastest) and translated the same table into percentages.

Algorithm|Real|CPU|Gen0|Gen1|Gen2
:--------|---:|--:|---:|---:|---:
Adjacency List Dict|100%|100%|100%|100%|-
Adjacency List Map|103%|103%|102%|86%|-
Inductive Graph|117%|118%|131%|131%|-
Adjacency Matrix|532%|538%|509%|313%|100%
Edge List|42499%|43135%|66120%|244%|600%

Table: _Average run-time and garbage collection performance as a percentage, with Adjacency List Dict as the baseline._

### Observations

I was not surprised to find that the Edge List was the worst performer.  However, I was surprised by how much worse its performance was, even on a small board, than the next worst algorithm, the Adjacency Matrix.

Along those same lines, I was surprised by how poor the performance of the Adjacency Matrix was.  However, I have a feeling that a more optimized implementation could do much better than my naive attempt. I believe my observation (as to my poor Adjacency Matrix writing skills) is accurate considering that it, along with the Adjacency List, is one of the most popular data structures for storing graphs.

The inductive graph performed very well for a relatively new data structure that was intended, from the beginning, to be a new way of storing graphs in functional languages.  Its run-time performance was approximately 17% worse than the baseline and memory performance (based on garbage collection) was approximately 31% worse than the baseline. For someone who needs a graph data structure for more than just a `getNext` function, it could be a serious contender and warrants additional testing.

## Next steps

I am satisfied with the results of this investigation into graph data structures in F#.  I was able to implement all algorithms in a functional manner within a generic framework. I believe I have accomplished the goal that I set out with, which was to find a more robust and 'realistic' way of representing graph-like structures in F#.

However, here are some items that I could not (or did not) complete and may be worth looking into:

- Change the base data structures to enforce more of the expected properties / invariants.
- Make the types for `Vertex`, `Edge`, `Direction`, etc. private and move them within their respective modules.
- Test with larger / more complex boards to better understand performance.
- Change the way boards are parsed from text files so that when the same character appears multiple times on a board, it is treated as the same Vertex (allowing for more complex board structures).
- Change boards to allow 3D representations.
- Remove copying from Adjacency Matrix's `create` function.
- Implement (or find existing libraries) and test more functions of graph data structures.

I will now be returning to blogging about my solutions for Advent of Code 2016 problems.  I am greatly looking forward to this because I have temporarily stopped solving more problems until I catch up with my blog posts.  Hopefully, I will be able to post my Day 3 solutions in the next week.

See you next time!
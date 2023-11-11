---
title: "Triangle"
date: 2023-10-31T17:11:09+01:00
---

{{< rawhtml >}}
<script type="text/javascript" charset="UTF-8" src="https://cdn.jsdelivr.net/npm/jsxgraph/distrib/jsxgraphcore.js"></script>
<link rel="stylesheet" type="text/css" href="https://cdn.jsdelivr.net/npm/jsxgraph/distrib/jsxgraph.css" />
{{< /rawhtml >}}

Since it has been a few years I started working on my [music notation grammar](https://github.com/kwon-young/music), I thought about how I could explain what I am doing to other peoples.
I believe that for anyone else than me, some of the prolog code in that repository might be quite hard to understand.

That's why I thought of a simplified example that could explain some of the logic and design decision behind the project.
This post is only part 1 of a series I plan to do on the subject.

## Introduction

So first, here is what we want to do.
The basic idea of the project is to write a program that relates graphical primitives with high level semantic structure.
Let's take concrete examples of program that do exactly that:

* programs like LaTeX, MuseScore or even Word take a high level description of a document and can produce a graphical representation of the document consisting of a few graphical primitives like glyphes, segment, etc.
* programs like OCRs can take an image of a document and can recognize the text inside of the document image in order to produce a high level representation of the document.
  Although OCRs takes as input an image of a document instead of graphical primitives, most of them will first recognize a small number of graphical primitives like letters or words and then reconstruct high level semantic structures like paragraph, titles etc.

* My own project linked above target is to write a graphical grammar of the common music notation.
  This notation is used to typeset music score since the early of 18th century and is a very complex set of rules specifying the use and placement of music symbols to transcribe music on paper.
  Therefore, the goal I have for this project is to be able to write a prolog program that relates music scores in the MEI format (a format akin to MusicXML) with a set of graphical primitives like music symbols and segments.

Music notation is a very complicated beast so we will simplify things a lot in this first post and try something simpler: middle school geometry!

## Basics

Let's start with a much more basic program.
A program that relates basic 2D euclidean geometrical figures with their graphical primitives.
So, for example, let's take the most basic geometric figure: the triangle.
And for graphical primitives, let's limit ourselves to points and segments.

We will use prolog compound terms to manipulate these data structures:

* A 2D point with coordinates X, Y: `point(X, Y)`
* A segment, which is a pair of points Start and End: `seg(Start, End)`
* A triangle, which can be represented as three points: `triangle(A, B, C)`

The graphical representation of a triangle ABC is done by drawing three segments:

* one from point A to point B
* one from point B to point C
* one from point C to point A

{{< rawhtml >}}
<div id="jxgbox" class="jxgbox" style="width:400px; height:400px;"></div>
<script type="text/javascript">
 var board = JXG.JSXGraph.initBoard('jxgbox', {boundingbox: [-7, 7, 7, -7], axis: true});
 var p = board.create('point',[-3,3], {name: 'A'});
 var p = board.create('point',[3,3], {name: 'B'});
 var p = board.create('point',[2,-2], {name: 'C'});
 var li = board.create('line',['A','B'], {straightFirst:false, straightLast:false, strokeColor:'#000000', strokeWidth:2});
 var li = board.create('line',['B','C'], {straightFirst:false, straightLast:false, strokeColor:'#000000', strokeWidth:2});
 var li = board.create('line',['C','A'], {straightFirst:false, straightLast:false, strokeColor:'#000000', strokeWidth:2});
</script>
{{< /rawhtml >}}

Let's translate this to prolog:

```prolog
triangle_cond(triangle(A, B, C), seg(A, B), seg(B, C), seg(C, A)).
```

> **NOTE**:
> A little note about code convention in this post: predicates that relates geometrical figures with graphical primitives will always end in `_cond`


Interestingly, this knowledge can be reduced to a single prolog fact.
One thing I love about this line of code is that prolog gives us the ability to specify only the knowledge we are interested in.
Notice how we didn't had to define how a point is represented here.
We could use this code with 2D points, but also 3D or in any number of dimension.
We could also use atoms as points, like `'A'`, if we are doing some abstract geometry application.
Finally, by using variables instead of ground term for points, we can represent any triangle in the infinite set of all ground triangles!

Let's move on to the next step, which is to relate this triangle with a list of graphical primitives.
Since, we want to compose and manipulate multiple geometrical figures with a single list of graphical primitives, we will use the prolog DCG notation which is well suited for this use case:

```prolog
triangle_dcg(Triangle) -->
   { triangle_cond(Triangle, Seg1, Seg2, Seg3) },
   [Seg1, Seg2, Seg3].
```

> **NOTE**:
> A little note about code convention in this post: DCG predicates that could be confused with plain compound terms will always end in `_dcg`

For those that are not familiar with the DCG notation, [here is a good primer](https://www.metalevel.at/prolog/dcg).
If you are already familiar with DCGs, you can just skip the paragraph below.

To just explain briefly, a DCG predicate can be used to describe a list.
It has this weird neck `-->` and the arity is annotated with a double shlash like this: `triangle_dcg//1`.
The reason for the double slash is because, internally, prolog translate DCG predicates by adding 2 additional parameters to the head of the clause.
This two additional predicates represent a difference list, which is threaded through inside the body of the DCG predicate.
If you don't know what is a difference list, it's okay, we won't need it here, but it is a very useful prolog pattern to learn.
The braces `{}` around `triangle_cond/4` is there to tell prolog that `triangle_cond/4` is not a DCG predicate but a plain prolog predicate to call normally.

> **NOTE**:
> A little remark on this idea to relates a list of graphical primitives with high level construct with a grammar.
> I believe this type of idea was very popular in the 90s.
> I was exposed to it through my PhD adviser, Bertrand Coüasnon, which implemented this idea in his thesis with the [DMOS system](https://ieeexplore.ieee.org/abstract/document/953786) in a lambda prolog dialect with a grammar formalism called EPF.

Let's see how we can use this DCG in the prolog top level by posting the most general query:

```prolog
?- phrase(triangle_dcg(Triangle), L, R).
Triangle = triangle(_A, _B, _C),
L = [seg(_A, _B), seg(_B, _C), seg(_C, _A)|R].
```

We can see that prolog has build us a `triangle` compound term together with 3 segments with the correct set of points.
As you can see the pair `L` and `R` is a difference list where `R` is the tail of the list `L`.
This allows us to compose multiple DCG predicates as we will see later.
If we want to close the list, we can just call the `phrase/2` predicate without the `R` argument.

We can already do a lot of different things with this predicate.
We can _draw_ a specific triangle:

```prolog
?- phrase(triangle_dcg(triangle('A', 'B', 'C')), L, R).
L = [seg('A', 'B'), seg('B', 'C'), seg('C', 'A')|R].
```

We can recognize a triangle from a set of segments:

```prolog
?- phrase(triangle_dcg(Triangle), [seg('A', 'B'), seg('B', 'C'), seg('C', 'A')]).
Triangle = triangle('A', 'B', 'C').
```

Or reject sets of graphical primitives that do not form a triangle:

```prolog
?- phrase(triangle_dcg(Triangle), [seg('A', 'B'), seg('B', 'C')]).
false.
```

Or test that a specific triangle is related to a set of segments:

```prolog
?- phrase(triangle_dcg(triangle('A', 'B', 'C')), [seg('A', 'B'), seg('B', 'C'), seg('C', 'A')]).
true.
?- phrase(triangle_dcg(triangle('A', 'B', 'C')), [seg('A', 'B'), seg('B', 'C')]).
false.
```

## The ordering problem

Unfortunately, our DCG clause `triangle_dcg` is too strict about the possible segment lists we can use to build a triangle.
For example, the following query fails:

```prolog
?- phrase(triangle_dcg(Triangle), [seg('A', 'B'), seg('C', 'A'), seg('B', 'C')]).
false.
```

There is no reason why segment BC should be before segment CA.
This is especially true when manipulating graphical primitives.
There is no evident ordering of the primitives, and we cannot assume a specific ordering when defining our grammar.

In order to fix this, we will define a predicate `term` which job will be to find and consume a given graphical primitive:

```prolog
term(X) -->
   [X].
term(X), [CurX] -->
   [CurX],
   term(X).
```

The first clause tries to match the given primitive `X`.
If it fails, we go to the second clause and skip the current element `CurX` and try to match the next element by doing a recursive call.
Coincidentally, the standard `select/3` predicate can be used instead transparently and will do exactly the same thing.

Now, let's rewrite our `triangle_dcg//1` DCG predicate to use `term//1`:

```prolog
triangle_dcg(Triangle) -->
   { triangle_cond(Triangle, Seg1, Seg2, Seg3) },
   term(Seg1),
   term(Seg2),
   term(Seg3).
```

Let's retry our most general query:

```prolog
?- phrase(triangle_dcg(Triangle), L, R).
Triangle = triangle(_A, _B, _C),
L = [seg(_A, _B), seg(_B, _C), seg(_C, _A)|R] ;
Triangle = triangle(_A, _B, _C),
L = [seg(_A, _B), seg(_B, _C), _D, seg(_C, _A)|_E],
R = [_D|_E] 
Action (h for help) ? abort
% Execution Aborted
```

As you can see, the first solution is the same as our previous result but now, we get multiple other solutions.
In fact, we get an infinity many more solutions.
From this little change in the code, we went from a single deterministic solution to infinitely many.
This can be good, because it will allow us to get more solutions as we will see below.
This could be also bad, since the complexity of the code has risen dramatically and it could be quite easy to be trapped into an infinite recursion loop.
But we will worry about this later.

One new characteristic of the second results is that the grammar is now resistant to noise, i.e. non relevant graphical primitives.

In the query above, we enumerate the solutions in a depth first manner, lengthening the list to infinity by searching for the last segment deeper and deeper.
Instead, a breadth first search strategy could give us some more interesting solutions.
This can be done by using the `length/2` predicate before the call to `phrase/3`.
Let's see all solutions with a list `L` of length 3:

```prolog
?- length(L, 3), phrase(triangle_dcg(Triangle), L, R).
L = [seg(_A, _B), seg(_B, _C), seg(_C, _A)],
Triangle = triangle(_A, _B, _C),
R = [] ;
L = [seg(_A, _B), seg(_C, _A), seg(_B, _C)],
Triangle = triangle(_A, _B, _C),
R = [] ;
L = [seg(_A, _B), seg(_C, _A), seg(_B, _C)],
Triangle = triangle(_C, _A, _B),
R = [] ;
L = [seg(_A, _B), seg(_B, _C), seg(_C, _A)],
Triangle = triangle(_B, _C, _A),
R = [] ;
L = [seg(_A, _B), seg(_B, _C), seg(_C, _A)],
Triangle = triangle(_C, _A, _B),
R = [] ;
L = [seg(_A, _B), seg(_C, _A), seg(_B, _C)],
Triangle = triangle(_B, _C, _A),
R = [] ;
false.
```

If you count them, that's 6 solutions which are all the permutations of list of length 3 with no replacements.

So, we can post our previous wrongly failing query it should now work:

```prolog
?- phrase(triangle_dcg(Triangle), [seg('A', 'B'), seg('C', 'A'), seg('B', 'C')]).
Triangle = triangle('A', 'B', 'C') ;
Triangle = triangle('C', 'A', 'B') ;
Triangle = triangle('B', 'C', 'A') ;
false.
```

Great, it even gave us all the different representation of the ABC triangle!

Let's move on to another type of triangle, the equilateral triangle.

## The equilateral triangle

The property of an equilateral triangle is that all three sides have the same length.
Let's define the compound term for such a triangle:

* an equilateral triangle ABC: `equilateral_triangle(A, B, C)`

And let's extend the `triangle_cond/4` predicate to relate such a triangle with its three sides:

```prolog
triangle_cond(equilateral_triangle(A, B, C), Seg1, Seg2, Seg3) :-
   triangle_cond(triangle(A, B, C), Seg1, Seg2, Seg3),
   seg_length(Seg1, Len),
   seg_length(Seg2, Len),
   seg_length(Seg3, Len).
```

We first reuse the `triangle_cond/4` first clause since an equilateral triangle is itself a triangle.
This will correctly setup the three segments with the triangle points.
Next, we unify all three segment length to the same length `Len` using the yet to be defined `seg_length/2` predicate.

To compute a segment length, we will use the well known Pythagoras theorem:

```prolog
seg_length(seg(point(X1, Y1), point(X2, Y2)), Len) :-
   Len is sqrt((X2 - X1)**2 + (Y2 - Y1)**2).
```

This predicate will only work on 2D segments, but we have to start from somewhere.

Let's try our new equilateral triangle predicate with the most general query:

```prolog
?- phrase(triangle_dcg(equilateral_triangle(A, B, C)), L, R).
ERROR: Arguments are not sufficiently instantiated
ERROR: In:
ERROR:   [15] _31816 is sqrt((...)+(...))
...
```

Well... As you probably would have predicted, reality hit us on the face with the fact that prolog arithmetic is not pure.
It will only work in a single direction.
So one solution would be to try and modify the order of predicates in the DCG like this:

```prolog
triangle_dcg(Triangle) -->
   term(Seg1),
   term(Seg2),
   term(Seg3),
   { triangle_cond(Triangle, Seg1, Seg2, Seg3) }.
```

In this case, `triangle_cond/4` would become a post condition instead of a precondition.
However, the search strategy of the grammar would become a kind of generate and test strategy, which would be intractable with only a small number of graphical primitives.
In situation like this, modern prolog has come up with a solution called **Constraints**.

## Constraints programming

The idea of constraint programming is to specify as early as possible a maximum of knowledge about the problem at hand.
This knowledge can be used to prune the search tree and find results with less computation.
A very well known type of constraint programming is constraints over integers with libraries like [`clpfd`](https://www.swi-prolog.org/pldoc/man?section=clpfd).
A very useful byproduct of such library is that it introduce pure arithmetic, with arithmetic operations that can be used in any direction.
That's perfect for what we want to do!

However, our problem domain is not over integers but over reals.
Coordinates or segment lengths are continuous values in the euclidean 2D space.
From my current knowledge about the swi-prolog landscape, I only know of 2 libraries that does constraint programming over reals:

* [`clpqr`](https://www.swi-prolog.org/pldoc/man?section=clpqr) installed by default in the swi-prolog
* [`clpBNR`](https://github.com/ridgeworks/clpBNR) which you can install as a pack in swi-prolog

`clpqr` idea is to delay the evaluation of an operation until all input arguments are ground.
This make the constraint propagation very weak, which is a big problem for us.

On the other hand, `clpBNR` takes another approach, closer to the one used by `clpfd` by using interval arithmetic.
In this case, constraint propagation are much stronger at the expense of results precision.
If you want to know more, you should have a read at the [Guide to `clpBNR`](https://ridgeworks.github.io/clpBNR/CLP_BNR_Guide/CLP_BNR_Guide.html).

To come back to our grammar, `cplBNR` will allow us to use real arithmetic in all directions.
This will come at the expense of efficiency and somewhat imprecise results.

Let's try to rework our `seg_length/2` predicate to use `clpBNR`.

```prolog
:- use_module(library(clpBNR)).

seg_length(seg(point(X1, Y1), point(X2, Y2)), Len) :-
   { Len == sqrt((X2 - X1)**2 + (Y2 - Y1)**2) }.
```

First of all, the braces `{}` wrapping the mathematical expression is to indicate that we are using `clpBNR` arithmetic.
Next, we have replace the `is` operator with `==` equality operator.
With pure arithmetic, it does not make sense anymore to have an evaluation predicate since there are no predefined direction on how to evaluate things.
All you can do is to use comparison predicate such as equality `==` to say that both sides should be equal.

Let's try our new predicate with the most general query:

```prolog
?- seg_length(Seg, Len).
Seg = seg(point(_A, _B), point(_C, _D)),
Len::real(0.0, 1.0Inf),
_C::real(-1.0Inf, 1.0Inf),
_A::real(-1.0Inf, 1.0Inf),
_D::real(-1.0Inf, 1.0Inf),
_B::real(-1.0Inf, 1.0Inf).
```

The last four lines tells us that the segment coordinates can take real values going from -infinity to infinity.
If we look at the `Len` variable, we can already see some constraint propagation happening!
`clpBNR` correctly deduced that `Len` is related to the output of the square root function which can only be positive...
Well, not in math, but in computers, the `sqrt` often only returns the non negative square root only.
In our application of computing the length of a segment, this is exactly what we want, since a negative segment length would not make sense.

Let's try some more queries, by adding some more information about the segment first point and the segment length _after_ the call to `seg_length/2` to see some more constraint propagations:

```prolog
?- seg_length(Seg, Len), Seg = seg(P1, P2), P1 = point(0, 0), Len=1.
Seg = seg(point(0, 0), point(_A, _B)),
Len = 1,
P1 = point(0, 0),
P2 = point(_A, _B),
_A::real(-1, 1),
_B::real(-1, 1).
```

With this, `clpBNR` has correctly deduced that the end point of `Seg` can by anywhere between `(-1, 1)`.
In fact, this second point has to be on the _circle_ of center (0, 0) and radius 1:

{{< rawhtml >}}
<div id="jxgbox2" class="jxgbox" style="width:400px; height:400px;"></div>
<script type="text/javascript">
 var board2 = JXG.JSXGraph.initBoard('jxgbox2', {boundingbox: [-2, 2, 2, -2], axis: true});
 var p = board2.create('point',[0,0], {name: 'P1'});
 var ci = board2.createElement('circle',["P1", 1], {strokeColor:'#00ff00',strokeWidth:2});
</script>
{{< /rawhtml >}}

Now that we have pure real arithmetic, let's go back to our definition of an equilateral triangle:

```prolog
?- phrase(triangle_dcg(equilateral_triangle(A, B, C)), L, R).
A = point(_A, _B), B = point(_C, _D), C = point(_E, _F),
L = [seg(point(_A, _B), point(_C, _D)), seg(point(_C, _D), point(_E, _F)), seg(point(_E, _F), point(_A, _B))|R],
_C::real(-1.0Inf, 1.0Inf),
_A::real(-1.0Inf, 1.0Inf),
_D::real(-1.0Inf, 1.0Inf),
_B::real(-1.0Inf, 1.0Inf),
_E::real(-1.0Inf, 1.0Inf),
_F::real(-1.0Inf, 1.0Inf) .
```

Success!
Can we now _draw_ an equilateral triangle using from a fully ground triangle ?
Here is an example of an equilateral triangle A, B, C of with side length of 2:

* point A (0, 0)
* point B (2, 0)
* point C (1, sqrt(3))

{{< rawhtml >}}
<div id="jxgbox3" class="jxgbox" style="width:400px; height:400px;"></div>
<script type="text/javascript">
 var board = JXG.JSXGraph.initBoard('jxgbox3', {boundingbox: [-1, 4, 4, -1], axis: true});
 var p = board.create('point',[0,0], {name: 'A'});
 var p = board.create('point',[2,0], {name: 'B'});
 var p = board.create('point',[1,1.732051], {name: 'C'});
 var li = board.create('line',['A','B'], {straightFirst:false, straightLast:false, strokeColor:'#000000', strokeWidth:2});
 var li = board.create('line',['B','C'], {straightFirst:false, straightLast:false, strokeColor:'#000000', strokeWidth:2});
 var li = board.create('line',['C','A'], {straightFirst:false, straightLast:false, strokeColor:'#000000', strokeWidth:2});
</script>
{{< /rawhtml >}}

So this *should* work ?

```prolog
?- phrase(triangle_dcg(equilateral_triangle(point(0, 0), point(2, 0), point(1, sqrt(3)))), L, R).
L = [seg(point(0, 0), point(2, 0)), seg(point(2, 0), point(1, sqrt(3))), seg(point(1, sqrt(3)), point(0, 0))|R] .
```

Yes, and we should be able to recognize an equilateral triangle from its segments too:

```prolog
?- {Y == sqrt(3)},
   phrase(triangle_dcg(equilateral_triangle(A, B, C)),
          [seg(point(0, 0), point(2, 0)),
           seg(point(2, 0), point(1, Y)),
           seg(point(1, Y), point(0, 0))]).
Triangle = equilateral_triangle(point(0, 0), point(2, 0), point(1, Y)),
Y:: 1.732050807568877... .
```

> **NOTE**
> We precompute `sqrt(3)` because `clpBNR` does not support expression for attributed variables, only numbers.

Nice!

# Conclusion

In this article, I have introduced a way to write pure prolog grammar to relate geometrical figures with a list of their graphical primitives by combining the DCG notation and constraint programming.
Stay tune for the next blog post of this subject!

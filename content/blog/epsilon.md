---
title: "Epsilon"
date: 2023-11-13T09:09:04+09:00
---

# Epsilon

This blog post is the second of a series on writing a pure prolog grammar to relate graphical primitives with high level semantic constructs.
If you have not already, I advise to read the [first post of the series](https://kwon-young.github.io/blog/2023/triangle) that introduce the context and motivation of the subject at hand.

In the previous post, I have introduced how we can write a pure DCG predicate to relate a list of segments with geometrical figures like triangles and equilateral triangles.
Let's begin right from the last example of recognizing equilateral triangles:

```prolog
:- use_module(library(clpBNR)).

seg_length(seg(point(X1, Y1), point(X2, Y2)), Len) :-
   { Len == sqrt((X2 - X1)**2 + (Y2 - Y1)**2) }.

triangle_cond(triangle(A, B, C), seg(A, B), seg(B, C), seg(C, A)).
triangle_cond(equilateral_triangle(A, B, C), Seg1, Seg2, Seg3) :-
   triangle_cond(triangle(A, B, C), Seg1, Seg2, Seg3),
   seg_length(Seg1, Len),
   seg_length(Seg2, Len),
   seg_length(Seg3, Len).

term(X) -->
   [X].
term(X), [CurX] -->
   [CurX],
   term(X).

triangle_dcg(Triangle) -->
   { triangle_cond(Triangle, Seg1, Seg2, Seg3) },
   term(Seg1),
   term(Seg2),
   term(Seg3).
```

```prolog
?- {Y == sqrt(3)},
   phrase(triangle_dcg(equilateral_triangle(A, B, C)),
          [seg(point(0, 0), point(2, 0)),
           seg(point(2, 0), point(1, Y)),
           seg(point(1, Y), point(0, 0))]).
Triangle = equilateral_triangle(point(0, 0), point(2, 0), point(1, Y)),
Y:: 1.732050807568877... .
```

## Reals

Here, \\( \sqrt{3} \\) is a real number. It is even called an irrational number, meaning that there is no finite finite floating point representation of that number.

One of the immediate consequence of this is that if we copy the number printed by the prolog top level and use it in our query, the query will fail:

```prolog
?- phrase(triangle_dcg(equilateral_triangle(A, B, C)),
          [seg(point(0, 0), point(2, 0)),
           seg(point(2, 0), point(1, 1.732050807568877)),
           seg(point(1, 1.732050807568877), point(0, 0))]).
false.
```

This is because clpBNR is able to distinguish \\( \sqrt{3} \\) and 1.732050807568877 and say that the equality relation `==` does not hold between these two numbers.

We introduced this imprecision by copy pasting the number, but much bigger imprecision could be introduced if the graphical primitives were produced by external components.
Maybe we extracted the graphical primitives from a geometry application, or recognize them from a learned shape detector model.

For example, the music engraving software I am using for my music grammar called Verovio engrave music scores with a precision of 0.05 pixels.
This means that sometimes, symbols or lines are not perfectly aligned as they should but have small defects invisible when the music score is looked at at a reasonable scale.

Logically speaking, if the `Y` coordinate of the second point of our triangle is not exactly \\( \sqrt{3} \\), it should not be an equilateral triangle.
But, as we have seen, for many extra logical reasons, we can be in a situation where we want to recognize, generate and test equilateral triangles with only approximate coordinate values.

In order to deal with such imprecision, we are going to fully forget about the `==` relation and instead use an upper bound imprecision called `Epsilon`.

## Epsilon as an upper bound imprecision

So, let's go back to our `seg_length/2` predicate:

```prolog
seg_length(seg(point(X1, Y1), point(X2, Y2)), Len) :-
   { Len == sqrt((X2 - X1)**2 + (Y2 - Y1)**2) }.
```

The goal is to replace the `==` relation with something less restrictive.
Let's say we want the predicate to succeed if difference between the length computed from the segment and the given length `Len` is small enough.
Let's say if the difference is smaller than 0.1, we consider that this relation should hold.
For this, let's write a predicate called `eps(A, B)` which holds if the distance between A and B is smaller than 0.1:

```prolog
eps(A, B) :-
   { abs(A - B) =< 0.1 }.
```

Since we don't know whether A or B is the bigger one, we use the absolute value of the difference between A and B.

Let's test this predicate with different queries:

```prolog
?- eps(A, B).
A::real(-1.0Inf, 1.0Inf),
B::real(-1.0Inf, 1.0Inf).

?- eps(1, B).
B::real(0.8999999999999999, 1.1).

?- eps(A, 1).
A::real(0.8999999999999999, 1.1).

?- eps(1, 1).
true.

?- eps(1, 1.2).
false.
```

The most general query is not very informative here, but clpBNR does remember the constraints imposed inside the `eps/2` predicate.
The following two queries does shows us more information and correctly narrows down the variable to \\( \pm 0.1 \\) of 1.

The final two queries could have been done with basic prolog arithmetic so no real surprises here.

Let's apply `eps/2` to our original `seg_length/2` predicate:

```
seg_length(seg(point(X1, Y1), point(X2, Y2)), Len) :-
   eps(Len, sqrt((X2 - X1)**2 + (Y2 - Y1)**2)).
```

And retry our previously failing query:

```
?- phrase(triangle_dcg(equilateral_triangle(A, B, C)),
          [seg(point(0, 0), point(2, 0)),
           seg(point(2, 0), point(1, 1.732050807568877)),
           seg(point(1, 1.732050807568877), point(0, 0))]).
A = point(0, 0),
B = point(2, 0),
C = point(1, 1.732050807568877) .
```

As expected, the queries now succeeds !

## Eps as a **configurable** upper bound precision

Let's go back to our `eps/2` predicate.

```prolog
eps(A, B) :-
   { abs(A - B) =< 0.1 }.
```

Using 0.1 as an upper bound imprecision is an arbitrary choice.
What if our application needs a smaller imprecision so that it does not show on the screen ?
What if we want to recognize some triangles, but we don't know how precise was the recognition process of the graphical primitives ?

Well, let's make the upper bound configurable and see what happens:

```prolog
eps(Eps, A, B) :-
   { abs(A - B) =< Eps }.
```

This means we will have to update the whole code and grammar to carry this new variable.
Updating `seg_length/2` is easy:

```prolog
seg_length(seg(point(X1, Y1), point(X2, Y2)), Len, Eps) :-
   eps(Eps, Len, sqrt((X2 - X1)**2 + (Y2 - Y1)**2)).
```

Let's continue with `triangle_cond/2`:

```prolog
triangle_cond(triangle(A, B, C), seg(A, B), seg(B, C), seg(C, A), _).
triangle_cond(equilateral_triangle(A, B, C, Len), Seg1, Seg2, Seg3, Eps) :-
   triangle_cond(triangle(A, B, C), Seg1, Seg2, Seg3, Eps),
   seg_length(Seg1, Len, Eps),
   seg_length(Seg2, Len, Eps),
   seg_length(Seg3, Len, Eps).
```

Okay, we have introduced the use of epsilon for checking the segment length.
We also add a fourth argument to the `equilateral_triangle/4` term to be able to see and specify the length of the sides.

We are almost finished, we just need to update the grammar:

```prolog
triangle_dcg(Triangle, Eps) -->
   { triangle_cond(Triangle, Seg1, Seg2, Seg3, Eps) },
   term(Seg1),
   term(Seg2),
   term(Seg3).
```

And we are done !

Let's now think what we can do with such a formalism.
For now, I have identified three mode of operations:

1. test
2. generate
3. recognize

### Test

In test mode, we can now test the grammar with a set of fully annotated data.
In our current example, we know that the equilateral triangle with points (0, 0), (2, 0), (1, ~1.73) and side length of 2 should match the segments with corresponding points.

Let's try it:

```prolog
?- phrase(triangle_dcg(equilateral_triangle(point(0, 0),
                                            point(2, 0),
                                            point(1, 1.73), 2), Eps),
          [seg(point(0, 0), point(2, 0)),
           seg(point(2, 0), point(1, 1.73)),
           seg(point(1, 1.73), point(0, 0))]).
Eps::real(0.0017757883560709509, 1.0Inf) ;
false.
```

Interesting !
The fact that we can simply provide both side of the equation and test if the code succeeds or fails means that the code is highly testable.
We could make a test suite with different triangles and test them in parallel.
This means that we can later expand the grammar without fear of regressions.

> **Note**:
> Trust me, I have spent my whole PhD writing this kind of graphical grammars with no test framework in place.
> It was a **Nightmare** :(

Here, clpBNR also tells us that this rule would succeed with an upper bound error over 0.0017757883560709509.
However, we still don't know if the real upper is ~0.00177... or higher.
But we can ask clpBNR to find the smallest error which would still allow the query to succeed:

```prolog
?- phrase(triangle_dcg(equilateral_triangle(point(0, 0),
                                            point(2, 0),
                                            point(1, 1.73), 2), Eps),
          [seg(point(0, 0), point(2, 0)),
           seg(point(2, 0), point(1, 1.73)),
           seg(point(1, 1.73), point(0, 0))]),
   global_minimize(Eps, Eps).
Eps:: 0.00177... .
false.
```

With the predicate `global_minimize/2`, clpBNR has gradually reduced the epsilon interval from its upper bound and found that it can not go smaller than 0.00177 while respecting all constraints.
Imagine now that the graphical primitives were extracted from an image.
This means that we can now compute the largest error of the recognition process that is relevant for the semantic reconstruction of the document !
Outside of this work, I have never encountered such a end-goal oriented computation of errors of a recognition process !
Of course, you can compute metrics about the recognition process.
But those metrics are only weakly linked to the quality of the semantic recognition process.

### Recognize

I categorize a query as a recognition query if the graphical primitives are given but not the semantic structure.
Unfortunately, in this mode, if we do not specify a reasonable epsilon, we will not get any useful answers because with a sufficiently big error, any semantic structures allowed by the grammar will be produced.

Fortunately, we can use the previous testing mode on a few annotated data to guestimate a reasonable epsilon value and use that for the recognition process:

```prolog
?- Eps::real(0, 0.00178), phrase(triangle_dcg(equilateral_triangle(A, B, C, Len), Eps),
       [seg(point(0, 0), point(2, 0)),
        seg(point(2, 0), point(1, 1.73)),
        seg(point(1, 1.73), point(0, 0))]).
A = point(0, 0),
B = point(2, 0),
C = point(1, 1.73),
Eps::real(0, 0.0017800000000000001),
Len:: 2.00... .
```

### Generate

Finally, by using a epsilon of 0, we can go back to a perfect logical world and generate graphical structures that perfectly matches a given semantic structures:

```prolog
?- phrase(triangle_dcg(equilateral_triangle(
      point(0, 0), point(2, 0), point(1, sqrt(3)), 2), 0), L).
L = [seg(point(0, 0), point(2, 0)),
     seg(point(2, 0), point(1, sqrt(3))),
     seg(point(1, sqrt(3)) , point(0, 0))] .
```

This mode is mostly useful to test the grammar code and verify its purity and bidirectionality.

## Conclusion

With this article, I have introduced a way to deal with imprecision over reals rejecting the equality relationship.
Instead, we use an explicit imprecision upper bound epsilon and do a comparison over a distance computed between two reals.

From this formulation, I showed how I can measure and constrain the imprecision for the different mode of testing, recognition and generation.

Now, let's say that we extend the grammar to recognize a lot of other kind of geometrical figures.
That means that each of the `{triangle,rectangle,...}_cond` predicates will need epsilon to measures their imprecision as well.
This means that we need to refer to our epsilon all over our grammar !
I think the subject of my next post will be: how to deal with state ?

### Exercise

By the way, have you notice a sneaky use of the `==` relationship in the first clause of `triangle_cond/3` ?

```prolog
triangle_cond(triangle(A, B, C), seg(A, B), seg(B, C), seg(C, A), _Eps).
```

Let me know if you find it and how you would use `eps/3` to replace them !

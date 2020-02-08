---
title: Tuning
permalink: /tuning
sidebar: docs_sidebar
folder: docs
toc: false
---

Determining a good execution schedule for a given rule requires some insight regarding the internal operation of the Datalog engine. In general, a rule like
```
C(X,Z) :- A(X,Y), B(Y,Z).
```
is translated into a loop nest similar to
```
for( e1 : A ) {
    for( e2 : B ) {
        if ( e1[1] == e2[0] ) {
            C.insert( e1[0], e2[1] );
        }
    }
} 
```
The cost of this loop is mainly dominated by the number of times the innermost loop body, comprising the condition and the insertion operation, needs to be evaluated. The objective of finding a good execution schedule is to minimize the number of inner-most loop-body executions.

Obviously, a naive conversion like this would be not very efficient. It would require |A| * |B| iterations of the innermost loop body. Fortunately, the second for loop and the condition can be combined into
```
for( e1 : A ) {
    for( e2 : { e in B | e.[0] == e1.[1] } ) {
        C.insert( e1[0], e2[1] );
    }
} 
```
Now, to speed up the execution, an ordered index on B can utilised to more effectively enumerate all the entries in B where the first component is equivalent to the second component of the currently processed tuple in A. This way, the execution cost is reduced to |A| * |fanout of A in B| which in most cases is considerably smaller.

In an extreme case, e.g. given by
```
C(X,Y) :- A(X,Y), B(X,Y).
```
which would be naively converted to
```
for( e1 : A ) {
    for( e2 : B ) {
        if ( e1[0] == e2[0] && e1[1] == e2[1] ) {
            C.insert( e1[0], e1[1] );
        }
    }
} 
```
the operation is contracted to
```
for( e1 : A ) {
    if ( e1 in B ) {
        C.insert( e1[0], e1[1] );
    }
} 
```
eliminating an entire loop, significantly reducing the number of times the loop body is processed.

## Datastructure
The datastructure for a relation can be specifically chosen by adding an appropriate attribute to the relation declaration, e.g.,
```
.decl A(x:number, y:number) btree
.decl B(x:number, y:number) brie
.decl C(x:number, y:number) eqrel
```
The default datastructure for most relations in Souffle is a B-Tree. A Brie structure can improve performance in some cases, and is more memory efficient for particularly large relations.

`eqrel` is a high performance datastructure optimised specifically for equivalence relations. In the example below, eqrel_fast has the same functionality as eqrel_slow, but with greatly improved performance. 
```
.decl rel1, rel2(x:number)
.decl eqrel_fast(x : number, y : number) eqrel
eqrel_fast(a,b) :- rel1(a), rel2(b).

.decl eqrel_slow(x : number, y : number)
eqrel_slow(a,b) :- rel1(a), rel2(b).
eqrel_slow(a,a) :- rel1(a).             // reflexivity
eqrel_slow(b,a) :- eqrel_slow(a,b).     // symmetry
eqrel_slow(a,c) :- eqrel_slow(a,b), eqrel_slow(b, c).   // transitivity
```

## Profiler

The performance impact of the rule order can be measured using the profiling tool of Souffle. The runtime of a rule, how many tuples are produced by the rule, and the performance behaviour for each iteration of a recursively defined rule can be measured and visualised. In practice, only a few rules are performance critical and need to be considered for performance tuning. 

A more detailed description follows. First, souffle is executed with the profiler log option enabled on a given Datalog specification and a set of input facts. Then the generated log file is opened with souffle profiler. As described in the user manual section of souffle profiler, the profiler can be made to list the rules in the descending order of the total time consumed by each rule of the given Datalog specification. This list is very useful in the sense that one can compare the time taken by each rule with the number of tuples it generated. The rules which consume more time by generating less tuples are the candidates for optimisation. Typically optimising the top 10 rules in the list is sufficient.

A simple way to optimise a rule is to reorder the relations in its body based on the insight explained in the above section (Best Practices). Note that due to the semi-naive evaluation technique some rules are transformed into multiple rules. They are listed as versions of the rule. A manual query planner may be used to reorder predicates corresponding to a particular version of the rule. An example is given below.

```
A(x) :- B(x), C(x).

.plan 1: (2,1)
```
Assume that the relations B and C depend on A. Then the relations A, B and C are mutually recursive. Then the different version of the above rule will look like

```
Version 0: A(x) :- Δ B(x), C(x)
Version 1: A(x) :- B(x), Δ C(x)
```
Δ B(x) includes only the new tuples of B generated in the previous stage of the semi-naive evaluation.

And the statement starting with .plan swaps the relations in the body of the Version 1 of the rule.
Souffle also provides an auto-scheduler which finds orders automatically. However, the auto-scheduler is still experimental and does not produce high-quality orders. Note that in order to fully understand the optimisations, a good understanding of the semi-naive algorithm is required.

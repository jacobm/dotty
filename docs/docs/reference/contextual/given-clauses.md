---
layout: doc-page
title: "Given Parameters"
---

Functional programming tends to express most dependencies as simple function parameterization.
This is clean and powerful, but it sometimes leads to functions that take many parameters and
call trees where the same value is passed over and over again in long call chains to many
functions. Given clauses can help here since they enable the compiler to synthesize
repetitive arguments instead of the programmer having to write them explicitly.

For example, with the [given instances](./delegates.md) defined previously,
a maximum function that works for any arguments for which an ordering exists can be defined as follows:
```scala
def max[T](x: T, y: T) given (ord: Ord[T]): T =
  if (ord.compare(x, y) < 0) y else x
```
Here, `ord` is an _implicit parameter_ introduced with a `given` clause.
The `max` method can be applied as follows:
```scala
max(2, 3) given IntOrd
```
The `given IntOrd` part passes `IntOrd` as an argument for the `ord` parameter. But the point of
implicit parameters is that this argument can also be left out (and it usually is). So the following
applications are equally valid:
```scala
max(2, 3)
max(List(1, 2, 3), Nil)
```

## Anonymous Given Clauses

In many situations, the name of an implicit parameter need not be
mentioned explicitly at all, since it is used only in synthesized arguments for
other implicit parameters. In that case one can avoid defining a parameter name
and just provide its type. Example:
```scala
def maximum[T](xs: List[T]) given Ord[T]: T =
  xs.reduceLeft(max)
```
`maximum` takes an implicit parameter of type `Ord` only to pass it on as an
inferred argument to `max`. The name of the parameter is left out.

Generally, implicit parameters may be defined either as a parameter list `(p_1: T_1, ..., p_n: T_n)`
or as a sequence of types, separated by commas.

## Inferring Complex Arguments

Here are two other methods that have an implicit parameter of type `Ord[T]`:
```scala
def descending[T] given (asc: Ord[T]): Ord[T] = new Ord[T] {
  def compare(x: T, y: T) = asc.compare(y, x)
}

def minimum[T](xs: List[T]) given Ord[T] =
  maximum(xs) given descending
```
The `minimum` method's right hand side passes `descending` as an explicit argument to `maximum(xs)`.
With this setup, the following calls are all well-formed, and they all normalize to the last one:
```scala
minimum(xs)
maximum(xs) given descending
maximum(xs) given (descending given ListOrd)
maximum(xs) given (descending given (ListOrd given IntOrd))
```

## Multiple Given Clauses

There can be several given clauses in a definition. Example:
```scala
def f given (u: Universe) given (x: u.Context) = ...
```
However, all `given` clauses in a definition must come after any normal parameter clauses.
Multiple given clauses are matched left-to-right in applications. Example:
```scala
given global as Universe { type Context = ... }
given ctx as global.Context { ... }
```
Then the following calls are all valid (and normalize to the last one)
```scala
f
(f given global)
(f given global) given ctx
```
But `f given ctx` would give a type error.

## Summoning Instances

A method `the` in `Predef` returns the given instance of a specific type. For example,
the given instance for `Ord[List[Int]]` is produced by
```scala
the[Ord[List[Int]]]  // reduces to ListOrd given IntOrd
```
The `the` method is simply defined as the (non-widening) identity function over a implicit parameter.
```scala
def the[T] given (x: T): x.type = x
```

## Syntax

Here is the new syntax of parameters and arguments seen as a delta from the [standard context free syntax of Scala 3](../../internals/syntax.md).
```
ClsParamClauses     ::=  ...
                      |  {ClsParamClause} {GivenClsParamClause}
GivenClsParamClause ::=  ‘given’ (‘(’ ClsParams ‘)’ | GivenTypes)
DefParamClauses     ::=  ...
                      |  {DefParamClause} {GivenParamClause}
GivenParamClause    ::=  ‘given’ (‘(’ DefParams ‘)’ | GivenTypes)
GivenTypes          ::=  AnnotType {‘,’ AnnotType}

InfixExpr           ::=  ...
                      |  InfixExpr ‘given’ (InfixExpr | ParArgumentExprs)
```

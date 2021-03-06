---
layout: docs
title:  "ANF Transformation using Algebraic Effects"
section: "guides"
---

# ANF Transformation using Algebraic Effects
ANF (or A-normal-form) is a program representation very commonly used
in compilers.
The basic idea is, that all non-trivial terms (in a very wide sense,
those that correspond to computation, or invoke side-effects) are
bound to variables. For example, translating

```tut:book:silent
1 - ((3 - 2) - 4)
```

to ANF gives

```tut:book:silent
var x0 = 3 - 2
var x1 = x0 - 4
var x2 = 1 - x1
x2
```

ANF transformations can be implemented using continuation passing style,
as demonstrated by [Matt Might](http://matt.might.net/articles/a-normalization/).

In this short guide, we will see how algebraic effects can be used to
describe an ANF transformation for a very simple expression language.

```tut:book:silent
trait Exp
case class Lit(n: Int) extends Exp
case class Sub(l: Exp, r: Exp) extends Exp
case class Var(name: String) extends Exp
case class Let(name: String, value: Exp, body: Exp) extends Exp
```

The language features numeric literals, subtraction, let-bindings
and variables. The above example in our small language reads as:

```tut:book:silent
val ex = Sub(Lit(1), Sub(Sub(Lit(3), Lit(2)), Lit(4)))
```

To implement the transformation we define a `Transform` effect that
we use to mark elements of our language as either `value`s or
`computation`s:

```tut:book:silent
import effekt._
trait Transform extends Eff {
  def value(e: Exp): Op[Exp]
  def computation(e: Exp): Op[Exp]
}
```
```tut:book:silent:decorate(.boilerplate)
object Transform {
  def value(e: Exp)(implicit c: Use[Transform]): Control[Exp] =
    use(c) { c.handler.value(e) }

  def computation(e: Exp)(implicit c: Use[Transform]): Control[Exp] =
    use(c) { c.handler.computation(e) }
}
import Transform._
```

We also define a recursive traversal function that uses `Transform`
to visit each node in our tree:

```tut:book:silent
def visit(e: Exp)(implicit h: Use[Transform]): Control[Exp] = e match {

    case v : Var => value(v)
    case l : Lit => value(l)

    case Let(name, v, b) => for {
      vv <- visit(v)
      bv <- visit(b)
      e  <- value(Let(name, vv, bv))(h)
    } yield e

    case Sub(l, r) => for {
      lv  <- visit(l)
      rv  <- visit(r)
      e <- computation(Sub(lv, rv))(h)
    } yield e
}
```

This traversal marks variables and literals as values, traverses the
subcomponents of `Let` and `Sub` and finally marks let bindings as
values and subtraction as computation.

## Intermezzo: Generating fresh names

To define our ANF transformation we need to come up with fresh names
such as `x1`, `x2` above. Generating fresh variable names / symbols
can be seen as an effect, so we define another effect signature for
`SymGen`:

```tut:book:silent
trait SymGen extends Eff {
  def fresh(): Op[String]
}
```
```tut:book:silent:decorate(.boilerplate)
object SymGen {
  def fresh()(implicit c: Use[SymGen]): Control[String] =
    use(c) { c.handler.fresh() }
}
import SymGen._
```

A handler for `fresh` can be implemented using stateful-handlers, as
provided by effekt:

```tut:book:silent
class SymState[R] extends SymGen with Handler.Stateful[R, R, Int] {
  def unit = identity
  def fresh() = state => resume => resume("x" + state)(state + 1)
}
```
```tut:book:silent:decorate(.boilerplate)
def SymState[R](init: Int)(b: Use[SymGen] => Control[R]): Control[R] =
    (new SymState).apply(init)(b)
```

## Defining an ANF handler for `Transform`
With the ability to generate fresh names, we are now ready to define
an ANF transformation as a handler for `Transform`:

```tut:book:silent
class ANF(implicit symGen: Use[SymGen]) extends Transform with Handler.Basic[Exp, Exp] {
  def unit = identity
  def value(e: Exp) = _ => resume => resume(e)(())
  def computation(e: Exp) = _ => resume => for {
    x <- fresh()
    body <- resume(Var(x))(())
  } yield Let(x, e, body)
}
```
```tut:book:silent:decorate(.boilerplate)
def ANF(b: Use[Transform] => Control[Exp])(implicit c: Use[SymGen]): Control[Exp] =
    (new ANF).apply(b)
```

Note that this handler itself is effectful and uses the `SymGen` effect. To do so, it
requires a capability as constructor argument. We only need the `Basic` handler here
and discard the handler state (hence the `_`). For values, we just resume with the
expression unchanged.
However, for computations we generate a fresh name, build the corresponding tree using
the freshly generated variable and wrap it finally in a let-binding.
It is important to note that this let-binding will be introduced at the point where
the ANF handler is used.

Finally the anf-transformation can be defined in terms of the `SymState` handler and
the `ANF` handler:

```tut:book:silent
def anfTransform(e: Exp): Control[Exp] = SymState(0) { implicit sym =>
  ANF { implicit anf => visit(e) }
}
```

Calling `anfTransform` on our running example, we obtain:

```tut
anfTransform(ex).run()
```

which exactly corresponds to our manual translation from above.

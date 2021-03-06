# Poi
a pragmatic point-free theorem prover assistant

```text
=== Poi Reduce 0.3 ===
Type `help` for more information.
> and[not]
and[not]
or			( and[not] => or )
```

To run Poi Reduce from your Terminal, type:

```text
cargo install --example poireduce poi
```

Then, to run:

```text
poireduce
```

### Example

When computing the length of two concatenated lists,
there is a faster way, which is to compute the length of each list and add them together:

```text
> (len . concat)(a, b)
(len · concat)(a, b)
(len · concat)(a)(b)
(concat[len] · (len · fst, len · snd))(a)(b)
(add · (len · fst, len · snd))(a)(b)
<=>  add((len · fst)(a)(b), (len · snd)(a)(b))
> add((len · fst)(a)(b), (len · snd)(a)(b))
add((len · fst)(a)(b), (len · snd)(a)(b))
add((len · fst)(a)(b))((len · snd)(a)(b))
add(len(a))((len · snd)(a)(b))
add(len(a))(len(b))
```

### Introduction to Poi and Path Semantics

In "point-free" or "tacit" programming, functions do not identify the arguments
(or "points") on which they operate. See [Wikipedia article](https://en.wikipedia.org/wiki/Tacit_programming).

Poi is an implementation of a small subset of [Path Semantics](https://github.com/advancedresearch/path_semantics).
In order to explain how Poi works, one needs to explain a bit about Path Semantics.

[Path Semantics](https://github.com/advancedresearch/path_semantics) is an extremely expressive language for mathematical programming,
which has a "path-space" in addition to normal computation.
If normal programming is 2D, then Path Semantics is 3D.
Path Semantics is often used in combination with [Category Theory](https://en.wikipedia.org/wiki/Category_theory), [Logic](https://en.wikipedia.org/wiki/Logic), etc.

A "path" (or "normal path") is a way of navigating between functions, for example:

```text
and[not] <=> or
```

Translated into words, this sentence means:

```text
If you flip the input and output bits of an `and` function,
then you can predict the output directly from the input bits
using the function `or`.
```

In normal programming, there is no way to express this idea directly,
but you can represent the logical relationship as an equation:

```text
not(and(a, b)) = or(not(a), not(b))
```

This is known as one of [De Morgan's laws](https://en.wikipedia.org/wiki/De_Morgan's_laws).

When represented as a commutative diagram, one can visualize the dimensions:

```text
         not x not
      o ---------> o           o -------> path-space
      |            |           |
  and |            | or        |
      V            V           |
      o ---------> o           V
           not            computation
```

This is written in asymmetric path notation:

```text
and[not x not -> not] <=> or
```

In symmetric path notation:

```text
and[not] <=> or
```

Both computation and path-space are directional,
meaning that one can not always find the inverse.
Composition in path-space is just function composition:

```text
f[g][h] <=> f[h . g]
```

If one imagines `computation = 2D`, then `computation + path-space = 3D`.

Path Semantics can be thought of as "point-free style" sub-set of equations.
This sub-set of equations is particularly helpful in programming.

### The Problem of Complexity

Efficient mathematical knowledge useful for programming depends on knowing
the identity of functions. This means that the more knowledge you want to build,
the more functions you need to name and refer to symbolically.

This means that mathematical theories using a "birds-eye view" are not as
useful to solve specific problems, except as a guide to find the solution.
The more general and expressive a theory is, the harder it is to do proof search.

As a consequence, theorem proving along both `computation + path-space` is
much harder than just theorem proving for `computation`.

For example, [Type Theory](https://en.wikipedia.org/wiki/Type_theory) is useful
to check that programs are correct, but for higher categories, it becomes
increasingly hard to ground the semantics while staying efficient and usable.

[Path Semantics](https://github.com/advancedresearch/path_semantics) uses a
different approach, which is based on symbols.
When a symbol is created, the theory "commits" to preserving the "paths"
from the symbol, which is known in [Homotopy Type Theory](https://homotopytypetheory.org/) to correspond to "proofs".
Since the symbols themselves encode this relationship to proofs,
it means that proofs can be arbitrary complex without affecting complexity.

This is different from a pure axiomatic system.
In a pure axiomatic system, the symbols do not have meaning except
the relationship to each other (the axioms).
As a result, you get non-standard interpretations of the Peano axioms.
In Path Semantics, if you say "the natural numbers", you *mean* the natural numbers, not the natural numbers as described by the Peano axioms.
The symbol "the natural numbers" *is the proof* of what you mean,
using the background of path semantical knowledge to interpret it.
This is what the "semantics" in Path Semantics means.

It is possible to express ideas in Path Semantics which are believed to be true,
yet can not be proven to be true in any formal language. Someday, a formal
language might be invented to prove the sentence true, but programmers do not
wait for this to happen. Instead, they default to pragmatic strategies, such as
testing extensively.
For example, the [Goldenbach conjecture](https://en.wikipedia.org/wiki/Goldbach%27s_conjecture)
has been tested up to some limit, so it holds for all natural numbers below that limit.
A pragmatic strategy is what you do when you can not idealize the problem away.

Poi uses a pragmatic approach in its design because a lot of proofs in
Path Semantics requires no or little type checking in the "point-free style".

### Design of Poi

Poi is designed to be used as a Rust library.

It means that anybody can create their own tools on top of Poi,
without needing a lot of dependencies.

Poi uses primarily rewriting-rules for theorem proving.
This means that the core design is "stupid" and will do dumb things like running
in infinite loops when given the wrong rules.

However, this design makes also Poi very flexible, because it can pattern match
in any way, independent of computational direction.
It is relatively easy to define such rules in Rust code.

#### Syntax

Poi uses [Piston-Meta](https://github.com/pistondevelopers/meta) to describe its syntax. Piston-Meta is a meta parsing language for human readable text documents.
It makes it possible to easily make changes to Poi's grammar,
and also preserve backward compatibility.

Since Piston-Meta can describe its own grammar rules, it means that future
versions of Piston-Meta can parse grammars of old versions of Poi.
The old documents can then be transformed into new versions of Poi using synthesis.

#### Core Design

At the core of Poi, there is the `Expr` structure:

```rust(ignore)
/// Function expression.
#[derive(Clone, PartialEq, Debug)]
pub enum Expr {
    /// A symbol that is used together with symbolic knowledge.
    Sym(Symbol),
    /// Some function that returns a value, ignoring the argument.
    ///
    /// This can also be used to store values, since zero arguments is a value.
    Ret(Value),
    /// A binary operation on functions.
    Op(Op, Box<Expr>, Box<Expr>),
    /// A tuple for more than one argument.
    Tup(Vec<Expr>),
    /// A list.
    List(Vec<Expr>),
}
```

The simplicity of the `Expr` structure is important and heavily based on
advanced path semantical knowledge.

A symbol contains every domain-specific symbol and "avatar extensions"
of symbols. An "avatar extension" is a technique of integrating information
processing from building blocks that have no relations for introspection.
This means, that even some variants of `Symbol` are not symbols in a direct sense,
they are put there because they "integrate information" of symbols.
For example, a variable is classified as a `1-avatar` since it "integrates information" of a single symbol or expression. "Avatar extensions" occur
frequently in Path Semantics for very sophisticated mathematical relations, but usually do not need to be represented explicitly.
Instead, they are used as a "guide" to design.
See the paper [Avatar Graphs](https://github.com/advancedresearch/path_semantics/blob/master/papers-wip/avatar-graphs.pdf) for more information.

The `Ret` variant comes from the notation used in [Higher Order Operator Overloading](https://github.com/advancedresearch/path_semantics/blob/master/sequences.md#higher-order-operator-overloading). Instead of describing a value as value,
it is thought of as a function of some unknown input type, which returns a known value. For example, if a function returns `2` for all inputs, this is written `\2`.
This means that point-free transformations on functions sometimes can compute stuff, without explicitly needing to reference the concrete value directly.
See paper [Higher Order Operator Overloading and Existential Path Equations](https://github.com/advancedresearch/path_semantics/blob/master/papers-wip/higher-order-operator-overloading-and-existential-path-equations.pdf) for more information.

The `Op` variant generalizes binary operators on functions,
such as `Composition`, `Path` (normal path),
`Apply` (call a function) and `Constrain` (partial functions).

The `Tup` variant represents tuples of expressions, where a singleton (a tuple of one element) is
"lifted up" one level. This is used e.g. to transition from `and[not x not -> not]` to `and[not]` without having to write rules for asymmetric cases.

The `List` variant represents lists of expressions, e.g. `[1, 2, 3]`.
This differs from `Tup` by the property that singletons are not "lifted up".

#### Representing Knowledge

In higher dimensions of functional programming, the definition of "normalization"
depends on the domain specific use of a theory. Intuitively, since there are
more directions, what counts as progression toward an answer is somewhat
chosen arbitrarily. Therefore, the subjectivity of this choice must be
reflected in the representation of knowledge.

Poi's representation of knowledge is designed for multi-purposes.
Unlike in normal programming, you do not want to always do e.g. evaluation.
Instead, you design different tools for different purposes, using the same
knowledge.

The `Knowledge` struct represents mathematical knowledge in form of rules:

```rust(ignore)
/// Represents knowledge about symbols.
pub enum Knowledge {
    /// A symbol has some definition.
    Def(Symbol, Expr),
    /// A reduction from a more complex expression into another by normalization.
    Red(Expr, Expr),
    /// Two expressions that are equivalent but neither normalizes the other.
    Eqv(Expr, Expr),
}
```

The `Def` variant represents a definition.
A definition is inlined when evaluating an expression.

The `Red` variant represents what counts as "normalization" in a domain specific theory.
It can use computation in the sense of normal evaluation, or use path-space.
This rule is directional, which means it pattern matches on the first expression
and binds variables, which are synthesized using the second expression.

The `Eqv` variant represents choices that one can make when traveling along a path.
Going in one direction might be as good as another.
This is used when it is not clear which direction one should go.
This rule is bi-directional, which means one can treat it as a reduction both ways.

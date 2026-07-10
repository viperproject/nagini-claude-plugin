# Viper Language Reference

This document provides essential reference information about the Viper verification language.

## Overview

Viper is a verification infrastructure for SMT-based program verification. It provides native support for permission logics including separation logic and implicit dynamic frames, making it particularly suited for reasoning about heap-manipulating and concurrent programs.

By default, Viper verifies **partial correctness**: if a program state is reached, specified properties hold at that point. By specifying **decreases clauses**, Viper proves termination, verifying **total correctness**.

## Core Language Elements

### Method Specifications

Every method has preconditions (`requires`) and postconditions (`ensures`):

```viper
method sum(n: Int) returns (res: Int)
  requires 0 <= n
  ensures  res == n * (n + 1) / 2
{
  // implementation
}
```

**Key principles:**
- Preconditions restrict when methods may be safely called
- Postconditions guarantee properties upon return
- The postcondition is the ONLY information callers can use

### Loop Invariants

Loops require invariants that hold before entry, after each iteration, and at exit:

```viper
while(i <= n)
  invariant i <= (n + 1)
  invariant res == (i - 1) * i / 2
{
  // loop body
}
```

## Permission System

### Accessibility Predicates

Permissions to fields use `acc(x.f)`:

```viper
field f: Int

method inc(x: Ref, i: Int)
  requires acc(x.f)
  ensures acc(x.f)
{
  x.f := x.f + i
}
```

An accessibility predicate in a precondition means the caller transfers permission to the callee.

### Exclusive vs. Fractional Permissions

**Exclusive permissions** (`acc(x.f)` or `acc(x.f, write)`) prevent aliasing:
- Holding `acc(x.f)` and `acc(y.f)` guarantees x ≠ y
- Required for writing

**Fractional permissions** allow simultaneous read access:

```viper
method copyAndInc(x: Ref, y: Ref)
  requires acc(x.f) && acc(y.f, 1/2)
  ensures  acc(x.f) && acc(y.f, 1/2)
{
  x.f := y.f + 1
}
```

Permission amounts: fractions (e.g., `1/2`), `write` (exclusive), `none` (zero)

### Quantified Permissions

For unbounded data structures:

```viper
forall n: Ref :: { n.first } n in nodes ==>
  acc(n.first) && (n.first != null ==> n.first in nodes)
```

**Injectivity requirement**: Receiver expressions must be provably injective (different quantifier instances yield different references).

## Predicates

Bundle permissions and properties into reusable, potentially recursive assertions:

```viper
predicate tuple(this: Ref) {
  acc(this.left) && acc(this.right)
}

method setTuple(this: Ref, l: Int, r: Int)
  requires tuple(this)
  ensures tuple(this)
{
  unfold tuple(this)
  this.left := l
  this.right := r
  fold tuple(this)
}
```

### Unfold and Fold

- **`unfold P(...)`**: Exchanges predicate instance for its body
- **`fold P(...)`**: Packages permissions back into predicate form

For recursive structures:

```viper
predicate list(this: Ref) {
  acc(this.elem) && acc(this.next) &&
  (this.next != null ==> list(this.next))
}
```

Recursive predicates are interpreted with least fixpoint semantics: any predicate instance has finite (but unbounded) unfoldings.

## Functions

Define parameterized, side-effect-free expressions for specifications:

```viper
function listLength(l: Ref): Int
  requires list(l)
  ensures  result > 0
{
  unfolding list(l) in
    l.next == null ? 1 : 1 + listLength(l.next)
}
```

**Key differences from methods:**
- Bodies are expressions (not statements)
- Can be used in specifications
- Preconditions may require permissions; postconditions must not
- Results are equated with body expressions

## Magic Wands

`A --* B` represents resources that when combined with A can be exchanged for B:

```viper
package list(tmp) --* list(l1) &&
  elems(l1) == old(elems(l1)[..index]) ++ old[lhs](elems(tmp))
{
  fold list(l1)
}
```

**Package** creates a wand by:
1. Identifying necessary resources (footprint)
2. Verifying combined with LHS they produce RHS
3. Removing resources from current state

**Apply** uses a wand:
```viper
apply list(tmp) --* list(l1)
assert list(l1) // now holds
```

## Self-Framing

Specifications must include permission to all locations they read:
- `acc(x.f) && x.f > 0` is self-framing
- `x.f > 0` alone is NOT self-framing

Viper checks left-to-right: `acc(x.f) && 0 < x.f` succeeds, but `0 < x.f && acc(x.f)` fails.

## Domains

Extend Viper with new types and mathematical functions:

```viper
domain IArray {
  function loc(a: IArray, i: Int): Ref
  function len(a: IArray): Int

  axiom all_diff {
    forall a: IArray, i: Int :: {loc(a, i)}
      first(loc(a, i)) == a && second(loc(a, i)) == i
  }

  axiom length_nonneg {
    forall a: IArray :: len(a) >= 0
  }
}
```

Domain functions are uninterpreted; axioms define mathematical properties.

## Statements

**Assignments:**
- `x := e` — local variable
- `e1.f := e2` — heap location (requires `acc(e1.f)`)
- `x, y := m(...)` — method call with multiple returns
- `x := new(...)` — object creation with field initialization

**Assertions:**
- `assert A` — verify property without removing permissions
- `refute A` — verify property is not provable

**Internal Assertions:**
- `assume A` — assume property without adding permissions
- `exhale A` — verify and remove permissions
- `inhale A` — add permissions and assume properties
Never use internal assertions, they can trivially make verification unsound.

**Control flow:**
- `if (condition) { ... } else { ... }`
- `while (condition) invariant I { ... }`

## Built-in Types

- `Bool` — Boolean values
- `Int` — Mathematical unbounded integers
- `Perm` — Permission amounts
- `Ref` — Object references (with `null`)
- `Seq[T]` — Immutable sequences
- `Set[T]` — Immutable sets
- `Multiset[T]` — Immutable multisets
- `Map[K, V]` — Immutable maps

## Verification Workflow

1. **Method verification** is modular: verify body against contract
2. **Callers** see only signature and specification
3. **Framing** is automatic for permissions not transferred and variables not modified
4. **Loop verification** requires invariants for permissions and values

The verification replaces loops with:
1. Exhale loop invariant
2. Havoc loop targets (assigned variables)
3. Inhale invariant and negated condition

This enables framing: unmodified variables and heap locations not in the invariant retain their values.

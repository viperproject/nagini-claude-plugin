# Nagini Language Reference

Comprehensive reference for writing verified Python programs with Nagini.

## Imports

All Nagini contracts come from the `nagini_contracts` package:

```python
from nagini_contracts.contracts import *
from nagini_contracts.obligations import MustTerminate
from nagini_contracts.io_contracts import *  # For I/O verification
```

Nagini uses mypy for type checking, so to import local files, the source file's directory must be a Python package. This means the directory must contain an (usually empty) `__init__.py` file.

## Type Annotations

Nagini requires type annotations on all function parameters and return types. It uses Python's standard type annotation syntax:

```python
from typing import Optional, List, Tuple

def foo(x: int, y: Optional[MyClass]) -> bool:
    ...
```

For local variables, use modern Python annotation syntax — NOT legacy `# type:` comments:

```python
# Correct:
x: int = 0
found: bool = False

# Wrong (legacy syntax, do not use):
x = 0  # type: int
found = False  # type: bool
```

## Method Specifications

### Preconditions and Postconditions

```python
def method_name(param: Type) -> ReturnType:
    Requires(precondition_expression)
    Ensures(postcondition_expression)

    # ... body ...
```

Multiple `Requires`/`Ensures` are conjoined:

```python
def foo(x: int, y: int) -> int:
    Requires(x >= 0)
    Requires(y >= 0)         # Equivalent to Requires(x >= 0 and y >= 0)
    Ensures(Result() >= 0)

    return x + y
```

#### Permission Framing

Every field access needs permission. Methods must declare what permissions they need and return:

```python
def swap(a: Cell, b: Cell) -> None:
    Requires(Acc(a.value) and Acc(b.value))
    Ensures(Acc(a.value) and Acc(b.value))
    Ensures(a.value == Old(b.value) and b.value == Old(a.value))

    tmp: int = a.value
    a.value = b.value
    b.value = tmp
```

#### Constructors

In `__init__`, write the field assignments before the `Ensures` clauses: mypy infers each field's type from its first assignment, so a contract referencing `self.field` above it fails with `Cannot determine type [has-type]`.

```python
class ListNode:
    def __init__(self, val: int, next: 'Optional[ListNode]') -> None:
        self.val = val
        self.next = next
        Ensures(Acc(self.val) and Acc(self.next) and self.val is val and self.next is next)
```

### Result Reference

Use `Result()` in postconditions to refer to the return value:

```python
def double(x: int) -> int:
    Ensures(Result() == 2 * x)

    return 2 * x
```

### Old Values

Use `Old(expr)` in postconditions to refer to pre-state values:

```python
def increment(self: Counter) -> None:
    Requires(Acc(self.count))
    Ensures(Acc(self.count))
    Ensures(self.count == Old(self.count) + 1)

    self.count = self.count + 1
```

### Previous (Loop)

Use `Previous(var)` in loop invariants to refer to value at previous iteration:

```python
while i < n:
    Invariant(i > 0 and result > Previous(result))  # Strictly increasing
    result += i
    i += 1
```

## Permissions

### Basic Permission

```python
Acc(obj.field)          # Full (write) permission
Acc(obj.field, 1/2)     # Fractional (read) permission
Rd(obj.field)           # Read permission (shorthand for some fraction)
```

### Permission Arithmetic

Permissions are transferred through contracts:

```python
def transfer(src: Cell, dst: Cell) -> None:
    Requires(Acc(src.value))       # Receive full permission to src.value
    Requires(Acc(dst.value))       # Receive full permission to dst.value
    Ensures(Acc(src.value))        # Return permission to src.value
    Ensures(Acc(dst.value))        # Return permission to dst.value
    Ensures(dst.value == Old(src.value))

    dst.value = src.value
```

## Predicates

### Definition

```python
@Predicate
def list_pred(lst: MyList) -> bool:
    return (Acc(lst.value) and Acc(lst.next) and
            Implies(lst.next is not None, list_pred(lst.next)))
```

### Fold and Unfold

```python
Fold(list_pred(node))     # Package concrete permissions into predicate
Unfold(list_pred(node))   # Extract concrete permissions from predicate
```

### Unfolding Expression

Use `Unfolding` to temporarily unfold a predicate within an expression:

```python
@Pure
def get_value(lst: MyList) -> int:
    Requires(list_pred(lst))

    return Unfolding(list_pred(lst), lst.value)
```

## Pure Functions

Functions used in specifications must be marked `@Pure`:

```python
@Pure
def length(lst: Optional[MyList]) -> int:
    Requires(Implies(lst is not None, list_pred(lst)))
    Decreases(list_pred(lst))

    if lst is None:
        return 0
    return Unfolding(list_pred(lst), 1 + length(lst.next))
```

**Restrictions on pure functions:**
- No side effects (no field assignments, no object creation)
- Must return a value
- Can use `Unfolding` to access predicate contents
- Can be recursive — use `Decreases(measure)` for termination (see Termination section)
- Called in specifications by normal function call syntax

**Do not self-reference in postconditions.** A pure function's `Ensures` clause may *not* call the function itself in a postcondition.

State the property in terms of `Result()` and other pure helpers instead. For example, idempotence of a "round down to a block boundary" function should be written algebraically:

```python
# DO NOT: self-reference in own postcondition
@Pure
@ContractOnly
def network_address(addr: int, prefix: int) -> int:
    Requires(0 <= addr and addr < 4294967296)
    Requires(0 <= prefix and prefix <= 32)
    Ensures(network_address(Result(), prefix) == Result())   # rejected

# Do: state the same property via existing pure helpers and Result()
@Pure
@ContractOnly
def network_address(addr: int, prefix: int) -> int:
    Requires(0 <= addr and addr < 4294967296)
    Requires(0 <= prefix and prefix <= 32)
    Ensures(net_id(Result(), prefix) * block_size(prefix) == Result())
```

## ContractOnly Functions

For specification-only functions that don't need an implementation:

```python
@ContractOnly
def abstract_size(obj: MyObj) -> int:
    Requires(valid(obj))
    Ensures(Result() >= 0)
```

### Always add `Decreases` to `@Pure @ContractOnly` functions

There is no body to check a measure against, but callers need it: proving termination of anything that calls the stub — including using it as a specification function in contexts that must terminate — requires a `Decreases` measure on the stub itself.

## Quantification

### Universal Quantifier

Every `Forall` should provide an explicit trigger so Z3 knows when to instantiate it. A trigger is a list of terms (inside `[[...]]`) mentioning all bound variables; the quantifier fires whenever matching terms appear in the proof context.

```python
Forall(int, lambda i: (
    Implies(0 <= i and i < n, arr[i] >= 0),
    [[arr[i]]]  # Trigger
))
```

#### Trigger rules

- Every quantifier needs a trigger; nested quantifiers each need one (not just the innermost).
- Each quantified variable must appear in at least one trigger expression.
- Each trigger expression must mention at least one quantified variable, and must contain some structure beyond the variable itself (typically a function application — a bare variable is not a valid trigger).
- Arithmetic and boolean operators may not appear in trigger expressions.
- Accessibility predicates (`Acc(...)`) may not appear in trigger expressions.
- Avoid triggers where the quantifier body can re-instantiate on the trigger terms — that produces a matching loop and the verifier will time out.

Bad-trigger example (matching loop):

```python
# The body produces a[i + 1], which matches the trigger and re-instantiates indefinitely.
Forall(int, lambda i: (
    Implies(0 <= i and i < len(a) - 1, a[i] >= a[i + 1]),
    [[a[i], a[i + 1]]]
))
```

#### Quantifying over a collection

The first argument to `Forall` does not have to be a type — it can also be a collection value (a `list`, `PSeq`, `PSet`, `range`, etc.), in which case the bound variable ranges over the *elements* of that collection rather than over all values of a type. The element-form quantifier avoids the `0 <= i < len(xs)` guard and triggers on element-level expressions.

```python
xs: List[int] = ...
# Element-form: x ranges over the elements of xs
Assert(Forall(xs, lambda x: (x >= 0, [])))

# Equivalent index-form
Assert(Forall(int, lambda i: (Implies(0 <= i and i < len(xs), xs[i] >= 0), [[xs[i]]])))

# Works for range, PSeq, PSet too
Assert(Forall(range(0, n), lambda x: (x < n, [])))
```

Use the element form when the property is naturally per-element and does not depend on the index; use the index form when you need the index (e.g. to relate `xs[i]` and `xs[i+1]`, or to mix in a second collection at the same position).

### Multi-variable Quantification

For any property mentioning more than one bound variable (sortedness, monotonicity, pairwise relations, matrix predicates, …), prefer `Forall2` (and `Forall3`, …, `Forall6`) over nesting `Forall` calls. `ForallN` takes `N` domain types followed by a lambda of `N` variables, and accepts one trigger list spanning all of them.

```python
# Preferred: single trigger mentioning both variables
Forall2(int, int, lambda i, j: (
    Implies(0 <= i and i <= j and j < len(a), a[i] >= a[j]),
    [[a[i], a[j]]]  # trigger fires when both a[i] and a[j] are in scope
))

# Avoid: nested Forall
Forall(int, lambda i:
    Forall(int, lambda j:
        Implies(0 <= i and i < j and j < n, arr[i] <= arr[j])
    )
)
```

`Forall3` through `Forall6` extend the same pattern to more variables.

## Built-in Verified Types

### Sequences (PSeq)

Immutable mathematical sequences. The supported operations are:

```python
from nagini_contracts.contracts import PSeq

s = PSeq[int]()                   # Empty sequence
s = PSeq(1, 2, 3)                 # Sequence [1, 2, 3] (type inferred from args)
len(s)                            # Length (use in specs)
s[i]                              # Element access
s + t                             # Concatenation
s.take(n)                         # First n elements
s.drop(n)                         # All but first n elements
s.update(i, v)                    # Update at index i
x in s                            # Membership
```

The varargs form `PSeq(x, y, ...)` is usable inline in any spec expression. The empty form `PSeq[int]()` is statement-level only: it cannot appear directly inside `Requires` / `Ensures` / `Invariant` / `Assert` expressions. For `Invariant` / `Assert`, bind to a local first. For `Requires` / `Ensures`, express emptiness with `len(s) == 0`, or build an empty sequence from an existing one with `s.take(0)` or `s.drop(len(s))`.

### Sets (PSet)

Immutable mathematical sets:

```python
from nagini_contracts.contracts import PSet

s = PSet[int]()                   # Empty set (statement-level only)
s = PSet(1, 2, 3)                 # Set {1, 2, 3} (type inferred from args)
x in s                            # Membership
len(s)                            # Cardinality
s | t                             # Union
s & t                             # Intersection
s - t                             # Difference
```

Same constructor rules as `PSeq`: the varargs form is usable inline in specs; the empty `PSet[int]()` is statement-level only.

### Multisets (PMultiset)

```python
from nagini_contracts.contracts import PMultiset

m = PMultiset[int]()              # Empty multiset (statement-level only)
m = PMultiset(1, 2, 2)            # Multiset {|1, 2, 2|} (type inferred from args)
m.num(x)                          # Count of x in m
```

Same constructor rules as `PSeq`: the varargs form is usable inline in specs; the empty `PMultiset[int]()` is statement-level only.

## Built-in Functions with Verified Contracts

Nagini ships verified contracts for many Python built-ins, usable directly in specs and pure functions. This table is the canonical list — never write a custom `@Pure` helper that duplicates an entry:

| Use this | Don't write |
|----------|-------------|
| `abs(x)`, `abs(a - b)` | `abs_diff(a, b)`, custom `abs` |
| `max(a, b)`, `min(a, b)` | `max_of(a, b)`, `min_of(a, b)` |
| `len(xs)` | `list_len(xs)`, manual length recursion over a list/PSeq |
| `x in xs` (`List`, `PSeq`, `PSet`, `PMultiset`) | custom `contains(xs, x)`, existential over indices |
| `xs[i]`, `xs.take(n)`, `xs.drop(n)`, `xs + ys` (`PSeq`) | manual sequence rebuild via recursion |

Only write a custom pure function when no built-in covers the operation.

## Integer Model

`int` is unbounded: arithmetic (`+`, `-`, `*`, `//`, `%`) has no size limits and needs no configuration. Two constructs do have limits:

**Bitwise operations** on `int` are encoded through fixed-width bitvectors sized by the verifier's bitops-width setting (default 8; set per request via the `int_bitops_size` parameter of the verify tools — the width sticks for subsequent requests — or at CLI launch via `--int-bitops-size`). `&`, `|`, `^` require both operands in `[-(2**N), 2**N - 1]` on every application; shifts require a non-negative count and require the operand range only when the count exceeds 64. Out-of-range operands fail with a precondition error naming the flag. Prefer the arithmetic form when one exists — it is unbounded and needs no flag: `x & (2**k - 1)` is `x % 2**k`, `x >> k` is `x // 2**k`, `x << k` is `x * 2**k` (exact equalities for all ints).

**Power expressions**: `**` with a constant exponent is evaluated by unrolling one step per solver instantiation, so only small exponents evaluate (tens, not hundreds); with a symbolic exponent it is essentially opaque without manual lemmas. Exponents must be non-negative. Write large constants as numeral literals (decimal or hex), in code and contracts alike:

```python
MASK64 = 0xFFFFFFFFFFFFFFFF  # == 2**64 - 1; written as `2**64 - 1` it stays an opaque term
```

## Loop Invariants

```python
i = 0
total = 0
while i < n:
    Invariant(0 <= i and i <= n)
    Invariant(total == sum_up_to(items, i))
    Invariant(Acc(list_pred(items)))     # Permission invariant
    total += items[i]
    i += 1
```

**Every loop must have invariants** that:
1. Hold on entry to the loop
2. Are preserved by each iteration
3. Together with loop exit condition, imply the postcondition


### Invariant Structure

Typical loop invariants include:
1. **Bounds**: `0 <= i and i <= n`
2. **Permissions**: `Acc(list_pred(items))` or `Acc(obj.field)`
3. **Progress property**: What's been computed for elements `[0..i)`
4. **Current state**: Properties of loop variables
5. **Pure function facts**: Re-state pure function preconditions that the loop body needs (e.g., `Invariant(is_sorted(ToSeq(a)))`)

### `for` Loops

**Prefer indexed `while` loops over `for x in iterable:`.** Two reasons:

- The iterator holds part of the iterable's `list_pred` for the duration of the loop, which makes it hard to say much about the list itself in the loop invariant.
- There are assorted bugs and rough edges in the iterator translation that cause unexpected framing and permission failures.

The idiomatic replacement is `i = 0; while i < len(xs): ...; i += 1`, with index-form invariants. If you really do need a `for`-loop, Nagini's `tests/functional/verification/test_iterator_list.py` and `tests/obligations/verification/test_for_must_terminate.py` are the authoritative patterns.

## Termination

Nagini has two separate termination mechanisms. **They are not interchangeable.**

| Context | Mechanism | Where it goes |
|---------|-----------|---------------|
| `@Pure` recursive function | `Decreases(measure)` | Between `Requires` and `Ensures` |
| Non-pure recursive method | `Requires(MustTerminate(measure))` | As a `Requires` precondition |
| Loop (any method) | `Invariant(MustTerminate(measure))` | Inside the loop body |

### `Decreases` — `@Pure` functions only

`Decreases` is **only valid inside `@Pure` functions**. Using it in a regular (non-pure) method causes a translation error: `invalid.contract.position`.

**Placement is strict**: `Decreases` must appear **after all `Requires` and before any `Ensures`**. Any other position is a translation error.

```python
@Pure
def factorial(n: int) -> int:
    Requires(n >= 0)
    Decreases(n)          # ← after Requires, before Ensures
    Ensures(Result() >= 0)

    if n == 0:
        return 1
    return n * factorial(n - 1)
```

Tuple measures for lexicographic ordering:

```python
Decreases(a, b)  # a decreases, or a is equal and b decreases
```

For a `@Pure` function recursing over a heap predicate, the measure can be the predicate instance. The recursive call must then sit inside an `Unfolding` of that same instance. If the predicate guards its recursion (e.g. `Implies(l.next is not None, MyList(l.next))`), guard the call with the same condition.

### `MustTerminate` — non-pure methods and loops

For non-pure (regular) methods, use `Requires(MustTerminate(measure))`:

```python
from nagini_contracts.obligations import MustTerminate

def quicksort(arr: List[int]) -> List[int]:
    Requires(Acc(list_pred(arr), 2/3))
    Requires(MustTerminate(2 + len(arr)))   # ← termination measure
    Ensures(Acc(list_pred(arr), 2/3))
    ...
    quicksort(less)   # Nagini verifies len(less) < len(arr) satisfies the bound
    quicksort(more)
```

For loops inside any method:

```python
while condition:
    Invariant(MustTerminate(ranking_expression))
    # ...
```

The value of the loop's MT measure *at exit* is the "budget" available to any code that follows the loop.

### Non-pure methods without `MustTerminate`

If a non-pure method is recursive but has no `Requires(MustTerminate(...))`, Nagini **does not verify its termination** — it simply accepts it. This is acceptable for lemma-style helper methods where you are confident the recursion terminates but do not need Nagini to check it.

### Choosing the measure

Termination measures need only be **well-founded and strictly decreasing across the recursive call / back-edge** — they do not need to be tight. Tight measures make the proof brittle: a later edit that adds one extra recursive call or inserts a helper lemma forces measure bumps in every caller that passes a bound through. When choosing a measure, leave some slack, and if you need to increase it later, anticipate further increases.

## Assert and Assume

```python
Assert(x > 0)          # Checked by verifier (fails if unprovable)
Assume(x > 0)          # Assumed without proof (use sparingly)
```

## Let Bindings

For naming subexpressions in specifications:

```python
Ensures(Let(int, Result() + 1, lambda v: v > 0 and v < 100))
```

## Container Predicates

For working with Python lists as verified containers:

```python
def process_list(items: List[int]) -> int:
    Requires(Acc(list_pred(items)))
    Requires(len(items) > 0)
    Ensures(Acc(list_pred(items)))

    ...
```

The `list_pred` predicate represents ownership of a Python list and its elements.

### Container Mutators

Modeled list mutators: `append`, `extend`, `insert`, `remove`, `reverse`, `copy`, `xs[i] = v`, `xs.pop()` (no-argument form only), `del xs[i]`, and `del xs[i:j]` (any bound may be omitted or negative; no step). For `xs.pop(i)`, capture `xs[i]` and write `del xs[i]` instead. Modeled dict mutators: `d[k] = v`, `d.pop(k)` (no-default form only), `del d[k]`. Modeled set mutators: `add`, `remove`, `clear`. `list.sort`, `list.clear`, and `list.count` are not modeled. `pop`/`del` require a non-empty list, an in-range index, or a present key — the same obligations as the equivalent read.

## Global Variables

A module-level name assigned exactly once is a constant: read it freely in any function. A reassigned global needs `Acc(<name>)` in contracts and a `global` declaration to rebind. A global list/dict/set is a constant binding whose *contents* still need the usual container permission — e.g. `Requires(Acc(list_pred(P1), 1/100))` and matching `Ensures`. Module-init facts do not flow into defs: restate what the body needs (`len(P1) == 3`, element values) in the precondition; module-level callers hold the permissions and facts after initialization.

## Equality
Usually you want to use `is` not `==`. `is` checks for identity (same object), while `==` checks for value equality. For references `is` is almost always what you want. For primitive types like `int` it *should* be interchangeable, but due to some Nagini internals, there can be cases where `==` does not work as expected. So prefer `is`/`is not` for all comparisons.


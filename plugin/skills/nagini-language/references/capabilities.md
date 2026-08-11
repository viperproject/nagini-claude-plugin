# Nagini Capabilities & Common Misconceptions

Nagini and Viper are more expressive than you might assume. Before concluding that a property cannot be expressed or that a workaround is needed, check this list. Each entry corrects a commonly fabricated limitation.

**If you believe Nagini cannot express something and it is not listed here as a confirmed limitation, assume you are wrong.** Try the direct encoding first. Only treat it as a real limitation once you have a concrete error demonstrating it.

## Confirmed Capabilities

Things that Nagini CAN do, despite common misconceptions:

<!-- Add entries in this format:
- **Misconception**: "Nagini cannot do X"
  **Reality**: Nagini can do X. Here's how: `example code or approach`
-->

## Confirmed Limitations

Things that Nagini genuinely cannot do:

<!-- Add entries in this format:
- **Limitation**: description
  **Workaround**: recommended approach
-->

- **Limitation**: Use of Exists() quantifiers. Works in theory but will cause timeouts in practice.
  **Workaround**: Use explicit witnesses or witness functions.

- **Limitation**: Connecting quantified membership facts to the concrete contents of a collection can be difficult or impossible. For example, a set `s` known only through "`x in s` iff `P(x)`" has no terms to trigger the solver, so you cannot derive e.g. the contents `Assert(s == PSet(1))` or its cardinality (`len(s)`) from that alone. 
  **Workaround**: Keep aggregates constructive: build collections operation by operation, or track the count in its own variable updated alongside every mutation. Keep cardinality out of `Decreases` measures — use fuel-bounded recursion instead.

- **Limitation**: Bitwise operators on ints (`&`, `|`, `^`, `<<`, `>>`) encode through an int-to-bitvector bridge that is very expensive for the solver, especially under quantifiers or in loop invariants — proofs that touch them often time out.
  **Workaround**: When values are nonnegative and the width permits, use the arithmetic equivalents instead (`x & 1` -> `x % 2`, `x >> k` -> `x // 2**k`, `x << k` -> `x * 2**k`); they verify orders of magnitude faster.

- **Limitation**: String support beyond the basics (literals, `+`, `len()`, equality). String methods, formatting, and slicing are largely unsupported.
  **Workaround**: Use lists of integers to represent strings when string reasoning is needed.

- **Limitation**: `print()` is stubbed with a single `object` argument — no varargs, no keyword args, no f-strings. Anything beyond a single argument fails with `Unsupported version of builtin function`. 
  **Workaround**: A single concatenated string works fine (`print('Adding ' + name)`). Collapse multi-arg prints into one concatenated string, or drop them.

- **Limitation**: Comprehensions are only partially supported. Single-generator comprehensions translate, but the verifier can prove little about the result.  The body must be pure — statements in the body raise `impure.list.comprehension.body`. Multiple generators (`for x in xs for y in ys`) fail translation.
  **Workaround**: When facts about the resulting collection are needed, rewrite as an explicit loop with invariants.

  **Workaround**: Refactor to remove the import cycle (e.g., split a class that constructs its sub-components into a data-only class plus a separate factory module) so the import can be unconditional.

- **Limitation**: Operations on heap objects like string concatenation (`+`) and `list.copy()` allocate new heap objects and are thus rejected inside `@Pure` functions with `purity.violated`. 
  **Workaround**: They work in regular (non-pure) method bodies. For lists, you can also use translation to a PSeq.

- **Limitation**: User-defined `__lt__`/`__le__`/`__gt__`/`__ge__` dunders are not really supported. In particular they do not work for the `min()` and `max()` builtins or comparison operators (`<`, `<=`, `>`, `>=`).
  **Workaround**: Add explicit comparison-based helpers on the class and rewrite call sites.

- **Limitation**: `for x in iterable:` loops are difficult to verify. The iterator holds part of the iterable's `list_pred` for the loop's duration and references the elements in a way that makes it hard to state anything about it in invariants. On top of that, the iterator translation has some bugs and rough edges that cause unexpected framing and permission failures.
  **Workaround**: Use an indexed `while i < len(xs):` loop instead.
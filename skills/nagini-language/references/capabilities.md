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

- **Limitation**: String support beyond the basics (literals, `+`, `len()`, equality). String methods, formatting, and slicing are largely unsupported.
  **Workaround**: Use lists of integers to represent strings when string reasoning is needed.

- **Limitation**: `print()` is stubbed with a single `object` argument — no varargs, no keyword args, no f-strings. Anything beyond a single argument fails with `Unsupported version of builtin function`. 
  **Workaround**: A single concatenated string works fine (`print('Adding ' + name)`). Collapse multi-arg prints into one concatenated string, or drop them.

- **Limitation**: Re-exports from `__init__.py` (e.g. `from .helpers import foo` so consumers can `from pkg import foo`) crash translation with a `RecursionError` in `nagini_translation/lib/views.py`.
  **Workaround**: Keep `__init__.py` empty (it only needs to exist so mypy treats the directory as a package) and import directly from submodules: `from pkg.helpers import foo`.

- **Limitation**: Comprehensions are only partially supported. Single-generator comprehensions translate, but the verifier can prove little about the result.  The body must be pure — statements in the body raise `impure.list.comprehension.body`. Multiple generators (`for x in xs for y in ys`) fail translation.
  **Workaround**: When facts about the resulting collection are needed, rewrite as an explicit loop with invariants.

  **Workaround**: Refactor to remove the import cycle (e.g., split a class that constructs its sub-components into a data-only class plus a separate factory module) so the import can be unconditional.

- **Limitation**: Operations on heap objects like string concatenation (`+`) and `list.copy()` allocate new heap objects and are thus rejected inside `@Pure` functions with `purity.violated`. 
  **Workaround**: They work in regular (non-pure) method bodies. For lists, you can also use translation to a PSeq.

- **Limitation**: User-defined `__lt__`/`__le__`/`__gt__`/`__ge__` dunders are not really supported. In particular they do not work for the `min()` and `max()` builtins or comparison operators (`<`, `<=`, `>`, `>=`).
  **Workaround**: Add explicit comparison-based helpers on the class and rewrite call sites.

- **Limitation**: `for x in iterable:` loops are difficult to verify. The iterator holds part of the iterable's `list_pred` for the loop's duration and references the elements in a way that makes it hard to state anything about it in invariants. On top of that, the iterator translation has some bugs and rough edges that cause unexpected framing and permission failures.
  **Workaround**: Use an indexed `while i < len(xs):` loop instead.
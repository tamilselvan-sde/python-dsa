# Boolean (bool)

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
> Section: 02 — Data Types
> 🔗 Related: [../01_Python_Basics/operators.md](../01_Python_Basics/operators.md), [dictionary.md](./dictionary.md), [set.md](./set.md) · Back to [README](../README.md)

## 1. What is it?

`bool` is Python's two-valued truth type — `True` and `False`. It is a **subclass of `int`** (`True == 1`, `False == 0`), so it supports all integer arithmetic, but most code treats it as a standalone truth value.

- Constants: `True`, `False` (capitalised — they're keywords since 3.0).
- Logical operators: `and`, `or`, `not`.
- Comparison results (`==`, `<`, `is`, `in`) all return `bool`.
- Every object can be coerced to a bool by `bool(x)` / in an `if` context, via the `__bool__`/`__len__` dunder protocol.

## 2. Why do we use it?

- **Decision making** — `if`, `while`, `assert`, ternary `x if cond else y`.
- **Predicates / filter functions** — `filter(bool, xs)`, `any(xs)`, `all(xs)`.
- **Flags** — toggling features, retries, "isReady" state.
- **Return value of comparisons** — `==`, `<`, `in`, `is`.
- **Short-circuit logic** — combine conditions cleanly without nested `if`s.

## 3. When should I choose it?

| Need | Choose |
|------|--------|
| Two-state truth value | **bool** |
| More than 2 states | `enum.Enum` / sentinel objects |
| Optional value where "missing" ≠ "False" | `Optional[T]` / explicit `None` checks |
| Function as a flag | `bool` parameter with default `False` |
| Reverse-check truthiness | `not x` |

## 4. Syntax

```python
flag = True
done = False

# Coercion
bool(0)             # False
bool([])            # False
bool("text")        # True
bool(None)          # False

# Logical
a and b             # both truthy → b (short-circuits)
a or  b             # either truthy → first truthy (short-circuits)
not a               # negation (always returns True/False)

# Comparisons produce bool
1 < 2               # True
1 == 1.0            # True
[1,2] == [1,2]      # True
x is None           # True/False
"a" in "abc"        # True
```

## 5. Basic Example

```python
is_admin = True
logged_in = False

if is_admin and logged_in:
    print("welcome admin")
else:
    print("access denied")

# Ternary
status = "active" if is_admin else "guest"

# Short-circuit guard
config = config or default_config

# Boolean as int (subclass of int)
sum([True, True, False])   # 2
True + True                # 2
False * 3                 # 0
```

## 6. Step-by-Step Dry Run

Evaluate `a and b` with `a = True`, `b = "hello"`:

```
Step 1: evaluate a → True (truthy)
Step 2: 'and' requires both truthy → continue to b
Step 3: evaluate b → "hello" (truthy)
Step 4: 'and' returns the LAST evaluated operand (b)
Result: "hello"   ← NOT a boolean!
```

Evaluate `0 or [] or "fallback"`:
```
0   → falsy, skip
[]  → falsy, skip
"fallback" → truthy, return
Result: "fallback"
```

Evaluate `not "x"`:
```
"x" is truthy  →  not "x" = False
```

## 7. Built-in Methods / Operations

`bool` is a subclass of `int`; it inherits `int`'s methods but has none of its own besides what every object has. The interesting API is **operations on booleans**, not methods on the `bool` class.

| Operation | Purpose | Returns |
|-----------|---------|---------|
| `bool(x)` | Coerce to truth value | `True` / `False` |
| `x and y` | Logical AND, **short-circuit** | last evaluated (truthy) or first falsy |
| `x or y`  | Logical OR, **short-circuit** | first truthy or last falsy |
| `not x`   | Negation | always `bool` |
| `x == y`  | Equality | `bool` |
| `x is y`  | Identity | `bool` |
| `x in c`  | Membership | `bool` |
| `x < y`, `>`, `<=`, `>=`, `!=` | Comparison | `bool` |
| `any(it)` | True if any element truthy | `bool` |
| `all(it)` | True if every element truthy | `bool` |

### Truthy / Falsy table

| Truthy | Falsy |
|--------|-------|
| `True` | `False` |
| non-zero numbers (`1`, `-1`, `3.14`) | `0`, `0.0`, `0j` |
| non-empty `str`, `list`, `tuple`, `set`, `dict` | `""`, `[]`, `()`, `set()`, `{}` |
| any user object with `__bool__ → True` or `__len__ > 0` | `None`, `...` (Ellipsis is truthy though) |
| — | custom object with `__bool__` → False or `__len__ → 0` |

Example:
```python
bool(0)         # False
bool(0.0)       # False
bool([])        # False
bool([False])   # True  — list has 1 element, even if that element is falsy
bool({})        # False  ← confusing: `{}` literal is a dict, not a set!
bool(set())     # False
bool("False")   # True  — non-empty string, even though it spells "False"
```

### Short-circuit evaluation

```python
# Guard against None attribute access
user and user.name      # None if user is None, else user.name

# Default fallback
config = user_config or default_config

# Lazy side-effect only if needed
flag and expensive_operation()

# Truthy-default idiom
port = override_port or 8080   # but be careful: 0 is falsy!
# Safer when 0 is a valid override:
port = override_port if override_port else 8080
port = override_port or 8080   # 0 → 8080 (intended? maybe not)
```

### Comparison chaining

```python
1 < x < 10         # equivalent to (1 < x) and (x < 10), x evaluated once
1 < 2 == 2         # True  ( (1<2) and (2==2) )
1 < 2 > 3          # False
```

### `any` / `all`

```python
any([False, 0, "", "x"])    # True  — "x" is truthy, short-circuits
all([True, 1, [0]])         # True  — every element truthy
all([])                     # True  — vacuously true
any([])                     # False — no truthy element exists
```

## 8. Interview Example — Palindrome validation

```python
def is_palindrome(s: str) -> bool:
    s = ''.join(c.lower() for c in s if c.isalnum())
    return s == s[::-1]
```

`return` yields a `bool` — note how the comparison `s == s[::-1]` directly produces the answer without an `if/else`.

## 9. When NOT to use

- **"False" is a valid value** with a distinct meaning from "missing" — use `None` or `Optional[bool]` and check `x is None`.
- **You need to distinguish False, 0, "", [], None** — `== False` collapses all falsies; use `is`.
- **Counting truthies** — `sum(map(bool, xs))` works (because `bool` is `int`), but `sum(1 for x in xs if x)` is clearer.
- **Do not abuse booleans for ternary state** — use `enum.Enum` instead.

## 10. Common Mistakes

1. **`if x == None`** — use `if x is None`. `None` is a singleton; `==` can be overridden and lie.
2. **`if x:` when 0 is a legitimate value** — `0` is falsy! Use `if x is not None:` for `Optional` checks.
3. **Comparing with `is` for primitive values** — `is` checks identity; `256 is 256` may work (cached ints) but `1000 is 1000` is **not** guaranteed. Always use `==` for values.
4. **Chained comparisons without parentheses ambiguity** — `not a == b` parses as `not (a == b)`; `not a and b` parses as `(not a) and b`.
5. **`bool == 1` ambiguity** — `True == 1` is True (subclass of int) — surprising if you meant strict type checks.
6. **`any([])` returns `False`, `all([])` returns `True`** — vacuous truth, often counter-intuitive.
7. **Assuming `and`/`or` return `bool`** — they return the **operands**, not `True`/`False`. Wrap in `bool(...)` or use `not not (...)` when you really need a bool.
8. **`if x == False:`** — bad style. Use `if not x:` (or `if x is False:` to distinguish from `None`).
9. **Floating point `==`** — `0.1 + 0.2 == 0.3` is False due to FP error. Use `math.isclose` or `abs(a - b) < eps`.
10. **Mutating in `and`/`or` chains** — short-circuit can skip side effects you expected. Keep effects out of boolean expressions.

## 11. Memory Tricks

- **"Empty = False"** — every empty container/zero/None is falsy.
- **"But [False] is True"** — a non-empty container is truthy regardless of contents.
- **"AND returns last truthy, OR returns first truthy"** — operands are returned, not booleans.
- **"NOT always returns bool"** — only `not` guarantees bool output.
- **"is for None, == for values"** — `x is None` is the Pythonic idiom.
- **"Any short-circuits on True; All short-circuits on False"**.

## 12. Interview Shortcuts

- **Default value**: `x = x or default` (beware 0!).
- **Safe attribute access**: `obj and obj.attr and obj.attr.value`.
- **Count truthies**: `sum(map(bool, xs))` (works because `True == 1`).
- **Boolean indexing on list**: `[v for v in xs if v]`.
- **Ternary**: `result = a if cond else b`.
- **Chain**: `0 <= idx < len(arr)` (idiomatic; idiomatic Python).
- **Negated membership**: `x not in c` (NOT `not x in c`).
- **Skip None branch first**: `if obj is None: return None` — guard early.
- **`functools.reduce(operator.or_, flags)`** to combine truth bits.
- **For ambiguous truthiness**, always ask: "is 0 a valid answer? is empty list a valid answer?"

## 13. Cheat Sheet Table

| Operator / Function | Use |
|---------------------|-----|
| `True`, `False` | The two boolean constants |
| `bool(x)` | Coerce any object to bool |
| `x and y` | Short-circuit AND |
| `x or y` | Short-circuit OR |
| `not x` | Negation (always bool) |
| `x == y`, `!=` | Value equality |
| `x is y` | Identity (None, singletons, sentinels) |
| `x in c` | Membership |
| `any(it)` | Any truthy → True |
| `all(it)` | All truthy → True |
| `a < x < b` | Chained comparison |
| `True + True` | = 2 (bool subclass of int) |

## 14. Time Complexity Table

| Operation | Complexity |
|-----------|-----------|
| `bool(x)` | O(1) |
| `and` / `or` eval | O(eval of operands) — short-circuits |
| `not x` | O(1) |
| `==`, `is` | O(1) typically; `==` may be O(n) for sequences/structs |
| `in` on list | O(n) |
| `in` on set / dict | O(1) avg |
| `any(it)` / `all(it)` | O(len(it)) worst, short-circuits |

## 15. Visual Diagram (ASCII)

### Truth table

```
   ┌─────┬─────┬──────┬─────┬──────┬──────┐
   │  A  │  B  │ A&B  │ A|B │ A^B* │  ¬A  │
   ├─────┼─────┼──────┼─────┼──────┼──────┤
   │  T  │  T  │  T   │  T  │  F   │  F   │
   │  T  │  F  │  F   │  T  │  T   │  F   │
   │  F  │  T  │  F   │  T  │  T   │  T   │
   │  F  │  F  │  F   │  F  │  F   │  T   │
   └─────┴─────┴──────┴─────┴──────┴──────┘
   * Python has no XOR operator on bool; use (A and not B) or (B and not A),
     or simply bool(A) != bool(B).
```

### Short-circuit decision flow

```
                  A and B
                    │
            ┌───────┴───────┐
        A falsy?          A truthy?
            │                │
            ▼                ▼
        return A         evaluate B
                                    │
                                    ▼
                                return B
```

```
                  A or B
                    │
            ┌───────┴───────┐
        A truthy?         A falsy?
            │                │
            ▼                ▼
        return A         evaluate B
                                    │
                                    ▼
                                return B
```

### Truthiness evaluation flow (used by `if`, `while`, `bool()`)

```
        object x
           │
           ▼
   ┌────────────────┐     defined  ┌──────────────┐
   │ has __bool__?  │────────────▶│ call __bool__ │──▶ bool
   └────────────────┘             └──────────────┘
           │ not defined
           ▼
   ┌────────────────┐     defined  ┌──────────────┐
   │ has __len__?  │────────────▶│ call __len__  │──▶ 0? False : True
   └────────────────┘             └──────────────┘
           │ not defined
           ▼
              return True  ← all other objects are truthy
```

## 16. Beginner Notes

> **Remember:**
> - `True` and `False` are capitalized keywords.
> - Empty + zero + `None` → falsy. Everything else → truthy.
> - `and`/`or` return **operands**, not boolean — only `not` always returns bool.
> - Use `is None` for None checks, never `== None`.
> - Watch out: `0`, `""`, `[]`, `{}`, `set()`, `None`, `False` are all falsy.
> - `bool` is a subclass of `int` — `True + True == 2`.
> - `any([])` is `False`; `all([])` is `True`.

## 17. FAANG Tips

1. **Always use `is` with `None`** — the canonical Python idiom: `if x is None`.
2. **`if x:` is fine for collections** unless `0`/empty is a valid value to preserve — then `if x is not None:`.
3. **Avoid `not not x`** — prefer `bool(x)`.
4. **Comparisons return bool** — you can directly `return a == b` instead of wrapping in `if`.
5. **`any`/`all` with generators** let you short-circuit expensive checks: `any(expensive(x) for x in xs)`.
6. **Don't overload a single bool as a tri-state** — use enum.
7. **In type hints, `bool` ≠ `int`** even though they're interchangeable at runtime — be explicit.
8. **Comparing floats**: `from math import isclose`; never `==`.
9. **Bitwise `&`, `|`, `~` differ from logical `and`, `or`, `not`** — they always return ints and have higher precedence; use parentheses to be safe.
10. **`match`/`case` patterns** (3.10+) use equality / structural matching, not boolean coercion.

## 18. Practice Problems

### Easy
1. **Detect Capital** — LeetCode 520
2. **Valid Palindrome** — LeetCode 125
3. **Power of Two** — LeetCode 231 (`n > 0 and n & (n-1) == 0`)

### Medium
4. **Prime Palindrome** — LeetCode 866
5. **Stone Game** — LeetCode 877 (boolean return trick)

### Hard
6. **Regular Expression Matching** — LeetCode 10
7. **Wildcard Matching** — LeetCode 44
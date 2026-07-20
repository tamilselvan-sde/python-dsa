# reduce() in Python

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com
> Section: 04 — Functions
> 🔗 Related: [functions.md](./functions.md) · [lambda.md](./lambda.md) · [map.md](./map.md) · [filter.md](./filter.md) · [zip.md](./zip.md) · [functools](https://docs.python.org/3/library/functools.html) · Back to [README](../README.md)

---

## 1. What is it?

`functools.reduce(function, iterable, initializer=None)` is a **higher-order function** that **applies a two-argument function cumulatively** to the items of `iterable`, reducing them **left to right** to a **single value**.

```python
from functools import reduce
reduce(lambda a, b: a + b, [1, 2, 3, 4])    # 10
```

It walks step-by-step:

```
((((1) + 2) + 3) + 4) = 10
```

Key points:

- Import from `functools` (it's **not** a builtin in Python 3).
- `function` must take **two arguments**: an accumulator and the next item.
- `initializer` (optional) seeds the accumulator; without it the first item is used.
- Returns the final accumulator.
- Reduces **left → right**.
- `functools` also offers `reduce`'s siblings: `lru_cache`, `partial`, `wraps`.

## 2. Why do we use it?

- **Collapse a sequence to a single value** when no built-in (like `sum`, `max`, `min`) fits.
- Operations like **product**, **flatten nested lists**, **merge dicts cumulatively**, **compute GCD chain**, **left-fold parsing states**.
- **Functional programming** style: pipeline of `map` → `filter` → `reduce`.
- Avoids an explicit mutable accumulator + `for` loop.

## 3. When should I choose it?

| Goal | Use `reduce`? | Alternative |
|------|----------------|-------------|
| Sum all numbers | ❌ Use `sum(xs)` | `sum(xs)` |
| Product of all numbers | ⚠️ Consider `math.prod(xs)` (3.8+) | `reduce(mul, xs)` |
| Min / Max | ❌ `min(xs)`, `max(xs)` | — |
| All truthy | ❌ `all(xs)` | — |
| Any truthy | ❌ `any(xs)` | — |
| Build a dict from pairs | ⚠️ dict comprehension is clearer | `{k: v for (k, v) in pairs}` |
| Flatten a list of lists | ⚠️ `itertools.chain.from_iterable` | `sum(xs, [])` |
| Cumulative merge requiring custom binary op | ✅ Yes | `for` loop |
| Compose functions: `f3(f2(f1(x)))` | ✅ Yes | explicit composition |
| Walk a state machine over tokens | ✅ Yes | explicit `for` |
| Anything where clarity suffers | ❌ write `for` loop | — |

## 4. Syntax

```python
from functools import reduce
reduce(function, iterable[, initializer])
```

- `function(accumulator, item)` → returns the **new** accumulator.
- `iterable` — the sequence of items; must not be empty unless `initializer` is given.
- `initializer` — initial accumulator; **default is None** meaning the first item becomes the initial accumulator (skipping one function call).
- Returns: the final accumulator.

### Signature

```python
functools.reduce(function, iterable, initializer=_NO_INIT)
```

## 5. Basic Example

### 5.1 Sum

```python
from functools import reduce
reduce(lambda a, b: a + b, [1, 2, 3, 4])        # 10
```

Same as:

```python
total = 0
for n in [1, 2, 3, 4]:
    total += n
```

### 5.2 Product (no initializer)

```python
reduce(lambda a, b: a * b, [1, 2, 3, 4])       # 24
```

### 5.3 With initializer (safe for empty lists!)

```python
reduce(lambda a, b: a * b, [1, 2, 3, 4], 1)    # 24
reduce(lambda a, b: a * b, [], 1)              # 1   ← safe!
reduce(lambda a, b: a * b, [])                 # ❌ TypeError — empty iterable
```

### 5.4 Find max

```python
reduce(lambda a, b: a if a > b else b, [3, 1, 4, 1, 5, 9, 2])   # 9
```

## 6. Step-by-Step Dry Run

`reduce(lambda a, b: a + b, [1, 2, 3, 4], 0)`:

| Step | accumulator `a` | item `b` | `a + b` → new acc |
|------|------------------|----------|---------------------|
| init | 0 (initializer) | — | 0 |
| 1 | 0 | 1 | 1 |
| 2 | 1 | 2 | 3 |
| 3 | 3 | 3 | 6 |
| 4 | 6 | 4 | 10 |

Final: `10`.

Without initializer, `reduce` starts with the first item:

`reduce(lambda a, b: a + b, [1, 2, 3, 4])`:

| Step | `a` | `b` | new acc |
|------|-----|-----|---------|
| 1 | 1 | 2 | 3 |
| 2 | 3 | 3 | 6 |
| 3 | 6 | 4 | 10 |

## 7. Built-in Methods / Idioms

### 7.1 Sum, product, min/max

```python
from functools import reduce
from operator import add, mul

reduce(add, [1, 2, 3, 4])            # 10
reduce(mul, [1, 2, 3, 4])            # 24
reduce(max, [3, 1, 4, 1, 5, 9, 2])   # 9
reduce(min, [3, 1, 4, 1, 5, 9, 2])   # 1
```

In practice prefer `sum`, `math.prod` (Python 3.8+), `max`, `min`.

### 7.2 Flatten a list of lists

```python
reduce(lambda a, b: a + b, [[1,2],[3],[4,5,6]])   # [1,2,3,4,5,6]
```

But this is **O(n²)** due to repeated concatenation. Prefer:

```python
from itertools import chain
list(chain.from_iterable([[1,2],[3],[4,5,6]]))    # faster
```

Or list comprehension: `[x for sub in nested for x in sub]`.

### 7.3 Merge dicts (later wins)

```python
dicts = [{"a":1}, {"b":2}, {"a":3}]
reduce(lambda a, b: {**a, **b}, dicts)             # {'a': 3, 'b': 2}
```

With an initializer:

```python
reduce(lambda a, b: {**a, **b}, dicts, {})        # safe for empty list
```

### 7.4 Compose functions

```python
def compose(*funcs):
    def compose_two(f, g):
        return lambda x: f(g(x))
    return reduce(compose_two, funcs)             # f3 ∘ f2 ∘ f1
```

Then `f1(x)` runs first, `fN` last.

The composition order is **last function first** — typical mathematical order.

### 7.5 GCD chain

```python
from math import gcd
reduce(gcd, [12, 18, 24, 36])    # 6
```

### 7.6 LCM chain

```python
from math import gcd
def lcm(a, b):
    return a * b // gcd(a, b)
reduce(lcm, [4, 6, 8])            # 24
```

### 7.7 Build a Counter-style dict

```python
words = ["a","b","a","c","b","a"]
reduce(lambda d, w: {**d, w: d.get(w, 0) + 1}, words, {})
# {'a': 3, 'b': 2, 'c': 1}
```

### 7.8 Multi-dimensional indexing

```python
matrix = [[1,2,3],[4,5,6],[7,8,9]]
reduce(lambda acc, i: acc[i], [1, 2], matrix)        # 6 → matrix[1][2]
```

### 7.9 Left → right reduction order (default)

`reduce(f, [1, 2, 3, 4])` is equivalent to `f(f(f(1, 2), 3), 4)`.

### 7.10 Right fold: with initial reversed (Python 3.8+ offers no built-in right fold)

If you need right-to-left fold, just reverse and provide an initializer:

```python
def foldr(f, xs, init):
    return reduce(lambda acc, x: f(x, acc), reversed(xs), init)

foldr(lambda x, acc: x + acc, [1, 2, 3, 4], 0)   # 10  (same for sum but differs for subtraction)
```

For subtraction you'd see `foldr(-, [1,2,3], 0)` = `1 - (2 - (3 - 0)) = 2`, while `reduce(-, [1,2,3])` = `(1 - 2) - 3 = -4`.

## 8. Interview Example

**Problem (Apple-style):** Flatten a nested list of arbitrary depth.

```python
def flatten(xs):
    return reduce(
        lambda acc, x: acc + (flatten(x) if isinstance(x, list) else [x]),
        xs,
        []
    )

flatten([1, [2, [3, [4, 5]]], 6])   # [1, 2, 3, 4, 5, 6]
```

But the iterative version is faster (no `+` quadratic) and clearer:

```python
def flatten(xs):
    out = []
    for x in xs:
        if isinstance(x, list):
            out.extend(flatten(x))
        else:
            out.append(x)
    return out
```

**Evaluate Reverse Polish Notation** (LeetCode 150):

```python
def evalRPN(tokens):
    ops = {
        "+": lambda a, b: a + b,
        "-": lambda a, b: a - b,
        "*": lambda a, b: a * b,
        "/": lambda a, b: int(a / b),
    }
    stack = []
    for t in tokens:
        if t in ops:
            b = stack.pop()
            a = stack.pop()
            stack.append(ops[t](a, b))
        else:
            stack.append(int(t))
    return stack[0]
```

A **stack**-based reduce — the function folds tokens but maintains a stack of intermediate values.

## 9. When NOT to use

- **Anything with a built-in** — `sum`, `math.prod`, `min`, `max`, `any`, `all`, `len` — always prefer those.
- **When a simple `for` loop is clearer** — interviewers expect readable code; `reduce`'s hidden state can confuse.
- **Adding lists** with `reduce(lambda a, b: a + b, lists_of_lists, [])` is **O(n²)** because of repeated concatenation.
- **Building a Counter** — just `collections.Counter(xs)` — one line, no `reduce`.
- **When you need access to indices** — use `enumerate` ([enumerate.md](./enumerate.md)).
- **When performance matters** — built-in `sum`, `math.prod` are C-optimized.

## 10. Common Mistakes

1. **Forgetting to import**: `reduce` is not a builtin anymore — `from functools import reduce`.
2. **Empty iterable without initializer**:
   ```python
   reduce(lambda a,b: a+b, [])                  # ❌ TypeError
   reduce(lambda a,b: a+b, [], 0)               # ✅ 0
   ```
3. **Operator order**: `lambda a, b: a - b` gives `((1 - 2) - 3)` — different from right-fold for non-associative ops.
4. **Mutating the accumulator in place** returning `None`:
   ```python
   reduce(lambda a, b: a.append(b) or a, [[1],[2]], [])   # works due to `or a`, but ugly
   ```
   Prefer `list.extend` with a `for` loop or `itertools.chain`.
5. **Using `+` on lists** creates O(n²) flattening.
6. **Passing a lambda that returns the wrong type** — accumulator type changes mid-reduction.
7. **Confusing `reduce` with `accumulate`**: `itertools.accumulate` yields **intermediate** prefixes, not a single value — often what you want.

## 11. Memory Tricks

- `reduce(fn, xs, init)` ⇒ "keep folding left": `((((init ⊕ x₁) ⊕ x₂) ⊕ x₃)…)`.
- Picture a **snowball**: rolling left to right accumulating one item at a time.
- "Reduce = squeeze one answer out of many."
- Recall siblings: `sum` → add; `math.prod` → multiply; `all`/`any` → booleans; quickest mental shortcuts point to **prefer built-ins first**.

## 12. Interview Shortcuts

- **Default to `sum`/`math.prod`/`min`/`max`** — they are faster and clearer than any `reduce`.
- If you really need a custom fold, **add an initializer** so empty inputs don't crash.
- **Import** `from functools import reduce` first thing — interviewers verify.
- Show equivalent `for`-loop alongside reduce to demonstrate understanding.
- Use `operator.add` / `operator.mul` for slight speedup + cleaner idiomatic form.
- If asked "find longest increasing subsequence" — don't reach for `reduce` — there's an O(n log n) DP-style binary search approach.

## 13. Cheat Sheet Table

| Use | Form | Better |
|----|----------|---------|
| Sum | `reduce(add, xs)` | `sum(xs)` |
| Product | `reduce(mul, xs)` | `math.prod(xs)` (3.8+) |
| Max | `reduce(max, xs)` | `max(xs)` |
| Min | `reduce(min, xs)` | `min(xs)` |
| All | `reduce(and_, xs, True)` | `all(xs)` |
| Any | `reduce(or_, xs, False)` | `any(xs)` |
| Flatten | `reduce(lambda a,b: a+b, lol)` | `itertools.chain.from_iterable` |
| Compose | `reduce(compose, funcs)` | — |
| Merge dicts | `reduce(lambda a,b: {**a,**b}, ds, {})` | — |
| GCD chain | `reduce(gcd, xs)` | — |

## 14. Time Complexity Table

| Operation | Cost |
|-----------|------|
| `reduce(f, xs)` creation | O(1) |
| per step | O(1) + O(f) call |
| total | O(n · cost of f) |
| memory | O(1) (only accumulator) |
| Using `+` for list flatten | O(n²) — repeated list concatenation |
| `itertools.chain.from_iterable` | O(n) |
| built-in `sum` | O(n) (C-optimized) |

## 15. Visual Diagram (ASCII)

### Left fold snowball

```
   items:     1      2      3      4
              │      │      │      │
   init:0  ───┴──▶ ──┴──▶ ──┴──▶ ──┴──▶  final: 10
   acc:    0   →   1   →   3   →   6   →   10
           ↑       ↑       ↑       ↑
        init      f(0,1) f(1,2) f(3,3) f(6,4)

   Function applied: (( ((0 + 1)) + 2 ) + 3) + 4 = 10
```

### Without initializer (uses first item as seed)

```
   items:     1      2      3      4
   acc:    1   →   3   →   6   →   10
                  ↑       ↑       ↑
               f(1,2)  f(3,3)  f(6,4)

   Missing the initial f(0,1) call.
   Empty xs → TypeError!
```

### Compose (function composition)

```
   funcs:  [f3, f2, f1]
           │    │    │
   init: identity ──▶  ──┐
                         │
   composed(x) = f3(f2(f1(x)))
                      └─ left fold builds this
```

## 16. Beginner Notes

> Remember:
> - `reduce` needs to be imported: `from functools import reduce`.
> - It folds an iterable **left to right** into one value using a 2-arg function.
> - **Always pass an `initializer`** if you might receive an empty iterable.
> - Reach for built-ins `sum`, `math.prod`, `min`, `max`, `any`, `all` first — faster and clearer.
> - For **flattening lists**, use `itertools.chain.from_iterable`, not `reduce(+, ...)`.
> - The accumulator's accumulator? Treasure plain `for` loop over fancy `reduce` when clarity matters.

## 17. FAANG Tips

- **Use built-ins** — `sum`, `math.prod`, `max`, `min` — `reduce` is rarely the right interview answer.
- `reduce`'s real niche: function composition, left-state parsing, **custom merge of dicts/objects**, **composing validators**.
- When asked "**functional programming**", the righeous answer: chain `map` → `filter` → `reduce` with `lambda` — show trio fluency ([map](./map.md) [filter](./filter.md) [lambda](./lambda.md)).
- **Add an initializer** defensively — saves you from empty-input crashes and signals thoughtful code.
- Use **`operator.add`, `operator.mul`, `operator.iconcat`** for cleaner idiomatic reduce chains.
- Pair with `itertools.accumulate` when **intermediate** values are interesting (running totals, cumulative sums).
- Right-fold (`foldr`) is not built-in; demonstrate `reduce(lambda acc, x: f(x, acc), reversed(xs), init)` cleanly when asked.

## 18. Practice Problems

**Easy**
1. [LeetCode 258 — Add Digits](https://leetcode.com/problems/add-digits/) — recursive reduce over digits.
2. [LeetCode 257 — Binary Tree Paths](https://leetcode.com/problems/binary-tree-paths/) — reduce strings.
3. [LeetCode 1360 — Number of Days Between Two Dates](https://leetcode.com/problems/number-of-days-between-two-dates/) — sum days.

**Medium**
4. [LeetCode 150 — Evaluate Reverse Polish Notation](https://leetcode.com/problems/evaluate-reverse-polish-notation/) — stack-based reduce.
5. [LeetCode 394 — Decode String](https://leetcode.com/problems/decode-string/) — stack-style fold.
6. [LeetCode 43 — Multiply Strings](https://leetcode.com/problems/multiply-strings/) — reduce digit-walk.

**Hard**
7. [LeetCode 224 — Basic Calculator](https://leetcode.com/problems/basic-calculator/) — stack + operators = manual reduce.

---

🧭 Next: [zip.md](./zip.md) — pair iterables element by element. Then [enumerate.md](./enumerate.md).

Related loops: [../03_Control_Flow/for_loop.md](../03_Control_Flow/for_loop.md) · Lists: [../02_Data_Types/list.md](../02_Data_Types/list.md)
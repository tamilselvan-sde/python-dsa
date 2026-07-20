# Lambda Functions in Python

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
> Section: 04 — Functions
> 🔗 Related: [functions.md](./functions.md) · [map.md](./map.md) · [filter.md](./filter.md) · [reduce.md](./reduce.md) · [sorted key](https://docs.python.org/3/library/functions.html#sorted) · Back to [README](../README.md)

---

## 1. What is it?

A **lambda** is a **small anonymous (unnamed) function** defined in a single expression. It is created with the `lambda` keyword and returns **the value of that expression** — no `return` keyword needed.

```python
double = lambda x: x * 2
double(5)                       # 10
```

It is identical to:

```python
def double(x):
    return x * 2
```

…but written inline as one expression, with no `def`, no name (unless you bind it), and no statements — only an expression.

Lambda is **first-class**: it can be passed as an argument, stored in a variable, returned from a function. That's why it shines with `map`, `filter`, `sorted`, `max`/`min`'s `key=` parameter.

## 2. Why do we use it?

- **Tiny throwaway functions** — no need to pollute your namespace with a `def`.
- **Inline callbacks** — `sorted(xs, key=lambda x: x.age)`, `map(lambda x: x*2, xs)`.
- **No statement logic** — pure expression-only behavior is the perfect fit.
- **Function as value** — pass behavior into higher-order functions without ceremony.

## 3. When should I choose it?

| Situation | Use `lambda`? | Alternative |
|-----------|----------------|-------------|
| One-line expression callback for `sorted`/`map`/`filter`/`reduce` | ✅ Yes | named `def` |
| tiny `key=` for `max`/`min` | ✅ Yes | operator module e.g. `itemgetter(1)` |
| Logic needs multiple statements (`if/else` blocks, loops) | ❌ No | named `def` |
| Need docstring, name, or recursion | ❌ No | named `def` |
| Same lambda used 2+ times | ❌ No | name it with `def` |
| Need to debug with breakpoint inside | ❌ No | named `def` |
| Default mutable arg needed | ❌ No | named `def` with `None` sentinel |

## 4. Syntax

```
lambda parameters: expression
└──┬──┘ └────┬───┘ └────┬────┘
  │         │           │
 keyword    inputs    single expression
                      (this is the return value)
```

- **No `return` keyword** — the expression *is* the return value.
- **No statements** — no assignments, no `for`, no `while`, no `try`.
- **No annotations** — `lambda x: int: x` is a syntax error.
- Multiple parameters: `lambda x, y: x + y`
- Default args: `lambda x, y=10: x + y`
- `*args` / `**kwargs`: `lambda *a, **k: (a, k)`
- Conditional expression (ternary) is fine: `lambda x: "even" if x % 2 == 0 else "odd"`

### Anatomy

```
                lambda x, y=2: x ** y
            ┌──────┬──────┬─────────┐
            │      │      │         │
         keyword  params  colon   single expression
                          │     (the return value)
                          └─ required
```

## 5. Basic Example

```python
square = lambda x: x ** 2
square(6)                       # 36

add = lambda a, b: a + b
add(2, 3)                       # 5

with_default = lambda x, base=10: x + base
with_default(5)                 # 15
with_default(5, base=100)       # 105

classify = lambda n: "positive" if n > 0 else ("zero" if n == 0 else "negative")
classify(-3)                    # "negative"
```

### Most common real use — as a `key=` callback

```python
people = [("Tamil", 30), ("Ana", 25), ("Bob", 35)]

# sort by age
sorted(people, key=lambda p: p[1])
# [('Ana', 25), ('Tamil', 30), ('Bob', 35)]

# max by name length
max(people, key=lambda p: len(p[0]))
# ('Tamil', 30)
```

## 6. Step-by-Step Dry Run

```python
nums = [1, 2, 3, 4]
doubled = list(map(lambda x: x * 2, nums))
```

| Step | Iteration | `x` | `x * 2` | Result list |
|------|-----------|------|---------|--------------|
| 1 | first | 1 | 2 | [2] |
| 2 | second | 2 | 4 | [2, 4] |
| 3 | third | 3 | 6 | [2, 4, 6] |
| 4 | fourth | 4 | 8 | [2, 4, 6, 8] |

Final `doubled = [2, 4, 6, 8]`.

## 7. Built-in Methods / Idioms

### 7.1 Lambda with `map`

```python
list(map(lambda x: x.upper(), ["hi", "yo"]))  # ['HI', 'YO']
```
See [map.md](./map.md) for full coverage.

### 7.2 Lambda with `filter`

```python
list(filter(lambda x: x % 2 == 0, range(10)))  # [0, 2, 4, 6, 8]
```
See [filter.md](./filter.md).

### 7.3 Lambda with `reduce`

```python
from functools import reduce
reduce(lambda a, b: a + b, [1, 2, 3, 4])  # 10
```
See [reduce.md](./reduce.md).

### 7.4 Lambda with `sorted` / `min` / `max` — `key=`

```python
words = ["apple", "pie", "banana"]
sorted(words, key=lambda w: len(w))   # ['pie', 'apple', 'banana']
min(words,  key=lambda w: len(w))    # 'pie'
max(words,  key=lambda w: len(w))    # 'banana'
```

### 7.5 Lambda with multiple args & defaults

```python
greet = lambda name, greeting="Hi": f"{greeting}, {name}!"
greet("Tamil")                  # 'Hi, Tamil!'
greet("Ana", greeting="Hey")    # 'Hey, Ana!'
```

### 7.6 Lambda capturing variables (classic closure pitfall)

```python
funcs = []
for i in range(3):
    funcs.append(lambda: i)              # all capture the SAME i

[f() for f in funcs]                     # [2, 2, 2]  ❌ not [0, 1, 2]
```

Fix — bind via default arg:

```python
funcs = [lambda i=i: i for i in range(3)]
[f() for f in funcs]                     # [0, 1, 2]  ✅
```

### 7.7 Lambda with `*args` / `**kwargs`

```python
f = lambda *a, **k: (a, k)
f(1, 2, x=10)                   # ((1, 2), {'x': 10})
```

### 7.8 Immediately-Invoked Lambda (IIFE)

```python
result = (lambda x: x * 10)(5)          # 50
```

Rare in Python (comprehensions are usually better) but handy for one-off scopes.

### 7.9 `operator` module replacing simple lambdas

```python
from operator import itemgetter, attrgetter
sorted(people, key=itemgetter(1))                 # same as lambda p: p[1]
sorted(objs,  key=attrgetter("age"))
```

Faster and clearer than a lambda for trivial accessors.

### 7.10 Comparison to `def`

| Trait | `lambda` | `def` |
|-------|----------|-------|
| anonymous | yes | no |
| statements allowed | no | yes |
| multiple expressions | no | yes |
| docstring | no | yes |
| default args | yes | yes |
| `*args`/`**kwargs` | yes | yes |
| recursion | impractical (no name) | yes |
| debugging | poor | good |

## 8. Interview Example

**Problem (Google-style):** Given a list of intervals `intervals = [[1,3],[2,6],[8,10]]`, sort by start, then merge overlapping.

```python
def merge(intervals):
    intervals.sort(key=lambda i: i[0])       # sort by start
    merged = []
    for iv in intervals:
        if merged and iv[0] <= merged[-1][1]:
            merged[-1][1] = max(merged[-1][1], iv[1])
        else:
            merged.append(iv)
    return merged

merge([[1,3],[2,6],[8,10],[15,18]])
# [[1, 6], [8, 10], [15, 18]]
```

The lambda `key=lambda i: i[0]` is the classic use — sort by the first element of each inner list.

**Top-K by frequency:**

```python
from collections import Counter
def top_k(nums, k):
    freq = Counter(nums)
    return [x for x, _ in sorted(freq.items(), key=lambda kv: -kv[1])[:k]]

top_k([1,1,1,2,2,3], 2)          # [1, 2]
```

`lambda kv: -kv[1]` ranks by descending frequency.

## 9. When NOT to use

- **Multi-step logic** with loops, `if/elif`, side effects → use `def`.
- **Need a backtrace** — lambdas show as `<lambda>` in stack traces; harder to debug.
- **Same lambda repeated** — name it once with `def`.
- **Default mutable args** (`lambda x, lst=[]: ...`) — the same shared-list bug.
- **Complex dictionary access** — prefer `operator.itemgetter`.
- **Team standards forbid inline lambdas** for readability.

## 10. Common Mistakes

1. **Forgetting colon**: `lambda x x*2` → SyntaxError. Must be `lambda x: x*2`.
2. **Using statements**: `lambda x: return x` → SyntaxError. `return` is implicit, not allowed.
3. **Multi-expression attempt**: `lambda x: x; x*2` → only the first parses; semicolon isn't an expression combiner.
4. **Late-binding closure**: `lambda: i` in a loop captures the variable, not its value (see 7.6).
5. **Assigning to lambda name** for reuse — fine syntactically, but defeats the point; use `def`.
6. **Trying to annotate**: `lambda x: int -> x` is a `SyntaxError`.
7. **Using lambda as a one-off inside `map` you immediately `list()`** — a list comprehension is usually clearer:
   ```python
   list(map(lambda x: x*2, xs))     # ok but verbose
   [x*2 for x in xs]                # more Pythonic
   ```
8. **Mutating inside lambda**: `lambda x: lst.append(x)` returns `None`, returns the append result — usually you didn't mean it.

## 11. Memory Tricks

- **`lambda` = "leave it nameless"** — anonymous function.
- Picture the colon `:` as the **assignment arrow** of the implicit return.
- **Single expression rule**: think "one breath" — if you need to take another breath, write `def`.
- **Late-binding fix**: "bind with default" → `lambda i=i: ...` snapshots `i`.
- `lambda` looks like the Greek letter λ — the input flows in on the left, the result flows out on the right.

## 12. Interview Shortcuts

- Use `key=lambda x: x[1]` to sort by second element — instantly recognized by interviewers.
- `sorted(nums, key=lambda x: -x)` ⇒ "descending" without `reverse=True`.
- `sorted(words, key=lambda w: (-len(w), w))` ⇒ sort by length desc, then alphabetical — packs two keys into a tuple.
- For nested structures use a tuple key: `key=lambda p: (p[0], -p[1])`.
- If the callback needs `key=lambda x: x.attr` — use `operator.attrgetter`.
- Avoid lambdas in unit-tests where stack traces matter; prefer named functions.

## 13. Cheat Sheet Table

| Form | Example | Result |
|------|---------|--------|
| no args | `lambda: 5` | 5 |
| one arg | `lambda x: x + 1` | x+1 |
| several | `lambda a, b: a + b` | sum |
| default | `lambda x, y=10: x+y` | x+10 default |
| `*args` | `lambda *a: a` | tuple of inputs |
| `**kwargs` | `lambda **k: k` | dict of inputs |
| ternary | `lambda n: n%2==0 and "even" or "odd"` | classification |
| IIFE | `(lambda x: x*2)(5)` | 10 |
| with `map` | `map(lambda x: x**2, xs)` | iterator |
| with `sorted` | `sorted(xs, key=lambda x: -x)` | desc |

## 14. Time Complexity Table

The lambda itself is O(1) (a function object); the complexity depends on the **body** and how often it's called.

| Call site | Cost |
|-----------|------|
| `lambda` creation | O(1) |
| `lambda` call | O(1) overhead + body cost |
| `map(f, n_items)` | O(n · body) |
| `filter(f, n_items)` | O(n · body) |
| `sorted(xs, key=f)` | O(n log n) calls to `f`, plus O(n log n) comparisons |
| `max(xs, key=f)` | O(n) calls to `f` |
| `reduce(f, xs)` | O(n) calls to `f` |

## 15. Visual Diagram (ASCII)

### Lambda anatomy

```
         lambda  x, y=2 :  x ** y
            ▲    ▲ ▲ ▲  ▲  ▲  ▲
            │    │ │ │  │  │  └─ the return expression
            │    │ │ │  │  └──── colon (required)
            │    │ │ └──┘
            │    │ └─ default arg
            │    └─ parameters (comma-separated)
            └─ keyword `lambda`
```

### Lambda used with `sorted`

```
   people = [("Tamil", 30), ("Ana", 25), ("Bob", 35)]

   sorted(people, key=lambda p: p[1])
                     │   │   ▲
                     │   │   └─ "1" index → age
                     │   └─ each pair `p` flows in
                     └─ produces the comparison key

   Result:  [('Ana', 25), ('Tamil', 30), ('Bob', 35)]
                                       (sorted by age asc)
```

### Comparison: lambda vs def

```
 lambda x: x*2          def f(x): return x*2
 ┌────────────┐         ┌──────────────────────┐
 │ anonymous  │         │ named "f"             │
 │ 1 expr     │         │ multiple statements   │
 │ no return  │         │ explicit return       │
 │ no docstr  │         │ docstring allowed    │
 └────────────┘         └──────────────────────┘
```

## 16. Beginner Notes

> Remember:
> - `lambda` is just an inline **anonymous function** — value of its expression is returned.
> - The syntax is always `lambda params: expression` with a colon.
> - No statements, no `return`, no docstring, no multi-line body.
> - Best use: tiny `key=` callbacks for `sorted`/`min`/`max` and as `map`/`filter` functions.
> - Bind loop variables with defaults if you build lambdas inside a loop: `lambda i=i: i`.
> - If your lambda grows past ~30 chars or has nested ternaries → write a `def`.
> - `list(map(lambda, xs))` is often clearer as a **list comprehension**.

## 17. FAANG Tips

- Lambdas in **`sorted(key=...)`** are the most interview-friendly usage. Two-line lambdas with tuple keys sort multi-field queries elegantly:
  ```python
  sorted(items, key=lambda x: (x.priority, -x.timestamp))
  ```
- For **stable secondary sorts**, prefer chaining `sorted` calls (sorting by least-significant key first), avoiding complex tuple keys.
- **Never** assign a lambda and use it like a real function in production code — the stack trace shows `<lambda>`.
- For `min`/`max` with custom comparison — wrap in a tuple key, e.g. `max(xs, key=lambda x: (x.grade, -x.age))` ⇒ highest grade, then youngest.
- When a lambda is used only once and you need debuggability — wrap it as a named `def`.
- Interviewers love seeing lambdas **and** a one-line comprehension alternative — say which you'd choose and why.

## 18. Practice Problems

**Easy**
1. [LeetCode 977 — Squares of a Sorted Array](https://leetcode.com/problems/squares-of-a-sorted-array/) — `sorted(xs, key=lambda x: abs(x))`.
2. [LeetCode 27 — Remove Element](https://leetcode.com/problems/remove-element/) — `filter` + lambda approach.
3. [LeetCode 1636 — Sort Array by Increasing Frequency](https://leetcode.com/problems/sort-array-by-increasing-frequency/) — tuple key in lambda.

**Medium**
4. [LeetCode 56 — Merge Intervals](https://leetcode.com/problems/merge-intervals/) — `sort(key=lambda i: i[0])` then merge.
5. [LeetCode 215 — Kth Largest Element in an Array](https://leetcode.com/problems/kth-largest-element-in-an-array/) — `sorted(nums, key=lambda x: -x)`.
6. [LeetCode 347 — Top K Frequent Elements](https://leetcode.com/problems/top-k-frequent-elements/) — Counter items sorted by `lambda kv: -kv[1]`.

**Hard**
7. [LeetCode 354 — Russian Doll Envelopes](https://leetcode.com/problems/russian-doll-envelopes/) — `sorted(envelopes, key=lambda e: (e[0], -e[1]))` then LIS.

---

🧭 Next: [map.md](./map.md) — applying a function to every item. Then [filter.md](./filter.md), [reduce.md](./reduce.md), [zip.md](./zip.md), [enumerate.md](./enumerate.md).

Related loops: [../03_Control_Flow/for_loop.md](../03_Control_Flow/for_loop.md) · Lists: [../02_Data_Types/list.md](../02_Data_Types/list.md)
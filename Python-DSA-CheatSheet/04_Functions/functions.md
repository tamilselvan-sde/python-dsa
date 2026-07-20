# Functions in Python

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com
> Section: 04 — Functions
> 🔗 Related: [lambda.md](./lambda.md) · [map.md](./map.md) · [filter.md](./filter.md) · [reduce.md](./reduce.md) · [zip.md](./zip.md) · [enumerate.md](./enumerate.md) · Back to [README](../README.md)

---

## 1. What is it?

A **function** is a reusable, named block of code that takes **inputs (arguments)**, performs some computation, and **returns an output**. Functions let you **package logic** so you can call it many times without repeating yourself (the DRY principle: Don't Repeat Yourself).

In Python you define a function with the `def` keyword:

```python
def greet(name):           # def + name + (parameters):
    return f"Hi, {name}!"  # body, indented; return sends a value back

print(greet("Tamil"))      # "Hi, Tamil!"  → calling the function
```

Key parts:

| Part | Example | Purpose |
|------|----------|---------|
| `def` | `def greet` | keyword that starts a function definition |
| name | `greet` | identifier you call later |
| parameters | `(name)` | placeholders for inputs |
| body | indented block | what the function does |
| `return` | `return f"Hi, {name}!"` | sends a value back to caller |
| call | `greet("Tamil")` | actually runs the function |

Python functions are **first-class objects**: they can be assigned to variables, stored in lists, passed as arguments, and returned from other functions. This is what makes `map`, `filter`, `sorted(key=...)`, decorators, and callbacks possible.

## 2. Why do we use it?

- **Reuse** — write once, call many times (no copy-paste).
- **Abstraction** — hide implementation; caller only cares about inputs/outputs.
- **Testability** — test one function in isolation.
- **Readability** — `sum_list(xs)` reads better than a 20-line loop inline.
- **Modularity** — split a big problem into small named steps.
- **Recursion** — many algorithms (DFS, tree traversal, divide & conquer) are naturally recursive.
- **Callbacks** — pass behavior into other code (`sorted(key=len)`, `map(fn, xs)`).

## 3. When should I choose it?

| Situation | Use function? | Alternative |
|-----------|----------------|-------------|
| Logic used 2+ times | ✅ Yes | inline each time (bad) |
| Need a reusable helper | ✅ Yes | nested loop (bad) |
| One-line throwaway transform | ⚠️ maybe `lambda` | see [lambda.md](./lambda.md) |
| Need to keep state across calls | ⚠️ a class or closure | generator (yields) |
| Need to transform each item of an iterable | `map` (this file's friends) | list comprehension |
| Need to transform a list to a list | list comprehension | `map` — see [map.md](./map.md) |
| Need to fold a list to one value | `reduce` | plain `for` loop |
| Need index + value while looping | `enumerate` | `range(len(xs))` |
| Need to pair two lists | `zip` | manual index |
| Behavior changes via small "strategy" | `lambda` or `def` | subclassing (overkill) |

## 4. Syntax

### 4.1 Basic definition & call

```python
def function_name(parameters):     # header, ends with colon
    """Optional docstring."""       # triple-quoted
    body                            # indented block
    return value                    # optional; default returns None
```

### 4.2 Parameters — full menu (Python 3.8+)

Order of parameters in a function signature:

```
def f(positional_only, /, normal, *, keyword_only): ...
```

| Position | Syntax | Meaning |
|----------|--------|---------|
| 1 | `x, /` (before `/`) | **positional-only** — must pass by position, not by name |
| 2 | `y` (between `/` and `*`) | normal — positional or keyword |
| 3 | `*, z` (after `*`) | **keyword-only** — must pass by name |

```python
def f(a, /, b, *, c):       # a positional-only, b normal, c keyword-only
    return a + b + c

f(1, 2, c=3)               # OK
f(1, b=2, c=3)             # OK
f(a=1, b=2, c=3)           # ❌ TypeError: a is positional-only
f(1, 2, 3)                 # ❌ TypeError: c is keyword-only
```

### 4.3 Default arguments

```python
def power(base, exp=2):     # exp defaults to 2 if not given
    return base ** exp

power(5)                    # 25
power(5, 3)                 # 125
```

⚠️ Default values are evaluated **once** at definition time — never use mutable defaults (`[]`, `{}`). Use `None` sentinel instead:

```python
def add_item(x, lst=None):  # ✅ safe
    if lst is None:
        lst = []
    lst.append(x)
    return lst
```

### 4.4 `*args` and `**kwargs`

```python
def f(*args, **kwargs):    # args: tuple of positional, kwargs: dict of keyword
    print(args)             # (1, 2, 3)
    print(kwargs)          # {'a': 1, 'b': 2}

f(1, 2, 3, a=1, b=2)
```

- `*args` — collects extra positional args into a **tuple**.
- `**kwargs` — collects extra keyword args into a **dict**.
- Use them when the function must accept **arbitrary** arguments (wrappers, decorators, `print`-like helpers).

### 4.5 `return`

- A function without `return` returns `None`.
- `return` alone (no value) returns `None`.
- `return a, b` returns a **tuple** `(a, b)` — tuple packing.
- After `return`, the function exits — code below is unreachable.

```python
def min_max(xs):
    return min(xs), max(xs)     # returns tuple
lo, hi = min_max([3, 1, 2])     # tuple unpacking
```

### 4.6 Scope — LEGB rule

Python looks up a name in this order:

```
L → Local      (inside current function)
E → Enclosing  (outer function, for nested funcs)
G → Global      (module-level)
B → Built-in   (print, len, range, ...)
```

```python
x = 10                    # G: global
def outer():
    x = 20                # E: enclosing
    def inner():
        x = 30            # L: local
        print(x)
    inner()
outer()   # prints 30
```

To **modify** an outer variable from inside a function:

```python
counter = 0
def inc():
    global counter       # declares counter refers to the global
    counter += 1
```

For **nested** functions modify the enclosing (not global):

```python
def make_counter():
    n = 0
    def inc():
        nonlocal n       # refers to enclosing n, not global
        n += 1
        return n
    return inc
```

### 4.7 Nested functions, closures

A **closure** is a function that "remembers" variables from the scope where it was defined — even after that outer scope is gone.

```python
def make_multiplier(factor):
    def multiply(x):       # inner function
        return x * factor  # factor is captured from outer scope
    return multiply

double = make_multiplier(2)
double(5)                  # 10  → factor=2 still remembered
```

### 4.8 Lambda — anonymous one-line function

```python
double = lambda x: x * 2
```

Used inline with `map`, `filter`, `sorted`. Full coverage in [lambda.md](./lambda.md).

### 4.9 Decorators (basic)

A **decorator** is a function that takes another function and returns a new function — usually to *wrap* it with extra behavior.

```python
def shout(func):
    def wrapper(name):
        return func(name).upper() + "!!!"
    return wrapper

@shout
def greet(name):
    return f"hi {name}"

greet("tamil")   # "HI TAMIL!!!"
```

`@shout` above `def greet` is just sugar for `greet = shout(greet)`. See Section 7.8 for a timing decorator.

### 4.10 Generators (brief — `yield`)

A **generator** function returns a lazy iterator using `yield` instead of `return`.

```python
def count_up(n):
    i = 0
    while i < n:
        yield i       # pauses and returns one value at a time
        i += 1

for x in count_up(3):
    print(x)          # 0 1 2
```

Great for huge or infinite sequences — they produce one value at a time, so memory stays O(1).

## 5. Basic Example

```python
def is_prime(n):
    """Return True if n is prime, else False."""
    if n < 2:
        return False
    for i in range(2, int(n ** 0.5) + 1):
        if n % i == 0:
            return False
    return True

print(is_prime(7))    # True
print(is_prime(10))   # False
```

### With `*args`, `**kwargs`

```python
def log(*args, **kwargs):
    print("args:", args, "kwargs:", kwargs)

log(1, 2, 3, user="tamil", level="info")
# args: (1, 2, 3)  kwargs: {'user': 'tamil', 'level': 'info'}
```

## 6. Step-by-Step Dry Run

Dry run of `is_prime(15)`:

| Step | `n` | `i` | `n % i == 0`? | Action |
|------|-----|-----|-----------------|--------|
| call `is_prime(15)` | 15 | — | — | enter function |
| `n < 2`? | 15 | — | — | false, continue |
| `i = 2` | 15 | 2 | `15 % 2 = 1` → no | continue |
| `i = 3` | 15 | 3 | `15 % 3 = 0` → yes | return False |

Caller sees `False`. Note that `range(2, int(15 ** 0.5) + 1)` = `range(2, 4)` so we never even check 4, 5, …

## 7. Built-in Methods / Idioms

### 7.1 Positional-only with `/`

```python
def len_(obj, /):           # caller cannot use len_(obj=xs)
    return obj.__len__() if hasattr(obj, "__len__") else None
```

### 7.2 Keyword-only with `*`

```python
def connect(host, *, port=80, ssl=False):   # port, ssl must be named
    ...

connect("x", port=443)     # OK
connect("x", 443)          # ❌ TypeError
```

### 7.3 Default args with `None` sentinel

```python
def f(x, items=None):
    items = items or []      # creates new list if falsy
    items.append(x)
    return items
```

### 7.4 `*args` for variadic

```python
def sum_all(*nums):          # nums is a tuple
    return sum(nums)
sum_all(1, 2, 3, 4)          # 10
```

### 7.5 `**kwargs` for flexible config

```python
def build(**opts):
    return opts
build(count=3, debug=True)  # {'count': 3, 'debug': True}
```

### 7.6 Argument unpacking — `*` and `**` at call site

```python
def f(a, b, c):
    return a + b + c

nums = [1, 2, 3]
f(*nums)              # 6  → unpacks list into positional args

cfg = {"a": 1, "b": 2, "c": 3}
f(**cfg)              # 6  → unpacks dict into keyword args
```

### 7.7 Closures — counter

```python
def make_counter():
    n = 0
    def inc():
        nonlocal n
        n += 1
        return n
    return inc

c = make_counter()
c(); c(); c()        # 1, 2, 3
```

### 7.8 Basic decorator — timing

```python
import time

def timer(func):
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        print(f"{func.__name__} took {time.perf_counter() - start:.4f}s")
        return result
    return wrapper

@timer
def slow():
    time.sleep(0.5)
    return "done"

slow()                       # prints elapsed time, returns "done"
```

### 7.9 Generator with `yield`

```python
def fib():
    a, b = 0, 1
    while True:
        yield a
        a, b = b, a + b

g = fib()
[next(g) for _ in range(6)]   # [0, 1, 1, 2, 3, 5]
```

### 7.10 Multiple return values (tuple packing)

```python
def divmod_(a, b):
    return a // b, a % b
q, r = divmod_(17, 5)         # q=3, r=2
```

### 7.11 Type hints (briefly)

```python
def add(a: int, b: int) -> int:
    return a + b
```

Type hints are **optional, not enforced** by Python, but help with IDE support and interviews.

## 8. Interview Example

**Problem (Amazon-style):** Write a function `group_anagrams(strs)` that groups strings which are anagrams of each other.

```python
from collections import defaultdict

def group_anagrams(strs):
    buckets = defaultdict(list)
    for s in strs:
        key = "".join(sorted(s))          # sorted signature
        buckets[key].append(s)
    return list(buckets.values())

group_anagrams(["eat", "tea", "tan", "ate", "nat", "bat"])
# [['eat', 'tea', 'ate'], ['tan', 'nat'], ['bat']]
```

Why this works:
- Each anagram group shares the same `sorted(s)` signature → use it as dict key.
- Using `defaultdict(list)` avoids `if key not in d: d[key] = []`.
- Time **O(N · K log K)** where N = #words, K = max word length.

**Decorator interview follow-up:** Show a memoization decorator using a closure.

```python
def memoize(func):
    cache = {}
    def wrapper(*args):
        if args not in cache:
            cache[args] = func(*args)
        return cache[args]
    return wrapper

@memoize
def fib(n):
    return n if n < 2 else fib(n - 1) + fib(n - 2)

fib(50)        # fast despite naive recursion
```

The `cache` dict lives in the enclosing scope of `wrapper` — a classic closure pattern.

## 9. When NOT to use

- **One-line transforms inside `map`/`filter`/`sorted`** — use [`lambda`](./lambda.md) instead of a 5-line `def`.
- **State-heavy** behavior with many mutating methods — make a **class**, not a function with global vars.
- **Simple data munging** better expressed as a list/dict comprehension — comprehensions are often clearer than `def` + `for`.
- **The function is called exactly once and would never be reused** — inlining keeps the logic local.
- **When the function grows past ~50 lines** — split into smaller named helpers.

## 10. Common Mistakes

1. **Mutable default arguments**:
   ```python
   def f(x, lst=[]):      # ❌ same list shared across calls
       lst.append(x); return lst
   def f(x, lst=None):    # ✅
       if lst is None: lst = []
       ...
   ```
2. **Forgetting `return`** — the function returns `None` silently.
3. **Using a global in a function without `global`** — Python treats it as a new local; you get `UnboundLocalError` when reading before writing.
4. **Confusing `*args` with a list** — `args` is a **tuple**, immutable.
5. **Returning early in a generator** — `return value` in a generator stops iteration; only `yield` produces values.
6. **Decorators that don't return `wrapper`** — if the decorator returns `None`, calling the decorated function crashes.
7. **Modifying `kwargs` while iterating** — make a copy: `opts = dict(kwargs)`.
8. **Keyword-only vs positional-only confusion** — `*` makes everything after it keyword-only; `/` makes everything before it positional-only.
9. **Shadowing builtins** — `def sum(...)` overrides `sum()` for that scope; name it `sum_` instead.
10. **Using `nonlocal` on a global variable** — `nonlocal` only works on enclosing function variables, use `global` for module globals.

## 11. Memory Tricks

- **`*args` → "asterisk = args = many positionals into a tuple."**
- **`**kwargs` → "double-star = keywords = dict."**
- **LEGB → "Local, Enclosing, Global, Built-in" = "Laughs Every Game Builds."**
- **`/` only positional** — the slash *slashes out* keyword usage.
- **`*` keyword-only** — the star *forces you to name* the arg.
- **`None` sentinel** — "None means *no list yet* → create inside."
- **Decorator = wrapper** — `@deco` literally does `f = deco(f)`.
- **`yield` = pause** — function freezes until `next()` call.

## 12. Interview Shortcuts

- For **prime check**: loop only up to `int(sqrt(n)) + 1` — classic optimization.
- Use **`defaultdict(list)`** to group items without `if key not in d`.
- **`@cache` / `@lru_cache`** from `functools` for instant memoization (cleaner than a hand-rolled decorator).
- **Tuple unpacking return** — `return min_, max_` is idiomatic for two-value returns.
- **`*` to force keyword-only args** — improves the readability of API calls: `connect(host, *, port=80)`.
- **Type hints impress interviewers** even if Python ignores them at runtime.

## 13. Cheat Sheet Table

| Feature | Syntax | Returns | Notes |
|---------|--------|---------|-------|
| basic def | `def f(x): return x` | value or `None` | — |
| default arg | `def f(x=0): ...` | — | evaluated once |
| `*args` | `def f(*a): ...` | tuple | variadic positional |
| `**kwargs` | `def f(**k): ...` | dict | variadic keyword |
| positional-only | `def f(x, /): ...` | — | name not allowed |
| keyword-only | `def f(*, x): ...` | — | must name it |
| unpack call | `f(*lst)`, `f(**d)` | — | explode list/dict |
| `return a, b` | tuple packing | tuple | unpack: `x, y = f()` |
| lambda | `lambda x: x * 2` | value | one expression only |
| closure | inner uses enclosing vars | function | use `nonlocal` to mutate |
| decorator | `@deco` | wrapped func | `f = deco(f)` |
| generator | `yield` | iterator | lazy, one value at a time |
| `global` | declare inside fn | — | binds to module global |
| `nonlocal` | declare inside fn | — | binds to enclosing fn |

## 14. Time Complexity Table

| Operation | Time |
|-----------|------|
| Function call overhead | ~O(1) (small constant) |
| Default arg lookup | O(1) |
| Positional arg access `args[i]` | O(1) |
| `**kwargs` build | O(k) where k = #kwargs |
| Argument unpacking `f(*lst)` | O(len(lst)) |
| Closure variable lookup | O(1) (cell access) |
| Generator `next()` | O(body work per yield) |
| Decorator wrap cost | O(1) at decoration time; +O(1) per call wrapper |

## 15. Visual Diagram (ASCII)

### Function call flow

```
   caller                          callee
 ----------                      def f(a, b):
   args + -- ─ ─ ─ ─ ─┐             body
                      │             ...
                      ▼         return value
   result <─── value ─┘
```

### LEGB scope chain

```
 ┌─ B  Built-in  (print, len, range...)
 │
 ├─ G  Global    (module-level names)
 │
 ├─ E  Enclosing (outer function's locals)
 │
 └─ L  Local     (current function's locals)

 Lookup: L → E → G → B  (innermost first)
```

### Call → return path through a decorator

```
  greet("tamil")
      │
      ▼
  @shout wrapper(name)
      │  calls original greet(name)
      ▼
  greet("tamil")  →  "hi tamil"
      │  back in wrapper
      ▼  transform → upper + "!!!"
  "HI TAMIL!!!"
```

### *args / **kwargs gathering

```
   f(1, 2, 3, user="x", level="info")
       │      │       │
       ▼      ▼       ▼
   def f(*args, **kwargs): ...
          │            │
          ▼            ▼
       (1, 2, 3)  {'user':'x', 'level':'info'}
```

## 16. Beginner Notes

> Remember:
> - `def` *defines* a function; `()` *calls* it.
> - No `return` → function returns `None`.
> - Never use `[]` or `{}` as default args — use `None`.
> - `*args` collects extra positionals; `**kwargs` collects extra keywords.
> - Use `/` for positional-only, `*` for keyword-only.
> - LEGB is the lookup order for names: Local → Enclosing → Global → Built-in.
> - `global`/`nonlocal` only let you *modify* outer variables; reading is automatic.
> - A decorator `@deco` is exactly `f = deco(f)` — the @ is just sugar.
> - `yield` freezing-and-resuming creates a generator; `return` ends the function.

## 17. FAANG Tips

- **Favor pure functions** (no side effects) — they're easier to test, memoize, and reason about. Interviewers love when you say "this function has no side effects."
- **Name helpers clearly**: `is_prime`, `group_by_signature`, `with_cache` — verbs-or-predicates beat single letters (except in lambdas/loops).
- **Add a docstring**: a one-line `"""..."""` describing input/output helps reviewers and is interview-friendly.
- **Use type hints** for clarity: `def f(xs: list[int]) -> int:`.
- **Memoize recursive solutions**: `@functools.lru_cache(None)` turns exponential DP into linear.
- **Keyword-only args for clarity**: `range(start, end, step)` is famously ambiguous; `def f(start, end, *, step=1)` reads better.
- **Closures & decorators are interview gold** in Flask/Selenium-style questions and in writing `retry`/`timer` utilities.
- **Tuple-packed returns** (`return lo, hi`) feel natural and avoid writing a class.

## 18. Practice Problems

**Easy**
1. [LeetCode 1 — Two Sum](https://leetcode.com/problems/two-sum/) — write `two_sum(nums, target)` returning indices.
2. [LeetCode 344 — Reverse String](https://leetcode.com/problems/reverse-string/) — function with in-place reversal.
3. [LeetCode 412 — Fizz Buzz](https://leetcode.com/problems/fizz-buzz/) — function returning list of strings.

**Medium**
4. [LeetCode 49 — Group Anagrams](https://leetcode.com/problems/group-anagrams/) — illustrated above; defaultdict inside a function.
5. [LeetCode 46 — Permutations](https://leetcode.com/problems/permutations/) — recursive function with backtracking.
6. [LeetCode 322 — Coin Change](https://leetcode.com/problems/coin-change/) — recursive + memoize with `@lru_cache`.

**Hard**
7. [LeetCode 124 — Binary Tree Maximum Path Sum](https://leetcode.com/problems/binary-tree-maximum-path-sum/) — recursive helper with closure-style returning two values.

---

🧭 Next: [lambda.md](./lambda.md) — anonymous one-liner functions. Then [map.md](./map.md), [filter.md](./filter.md), [reduce.md](./reduce.md), [zip.md](./zip.md), [enumerate.md](./enumerate.md).

Related loops: [../03_Control_Flow/for_loop.md](../03_Control_Flow/for_loop.md) · Lists: [../02_Data_Types/list.md](../02_Data_Types/list.md)
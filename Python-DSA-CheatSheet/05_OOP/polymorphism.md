# Polymorphism in Python

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
> Section: 05 — OOP
> 🔗 Related: [classes.md](classes.md), [objects.md](objects.md), [inheritance.md](inheritance.md), [encapsulation.md](encapsulation.md) · Back to [README](../README.md)

## 1. What is it?

**Polymorphism** = "**many forms**". The same operation works on objects of different types
as long as they expose the right interface. Python polymorphism comes in three flavours:

| Flavour | Description |
|---------|-------------|
| **Duck typing** | "If it walks like a duck and quacks like a duck, treat it as a duck." No `implements` keyword. |
| **Method overriding** | Subclass redefines a parent method; runtime dispatches the correct version. |
| **Operator overloading** | Implement dunders (`__add__`, `__eq__`, `__len__`, ...) so operators (`+`, `==`, `len()`) work on your object. |

```python
def speak_all(animals):
    for a in animals:
        print(a.speak())        # polymorphic: works for Dog, Cat, Duck, ...
```

No interfaces required — anything with a `speak()` method is welcome.

## 2. Why do we use it?

| Benefit | Explanation |
|---------|-------------|
| **Decoupling** | Caller doesn't care about the concrete type |
| **Extensibility** | Add a new subclass without touching callers |
| **Readability** | `a + b` reads better than `a.add(b)` |
| **Built-in dispatch** | `len(x)`, `for x in y`, `x in y`, `x == y` all uniform |
| **Generic algorithms** | `sorted`/`min`/`max` work on anything orderable |
| **Protocol reuse** | Iterator, Container, Sized, Sequence — all Python "protocols" |

## 3. When should I choose it?   (decision table)

| Situation | Use |
|-----------|-----|
| Same conceptual operation, different types | Override methods (subclass) |
| Need `+`, `==`, `<`, `len()` on your class | Operator overloading via dunders |
| Different unrelated types share one method | Just rely on duck typing |
| Stable interface enforced | `abc.ABC` + `@abstractmethod` (EAFP-style interface) |
| Many variants of an algorithm | Strategy pattern — pass closures/objects with a single method |
| Need static type checking | `Protocol` from `typing` (PEP 544 — structural subtyping) |

## 4. Syntax

```python
class Vector:
    def __init__(self, x, y): self.x, self.y = x, y
    def __repr__(self):       return f"Vector({self.x}, {self.y})"
    def __eq__(self, o):      return (self.x, self.y) == (o.x, o.y)
    def __add__(self, o):     return Vector(self.x + o.x, self.y + o.y)
    def __mul__(self, k):     return Vector(self.x * k, self.y * k)
    def __rmul__(self, k):    return self.__mul__(k)         # k * v
    def __len__(self):        return int((self.x**2 + self.y**2) ** 0.5)
    def __bool__(self):       return bool(self.x or self.y)
    def __hash__(self):       return hash((self.x, self.y))

v = Vector(1, 2); w = Vector(3, 4)
print(v + w)      # Vector(4, 6)
print(2 * v)      # Vector(2, 4)  thanks to __rmul__
print(len(w))     # 5 (Pythagorean length!)
print(v == Vector(1, 2))   # True
```

## 5. Basic Example — Duck Typing

```python
class Duck:  def speak(self): return "Quack"
class Dog:   def speak(self): return "Woof"
class Car:   pass

def make_speak(thing):
    try:
        return thing.speak()        # ≈ "if it can speak, let it"
    except AttributeError:
        return f"{type(thing).__name__} cannot speak"

print(make_speak(Duck()))   # Quack
print(make_speak(Dog()))    # Woof
print(make_speak(Car()))    # Car cannot speak
```

No `Animal` superclass, no interfaces — duck typing relies solely on the presence of methods.

### Operator overloading with a `Vector` class

```
   v = (1, 2)        w = (3, 4)           v + w  =  (4, 6)
       │                 │                    ▲
       └──┐         ┌────┘                    │
          ▼         ▼                       calls v.__add__(w)
        Vector.__add__ → Vector(4, 6)
```

## 6. Step-by-Step Dry Run

`v + w` for `Vector(1, 2)` and `Vector(3, 4)`:

```
Step 1  Python sees v + w        →   looks for Vector.__add__
Step 2  v.__add__(w)             →  found? yes  → call it
Step 3  inside __add__: self=v, o=w  →  Vector(x=1+3, y=2+4) = Vector(4, 6)
Step 4  Python receives a Vector instance; this becomes the result of the expression.

Reverse order: 2 * v
Step 1  Python sees int.__mul__(2, v) → returns NotImplemented (int doesn't know Vector)
Step 2  Python then tries v.__rmul__(2) → calls Vector.__rmul__(self, 2)
Step 3  self.__mul__(2) → Vector(2, 4)
```

The `NotImplemented` → fallback → `__rmul__` chain is the **reverse operand protocol**.

## 7. Built-in Methods

### 7.1 `__init__`

| Field | Detail |
|-------|--------|
| **Purpose** | Initialise polymorphic state |
| **Example** | `Vector(1,2)` |
| **Mistakes** | Returning non-None |

### 7.2 `__repr__` / `__str__`

| Field | Detail |
|-------|--------|
| **Purpose** | Used by `print`, format strings, debugger |
| **Example** | `Vector(1, 2)` |
| **Mistakes** | Returning non-str; not using `!r` on strings |

### 7.3 `__eq__`

| Field | Detail |
|-------|--------|
| **Purpose** | `==`; required for hashing consistency |
| **Syntax** | `def __eq__(self, o): return self.x == o.x …` |
| **Interview use** | Defining it auto-sets `__hash__ = None` unless you also define `__hash__` |
| **Mistakes** | Returning `True` for cross-type; not returning `NotImplemented` for incompatible types |

### 7.4 `__hash__`

| Field | Detail |
|-------|--------|
| **Purpose** | Polymorphic for `set`/`dict` keys |
| **Mistakes** | Mutable attributes used to compute hash → object lost in set |

### 7.5 `__len__`

| Field | Detail |
|-------|--------|
| **Purpose** | `len(obj)` |
| **Interview use** | Drives truthiness if `__bool__` is missing (a 0-length object is falsy) |
| **Mistakes** | Returning negative numbers (Python doesn't disallow but it's nonsense) |

### 7.6 `__iter__` / `__next__`

| Field | Detail |
|-------|--------|
| **Purpose** | Polymorphic iteration — `for x in obj` |
| **Syntax** | `__iter__` returns an iterator; `__next__` advances it |
| **Mistakes** | Forgetting `StopIteration`; making `__iter__` a generator and `__next__` simultaneously |

### 7.7 `__getitem__` / `__setitem__`

| Field | Detail |
|-------|--------|
| **Purpose** | `obj[k]` |
| **Interview use** | Build a polymorphic sparse matrix / dict-like |
| **Mistakes** | Forgetting slice objects for `list`-like classes |

### 7.8 `__call__`

| Field | Detail |
|-------|--------|
| **Purpose** | Polymorphic "callable" protocol — `obj()` |
| **Interview use** | Strategy objects, dependency injection |

### 7.9 `__bool__`

| Field | Detail |
|-------|--------|
| **Purpose** | `if obj:` |
| **Mistakes** | Returning `None`; missing → falls back to `__len__` if present |

### 7.10 `__lt__` / `__le__` / `__gt__` / `__ge__`

| Field | Detail |
|-------|--------|
| **Purpose** | Polymorphic ordering — required by `sorted`, `heapq`, `min`/`max` |
| **Example** | `def __lt__(self, o): return (self.x, self.y) < (o.x, o.y)` |
| **Interview use** | `@functools.total_ordering` derives missing comparison methods from `__eq__` + one of `__lt__`/`__le__`/etc. |

### 7.11 `__add__` / `__sub__` / `__mul__` / `__truediv__` / `__floordiv__` / `__mod__` / `__pow__`

| Field | Detail |
|-------|--------|
| **Purpose** | Math operator overloading |
| **Example** | `def __add__(self, o): return Vector(self.x + o.x, ...)` |
| **Reverse variants** | `__radd__`, `__rmul__` etc. for `other_type + self` |
| **Inplace variants** | `__iadd__` +=, `__imul__` *= ... mutate-and-return |
| **Mistakes** | Mutating self in `__add__` (should return new); returning `None` |

### 7.12 `__enter__` / `__exit__`

| Field | Detail |
|-------|--------|
| **Purpose** | Polymorphic context-manager protocol |
| **Mistakes** | Forgetting `__exit__` returns truthy to suppress exception |

### 7.13 `__contains__`

| Field | Detail |
|-------|--------|
| **Purpose** | `x in obj` |
| **Mistakes** | Default behavior falls back to `__iter__` + comparison; define explicitly for O(1) lookup |

### 7.14 `__new__` / `__del__`

| Field | Detail |
|-------|--------|
| **Purpose** | Creation/finalisation; needed for immutable subclasses / singletons |
| **Mistakes** | `__del__` non-deterministic; over-relying on it |

### 7.15 `__getattr__` / `__getattribute__` / `__setattr__` / `__delattr__`

| Field | Detail |
|-------|--------|
| **Purpose** | Polymorphic attribute access — used to mimic, proxy or virtualise attributes |
| **Difference** | `__getattr__` is called only on **missing** attributes (cheaper); `__getattribute__` is called on **every** access (careful: infinite recursion) |
| **Mistakes** | `__setattr__` that doesn't go through `super()` → infinite recursion |

## 8. Interview Example

> **Q:** Implement a `Vector` supporting `+`, `==`, `len()`, `2 * v` (left and right multiply) and iteration. Make it work in `sorted()`.

```python
import math, functools

@functools.total_ordering
class Vector:
    __slots__ = ("x", "y")

    def __init__(self, x, y):
        self.x, self.y = x, y

    def __repr__(self):
        return f"Vector({self.x}, {self.y})"

    def __eq__(self, o):
        if not isinstance(o, Vector):
            return NotImplemented
        return (self.x, self.y) == (o.x, o.y)

    def __hash__(self):
        return hash((self.x, self.y))

    def __lt__(self, o):                  # for sorted / min / max / heap
        return (self.x, self.y) < (o.x, o.y)

    def __add__(self, o):
        if not isinstance(o, Vector):
            return NotImplemented
        return Vector(self.x + o.x, self.y + o.y)

    def __mul__(self, k):
        if not isinstance(k, (int, float)):
            return NotImplemented
        return Vector(self.x * k, self.y * k)

    __rmul__ = __mul__

    def __len__(self):
        return int(math.hypot(self.x, self.y))      # Euclidean length

    def __bool__(self):
        return bool(self.x or self.y)

    def __iter__(self):
        yield self.x
        yield self.y

v = Vector(3, 4); w = Vector(1, 1)
print(v + w)               # Vector(4, 5)
print(2 * v)               # Vector(6, 8)
print(sorted([w, v]) == [w, v])   # True
print(len(v))              # 5
print(list(v))             # [3, 4]
```

Highlights to mention: `__slots__` for memory, `total_ordering` for sorting free of charge,
`NotImplemented` to enable reverse-operand fallback, `__rmul__` for commutative multiplication.

## 9. When NOT to use

- **One-off helper** — overloading `__add__` for a side effect blows mental models; just provide a `.combine()` method.
- **Heavy operator soup** — faux DSLs become unreadable; operators should *feel* like the original meaning.
- **`__getattr__` magic everywhere** — kills static analysis, autocomplete, refactoring.
- **Methods called `__call__`** for non-callable semantics — surprising.
- **Subclass `list`** to "just add a method" — `list.sort` does not call `__lt__` on equal keys; rely on sequences.

## 10. Common Mistakes

1. **Forgetting `NotImplemented`** — return `NotImplemented`, **not** `raise NotImplemented`; the latter is wrong (it's a constant not an exception class).
2. **Not re-defining `__hash__`** when you define `__eq__` — silently makes objects unhashable.
3. **Mutating in `__add__`** — should return new; mutating ops belong to `__iadd__`.
4. **Inconsistent `__lt__` & `__eq__`** — `sort` assumes total ordering; mismatch → tricky bugs.
5. **`__getattr__` vs `__getattribute__`** confusion — the first is only called on miss; the second on every access.
6. **Recursion in `__setattr__`** — never write `self.x = v` *inside* `__setattr__`; use `super().__setattr__()` or `self.__dict__[...]`.
7. **`__rmul__` forgotten** — `2 * v` only works if you implement `__rmul__`.
8. **Comparison operators with `None`** — Python 3 raises `TypeError`; use `is` for identity checks.

## 11. Memory Tricks

- **Duck typing ↔ "Don't ask, just try."** Python's **EAFP** ("Easier to Ask Forgiveness than Permission") philosophy.
- "**Magic methods** are *implicit*." You don't call `obj.__len__()` — you call `len(obj)`.
- "**Operator dunder sandwich**": `__add__` (right-sided use), `__radd__` (reverse), `__iadd__` (in-place). Same pattern for any binary op.
- "**Defining `__eq__` → `__hash__ = None`** unless **you** `def __hash__` again" — **"EQ nukes HASH"**.
- "`@functools.total_ordering` gives 4 operators from 2 (**E**Q + **L**T)" — "**EL** total way".

## 12. Interview Shortcuts

- `len()`, `iter()`, `bool()`, `repr()`, `+`, `==`, `<` — **all** dispatch to dunders; **never** call dunder methods directly via dot.
- `NotImplemented` (a singleton value) is **different** from the `NotImplementedError` exception class.
- Whip out `@functools.total_ordering` — interviewers are impressed.
- `Protocol` from `typing` is **structural typing** = namesake-checked "duck typing for type checkers".
- Magic methods are looked up on the **type** (class) of the object, not the instance — `len(x)` calls `type(x).__len__(x)`. (Implicit dunder dispatch.)
- `__contains__` short-circuits when defined; otherwise Python falls back to iterating.
- For `for ... in` polymorphism: define `__iter__` (preferred) or `__getitem__` (legacy integer protocol).

## 13. Cheat Sheet Table

| Concept | Snippet |
|---------|---------|
| Duck typing | `obj.speak()` — no interface needed |
| Override | redef method in child |
| `+` overloading | `def __add__(self, o):` |
| `==` overloading | `def __eq__(self, o):` |
| `len()` | `def __len__(self):` |
| `next()` / `for` | `__iter__` + `__next__` |
| `obj()` callable | `def __call__(self, *a, **k):` |
| `x in obj` | `__contains__` (fallback `__iter__`) |
| Sort / heap | `__lt__` + `__eq__` (+ `total_ordering`) |
| Reverse math | `__radd__`, `__rmul__` |
| In-place math | `__iadd__`, `__imul__` |
| Context manager | `__enter__` + `__exit__` |
| Enforced interface | `abc.ABC` + `@abstractmethod` |
| Static-typed duck | `typing.Protocol` |

## 14. Time Complexity Table

| Operation | Cost | Notes |
|-----------|------|-------|
| `==` for n-field objects | O(n) | compares tuple of fields |
| `hash()` | O(fields) once per call; cached only for ints |
| `len()` | O(1) typical — depends on impl |
| `+` overloaded | O(cost of new object construction) |
| `for x in obj` | O(N) (N = number of items; default via `__iter__`) |
| `x in obj` | O(1) if `__contains__` defined; else O(N) |
| Model lookup for dunder | O(MRO depth) — **on the class**, not instance |
| `sorted(objs, key=lambda)` | O(N log N) calls |

## 15. Visual Diagram (ASCII)

```
DUCK TYPING POLYMORPHISM
    caller code:  thing.speak()
                          │
              no isinstance / no interface
                          ▼
   ┌────────────────────────────────────────┐
   │ Duck ─► Quack                          │
   │ Dog  ─► Woof                           │
   │ Cat  ─► Meow                           │
   │ Robot ─▶ AttributeError (try/except)   │
   └────────────────────────────────────────┘

OPERATOR OVERLOADING
   v + w
       │
       ▼
   Vector.__add__(v, w)
       │          │
       │   if NotImplemented?   ▼
       ▼                     w.__radd__(v)  (reverse protocol)
   returns Vector(4, 6)         │
       ▲                         │
       └── v.__add__ path  ◀──────┘
   either returns the value or both fail → TypeError

POLYMORPHIC DISPATCH (runtime)
   for a in [Dog(), Cat(), Robot()]:
        a.speak()
        └ dispatched via type(a).__mro__ → finds speak method
```

## 16. Beginner Notes

> **Remember:**
> - **Polymorphism = one interface, many shapes.** Python doesn't force you to declare interfaces — duck typing does it.
> - `obj + other` calls `obj.__add__(other)`. Numbers, strings, custom classes all share that same syntax.
> - Whenever you call `len(x)`, `bool(x)`, `==`, `<`, `+`, `*`, `for ... in`, `obj()`, `obj[i]`, `x in obj`, `with obj:` — Python is dispatching to a **magic method**.
> - Magic methods are looked up on the **class**, not the instance, so adding them to a single instance via `obj.__len__ = ...` won't work with `len(obj)`.
> - Use `@functools.total_ordering` to short-circuit the comparison boilerplate.
> - Define `__eq__` and `__hash__` **together** — otherwise your object is silently unhashable.

## 17. FAANG Tips

- **LRU Cache** and **LFU Cache** questions hinge on which dunders to implement (`__getitem__` to mimic dict access, `__contains__` for lookup).
- "Implement a Vector / Complex Number / Money class" appears often — be ready with `__add__`, `__eq__`, `__hash__`, `__repr__`, `__mul__` in your **finger memory**.
- For **`sorted`/`heapq`** questions, `@functools.total_ordering` saves time.
- Mentioning **structural typing via `typing.Protocol`** signals modern Python awareness — pair with `@runtime_checkable` if needed at runtime.
- For strategy-style problems, prefer **duck-typed callable objects** over multiple subclass branches — composition over inheritance again.
- Cross-link: [classes.md](classes.md) (where dunders are declared), [inheritance.md](inheritance.md) (method overriding), [encapsulation.md](encapsulation.md) (keep the inside safe), [../07_Algorithms/linked_list.md](../07_Algorithms/linked_list.md) (`__iter__` makes a linked list loopable).

## 18. Practice Problems

| # | Difficulty | Problem | Overload to drill |
|---|-----------|---------|-------------------|
| 1 | Easy | [933. Number of Recent Calls](https://leetcode.com/problems/number-of-recent-calls/) | Class with `__init__` + `ping` — `__len__` shortcut |
| 2 | Easy | [155. Min Stack](https://leetcode.com/problems/min-stack/) | Stack with custom `top()` — implement via list wrapper |
| 3 | Medium | [146. LRU Cache](https://leetcode.com/problems/lru-cache/) | `__getitem__` / `__setitem__`-like `get`/`put` |
| 4 | Medium | [380. Insert Delete GetRandom O(1)](https://leetcode.com/problems/insert-delete-getrandom-o1/) | Container supporting `__contains__` |
| 5 | Medium | [707. Design Linked List](https://leetcode.com/problems/design-linked-list/) | `__len__` + `__getitem__` |
| 6 | Medium | [208. Implement Trie (Prefix Tree)](https://leetcode.com/problems/implement-trie-prefix-tree/) | Composition + recursive dispatch |
| 7 | Hard | [295. Find Median from Data Stream](https://leetcode.com/problems/find-median-from-data-stream/) | Two heaps — uses operator ordering |
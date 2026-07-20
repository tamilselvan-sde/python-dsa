# Classes in Python

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com
> Section: 05 — OOP
> 🔗 Related: [objects.md](objects.md), [inheritance.md](inheritance.md), [polymorphism.md](polymorphism.md), [encapsulation.md](encapsulation.md) · Back to [README](../README.md)

## 1. What is it?

A **class** is a blueprint for creating objects. It bundles **data (attributes)** and
**behavior (methods)** into a single unit. In Python everything is an object, and every
object is created from a class (`int`, `list`, `dict`, your own types).

```python
class Dog:
    species = "Canis familiaris"        # class attribute
    def __init__(self, name):           # constructor
        self.name = name                # instance attribute
    def bark(self):                     # instance method
        return f"{self.name} says Woof!"
```

**Real-world analogy:** A class is an **architect's blueprint**; an object is the **house**
built from it. One blueprint → many houses, each with its own paint colour.

## 2. Why do we use it?

| Benefit | Explanation |
|---------|-------------|
| **Encapsulation** | Keep related state + behavior together |
| **Reusability** | One class → infinite instances |
| **Inheritance** | Share code via parent/child hierarchy |
| **Polymorphism** | Same interface, different behavior |
| **Namespace** | `obj.method()` avoids global function soup |
| **Maintainability** | Change blueprint once, every instance benefits |

## 3. When should I choose it?   (decision table)

| Scenario | Use a class? | Reason |
|----------|---------------|--------|
| Need shared state across several functions | ✅ Yes | State lives in `self` |
| Bundle data + behavior | ✅ Yes | OOP core idea |
| One-off pure function | ❌ No | Just `def` |
| Simple data container, no behavior | 🤔 Maybe | Try `dataclass` / `NamedTuple` first |
| Need class-level constants / config | ✅ Yes | Class attributes |
| Alternative constructor (`from_string`) | ✅ Yes | `@classmethod` |
| Utility function, no `self` needed | 🤔 Maybe | `@staticmethod` inside class for namespacing |

## 4. Syntax

```python
class ClassName(BaseClass, ...):          # base classes optional
    class_attr = value                     # shared by all instances

    def __init__(self, arg1, arg2):        # instance initializer
        self.arg1 = arg1                   # instance attribute
        self.arg2 = arg2

    def instance_method(self):             # takes instance
        ...

    @classmethod
    def class_method(cls, arg):            # takes the class
        return cls(arg)

    @staticmethod
    def static_method(arg):                # takes neither
        return arg * 2

    def __repr__(self):                    # official repr
        return f"ClassName({self.arg1!r})"
```

### Three method flavours at a glance

```
 ┌─────────────────┬──────────┬───────────────┬────────────────────┐
 │ Method type      │ Decorator│ First param    │ Can touch?         │
 ├─────────────────┼──────────┼───────────────┼────────────────────┤
 │ instance method  │ (none)   │ self (instance)│ self + cls attrs   │
 │ class method     │ @classmethod│ cls (class) │ cls attrs only      │
 │ static method    │ @staticmethod│ (nothing)   │ neither — pure util │
 └─────────────────┴──────────┴───────────────┴────────────────────┘
```

## 5. Basic Example

```python
class BankAccount:
    # ---- class attribute (shared) ----
    bank_name = "Python National Bank"
    _next_id = 1000                      # protected counter

    # ---- alternative constructor ----
    @classmethod
    def from_rich_list(cls, amounts):
        acc = cls(0)
        for a in amounts:
            acc.deposit(a)
        return acc

    # ---- static helper (no self/cls) ----
    @staticmethod
    def _is_positive(amount):
        return amount > 0

    # ---- initializer ----
    def __init__(self, opening=0):
        self.id = BankAccount._next_id
        BankAccount._next_id += 1
        self.balance = opening

    # ---- instance method ----
    def deposit(self, amount):
        if not self._is_positive(amount):
            raise ValueError("amount must be positive")
        self.balance += amount
        return self.balance

    def __repr__(self):
        return f"BankAccount(id={self.id}, balance={self.balance})"

a = BankAccount(500)
b = BankAccount.from_rich_list([100, 200, 50])
print(a)                   # BankAccount(id=1000, balance=500)
print(b)                   # BankAccount(id=1001, balance=350)
print(a.bank_name)         # Python National Bank      (class attr)
a.bank_name = "Local"      # shadows on instance only
print(BankAccount.bank_name)  # still Python National Bank
```

## 6. Step-by-Step Dry Run

`acc = BankAccount(500)` — what happens line by line:

```
Step 1  Python sees  BankAccount(500)  →  calls __new__  (creates empty object)
Step 2  __new__ returns a fresh instance  →  calls __init__(acc, 500)
Step 3  self = acc
Step 4  self.id = BankAccount._next_id      # 1000
Step 5  BankAccount._next_id += 1            # now 1001
Step 6  self.balance = 500
Step 7  __init__ returns None, acc is bound to the name.
```

```
   ┌─── BankAccount (class) ───┐
   │  bank_name = "Python Nat…"│  ← class attr (shared)
   │  _next_id  = 1001         │  ← class attr (shared)
   ├───────────────────────────┤
   │  __init__   deposit  …    │  ← methods
   └───────────────────────────┘
            │  instantiates
            ▼
   ┌─── acc (instance) ────────┐
   │  id       = 1000          │  ← instance attr (per object)
   │  balance  = 500           │
   │  __dict__ ───────────────┐│
   └──────────────────────────┘│
        ▲                      │
        │  lookup order         │
   instance.__dict__ → class.__dict__ → bases
```

## 7. Built-in Methods

> Magic / **dunder** ("double underscore") methods customise class behavior.

### 7.1 `__init__`  —  initialiser

| Field | Detail |
|-------|--------|
| **Purpose** | Initialise a freshly created instance |
| **Syntax** | `def __init__(self, *args, **kwargs):` |
| **Example** | `def __init__(self, x): self.x = x` |
| **Interview use** | "When is `__init__` called?" → Right after `__new__` returns the instance |
| **Mistakes** | Returning a value (must return `None`); forgetting `self` |

### 7.2 `__new__`  —  creator

| Field | Detail |
|-------|--------|
| **Purpose** | Actually *creates* the instance (rarely overridden except for singletons / immutables) |
| **Syntax** | `def __new__(cls, *args, **kwargs): return super().__new__(cls)` |
| **Interview use** | Singleton pattern, immutable subclasses (`str`, `tuple`) |
| **Mistakes** | Forgetting to call `super().__new__(cls)`; returning wrong type |

### 7.3 `__repr__`  —  official string

| Field | Detail |
|-------|--------|
| **Purpose** | Unambiguous, developer-facing representation; used by `repr()`, REPL, debugger |
| **Syntax** | `def __repr__(self): return f"Class({self.x!r})"` |
| **Interview use** | "How do you debug a list of objects?" — define `__repr__` |
| **Mistakes** | Not defining it (you get `<Class object at 0x…>`), returning non-string |

### 7.4 `__str__`  —  pretty string

| Field | Detail |
|-------|--------|
| **Purpose** | End-user friendly string; `print(obj)`, `str(obj)` |
| **Syntax** | `def __str__(self): return self.name` |
| **Interview use** | Difference from `__repr__` — `__str__` falls back to `__repr__` if missing, not vice-versa |
| **Mistakes** | Confusing the two; Python falls back to `__repr__` so always define at least that |

### 7.5 `__eq__`  —  equality

| Field | Detail |
|-------|--------|
| **Purpose** | `==` comparison between objects |
| **Syntax** | `def __eq__(self, other): return self.x == other.x` |
| **Interview use** | After defining `__eq__` you lose `__hash__` → set `__hash__ = None` for unhashable, or re-define it |
| **Mistakes** | Not handling `NotImplemented`; returning `NotImplemented` vs raising |

### 7.6 `__hash__`

| Field | Detail |
|-------|--------|
| **Purpose** | Hash for `set` / `dict` keys |
| **Syntax** | `def __hash__(self): return hash((self.x, self.y))` |
| **Interview use** | "Why can't I put my custom object in a set?" — missing/None `__hash__` |
| **Mistakes** | Mutable attributes with a defined `__hash__` (object changes while in a set) |

### 7.7 `__len__`

| Field | Detail |
|-------|--------|
| **Purpose** | `len(obj)` support |
| **Syntax** | `def __len__(self): return len(self.items)` |
| **Interview use** | Make a custom container act like a sequence |
| **Mistakes** | Returning `None` or non-int |

### 7.8 `__iter__` / `__next__`

| Field | Detail |
|-------|--------|
| **Purpose** | Make object usable in `for` loops |
| **Syntax** | `def __iter__(self): return self` then `def __next__(self): ... raise StopIteration` |
| **Interview use** | Building a custom iterator / generator-like class |
| **Mistakes** | Forgetting `StopIteration`; `__iter__` returning something non-iterable |

### 7.9 `__getitem__` / `__setitem__`

| Field | Detail |
|-------|--------|
| **Purpose** | `obj[i]` read / write |
| **Syntax** | `def __getitem__(self, key): ...` |
| **Interview use** | Implementing a sparse array, Matrix, Trie |
| **Mistakes** | Swallowing `KeyError`; not supporting slice objects |

### 7.10 `__call__`

| Field | Detail |
|-------|--------|
| **Purpose** | Make instances callable like functions (`obj(args)`) |
| **Syntax** | `def __call__(self, *args, **kwargs): ...` |
| **Interview use** | "Functor" pattern — stateful callable, e.g. cached web scraper |
| **Mistakes** | Overusing where a plain function would do |

### 7.11 `__bool__`

| Field | Detail |
|-------|--------|
| **Purpose** | Truthiness of an instance (`if obj:`) |
| **Syntax** | `def __bool__(self): return self.count > 0` |
| **Interview use** | Custom container — empty means falsy |
| **Mistakes** | Forgetting it falls back to `__len__`; returning non-bool / non-int |

### 7.12 `__lt__` / `__le__` / `__gt__` / `__ge__`

| Field | Detail |
|-------|--------|
| **Purpose** | Rich comparisons for `<`, `<=`, `>`, `>=` |
| **Syntax** | `def __lt__(self, o): return self.rank < o.rank` |
| **Interview use** | Sort custom objects, heaps; pair with `@functools.total_ordering` to derive others |
| **Mistakes** | Defining `__lt__` but `sort()` still complains — Python 3 needs explicit ordering |

### 7.13 `__add__` / `__mul__`

| Field | Detail |
|-------|--------|
| **Purpose** | Operator overloading — `+`, `*` |
| **Syntax** | `def __add__(self, o): return Vector(self.x + o.x, ...)` |
| **Interview use** | Vector class; money class; concatenation |
| **Mistakes** | Returning `None`; not handling incompatible types (return `NotImplemented`) |

### 7.14 `__enter__` / `__exit__`

| Field | Detail |
|-------|--------|
| **Purpose** | Context manager protocol — `with obj:` |
| **Syntax** | `def __enter__(self): return self` then `def __exit__(self, exc_type, exc, tb): ...` |
| **Interview use** | Implementing a connection / lock / transaction class |
| **Mistakes** | Forgetting to return `self` from `__enter__`; swallowing exceptions in `__exit__` |

### 7.15 `__del__`

| Field | Detail |
|-------|--------|
| **Purpose** | Finaliser — called when GC collects the object |
| **Syntax** | `def __del__(self): ...` |
| **Interview use** | "Why not rely on `__del__`?" → timing not guaranteed; use `with` / `try-finally` |
| **Mistakes** | Reference cycles, exceptions inside `__del__` are silently ignored |

## 8. Interview Example

> **Question:** Implement a `class` that counts how many instances of itself exist.

```python
class Tracked:
    instance_count = 0                       # class-level counter

    def __init__(self, name):
        self.name = name
        Tracked.instance_count += 1          # bump class attribute

    def __del__(self):
        Tracked.instance_count -= 1          # decrease on GC (best effort)

    def __repr__(self):
        return f"Tracked({self.name!r})"

    @classmethod
    def total(cls):
        return cls.instance_count

a = Tracked("a"); b = Tracked("b")
print(Tracked.total())   # 2
del b
print(Tracked.total())   # 1  (CPython ref-counting makes this immediate)
```

**Why it's asked:** Tests understanding of class attribute vs instance attribute, `@classmethod`, and the limitation of `__del__`.

## 9. When NOT to use

- **Pure functions / stateless transforms** — `def sort_lines(lines)` beats a class.
- **Simple value grouping** — prefer `dataclass`, `NamedTuple`, or `TypedDict`; fewer dunder headaches.
- **Tight performance loop** — function calls on `self` add attribute lookups; sometimes a `list` of tuples is faster.
- **Need immutability strictness** — plain `class` is mutable by design; subclass `tuple` or use `frozen=True` dataclass.
- **Multiple-instance **global** state** — class attributes look shared but are easy to shadow; prefer a module-level singleton or `__init__`-injected dependency.

## 10. Common Mistakes

1. **Mutating a class attribute from an instance via `=`** — silently creates an instance attribute that *shadows* the class one; other instances still see the old value.
2. **Mutable class attribute default** (e.g. `items: list = []`) — shared across all instances!  Fix: assign in `__init__`.
3. **Forgetting `self`** in method signature → `NameError: name 'self' is not defined`.
4. **Returning a value from `__init__`** → raises `TypeError`.
5. **Comparing `is` for equality of values** — `is` checks identity; use `==`.
6. **`@staticmethod` taking `self`** — that's actually an instance method, decorator is wrong.
7. **Forgetting `@classmethod` needs `cls`** — decorator without the right first param is a runtime error when called.
8. **Calling `cls(arg)` in a `@classmethod` when someone subclasses** — actually that's the *right* pattern! Many devs hard-code parent name, breaking inheritance.

## 11. Memory Tricks

- **"bluePRINT"** → PRINT = **P**roperties + **R**elations + **I**nheritance + **N**ew instances + **T**ypes.
- `@classmethod` ↔ **cls** (both start with **c**-ish letter): "Class → cls".
- `@staticmethod` ↔ **no first param**: "Static stands alone".
- `__init__` ↔ "IN it" — initialiser **IN**jects state.
- Class attribute is like the **building name plate** — same for every flat; instance attribute is each flat's **paint colour**.

## 12. Interview Shortcuts

- `__init__` is called AFTER `__new__` returns the instance.
- Defining `__eq__` silently sets `__hash__ = None` (object becomes unhashable) unless you re-set it.
- `obj.__dict__` → instance namespace; `ClassName.__dict__` → class namespace (a `mappingproxy`).
- `type(obj)` → the class; `isinstance(obj, Class)` honours inheritance, `type(obj) is Class` does not.
- Class attributes live in `ClassName.__dict__`; attribute lookup order: **instance → class → bases → `object`**.
- `@classmethod` receives the **class**, `@staticmethod` receives **nothing** implicit.
- `super()` in `__init__` ensures parent attributes are initialised.

## 13. Cheat Sheet Table

| Concept | Snippet | Notes |
|---------|---------|-------|
| Define class | `class Foo:` | PascalCase by convention |
| Initialiser | `def __init__(self, x):` | returns `None` |
| Class attr | `count = 0` | shared by all instances |
| Instance attr | `self.x = x` | per-object |
| Instance method | `def m(self):` | called as `obj.m()` |
| Class method | `@classmethod` `def m(cls):` | alternative constructors |
| Static method | `@staticmethod` `def m():` | utility, no self/cls |
| `__repr__` | dev-facing repr | define at least this |
| `__str__` | user-facing repr | falls back to `__repr__` |
| `type(obj)` | the class | strict |
| `isinstance(obj, C)` | honors inheritance | preferred |
| `hasattr / getattr / setattr` | runtime introspection | string-based |

## 14. Time Complexity Table

| Operation | Cost | Why |
|-----------|------|-----|
| Attribute read `obj.x` | O(1) amortised | dict lookup chain: inst → class → bases |
| Attribute write `obj.x = v` | O(1) | inserts into `obj.__dict__` |
| Method call `obj.m()` | O(1) + call overhead | attribute lookup + function frame |
| `isinstance` | O(depth of MRO) | walks the MRO tuple |
| Instantiation `Foo()` | O(1) + `__init__` body | `__new__` then `__init__` |
| `dir(obj)` / introspection | O(attrs) | lists all attributes |
| Adding class attribute | O(1) | `ClassName.__dict__` update |

## 15. Visual Diagram (ASCII)

```
   ┌──────────────── BLUEPRINT ─────────────────┐
   │                 class Dog                   │
   │  ─────────────────────────────────────────  │
   │   class attr   : species = "Canis fam."     │  ← shared
   │   class attr   : _registry = []             │  ← shared (mutable! danger)
   │  ─────────────────────────────────────────  │
   │   __init__(self, name)   → self.name = name │
   │   bark(self)             → returns str       │
   │   @classmethod from_puppy(cls, name) → Dog  │
   └────────────────────┬────────────────────────┘
                        │  build house from blueprint
                        ▼
   ┌─── INSTANCE: dog1 ───┐   ┌─── INSTANCE: dog2 ───┐
   │  name    = "Rex"     │   │  name    = "Luna"    │  ← own data
   │  __dict__ = {name:…} │   │  __dict__ = {name:…} │
   └──────────────────────┘   └──────────────────────┘
            │                            │
            └───── both share ───────────┴─→  Dog.species
                                             Dog._registry
```

## 16. Beginner Notes

> **Remember:**
> - A **class** is the *template*; an **object** is the *real thing* built from it.
> - `self` is just the first parameter name — it refers to **the instance itself**.
> - `__init__` is *not* a constructor in the C++ sense; Python already constructed the object with `__new__`. `__init__` just *fills it in*.
> - Class attributes are **shared** across all instances; instance attributes are stored in `obj.__dict__`.
> - `@classmethod` uses `cls` (the class) — perfect for **alternative constructors** like `from_json` or `from_string`.
> - `@staticmethod` is just a namespaced utility — no `self`, no `cls`.
> - If you only define **one** string method, make it `__repr__`. `__str__` falls back to it.
> - Mutating a class-level `list`/`dict` from one instance **leaks** to all others — assign per-instance in `__init__`.

## 17. FAANG Tips

- **LeetCode design problems** (Linked List, Trie, Min Stack, LRU Cache) are 100 % class writing. Drilling the skeleton *fast* is half the battle.
- Define `__repr__` immediately — your debugging speed in interviews depends on it.
- For **LRU Cache**, prefer `__init__` with a `dict` + `OrderedDict` pattern; the class boundary is the cache itself.
- Use `@classmethod` `from_list` constructors in design questions — interviewers love the polish.
- **Frozen dataclasses** (`@dataclass(frozen=True)` auto-implement `__hash__`/`__eq__` for free; just learn the import.
- For **Trie / Linked List** problems, write the `__init__` + one method stub first, then iterate — gives the interviewer confidence you know the shape.

Cheatsheet crossover:
- See [objects.md](objects.md) for what you can *do* once you have a class.
- See [inheritance.md](inheritance.md) for sharing code via parents.
- See [polymorphism.md](polymorphism.md) to overload `+`, `==`, `len()`.
- See [encapsulation.md](encapsulation.md) for `_protected` and `__mangled`.
- Real-world class examples: [../06_Collections/counter.md](../06_Collections/counter.md) (Counter is essentially a dict subclass) and [../07_Algorithms/linked_list.md](../07_Algorithms/linked_list.md) (classic Node class).

## 18. Practice Problems

| # | Difficulty | Problem | Class concept to drill |
|---|-----------|---------|------------------------|
| 1 | Easy | [707. Design Linked List](https://leetcode.com/problems/design-linked-list/) | `__init__`, Node class, instance methods |
| 2 | Easy | [155. Min Stack](https://leetcode.com/problems/min-stack/) | Two-list state inside one class |
| 3 | Easy | [232. Implement Queue using Stacks](https://leetcode.com/problems/implement-queue-using-stacks/) | Class wrapping two stacks |
| 4 | Medium | [146. LRU Cache](https://leetcode.com/problems/lru-cache/) | `__init__`, `OrderedDict`, `@property`-ish getters |
| 5 | Medium | [208. Implement Trie (Prefix Tree)](https://leetcode.com/problems/implement-trie-prefix-tree/) | Recursive `children: dict` pattern |
| 6 | Medium | [380. Insert Delete GetRandom O(1)](https://leetcode.com/problems/insert-delete-getrandom-o1/) | Class with two structures synced |
| 7 | Hard | [460. LFU Cache](https://leetcode.com/problems/lfu-cache/) | Multi-state class + freq buckets |
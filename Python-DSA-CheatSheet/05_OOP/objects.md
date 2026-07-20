# Objects in Python

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com
> Section: 05 — OOP
> 🔗 Related: [classes.md](classes.md), [inheritance.md](inheritance.md), [polymorphism.md](polymorphism.md), [encapsulation.md](encapsulation.md) · Back to [README](../README.md)

## 1. What is it?

An **object** is a concrete instance built from a class. The act of building it is called
**instantiation**. Every object owns three things:

1. **Identity** — `id(obj)`, never changes while the object lives.
2. **Type**   — `type(obj)`, the class it was created from.
3. **State**  — `obj.__dict__`, the attributes it carries.

```python
class Point:
    def __init__(self, x, y): self.x, self.y = x, y
    def __repr__(self):        return f"Point({self.x}, {self.y})"

p = Point(3, 4)            # instantiation
print(id(p), type(p), p)   # 4387123344  <class '__main__.Point'>  Point(3, 4)
```

**Analogy:** Blueprint (class) → Building (object). Same blueprint can be cloned into
many buildings on different plots (`id`). Each building has its own paint (state).

## 2. Why do we use it?

| Reason | Explanation |
|--------|-------------|
| **Hold state** | Each instance remembers its own data |
| **Identity** | Two cars with same spec still have different VINs (`id`, `is`) |
| **Equality semantics** | Decide what `==` means for your type |
| **Copy semantics** | Decide mutation safety — shallow vs deep |
| **Debuggability** | `__repr__` makes `print(list_of_objs)` readable |

## 3. When should I choose it?   (decision table)

| Situation | Choose |
|-----------|--------|
| Need a single object with state | Just instantiate the class |
| Want `obj1 == obj2` by value | Implement `__eq__` |
| Want a clone that you can mutate freely | `copy.deepcopy(obj)` |
| Sharing a reference is fine | pass the object as-is |
| Need a hashable key for `set`/`dict` | Define `__hash__` (+ keep state immutable) |
| Need user-friendly print | `__str__`; for logs/repl → `__repr__` |
| Nested mutable graph (list of dicts of lists) | `deepcopy` not `copy.copy` |

## 4. Syntax

```python
# 1. Instantiate
obj = MyClass(arg1, arg2)             # __new__ + __init__

# 2. Modify state
obj.x = 10                              # add / update attribute
del obj.x                               # remove attribute
obj.__dict__['y'] = 5                   # rarely used bypass — works

# 3. Inspect identity / type
id(obj)                                 # memory address (CPython: pointer)
type(obj) is MyClass                    # strict class match
isinstance(obj, MyClass)                # honors inheritance

# 4. Identity vs equality
a is b          # same object in memory?
a == b          # same value? (calls __eq__)

# 5. Copy
import copy
shallow = copy.copy(obj)
deep    = copy.deepcopy(obj)

# 6. Repr / str
repr(obj)       # dev repr
str(obj)        # user facing (falls back to __repr__)
```

## 5. Basic Example

```python
class Person:
    species = "human"                    # class attr (shared)

    def __init__(self, name, age, hobbies=None):
        self.name = name
        self.age = age
        self.hobbies = hobbies or []     # NEVER use [] default directly

    def __repr__(self):
        return f"Person(name={self.name!r}, age={self.age})"

    def __str__(self):
        return f"{self.name} ({self.age} y/o)"

    def __eq__(self, other):
        if not isinstance(other, Person):
            return NotImplemented
        return (self.name, self.age) == (other.name, other.age)

    def __hash__(self):
        return hash((self.name, self.age))

p1 = Person("Alice", 30, ["dance"])
p2 = Person("Alice", 30)                 # different list, but eq
p3 = p1                                   # same reference

print(p1 == p2)        # True   (value equality via __eq__)
print(p1 is p2)        # False  (different objects)
print(p1 is p3)        # True   (same object)
print({p1, p2})        # {Person(name='Alice', age=30)}  (deduped via __hash__)
```

## 6. Step-by-Step Dry Run

`p1 = Person("Alice", 30, ["dance"])` → `p2 = Person("Alice", 30)` → `print(p1 == p2, p1 is p2)`:

```
Step 1  Person("Alice", 30, ["dance"])  →  __new__ → __init__ → object #4387123344
Step 2  self.name = "Alice"; self.age = 30; self.hobbies = ["dance"]
Step 3  p1 ← → object #4387123344

Step 4  Person("Alice", 30) → object #4387123968  (no hobbies → defaults [])
Step 5  p2 ← → object #4387123968

Step 6  p1 == p2  →  Person.__eq__(p1, p2)
Step 7  isinstance(p2, Person)?  yes
Step 8  (("Alice",30)==("Alice",30)) → True
Step 9  p1 is p2  →  id(p1)==id(p2)  →  4387123344 != 4387123968 → False

   [p1] ──► #4387123344  name="Alice" age=30 hobbies=["dance"]
   [p2] ──► #4387123968  name="Alice" age=30 hobbies=[]
              both hash((name,age)) → same bucket in a set
```

## 7. Built-in Methods

### 7.1 `__repr__`

| Field | Detail |
|-------|--------|
| **Purpose** | Official/dev string |
| **Syntax** | `def __repr__(self): return f"Person(name={self.name!r})"` |
| **Example** | `repr(p1)` → `Person(name='Alice', age=30)` |
| **Interview use** | "Two rules": (1) _(ideally)_ valid Python that recreates the object, (2) always returns `str` |
| **Mistakes** | Returning non-str; omitting `!r` for strings (lose quotes) |

### 7.2 `__str__`

| Field | Detail |
|-------|--------|
| **Purpose** | User-facing string |
| **Syntax** | `def __str__(self): return self.name` |
| **Example** | `print(p1)` → `Alice (30 y/o)` |
| **Interview use** | Fallback chain: `__str__` → `__repr__` → `<Person at 0x…>` |
| **Mistakes** | Forgetting fallback; printing a `list` of objects calls `__repr__` not `__str__`! |

### 7.3 `__eq__`

| Field | Detail |
|-------|--------|
| **Purpose** | `==` value equality |
| **Syntax** | `def __eq__(self, o): return ...` |
| **Example** | `p1 == p2` → `True` |
| **Interview use** | The **gotcha**: defining `__eq__` automatically sets `__hash__ = None` — your object becomes unhashable unless you also define `__hash__` |
| **Mistakes** | Not returning `NotImplemented` on type mismatch; `return True` when `o is self` is missing for speed |

### 7.4 `__hash__`

| Field | Detail |
|-------|--------|
| **Purpose** | Bucket key for `set`, `dict`, `frozenset` |
| **Syntax** | `def __hash__(self): return hash((self.name, self.age))` |
| **Example** | `{p1, p2}` → one entry |
| **Interview use** | "When is `__hash__` auto-removed?" → when you define `__eq__` without also setting `__hash__` |
| **Mistakes** | Mutable hashable objects — change one field and it's *lost* inside a set |

### 7.5 `__len__`

| Field | Detail |
|-------|--------|
| **Purpose** | `len(obj)` |
| **Syntax** | `def __len__(self): return len(self.items)` |
| **Example** | `len(stack)` |
| **Interview use** | Also drives `bool(obj)` if `__bool__` is not defined |
| **Mistakes** | Returning `True`/`False` (use `__bool__` for that) |

### 7.6 `__iter__` / `__next__`

| Field | Detail |
|-------|--------|
| **Purpose** | Iterate the object |
| **Syntax** | `def __iter__(self): return iter(self.items)` |
| **Example** | `for h in p1: print(h)` |
| **Interview use** | Build a custom iterator class |
| **Mistakes** | `__next__` must `raise StopIteration` when exhausted |

### 7.7 `__getitem__` / `__setitem__`

| Field | Detail |
|-------|--------|
| **Purpose** | `obj[k]` get / set |
| **Syntax** | `def __getitem__(self, k): return self.data[k]` |
| **Example** | `p["name"]` → `p.name` |
| **Interview use** | Sparse matrix, dict-like wrapper |
| **Mistakes** | Not handling `KeyError`; forgetting slice for `list`-like classes |

### 7.8 `__call__`

| Field | Detail |
|-------|--------|
| **Purpose** | Make instances callable |
| **Syntax** | `def __call__(self, *a, **kw):` |
| **Example** | `cached = Cache(); cached("key")` |
| **Interview use** | Stateful functor with state remembered between calls |
| **Mistakes** | Using when a plain function would do |

### 7.9 `__bool__`

| Field | Detail |
|-------|--------|
| **Purpose** | `if obj:` |
| **Syntax** | `def __bool__(self): return bool(self.items)` |
| **Example** | `if stack: stack.pop()` |
| **Interview use** | Empty container is falsy |
| **Mistakes** | Returning `0` truthy by accident — never return `None` |

### 7.10 `__lt__` / `__le__` / `__gt__` / `__ge__`

| Field | Detail |
|-------|--------|
| **Purpose** | Comparison ops |
| **Syntax** | `def __lt__(self, o): return self.age < o.age` |
| **Example** | `sorted(people)` works |
| **Interview use** | Use `@functools.total_ordering` — define `__eq__` + `__lt__`, get the rest free |
| **Mistakes** | Inconsistent ordering raises `TypeError` in Py3 sort |

### 7.11 `__add__` / `__mul__`

| Field | Detail |
|-------|--------|
| **Purpose** | `+`, `*` |
| **Syntax** | `def __add__(self, o): return Person(...)` |
| **Example** | Money class addition |
| **Interview use** | Vector / Complex / Money classes |
| **Mistakes** | Mutating self instead of returning new |

### 7.12 `__enter__` / `__exit__`

| Field | Detail |
|-------|--------|
| **Purpose** | `with obj:` |
| **Syntax** | `def __enter__(self): return self` `def __exit__(self, *exc): ...` |
| **Example** | `with DBConn() as db: db.query(...)` |
| **Interview use** | Safe resource lifecycle pattern |
| **Mistakes** | Not returning `self`; swallowing exceptions in `__exit__` |

### 7.13 `__new__`

| Field | Detail |
|-------|--------|
| **Purpose** | Create the object (rarely overridden) |
| **Syntax** | `def __new__(cls, ...): return super().__new__(cls)` |
| **Example** | Singleton class |
| **Interview use** | Singleton factories, immutable subclasses |
| **Mistakes** | Returning wrong type or `None` |

### 7.14 `__del__`

| Field | Detail |
|-------|--------|
| **Purpose** | Finaliser |
| **Syntax** | `def __del__(self): ...` |
| **Example** | Close a socket-ish thing |
| **Interview use** | Understand timing is NOT deterministic |
| **Mistakes** | Relying on `__del__` for cleanup — use `with` instead |

## 8. Interview Example

> **Q:** Implement a `Money` class with `__eq__`, `__add__`, `__repr__` and explain why `__hash__` is None after defining `__eq__`.

```python
class Money:
    __slots__ = ("amount", "currency")        # save memory, fast attrs
    def __init__(self, amount, currency="USD"):
        self.amount = amount
        self.currency = currency

    def __repr__(self):
        return f"Money({self.amount} {self.currency})"

    def __eq__(self, other):
        if not isinstance(other, Money):
            return NotImplemented
        return (self.amount, self.currency) == (other.amount, other.currency)

    def __add__(self, other):
        if not isinstance(other, Money) or other.currency != self.currency:
            return NotImplemented
        return Money(self.amount + other.amount, self.currency)

m1 = Money(100); m2 = Money(100); m3 = Money(50)
print(m1 == m2)    # True
print(m1 + m3)     # Money(150 USD)
print(hash(m1))    # TypeError! __eq__ defined → __hash__ is None
```

The fix is to either (a) make `Money` immutable + define `__hash__ = lambda self: hash((self.amount, self.currency))`,
or (b) explicitly opt out: `__hash__ = None`.

## 9. When NOT to use

- **Single value with one method** — a function is simpler.
- **Immutable record** — `NamedTuple` / `dataclass(frozen=True)` covers hashing/equality for free.
- **Performance hot path** — `__eq__`/`__add__` add method dispatch; a `tuple` of `(x, y)` is faster.
- **You want struct-like speed** — `__slots__` reduces memory and lookup time but limits flexibility; use only when you control attribute set.

## 10. Common Mistakes

1. **`a == b` not implying they share state** — two equal objects can still have different `id`; mutating one does not affect the other.
2. **`copy.copy` vs `copy.deepcopy`** — shallow copies share *nested* references; deep copies walk the graph.
3. **Defining `__eq__` and wondering why the object is unhashable** — Python sets `__hash__ = None` automatically.
4. **Print a list of objects → ugly `<Person at 0x…>`** — list `__repr__` calls `repr()` not `str()` on items; define `__repr__`.
5. **Mutable default `hobbies=[]`** — shared across instances! Use `hobbies=None` then `or []`.
6. **`is` where `==` is meant** — `is` is identity; small int caching tricks you (a is b works for `256` only).
7. **Iterating a dict mutating itself** — sets/dict depend on `__hash__`; changing fields mid-iteration breaks it.

## 11. Memory Tricks

- **`id` → "address tag"** stuck on the box. Identical twins share look but not tags.
- **`is` → "identical address"**, **`==` → "equal value"**.
- **shallow copy** = new box, same toys inside. **deep copy** = new box + new toys + new toys of toys…
- **`__repr__` ↔ R.E.P.R.** = *Real Engineer's Printout,ダRep* — developer-facing. **`__str__`** = Story for users.
- Defining `__eq__` → `__hash__ = None`: **"Equal ≠ Hashable"**.

## 12. Interview Shortcuts

- `id(a) == id(b)` ⇔ `a is b`.
- Defining `__eq__` resets `__hash__ = None` unless you redefine it.
- `__slots__` makes objects **smaller + faster** but blocks `__dict__` (and pickling).
- `copy.copy` doesn't call `__init__` — it uses `__reduce_ex__` / `__copy__` protocol.
- `copy.deepcopy` handles cycles via a `memo` dict — implement `__deepcopy__` for custom graphs.
- A list's `__repr__` of objects uses `repr()` on each — so define `__repr__` to debug.
- `print(obj)` uses `__str__`; `print([obj])` uses `__repr__`.

## 13. Cheat Sheet Table

| Topic | Tool / Method |
|-------|---------------|
| Identity | `id(obj)`, `is` |
| Strict type | `type(obj) is C` |
| Polymorphic type | `isinstance` |
| Value equality | `__eq__` → `==` |
| Hash key fitness | `__hash__` |
| Cheap shallow copy | `copy.copy` |
| Deep copy (cycle-safe) | `copy.deepcopy` |
| User string | `str()` / `__str__` |
| Dev string | `repr()` / `__repr__` |
| Inspect attrs | `obj.__dict__`, `vars()` |
| Disabled dict | `__slots__` |
| Is callable? | `callable(obj)` (checks `__call__`) |

## 14. Time Complexity Table

| Operation | Cost | Notes |
|-----------|------|-------|
| `id(obj)` | O(1) | address |
| `type(obj)` | O(1) | internal pointer |
| `isinstance` | O(MRO length) | walks the MRO tuple |
| Attribute read | O(1) amortised | dict lookup chain |
| `==` | O(impl of `__eq__`) | usually O(field count) |
| `hash(obj)` | O(field hashes) | one-time, cached only for ints |
| `copy.copy` | O(1) + dict copy | shallow |
| `copy.deepcopy` | O(total graph size) | full traversal, cycle-aware |
| `del obj` | O(1) + refcount-drop | GC kicks in on cycle |

## 15. Visual Diagram (ASCII)

```
Bucket:τείnh
   Person("Alice", 30)             Person("Alice", 30)
   ┌───────────────┐               ┌───────────────┐
   │ id: 0x4387123344 │              │ id: 0x4387123968 │
   │ name: "Alice"   │              │ name: "Alice"   │   ← state can be equal
   │ age:  30        │              │ age:  30        │
   │ hobbies: [dance] │              │ hobbies: []     │
   └─────┬───────────┘              └─────────────────┘
         │  is            == (via __eq__)
         │  False ▶       True ▲
         ▼                       │
   different addresses           equal by value

   COPY:
   shallow = copy.copy(p)        deep = copy.deepcopy(p)
   shallow.hobbies ──►  same [dance]   deep.hobbies ──► new [dance]
   shallow independent of p for attrs but shares nested.
```

## 16. Beginner Notes

> **Remember:**
> - **Instantiate** = create an object from a class.
> - `id()` is the object's address tag; `is` compares tags; `==` compares values (via `__eq__`).
> - **`copy.copy`** gives you a new outer object but the same inner references. **`copy.deepcopy`** gives you a fully detached duplicate.
> - Define **`__repr__`** before anything else so you can debug.
> - For a **hashable** object it must be immutable in the fields used for `__hash__`.
> - Be careful with **mutable class-level defaults** — they leak across instances.

## 17. FAANG Tips

- Most "Min Stack" / "LRU Cache" / "Trie" LeetCode questions are really exercises in writing clean objects with controlled `__init__`, equality and copy semantics.
- If asked to "design a class to support X in O(1)", always start by listing `__init__` state; reviewers love seeing the **state diagram first**.
- **`__slots__`** mention in design-round interviews signals memory-aware thinking.
- For `__eq__` in a subclass hierarchy, compare `type(self) == type(o)` first if you want *strict* equality (otherwise subclasses can match).
- When designing a class that goes in a `set`/`dict`, **freeze** the relevant fields or use `@dataclass(frozen=True)`.
- **LRU Cache** → `copy.copy` won't preserve order; `OrderedDict.move_to_end` is the right tool.
- Cross-link: [classes.md](classes.md) for the blueprint, [inheritance.md](inheritance.md) for `is-a` hierarchies, [../06_Collections/counter.md](../06_Collections/counter.md) (Counter subclasses dict), [../07_Algorithms/linked_list.md](../07_Algorithms/linked_list.md) (Node objects are the leaf state of a linked list).

## 18. Practice Problems

| # | Difficulty | Problem | Object concept to drill |
|---|-----------|---------|--------------------------|
| 1 | Easy | [155. Min Stack](https://leetcode.com/problems/min-stack/) | `__init__` state, methods, mutate instance |
| 2 | Easy | [707. Design Linked List](https://leetcode.com/problems/design-linked-list/) | Node objects, references, copy vs share |
| 3 | Easy | [705. Design HashSet](https://leetcode.com/problems/design-hashset/) | `__eq__` / hashing helper logic |
| 4 | Medium | [146. LRU Cache](https://leetcode.com/problems/lru-cache/) | `OrderedDict`, equality, copy semantics |
| 5 | Medium | [380. Insert Delete GetRandom O(1)](https://leetcode.com/problems/insert-delete-getrandom-o1/) | Object identity ↔ dict pointer sync |
| 6 | Medium | [138. Copy List with Random Pointer](https://leetcode.com/problems/copy-list-with-random-pointer/) | Effectively **deepcopy** of a graph |
| 7 | Hard | [460. LFU Cache](https://leetcode.com/problems/lfu-cache/) | Multi-level object relationships |
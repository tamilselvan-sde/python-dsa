# Inheritance in Python

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com
> Section: 05 — OOP
> 🔗 Related: [classes.md](classes.md), [objects.md](objects.md), [polymorphism.md](polymorphism.md), [encapsulation.md](encapsulation.md) · Back to [README](../README.md)

## 1. What is it?

**Inheritance** lets a class (**child / subclass**) reuse the data and behavior of another
class (**parent / base**). The relationship is **"is-a"**: a `Dog` *is an* `Animal`.

Python supports single inheritance, **multiple inheritance**, **multilevel**, and
**hierarchical** inheritance. Method lookup order is governed by the **MRO** (Method
Resolution Order), computed by the **C3 linearisation** algorithm.

```python
class Animal:
    def __init__(self, name):
        self.name = name
    def speak(self):
        return f"{self.name} makes a sound"

class Dog(Animal):                     # Dog is an Animal
    def speak(self):
        return f"{self.name} says Woof"

d = Dog("Rex")
print(d.speak())                        # "Rex says Woof"
print(isinstance(d, Animal))            # True  (inherits the relationship)
```

## 2. Why do we use it?

| Benefit | Explanation |
|---------|-------------|
| **Code reuse** | Don't copy-paste methods |
| **Subtyping** | Functions accepting `Animal` also accept `Dog` |
| **Polymorphism** | Override `speak` per subclass |
| **Extensibility** | Add new subclasses without touching the parent |
| **Abstract contracts** | `abc.ABC` forces subclasses to implement methods |
| **Mixins** | Bundle small reusable units of behavior (e.g. `JsonMixin`) |

## 3. When should I choose it?   (decision table)

| Situation | Pattern |
|-----------|---------|
| Two classes share most behavior, child specialises one | **Single inheritance** |
| Reusable orthogonal behavior across classes (logging, json) | **Mixin** |
| Multiple unrelated parents (rare) | **Multiple inheritance** + be careful with MRO |
| Want a stable interface, variable impl | `abc.ABC` + `@abstractmethod` |
| Code reuse only, NOT an *is-a* relation | **Composition**, NOT inheritance |
| Different impls that share interface but unrelated | Inherit from common `abc.ABC` |

**Fowler's Rule:** *Favor composition over inheritance.* Use inheritance only when the child truly **is-a** parent.

## 4. Syntax

```python
# Single inheritance
class Parent:
    def __init__(self, x): self.x = x
class Child(Parent):
    def __init__(self, x, y):
        super().__init__(x)             # call Parent.__init__
        self.y = y

# Multiple inheritance
class A: ...
class B: ...
class C(A, B): ...                      # MRO = C → A → B → object

# Mixin
class JsonMixin:
    def to_json(self):
        import json
        return json.dumps(self.__dict__)

class User(JsonMixin):
    def __init__(self, name): self.name = name

# Abstract class
from abc import ABC, abstractmethod
class Shape(ABC):
    @abstractmethod
    def area(self): ...
```

## 5. Basic Example

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    """Abstract base — cannot be instantiated."""
    def __init__(self, name):
        self.name = name
    @abstractmethod
    def area(self):                     # subclasses must override
        ...
    def describe(self):                 # shared behavior, do NOT override
        return f"{self.name} area = {self.area()}"

class Circle(Shape):
    def __init__(self, r):
        super().__init__("Circle")
        self.r = r
    def area(self):
        import math
        return math.pi * self.r ** 2

class Square(Shape):
    def __init__(self, s):
        super().__init__("Square")
        self.s = s
    def area(self):
        return self.s * self.s

shapes = [Circle(2), Square(3)]
for s in shapes:
    print(s.describe())                  # uses base class + overridden area

# Shape("Foo")  # TypeError: abstract class!
```

## 6. Step-by-Step Dry Run

`Circle(2).describe()`:

```
Step 1  Circle(2)  →  Circle.__init__(self, 2)
Step 2  super().__init__("Circle")  →  Shape.__init__  sets self.name="Circle"
Step 3  self.r = 2
Step 4  obj.describe()  →  MRO search  Circle → Shape  →  found in Shape
Step 5  describe calls self.area()  →  Python uses runtime type → Circle.area
Step 6  area()  →  π * 2 ** 2  ≈  12.566

   Shape (ABC)
       │  ─ describe(self)  (concrete)
       │  ─ area(self)       (abstract)
       ▲
       │
   ┌───┴───┐
 ┌─┴─┐   ┌─┴────┐
Circle    Square
area = πr²  area = s²
```

## 7. Built-in Methods  (inheritance-flavoured)

### 7.1 `__init__`

| Field | Detail |
|-------|--------|
| **Purpose** | Initialise attributes; `super().__init__` initialises parent fields |
| **Syntax** | `super().__init__(*args)` |
| **Example** | `class Dog(Animal): def __init__(self, name, breed): super().__init__(name); self.breed = breed` |
| **Interview use** | Cooperative multiple inheritance — in diamond always use `super().__init__(**kwargs)` not the parent name |
| **Mistakes** | Hard-coding `Animal.__init__` (breaks MRO); forgetting to forward args |

### 7.2 `__repr__` / `__str__`

| Field | Detail |
|-------|--------|
| **Purpose** | Child inherits parent's string methods unless overridden |
| **Syntax** | `def __repr__(self): return f"Dog({self.name!r})"` |
| **Interview use** | Subclass override to add fields: `f"Dog({super().__repr__()})"` |
| **Mistakes** | Forgetting to call super → losing parent's repr fields |

### 7.3 `__eq__`

| Field | Detail |
|-------|--------|
| **Purpose** | Equality often inherited; subclass may add fields |
| **Syntax** | `def __eq__(self, o): return isinstance(o, Dog) and super().__eq__(o) and self.breed == o.breed` |
| **Interview use** | Always **narrow** the comparison with `isinstance(o, MyClass)` — comparing different subclasses should usually return False |
| **Mistakes** | Using `type(o) == type(self)` so strict it fails for sibling subclasses; or so loose parent equal to child |

### 7.4 `__hash__`

| Field | Detail |
|-------|--------|
| **Purpose** | Stays consistent with `__eq__` |
| **Syntax** | `def __hash__(self): return hash((super().__hash__(), self.breed))` |
| **Interview use** | Subclassing an immutable (`str`, `int`) requires overriding `__new__` AND keeping `__hash__` consistent |
| **Mistakes** | Forgetting that defining `__eq__` in child drops `__hash__` |

### 7.5 `__len__`, `__iter__`, `__next__`

| Field | Detail |
|-------|--------|
| **Purpose** | Container classes — often inherit from `collections.abc.Sequence` / `Iterable` |
| **Syntax** | `class MyList(list): def __len__(self): return ... ` |
| **Interview use** | Subclass `list` / `dict` to add behavior cheaply |
| **Mistakes** | Subclassing built-ins when composition would be simpler |

### 7.6 `__getitem__` / `__setitem__`

| Field | Detail |
|-------|--------|
| **Purpose** | Inherited from `Mapping` ABCs |
| **Example** | `class CaseInsensitiveDict(dict): def __getitem__(self, k): return super().__getitem__(k.lower())` |
| **Mistakes** | Forgetting to normalize in both get and set |

### 7.7 `__lt__` / `__le__` / `__gt__` / `__ge__`

| Field | Detail |
|-------|--------|
| **Example** | Inherit from parent + add field: `def __lt__(self, o): return (super().__lt__(o)) or (…)` — be careful, compare on tuple keys instead |
| **Mistakes** | Inconsistent comparisons across hierarchy |

### 7.8 `__add__` / `__mul__`

| Field | Detail |
|-------|--------|
| **Example** | `class Vector2D(Vector): def __add__ ...` |
| **Mistakes** | Returning a different subclass is legal but surprising |

### 7.9 `__enter__` / `__exit__`

| Field | Detail |
|-------|--------|
| **Example** | Subclass of `Connection` that overrides only `__exit__` to retry |
| **Mistakes** | Not calling `super().__enter__()` when initialising parent resources |

### 7.10 `__new__`

| Field | Detail |
|-------|--------|
| **Purpose** | Subclassing immutable types (`str`, `tuple`) requires overriding `__new__` not `__init__` |
| **Example** | `class UpperStr(str): def __new__(cls, v): return super().__new__(cls, v.upper())` |

### 7.11 `__del__`

| Field | Detail |
|-------|--------|
| **Mistakes** | Inherited `__del__` runs even if you forgot to release parent resources; chain explicitly |

### 7.12 `__contains__`

| Field | Detail |
|-------|--------|
| **Example** | `class Range: def __contains__(self, x): return self.lo <= x <= self.hi` |

### 7.13 `__bool__`

| Field | Detail |
|-------|--------|
| **Example** | `class Stack(list): def __bool__(self): return True` overrides empty container default |

### 7.14 `__call__`

| Field | Detail |
|-------|--------|
| **Example** | Subclass override adds pre/post hooks transparently |

### 7.15 `__copy__` / `__deepcopy__`

| Field | Detail |
|-------|--------|
| **Purpose** | Custom copy support for inheritors |

## 8. Interview Example

> **Q:** Diamond inheritance — `D` inherits `B` and `C`, both inherit `A`. What's the MRO and which `greet` runs?

```python
class A:
    def greet(self): return "A"
class B(A):
    def greet(self): return "B->" + super().greet()
class C(A):
    def greet(self): return "C->" + super().greet()
class D(B, C):
    def greet(self): return "D->" + super().greet()

print(D().greet())        # D->B->C->A
print(D.__mro__)          # D, B, C, A, object
```

**Why:** MRO is `D → B → C → A → object` (C3 linearisation). `super()` in `B` doesn't mean
"parent of B" — it means "next class in *the calling instance's* MRO". So `super().greet()`
from `B` jumps to `C`, not back to `A`.

```
       A  greet → "A"
      / \
     B   C
      \ /
       D
   D MRO: D, B, C, A, object
   D.greet  → D->  super (B)  →  B->  super (C)  →  C->  super (A)  →  A
   Output:  "D->B->C->A"
```

## 9. When NOT to use

- **Not an "is-a" relationship** — `Car` should *not* inherit `Engine`; **compose** instead.
- **Multiple inheritance for code reuse, no real subtype** — use **mixins** or composition.
- **When subclass overrides half a parent** — too tight coupling; you're fighting the parent.
- **Deep inheritance trees** (>3 levels) — fragile; small changes cascade.
- **Subclassing built-ins** purely to add one method — `collections.UserDict` is usually safer than `dict` (some built-ins' methods don't call overridables — `dict.update` skips your `__setitem__`!).

## 10. Common Mistakes

1. **`super().__init__()` forgotten** → parent attributes never set → `AttributeError`.
2. **Hard-coding parent name** `Parent.__init__(self)` in cooperative multiple inheritance — breaks MRO.
3. **Diamond without cooperative `super`** → A's `__init__` called twice.
4. **`isinstance` lies for `type()`** — `type(d) is Animal` returns False even though `isinstance(d, Animal)` is True.
5. **Subclassing `dict`/`list`** to add a method — `dict.update` *won't* call your `__setitem__`; use `UserDict`.
6. **Forgetting `@abstractmethod`** — `Shape()` constructs silently if you forget the decorator.
7. **Mixin depends on attributes the consumer hasn't set** — make a mixin that just *adds methods*.
8. **Overriding `__init__`** of an `ABC` subclass without calling `super()` — abstract methods are checked at construction.

## 11. Memory Tricks

- **"S.U.P.E.R."** = "**S**elects **U**p the chain (MRO), **P**asses **E**verything **R**ight".
- `super()` ≠ parent — `super()` = **next in MRO**.
- MRO = **C**3 linearisation. **C** for "**C**ooperative, **C**hained, **C**alculated".
- Diamond tree → think **Pyramid**: one apex, two middle, one base. MRO walks left first.
- "**Is-A** vs **Has-A**": Dog **is an** Animal (inherit); Car **has an** Engine (compose).
- Mixin = "**Mix**-in the behavior, no instance alone".

## 12. Interview Shortcuts

- `ClassName.__mro__` is a **tuple** of classes in resolution order.
- `super()` with no args inside a method works because Python injects `__class__` cell + `self`.
- `abc.ABC` + `@abstractmethod` blocks instantiation.
- Mixins are conventionally suffixed `Mixin` (e.g. `JsonMixin`); they're not enforced.
- `isinstance(x, C)` honours inheritance; `type(x) is C` does not.
- Multiple inheritance fails MRO construction with `TypeError: Cannot create a consistent MRO`.
- For subclassing immutables (`str`, `tuple`, `int`), override `__new__`, not `__init__`.

## 13. Cheat Sheet Table

| Concept | Snippet |
|---------|---------|
| Single inheritance | `class Child(Parent):` |
| Multiple inheritance | `class C(A, B):` |
| `super()` call | `super().__init__(...)` |
| MRO | `C.__mro__` |
| Abstract class | `class S(ABC): @abstractmethod …` |
| Mixin | small class intended to be combined, no `__init__` of its own ideally |
| Override | redefine method with same name in child |
| Cooperative super | `super().__init__(**kwargs)` forwards everything |
| Subclass immutable | override `__new__` |
| Avoid subclassing `dict` | use `UserDict` from `collections` |

## 14. Time Complexity Table

| Operation | Cost | Notes |
|-----------|------|-------|
| Method lookup via MRO | O(depth of MRO) but cached | CPython caches `__mro__` lookup |
| `isinstance` | O(MRO length) | linear walk |
| `issubclass` | O(MRO length) |  |
| `super()` call | O(1) to construct + cached call | look-up table internally |
| Diamond inheritance init | O(levels × super calls) | cooperative chain |
| Single attribute lookup | O(1) amortised | same as regular class |

## 15. Visual Diagram (ASCII)

```
Single            Multiple            Hierarchical       Multilevel

Animal            A                   Animal              Animal
  ▲               /\                    ▲                   ▲
  │              /  \                  │                   │
 Dog            B    C                Cat                  Mammal
                                          Dog  Cat           ▲
                                                              │
                                                           Dog
                                                  (Dog is an Animal *via* Mammal)

DIAMOND (multiple + multilevel)         MRO via C3:
                                                 D, B, C, A, object
       A
      / \                                  super() resolution
     B   C                                   for instance of D
      \ /
       D                                   D.greet()
                                            → super → B
                                              → super → C      (NOT A!)
                                                → super → A
                                                  → super → object

ABC + Mixin pattern:
   Core(ABC) ← JsonMixin ← User
   Core defines interface; JsonMixin adds serialization; User combines both.
```

## 16. Beginner Notes

> **Remember:**
> - **Inheritance = "is-a"**. If you can't say "Dog **is an** Animal", don't inherit.
> - `super()` is not necessarily "the parent"; it's the **next class in MRO** of the *current* instance.
> - Always call `super().__init__(...)` when overriding `__init__` in a subclass to make sure parent state is set up.
> - Use `abc.ABC` + `@abstractmethod` when you want **declarative "must implement this"** behavior.
> - Use **mixins** for reusable orthogonal behavior (logging, json, repr); a mixin should ideally not own state.
> - Multiple inheritance is OK in Python but you **must** use cooperative `super()` and design the diamond rules carefully.
> - `isinstance` is the polymorphic way (accounts for inheritance); `type(obj) is C` is the strict way.

## 17. FAANG Tips

- For **Trie / Linked List / Stack / Cache** problems, build a small **abstract base** once, then subclass alternatives — interviewers notice this cleanliness.
- Mentioning **C3 MRO** by name signals deep understanding; demonstrate by printing `Cls.__mro__`.
- When asked "**Favor composition over inheritance**" — answer: "I prefer composition when child only reuses behavior, and inheritance when the child truly is a subtype, e.g., `TrieNode` is uniquely a `TrieNode` so I instantiate directly, but a `Stack` IS a `Container`".
- Design question shortcut: implement an `abc.ABC` interface, then write **two** concrete subclasses — proves polymorphic thinking.
- Mention `collections.UserDict` / `UserList` if asked about subclassing dict — wins senior points.
- Cross-link: [polymorphism.md](polymorphism.md) (overriding in action), [encapsulation.md](encapsulation.md) (protect inherited state), [../06_Collections/counter.md](../06_Collections/counter.md) (Counter subclasses `dict`), [../07_Algorithms/linked_list.md](../07_Algorithms/linked_list.md) (single linked list node inheritance patterns).

## 18. Practice Problems

| # | Difficulty | Problem | Inheritance concept to drill |
|---|-----------|---------|------------------------------|
| 1 | Easy | [707. Design Linked List](https://leetcode.com/problems/design-linked-list/) | Node class + main class `is-a` warm-up |
| 2 | Easy | [155. Min Stack](https://leetcode.com/problems/min-stack/) | Subclass `list`-style stack, override `push/pop` |
| 3 | Medium | [208. Implement Trie (Prefix Tree)](https://leetcode.com/problems/implement-trie-prefix-tree/) | Recursive `TrieNode` parent/child pattern |
| 4 | Medium | [1804. Implement Trie II](https://leetcode.com/problems/implement-trie-ii-prefix-tree/) | Subclass extension of trie |
| 5 | Medium | [146. LRU Cache](https://leetcode.com/problems/lru-cache/) | Inline `OrderedDict` (subclass dict) |
| 6 | Medium | [304. Range Sum Query 2D — Immutable](https://leetcode.com/problems/range-sum-query-2d-immutable/) | Class hierarchy wrapping matrices |
| 7 | Hard | [460. LFU Cache](https://leetcode.com/problems/lfu-cache/) | Multiple data structures cooperating |
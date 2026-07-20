# Encapsulation in Python

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
> Section: 05 — OOP
> 🔗 Related: [classes.md](classes.md), [objects.md](objects.md), [inheritance.md](inheritance.md), [polymorphism.md](polymorphism.md) · Back to [README](../README.md)

## 1. What is it?

**Encapsulation** bundles data and the methods that operate on that data into one class
**and** controls access to that data. In Python this is **convention-driven**, not enforced:

| Convention | Meaning |
|------------|---------|
| `name`     | **public** — anyone can access |
| `_name`    | "protected" — signal that it's internal; Python doesn't stop you |
| `__name`   | "private" — **name-mangled** to `_ClassName__name` (a soft barrier) |
| `@property` | controlled accessor with **getter** + **setter** Validated access |

```python
class Account:
    def __init__(self, owner, balance=0):
        self.owner = owner
        self._balance = balance          # "protected" by convention
        self.__id = id(self)             # name-mangled → _Account__id

    @property
    def balance(self):                   # getter
        return self._balance

    @balance.setter
    def balance(self, value):            # setter, with validation
        if value < 0:
            raise ValueError("balance cannot be negative")
        self._balance = value
```

**Analogy:** Encapsulation is the **vault door** of a bank. The class is the bank, internal
fields are money in the vault, methods are the teller — you don't reach past the teller
directly into the vault; you ask the teller to do it for you.

## 2. Why do we use it?

| Benefit | Explanation |
|---------|-------------|
| **Information hiding** | Hide implementation from callers |
| **Validation** | Setters guard invariants (e.g. age >= 0) |
| **Stability** | Change internal representation without breaking callers |
| **Naming safety** | `__mangled` prevents collisions in subclasses |
| **Invariants** | Class-level rules enforced through methods only |
| **Controlled API** | Public surface area is explicit (only `public` methods) |

## 3. When should I choose it?   (decision table)

| Situation | Convention / Tool |
|-----------|------------------|
| Public API method | plain `def name(self):` |
| Internal helper | `def _helper(self):` |
| Truly private, avoid name collision in subclasses | `__private` (name-mangling) |
| Validate on read/write | `@property` + `@x.setter` |
| Computed derived value (read-only) | `@property` only (no setter) |
| Avoid subclassing built-in internals | start everything `_protected` then expose minimally |
| Need immutable attribute that subclass overrides | Use `@property` + `super()` |

**Rule of thumb:** Start with `_protected`. Expose via `@property` when you need
validation or computed behaviour. Reserve `__double` for genuine naming conflicts in
subclass hierarchies.

## 4. Syntax

### Naming conventions

```python
class Example:
    public_attr = 1
    _protected_attr = 2
    __private_attr = 3          # silently becomes _Example__private_attr

    def __init__(self):
        self.x = 10
        self._y = 20
        self.__z = 30            # only accessed via self.__z from inside Example
                                 # from outside: obj._Example__z  (mangled name)
```

### `@property` full example

```python
class Temperature:
    def __init__(self, celsius):
        self.celsius = celsius          # goes through the setter

    @property
    def celsius(self):
        return self._celsius

    @celsius.setter
    def celsius(self, value):
        if value < -273.15:
            raise ValueError("below absolute zero")
        self._celsius = value

    @property
    def fahrenheit(self):
        return self._celsius * 1.8 + 32      # read-only computed property

    @fahrenheit.setter
    def fahrenheit(self, value):
        self.celsius = (value - 32) / 1.8
```

### Data descriptor approach (advanced)

```python
class Validated:
    def __set_name__(self, owner, name): self.name = name
    def __get__(self, obj, owner):    return getattr(obj, f"_{self.name}")
    def __set__(self, obj, value):
        if not isinstance(value, int): raise TypeError("int only")
        setattr(obj, f"_{self.name}", value)

class WithAge:
    age = Validated()                       # full descriptor; intercepts attribute set
```

## 5. Basic Example

```python
class BankAccount:
    def __init__(self, owner, initial=0):
        self.owner = owner                  # public
        self._balance = initial             # protected (convention)
        self.__pin = "1234"                 # private (name-mangled)

    @property
    def balance(self):                      # read-only public access
        return self._balance

    def deposit(self, amount):
        if amount <= 0:
            raise ValueError("deposit must be positive")
        self._balance += amount
        return self.balance

    def __str__(self):
        # Inside the class, __pin resolves normally
        return f"{self.owner}: ${self._balance} (pin: {self.__pin})"

acc = BankAccount("Alice", 1000)
print(acc.owner)         # Alice
print(acc.balance)       # 1000  (via @property)
# acc.balance = -50      # AttributeError: setter not defined → read-only
acc.deposit(500)
print(acc.balance)       # 1500
# print(acc.__pin)       # AttributeError (mangled to _BankAccount__pin)
print(acc._BankAccount__pin)   # "1234"  (name-mangling bypass — discouraged!)
print(acc._balance)      # 1500    (convention not enforced)
```

## 6. Step-by-Step Dry Run

`acc = BankAccount("Alice", 1000)` then `acc.deposit(500)`:

```
Step 1  BankAccount("Alice", 1000) → __init__ called
Step 2  self.owner = "Alice"                ← stored in self.__dict__['owner']
Step 3  self._balance = 1000                ← stored as '_balance' in __dict__
Step 4  self.__pin = "1234"                 ← mangled → stored as '_BankAccount__pin'

Step 5  acc.balance                         ← lookup:
          → no 'balance' in acc.__dict__
          → looks up class; finds @property descriptor
          → calls getter → returns self._balance (1000)

Step 6  acc.deposit(500):
          amount = 500 (>0)
          self._balance += 500 → 1500

Step 7  acc.__pin                            ← mangled to acc._acc__pin  (NOT found!)
        because the mangling happens *in the class scope*, not generic.
        Outside: acc._BankAccount__pin works but is an anti-pattern.
```

### Name-mangling visualised

```
        class BankAccount:
            __pin  ───────────►  _BankAccount__pin     (transformation)

        inside class body:  self.__pin     ──►  self._BankAccount__pin  ✓
        outside (instance): acc.__pin       ──►  AttributeError
        outside (instance): acc._BankAccount__pin   ──►  "1234"   (pryable)
        subclass (class Child(BankAccount)):
            self.__pin                      ──►  _Child__pin  (different!)
            cannot accidentally overwrite parent's pin.
```

## 7. Built-in Methods  (encapsulation-flavoured)

### 7.1 `__init__`

| Field | Detail |
|-------|--------|
| **Purpose** | Initialise **all** private/protected state; hook invariants |
| **Mistakes** | Mutating parent's `__field` from subclass (gets renamed to subclass name) |

### 7.2 `__repr__` / `__str__`

| Field | Detail |
|-------|--------|
| **Purpose** | Public representation; avoid exposing internal `__` fields verbatim |
| **Mistakes** | Leaking sensitive fields like `__pin`, `__password` in `__repr__` |

### 7.3 `__eq__`

| Field | Detail |
|-------|--------|
| **Purpose** | Equality usually compares exposed attributes; often **avoids** comparing `__id` / `__pin` |
| **Mistakes** | Comparing by sensitive private fields |

### 7.4 `__hash__`

| Field | Detail |
|-------|--------|
| **Purpose** | Hash by *immutable* attributes (often `_balance` invalidates cache) |
| **Mistakes** | Hashing mutable private fields |

### 7.5 `__getattr__` / `__getattribute__` / `__setattr__` / `__delattr__`

| Field | Detail |
|-------|--------|
| **Purpose** | The lowest-level encapsulation — intercept all attribute access |
| **`__getattr__`** | Called *only when normal lookup fails* — perfect for lazy attributes |
| **`__getattribute__`** | Called on **every** access; super easy to recurse; use `super()` carefully |
| **`__setattr__`** | Called on every `self.x = ...`; guard invariants here |
| **`__delattr__`** | Called on `del self.x` |
| **Mistakes** | Infinite recursion in `__setattr__`/`__getattribute__` — always use `super().__setattr__()` or `self.__dict__[...]` |

### 7.6 `@property`

| Field | Detail |
|-------|--------|
| **Purpose** | Convert attribute access to a method call transparently |
| **Syntax** | `@property` `def x(self): ...` `@x.setter` `def x(self, v): ...` |
| **Interview use** | Computed properties (cached lazily), validation, backward-compatible rename |
| **Mistakes** | Using `@property` everywhere as a micro-validation fetish; can hurt testability |

### 7.7 `__len__`, `__iter__`, `__next__`, `__getitem__`, `__setitem__`

| Field | Detail |
|-------|--------|
| **Purpose** | Encapsulate **container semantics** — class manages internal list, callers only see protocol |
| **Example** | `class Stack: def __len__(self): return len(self._items)` |
| **Mistakes** | Letting callers reach into `_items` directly; declare `_items` private |

### 7.8 `__enter__` / `__exit__`

| Field | Detail |
|-------|--------|
| **Purpose** | RAII-style encapsulation of a resource (file, lock, conn) |
| **Mistakes** | Forgetting to **`__exit__` properly** → leaks; not setting `__slots__` for hot resources |

### 7.9 `__new__` / `__del__`

| Field | Detail |
|-------|--------|
| **Purpose** | `__new__` for enforcing singletons/immutable invariants; `__del__` (rare) for last-resort cleanup |
| **Mistakes** | Relying on `__del__` for release semantics — use `with` |

### 7.10 `__slots__`

| Field | Detail |
|-------|--------|
| **Purpose** | Best encapsulation-by-strictness: lists fixed attribute names → blocks ad-hoc attributes |
| **Syntax** | `class C: __slots__ = ('x', '_y')` |
| **Mistakes** | Combined with `@property` requires care (slots cannot have a property of same name) |

### 7.11 `__weakref__` (companion to slots)

| Field | Detail |
|-------|--------|
| **Purpose** | Add to `__slots__` if you need weak references; otherwise slots disable them |

### 7.12 `__dict__`

| Field | Detail |
|-------|--------|
| **Purpose** | Public dict of instance attributes — **avoids** `_protected`/`__private` unless you use `__slots__` |
| **Mistakes** | Encapsulation doesn't survive if you expose `__dict__` to clients |

### 7.13 `__bool__`

| Field | Detail |
|-------|--------|
| **Example** | `class Stack: def __bool__(self): return bool(self._items)` |

### 7.14 `__call__`

| Field | Detail |
|-------|--------|
| **Example** | Encapsulate a closure-like state without leaking internals |

### 7.15 `__contains__`

| Field | Detail |
|-------|--------|
| **Example** | `x in obj` lets you encapsulate a hash table internally |

## 8. Interview Example

> **Q:** Implement a `Person` class whose `age` cannot be set to a negative number **and** whose `age` must always return the live age computed from `birth_year`.

```python
from datetime import date

class Person:
    def __init__(self, name, birth_year):
        self.name = name
        self._birth_year = birth_year

    @property
    def age(self):                                 # computed, read-only
        return date.today().year - self._birth_year

    @property
    def birth_year(self):                           # read-only public attribute
        return self._birth_year

    @birth_year.setter
    def birth_year(self, value):
        if not (1900 <= value <= date.today().year):
            raise ValueError("invalid birth_year")
        self._birth_year = value

p = Person("Alice", 1990)
print(p.age)            # age computed live (35 in 2025)
p.birth_year = 1995    # validated
# p.age = -5            # AttributeError: age is read-only
p.birth_year = 1800    # ValueError: invalid birth_year
```

Why this is a strong answer:
- The `_birth_year` is **the single source of truth** — `age` is *derived*, not stored, so it can never go stale.
- Setter enforces the invariant before mutating.
- Read-only computed property is one of Python's cleanest encapsulation idioms.

## 9. When NOT to use

- **Tiny script where behavior is trivial** — full `@property` discipline is overkill; you'll spend more time writing boilerplate than the bug would have caused.
- **Need mutable internal list and don't want defensive copies** — encap is about *controlling exposure*, not erasing access; use `_items` and document the contract.
- **Performance hot loop** — a `@property` call per access adds overhead; expose attributes directly and document mutability externally.
- **When the framework expects simple attributes** — some serializers (`attrs` / `pydantic`) work better without manual `@property` machinery.

## 10. Common Mistakes

1. **Trusting `_single_underscore` for security** — it's **convention only**, not enforced; `_balance` is freely accessible from outside.
2. **Believing `__double` is "private"** — it's only **name-mangled**, still accessible as `obj._ClassName__field`.
3. **Using `__double` everywhere** — problems with subclassing, introspection, pickling, `dataclasses`. Reserve for **genuine naming conflicts**.
4. **Setter exception leaves inconsistent state** — if your setter validates A then B then sets, and B raises, A was already half-set; set into **local variables** first, then commit.
5. **Returning internal mutable list** — `def get_items(self): return self._items` leaks the reference! Return `self._items[:]` or `list(self._items)` if you want immutability.
6. **Forgetting `@x.setter` and wondering why `obj.x = 1` raises `AttributeError`** — a `@property` without setter is read-only by design.
7. **Defining `@property` on a class but expecting subclasses to override** — properties don't naturally override; subclass must redefine the whole descriptor or use a hook method.
8. **`__slots__` + `@property` with same name** — slots reserves the storage slot, the property wants to be a descriptor; **conflict** — use a different name (`_x` slot, `x` property).
9. **Exposing `__dict__` accidentally** — without `__slots__`, users can `obj.foo = 1` arbitrarily; that defeats encapsulation by inertia.

## 11. Memory Tricks

- **`_score`** = "soft secret" (convention); **`__secret`** = "scrambled secret" (mangled to `_Class__secret`).
- "**Underscores vs Prying Eyes**": 1 underscore = "please don't"; 2 underscores = "I made it harder".
- Mangled name = `"_" + Class.__name__ + "__" + original_name`. Mnemonic: **"Class lockbox Compass"** → `_ClassName__field`.
- `@property` = method-that-looks-like-an-attribute. **"P-roperty = Procedure behind a variable"**.
- "**Private by Convention**" — that's all the privacy Python offers. Use `@property` for *control*, not `__` for *secrecy*.
- "**Start public, retreat to private**" — refactor from public → `_protected` → `@property` as invariants emerge.

## 12. Interview Shortcuts

- Python has **no true `private` keyword**. `_` and `__` are conventions / mild protection only.
- `__name` inside `class Foo` becomes `_Foo__name` — only inside `Foo`. From subclass `Bar(Foo)`, `self.__name` becomes `_Bar__name` (different slot!).
- `@property` is a **descriptor** (class-level, not instance-level) — overriding in subclasses is tricky.
- `__slots__` blocks `__dict__`, saves memory, prevents ad-hoc attributes → strong encapsulation + speed.
- A `@property` without a setter is **read-only**; subclasses must redefine the whole property to make it writable.
- Encapsulation's real test: "Can I change the internal representation without breaking callers?" If **yes**, you've encapsulated properly.
- Return **defensive copies** of internal mutable collections to keep encapsulation intact.

## 13. Cheat Sheet Table

| Convention/Mechanism | Enforced? | Use case |
|----------------------|-----------|----------|
| `public` | no | external API |
| `_protected` | convention only | internal helpers / attrs |
| `__double` | name-mangled (mild) | avoid subclass collisions |
| `__slots__` | yes (no new attrs) | memory + strict attr control |
| `@property` (getter only) | yes (read-only) | derived / invariant |
| `@x.setter` | yes (validation) | guarded mutation |
| `@x.deleter` | yes | cleanup on `del obj.x` |
| Descriptor (`__get__`/`__set__`) | yes | reusable controlled attrs |
| `__getattr__` | no (only on miss) | lazy/virtual attrs |
| `__setattr__` | yes | intercept everything |

## 14. Time Complexity Table

| Operation | Cost | Notes |
|-----------|------|-------|
| Plain attribute read/write | O(1) | dict lookup |
| `@property` read | O(1) + body cost | method dispatch |
| `_mangled_lookup` | O(1) | dict lookup after mangling |
| `__getattr__` (miss path) | O(1) + body cost | extra dispatch cost on miss |
| `__getattribute__` (every access) | O(1) + body cost | careful, every access |
| `hasattr` | looks up + tries `__getattr__` | can fail with `__getattribute__` side-effect |
| Validation in setter | O(body cost) | e.g., O(len) for "is unique" checks |

## 15. Visual Diagram (ASCII)

```
ENCAPSULATION LAYERS

   Public layer  (obj.owner, obj.balance)
   ─────────────────────────────────────
                  │
                  ▼  @property gateway
   ┌────────────────────────────────┐
   │   Class internal namespace     │
   │   _balance = 1000   (convention)│
   │   __pin    = "****" (mangled)   │
   └────────────────────────────────┘
                  │
                  ▼
   Actual storage in  self.__dict__  (or __slots__)
       owner         → "Alice"
       _balance      → 1000
       _BankAccount__pin → "1234"   ← mangling makes it harder, not impossible

NAME MANGLING
   self.__pin ──► (inside class) ──► self._Account__pin
   obj.__pin  ──► (outside)        ──► AttributeError
   obj._Account__pin ──► "1234"    (pryable but discouraged)

DEFENSIVE COPY
   class Stack:
       def __init__(self): self._items = []
       def get_items(self): return list(self._items)   # SNAPSHOT, not reference

             caller                              Stack
             items = obj.get_items()  ───────►  new list copy
             items.append(99)                    (Stack immutable in user's eyes)
```

## 16. Beginner Notes

> **Remember:**
> - In Python, **privacy is by convention**, not enforcement. `_x` asks callers nicely not to use it; `__x` makes it slightly harder.
> - `__double_underscore` triggers **name mangling**: inside `class Foo`, `self.__x` is rewritten as `self._Foo__x`. From a subclass `Bar(Foo)`, the same `self.__x` becomes `_Bar__x` — a *different* attribute. That's by design — it avoids accidentally clashing with parent internals.
> - `_single_underscore` does **not** trigger mangling — it's purely a hint.
> - `@property` converts method calls into attribute access: `obj.balance` runs the getter transparently.
> - A `@property` **without** a `@x.setter` is read-only.
> - Avoid returning **internal mutable collections** directly — return a copy (`list(self._items)`).
> - Use `__slots__` when you want **strict attribute control + memory savings**.
> - Start by making everything **public**, then **retreat** to `_protected` and `__private` / `@property` as you discover invariants. Refactoring toward privacy is easier than bolting it on too early.

## 17. FAANG Tips

- In design questions, **state** the encapsulation policy: "I keep the inner `Node` class private, the outer `LinkedList` exposes only `add/get/delete`".
- `@property` is the **idiomatic** way to evolve an attribute into a computed value — interviewers love seeing "I might add an `expiration` property later without changing the public API".
- For **LeetCode LRU Cache**, encapsulate `OrderedDict` inside a class and only expose `get` / `put`; treat `_node` as private.
- **`__slots__` mention is senior signal**: "Because these objects are created millions of times in benchmarks, I'd add `__slots__`".
- For "design a URL shortener / twitter" system-design round, explicitly call out **private fields** + **public API** so reviewers don't drill on accidental coupling.
- Returning **defensive copies** (or using `tuple` instead of `list`) prevents bugs where an external mutator breaks invariants — explicitly mention it.
- Cross-link: [classes.md](classes.md) for the full class setup, [objects.md](objects.md) (objects often expose internals through `__dict__` / `__repr__`), [inheritance.md](inheritance.md) (inherited `__init__` must respect parent invariants), [polymorphism.md](polymorphism.md) (use `@property`-driven protocols for clean polymorphic APIs).
- Try `Counter` from [../06_Collections/counter.md](../06_Collections/counter.md): the public name is `Counter`, but `__missing__` is the protected hook — **textbook encapsulation of a specialisation rule**. Node internals of [../07_Algorithms/linked_list.md](../07_Algorithms/linked_list.md) wrap `_next` pointers — only the Linked List class exposes `head` mutating operations.

## 18. Practice Problems

| # | Difficulty | Problem | Encapsulation idea to drill |
|---|-----------|---------|------------------------------|
| 1 | Easy | [155. Min Stack](https://leetcode.com/problems/min-stack/) | `_min_stack` protected attribute |
| 2 | Easy | [705. Design HashSet](https://leetcode.com/problems/design-hashset/) | Internal buckets private |
| 3 | Easy | [706. Design HashMap](https://leetcode.com/problems/design-hashmap/) | `@property` for `size`; `_buckets` list |
| 4 | Medium | [146. LRU Cache](https://leetcode.com/problems/lru-cache/) | Encapsulate `OrderedDict`; only `get`/`put` public |
| 5 | Medium | [208. Implement Trie (Prefix Tree)](https://leetcode.com/problems/implement-trie-prefix-tree/) | `_children` dict private |
| 6 | Medium | [380. Insert Delete GetRandom O(1)](https://leetcode.com/problems/insert-delete-getrandom-o1/) | `_pos` map + `_items` list, kept in sync |
| 7 | Hard | [460. LFU Cache](https://leetcode.com/problems/lfu-cache/) | Multiple `_freq_buckets` dict private |
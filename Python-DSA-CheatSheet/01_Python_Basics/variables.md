# Variables in Python

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com
> Section: 01 — Python Basics
> 🔗 Related: [type casting](./type_casting.md) · [operators](./operators.md) · [input/output](./input_output.md) · Back to [README](../README.md)

## 1. What is it?

A **variable** is a named label that points to a value stored in memory. In Python, a variable is **not a box that holds a value** — it's a **name tag** attached to an object that lives somewhere in memory.

Think of Python variables like sticky notes with names written on them, stuck onto objects.

```python
x = 10        # 'x' is a sticky note on the integer object 10
name = "Asha"  # 'name' is a sticky note on the string object "Asha"
```

Key ideas:
- **Dynamic typing** — you don't declare the type; Python figures it out.
- **Dynamic binding** — the same name can be re-pointed to a different type later.
- Every value is an **object** with an identity, a type, and a value.

## 2. Why do we use it?

We use variables to:
- **Store** data so we can reuse it.
- **Name** data with human-readable labels (`pi` is clearer than `3.14159`).
- **Refer** to results of computations without recomputing them.
- Make code **readable** and **maintainable**.

Without variables, you'd have to repeat literals everywhere — error-prone and impossible to scale.

## 3. When should I choose it?

You use variables **always** — but *how* you use them depends on intent.

| Situation                                   | Use                                | Example                |
|---------------------------------------------|------------------------------------|------------------------|
| Value used more than once                   | Variable                           | `pi = 3.14`            |
| Value never changes                         | Constant (UPPER_CASE convention)   | `MAX_USERS = 100`      |
| Need to remember user input                 | Variable                           | `name = input()`       |
| Temp value, used once                        | Inline expression                  | `print(len("abc"))`    |
| Multiple values together                     | Tuple unpacking                    | `x, y = 1, 2`          |
| Need to track identity (same object?)        | `id()` / `is`                      | `id(x) == id(y)`       |

## 4. Syntax

```python
# Assignment
variable_name = value

# Multiple assignment (same value)
a = b = c = 0

# Multiple assignment (different values)
x, y, z = 1, 2, 3

# Swap (no temp variable needed!)
a, b = b, a

# Re-assign to a different type (dynamic typing)
v = 10       # int
v = "ten"    # now str — legal in Python
```

**Naming rules:**
- Start with a letter or `_`, followed by letters, digits, or `_`.
- Case-sensitive: `age`, `Age`, `AGE` are 3 different variables.
- Cannot be a keyword (`if`, `for`, `class`, ...).
- Convention: `snake_case` for variables, `UPPER_SNAKE` for constants, avoid `l`/`O`/`I` (look like digits).

## 5. Basic Example

```python
# Basic variable usage
age = 25
name = "Asha"
height = 5.6
is_student = True

print(name, "is", age, "years old and", height, "ft tall.")
print("Student?", is_student)

# Swap two variables
a, b = 10, 20
a, b = b, a
print("After swap -> a:", a, "b:", b)
```

**Output:**

```output
Asha is 25 years old and 5.6 ft tall.
Student? True
After swap -> a: 20 b: 10
```

## 6. Step-by-Step Dry Run

```python
x = 5
y = x
x = x + 1
```

| Step | `x` | `y` | Note                                   |
|------|-----|-----|----------------------------------------|
| 1    | 5   | —   | `x` points to int object `5`           |
| 2    | 5   | 5   | `y` also points to the same object `5` |
| 3    | 6   | 5   | `x + 1` creates new object `6`; `y` is unchanged |

**Why is `y` still 5?** Ints are **immutable**. `x + 1` doesn't modify the `5` object — it creates a new `6` object, and `x` is re-pointed to it. `y` still points to the original `5`.

## 7. Built-in Methods

| Method        | Purpose                                              | Syntax                | Input           | Output             | Example                        | Time      | Interview Use                  | Common Mistake                         | Shortcut                       |
|---------------|------------------------------------------------------|-----------------------|-----------------|--------------------|--------------------------------|-----------|--------------------------------|----------------------------------------|--------------------------------|
| `print()`     | Print to stdout                                      | `print(*objs, sep, end)` | Any objects | `None`             | `print(x)`                    | O(n) len  | Debugging, output formatting   | Forgetting `sep`/`end` args            | `print(*list)` unpacks        |
| `id()`        | Memory address (identity) of an object               | `id(obj)`             | Any object      | `int` address      | `id(x)`                       | O(1)      | Check if two vars are same obj  | Treating `id` as a meaningful number   | `a is b` is shorthand         |
| `type()`      | Type of an object                                    | `type(obj)`           | Any object      | `type` object      | `type(10)` → `<class 'int'>`  | O(1)      | Runtime type checking           | Using `type` for subclass checks        | Use `isinstance` for subtypes|
| `isinstance()`| Check if obj is instance of a class (or subclasses)   | `isinstance(o, T)`    | obj, type/tuple | `bool`             | `isinstance(x, int)`          | O(1)      | Safe type-checking              | Passing multiple types as separate args | Tuple of types ORs them        |
| `dir()`       | List attributes of an object/type                    | `dir(obj)`            | Any object      | `list` of strings  | `dir(x)`                      | O(n)      | Discover available methods      | Calling `dir()` with no args           | `dir(int)` for int methods    |
| `help()`      | Interactive help / docstring                         | `help(obj)`           | Any object      | docs to stdout     | `help(print)`                 | O(1)      | Learning APIs                   | Relying on it in interviews             | `__doc__` is raw docstring   |
| `del`         | Delete a variable (unbind name)                      | `del name`            | name            | —                  | `del x`                      | O(1)      | Cleanup, memory mgmt            | Deleting then re-using                    | Also slices list elements     |

### Examples

```python
x = 10
print(id(x))            # memory address — e.g. 430972layout
print(type(x))          # <class 'int'>
print(isinstance(x, int))   # True
print(isinstance(x, (int, float)))  # True — tuple means "int OR float"
print(dir(x))           # ['__abs__', '__add__', ..., 'bit_length', ...]
del x                   # unbinds the name x
```

**Output (address varies):**

```output
4309781360
<class 'int'>
True
True
['__abs__', '__add__', '__and__', ..., 'bit_length', ...]
```

## 8. Interview Example

> **Q: Swap two variables without using a temporary variable.**

```python
a, b = 7, 3
a, b = b, a
print(a, b)  # 3 7
```

**How it works:** Python evaluates the RHS tuple `(b, a)` → `(3, 7)` first, then unpacks it into `a, b`. No temp needed.

> **Q: Given a variable `x`, write a one-liner to print its type name.**

```python
x = 3.14
print(type(x).__name__)   # float
```

> **Q: Check if two variables point to the same object (not just equal value).**

```python
a = [1, 2]
b = a
c = [1, 2]
print(a is b)   # True  — same list object
print(a is c)   # False — different list objects (even with same contents)
print(a == c)   # True  — values are equal
```

**Key interview takeaway:** use `==` for value equality, `is` for identity equality. Small ints (-5..256) are **cached** in CPython, so `a is b` can be `True` for `a = 100; b = 100` purely by implementation luck — never rely on it.

## 9. When NOT to use

- Don't create a variable for a **value used only once** inline — it just adds noise.
- Don't reuse an existing variable name with a **completely different meaning** — confusing for readers.
- Don't use a variable to hold a **configuration constant** — use a real module-level `UPPER_CASE` name.
- Avoid single-letter names outside short loops/math contexts (`i`, `j`, `x`, `y` are fine in those).
- Don't depend on `id()` returning specific numbers across runs — they're addresses, not stable IDs.

## 10. Common Mistakes

| Mistake                                            | Why it's wrong                                          | Fix                                |
|----------------------------------------------------|----------------------------------------------------------|------------------------------------|
| `1x = 5`                                           | Variable names can't start with a digit                  | `x1 = 5`                          |
| `my-var = 5`                                       | `-` is not allowed in names                              | `my_var = 5`                       |
| `for = 5`                                          | Keyword collision                                         | `loop_count = 5`                   |
| Assuming `a is b` means `a == b`                   | Identity ≠ equality                                       | Use `==` for value checks          |
| Treating `int` as mutable (`x += 1` mutates x)     | Ints are immutable; `+=` rebinds the name                 | Remember new object is created     |
| Re-using a variable name with a different type      | Confuses readers and tools                               | Rename or use a fresh variable     |
| `print("x = " + 5)`                                | Can't concatenate str + int                              | `print("x = " + str(5))` or f-str  |
| Forgetting that `a = b` for lists shares reference | Mutations on `a` show up on `b`                          | Use `b = a.copy()` for independence|
| Using `l` or `O` as variable names                   | Look like `1` and `0`                                    | Use descriptive names              |
| `MAX = 100; MAX = 200`                              | Constants are convention only; Python doesn't enforce    | Treat as immutable by discipline   |

## 11. Memory Tricks

- Think **"name tag → object"**, not "box with value inside".
- `x = 5` ⇒ read right-to-left: assign `5` to `x`.
- **PEP 8 rhyme:** *"Constants SHOUT, variables whisper, keywords stay silent."*
- `is` = **identity** (same object); `==` = **equality** (same value).
- For swap: **"tuple the right, unpack the left"** — `a, b = b, a` means `(b, a)` is a tuple on the RHS.
- Immutable types: `int, float, str, tuple, bool, frozenset, bytes` — "you can't mutate me, you must rebind."
- Mutable types: `list, dict, set, bytearray` — "mutating me affects everyone pointing at me."

## 12. Interview Shortcuts

- `a, b = b, a` — swap without temp (RHS-formed tuple is unpacked).
- `type(x).__name__` — get the type's name as a string.
- `isinstance(x, (int, float, bool))` — check multiple types at once.
- `id(a) == id(b)` ⇔ `a is b` (technically same underlying op).
- Cached integers: CPython caches `-5` to `256`, so `a is b` may be `True` for those even if defined separately. **Don't rely on it.**
- `x = y = []` creates a **single shared list** — subtle bug source.
- `del x` only removes the name; the object gets garbage-collected when refcount hits 0.
- Variables are scoped to **functions, classes, modules** — Python has **no block scope** (a `for` loop variable leaks to outer scope).

## 13. Cheat Sheet Table

| Method / Operation        | Use                                                    |
|---------------------------|--------------------------------------------------------|
| `x = v`                   | Bind name `x` to value `v`                             |
| `a = b = c = 0`           | Bind many names to one value                            |
| `x, y, z = 1, 2, 3`       | Tuple unpacking                                         |
| `x, y = y, x`             | Swap                                                   |
| `del x`                   | Unbind name `x`                                        |
| `type(x)`                 | Get the type of `x`                                    |
| `isinstance(x, T)`        | Check `x` is instance of `T` (or subclass)            |
| `id(x)`                   | Memory address (identity)                              |
| `dir(x)`                  | List attributes                                         |
| `help(x)`                 | Show docs                                               |
| `x is y`                  | Identity check                                          |
| `x == y`                  | Value equality                                          |
| `x is None`               | Recommended null check                                  |
| `snake_case`              | Variable naming convention                              |
| `UPPER_SNAKE`             | Constant naming convention                              |

## 14. Time Complexity Table

| Method / Operation      | Complexity | Notes                                  |
|-------------------------|------------|----------------------------------------|
| Assignment `x = v`      | O(1)       | Just creates a name → object binding   |
| `id(x)`                 | O(1)       | Identity lookup                        |
| `type(x)`               | O(1)       | Returns type object                    |
| `isinstance(x, T)`      | O(1)       | (or O(depth of inheritance))           |
| `dir(x)`                | O(n)       | n = number of attributes               |
| `help(x)`               | O(1)       | Just prints docstring                  |
| `del x`                 | O(1)       | Decrements refcount; GC may run later |
| Multiple assignment      | O(k)       | k = number of targets                  |
| Tuple unpacking swap    | O(1)       | Just two bindings flipped              |

## 15. Visual Diagram (ASCII)

### A variable is a name pointing at an object in memory

```
   Python source             Memory
  ┌────────────┐         ┌────────────────────────┐
  │  x = 10    │         │  Name Table             │
  │            │         │  ┌──────┐   ┌────────┐  │
  │  y = x     │  ──────>│  │  x ──┼──>│ int 10 │  │
  │            │         │  │  y ──┼──┘  (id=0x1) │  │
  │  x = x + 1 │         │  └──────┘   └────────┘  │
  └────────────┘         │                         │
                         │  After "x = x + 1":     │
                         │  ┌──────┐   ┌────────┐  │
                         │  │  x ──┼──>│ int 11 │  │
                         │  │  y ──┼──>│ int 10 │  │
                         │  └──────┘   └────────┘  │
                         └────────────────────────┘
```

### Variable lifecycle

```
  ┌───────────┐     ┌──────────┐     ┌───────────────┐     ┌────────────┐
  │ Assign x=5│ ──> │ Use  x   │ ──> │ Rebind x = x+1│ ──> │ del x  /GC │
  └───────────┘     └──────────┘     └───────────────┘     └────────────┘
   (create)         (read)            (mutate-vs-rebind)    (cleanup)
```

### Identity vs equality

```
      a = [1,2]            b = a              c = [1,2]
        │                    │                    │
        v                    v                    v
     [1, 2]  <─────────── (same object)        [1, 2] (different object)
                                              (looks the same but separate)

     a is b  => True           a is c => False           a == c => True
     (same identity)            (different identity)      (same value)
```

## 16. Beginner Notes

> **Remember:**
> - A Python variable is a **name pointing to an object**, not a box holding a value.
> - Use **`snake_case`** for variables, **`UPPER_SNAKE`** for constants.
> - `==` compares **values**; `is` compares **identity**.
> - Immutable types (int, str, tuple) **rebind on every change**.
> - Mutable types (list, dict, set) **mutate in place** — sharing them shares mutations.
> - Use `isinstance(x, T)` for type checks, **not** `type(x) == T`.
> - For None checks, always use `x is None`.

## 17. FAANG Tips

- **Interviewers love swap questions** — the one-line tuple unpacking answer is the "Senior" answer.
- When asked about "pass by reference", clarify: Python is **pass-by-object-reference** (a.k.a. "pass by assignment"). Reassigning the name inside a function doesn't affect the caller; mutating a mutable object **does**.
- If an interviewer asks "does `x = y` copy the object?", answer: **no, it copies the reference**. Use `.copy()` or `copy.deepcopy()` for independent copies.
- The cached-integer range (`-5..256`) is a CPython implementation detail — useful trivia, but **not** portable.
- Code clarity wins bonus points: prefer `is_student = True` over `flag = 1`.
- When you write `x = 0; x += 1`, that creates and rebinds a new int object every iteration — relevant for hot loops; precompute where possible.
- Use `_` for values you must unpack but won't use: `_, y = some_tuple`.
- Distinguishing reference vs equality confusion is one of the most common root causes in debugging short snippets — practice explaining it out loud.

## 18. Practice Problems

### Easy
1. **Two Sum** — LeetCode #1 (use variables to track complement indexes)
2. **Swap two numbers without temp** — GeeksforGeeks basics
3. **Number of Good Pairs** — LeetCode #1512
4. **Reverse Integer** — LeetCode #7 (consider variable bound checks)

### Medium
1. **Integer to Roman** — LeetCode #12
2. **String to Integer (atoi)** — LeetCode #8
3. **Rotate Image** — LeetCode #48 (in-place variable swaps)

### Hard
1. **Candy** — LeetCode #135 (multi-pass variable bookkeeping)
2. **Trapping Rain Water** — LeetCode #42 (two-pointer variable strategy)

---

🔗 Next: [Input/Output](./input_output.md) · [Operators](./operators.md) · [Type Casting](./type_casting.md) · Back to [README](../README.md)
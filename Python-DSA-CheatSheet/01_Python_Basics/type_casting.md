# Type Casting in Python

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com
> Section: 01 — Python Basics
> 🔗 Related: [variables](./variables.md) · [operators](./operators.md) · [strings](../02_Data_Types/strings.md) · Back to [README](../README.md)

## 1. What is it?

**Type casting** (or **type conversion**) is the process of changing a value from one data type into another — for example, turning a string `"10"` into the integer `10`.

Python offers **two kinds** of casting:

- **Implicit casting** — Python converts for you automatically (e.g. `int + float → float`).
- **Explicit casting** — you call a constructor (`int()`, `float()`, `str()`, `bool()`, `list()`, `tuple()`, `set()`, `dict()`) to force the conversion.

Without casting, things like `input()` results (which are always `str`) can't be used in math.

## 2. Why do we use it?

- **`input()` returns a `str`.** To do arithmetic, you must cast: `n = int(input())`.
- To **convert structures** (list ↔ tuple ↔ set).
- To **standardize data** (strip NaN-like floats into None, normalize numeric strings).
- To **make safe comparisons** (`"10" == 10` is `False`; cast one so you compare same type).
- To pass values into functions / libraries that expect a specific type.
- To format output (`str(x)` for concatenation).

## 3. When should I choose it?

| Situation                                          | Use                                                | Example                                  |
|----------------------------------------------------|----------------------------------------------------|------------------------------------------|
| User typed a number; want `int` n                  | Explicit cast                                       | `n = int(input())`                       |
| Mixed math int + float                              | Implicit cast (Python handles it)                  | `1 + 2.0 → 3.0`                          |
| Number to text for printing                         | `str(x)`                                            | `str(10) + "!" → "10!"`                  |
| Text to number for math                              | `int(s)`, `float(s)`                                | `int("42")`, `float("3.14")`             |
| Boolean from truthy value                           | `bool(x)`                                            | `bool([])` → `False`                     |
| List to set to remove duplicates                   | `set(lst)`                                          | `set([1,1,2]) → {1,2}`                   |
| Set/tuple to list to index/mutate                   | `list(s)`                                            | `list({1,2,3})`                          |
| List to tuple (immutable)                           | `tuple(lst)`                                         | `tuple([1,2])`                           |
| List of `[k,v]` pairs to a dict                    | `dict(pairs)`                                        | `dict([("a",1),("b",2)])`                |
| Two-key strings into a dict-like                    | `dict(zip(keys, vals))`                             | `dict(zip(['a','b'],[1,2]))`             |

## 4. Syntax

```python
# Implicit (automatic by Python)
x = 10 + 2.5   # int 10 implicitly becomes float for the add; result is 12.5 float

# Explicit forms
int(x)         # int of a float (truncates toward 0), int of str (must look like int)
float(x)       # float of int/str
str(x)         # any type → string
bool(x)        # truthiness — falsy = 0, "", [], {}, (), None, False
list(x)        # any iterable → list
tuple(x)       # any iterable → tuple
set(x)         # any iterable → set (unordered, unique)
dict(x)        # iterable of key/value pairs → dict
bytes(x)       # int → zero bytes, str (with encoding) → bytes, iterable of ints → bytes

# .fromhex / etc. for special conversions live on the type, e.g.
bytes.fromhex("48656c6c6f")          # -> b'Hello'
```

**Important:** casts like `int("3.14")` **raise `ValueError`** — `int()` doesn't know how to parse a string with a `.`! Cast to float first, then int:
```python
int("3.14")        # ValueError
int(float("3.14")) # 3
```

## 5. Basic Example

```python
# str -> int / float
age_text = "25"
age = int(age_text)
print(age, age + 5)         # 25 30

# int -> float automatically (implicit)
total = 10 + 2.5
print(total, type(total))   # 12.5 <class 'float'>

# float -> int (truncation)
print(int(3.99), int(-3.99))    # 3 -3  (toward zero, not floor!)

# int / float -> str
print("Count = " + str(10))     # Count = 10

# list <-> set <-> tuple
arr = [1, 1, 2, 3, 3, 3]
uniq = set(arr)
print(uniq)                     # {1, 2, 3}
print(tuple(uniq))              # (1, 2, 3)   (or ordered minor editions)

# pairs -> dict
pairs = [("a", 1), ("b", 2)]
print(dict(pairs))              # {'a': 1, 'b': 2}

# truthiness / bool
print(bool(""), bool(0), bool(None), bool([]), bool({}), bool(()))   # False False False False False False
print(bool("0"), bool(0.0001))  # True True   (non-empty strings are truthy!)
```

**Output:**

```output
25 30
12.5 <class 'float'>
3 -3
Count = 10
{1, 2, 3}
(1, 2, 3)
{'a': 1, 'b': 2}
False False False False False False
True True
```

## 6. Step-by-Step Dry Run

```python
line = "3.14"
x = int(float(line))
y = str(x + 1)
z = list(y)
print(x, y, z)
```

| Step | Operation                  | Intermediate value                            | Resulting state                  |
|------|----------------------------|------------------------------------------------|----------------------------------|
| 1    | `line = "3.14"`             | str `"3.14"`                                   | `line = "3.14"`                  |
| 2    | `float("3.14")`             | `3.14` (float)                                 | intermediate                     |
| 3    | `int(3.14)`                 | truncates toward 0 → `3`                       | `x = 3`                          |
| 4    | `x + 1`                     | `3 + 1 = 4`                                     |                                  |
| 5    | `str(4)`                    | `"4"`                                           | `y = "4"`                         |
| 6    | `list("4")`                 | char-by-char: `["4"]`                           | `z = ['4']`                       |
| 7    | `print(x, y, z)`            |                                                | Output: `3 4 ['4']`              |

**Output:**

```output
3 4 ['4']
```

## 7. Built-in Methods

| Constructor | Purpose                                          | Syntax                       | Input                              | Output            | Example                                          | Time       | Interview Use                          | Common Mistake                                       | Shortcut                                       |
|-------------|--------------------------------------------------|------------------------------|------------------------------------|-------------------|--------------------------------------------------|------------|-----------------------------------------|------------------------------------------------------|------------------------------------------------|
| `int()`     | Convert to int                                    | `int(x, base=10)`             | str (int-like), float, bool         | `int`             | `int("42")`, `int(3.99)→3`, `int("ff", 16)→255`  | O(n) str   | Parse numerics, base conversion        | `int("3.14")` raises ValueError                      | Try/except ValueError for safety                |
| `float()`   | Convert to float                                  | `float(x)`                    | str (numeric), int                  | `float`           | `float("3.14")`, `float(5)`                       | O(n) str   | Math with floats                        | `float("")` raises ValueError                         | Use `try/except` to be safe                     |
| `str()`     | Convert anything to printable text               | `str(x)`                     | any object                          | `str`             | `str(10)`, `str([1,2])`, `str(None)` → "None"      | O(n)       | Concatenation, output                  | Using `+` between str and int TypeError              | Use f-strings instead where possible            |
| `bool()`    | Convert to truthy/falsy                            | `bool(x)`                    | any object                          | `bool`            | `bool(0)` → `False`, `bool("0")` → `True`         | O(1)       | Conditionals                            | `bool("0")` is True (non-empty string!)              | Just `if x:` directly is more Pythonic          |
| `list()`    | Convert iterable to list                           | `list(iter)`                  | any iterable                         | `list`            | `list("ab")` → `['a','b']`, `list((1,2))` → `[1,2]` | O(n)       | Mutate tuples/sets                     | `list(5)` raises TypeError (int is not iterable)     | `list(d.keys())` for dict keys                 |
| `tuple()`   | Convert iterable to tuple (immutable list)        | `tuple(iter)`                 | any iterable                         | `tuple`           | `tuple([1,2])` → `(1,2)`                          | O(n)       | Hashable keys (for dict keys)         | Using set operations on tuples (immutable)          | Use as dict keys                                |
| `set()`     | Convert iterable to set (unique, unordered)       | `set(iter)`                  | any iterable                         | `set`             | `set([1,1,2])` → `{1, 2}`                         | O(n)       | Deduplication                          | Order lost — don't rely on order                     | O(1) `in` test                                  |
| `dict()`    | Convert iterable of pairs to dict                 | `dict(iter_pairs)`            | iterable of `(k,v)` / `zip(k,v)`     | `dict`            | `dict([("a",1)])`, `dict(zip([1,2],[3,4]))`       | O(n)       | Build maps from lists                  | Passing single list of non-pairs raises ValueError   | `dict.fromkeys(arr, default)`                  |
| `bytes()`   | Convert to bytes (immutable byte string)         | `bytes(x)` / `bytes(s, enc)`  | int (count), iterable of ints, str (with encoding) | `bytes` | `bytes(3)` → `b'\x00\x00\x00'`, `bytes("hi","utf8")` → `b'hi'` | O(n)       | Binary protocols                       | Passing a str with no encoding raises TypeError     | `b.decode("utf8")` reverse                     |
| `ord()`     | Character → Unicode codepoint (`int`)            | `ord(c)`                      | single-char str                     | `int`             | `ord('A')` → 65                                   | O(1)       | Caesar cipher, hashing                 | Passing multi-char strings raises TypeError          | `chr(65)` → `'A'` is the inverse                |
| `chr()`     | Unicode codepoint → character                     | `chr(n)`                      | int                                 | `str`             | `chr(65)` → `'A'`                                 | O(1)       | Build chars from numeric IDs           | Negative or out-of-range codes raise ValueError       | `chr(ord(c)+1)` for next char                  |
| `hex()`     | int → hex string (with `0x`)                     | `hex(n)`                      | int                                 | `str`             | `hex(255)` → `'0xff'`                              | O(log n)   | Debug / print addresses                | Includes `0x` prefix; for raw use `'%x' % n`         | `int(hex_str, 16)` reverse                     |
| `bin()`     | int → bin string (with `0b`)                     | `bin(n)`                      | int                                 | `str`             | `bin(10)` → `'0b1010'`                             | O(log n)   | Bit manipulation output                | Negative ints return `'-0b...'`                      | `bin(n & 0xff)` for masked view                |
| `oct()`     | int → octal string (with `0o`)                   | `oct(n)`                      | int                                 | `str`             | `oct(8)` → `'0o10'`                                | O(log n)   | File permissions (rarely)              | Same prefix quirk                                    | `int(oct_str, 8)` reverse                       |

### Examples

```python
# Numeric conversions
print(int("42"), int("ff", 16), int("0b1010", 2))     # 42 255 10
print(float("3.14"), int(3.99), int(-3.99))           # 3.14 3 -3
print(int(3.99))                                      # 3 (truncation, not round)

# String conversions
print(str(10) + "!", len(str([1, 2, 3])))             # 10! 9

# Truthiness — surprising ones
print(bool("0"), bool("False"), bool([]))             # True True False

# Container conversions
print(set([1, 1, 2, 2, 3]))                            # {1, 2, 3}
print(list("hello"))                                  # ['h','e','l','l','o']
print(tuple({1, 2, 3}))                               # (1, 2, 3)
print(dict(zip(["a", "b"], [1, 2])))                  # {'a': 1, 'b': 2}

# Char <-> codepoint
print(ord('A'), chr(66))                              # 65 B
print(hex(255), bin(10), oct(8))                      # 0xff 0b1010 0o10
```

**Output:**

```output
42 255 10
3.14 3 -3
3
10! 9
True True False
{1, 2, 3}
['h', 'e', 'l', 'l', 'o']
(1, 2, 3)
{'a': 1, 'b': 2}
65 B
0xff 0b1010 0o10
```

## 8. Interview Example

> **Q: Read two integers from one line of input and report their sum.** (raw `input()` returns `str`)

```python
a, b = map(int, input().split())
print(a + b)
```

For input `10 20`:

```output
30
```

> **Q: Given a string of digits, return the integer it represents WITHOUT using `int()`.**

```python
def my_atoi(s):
    s = s.strip()
    if not s: return 0
    negative = False
    i = 0
    if s[0] in '+-':
        negative = (s[0] == '-')
        i = 1
    num = 0
    while i < len(s) and s[i].isdigit():
        num = num * 10 + (ord(s[i]) - ord('0'))   # cast char -> int via ord
        i += 1
    return -num if negative else num

print(my_atoi("  -12345 "))    # -12345
```

> **Q: Convert a decimal integer to its binary string WITHOUT `bin()`.**

```python
def to_bin(n):
    if n == 0: return "0"
    bits = []
    while n:
        bits.append(str(n & 1))
        n >>= 1
    return "".join(reversed(bits))

print(to_bin(10))   # 1010
```

> **Q: Count duplicates in a list efficiently.**

```python
arr = [1, 1, 2, 3, 3, 3, 4]
print(len(arr) - len(set(arr)))   # 3
```

> **Q: Convert a list of two-item tuples into a dict.**

```python
pairs = [("a", 1), ("b", 2), ("c", 3)]
print(dict(pairs))
# {'a': 1, 'b': 2, 'c': 3}
```

## 9. When NOT to use

- Don't cast strings to numerics **without validating** first — `int("abc")` raises `ValueError`. Wrap in `try / except`.
- Don't cast `int(x)` on huge floats — you may lose precision silently.
- Don't cast a `list` to a `set` if you need to **preserve insertion order** — pre-3.7 sets are unordered. Use a comprehension or `dict.fromkeys` to dedup while preserving order:
  ```python
  arr = [1, 1, 2, 3, 3, 4]
  uniq_in_order = list(dict.fromkeys(arr))   # [1, 2, 3, 4]
  ```
- Don't `bool("0")` thinking it's `False` — `"0"` is a non-empty string; truthy!
- Don't `float`-cast results where you need exact decimal math (money!) — use `decimal.Decimal`.
- Don't use `set()` if you'll have **unhashable** items (lists, dicts) inside — raises `TypeError`.
- Don't cast `dict` from a flat list of values — it must be a list of `(key, value)` pairs.
- Avoid `eval(input())` — use `ast.literal_eval` for safe structure parsing.

## 10. Common Mistakes

| Mistake                                          | Why                                              | Fix / Alternative                                  |
|--------------------------------------------------|--------------------------------------------------|----------------------------------------------------|
| `int("3.14")`                                    | `int()` can't parse "with dot" strings            | `int(float("3.14"))`                               |
| `int("10 ", base=10)` works but `int("10a")` fails | Whitespace tolerance is limited                  | Strip and validate; use regex                      |
| `bool("False")` returns `True`                   | Any non-empty str is truthy                       | `s.lower() in ("false", "0", "no")` mapping         |
| `int(7/2)` vs `7//2`                              | `int(7/2)` truncates; `7//2` floors — differ for negatives | Use the one you intend                            |
| `set([1,2,3])` for hashable OK, but `set([[1,2]])` | Unhashable element → TypeError                    | Convert inner lists to tuples first: `set((1,2),)` |
| `list(5)`                                         | int is not iterable                                | `list([5])` or just wrap                           |
| `dict([1, 2, 3])`                                 | Not iterable of 2-tuples                          | `dict(zip(keys, vals))`                            |
| `float("inf")` accidentally passed as `int`       | `int(float('inf'))` raises OverflowError          | Cap with `min/max` first                           |
| `str(b"abc")` → `"b'abc'"`                        | Bytes' `__str__` includes the `b'...'` syntax    | Use `b"abc".decode("utf-8")`                       |
| `list(d)` vs `list(d.items())`                    | First gives keys; second gives `(k,v)` pairs       | Pick what you actually want                         |
| Casting nested mutable structures (no deepcopy)   | Shallow cast shares inner objects                  | `copy.deepcopy` if needed                          |
| `tuple({1: 'a'})` returns `('a', None)` assumption | It returns the keys `(1,)`                         | Use `.items()` to get key/value pairs              |

## 11. Memory Tricks

- `"3.14"` has a dot ⇒ `int()` rejects it ⇒ you need **`int(float(s))`** — "float the dot, then int it."
- `bool("0") is True` because `"0"` is a non-empty string. When in doubt: digits aren't booleans — they're text! Convert explicitly: `s != "0"` etc.
- **List → set = unique. Set → list = deduplicated.** "Set is the doctor that removes duplicates."
- **`tuple` = "list locked in a vault"** — immutable `list`. Use as dict keys.
- `ord(c) - ord('0')` ⇒ integer value of digit `c` — common interview trick.
- `chr(n)` ⇒ the single-char string with codepoint `n` — `chr(ord('A') + k)` shifts in alphabets.
- `int(x, base)` ⇒ for `"0x.."` use `16`, `"0b.."` use `2`, `"0o.."` use `8`.
- `bin(x)`, `hex(x)`, `oct(x)` ⇒ always have `0b / 0x / 0o` prefixes.
- For `dict(zip(keys, vals))` ⇒ **"zip makes pairs, dict makes the map."**
- For money, never use float — `from decimal import Decimal`.
- For "preserve order while dedup": `list(dict.fromkeys(seq))`. (dicts retain insertion order from 3.7+).

## 12. Interview Shortcuts

- `int(x, 16)` parses hex strings without manual nibble work.
- `bin(n).count("1")` for hamming weight (slow but readable — good for cross-checks).
- `chr(ord('A') + k)` to walk the alphabet.
- `bool(x)` is the same as `x != 0` for numbers... but for strings `bool(x)` is `len(x) > 0`.
- `list(set(arr))` — **deduplicate** but you lose order. Use `list(dict.fromkeys(arr))` to keep order.
- `tuple(lst)` — **freeze** a list (use as a hashable dict key).
- `dict.fromkeys(keys, default)` — initialize a dict with identical defaults.
- `dict(zip(keys, vals))` — build a dict from two parallel lists.
- `str.join` over `+` for string building — much faster for big strings.
- Wrap `int(input())` in `try / except ValueError` for OA robustness:
  ```python
  try:
      n = int(input())
  except ValueError:
      n = 0
  ```
- Use `Decimal` for monetary problems; never binary floats.
- `bytes.decode()` / `str.encode()` — bridge binary ↔ text.

## 13. Cheat Sheet Table

| Conversion              | Use                                                  |
|------------------------|------------------------------------------------------|
| `int(s, base)`          | Parse numeric string → int (with optional base)      |
| `int(x)`                | Truncate float → int (toward 0)                       |
| `float(s)` / `float(n)` | Parse numeric string → float / int → float             |
| `str(x)`                | Anything → printable text                             |
| `bool(x)`               | TRUTHINESS — empty/zero/None = False                 |
| `list(iter)`            | Any iterable → list                                   |
| `tuple(iter)`           | Any iterable → tuple (immutable list)                |
| `set(iter)`             | Any iterable → set (unique, unordered, hashable only) |
| `dict(pairs_iter)`      | Iterable of `(k,v)` pairs → dict                     |
| `dict(zip(k, v))`       | Two parallel lists → dict                            |
| `dict.fromkeys(k, d)`   | Keys with default value `d`                           |
| `bytes(n)` / `bytes(s, enc)` | Numeric count, or encode string to bytes       |
| `b.decode(enc)`         | bytes → str                                          |
| `ord(c)`                | char → Unicode code (int)                            |
| `chr(n)`                | Unicode code (int) → single-char string               |
| `bin / oct / hex(n)`    | int → string with `0b/0o/0x` prefix                  |
| `int("0xff", 16)`       | Parse hex string → int                                 |
| `int("0b1010", 2)`      | Parse binary string → int                             |

## 14. Time Complexity Table

| Conversion                   | Complexity | Notes                                                |
|------------------------------|------------|------------------------------------------------------|
| `int(s)`                     | O(n)       | n = length of string                                 |
| `int(s, base)`               | O(n)       | Same                                                |
| `float(s)`                   | O(n)       |                                                      |
| `int(float_number)`          | O(1)       | Truncation of bounded float                           |
| `str(x)`                     | O(n)       | n = output length                                    |
| `bool(x)`                    | O(1)       | Boolean of one object                                |
| `list(iter)`                 | O(n)       | Iteration cost                                       |
| `tuple(iter)`                | O(n)       |                                                      |
| `set(iter)`                  | O(n)       | n hashes computed                                    |
| `dict(pairs)`                | O(n)       | n hashes for keys                                    |
| `dict.fromkeys(k, v)`        | O(n)       | n = number of keys                                  |
| `dict(zip(k, v))`            | O(n)       | O(n) for zip + O(n) for dict                         |
| `bytes(s, enc)`              | O(n)       | Encode cost                                          |
| `bytes(iter)`                | O(n)       | Copy                                                  |
| `ord / chr`                  | O(1)       | Single-char conversions                             |
| `bin / oct / hex`            | O(log n)   | Number of bits/digits in representation               |
| `int.from_bytes / to_bytes`  | O(n)       | n = number of bytes                                  |

## 15. Visual Diagram (ASCII)

### `"10"` (str) → `10` (int) → `10.0` (float)

```
   ┌──────┐                  ┌──────┐                   ┌──────┐
   │ "10" │   int("10")     │  10  │    float(10)     │ 10.0 │
   │ str  │ ─────────────>  │ int  │  ─────────────>   │ float│
   └──────┘                  └──────┘                   └──────┘
   Memory                      Memory                    Memory
   2 chars, codepoints         single int object         single float object
   '1' (49), '0' (50)          value 10                  value 10.0
```

### `int("3.14")` fails — must route via `float()`

```
            "3.14"`
              │
   ── int()┬──>  ✗ ValueError ("invalid literal for int(): 3.14")
              │
              v
            float("3.14") ──> 3.14 (float)
                                │
                                v
                              int(3.14) ──> 3 (toward zero)
```

### List ↔ Set ↔ Tuple ↔ List

```
       [1, 1, 2, 3]            {1, 2, 3}              (1, 2, 3)
           │  set()            │  tuple()              │  list()
           v                    v                       v
       set: unique          tuple: immutable         list: mutable, ordered
            & unordered
```

### Two parallel lists → `zip` → `dict`

```
   keys   = [ "a" , "b" , "c" ]
   vals   = [ 1    , 2    , 3    ]
                  │   zip(keys, vals)
                  v
   [("a", 1), ("b", 2), ("c", 3)]
                  │
                  v dict(...)
   { "a": 1, "b": 2, "c": 3 }
```

### Char ↔ codepoint

```
   ord('A') ────> 65          chr(65) ────> 'A'
   ord('1') ────> 49          chr(49) ────> '1'
   digit value:  ord(c) - ord('0')    (e.g., ord('5')-ord('0') = 5)
```

## 16. Beginner Notes

> **Remember:**
> - `input()` **always returns a `str`**. To do math, wrap with `int()` / `float()`.
> - `int("3.14")` raises `ValueError`. Use `int(float("3.14"))`.
> - `int(3.99) == 3` (truncates toward zero), but `7 // 2 == 3` (floors toward `-∞`). They differ for negatives.
> - `bool("0")` is `True` because `"0"` is a **non-empty string**. Truthiness isn't the same as numeric-zero.
> - `set(arr)` **dedups** but loses order. To preserve order: `list(dict.fromkeys(arr))`.
> - `tuple(lst)` freezes the list — useful for dict keys.
> - `dict(zip(keys, vals))` builds a dict from two parallel lists.
> - For money, use `decimal.Decimal`, never `float`.
> - Always guard `int(input())` with `try / except ValueError` for safety.
> - `int(s, base)` lets you parse `"0xff"` as hex or `"0b1010"` as binary directly.

## 17. FAANG Tips

- Most OA pipelines boil down to: **read string → cast → compute**. Master `list(map(int, line.split()))`.
- For **string → int** without `int()`, interviewers want you to demonstrate `ord(c) - ord('0')` digit-by-digit parsing (atoi pattern).
- For **int → string** without `str()`, build from `chr(ord('0') + digit)` parallels.
- **Order-preserving dedup**: `list(dict.fromkeys(seq))` is a one-liner the interviewer will love.
- Always **sanitize** user inputs in interview questions — `try/except ValueError` shows maturity.
- Be fluent in `ord`/`chr` — fundamental for character classes (lowercase/uppercase shifts, Caesar ciphers, ASCII → integer ID mapping for arrays indexed by character).
- Explain **why** floats are dangerous for money (binary representation rounds `0.1 + 0.2 ≠ 0.3`) — punt to `decimal.Decimal`.
- **2D ↔ 1D index**: `idx = r * cols + c` and `(r, c) = divmod(idx, cols)` — common in matrix flattening.
- For **base conversion** problems (e.g. LeetCode 405: Hex Digit), the `ord/str` mapping shows mastery.
- Know that `int("0x1f", 16)` returns `31` (Python automatically skips the `0x` prefix when given `16`).
- `dict(zip(...))` is much faster than building the dict with a loop — use it.
- For DP-on-bitmask problems: cast lists to tuples for dictionary keys quickly with `tuple(lst)`.

## 18. Practice Problems

### Easy
1. **A + B Problem** — starter (read ints from input, cast, sum)
2. **Find the Difference** — LeetCode #389 (ord/chr-shift algorithm)
3. **String to Integer (atoi)** — LeetCode #8 (manual parsing using `ord`)
4. **Excel Sheet Column Number** — LeetCode #171 (base-26 conversion via ord)
5. **Excel Sheet Column Title** — LeetCode #168 (reverse base-26)

### Medium
1. **Encode and Decode TinyURL** — LeetCode #535 (string ↔ container casts)
2. **Integer to Roman** — LeetCode #12 (int → string conversion pattern)
3. **Roman to Integer** — LeetCode #13 (string parsing → int)
4. **String to Integer (atoi)** — LeetCode #8 (string → int)

### Hard
1. **Serialize and Deserialize Binary Tree** — LeetCode #297 (string ↔ data structure)
2. **Encode and Decode Strings** — LeetCode #271 (string serialization with int-length prefixes)
3. **Smallest Range Covering Elements from K Lists** — LeetCode #632 (often needs cast/oracle strategies)

---

🔗 Next: [Strings](../02_Data_Types/strings.md) · [Variables](./variables.md) · [Operators](./operators.md) · Back to [README](../README.md)
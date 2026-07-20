# Operators in Python

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com
> Section: 01 — Python Basics
> 🔗 Related: [variables](./variables.md) · [type casting](./type_casting.md) · [input/output](./input_output.md) · Back to [README](../README.md)

## 1. What is it?

Operators are **symbols (and keywords)** that perform an operation on one or more values (called *operands*) and produce a result.

Python has **7 operator categories**:

1. **Arithmetic** — `+ - * / // % **`
2. **Comparison / Relational** — `== != < > <= >=`
3. **Logical** (boolean) — `and`, `or`, `not`
4. **Bitwise** — `& | ^ ~ << >>`
5. **Assignment** — `= += -= *= /= //= %= **= &= |= ^= <<= >>=`
6. **Membership** — `in`, `not in`
7. **Identity** — `is`, `is not`

There are also **augmented assignment** operators and **special methods** (`__add__`, `__eq__`, ...) that let custom classes redefine most operators.

## 2. Why do we use it?

- To **compute** values (`a + b`, `c * d`).
- To **compare** values (`x == y`, `score > 90`).
- To **decide** flow (`if x and not y:`).
- To **manipulate bits** (low-level / crypto / DP bitmask tricks).
- To **check membership** (`key in dict`, `item in list`).
- To **assign compactly** (`x += 1`).
- To **test identity** (`x is None`).

## 3. When should I choose it?

| Situation                                  | Category            | Operator                                 |
|--------------------------------------------|---------------------|------------------------------------------|
| Compute numerical result                   | Arithmetic          | `+ - * / // % **`                         |
| Quick exponentiation                       | Arithmetic          | `**`                                     |
| Integer division remainder                 | Arithmetic          | `%`, `//`                                |
| Compare two values                          | Comparison          | `== != < > <= >=`                         |
| Compound boolean test                      | Logical             | `and or not`                              |
| Bit-level mask / DP bitmask                | Bitwise             | `& | ^ << >> ~`                           |
| Set operations on bitmasks                 | Bitwise             | `& | ^ ~`                                |
| Modify value in place                       | Augmented           | `+= -= *= /= //= %=`                      |
| Check if element exists in container        | Membership          | `in not in`                              |
| Check if `x` is the same object             | Identity             | `is is not`                              |
| Null comparison                             | Identity             | `x is None`                              |

## 4. Syntax

```python
# Arithmetic
a + b        # addition
a - b        # subtraction
a * b        # multiplication
a / b        # true division (always float)
a // b       # floor division (integer floor)
a % b        # modulus (remainder — Python-sign)
a ** b       # exponent (pow)

# Comparison
a == b   a != b   a < b   a > b   a <= b   a >= b

# Logical (TRUE if both / either / not)
a and b
a or b
not a

# Bitwise
a & b    # AND  — set intersection of bits
a | b    # OR   — set union of bits
a ^ b    # XOR  — set symmetric difference
~a       # NOT  — one's complement (≈ -(a+1))
a << n   # left shift n bits   == a * (2**n)
a >> n   # right shift n bits  == a // (2**n)

# Assignment + augmented
x = 5
x += 1    # x = x + 1
x //= 2   # x = x // 2
x **= 3   # x = x ** 3

# Membership
3 in [1, 2, 3]              # True
"ab" in "abc"               # True
10 not in {1, 2, 3}         # True

# Identity
x = None
print(x is None)    # True
print(x is not None) # False
```

## 5. Basic Example

```python
a, b = 13, 6

print("Arithmetic")
print(a + b)   # 19
print(a - b)   # 7
print(a * b)   # 78
print(a / b)   # 2.16666...   (true division)
print(a // b)  # 2            (floor division)
print(a % b)   # 1            (remainder)
print(a ** b)  # 4826809

print("\nBitwise")
print(a & b)   # 4   (110 & 010 = 010)
print(a | b)   # 15  (110 | 011 = 111)
print(a ^ b)  # 11  (110 ^ 011 = 101)
print(~a)       # -14
print(a << 1)  # 26  (left shift one bit)
print(a >> 1)  # 6   (right shift one bit)

print("\nLogical")
print(a > b and a < 100)   # True and True -> True
print(a > b or a == 0)    # True or False  -> True
print(not (a == b))        # not False -> True

print("\nMembership & Identity")
print(6 in [1, 6, 9])           # True
print(7 not in [1, 6, 9])       # True
print(a is b)                   # False
print(None is None)             # True
```

**Output:**

```output
Arithmetic
19
7
78
2.1666666666666665
2
1
4826809

Bitwise
4
15
11
-14
26
6

Logical
True
True
True

Membership & Identity
True
True
False
True
```

## 6. Step-by-Step Dry Run

```python
a = 7
b = 3
c = a // b * 2 + (a % b) ** 2
flag = a > b and not (b == 0)
```

**Precedence (high → low):** `**` → unary → `* / // %` → `+ -` → comparison → `not` → `and` → `or` → `=`.

| Step | Expression evaluated           | Result                              |
|------|-------------------------------|-------------------------------------|
| 1    | `a // b`                       | `7 // 3 = 2`                        |
| 2    | `(a // b) * 2`                  | `2 * 2 = 4`                         |
| 3    | `a % b`                        | `7 % 3 = 1`                         |
| 4    | `(a % b) ** 2`                  | `1 ** 2 = 1`                        |
| 5    | `4 + 1`                        | `5` → `c = 5`                       |
| 6    | `a > b`                        | `7 > 3 → True`                       |
| 7    | `b == 0`                       | `3 == 0 → False`                     |
| 8    | `not (b == 0)`                  | `not False → True`                   |
| 9    | `a > b and not (b == 0)`        | `True and True → True` → `flag = True` |

Final: `c = 5`, `flag = True`.

## 7. Built-in Methods

| Method / Operator       | Purpose                                                | Syntax          | Input               | Output            | Example                          | Time        | Interview Use                        | Common Mistake                                  | Shortcut                            |
|------------------------|--------------------------------------------------------|-----------------|---------------------|-------------------|----------------------------------|-------------|-----------------------------------------|--------------------------------------------------|--------------------------------|
| `+ - *`                | Arithmetic basics                                      | `a + b`, etc.   | numbers / strings   | number / str      | `2+3` → `5`, `"a"+"b"`→"ab"     | O(1)        | Math basics                          | Using `+` on different types causes TypeError    | `sum(x for x in arr)`           |
| `/` and `//`           | True vs floor division                                 | `a / b`, `a // b` | numbers             | float / int       | `7/2=3.5`, `7//2=3`            | O(1)        | Avoid float bugs                     | `7//2 == 3` but `-7//2 == -4` (floors toward -∞) | Use `int(a/b)` for truncation   |
| `%`                    | Remainder (signed by Python)                           | `a % b`         | numbers             | same-sign result  | `7%3=1`, `-7%3=2`              | O(1)        | Cyclic counters, parity             | `%` with negative operands surprises people      | `a % n` always in [0, n)        |
| `**`                   | Power                                                   | `a ** b`        | numbers             | number            | `2**10 = 1024`                | O(log b) for ints? Benchmarks vary | Powers of 2, fast doubling              | `**` has right-binding & is high precedence      | `pow(a, b, m)` for modular pow |
| `pow(a, b, m)`         | Built-in modular exponentiation                         | `pow(a, b, m)`  | ints                | int               | `pow(2, 10, 1000) = 24`        | O(log b)    | Modpow for crypto/RSA/coding        | Forgetting mod → very large ints                  | Faster than `(a**b) % m`        |
| `divmod(a, b)`         | Both quotient and remainder in one call                | `divmod(a, b)`  | ints                | tuple `(q, r)`    | `divmod(7, 3)` → `(2, 1)`       | O(1)        | Euclidean algorithm, base conversion | Re-deriving r from `a-b*q`                       | Use in GCD loop               |
| `abs(x)`               | Absolute value                                          | `abs(x)`        | number              | non-negative      | `abs(-3.5)` → `3.5`            | O(1)        | Man dist for grids                  | Using `math.fabs` unnecessarily                   | `(x*x)**0.5` ≠ `abs` (signed safer)|            |
| `round(x, n)`          | Round to n decimals (banker's rounding!)                | `round(x, n=0)` | float, int          | number            | `round(2.5)` → `2`, `round(3.5)` → `4` | O(1) | Outwit rounding bugs                | Banker's rounding biases toward even             | `Decimal` if precise rounding required |
| `min`/`max`            | Min / max of iterable or args                          | `min(*args)`, `min(iter)` | args / iterable | element of same type | `max(3, 1, 4)` → `4`            | O(n)        | Stream min/max for greedy           | `min([])` raises ValueError                     | Use `default=` (3.4+)          |
| `sum(iter, start=0)`   | Sum an iterable                                          | `sum(arr, 0)`   | iterable of numbers | number            | `sum([1,2,3])` → `6`           | O(n)        | Save a loop                          | `sum(arr, start)` — start is the 2nd arg         | `sum(2*x for x in arr)`         |
| `in` / `not in`        | Membership                                              | `x in c`        | element + container | `bool`            | `3 in (1,2,3)` → `True`        | O(1) for set/dict, O(n) for list | Set/dict lookups                    | Using `in` on a list when set would do             | Convert to set for fast lookup  |
| `is` / `is not`        | Identity                                                | `x is y`        | objects             | `bool`            | `x is None`                     | O(1)        | Null checks, sentinel                 | Using `is` for value equality (cached ints trick) | Always `is None`               |

### Examples

```python
print(7 // 2, -7 // 2)         # 3 -4   (floor, not truncation)
print(7 % 3, -7 % 3)           # 1 2    (Python sign follows divisor)
print(2 ** 10)                 # 1024
print(pow(2, 10, 1000))        # 24
print(divmod(7, 3))            # (2, 1)
print(round(2.5), round(3.5))  # 2 4   (banker's rounding)
print(min([3, 1, 4]), max(3, 1, 4))   # 1 4
print(sum(range(1, 6)))               # 15
```

**Output:**

```output
3 -4
1 2
1024
24
(2, 1)
2 4
1 4
15
```

## 8. Interview Example

> **Q: Check if `n` is even without using `if/else` or `%`.**

```python
n = 14
is_even = (n & 1) == 0
print(is_even)   # True
```

Bit-AND with `1` returns the lowest bit. Even numbers have LSB = 0.

> **Q: Given a number `n`, count the number of 1-bits in its binary representation.**

```python
n = 23      # binary 10111 → three? 1+1+1+0+1 = 4 ones
count = 0
while n:
    count += n & 1
    n >>= 1
print(count)       # 4
# or one-liner: print(bin(23).count('1'))
```

> **Q: Implement `a^b mod m` efficiently — but no `pow`.**

```python
def modpow(a, b, m):
    res = 1
    a %= m
    while b:
        if b & 1:
            res = (res * a) % m
        a = (a * a) % m
        b >>= 1
    return res

print(modpow(2, 10, 1000))   # 24
```

> **Q: Short-circuit demo: avoid division by zero using logical AND.**

```python
def safe_inv(x):
    return 1 / x if x != 0 else float('inf')

print(safe_inv(0))   # inf
```

Or using short-circuit:

```python
print(x != 0 and 1 / x or float('inf'))
```

## 9. When NOT to use

- Don't use `//` when you want **truncation toward zero** — Python floors toward negative infinity. For truncation toward zero use `int(a / b)` or `math.trunc(a / b)`.
- Don't use `is` for value equality (e.g. `x is 5` — might work due to int caching, but undefined).
- Don't use `^` as a power operator — `^` is **XOR**, not exponentiation. Use `**`.
- Don't use chained comparisons that you didn't mean: `0 < x < 100` is great, but `x < y > z` can read ambiguously.
- Don't use `and`/`or` as if statements in confusing ways; explicit `if` reads better.
- Don't use bitwise ops as clever micro-optimizations unless you actually measured hotspots.
- Don't chain augmented ops on mutable types if you don't understand in-place behavior (`list += [x]` mutates, `list = list + [x]` creates a new list).

## 10. Common Mistakes

| Mistake                                                  | Why it's wrong                                            | Fix                                                  |
|----------------------------------------------------------|-----------------------------------------------------------|------------------------------------------------------|
| `^` for power                                            | `^` is XOR; `**` is power                                  | Use `a ** b`                                          |
| `/` returns int                                          | `/` always returns `float`                                | Use `//` for int结果的 int floor                       |
| `-7 // 2 == -3`                                          | Floor division yields `-4` in Python                       | Use `math.trunc(-7 / 2)` for truncation toward zero    |
| `2 ** 3 ** 2 = 64`                                       | `**` binds right-associatively → `2**(3**2) = 2**9 = 512` | Parenthesize to be explicit                            |
| `a < b < c` parsed as `(a<b) and (b<c)`                  | Surprises beginners                                        | Use it deliberately, it's correct                     |
| `x is True`                                               | Use `== True` or (better) just truthiness: `if x:`        | Use `x` as a boolean directly                         |
| `round(2.5)` returns `2` not `3`                          | Banker's rounding (round-half-even)                       | Use `Decimal` for exact rounding                       |
| `flag and x or y` shorter than `x if flag else y`         | Only equivalent when `x` is truthy                         | Use ternary `x if cond else y`                         |
| Comparing strings with `==` is value equality            | Correct, fast                                              | Don't use `is`                                        |
| Side effects with `+=` on lists                          | `+=` mutates in place, `= x + y` rebinds                  | Pick the one you mean                                  |
| Using `id(a) == id(b)` to test equality                  | Identity ≠ equality                                        | Use `a == b`                                          |
| `if x == None`                                            | Should be `if x is None`                                   | Use `is` for None                                      |

## 11. Memory Tricks

- **Floor division floor-to-negative:** think `→ -∞`, not "drop the decimal".
- **Modulo follows divisor's sign:** `7 % 3 = 1`, `-7 % 3 = 2`, `7 % -3 = -2`.
- Bitwise AND is set intersection: `a & b`.`OR` is union, `XOR` is symmetric difference.
- **`<<` doubles, `>>` halves (floor)** — `x << n == x * 2**n`.
- **`~x == -(x + 1)`** — a one's-complement consequence.
- **`**` migrates right:** `2 ** 3 ** 2 == 2 ** 9 == 512`.
- Logical ops **short-circuit**: `a and b` stops at first falsy; `a or b` stops at first truthy.
- `x is y` ⇒ identity check (exact same object); `x == y` ⇒ value check (`__eq__`).
- Equality vs Identity rhyme: *"equal value? use `==`. The exact same object is? `is`. "*
- For bitmask counting: `bin(x).count('1')` (correctness), `x & (x-1)` removal (loop).
- Postfix thought: ternary `a if cond else b` reads as English.

## 12. Interview Shortcuts

- **Even/odd**: `n & 1 == 0` ⇒ even; `n & 1` ⇒ odd. Faster than `n % 2`.
- **Multiply / divide by powers of 2**: `x << k` = `x * 2**k`; `x >> k` = `x // 2**k`.
- **Modpow**: `pow(a, b, m)` is built-in modular exponentiation. Use it for crypto problems.
- **Toggle bit k**: `x ^= (1 << k)`. **Set bit k**: `x |= (1 << k)`. **Clear bit k**: `x &= ~(1 << k)`. **Test bit k**: `(x >> k) & 1`.
- **Swap two ints without temp**: `a ^= b; b ^= a; a ^= b` (or simpler, `a, b = b, a`).
- **Floor division for negatives**: remember Python floors toward `-∞`, not zero.
- **Banker's rounding**: `round()` uses round-half-to-even; don't be surprised.
- **Truthiness defaults**: `[]`, `{}`, `()`, `""`, `0`, `None`, `False` are falsy; everything else is truthy.
- **Chained comparison**: `0 <= x <= 100` is Pythonic and fast.
- **`in`**: O(1) on set/dict, O(n) on list/tuple/str. **Cast to set** for lookups.
- **Division by 0** in Python raises `ZeroDivisionError` — guard explicitly.

## 13. Cheat Sheet Table

### Arithmetic
| Operator | Use                                  | Example             |
|----------|--------------------------------------|---------------------|
| `+`      | Add numbers / concat str / list      | `2+3 → 5`           |
| `-`      | Subtract                              | `5-2 → 3`           |
| `*`      | Multiply / repeat str / list          | `2*3 → 6`, `"ab"*2` → "abab"  |
| `/`      | True division (always float)         | `7/2 → 3.5`         |
| `//`     | Floor division (floor toward -∞)    | `7//2 → 3`, `-7//2 → -4` |
| `%`      | Modulus (sign follows divisor)        | `7%3 → 1`, `-7%3 → 2` |
| `**`     | Exponent (right-associative)         | `2**3 → 8`, `2**3**2 → 512` |

### Comparison / Relational
| Operator | Use                                  |
|----------|--------------------------------------|
| `==`     | Value equality                        |
| `!=`     | Value inequality                      |
| `<`      | Less than                             |
| `>`      | Greater than                          |
| `<=`     | Less than or equal                    |
| `>=`     | Greater than or equal                  |
| `a < b < c` | Chained correctness               |

### Logical
| Operator | Use                                  | Short-circuits on            |
|----------|--------------------------------------|------------------------------|
| `and`    | Both truthy                          | First falsy                  |
| `or`     | At least one truthy                  | First truthy                 |
| `not`    | Invert truthiness                    | N/A                          |

### Bitwise
| Operator | Use                                   |
|----------|---------------------------------------|
| `&`      | AND (intersection of bits)             |
| `|`      | OR (union of bits)                    |
| `^`      | XOR (symmetric difference)            |
| `~`      | NOT (one's complement = `-(x+1)`)     |
| `<<`     | Left shift `n` bits = `* 2**n`         |
| `>>`     | Right shift `n` bits = `// 2**n`       |

### Assignment
| Operator | Equivalent |
|----------|------------|
| `x = v`  | plain      |
| `x += v` | `x = x + v` |
| `x -= v` | `x = x - v` |
| `x *= v` | `x = x * v` |
| `x /= v` | `x = x / v` |
| `x //= v`| `x = x // v`|
| `x %= v` | `x = x % v`|
| `x **= v`| `x = x ** v`|
| `x &= v` | `x = x & v` |
| `x |= v` | `x = x | v` |
| `x ^= v` | `x = x ^ v` |
| `x <<= v`| `x = x << v`|
| `x >>= v`| `x = x >> v`|
| `x @= M` | matrix mul (NumPy) |

### Membership & Identity
| Operator  | Use                          |
|-----------|------------------------------|
| `in`      | Element in container          |
| `not in`  | Element not in container      |
| `is`      | Same object (identity)        |
| `is not`  | Different object              |

### Operator precedence (highest → lowest)
| Level | Operators                                            |
|-------|------------------------------------------------------|
| 1     | `(expressions...)`, `[...]`, `{...}`, indexing, call |
| 2     | `**` (right-associative)                              |
| 3     | unary `+ - ~`                                        |
| 4     | `* / // %`                                           |
| 5     | `+ -`                                                |
| 6     | `<< >>`                                              |
| 7     | `&`                                                  |
| 8     | `^`                                                  |
| 9     | `|`                                                  |
| 10    | `in`, `not in`, `is`, `is not`, `< <= > >= != ==`     |
| 11    | `not x` (logical NOT)                                |
| 12    | `and`                                                |
| 13    | `or`                                                 |
| 14    | ternary `a if cond else b`                           |
| 15    | lambda                                                |
| 16    | `=`, augmented assignments                            |

## 14. Time Complexity Table

| Operator / Function      | Complexity | Notes                                            |
|--------------------------|------------|--------------------------------------------------|
| `+`, `-`, `*`            | O(1)       | For numbers of bounded size                       |
| `/`, `//`, `%`           | O(1)       | Bounded precision                                 |
| `**` (with int b)        | O(log b * M(log a)) | Bignum, depends on size                  |
| `pow(a, b, m)`           | O(log b)   | Modular exponentiation                            |
| Bitwise `& | ^ ~ << >>`  | O(1)       | For bounded ints; bignum shifts scale linearly   |
| `in` on `set`/`dict`     | O(1)       | Hash lookup, amortized                            |
| `in` on `list`/`tuple`   | O(n)       | Linear scan                                       |
| `in` on `str`            | O(n+m)     | Substring search                                  |
| `is`                     | O(1)       | Pointer comparison                                |
| `==` on numeric          | O(1)       |                                                   |
| `==` on `str`           | O(n)       | Per-char comparison                               |
| `==` on `list`/`tuple`  | O(n)       | Element-wise                                       |
| Chained comparison       | O(k)       | k = number of comparison ops                      |

## 15. Visual Diagram (ASCII)

### Arithmetic flow on `7 / 3 vs 7 // 3 vs 7 % 3`

```
     7 / 3  ───────────>   2.333...                (float, always)
     7 // 3 ──> floor(2.333)──> 2                  (integer floor)
     7 %  3 ──> 7 - (3 * 7//3) = 7 - 6 = 1         (remainder)
```

### Floor division sign behavior

```
            Number line (negative left, positive right):

       -7 // 2 = -4              -7 / 2 = -3.5            -7 % 2 = +1
                  ──────────────────────────────────> X
        ▼                ▼
       -3.5       floor = -4 (toward -∞)        rem = -7 - (-4*2) = 1

           Python floored division: quotient floor to -∞,
           remainder has the same sign as the divisor.
```

### Truth tables for `and / or / not`

```
   A │ B │ A and B │ A or B │ not A
   ──┼───┼─────────┼────────┼───────
   F │ F │    F    │   F    │   T
   F │ T │    F    │   T    │   T
   T │ F │    F    │   T    │   F
   T │ T │    T    │   T    │   F
```

### Bitwise view of `13 & 6`, `13 | 6`, `13 ^ 6`

```
        binary     decimal
   a =  1 1 0 1   =    13
   b =  0 1 1 0   =     6
       ───────
   a & b = 0 1 0 0   =  4
   a | b = 1 1 1 1   = 15
   a ^ b = 1 0 1 1   = 11
   ~a    = ...1111 0010  = -14   (two's complement)
   a << 1 = 1 1 0 1 0  = 26    (doubled)
   a >> 1 =   0 1 1 0  = 6     (halved)
```

### Bit-AND parity check

```
              n  = 13 = binary 1 1 0 1
              n & 1 →              1 (last bit)  → odd
              n & 1 → 0 for even n
```

## 16. Beginner Notes

> **Remember:**
> - `**` is power, `^` is XOR. Common slip on day 1.
> - `/` always gives a **float**; `//` gives the integer floor (toward `-∞`).
> - `%` has the **same sign as the divisor** — useful for circular buffer indices.
> - `and / or` **short-circuit**: handy for guards like `x != 0 and 1/x`.
> - `is` compares identity (same object). `==` compares value. Use **`is None`** for None checks.
> - `in` is **O(1) for sets/dicts, O(n) for lists/tuples**.
> - `**` is **right-associative**: `2**3**2 == 2**9 == 512`.
> - Bitwise ops are great for **subsets, masks, and DP-on-bits** tricks.
> - Build readable expressions using **chained comparisons** (`0 <= x <= 100`).
> - Avoid `^` for power and `is` for numeric equality — both big-newbie mistakes.

## 17. FAANG Tips

- Interviews love **bitwise tricks**: even/odd with `n & 1`, swap with XOR, count set bits, toggle bit k, single number problems (XOR). Master `n & (n-1)` for "drop lowest set bit".
- For "power of 2" check: `n > 0 and (n & (n - 1)) == 0`.
- **Modular arithmetic** and `pow(a, b, m)` recur in cryptography, combinatorics, and number theory problems.
- Watch for **integer overflow** — Python ints are arbitrary precision, but if translating to C++/Java, mind the limits.
- Be prepared to explain **short-circuit evaluation** — especially when it has side effects (e.g., guard patterns).
- **Floor division direction** is a common gotcha: Python floors toward `-∞`, not zero.
- For DP-with-bitmask: think of subsets as ints from `0` to `(1<<n) - 1`.
- Comparison chaining: `a == b == c` means `a == b and b == c` — show off by spotting the bug `a == b == c` when the candidate intended `a == b and a == c`.
- For range/overlap problems: think `max(a, c) <= min(b, d)` (no comparison-chain abuse, but compact).
- Interviewers expect mental fluency: `1 << 20 ≈ 1M`, `1 << 30 ≈ 1B` — powers of two lore.

## 18. Practice Problems

### Easy
1. **Number of 1 Bits** — LeetCode #191 (bit-count via `n & (n-1)`)
2. **Power of Two** — LeetCode #231 (`n & (n-1) == 0`)
3. **Power of Three** — LeetCode #326 (mathematical/logarithmic test)
4. **Single Number** — LeetCode #136 (XOR all elements)
5. **Missing Number** — LeetCode #268 (XOR trick or sum)
6. **Find the Difference** — LeetCode #389 (XOR trick)

### Medium
1. **Single Number II** — LeetCode #137 (bit manipulation state machines)
2. **Single Number III** — LeetCode #260 (XOR + lowbit split)
3. **Divide Two Integers** — LeetCode #29 (bit shifts only, no `*`, `/`, `%`)
4. **Maximum XOR of Two Numbers in an Array** — LeetCode #421 (bitwise trie)

### Hard
1. **Reverse Bits** — LeetCode #190 (sometimes hard-rated variants)
2. **Minimum Number of One Bits to Flip** — LeetCode #1284 (BFS + bit flipping)
3. **Find K-th Smallest Pair Distance** — LeetCode #719 (bitmask counting — sometimes)

---

🔗 Next: [Type Casting](./type_casting.md) · [Variables](./variables.md) · [Input/Output](./input_output.md) · Back to [README](../README.md)
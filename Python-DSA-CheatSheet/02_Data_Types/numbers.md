# Numbers in Python

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
> Section: 02 — Data Types
> 🔗 Related: [strings.md](./strings.md) · [list.md](./list.md) · [tuple.md](./tuple.md)
> Back to [README](../README.md)

---

## 1. What is it?

Python numeric types:

| Type      | Example                  | Notes                                |
|-----------|--------------------------|--------------------------------------|
| `int`     | `5`, `-3`, `10**100`     | arbitrary precision (no overflow)    |
| `float`   | `3.14`, `-0.5`, `1e9`    | IEEE 754 double (64-bit)             |
| `complex` | `3+4j`                   | real + imaginary parts (two floats)  |
| `bool`    | `True` / `False`         | subclass of `int` (1 / 0)            |
| `Decimal` | `Decimal("0.10")`        | exact base-10 (decimal module)       |
| `Fraction`| `Fraction(1,3)`          | exact rational                       |

```python
type(5)           # <class 'int'>
type(3.14)        # <class 'float'>
type(True)        # <class 'bool'>  (subclass of int)
type(3+4j)        # <class 'complex'>
isinstance(True, int)   # True!
True + True       # 2
```

---

## 2. Why do we use it?

- `int`: counting, indices, big-integer math without overflow worry.
- `float`: physical measurements, statistics, division results.
- `complex`: signal processing, electrical engineering.
- `bool`: logic gates / flags.
- `Decimal` / `Fraction`: where exactness matters (money, fractions).

---

## 3. When should I choose it?

| Need                                  | Use               |
|---------------------------------------|--------------------|
| Integer counts / indices              | `int`              |
| Fractional real values                | `float`            |
| Money / exact decimals                | `Decimal`          |
| Exact fractions                       | `Fraction`         |
| Logic flags                           | `bool`             |
| Complex / imaginary math              | `complex`          |
| Bit-level on ints                     | `int` operators    |
| Production heavy math                 | `numpy`            |
| Sequence of values                    | `list` / `tuple`   |
| Text                                  | `str`              |

---

## 4. Syntax

```python
i = 5              # int
j = 0b101           # binary -> 5
k = 0o17            # octal  -> 15
m = 0xFF            # hex    -> 255
n = 10_000_000      # underscores for readability
big = 10 ** 100     # arbitrary precision

f = 3.14
g = 1.5e3           # 1500.0
h = .5              # 0.5
inf = float('inf')  # math.inf
nan = float('nan')  # math.nan

c = 3 + 4j
c.real              # 3.0
c.imag              # 4.0
abs(c)              # 5.0

# Division flavors
7 / 2               # 3.5   float
7 // 2              # 3     floor (int)
7 % 2               # 1     modulo (int)
divmod(7, 2)        # (3, 1)
2 ** 10              # 1024  power
-7 // 2             # -4 (floored, not truncated!)
```

---

## 5. Basic Example

```python
print(10 // 3)            # 3
print(10 % 3)             # 1
print(2 ** 5)             # 32
print(abs(-5))             # 5
print(round(3.567, 2))     # 3.57
print(divmod(17, 5))       # (3, 2)
print(pow(2, 10, 1000))    # 24  (modular pow)
print(int("ff", 16))       # 255
print(bin(255))            # '0b11111111'
print(hex(255))            # '0xff'
print(oct(255))            # '0o377'
```

---

## 6. Step-by-Step Dry Run

**Counting set bits of `n = 23` (`10111`) using Brian Kernighan's algorithm:**

```python
def popcount(n: int) -> int:
    c = 0
    while n:
        n &= n - 1       # clears lowest set bit
        c += 1
    return c

popcount(23)     # 4
```

Dry run `n = 23 = 0b10111`:
```
n=23 (10111)   -> n &= n-1=22 (10110)   => n=22   count=1
n=22 (10110)   -> n &= n-1=21 (10101)   => n=20   count=2
n=20 (10100)   -> n &= n-1=19 (10011)   => n=16   count=3
n=16 (10000)   -> n &= n-1=15 (01111)   => n=0    count=4
loop ends; result = 4
```

Or just `n.bit_count()` (3.10+) — same answer, much faster.

---

## 7. Built-in Methods

### 7.1 int methods

| Method / Attribute       | Description                              |
|--------------------------|------------------------------------------|
| `int.bit_length()`       | number of bits needed to represent abs(n) (excluding sign) |
| `int.bit_count()` (3.10+)| count of 1-bits in binary representation |
| `int.to_bytes(length, byteorder, *, signed=False)` | represent as bytes |
| `int.from_bytes(bytes, byteorder, *, signed=False)` | int from bytes |
| `int.conjugate()`        | returns self (no conjugate for int)      |
| `int.real`, `int.imag`  | `real == self`, `imag == 0` (numeric tower) |

Examples:
```python
(255).bit_length()        # 8
(255).bit_count()         # 8  (3.10+)
(1024).bit_count()        # 1
(-5).bit_count()          # 3   (counts ones in 2-complement repr)
(1024).to_bytes(2, "big")        # b'\x04\x00'
(1024).to_bytes(2, "little")    # b'\x00\x04'
int.from_bytes(b'\xff', 'big')  # 255
```

### 7.2 float methods

| Method              | Description                          |
|---------------------|--------------------------------------|
| `float.is_integer()`| True if value has no fractional part |
| `float.hex()`       | hex string repr (IEEE 754)           |
| `float.fromhex(s)`  | float from hex string                |
| `float.as_integer_ratio()` | (numerator, denominator) of exact fraction |
| `math.modf(x)`      | (fractional, integer) parts (sign-preserving) |

Examples:
```python
(3.0).is_integer()         # True
(3.14).is_integer()        # False
(0.5).as_integer_ratio()  # (1, 2)
0.1.as_integer_ratio()    # (3602879701896397, 36028797018963968)
(3.14).hex()               # '0x1.91eb851eb851fp+1'
float.fromhex('0x1.0p0')   # 1.0
```

### 7.3 complex methods
| Method / Attr         | Description                          |
|-----------------------|--------------------------------------|
| `complex.real`        | real part                            |
| `complex.imag`        | imaginary part                       |
| `complex.conjugate()` | conjugate (replace j with -j)        |
| `abs(c)`              | magnitude = √(real² + imag²)        |

```python
c = 3 + 4j
c.conjugate()    # (3-4j)
abs(c)           # 5.0
```

### 7.4 bool methods
`bool` is a subclass of `int`; uses all `int` methods. Only two instances: `True == 1`, `False == 0`.

```python
True + 5        # 6
sum([True, False, True])   # 2
isinstance(True, int)      # True
```

### 7.5 Built-in numeric functions

| Function        | Use                                |
|-----------------|------------------------------------|
| `abs(x)`        | absolute value / magnitude         |
| `round(x, n=0)`| banker's rounding (round-half-even)|
| `pow(x, y[, m])`| x ** y or (x ** y) % m (modular)  |
| `divmod(a, b)`  | (a // b, a % b)                    |
| `int(x[, base])`| convert to int                     |
| `float(x)`      | convert to float                   |
| `complex(re, im)` | construct complex                |
| `min/max(iterable)`| extremum                         |
| `sum(iter[, start])` | sum                           |
| `bin(n) / oct(n) / hex(n)` | string reprs            |

### 7.6 math module reference

```python
import math
```

| Function                  | Use                                    |
|---------------------------|----------------------------------------|
| `math.gcd(a, b)`          | greatest common divisor               |
| `math.lcm(*ints)`         | least common multiple (3.9+)           |
| `math.isqrt(n)`           | integer square root (floor)           |
| `math.sqrt(x)`            | floating sqrt                          |
| `math.pow(x, y)`          | x ** y as float                       |
| `math.exp(x), math.log(x[, base])` | ex, log                  |
| `math.log2, math.log10`   | log base 2 / 10                        |
| `math.ceil(x), math.floor(x)` | round up/down                      |
| `math.trunc(x)`           | truncate toward zero                   |
| `math.fabs(x)`            | float absolute                         |
| `math.fmod(a, b)`         | float modulo (sign of a)              |
| `math.copysign(x, y)`     | magnitude of x, sign of y              |
| `math.fsum(iter)`         | float sum, no precision loss           |
| `math.prod(iter[, start])`| product (3.8+)                         |
| `math.factorial(n)`       | n!                                     |
| `math.comb(n, k)`         | n choose k (3.8+)                      |
| `math.perm(n, k=None)`    | permutations (3.8+)                    |
| `math.inf`, `math.nan`    | infinity, NaN                          |
| `math.isinf(x), math.isnan(x)` | tests                            |
| `math.isclose(a, b, *, rel_tol=1e-9, abs_tol=0.0)` | tolerance compare |
| `math.pi`, `math.e`, `math.tau` | constants                       |
| `math.sin/cos/tan/atan/atan2/asin/acos` | trigonometry               |
| `math.degrees(rad) / radians(deg)` | degree-radian conversions    |
| `math.hypot(*coords)`     | √(x²+y²+z²+...)                       |
| `math.gammapp` / `math.lgamma` | gamma functions                  |

Examples:
```python
math.gcd(12, 18)            # 6
math.lcm(4, 6)              # 12
math.isqrt(15)              # 3
math.sqrt(2)                # 1.4142135623730951
math.log(1024, 2)           # 10.0
math.ceil(3.01)             # 4
math.floor(3.99)            # 3
math.factorial(5)           # 120
math.comb(5, 2)            # 10
math.perm(5, 2)            # 20
math.isclose(0.1+0.2, 0.3)  # True
math.log2(1024)            # 10.0
math.hypot(3, 4)            # 5.0
```

### 7.7 Bitwise operators on int

| Op      | Meaning          | Example           |
|---------|------------------|-------------------|
| `a & b` | bitwise AND      | `5 & 3` -> `1`    |
| `a \| b`| bitwise OR       | `5 \| 3` -> `7`   |
| `a ^ b` | bitwise XOR      | `5 ^ 3` -> `6`    |
| `~a`    | bitwise NOT      | `~5` -> `-6`      |
| `a << n`| left shift       | `5 << 1` -> `10`  |
| `a >> n`| right shift      | `5 >> 1` -> `2`   |

Common idioms:
- Test bit k: `(n >> k) & 1`
- Set bit k: `n | (1 << k)`
- Clear bit k: `n & ~(1 << k)`
- Toggle bit k: `n ^ (1 << k)`
- Lowest set bit: `n & -n`
- Remove lowest set bit: `n & (n-1)`

---

## 8. Interview Example

**LeetCode 67 – Add Binary (string manipulation + bit logic):**

```python
def addBinary(a: str, b: str) -> str:
    return bin(int(a, 2) + int(b, 2))[2:]
```

**LeetCode 50 – Pow(x, n) via fast exponentiation:**
```python
def myPow(x: float, n: int) -> float:
    if n < 0: return 1 / myPow(x, -n)
    res = 1.0
    while n:
        if n & 1: res *= x
        x *= x
        n >>= 1
    return res
```

**Modular combinations (nCk mod p):**
```python
from math import comb
comb(40, 20)        # 137846528820
pow(7, 100, 1009)   # fast modular pow
```

---

## 9. When NOT to use

- Need exact decimal money → `Decimal` (NOT `float`).
- Need exact rational arithmetic → `Fraction`.
- Heavy array math → `numpy`.
- Avoid using `float` as dict keys (technically allowed but fragile: `nan` ≠ `nan`.
- Don't compare floats with `==` — use `math.isclose`.

---

## 10. Common Mistakes

```python
# 1. Float precision
0.1 + 0.2 == 0.3          # False  (!)
(0.1 + 0.2)               # 0.30000000000000004
# Fix: use Decimal or math.isclose

# 2. Floor vs trunc on negatives
7 // 2                   # 3
-7 // 2                  # -4  (floored, not -3!)
int(-3.7)                # -3  (truncates toward 0)

# 3. round is "banker's rounding"
round(0.5)               # 0   (!)
round(1.5)               # 2
round(2.5)               # 2   (round half to even)
round(3.5)               # 4

# 4. bool is int
sum([True, False, True]) # 2
len([x for x in xs if x])  # better than sum(1 for x in xs if x) ... but bool-as-int is normal

# 5. Mixing Decimal and float
Decimal("0.1") + 0.1     # TypeError
# Fix: stick to Decimal end-to-end

# 6. is check on integer caching
a, b = 256, 256
a is b                    # True (small int cache)
a, b = 1000, 1000
a is b                    # depends; may be True or False

# 7. / vs // vs %
7 / 2                     # 3.5
7 // 2                    # 3
7.0 // 2                  # 3.0

# 8. Using lists as bool truthy truth-tests
bool([])                  # False
bool([0])                 # True  (non-empty -> truthy)

# 9. math.isnan comparisons
float('nan') == float('nan')   # False! use math.isnan
```

---

## 11. Memory Tricks

- **int = unlimited** — no overflow in Python.
- **float = 64-bit IEEE** — same precision as JavaScript number; watch negatives (`-7 // 2 = -4`).
- **bool is int's tiny sibling** — `True + True = 2`.
- **bank round** — `round` uses banker's rounding (round half to **even**).
- **Brian Kernighan** — `n & (n-1)` kills the lowest set bit (popcount trick).
- **`gcd`, `lcm`, `comb`, `perm`** live in `math` — don't reinvent.

---

## 12. Interview Shortcuts

- `math.comb(n, k)` for combinations (3.8+).
- `math.gcd(a, b)` & `math.lcm(*xs)` for number theory.
- `pow(a, b, MOD)` is fast (uses binary exponentiation internally).
- `x.bit_length()` to find storage bits (e.g., `1 << n` fits in `n.bit_length()`).
- `x.bit_count()` for popcount (3.10+) — replaces manual Brian Kernighan loop.
- `math.isclose(a, b)` for float equality.
- `math.isqrt(n)` for integer sqrt — avoids float drift.
- `int(s, base)` parses binary/octal/hex; `bin/oct/hex` for the reverse.
- For modular inverses: `pow(a, MOD-2, MOD)` (Fermat's little theorem, MOD prime).

---

## 13. Cheat Sheet Table

| Method / Func             | Use                              |
|---------------------------|----------------------------------|
| `int.bit_length()`        | bits to represent int            |
| `int.bit_count()`         | number of 1-bits                 |
| `int.to_bytes/from_bytes` | bytes <-> int                    |
| `float.is_integer()`      | int test                         |
| `float.as_integer_ratio()`| exact fraction                   |
| `complex.real/imag`       | parts                            |
| `complex.conjugate()`     | conjugate                        |
| `abs(x)`                  | absolute / magnitude             |
| `round(x, n)`             | banker's rounding                |
| `pow(x, y, m)`            | modular exponentiation           |
| `divmod(a, b)`            | (quotient, remainder)            |
| `math.gcd / lcm`          | gcd / lcm                        |
| `math.isqrt`              | integer sqrt                     |
| `math.sqrt`               | float sqrt                       |
| `math.log, log2, log10`   | logarithms                       |
| `math.ceil, floor, trunc` | rounding                          |
| `math.factorial`          | n!                               |
| `math.comb, perm`         | nCk, nPk                         |
| `math.fsum`               | precision-preserving sum         |
| `math.prod`               | product                          |
| `math.isclose`            | tolerance equality               |
| `math.inf, nan`           | infinities and NaN               |
| `math.isinf, isnan`       | tests                            |
| `math.pi, e, tau`         | constants                        |
| bit ops: `& \| ^ ~ << >>` | bitwise                          |

---

## 14. Time Complexity Table

| Operation                     | Complexity       | Notes                          |
|-------------------------------|------------------|--------------------------------|
| `+ - * // %` (small ints)     | O(1)             | < 2^63                         |
| `<op>` on big ints            | O(n)             | n = number of digits           |
| `int(n) / float(n)`           | O(1)             | small; O(n) for giant strings  |
| `x.bit_length()`              | O(1)             | cached                         |
| `x.bit_count()`               | O(b)             | b = bits                       |
| `x.to_bytes/from_bytes`       | O(b)             | b = bytes                      |
| `pow(x, y, m)` (modular)      | O(log y)         | binary exponentiation          |
| `pow(x, y)` (no mod)          | O(M(n) * log y)  | grows with size                |
| `abs`, `int.real/imag`        | O(1)             |                                |
| `math.gcd(a, b)`              | O(log min(a,b))  | Euclid                         |
| `math.isqrt(n)`               | O(M(n))           | uses Newton                    |
| `math.comb(n, k)`             | O(k)             |                                |
| `math.factorial(n)`           | O(n * M(n))       | bit complexity larger          |
| `math.log/sqrt/sin`           | O(1)             | hardware approx                |

(M(n) — cost of multiplying n-bit numbers.)

---

## 15. Visual Diagram (ASCII)

### Int range concept (unbounded)
```
   ... -3 -2 -1  0  1  2  3 ...            Python int (unbounded)
       ^^^^^^^^^^^^^^^^^^^^^
       arbitrary precision (no overflow!)
       only limit: RAM

   big = 10 ** 100   -> works, prints full decimal
```

### Float precision (IEEE 754 double)
```
float64 layout (64 bits):
   1 sign | 11 exponent | 52 mantissa
     s eeeeeeeeeee mmmmmmmm...52

Approx range:  ±1.8e308
Precision:     ~15-17 decimal digits

Spacing grows with magnitude:
   0.0       1e-300      |gap| -> tiny
   1.0                  |gap| -> 2^-52 ~ 2.2e-16
   1e300                |gap| -> huge

0.1 + 0.2 == 0.3   → False  (0.30000000000000004)
                  → use math.isclose or Decimal
```

### Brian Kernighan popcount flow
```
n = 23 = 0b10111
+----+
|1|0|1|1|1|        iter1:  n&=n-1 (clear lowest set bit)
+----+            n=22 (10110)  c=1
killing lowest:    iter2:  n=22 &(21=10101) = 20 (10100)  c=2
                   iter3:  n=20 &(19=10011) = 16 (10000)  c=3
                   iter4:  n=16 &(15=01111) =  0           c=4
                   return 4
```

### Banker's rounding visualization
```
value:  ... 1.5  2.5  3.5  4.5 ...
round:  ...  2   2   4   4   ...
              ^--- round half to EVEN neighbor

(x.5 -> nearest even integer)
```

### Float to int conversions differ
```
 3.7
   | floor  -> 3   (down)
   | ceil   -> 4   (up)
   | trunc  -> 3   (toward zero)
   | round  -> 4   (banker's)

-3.7
   | floor  -> -4
   | ceil   -> -3
   | trunc  -> -3
   | round  -> -4
```

### Bit operations on int
```
        5  = 0b 0 0 1 0 1
        3  = 0b 0 0 0 1 1
   5 & 3  = 0b 0 0 0 0 1   -> 1
   5 | 3  = 0b 0 0 1 1 1   -> 7
   5 ^ 3  = 0b 0 0 1 1 0  -> 6
   5 << 1 = 0b 0 1 0 1 0  -> 10
   5 >> 1 = 0b 0 0 0 1 0  -> 2
       ~5 = -6 (2's complement)
```

---

## 16. Beginner Notes

> Remember:
> - `int` in Python has **unlimited** size — no overflow.
> - `float` is IEEE 754 double — slow precision drift; `0.1 + 0.2 != 0.3`.
> - `bool` is a subclass of `int`: `True == 1`, `False == 0`.
> - `//` (floor) rounds **toward negative infinity**; `int()` truncates toward zero.
> - `round()` uses **banker's rounding** (round half to even).
> - Use `math.isclose` for float equality.
> - `pow(a, b, MOD)` is fast; `pow(a, -1, MOD)` gives modular inverse (3.8+).
> - Test bit k with `(n >> k) & 1`; popcount with `n.bit_count()` (3.10+).
> - Module companion: `from math import gcd, lcm, isqrt, comb, perm, factorial`.

---

## 17. FAANG Tips

- **Number theory**: `math.gcd`, `math.lcm`, `pow(a, b, MOD)`, `math.comb`, modular inverse via `pow(a, MOD-2, MOD)`.
- **Counting / bit problems**: prefer `bit_count()` over hand-rolled loops.
- **Range sum**: use `itertools.accumulate` (O(n) prefix) or Fenwick/Segment tree for O(log n) updates.
- **Floating**: use `math.isclose` or convert to `Decimal` for money; use `int` for ratios.
- **Probability / DP**: build a "prefix product mod p" with `pow`-based inverse — beats few-shot division.
- **Big integer output**: Python prints full decimal; for pretty bytes use `to_bytes(...).hex()`.
- **Iterate bits**: `while n: b = n & -n; ...; n ^= b` to walk set bits from low to high.

---

## 18. Practice Problems

**Easy**
1. LeetCode 67 – Add Binary
2. LeetCode 191 – Number of 1 Bits (use `bit_count`)
3. LeetCode 231 – Power of Two (`n and not (n & n-1)`)
4. LeetCode 326 – Power of Three

**Medium**
5. LeetCode 50 – Pow(x, n) (fast exp)
6. LeetCode 372 – Super Pow (modular)
7. LeetCode 204 – Count Primes (Sieve of Eratosthenes)
8. LeetCode 69 – Sqrt(x) (binary search vs `isqrt`)

**Hard**
9. LeetCode 1492 – The kth Factor of n
10. LeetCode 866 – Prime Palindrome
11. LeetCode 3120 – Count Pruning (bit DP if applicable)

---

> Back to [README](../README.md) · Previous: [tuple.md](./tuple.md) · [list.md](./list.md) · [strings.md](./strings.md)
# Strings in Python

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
> Section: 02 — Data Types
> 🔗 Related: [list.md](./list.md) · [tuple.md](./tuple.md) · [numbers.md](./numbers.md)
> Algorithms: [two_pointers.md](../07_Algorithms/two_pointers.md) · [sliding_window.md](../07_Algorithms/sliding_window.md)
> Back to [README](../README.md)

---

## 1. What is it?

A **string** is an **immutable sequence of Unicode characters** enclosed in single (`'`), double (`"`), or triple quotes (`'''` / `"""` for multi-line).

```python
s1 = 'hello'
s2 = "world"
s3 = """multi
line"""
```

Key idea: strings are **immutable** — every "modification" creates a **new string object**. Under the hood, Python uses a compact ASCII / UCS-1, UCS-2, or UCS-4 representation (PEP 393).

---

## 2. Why do we use it?

- Represent **textual data** (names, logs, JSON, source code, DNA, tokens).
- Natural input/output format for files, network, and humans.
- Rich set of methods make parsing / cleaning / formatting trivial.
- Python's string handling is a **first-class interview topic** — FAANG loves string problems (palindromes, anagrams, sliding window, regex).

---

## 3. When should I choose it?

| Need                              | Use          | Why                                |
|-----------------------------------|--------------|------------------------------------|
| Text / characters                 | `str`        | native, immutable, rich methods    |
| Mutable char buffer              | `list[char]` | strings are immutable              |
| Sequence of fixed records         | `tuple`      | hashable, lightweight              |
| Growing collection                | `list`       | mutable, dynamic                    |
| Numeric value                     | `int/float`  | arithmetic ops                      |
| Key-value mapping                | `dict`       | O(1) lookup                         |
| Unique items                     | `set`        | O(1) membership                     |
| Byte data (binary)               | `bytes`      | raw octets, not text                |
| Decimal money                    | `Decimal`    | exact base-10 arithmetic            |

---

## 4. Syntax

```python
s = 'abc'                          # single quotes
s = "abc"                          # double quotes (same)
s = """abc
def"""                             # multi-line
s = b'abc'                         # bytes
s = r'C:\path\n'                   # raw string (escapes disabled)
s = f'{name}!'                     # f-string (3.6+)
s = 'a' 'b' 'c'                    # implicit concat -> 'abc'
s = "a" * 3                        # repetition -> "aaa"
s[0], s[-1], s[1:3], s[::-1]       # indexing & slicing
```

---

## 5. Basic Example

```python
name = "Tamilselvan"
print(name[0])              # T
print(name[-1])             # n
print(name[0:5])            # Tamil
print(name.upper())         # TAMILSELVAN
print(name.lower())         # tamilselvan
print(len(name))            # 11
print("a" in name)          # True
```

---

## 6. Step-by-Step Dry Run

Problem: check if a string is a palindrome **ignoring case & non-alphanumerics**.

```python
def is_palindrome(s: str) -> bool:
    cleaned = ''.join(ch.lower() for ch in s if ch.isalnum())
    left, right = 0, len(cleaned) - 1
    while left < right:
        if cleaned[left] != cleaned[right]:
            return False
        left += 1
        right -= 1
    return True

print(is_palindrome("A man, a plan, a canal: Panama"))   # True
```

Dry run on `"Aba"`:

```
Step 1  s = "Aba"
Step 2  cleaned = "aba"   (lowercased, alnum filtered)
Step 3  left=0 right=2
Step 4  cleaned[0]='a' == cleaned[2]='a'  -> left=1 right=1
Step 5  left < right False -> exit
Step 6  return True
```

---

## 7. Built-in Methods   (EXHAUSTIVE)

> Format for each: **Purpose**, **Syntax**, **Input**, **Output**, **Example**, **Time**, **Interview use**, **Mistakes**, **Shortcut**.

### 7.1 `str.upper()`
- Purpose: return copy with all cased chars uppercased.
- Syntax: `s.upper()`
- Input: none
- Output: new `str`
- Example:
  ```python
  "Abc".upper()        # 'ABC'
  ```
- Time: **O(n)**
- Interview use: case-insensitive comparison.
- Mistakes: returns a new string (doesn't mutate).
- Shortcut: `s.casefold()` for stricter Unicode lowercasing.

### 7.2 `str.lower()` / `str.casefold()`
- Purpose: lowercase; `casefold` handles German ß etc.
- Example: `"Straße".casefold()` -> `'strasse'`
- Time: O(n)
- Shortcut: prefer `casefold()` for Unicode equality.

### 7.3 `str.replace(old, new, count=-1)`
- Purpose: replace occurrences of substring.
- Syntax: `s.replace(old, new[, count])`
- Example:
  ```python
  "a-b-c".replace("-", "_")       # 'a_b_c'
  "aaa".replace("a", "z", 2)      # 'zza'
  ```
- Time: O(n)
- Interview use: normalize input, remove chars.
- Mistakes: `replace` does NOT take regex; chains return new strings.
- Shortcut: `s.replace(' ', '')` to remove all spaces.

### 7.4 `str.split(sep=None, maxsplit=-1)`
- Purpose: split string into list.
- Example:
  ```python
  "a,b,c".split(",")              # ['a','b','c']
  "  a  b ".split()               # ['a','b']   (whitespace collapsing)
  "a-b-c".split("-", 1)           # ['a','b-c']
  ```
- Time: O(n)
- Interview use: parse CSV / log lines.
- Mistakes: `split(' ')` keeps empty fields; `split()` collapses.
- Shortcut: `s.split()` for whitespace tokenize.

### 7.5 `str.splitlines(keepends=False)`
- Purpose: split on line boundaries (`\n`, `\r\n`, `\r`...).
- Example: `"a\nb".splitlines()` -> `['a','b']`
- Time: O(n)

### 7.6 `str.rsplit(sep=None, maxsplit=-1)`
- Purpose: split from the right; maxsplit affects from right.
- Example: `"a.b.c".rsplit('.', 1)` -> `['a.b','c']`
- Time: O(n)

### 7.7 `str.join(iterable)`
- Purpose: glue items with separator.
- Syntax: `sep.join(iterable)`
- Example:
  ```python
  ",".join(['a','b','c'])         # 'a,b,c'
  "".join(['a','b','c'])           # 'abc'
  ```
- Time: O(n)
- Interview use: **the** efficient builder (vs `+` in loops).
- Mistakes: items must be strings; pass list of ints -> TypeError.
- Shortcut: `"".join(list_of_strings)` is fastest builder.

### 7.8 `str.startswith(prefix[, start[, end]])` / `str.endswith(suffix[, start[, end]])`
- Purpose: test prefix/suffix; supports tuple.
- Example:
  ```python
  "abc.mp4".endswith((".mp4", ".avi"))   # True
  ```
- Time: O(k) where k = len(prefix)
- Interview use: quick filename/ext checks.
- Mistakes: tuple only (not list).

### 7.9 `str.strip([chars])` / `str.lstrip([chars])` / `str.rstrip([chars])`
- Purpose: remove leading/trailing chars (whitespace by default).
- Example:
  ```python
  "  hi  ".strip()        # 'hi'
  "xxyhixx".strip("x")   # 'yhi'
  ```
- Time: O(n)
- Interview use: clean user input.
- Mistakes: strips **set** of chars, not the prefix string.
- Shortcut: `s.strip()` for default whitespace.

### 7.10 `str.find(sub[, start[, end]])` / `str.rfind(sub)`
- Purpose: return lowest index of sub, -1 if not found.
- Example: `"abcabc".find("b")` -> `1`
- Time: O(n) avg, O(n*m) worst
- Interview use: locate substr without exceptions.
- Mistakes: returns -1, not error; use `index()` to raise.
- Shortcut: `s.find(x) == -1` checks absence.

### 7.11 `str.rfind(sub)` — rightmost index; same semantics.

### 7.12 `str.index(sub[, start[, end]])` / `str.rindex(sub)`
- Purpose: like `find`, but raises `ValueError`.
- Example: `"abc".index("z")` -> ValueError
- Time: O(n)
- Mistakes: don't catch in hot loops — prefer `find`.

### 7.13 `str.count(sub[, start[, end]])`
- Purpose: non-overlapping occurrences.
- Example: `"aaaa".count("aa")` -> `2`
- Time: O(n)
- Interview use: count chars / substrings.
- Mistakes: counts non-overlapping only.

### 7.14 `str.capitalize()`
- Purpose: first char upper, rest lower.
- Example: `"hELLO wORLD".capitalize()` -> `'Hello world'`

### 7.15 `str.title()`
- Purpose: title-case each word.
- Example: `"hello world".title()` -> `'Hello World'`
- Mistakes: `"o'clock".title()` -> `"O'Clock"` (may misfire).

### 7.16 `str.istitle()`
- Purpose: True if uppercase follows each cased char.
- Example: `"Hello".istitle()` -> `True`

### 7.17 `str.swapcase()`
- Purpose: swap case of all chars.
- Example: `"AbC".swapcase()` -> `'aBc'`

### 7.18 `str.isalpha()`
- Purpose: True if non-empty and all chars are letters.
- Example: `"Hello".isalpha()` -> `True`; `"Hello1".isalpha()` -> `False`
- Interview use: validate names / words.

### 7.19 `str.isdigit()`
- Purpose: True if all chars are digits (includes superscripts etc).
- Example: `"123".isdigit()` -> `True`

### 7.20 `str.isnumeric()`
- Purpose: True if all chars have Unicode numeric property.
- Example: `"½".isnumeric()` -> `True`

### 7.21 `str.isdecimal()`
- Purpose: strictest — chars are decimal digits only.
- Example: `"½".isdecimal()` -> `False`; `"123".isdecimal()` -> `True`
- Mistakes: `isdigit > isnumeric > isdecimal` in coverage.

### 7.22 `str.isalnum()`
- Purpose: True if all chars alphanumeric (letters or digits).
- Example: `"abc123".isalnum()` -> `True`

### 7.23 `str.isidentifier()`
- Purpose: True if valid Python identifier.
- Example: `"my_var".isidentifier()` -> `True`; `"9abc".isidentifier()` -> `False`

### 7.24 `str.isprintable()`
- Purpose: True if no non-printing chars (e.g. `\n`).
- Example: `"abc".isprintable()` -> `True`; `"a\n".isprintable()` -> `False`

### 7.25 `str.isspace()`
- Purpose: True if all chars whitespace.
- Example: `" \t\n".isspace()` -> `True`

### 7.26 `str.islower()` / `str.isupper()`
- Purpose: True if cased chars all lower/upper.
- Example: `"abc".islower()` -> `True`; `"ABC".isupper()` -> `True`

### 7.27 `str.center(width[, fillchar])`
- Purpose: center string within width.
- Example: `"hi".center(6, "-")` -> `'--hi--'`

### 7.28 `str.ljust(width[, fillchar])` / `str.rjust(width[, fillchar])`
- Purpose: left / right justify within width.
- Example: `"hi".ljust(5, ".")` -> `'hi...'`; `"hi".rjust(5, ".")` -> `'...hi'`
- Interview use: pretty printing tables.

### 7.29 `str.partition(sep)` / `str.rpartition(sep)`
- Purpose: split at FIRST/LAST occurrence -> `(head, sep, tail)`.
- Example: `"a-b-c".partition("-")` -> `('a','-','b-c')`
- Time: O(n)

### 7.30 `str.zfill(width)`
- Purpose: left-pad with zeros.
- Example: `"42".zfill(5)` -> `'00042'`
- Interview use: fixed-width numeric ids.

### 7.31 `str.format(*args, **kwargs)`
- Positional + named placeholders.
- Example:
  ```python
  "{0}-{1}-{0}".format("a", "b")           # 'a-b-a'
  "{name}:{age}".format(name="T", age=10)  # 'T:10'
  "{:>5}".format(42)                        # '   42'
  "{:.2f}".format(3.14159)                  # '3.14'
  ```
- Interview use: formatting output.

### 7.32 `str.format_map(mapping)`
- Like `.format(**mapping)` but accepts a dict-like without copying.
- Example: `"{a}{b}".format_map({"a":1,"b":2})` -> `'12'`

### 7.33 f-strings (3.6+)
- Example:
  ```python
  name = "Tamil"
  f"Hello {name.upper()}!"          # 'Hello Tamil!'
  f"{3.14159:.2f}"                  # '3.14'
  f"{42:>5}"                        # '   42'
  f"{42:08b}"                       # '00101010' (binary)
  ```
- Most readable, fastest formatting in Python.

### 7.34 %-formatting (old style)
- Example: `"%s has %d" % ("Tamil", 10)` -> `'Tamil has 10'`
- Legacy — prefer f-strings.

### 7.35 `str.removeprefix(prefix)` / `str.removesuffix(suffix)` (3.9+)
- Purpose: remove prefix/suffix only if present (unlike `strip`).
- Example:
  ```python
  "abc.txt".removeprefix("abc")    # '.txt'
  "abc.txt".removesuffix(".txt")   # 'abc'
  ```
- Mistakes: `strip` ≠ `removeprefix`; `strip("ab")` removes set.

### 7.36 `str.expandtabs(tabsize=8)`
- Purpose: replace tabs with spaces.
- Example: `"a\tb".expandtabs(4)` -> `'a   b'`

### 7.37 `str.encode(encoding='utf-8', errors='strict')`
- Purpose: encode to `bytes`.
- Example: `"abc".encode()` -> `b'abc'`
- Interview use: network / file I/O.

### 7.38 `str.maketrans(x[, y[, z]])` + `str.translate(table)`
- Purpose: build & apply char translation table.
- Example:
  ```python
  t = str.maketrans("abc", "123")
  "cab".translate(t)               # '312'
  t2 = str.maketrans('', '', 'aeiou')  # delete vowels
  "hello".translate(t2)            # 'hll'
  ```
- Time: O(n)
- Interview use: **fast O(n) char deletion / remap** — beats `.replace` chains.

---

### 7.39 String Formatting Cheat Sheet

| Style       | Example                            | Use                       |
|-------------|------------------------------------|---------------------------|
| f-string    | `f"{x=}"`                          | default (3.6+)            |
| `.format()` | `"{:>5}".format(x)`                | dynamic templates         |
| `%`         | `"%d" % x`                         | legacy                    |
| Template    | `from string import Template`      | user-supplied safe format |

Format mini-language: `[[fill]align][sign][#][0][width][,][.precision][type]`
- align: `<` left, `>` right, `^` center, `=` pad after sign
- type: `s d f e b o x X %`

```python
f"{255:x}"        # 'ff'
f"{255:#08x}"     # '0x0000ff'
f"{1234567:,}"    # '1,234,567'
f"{0.15:+.2%}"    # '+15.00%'
```

---

### 7.40 Regex Basics (`import re`)

```python
import re
re.match(r"\d+", "123abc")     # <Match> from start
re.search(r"\d+", "ab123cd")    # <Match> first anywhere
re.findall(r"\d+", "a1b22c333") # ['1','22','333']
re.sub(r"\d", "#", "a1b2")      # 'a#b#'
re.split(r"[,;]", "a,b;c")      # ['a','b','c']
re.finditer(r"\d", "a1b2")      # iterator of matches
```

**Character classes**
- `.` any char (except newline), `\d` digit, `\D` non-digit, `\w` word `[A-Za-z0-9_]`, `\W`, `\s` whitespace, `\S`
- `[abc]` set, `[^abc]` negated, `[a-z]` range

**Anchors**
- `^` start of string, `$` end, `\b` word boundary, `\B`

**Quantifiers**
- `*` 0+, `+` 1+, `?` 0/1, `{n}` exactly n, `{n,m}` n..m
- lazy: `*?`, `+?`, `??`

**Groups**
- `(...)` capture, `(?:...)` non-capture, `(?P<name>...)` named, `\1 \2` backrefs

**Flags**
- `re.I` ignore case, `re.M` multiline, `re.S` dotall, `re.X` verbose

```python
m = re.match(r"(?P<year>\d{4})-(?P<month>\d{2})", "2024-07")
m.group("year")            # '2024'
```

Time: `match` / `search` O(n*m) worst, `findall` builds list O(matches).

---

### 7.41 ASCII Table (printable chars)

| Range   | Chars                       | Codes       |
|---------|-----------------------------|-------------|
| A–Z     | uppercase letters           | 65–90       |
| a–z     | lowercase letters           | 97–122      |
| 0–9     | digits                      | 48–57       |
| space   | ` `                         | 32          |
| special | `! " # $ % & ' ( ) * + , - . /` | 33–47   |
| special | `: ; < = > ? @`             | 58–64       |
| special | `[ \ ] ^ _ \``              | 91–96       |
| special | `{ | } ~`                   | 123–126     |

Quick relations:
- `ord('A') == 65`, `chr(65) == 'A'`
- lowercase = uppercase + 32 (`ord('a') - ord('A') == 32`)
- digit char -> int: `ord(c) - ord('0')`

### 7.42 Unicode Basics
- Python 3 strings are Unicode (PEP 393 compact storage).
- `ord('α') == 945`, `chr(945) == 'α'`
- Code points U+0000..U+10FFFF; UTF-8 encodes a code point to 1–4 bytes.
- `len("α") == 1` (one code point) but `len("α".encode()) == 2` (two bytes).
- Combining chars / grapheme clusters may surprise: `len("é")` (precomposed) = 1, but `len("e\u0301")` = 2.
- Normalize with `unicodedata.normalize('NFC', s)`.

---

## 8. Interview Example

**Anagram check (242. Valid Anagram)**

```python
def isAnagram(s: str, t: str) -> bool:
    if len(s) != len(t):
        return False
    from collections import Counter
    return Counter(s) == Counter(t)
```

Alternative O(1) space:
```python
def isAnagram(s, t):
    if len(s) != len(t): return False
    cnt = [0]*26
    for c in s:
        cnt[ord(c)-97] += 1
    for c in t:
        cnt[ord(c)-97] -= 1
    return not any(cnt)
```

---

## 9. When NOT to use

- Building a large string with `+` in a loop → use `"".join(...)`.
- Need mutable char buffer → use `list[str]` then `join`.
- Storing raw binary → use `bytes` / `bytearray`.
- Money / exact decimals → `decimal.Decimal`.
- Tiny "string keys" reused everywhere → consider interning or just use `str` (auto-interns identifiers).

---

## 10. Common Mistakes

```python
# 1. Mutating a string
s = "abc"
s[0] = "A"          # TypeError: strings immutable

# 2. strip removes a SET of chars, not a prefix
"abc".strip("abc")        # '' (all chars in set stripped)
"abc".removeprefix("abc") # ''

# 3. split(' ') vs split()
"a  b".split(" ")   # ['a', '', 'b']
"a  b".split()      # ['a', 'b']

# 4. + in a loop is O(n^2)
s = ""
for c in "abc": s += c   # copies every iter -> use join

# 5. find returning -1 vs index raising
s = "abc"; s.index("z")  # ValueError

# 6. count is non-overlapping
"aaaa".count("aa")       # 2  not 3

# 7. isdigit accepts superscript chars
"²".isdigit()             # True

# 8. endswith with list (not tuple)
"abc.mp4".endswith([".mp4"])   # TypeError -> use tuple
```

---

## 11. Memory Tricks

- **Strings are ice** — frozen sequences; melt into a `list` to mutate, refreeze with `""`.
- **join ≠ +** — `join` is the *factory*; `+` is slow handwork.
- **strip is scissors** — cuts any char from the set at both ends.
- **find = quiet**, **index = noisy** (raises).
- **start/endswith tuples** = OR check.

---

## 12. Interview Shortcuts

- `s[::-1]` reverse — idiomatic palindrome base.
- `s.isalnum()` + `s.lower()` clean for two-pointer.
- `Counter(s)` for anagram / frequency.
- `ord(c) - ord('a')` → 0..25 array index.
- `''.join(...)` for any builder.
- `re.findall(r'\w+', s)` for tokenization.
- `f"{x:0>5}"` zero/right pad.
- Compare case-insensitively with `.casefold()` not `.lower()`.

---

## 13. Cheat Sheet Table

| Method            | Use                                         |
|-------------------|----------------------------------------------|
| upper / lower     | case conversion                             |
| casefold          | aggressive lowercase (Unicode)              |
| replace           | substring substitute                        |
| split / rsplit    | string -> list                              |
| splitlines        | split on newlines                           |
| join              | list -> string                              |
| strip / l/rstrip  | trim chars                                  |
| startswith / endswith | prefix/suffix test                       |
| find / rfind      | substring index (-1 if not found)           |
| index / rindex    | same but ValueError                         |
| count             | non-overlapping occurrences                 |
| capitalize / title / swapcase | case transforms                 |
| isalpha/digit/alnum/... | classification                       |
| center / ljust / rjust / zfill | padding                          |
| partition / rpartition | 3-tuple split                          |
| format / format_map / f-string | templating                      |
| removeprefix / removesuffix | trim prefix/suffix only          |
| encode            | str -> bytes                                |
| maketrans / translate | fast char map / delete                    |
| expandtabs        | tabs -> spaces                              |

---

## 14. Time Complexity Table

| Operation                  | Complexity     | Notes                      |
|----------------------------|----------------|----------------------------|
| Index `s[i]`               | O(1)           |                            |
| Slice `s[a:b]`             | O(k)           | k = slice length           |
| `len(s)`                   | O(1)           | cached                     |
| `in` / `find` / `index`    | O(n) avg       | O(n*m) worst naive         |
| `count`                    | O(n)           | non-overlapping            |
| Concatenation `s + t`      | O(len(s)+len(t)) | new object                |
| `join`                     | O(n)           | preferred builder          |
| `s * k`                    | O(k*len(s))    |                            |
| `replace`                  | O(n)           |                            |
| `split` / `partition`      | O(n)           |                            |
| `strip` / `ljust`          | O(n)           |                            |
| `upper` / lower / etc.     | O(n)           |                            |
| `sorted(s)`                | O(n log n)     |                            |
| `re.match/search`          | O(n*m) worst   | often O(n)                 |

---

## 15. Visual Diagram (ASCII)

### A string as indexed characters
```
Index (pos):  0   1   2   3   4
             +---+---+---+---+---+
Chars:       | H | e | l | l | o |
             +---+---+---+---+---+
Neg index:   -5  -4  -3  -2  -1

Slices:
  s[1:4]   -> "ell"      (start incl, stop excl)
  s[::-1]  -> "olleH"    (reversal)
  s[::2]   -> "Hlo"      (every 2nd char)
```

### Memory model (immutability)
```
s = "abc"
t = s.replace("b", "B")
     ┌─────┐           ┌─────┐
s -> │ abc │   (id A)  │ aBc │  (id B) <- t
     └─────┘           └─────┘
   (original untouched)
```

### Two-pointer palindrome flow
```
 cleaned = "racecar"
  L                 R
  r a c e c a r
  ^             ^    match -> L++ R--
    ^         ^      match -> L++ R--
      ^     ^        match -> L++ R--
        ^ ^          L>=R -> palindrome TRUE
```

---

## 16. Beginner Notes

> Remember:
> - Strings are **immutable** — operations return **new** strings.
> - `s[0]`, `s[-1]`, `s[a:b:c]`, `s[::-1]` cover most indexing needs.
> - `"".join(list)` is the **fastest** way to build a string.
> - Use `s.casefold()` for case-insensitive equality.
> - `strip` removes a **set** of chars; `removeprefix`/`removesuffix` remove a **literal** prefix/suffix.
> - Regex via `re.match/search/findall/sub/split`; compile patterns reused in loops.
> - ASCII: A-Z = 65-90, a-z = 97-122, 0-9 = 48-57.

---

## 17. FAANG Tips

- For **two-pointer** palindrome/isomorphic style: clean with `isalnum()` + `casefold()`.
- For **sliding window**: when expanding needs `s[i:j+1]`, prefer tracking char counts in an array/dict instead of slicing (slice is O(k)).
- For **anagram / frequency** problems: `Counter(s)` is fastest to write; `[0]*26` is fastest to run.
- For **string matching** on huge texts: pre-compile regex or use KMP / Z-algorithm; Python's `str.find` uses a hybrid (Crochemore-Perrin Two-Way), very fast for typical cases.
- For **builder**: `io.StringIO` can beat `"".join` when streaming many appends; otherwise `join` wins.
- For **formatting large reports**: prefer f-strings (fastest).
- `translate(maketrans('','',chars))` is the fastest way to delete a set of chars → beats `replace` chains.
- Encoding pain points: always `.encode('utf-8')` for I/O; use `errors='replace'` / `'ignore'` defensively.

---

## 18. Practice Problems

**Easy**
1. LeetCode 344 – Reverse String (swap)
2. LeetCode 541 – Reverse String II
3. LeetCode 125 – Valid Palindrome
4. LeetCode 242 – Valid Anagram
5. LeetCode 392 – Is Subsequence

**Medium**
6. LeetCode 3 – Longest Substring Without Repeating Characters (sliding window)
7. LeetCode 5 – Longest Palindromic Substring (expand around center)
8. LeetCode 49 – Group Anagrams
9. LeetCode 767 – Reorganize String
10. LeetCode 8 – String to Integer (atoi)

**Hard**
11. LeetCode 76 – Minimum Window Substring
12. LeetCode 10 – Regular Expression Matching (DP)
13. LeetCode 214 – Shortest Palindrome (KMP)

---

> Next: [list.md](./list.md) · [tuple.md](./tuple.md) · [numbers.md](./numbers.md) · Back to [README](../README.md)
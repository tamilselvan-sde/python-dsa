# Input / Output in Python

> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com
> Section: 01 — Python Basics
> 🔗 Related: [variables](./variables.md) · [type casting](./type_casting.md) · [strings](../02_Data_Types/strings.md) · Back to [README](../README.md)

## 1. What is it?

Input/Output (I/O in Python) is how your program **receives data** from the user (or a file, or stdin) and **sends data** back (to the screen, a file, or stdout). The two most common built-in functions are:

- `print(...)` — **output** to standard output (`stdout`)
- `input(...)` — **input** from standard input (`stdin`)

Python also gives you f-strings, `sep`/`end` parameters, formatting helpers, and `sys.stdin` for fast bulk reads.

## 2. Why do we use it?

- To **show results** to the user.
- To **collect** data from the user at runtime.
- To **format** output cleanly (decimals, padding, alignment).
- To **read large inputs efficiently** — important in competitive programming and LeetCode-style problems.
- To **debug** by printing intermediate values.

## 3. When should I choose it?

| Situation                            | Use                              | Example                       |
|-------------------------------------|----------------------------------|-------------------------------|
| Print a simple message              | `print()`                        | `print("hi")`                 |
| Print multiple items separated      | `print(*, sep=...)`              | `print(1,2,3, sep=", ")`      |
| Print without newline               | `print(end="")`                  | `print("x", end="")`          |
| Take one line of user input         | `input()`                        | `name = input()`              |
| Take input as a number               | `int(input())` / `float(input())`| `n = int(input())`           |
| Format value into text               | f-strings / `.format()`          | `f"{pi:.2f}"`                 |
| Read many lines fast (large inputs) | `sys.stdin.read()` / `.split()`    | `data = sys.stdin.read().split()` |
| Reading until EOF                    | `for line in sys.stdin:`         | iterate lines of input       |
| Multi-line text block                | triple-quoted string             | `"""..."""`                   |

## 4. Syntax

```python
# Output
print(*objects, sep=' ', end='\n', file=sys.stdout, flush=False)

# Input
s = input(prompt)      # ALWAYS returns a str
n = int(input(prompt)) # parse the str into an int

# f-strings
name = "Asha"; age = 25
print(f"{name} is {age} years old")

# Reading many tokens efficiently (competitive programming)
import sys
data = sys.stdin.read().split()        # list of str tokens
n = int(data[0])
arr = list(map(int, data[1:1+n]))

# Reading line by line
for line in sys.stdin:
    line = line.rstrip()
    ...
```

## 5. Basic Example

```python
name = input("Enter your name: ")
age  = int(input("Enter your age:  "))
print(f"Hello {name}! In 10 years, you'll be {age + 10}.")
print("Counting:", 1, 2, 3, sep=" -> ", end=" <<<END>>>\n")
```

**Example session:**

```output
Enter your name: Asha
Enter your age:  25
Hello Asha! In 10 years, you'll be 35.
Counting: 1 -> 2 -> 3 <<<END>>>
```

## 6. Step-by-Step Dry Run

```python
a = int(input("a = "))     # user types "7"
b = int(input("b = "))     # user types "3"
print(f"a + b = {a + b}")
print(f"a / b = {a / b:.3f}")
```

| Step | Code executed            | State                  | Output produced (in order)   |
|------|--------------------------|------------------------|------------------------------|
| 1    | `input("a = ")`          | prompt shown; waits   | `a = `                       |
| 2    | user types `7` <Enter>   | raw str `"7"`           |                              |
| 3    | `int("7")`                | `a = 7`                |                              |
| 4    | `input("b = ")`          | waits                  | `b = `                       |
| 5    | user types `3` <Enter>   | `b = 3`                |                              |
| 6    | `print(f"a + b = {a+b}")`| evaluates `a+b = 10`    | `a + b = 10`                 |
| 7    | `print(f"a / b = {a/b:.3f}")` | 7/3 ≈ 2.333333  → .3f rounds to 3 decimals | `a / b = 2.333` |

**Final output:**

```output
a + b = 10
a / b = 2.333
```

## 7. Built-in Methods

| Method        | Purpose                                             | Syntax                          | Input                              | Output        | Example                                       | Time      | Interview Use                          | Common Mistake                                    | Shortcut                     |
|---------------|-----------------------------------------------------|---------------------------------|------------------------------------|---------------|-----------------------------------------------|-----------|-----------------------------------------|---------------------------------------------------|------------------------------|
| `print()`     | Write to stdout                                     | `print(*objs, sep, end, file)`  | any objects                        | `None`        | `print(1, 2, sep="-")`                       | O(n)      | Debugging, formatted output             | Default `sep=" "`, `end="\n"`                     | `print(*lst)` unpacks        |
| `input()`     | Read one line from stdin                            | `input(prompt="")`              | optional prompt str                | `str`         | `name = input("Name: ")`                     | O(L)      | Get user input                          | **Always** returns str; convert manually          | `int(input())` parse int      |
| `open()`      | Open a file                                         | `open(path, mode)`              | path, mode                         | file object   | `f = open("x.txt")`                            | O(1)      | File I/O                                | Forgetting `f.close()` ; use `with`               | `with open(...) as f:` algo   |
| `format()`    | Insert values into a template str                   | `"{0} {1}".format(a, b)`         | format string + args               | `str`         | `"{} {}".format("a", 1)`                      | O(n)      | Older formatting                        | Mixing slots and indexes                          | Prefer f-strings              |
| `enumerate()` | Pair each item with its index                        | `enumerate(iter, start=0)`      | iterable                            | iterator of `(i, v)` | `list(enumerate("ab"))` → `[(0,'a'),(1,'b')]` | O(n)      | Print indexed list                       | Forgetting `start` argument                       | Often used in print loops    |
| `zip()`       | Merge multiple iterables element-wise               | `zip(*iterables)`               | iterables                           | iterator of tuples | `list(zip([1,2],["a","b"]))`                  | O(n)      | Parallel printing                        | Stops at shortest iterable                        | `itertools.zip_longest`       |
| `map()`       | Apply a function to each item                       | `map(fn, iter)`                 | fn + iterable                       | iterator      | `list(map(int, ["1","2"]))` → `[1,2]`         | O(n)      | Parse lines of input                    | Forgetting to wrap in `list()`                    | Often combined with `input().split()` |
| `sys.stdin.read()` | Read all of stdin at once                       | `sys.stdin.read()`              | none                                | `str`         | `data = sys.stdin.read()`                     | O(L)      | Fast bulk reads (CP/ICPC)                | Calling `read()` again returns ""                 | `.split()` then map to int    |

### Examples

```python
# 1. print with sep and end
print("a", "b", "c", sep=" | ", end=".\n")

# 2. Reading a number from user
n = int(input("n = "))

# 3. Reading an entire array of ints on one line
nums = list(map(int, input().split()))
print("Got:", nums)

# 4. Reading many lines fast (competitive style)
import sys
data = sys.stdin.read().split()
print("Tokens:", len(data), "First:", data[0], "Last:", data[-1])

# 5. f-string formatting
price = 49.5
qty = 3
print(f"Total: ${price * qty:7.2f}")

# 6. enumerate for indexed output
for i, name in enumerate(["Asha", "Balu", "Che"], start=1):
    print(i, name)
```

**Sample outputs (with input `3` and `1 2 3`):**

```output
a | b | c.
Got: [1, 2, 3]
Tokens: 2 First: 1 Last: 3
Total: $ 148.50
1 Asha
2 Balu
3 Che
```

## 8. Interview Example

> **Q: Read two integers on one line, print their sum on a single line, then print their product on the next line, formatted to 2 decimals.**

```python
import sys
data = sys.stdin.read().split()
a, b = int(data[0]), int(data[1])
print(a + b)
print(f"{a * b:.2f}")
```

For input `5 3`:

```output
8
15.00
```

> **Q: Print the multiplication table from 1 to N where each row is space-separated.**

```python
n = int(input("n = "))
for i in range(1, n + 1):
    print(*range(i, i * n + 1, i))
```

For input `5`:

```output
1 2 3 4 5
2 4 6 8 10
3 6 9 12 15
4 8 12 16 20
5 10 15 20 25
```

> **Q: Print a right-aligned rectangle of width `W` containing text `S`.**

```python
print(f"{'rectangle':>20}")
print(f"{'*' * 20}")
```

```output
          rectangle
********************
```

## 9. When NOT to use

- Don't use `print()` for **logging in production** — use the `logging` module (it has levels, files, formatting).
- Don't `print()` inside a **hot loop** if you care about speed — I/O is slow.
- Don't use `input()` for more than one line's worth — use `sys.stdin.read()` for batch reads.
- Don't rely on `print(x, y)` formatting for serious tables — use the `str.format()` mini-language or the `tabulate` library.
- Don't use `eval(input())` — ever. It's a security hole.

## 10. Common Mistakes

| Mistake                                            | Problem                                            | Fix                          |
|----------------------------------------------------|----------------------------------------------------|------------------------------|
| `n = input(); n + 5`                               | `n` is a `str`                                     | `n = int(input())`           |
| `print(x, y, sep="-")` ignoring default `" "`       | Default `sep` is one space, not comma                | Pass `sep` explicitly         |
| `print("x", end="\n")` thinking this changes defaults | Default `end` is already `"\n"`                     | Use `end=" "` to suppress     |
| Mixing `print()` with `f"{x:5.2f}"` flags incorrectly | Format misuse                                        | Read format spec docs (`:` then `width.precision`) |
| `for line in sys.stdin: print(line)`                | Each `line` includes `\n` → double newlines         | Use `line.rstrip()`           |
| `sys.stdin.read()` called more than once             | Second call returns `""` (already consumed)         | Read once, store, then reuse   |
| `input()` inside online judge hidden tests           | Reading too much or with wrong separator            | Use `sys.stdin.read().split()`|
| Trying `print(x + " " + y)` when y is not a str     | TypeError on `+` for str and non-str                | Use f-strings or `print(x, y)` |
| Using `eval(input())` to read structures             | **Massive security risk**                            | Use `ast.literal_eval`         |
| Forgetting `flush=True` in interactive scripts       | Output buffered, never shown                        | `print(x, flush=True)`         |

## 11. Memory Tricks

- "`print` adds a newline **by default**" — to remove it, set `end=""`.
- `input()` returns **`str`** always — to convert, wrap with `int()` / `float()`.
- f-string format syntax: `f"{value:width.precision}"` — **"width left, precision right, dot in middle."**
- For fast competitive input: **"read it all, split, map"** — `sys.stdin.read().split()`.
- `print(*list)` unpacks like `print(lst[0], lst[1], ...)` — handy one-liner.
- f-strings are **Python 3.6+** only — older runtimes need `.format()` or `%`.

## 12. Interview Shortcuts

- `print(*lst, sep=" ")` prints a list as space-separated tokens — perfect for LeetCode array output.
- Use `print(*nums)` instead of `' '.join(map(str, nums))`.
- `sys.stdout.write(s)` is faster than `print(s)` (no newline added, no flush) — good for tight loops.
- Output buffering: wrap large output in a single `"\n".join(...)` then print once.
- `'\n'.join(f"{x}" for x in arr)` is the easiest "one item per line" idiom.
- For "print 2 decimals" → `f"{x:.2f}"`
- For "pad with zeros to width 5" → `f"{x:05d}"`
- For "center text in width 30" → `f"{s:^30}"`
- `input()` raises `EOFError` if stdin closes — guard for empty input.
- Online judges: prefer `sys.stdin.read()` to avoid per-call overhead.

## 13. Cheat Sheet Table

| Construct                          | Use                                            |
|------------------------------------|------------------------------------------------|
| `print(*objs)`                     | Print objects separated by spaces              |
| `print(*objs, sep=", ")`           | Custom separator                               |
| `print(*objs, end="")`             | No trailing newline                            |
| `print(*objs, file=f)`             | Print to a file                                |
| `print(*objs, flush=True)`         | Flush after print                              |
| `input(prompt)`                    | Read one line as `str`                         |
| `int(input())`                     | Read one int                                   |
| `list(map(int, input().split()))`  | Read ints on one line                          |
| `sys.stdin.read().split()`         | Bulk-read all tokens                          |
| `for line in sys.stdin:`           | Iterate lines of stdin                         |
| `f"{x:.2f}"`                       | Format float to 2 decimals                    |
| `f"{x:05d}"`                       | Pad int with zeros to width 5                  |
| `f"{s:>20}"` / `f"{s:<20}"`       | Right / left align text in width 20            |
| `f"{s:^20}"`                       | Center text                                    |
| `print(*range(1, n+1), sep="+")`   | Print with custom sep                          |

## 14. Time Complexity Table

| Method                       | Complexity | Notes                                            |
|------------------------------|------------|--------------------------------------------------|
| `print(*objs)`               | O(L)       | L = total output length                          |
| `input()`                    | O(L)       | L = length of the line                            |
| `sys.stdin.read()`          | O(L)       | Reads all input once                              |
| `sys.stdin.readline()`       | O(L)       | One line                                          |
| `for line in sys.stdin:`     | O(L total) | Streams line-by-line, low memory                  |
| `f"{x:.2f}"`                 | O(L)       | L = formatted length                              |
| `str.format(...)`            | O(L)       | Same as f-strings conceptually                   |
| `"\n".join(...)`             | O(L)       | Final concat in one pass                         |

## 15. Visual Diagram (ASCII)

### Input → Program → Output flow

```
   ┌───────────┐      ┌─────────────────┐       ┌───────────┐
   │  Keyboard │ ───> │   Python        │ ────> │  Screen   │
   │   user    │ stdin│   program       │ stdout│           │
   └───────────┘      │  ┌───────────┐  │       └───────────┘
                      │  │  input()  │  │
                      │  └─────┬─────┘  │
                      │        v        │         ┌───────────┐
                      │  ┌───────────┐   │  ─────> │  file.txt │  (when file=sys.stderr)
                      │  │ variables │   │         └───────────┘
                      │  └─────┬─────┘  │
                      │        v        │
                      │  ┌───────────┐  │
                      │  │  print()  │  │
                      │  └─────┬─────┘  │
                      │        │        │
                      └────────┴────────┘
```

### `input().split()` + `map(int, ...)` pipeline

```
   raw line:  "10 20 30 40 50"
                │
                v
        .split()  ──>  ['10','20','30','40','50']
                │
                v
       map(int, ...)  ──>  iterator of ints: 10 20 30 40 50
                │
                v
          list(...)  ──>  [10, 20, 30, 40, 50]   # ready to use
```

### `print()` separators and `end`

```
   print("A","B","C", sep=" | ", end=".\n")

  Tokens:    "A"  "B"  "C"
                 │    │    │
   sep=" | "    ┘────┴────┘     inserted BETWEEN tokens
   end=".\n"                       appended AFTER last token
                   output:  "A | B | C.\n"
```

## 16. Beginner Notes

> **Remember:**
> - `input()` always returns a `str`. Wrap with `int()`, `float()`, ... to convert.
> - `print()` separates args with a space and ends with a newline — override with `sep` and `end`.
> - For competitive programming, read everything at once with `sys.stdin.read().split()`.
> - f-strings are the cleanest formatting tool: `f"{x:width.precision}"`.
> - Always `rstrip()` lines from `for line in sys.stdin:` to remove the trailing `\n`.
> - Don't use `eval(input())` for parsing — use `ast.literal_eval` instead.

## 17. FAANG Tips

- **Speed matters in OA:** prefer `sys.stdin.read().split()` over multiple `input()` calls.
- For huge outputs, build the entire output string and print **once**: `sys.stdout.write("\n".join(lines) + "\n")`.
- Format strings matter in interviews: `f"{x:.2f}", f"{x:>10}", f"{x:,}"` (thousands separator) — show command of the format mini-language.
- Be prepared to **handle missing input / EOFError** in interview shells — wrap reads in `try / except EOFError`.
- Many platforms (LeetCode) handle I/O for you (driver code reads input, your function returns output); don't reinvent the wheel.
- For interactive problems (LeetCode "Interact with a service"), `flush=True` is mandatory.
- Code review signal: a candidate using `import sys; sys.stdin.read().split()` knows competitive-programming hygiene.
- **Never** leak secret values to `print` in production code; use structured logging.

## 18. Practice Problems

### Easy
1. **A + B Problem** — LeetCode #LCP1 / common starter (read two ints, print sum)
2. **Hello World** — LeetCode interview-style (return a specific output string)
3. **Print in Order** — LeetCode #1114 (console output / ordering)
4. **Number of Steps to Reduce a Number to Zero** — LeetCode #1342

### Medium
1. **Encode and Decode Strings** — LeetCode #271 (custom serialization read/write)
2. **String to Integer (atoi)** — LeetCode #8 (parsing input manually)
3. **Simplify Path** — LeetCode #71 (parse a path string from input)

### Hard
1. **Serialize and Deserialize Binary Tree** — LeetCode #297
2. **Text Justification** — LeetCode #68 (heavy formatting — width/spacing)
3. **Read N Characters Given Read4 II** — LeetCode #158 (custom buffered reader)

---

🔗 Next: [Operators](./operators.md) · [Type Casting](./type_casting.md) · [Variables](./variables.md) · Back to [README](../README.md)
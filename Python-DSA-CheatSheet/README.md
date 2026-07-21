# Python DSA Cheat Sheet — Beginner → FAANG Ready

> **SEO:** Python DSA cheat sheet covering variables, data types (list, dict, set, tuple, string), control flow, OOP, collections (Counter, deque, heapq), 20 algorithms (binary search, sliding window, DP, BFS/DFS), Big-O analysis, and FAANG interview patterns. Every topic with Mermaid diagrams, dry runs, and LeetCode links.
>
> A practical, interview-focused handbook to take you from Python beginner to FAANG-ready.
> Author: **Tamilselvan** · ✉️ tamilselvan.sde@gmail.com · 🔗 [LinkedIn](https://www.linkedin.com/in/tamilselvan-ai/)
>
> Philosophy: **Intuition first, theory later.** Every topic has plain-English explanations, dry runs, ASCII diagrams, time complexity, real interview patterns, and LeetCode problem links.

---

## What is this repo?

A collection of Markdown cheat sheets covering:

- Python fundamentals (variables, operators, type casting)
- Built-in data types (string, list, tuple, dict, set, boolean, numbers)
- Control flow (if / else, loops, break / continue)
- Functions & functional tools (lambda, map, filter, reduce, zip, enumerate)
- Object-oriented programming (classes, inheritance, polymorphism, encapsulation)
- Collections module (Counter, defaultdict, deque, heapq)
- Core DSA algorithms (sorting, binary search, sliding window, two pointers, BFS/DFS, DP, backtracking, greedy, trie, union-find, etc.)
- Big-O time complexity analysis
- FAANG interview patterns & shortcuts

Every file follows the **same 18-section template** — so once you learn one, you can navigate them all.

---

## How to navigate (interlinks)

Each cheat sheet cross-links to related topics using relative Markdown links like:

- ➡️ Next topic: [list.md](./02_Data_Types/list.md)
- ⬅️ Previous: [variables.md](./01_Python_Basics/variables.md)
- 🔗 See also: [dictionary.md](./02_Data_Types/dictionary.md) · [hash_map.md](./07_Algorithms/hash_map.md)

Look for these links at the **top** and **bottom** of every file.

---

## Directory structure

```text
Python-DSA-CheatSheet/
├── README.md                       ← you are here
├── 01_Python_Basics/              ← variables, I/O, operators, type casting
├── 02_Data_Types/                 ← strings, list, tuple, dict, set, boolean, numbers
├── 03_Control_Flow/               ← if/else, for, while, break/continue
├── 04_Functions/                   ← functions, lambda, map, filter, reduce, zip, enumerate
├── 05_OOP/                         ← classes, objects, inheritance, polymorphism, encapsulation
├── 06_Collections/                ← Counter, defaultdict, deque, heapq
├── 07_Algorithms/                 ← 21 algorithm cheat sheets
├── 08_Time_Complexity/            ← Big-O
└── 09_FAANG_Patterns/             ← interview pattern playbook
```

---

## Prerequisites

- A computer with Python 3.10+ installed (`python --version`)
- A code editor (VS Code recommended)
- A LeetCode account (free) — every practice problem links to LeetCode
- 1-2 hours of daily practice time, ~8 weeks commitment
- Zero prior programming experience required for Week 1

---

## Learning order (recommended sequence)

Follow this sequence **in order** — each section builds on the previous:

1. **Python Basics** → `01_Python_Basics/`
2. **Data Types** → `02_Data_Types/`
3. **Control Flow** → `03_Control_Flow/`
4. **Functions** → `04_Functions/`
5. **OOP** → `05_OOP/`
6. **Collections** → `06_Collections/`
7. **Algorithms** → `07_Algorithms/`
8. **Time Complexity** → `08_Time_Complexity/`
9. **FAANG Patterns** → `09_FAANG_Patterns/`

**Do not skip the data types section.** Lists, dicts, and sets are the foundation of 80% of DSA solutions.

---

## 4-Week DSA roadmap

### Week 1 — Python Foundations
| Day | Topic | File |
|-----|-------|------|
| Mon | Variables, I/O, operators, type casting | [01_Python_Basics](./01_Python_Basics/) |
| Tue | Strings | [strings.md](./02_Data_Types/strings.md) |
| Wed | Lists + slicing + comprehension | [list.md](./02_Data_Types/list.md) |
| Thu | Dictionary + Set | [dictionary.md](./02_Data_Types/dictionary.md) · [set.md](./02_Data_Types/set.md) |
| Fri | Tuple + Boolean + Numbers | [tuple.md](./02_Data_Types/tuple.md) |
| Sat | Control flow + loops | [03_Control_Flow](./03_Control_Flow/) |
| Sun | Functions + lambda + map/filter | [04_Functions](./04_Functions/) |

### Week 2 — Core Algorithms (Easy patterns)
| Day | Topic | File |
|-----|-------|------|
| Mon | HashMap pattern | [hash_map.md](./07_Algorithms/hash_map.md) |
| Tue | Two pointers | [two_pointers.md](./07_Algorithms/two_pointers.md) |
| Wed | Sliding window | [sliding_window.md](./07_Algorithms/sliding_window.md) |
| Thu | Prefix sum | [prefix_sum.md](./07_Algorithms/prefix_sum.md) |
| Fri | Stack + Queue | [stack.md](./07_Algorithms/stack.md) · [queue.md](./07_Algorithms/queue.md) |
| Sat | Binary search | [binary_search.md](./07_Algorithms/binary_search.md) |
| Sun | Sorting + Searching | [sorting.md](./07_Algorithms/sorting.md) |

### Week 3 — Trees, Graphs, Heap
| Day | Topic | File |
|-----|-------|------|
| Mon | Linked list | [linked_list.md](./07_Algorithms/linked_list.md) |
| Tue | Trees | [trees.md](./07_Algorithms/trees.md) |
| Wed | BFS + DFS | [bfs.md](./07_Algorithms/bfs.md) · [dfs.md](./07_Algorithms/dfs.md) |
| Thu | Graph | [graph.md](./07_Algorithms/graph.md) |
| Fri | Heap + heapq | [heap.md](./07_Algorithms/heap.md) |
| Sat | Trie | [trie.md](./07_Algorithms/trie.md) |
| Sun | Union-Find + collections | [union_find.md](./07_Algorithms/union_find.md) |

### Week 4 — Advanced Patterns
| Day | Topic | File |
|-----|-------|------|
| Mon | Recursion | [recursion.md](./07_Algorithms/recursion.md) |
| Tue | Backtracking | [backtracking.md](./07_Algorithms/backtracking.md) |
| Wed | Greedy | [greedy.md](./07_Algorithms/greedy.md) |
| Thu | Dynamic programming | [dynamic_programming.md](./07_Algorithms/dynamic_programming.md) |
| Fri | OOP + Collections deep dive | [05_OOP](./05_OOP/) · [06_Collections](./06_Collections/) |
| Sat | Big-O mastery | [big_o.md](./08_Time_Complexity/big_o.md) |
| Sun | FAANG interview patterns | [interview_patterns.md](./09_FAANG_Patterns/interview_patterns.md) |

---

## Revision checklist

Tick these off before any interview:

- [ ] I can implement binary search from memory (both inclusive & exclusive bounds)
- [ ] I can reverse a linked list in 5 lines
- [ ] I can write BFS and DFS on a graph without consulting notes
- [ ] I can solve "Two Sum" in under 3 minutes
- [ ] I know `list` vs `deque` complexity differences
- [ ] I can implement a heap using `heapq`
- [ ] I can recognize when to use a sliding window vs prefix sum
- [ ] I can write a top-down DP and bottom-up DP for the same problem
- [ ] I can explain Big-O of my solution in 1 sentence
- [ ] I have solved at least 5 problems per pattern

---

## Interview checklist (day-of interview)

- [ ] Rest well the night before
- [ ] Review the 12 standard patterns: [interview_patterns.md](./09_FAANG_Patterns/interview_patterns.md)
- [ ] Memorize these one-liners (below)
- [ ] Have Python docs for `collections`, `itertools` open in a tab
- [ ] Pen & paper ready for dry runs
- [ ] Test edge cases: empty, single element, negative numbers, duplicates
- [ ] Speak your thought process aloud — silence is the enemy
- [ ] Start with brute force, then optimize — never jump to optimal first without explanation

---

## Common Python one-liners (memorize these)

```python
# Math
max(nums)                       # largest
min(nums)                       # smallest
sum(nums)                       # total
abs(x)                          # absolute value
pow(x, n)                       # x ** n
divmod(a, b)                    # returns (a // b, a % b)

# Sorting
sorted(nums)                    # new sorted list (ascending)
sorted(nums, reverse=True)      # descending
sorted(nums, key=lambda x: -x)  # custom key
nums.sort()                     # in-place sort

# Reversal
nums[::-1]                      # reversed slice (new list)
reversed(nums)                  # reversed iterator
list.reverse()                  # in-place reverse

# Strings
''.join(lst)                    # list of chars → string
s.split(',')                    # string → list
s.strip()                       # remove whitespace
s.replace('a', 'b')             # replace chars

# Iteration
enumerate(lst)                  # (index, value) pairs
zip(a, b)                       # parallel iteration

# Counting
from collections import Counter
Counter(lst)                    # frequency map
Counter(lst).most_common(k)     # top-k frequent

# Deduplication
list(set(lst))                  # remove duplicates (no order)
dict.fromkeys(lst)              # remove duplicates (keep order)

# Initialization
[0] * n                         # list of n zeros
[[0] * n for _ in range(m)]    # 2D matrix

# Heap
import heapq
heapq.heapify(lst)              # min-heap in-place
heapq.heappushpop(h, x)         # push then pop (faster)
heapq.nsmallest(k, lst)         # k smallest
heapq.nlargest(k, lst)          # k largest

# Deque
from collections import deque
d = deque()
d.appendleft(x)                 # O(1) prepend
d.popleft()                     # O(1) pop-front

# Functional
list(map(fn, lst))              # apply fn to each
list(filter(pred, lst))         # keep where pred is true
from functools import reduce
reduce(lambda a, b: a + b, lst) # fold

# Any / All
any(lst)                        # True if any truthy
all(lst)                        # True if all truthy
```

---

## Built-in functions summary (most-used in DSA)

| Function | Purpose | Example |
|----------|---------|---------|
| `len(x)` | Size / length | `len([1,2,3])` → `3` |
| `range(n)` | Iterator 0..n-1 | `list(range(3))` → `[0,1,2]` |
| `enumerate(x)` | (index, item) pairs | see loops |
| `zip(a,b)` | Parallel tuples | `list(zip([1,2],[3,4]))` |
| `sorted(x)` | New sorted list | `sorted([3,1,2])` |
| `reversed(x)` | Reverse iterator | `list(reversed([1,2]))` |
| `map(f, x)` | Apply f to each | `list(map(str,[1,2]))` |
| `filter(f, x)` | Keep where f True | `list(filter(bool,x))` |
| `max/min/sum` | Aggregates | `max([1,5,3])` → `5` |
| `abs(x)` | Absolute value | `abs(-5)` → `5` |
| `round(x, n)` | Round decimals | `round(3.14159, 2)` → `3.14` |
| `any/all` | Boolean aggregates | `any([0,0,1])` → `True` |
| `type(x)` | Get type | `type([])` → `<class list>` |
| `isinstance(x, t)` | Type check | `isinstance(5, int)` |
| `bin/hex/oct` | Number base | `bin(10)` → `'0b1010'` |
| `ord/chr` | Char ↔ ASCII | `ord('A')` → `65` |

---

## FAANG roadmap (top patterns to master)

Master these 12 patterns and you cover ~80% of interview questions:

1. **HashMap / Frequency count** → [hash_map.md](./07_Algorithms/hash_map.md)
2. **Two pointers** → [two_pointers.md](./07_Algorithms/two_pointers.md)
3. **Sliding window** → [sliding_window.md](./07_Algorithms/sliding_window.md)
4. **Prefix sum** → [prefix_sum.md](./07_Algorithms/prefix_sum.md)
5. **Binary search** → [binary_search.md](./07_Algorithms/binary_search.md)
6. **BFS / DFS** → [bfs.md](./07_Algorithms/bfs.md) · [dfs.md](./07_Algorithms/dfs.md)
7. **Tree traversal** → [trees.md](./07_Algorithms/trees.md)
8. **Heap / Top-K** → [heap.md](./07_Algorithms/heap.md)
9. **Trie (strings)** → [trie.md](./07_Algorithms/trie.md)
10. **Backtracking** → [backtracking.md](./07_Algorithms/backtracking.md)
11. **Dynamic programming** → [dynamic_programming.md](./07_Algorithms/dynamic_programming.md)
12. **Greedy** → [greedy.md](./07_Algorithms/greedy.md)

Full playbook: [interview_patterns.md](./09_FAANG_Patterns/interview_patterns.md)

---

## Recommended external resources

- **LeetCode** — https://leetcode.com
- **NeetCode 150** — https://neetcode.io (curated problem list)
- **Python official docs** — https://docs.python.org/3/library/
- **Python Tutor** — https://pythontutor.com (visualize execution)
- **Visualgo** — https://visualgo.net (algorithm visualizations)

---

## About the author

**Tamilselvan** is a Software Development Engineer (SDE) passionate about teaching DSA in plain English.

- ✉️ tamilselvan.sde@gmail.com
- Built this handbook because most DSA resources are either too academic or too shallow.

> *"If you can't explain it to a 10-year-old, you don't really understand it yourself."*

---

## License

This cheat sheet is free for personal study. Please credit **Tamilselvan (tamilselvan.sde@gmail.com)** when sharing.

---

**Start here:** [01_Python_Basics/variables.md](./01_Python_Basics/variables.md) · Happy coding and good luck with your interviews! 🚀
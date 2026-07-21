# CTS (Cognizant) Coding Interview Preparation

## Company Overview

| Detail | Information |
|--------|-------------|
| **Interview Process** | Online Assessment → Technical Round → Managerial → HR |
| **Difficulty** | Easy to Medium |
| **Coding Round Pattern** | 2-3 coding questions |
| **Duration** | 60 minutes |
| **Platform Used** | HackerRank / AMCAT / Mettl |
| **Tips** | Focus on fundamentals, clear code, basic DSA. CTS values clean coding style. |

## 50 Coding Questions

---

### Q1: Reverse an Array

**Problem Statement:** Given an array of integers, reverse the array in-place and return the reversed array.

**Difficulty:** Easy

**Pattern:** Array manipulation / Two pointers

**Companies Asked:** CTS, TCS, Infosys, Wipro

**Concepts Needed:** Array indexing, two-pointer technique

**Constraints:**
- 1 ≤ N ≤ 10^5
- -10^9 ≤ arr[i] ≤ 10^9

**Approach 1 (Brute Force):** Create a new array and copy elements in reverse order using a loop. This uses O(N) extra space.

**Approach 2 (Optimized):** Use two pointers — one at start, one at end. Swap elements and move pointers towards center until they meet. In-place reversal using O(1) extra space.

**Python Solution:**
```python
from typing import List


def reverse_array(arr: List[int]) -> List[int]:
    left, right = 0, len(arr) - 1
    while left < right:
        arr[left], arr[right] = arr[right], arr[left]
        left += 1
        right -= 1
    return arr
```

**Dry Run:**
```
Input: [1, 2, 3, 4, 5]
Step 1: left=0, right=4 → swap(1,5) → [5, 2, 3, 4, 1]
Step 2: left=1, right=3 → swap(2,4) → [5, 4, 3, 2, 1]
Step 3: left=2, right=2 → stop (left >= right)
Output: [5, 4, 3, 2, 1]
```

**Complexity:**
- Time: O(N)
- Space: O(1)

**Common Mistakes:**
- Off-by-one errors in loop condition
- Modifying the original array when a copy was expected
- Not handling empty arrays

**Edge Cases:**
- Empty array → returns []
- Single element → returns same array
- Array with even/odd length

**Variations:** Reverse string, reverse linked list, reverse subarray

**Follow-up Questions:** Reverse array from index i to j, reverse in groups of k

**Interview Tips:** The interviewer values the two-pointer approach. Mention the in-place optimization without extra space.

**Expected Output:**
```
Input: [1,2,3,4,5] → Output: [5,4,3,2,1]
Input: [10,20,30] → Output: [30,20,10]
```

**Quick Revision Notes:**
- Two-pointer (left/right) is the optimal reverse technique
- Swapping in-place avoids O(N) extra space
- Loop condition `left < right` prevents double-swapping

---

### Q2: Check if String is Palindrome

**Problem Statement:** Given a string, determine if it is a palindrome (reads the same forwards and backwards). Ignore spaces, punctuation, and case.

**Difficulty:** Easy

**Pattern:** String manipulation / Two pointers

**Companies Asked:** CTS, TCS, Accenture, Infosys

**Concepts Needed:** String indexing, character comparison, two-pointer technique

**Constraints:**
- 1 ≤ S.length ≤ 10^5
- String contains printable ASCII characters

**Approach 1 (Brute Force):** Reverse the string and compare with original. O(N) time and O(N) space.

**Approach 2 (Optimized):** Use two pointers from both ends. Skip non-alphanumeric characters. Compare characters after converting to lowercase.

**Python Solution:**
```python
def is_palindrome(s: str) -> bool:
    left, right = 0, len(s) - 1
    while left < right:
        while left < right and not s[left].isalnum():
            left += 1
        while left < right and not s[right].isalnum():
            right -= 1
        if s[left].lower() != s[right].lower():
            return False
        left += 1
        right -= 1
    return True
```

**Dry Run:**
```
Input: "A man, a plan, a canal: Panama"
Step 1: left='A'(0), right='a'(29) → skip non-alnum → left='A', right='a' → lower: 'a'=='a'
Step 2: left='m'(2), right='m'(28) → 'm'=='m'
Step 3: left='a'(3), right='a'(27) → 'a'=='a'
... continues until center
Output: True
```

**Complexity:**
- Time: O(N)
- Space: O(1)

**Common Mistakes:**
- Not handling empty strings or single characters
- Forgetting to skip non-alphanumeric characters
- Case-sensitive comparison without conversion

**Edge Cases:**
- Empty string → True
- Single character → True
- String with only special characters → True

**Variations:** Valid palindrome II (delete at most one character), palindrome linked list

**Follow-up Questions:** Check if string is palindrome after removing at most one character

**Interview Tips:** Mention the filter + lowercase approach and two-pointer method.

**Expected Output:**
```
Input: "racecar" → Output: True
Input: "hello" → Output: False
Input: "A man, a plan, a canal: Panama" → Output: True
```

**Quick Revision Notes:**
- Two pointers from ends, skip non-alphanumeric
- Convert to lowercase before comparing
- Handle empty/single character as palindromes

---

### Q3: Find Factorial of a Number

**Problem Statement:** Given a non-negative integer N, find its factorial. Factorial of N (N!) is the product of all positive integers less than or equal to N.

**Difficulty:** Easy

**Pattern:** Recursion / Iteration

**Companies Asked:** CTS, TCS, Wipro, HCL

**Concepts Needed:** Recursion, iteration, handling large numbers

**Constraints:**
- 0 ≤ N ≤ 1000

**Approach 1 (Recursive):** Factorial(N) = N × Factorial(N-1) with base case Factorial(0) = 1.

**Approach 2 (Iterative):** Loop from 1 to N, multiply each number into result accumulator.

**Python Solution:**
```python
def factorial(n: int) -> int:
    if n < 0:
        raise ValueError("Factorial not defined for negative numbers")
    result = 1
    for i in range(2, n + 1):
        result *= i
    return result
```

**Dry Run:**
```
Input: n = 5
Step 1: result = 1, i=2 → result = 1*2 = 2
Step 2: result = 2, i=3 → result = 2*3 = 6
Step 3: result = 6, i=4 → result = 6*4 = 24
Step 4: result = 24, i=5 → result = 24*5 = 120
Output: 120
```

**Complexity:**
- Time: O(N)
- Space: O(1) iterative, O(N) recursive (call stack)

**Common Mistakes:**
- Not handling factorial of 0 (= 1)
- Integer overflow in languages without arbitrary precision
- Using recursion for large N causing stack overflow

**Edge Cases:**
- N = 0 → 1
- N = 1 → 1
- Negative N → not defined

**Variations:** Factorial of large numbers (using arrays), trailing zeroes in factorial

**Follow-up Questions:** Count trailing zeroes in N!, find last non-zero digit

**Interview Tips:** Python handles arbitrarily large integers, so overflow is not an issue.

**Expected Output:**
```
Input: 5 → Output: 120
Input: 0 → Output: 1
Input: 10 → Output: 3628800
```

**Quick Revision Notes:**
- 0! = 1 (base case)
- Iterative approach avoids stack overflow
- Python int can handle arbitrarily large factorials

---

### Q4: Generate Fibonacci Series

**Problem Statement:** Given N, generate the first N terms of the Fibonacci series. The series starts with 0, 1.

**Difficulty:** Easy

**Pattern:** Iteration / Recursion / DP

**Companies Asked:** CTS, TCS, Accenture, Infosys

**Concepts Needed:** Loop, recursion, dynamic programming basics

**Constraints:**
- 1 ≤ N ≤ 10^5

**Approach 1 (Recursive):** Fib(N) = Fib(N-1) + Fib(N-2). Exponential time complexity.

**Approach 2 (Iterative):** Start with a=0, b=1. Loop N times, update (a, b) = (b, a+b).

**Python Solution:**
```python
from typing import List


def fibonacci(n: int) -> List[int]:
    if n <= 0:
        return []
    if n == 1:
        return [0]
    fib = [0, 1]
    for i in range(2, n):
        fib.append(fib[i-1] + fib[i-2])
    return fib
```

**Dry Run:**
```
Input: n = 7
Step 1: fib = [0, 1]
Step 2: i=2 → fib[2] = 1+0 = 1 → fib = [0, 1, 1]
Step 3: i=3 → fib[3] = 1+1 = 2 → fib = [0, 1, 1, 2]
Step 4: i=4 → fib[4] = 2+1 = 3 → fib = [0, 1, 1, 2, 3]
Step 5: i=5 → fib[5] = 3+2 = 5 → fib = [0, 1, 1, 2, 3, 5]
Step 6: i=6 → fib[6] = 5+3 = 8 → fib = [0, 1, 1, 2, 3, 5, 8]
Output: [0, 1, 1, 2, 3, 5, 8]
```

**Complexity:**
- Time: O(N)
- Space: O(N) for storing series, O(1) if only printing

**Common Mistakes:**
- Starting with 1 instead of 0
- Off-by-one errors in loop (n vs n-1)
- Using naive recursion which is O(2^N)

**Edge Cases:**
- N = 0 → empty list
- N = 1 → [0]
- N = 2 → [0, 1]

**Variations:** Nth Fibonacci (return only the Nth term), Fibonacci using matrix exponentiation

**Follow-up Questions:** Find Nth Fibonacci modulo M, check if number belongs to Fibonacci

**Interview Tips:** Mention the iterative DP approach with O(N) time.

**Expected Output:**
```
Input: 7 → Output: [0, 1, 1, 2, 3, 5, 8]
Input: 1 → Output: [0]
Input: 5 → Output: [0, 1, 1, 2, 3]
```

**Quick Revision Notes:**
- First two terms: 0 and 1
- Each term = sum of previous two
- Use iterative approach for efficiency


### Q5: Check Prime Number

**Problem Statement:** Given a positive integer N, determine if it is a prime number.

**Difficulty:** Easy

**Pattern:** Number theory / Mathematics

**Companies Asked:** CTS, TCS, Wipro, HCL

**Concepts Needed:** Divisibility, optimization with square root

**Constraints:**
- 1 ≤ N ≤ 10^12

**Approach 1 (Brute Force):** Check divisibility from 2 to N-1. O(N) time.

**Approach 2 (Optimized):** Check divisibility from 2 to √N. Handle edge cases: numbers ≤ 1 are not prime.

**Python Solution:**
```python
import math


def is_prime(n: int) -> bool:
    if n <= 1:
        return False
    if n <= 3:
        return True
    if n % 2 == 0 or n % 3 == 0:
        return False
    for i in range(5, int(math.isqrt(n)) + 1, 6):
        if n % i == 0 or n % (i + 2) == 0:
            return False
    return True
```

**Dry Run:**
```
Input: n = 29
Step 1: n>1, n>3 → continue
Step 2: n%2=1, n%3=2 → continue
Step 3: sqrt(29)=5.38 → loop i=5, i+2=7
Step 4: 29%5=4, 29%7=1 → not divisible
Step 5: i=11 > 5.38 → stop
Output: True

Input: n = 15
Step 1: n%3=0 → return False
Output: False
```

**Complexity:**
- Time: O(√N)
- Space: O(1)

**Common Mistakes:**
- Not handling numbers ≤ 1
- Checking up to N instead of √N
- Missing optimization: check 2 and 3 separately, then i+=6

**Edge Cases:**
- N = 1 → False
- N = 2 → True (smallest prime)
- N = 3 → True

**Variations:** Sieve of Eratosthenes (multiple primes), prime factorization

**Follow-up Questions:** Find all primes up to N, check if N can be expressed as sum of two primes

**Interview Tips:** The 6k±1 optimization is impressive. Mention the sqrt boundary.

**Expected Output:**
```
Input: 29 → Output: True
Input: 15 → Output: False
Input: 2 → Output: True
Input: 1 → Output: False
```

**Quick Revision Notes:**
- Only check up to √N
- Check 2 and 3 separately, then 6k±1 pattern
- Numbers ≤ 1 are not prime

---

### Q6: Find GCD of Two Numbers

**Problem Statement:** Find the Greatest Common Divisor (GCD) of two integers.

**Difficulty:** Easy

**Pattern:** Number theory / Recursion

**Companies Asked:** CTS, TCS, Accenture, Infosys

**Concepts Needed:** Euclidean algorithm, recursion, modulo operation

**Constraints:**
- 1 ≤ A, B ≤ 10^9

**Approach 1 (Brute Force):** Find all divisors of both numbers and find the largest common one. O(min(A, B)).

**Approach 2 (Euclidean Algorithm):** GCD(A, B) = GCD(B, A % B). Repeat until B = 0.

**Python Solution:**
```python
def gcd(a: int, b: int) -> int:
    while b:
        a, b = b, a % b
    return a
```

**Dry Run:**
```
Input: a = 48, b = 18
Step 1: a=48, b=18 → a, b = 18, 48%18=12
Step 2: a=18, b=12 → a, b = 12, 18%12=6
Step 3: a=12, b=6 → a, b = 6, 12%6=0
Step 4: a=6, b=0 → return 6
Output: 6
```

**Complexity:**
- Time: O(log min(A, B))
- Space: O(1)

**Common Mistakes:**
- Not handling zero properly (GCD(0, B) = B)
- Using subtraction instead of modulo (slower)

**Edge Cases:**
- GCD(0, B) → B
- GCD(A, 0) → A
- GCD with prime numbers → 1

**Variations:** LCM (using GCD), GCD of array, extended Euclidean algorithm

**Follow-up Questions:** Find LCM using GCD, find GCD of N numbers

**Interview Tips:** The Euclidean algorithm is standard. LCM(A, B) = A * B / GCD(A, B).

**Expected Output:**
```
Input: (48, 18) → Output: 6
Input: (100, 25) → Output: 25
Input: (17, 13) → Output: 1
```

**Quick Revision Notes:**
- GCD(A, B) = GCD(B, A % B)
- LCM = A * B / GCD
- Euclidean algorithm is O(log min(A, B))

---

### Q7: Reverse Words in a String

**Problem Statement:** Reverse the order of words in a string. Remove extra spaces.

**Difficulty:** Easy

**Pattern:** String manipulation

**Companies Asked:** CTS, TCS, Accenture, Infosys

**Concepts Needed:** String splitting, joining

**Constraints:**
- 1 ≤ S.length ≤ 10^5

**Approach 1 (In-place):** Reverse entire string, then reverse each word.

**Approach 2 (Pythonic):** Split by whitespace, reverse list, join.

**Python Solution:**
```python
def reverse_words(s: str) -> str:
    words = s.split()
    return ' '.join(reversed(words))
```

**Dry Run:**
```
Input: "  Hello   World  Python  "
Step 1: split() → ["Hello", "World", "Python"]
Step 2: reversed → ["Python", "World", "Hello"]
Step 3: join → "Python World Hello"
Output: "Python World Hello"
```

**Complexity:**
- Time: O(N)
- Space: O(N)

**Common Mistakes:**
- Using split(' ') which doesn't handle multiple spaces
- Not trimming leading/trailing spaces

**Edge Cases:**
- Single word → returns same word
- Multiple spaces → handles automatically with split()
- Empty string → empty string

**Variations:** Reverse words preserving spaces, reverse each word individually

**Follow-up Questions:** Reverse words without using split()

**Interview Tips:** Python's split() without argument handles all whitespace automatically.

**Expected Output:**
```
Input: "Hello World" → Output: "World Hello"
Input: "  Hi   there  " → Output: "Hi there"
```

**Quick Revision Notes:**
- split() with no arg handles multiple spaces
- reversed() or [::-1] for reverse
- join() to combine back

---

### Q8: Check if Number is Armstrong

**Problem Statement:** An Armstrong number equals the sum of its digits each raised to the power of number of digits.

**Difficulty:** Easy

**Pattern:** Number theory / Digit manipulation

**Companies Asked:** CTS, TCS, Wipro, HCL

**Concepts Needed:** Digit extraction, modulo, exponentiation

**Constraints:**
- 1 ≤ N ≤ 10^6

**Approach 1 (String-based):** Convert to string, iterate over digits, compute power sum.

**Approach 2 (Math-based):** Extract digits using modulo 10.

**Python Solution:**
```python
def is_armstrong(n: int) -> bool:
    digits = [int(d) for d in str(n)]
    power = len(digits)
    total = sum(d ** power for d in digits)
    return total == n
```

**Dry Run:**
```
Input: n = 153
Step 1: digits = [1, 5, 3], power = 3
Step 2: total = 1^3 + 5^3 + 3^3 = 1 + 125 + 27 = 153
Step 3: 153 == 153 → True
Output: True

Input: n = 123
Step 1: total = 1^3 + 2^3 + 3^3 = 36
Step 2: 36 != 123 → False
Output: False
```

**Complexity:**
- Time: O(log₁₀ N)
- Space: O(log₁₀ N)

**Common Mistakes:**
- Using cubic power for all numbers (should use digit count as power)
- Not checking single-digit numbers (all are Armstrong)

**Edge Cases:**
- Single-digit numbers (0-9) → True
- N = 0 → True

**Variations:** Find all Armstrong numbers in a range

**Follow-up Questions:** Find next Armstrong number

**Interview Tips:** Exponent power equals number of digits.

**Expected Output:**
```
Input: 153 → Output: True
Input: 9474 → Output: True
Input: 123 → Output: False
```

**Quick Revision Notes:**
- Number of digits = exponent power
- Sum of each digit^power
- All single-digit numbers are Armstrong

---

### Q9: Find Second Largest in Array

**Problem Statement:** Find the second largest element in an array without sorting.

**Difficulty:** Easy

**Pattern:** Array traversal

**Companies Asked:** CTS, TCS, Accenture, Infosys

**Concepts Needed:** Variable tracking, single pass

**Constraints:**
- 2 ≤ N ≤ 10^5
- -10^9 ≤ arr[i] ≤ 10^9

**Approach 1 (Brute Force):** Sort and return second-to-last. O(N log N).

**Approach 2 (Optimized):** Single pass tracking largest and second_largest.

**Python Solution:**
```python
from typing import List, Optional


def second_largest(arr: List[int]) -> Optional[int]:
    if len(arr) < 2:
        return None
    largest = second = float('-inf')
    for num in arr:
        if num > largest:
            second = largest
            largest = num
        elif num > second and num != largest:
            second = num
    return None if second == float('-inf') else second
```

**Dry Run:**
```
Input: [10, 5, 20, 8, 12]
Step 1: largest=-∞, second=-∞
Step 2: num=10 → largest=10, second=-∞
Step 3: num=5 → second=5
Step 4: num=20 → second=10, largest=20
Step 5: num=8 → no change
Step 6: num=12 → second=12
Output: 12
```

**Complexity:**
- Time: O(N)
- Space: O(1)

**Common Mistakes:**
- Not handling duplicate largest values
- Forgetting negative numbers

**Edge Cases:**
- All elements equal → no second largest
- Only 2 elements → the smaller one

**Variations:** Kth largest element, third largest

**Follow-up Questions:** Find Kth largest using min-heap

**Expected Output:**
```
Input: [10, 5, 20, 8, 12] → Output: 12
Input: [5, 5, 5] → Output: None
```

**Quick Revision Notes:**
- Track largest and second_largest in single pass
- Handle duplicate largest (num != largest)
- Initialize with -infinity

---

### Q10: Print Star Pyramid Pattern

**Problem Statement:** Print a star pyramid with N rows.

**Difficulty:** Easy

**Pattern:** Pattern printing

**Companies Asked:** CTS, TCS, Wipro, HCL

**Concepts Needed:** Nested loops, spacing logic

**Constraints:**
- 1 ≤ N ≤ 100

**Approach:** Outer loop for rows, inner loops for spaces and stars.

**Python Solution:**
```python
def print_pyramid(n: int) -> None:
    for i in range(1, n + 1):
        spaces = ' ' * (n - i)
        stars = '*' * (2 * i - 1)
        print(spaces + stars)
```

**Dry Run:**
```
Input: n = 5
Row 1: spaces=4, stars=1 → "    *"
Row 2: spaces=3, stars=3 → "   ***"
Row 3: spaces=2, stars=5 → "  *****"
Row 4: spaces=1, stars=7 → " *******"
Row 5: spaces=0, stars=9 → "*********"
```

**Complexity:**
- Time: O(N²)
- Space: O(1)

**Common Mistakes:**
- Off-by-one in star count (2*i-1 vs 2*i+1)
- Wrong number of spaces

**Edge Cases:**
- N = 0 → no output
- N = 1 → single star

**Variations:** Inverted pyramid, diamond pattern

**Follow-up Questions:** Print hollow pyramid

**Expected Output:**
```
N=3:
  *
 ***
*****
```

**Quick Revision Notes:**
- Spaces = N - row_number
- Stars = 2 * row_number - 1
- Use string multiplication for clean code

---

### Pattern Summary (Q1-Q10)

| Pattern | Questions |
|---------|-----------|
| Array manipulation | Q1, Q9 |
| Two pointers | Q1, Q2 |
| String manipulation | Q2, Q7 |
| Recursion / Iteration | Q3, Q4 |
| Number theory | Q5, Q6, Q8 |
| Pattern printing | Q10 |

### Revision Table (Q1-Q10)

| Question | Pattern | Difficulty | Key Technique |
|----------|---------|------------|---------------|
| Q1: Reverse array | Two pointers | Easy | Swap left/right |
| Q2: Palindrome | Two pointers | Easy | Skip non-alnum |
| Q3: Factorial | Iteration | Easy | Loop multiply |
| Q4: Fibonacci | DP/Iteration | Easy | a, b = b, a+b |
| Q5: Prime check | Number theory | Easy | √N optimization |
| Q6: GCD | Euclidean | Easy | a, b = b, a%b |
| Q7: Reverse words | String | Easy | split() + join() |
| Q8: Armstrong | Digit manipulation | Easy | digit^power sum |
| Q9: Second largest | Array | Easy | Two variables |
| Q10: Pyramid | Pattern | Easy | Spaces + stars |

### Important Observations (Q1-Q10)
- Most easy problems require only basic loops and conditions
- Two-pointer technique appears frequently
- Number theory uses modulo and division extensively


### Q11: Check Perfect Number

**Problem Statement:** A perfect number equals the sum of its proper divisors (excluding itself).

**Difficulty:** Easy

**Pattern:** Number theory / Mathematics

**Companies Asked:** CTS, TCS, Wipro, HCL

**Concepts Needed:** Divisor finding, optimization with sqrt

**Constraints:**
- 1 ≤ N ≤ 10^8

**Approach 1 (Brute Force):** Iterate from 1 to N/2, sum all divisors. O(N).

**Approach 2 (Optimized):** Iterate from 2 to √N. If i divides N, add both i and N//i.

**Python Solution:**
```python
import math


def is_perfect(n: int) -> bool:
    if n <= 1:
        return False
    total = 1
    for i in range(2, int(math.isqrt(n)) + 1):
        if n % i == 0:
            total += i
            if i != n // i:
                total += n // i
    return total == n
```

**Dry Run:**
```
Input: n = 28
Step 1: total = 1, sqrt(28) = 5.29
Step 2: i=2 → 28%2=0 → total=1+2+14=17
Step 3: i=3 → 28%3=1 → skip
Step 4: i=4 → 28%4=0 → total=17+4+7=28
Step 5: i=5 → 28%5=3 → skip
Step 6: total=28 == 28 → True
Output: True
```

**Complexity:**
- Time: O(√N)
- Space: O(1)

**Common Mistakes:**
- Including the number itself in divisor sum
- Double counting for perfect squares

**Edge Cases:**
- N = 1 → False (sum of proper divisors = 0)
- N = 6 → True (1+2+3=6)

**Variations:** Find all perfect numbers up to N

**Follow-up Questions:** Check if number is abundant or deficient

**Interview Tips:** Known perfect numbers: 6, 28, 496, 8128.

**Expected Output:**
```
Input: 28 → Output: True
Input: 12 → Output: False
Input: 6 → Output: True
```

**Quick Revision Notes:**
- Sum proper divisors starting from 1
- Check up to √N and add both divisors
- Known perfect numbers: 6, 28, 496, 8128

---

### Q12: Remove Duplicates from Sorted Array

**Problem Statement:** Remove duplicates from sorted array in-place. Return new length.

**Difficulty:** Easy

**Pattern:** Array / Two pointers

**Companies Asked:** CTS, TCS, Accenture, Infosys

**Concepts Needed:** In-place modification, two-pointer

**Constraints:**
- 1 ≤ N ≤ 10^5
- Sorted non-decreasing order

**Approach 1 (Brute Force):** Use set, create new array. O(N) space.

**Approach 2 (Optimized):** Two-pointer. j tracks unique position.

**Python Solution:**
```python
from typing import List


def remove_duplicates(arr: List[int]) -> int:
    if not arr:
        return 0
    j = 1
    for i in range(1, len(arr)):
        if arr[i] != arr[j - 1]:
            arr[j] = arr[i]
            j += 1
    return j
```

**Dry Run:**
```
Input: [1, 1, 2, 2, 3, 4, 4, 5]
Step 1: j=1
Step 2: i=1, arr[1]=1 == arr[0]=1 → skip
Step 3: i=2, arr[2]=2 != arr[0]=1 → arr[1]=2, j=2
Step 4: i=3, arr[3]=2 == arr[1]=2 → skip
Step 5: i=4, arr[4]=3 != arr[1]=2 → arr[2]=3, j=3
Step 6: i=5, arr[5]=4 != arr[2]=3 → arr[3]=4, j=4
Step 7: i=6, arr[6]=4 == arr[3]=4 → skip
Step 8: i=7, arr[7]=5 != arr[3]=4 → arr[4]=5, j=5
Output: 5 (array becomes [1, 2, 3, 4, 5])
```

**Complexity:**
- Time: O(N)
- Space: O(1)

**Common Mistakes:**
- Off-by-one in j-1 versus j
- Not handling empty array

**Edge Cases:**
- Empty array → 0
- Single element → 1
- No duplicates → N

**Variations:** Remove duplicates from unsorted array

**Follow-up Questions:** Remove element from array

**Expected Output:**
```
Input: [1,1,2,2,3,4,4,5] → Output: 5
Input: [1,1,1] → Output: 1
```

**Quick Revision Notes:**
- Two-pointer (j = unique position, i = traversal)
- Works only on sorted arrays
- Compare arr[i] with arr[j-1]

---

### Q13: Find Frequency of Each Element

**Problem Statement:** Count occurrences of each distinct element in an array.

**Difficulty:** Easy

**Pattern:** Hashing / Frequency counting

**Companies Asked:** CTS, TCS, Wipro, HCL, Accenture

**Concepts Needed:** Hash map/dictionary

**Constraints:**
- 1 ≤ N ≤ 10^5
- -10^9 ≤ arr[i] ≤ 10^9

**Approach 1 (Brute Force):** For each element, count by scanning array. O(N²).

**Approach 2 (Optimized):** Use dictionary in single pass.

**Python Solution:**
```python
from typing import List, Dict


def find_frequencies(arr: List[int]) -> Dict[int, int]:
    freq = {}
    for num in arr:
        freq[num] = freq.get(num, 0) + 1
    return freq
```

**Dry Run:**
```
Input: [1, 2, 2, 3, 1, 4, 2, 3]
Step 1: num=1 → {1: 1}
Step 2: num=2 → {1: 1, 2: 1}
Step 3: num=2 → {1: 1, 2: 2}
Step 4: num=3 → {1: 1, 2: 2, 3: 1}
Step 5: num=1 → {1: 2, 2: 2, 3: 1}
Step 6: num=4 → {1: 2, 2: 2, 3: 1, 4: 1}
Step 7: num=2 → {1: 2, 2: 3, 3: 1, 4: 1}
Step 8: num=3 → {1: 2, 2: 3, 3: 2, 4: 1}
Output: {1: 2, 2: 3, 3: 2, 4: 1}
```

**Complexity:**
- Time: O(N)
- Space: O(N)

**Common Mistakes:**
- Using list count() in a loop (O(N²))
- Modifying dictionary while iterating

**Edge Cases:**
- Empty array → empty dict
- Single element → {element: 1}

**Variations:** Frequency of each character in string

**Follow-up Questions:** Find element with highest frequency

**Expected Output:**
```
Input: [1, 2, 2, 3, 1, 4] → Output: {1: 2, 2: 2, 3: 1, 4: 1}
```

**Quick Revision Notes:**
- Use dict.get(key, 0) + 1 for clean counting
- Single pass O(N) using hash map
- Python's collections.Counter does the same

---

### Q14: Matrix Addition

**Problem Statement:** Add two matrices of same dimensions.

**Difficulty:** Easy

**Pattern:** Matrix operation

**Companies Asked:** CTS, TCS, Accenture, Infosys

**Concepts Needed:** Nested loops, 2D array

**Constraints:**
- 1 ≤ M, N ≤ 100

**Approach:** Nested loops. Result[i][j] = A[i][j] + B[i][j].

**Python Solution:**
```python
from typing import List


def matrix_addition(A: List[List[int]], B: List[List[int]]) -> List[List[int]]:
    m, n = len(A), len(A[0])
    result = [[0] * n for _ in range(m)]
    for i in range(m):
        for j in range(n):
            result[i][j] = A[i][j] + B[i][j]
    return result
```

**Dry Run:**
```
Input: A=[[1,2],[3,4]], B=[[5,6],[7,8]]
Step 1: result[0][0] = 1+5 = 6
Step 2: result[0][1] = 2+6 = 8
Step 3: result[1][0] = 3+7 = 10
Step 4: result[1][1] = 4+8 = 12
Output: [[6, 8], [10, 12]]
```

**Complexity:**
- Time: O(M × N)
- Space: O(M × N)

**Common Mistakes:**
- Not handling matrices of different dimensions
- Modifying input matrix

**Variations:** Matrix subtraction, matrix multiplication

**Follow-up Questions:** Matrix multiplication

**Expected Output:**
```
Input: A=[[1,2],[3,4]], B=[[5,6],[7,8]]
Output: [[6,8],[10,12]]
```

**Quick Revision Notes:**
- Check dimensions match before adding
- Nested loops: outer=row, inner=column

---

### Q15: Linear Search

**Problem Statement:** Find first occurrence of target in array. Return -1 if not found.

**Difficulty:** Easy

**Pattern:** Searching

**Companies Asked:** CTS, TCS, Wipro, HCL

**Concepts Needed:** Array traversal, comparison

**Constraints:**
- 1 ≤ N ≤ 10^5

**Approach:** Traverse array from left to right. Return index on match.

**Python Solution:**
```python
from typing import List


def linear_search(arr: List[int], target: int) -> int:
    for i, num in enumerate(arr):
        if num == target:
            return i
    return -1
```

**Dry Run:**
```
Input: arr = [4, 2, 7, 1, 9, 3], target = 7
i=0 → 4!=7, i=1 → 2!=7, i=2 → 7==7 → return 2
Output: 2

target = 5: no match → return -1
Output: -1
```

**Complexity:**
- Time: O(N)
- Space: O(1)

**Common Mistakes:**
- Not returning first occurrence
- Forgetting to return -1

**Edge Cases:**
- Empty array → -1
- Target at first position → 0

**Variations:** Find last occurrence, count occurrences

**Follow-up Questions:** Binary search for sorted arrays

**Expected Output:**
```
Input: arr=[4,2,7,1,9,3], target=7 → Output: 2
Input: arr=[4,2,7,1,9,3], target=5 → Output: -1
```

**Quick Revision Notes:**
- O(N) time, works on unsorted arrays
- Return index immediately on match

---

### Q16: Binary Search

**Problem Statement:** Find target in sorted array using binary search.

**Difficulty:** Medium

**Pattern:** Searching / Divide and Conquer

**Companies Asked:** CTS, TCS, Accenture, Infosys

**Concepts Needed:** Divide and conquer, sorted array

**Constraints:**
- 1 ≤ N ≤ 10^5
- Sorted non-decreasing order

**Approach 1 (Brute Force):** Linear search. O(N). Ignores sorted property.

**Approach 2 (Binary Search):** Divide array in half repeatedly.

**Python Solution:**
```python
from typing import List


def binary_search(arr: List[int], target: int) -> int:
    left, right = 0, len(arr) - 1
    while left <= right:
        mid = left + (right - left) // 2
        if arr[mid] == target:
            return mid
        elif arr[mid] < target:
            left = mid + 1
        else:
            right = mid - 1
    return -1
```

**Dry Run:**
```
Input: arr = [1, 3, 5, 7, 9, 11, 13], target = 7
Step 1: left=0, right=6 → mid=3 → arr[3]=7 == 7 → return 3
Output: 3

Input: target = 6
Step 1: left=0, right=6 → mid=3 → arr[3]=7 > 6 → right=2
Step 2: left=0, right=2 → mid=1 → arr[1]=3 < 6 → left=2
Step 3: left=2, right=2 → mid=2 → arr[2]=5 < 6 → left=3
Step 4: left=3 > right=2 → exit
Output: -1
```

**Complexity:**
- Time: O(log N)
- Space: O(1)

**Common Mistakes:**
- Using `left < right` instead of `left <= right`
- mid calculation overflow (use `left + (right-left)//2`)

**Edge Cases:**
- Empty array → -1
- Single element → check or -1

**Variations:** First/last occurrence, search in rotated array

**Follow-up Questions:** Find insertion position, sqrt using binary search

**Expected Output:**
```
Input: [1,3,5,7,9,11,13], target=7 → Output: 3
Input: [1,3,5,7,9,11,13], target=6 → Output: -1
```

**Quick Revision Notes:**
- Array must be sorted
- `mid = left + (right-left)//2` prevents overflow
- Loop condition: `left <= right`

---

### Q17: Check Balanced Parentheses

**Problem Statement:** Check if parentheses are balanced (each opening has matching closing).

**Difficulty:** Medium

**Pattern:** Stack

**Companies Asked:** CTS, TCS, Accenture, Infosys

**Concepts Needed:** Stack data structure

**Constraints:**
- 1 ≤ S.length ≤ 10^5

**Approach 1 (Brute Force):** Repeatedly replace adjacent matching pairs. O(N²).

**Approach 2 (Stack):** Push opening brackets. Pop and match closing brackets.

**Python Solution:**
```python
def is_balanced(s: str) -> bool:
    stack = []
    matching = {')': '(', '}': '{', ']': '['}
    for char in s:
        if char in matching:
            if not stack or stack.pop() != matching[char]:
                return False
        else:
            stack.append(char)
    return not stack
```

**Dry Run:**
```
Input: "{[()]}"
Step 1: '{' → push → ['{']
Step 2: '[' → push → ['{', '[']
Step 3: '(' → push → ['{', '[', '(']
Step 4: ')' → pop '(' matches ')' → ['{', '[']
Step 5: ']' → pop '[' matches ']' → ['{']
Step 6: '}' → pop '{' matches '}' → []
Step 7: stack empty → True
Output: True

Input: "{[(])}"
Step 1-3: push {, [, (
Step 4: ']' → pop '(' != ']' → return False
Output: False
```

**Complexity:**
- Time: O(N)
- Space: O(N)

**Common Mistakes:**
- Not checking stack emptiness before pop
- Incorrect mapping of closing to opening brackets

**Edge Cases:**
- Empty string → True
- Single opening bracket → False

**Variations:** Minimum add to make parentheses valid

**Follow-up Questions:** Generate all balanced parentheses

**Expected Output:**
```
Input: "{[()]}" → Output: True
Input: "{[(])}" → Output: False
```

**Quick Revision Notes:**
- Use stack for opening brackets
- Dictionary maps closing → opening
- Stack must be empty at end

---

### Q18: Find Majority Element (> n/2)

**Problem Statement:** Find element appearing more than ⌊N/2⌋ times.

**Difficulty:** Medium

**Pattern:** Array / Moore's Voting Algorithm

**Companies Asked:** CTS, TCS, Accenture, Infosys

**Concepts Needed:** Voting algorithm, hash map

**Constraints:**
- 1 ≤ N ≤ 10^5

**Approach 1 (Brute Force):** Count frequency using nested loops. O(N²).

**Approach 2 (Hash Map):** Count frequencies using dictionary. O(N) time and O(N) space.

**Approach 3 (Boyer-Moore):** Maintain candidate and count. Cancel pairs.

**Python Solution:**
```python
from typing import List


def majority_element(arr: List[int]) -> int:
    candidate, count = None, 0
    for num in arr:
        if count == 0:
            candidate = num
            count = 1
        elif num == candidate:
            count += 1
        else:
            count -= 1
    return candidate
```

**Dry Run:**
```
Input: [2, 2, 1, 1, 1, 2, 2]
Step 1: num=2 → candidate=2, count=1
Step 2: num=2 → count=2
Step 3: num=1 → count=1
Step 4: num=1 → count=0
Step 5: count=0 → candidate=1, count=1
Step 6: num=2 → count=0
Step 7: count=0 → candidate=2, count=1
Output: 2
```

**Complexity:**
- Time: O(N)
- Space: O(1)

**Common Mistakes:**
- Forgetting to verify candidate (if majority not guaranteed)
- Using count == 1 instead of count == 0

**Edge Cases:**
- Single element → that element
- Two elements of same value → that value

**Variations:** Find element appearing > n/3 times

**Follow-up Questions:** Find majority in sorted array

**Expected Output:**
```
Input: [2,2,1,1,1,2,2] → Output: 2
Input: [3,3,4,2,4,4,2,4,4] → Output: 4
```

**Quick Revision Notes:**
- Boyer-Moore: cancel pairs, remaining is candidate
- O(1) space, O(N) time
- Verify candidate with second pass if needed

---

### Q19: Find Subarray with Given Sum

**Problem Statement:** Find if a contiguous subarray sums to target (non-negative numbers only).

**Difficulty:** Medium

**Pattern:** Sliding window / Two pointers

**Companies Asked:** CTS, TCS, Accenture, Wipro

**Concepts Needed:** Sliding window technique

**Constraints:**
- 1 ≤ N ≤ 10^5
- 0 ≤ arr[i], S ≤ 10^9

**Approach 1 (Brute Force):** Check all subarrays. O(N²).

**Approach 2 (Sliding Window):** Expand right pointer, shrink left when sum exceeds target.

**Python Solution:**
```python
from typing import List


def subarray_with_sum(arr: List[int], target: int) -> bool:
    left = 0
    current_sum = 0
    for right in range(len(arr)):
        current_sum += arr[right]
        while current_sum > target and left <= right:
            current_sum -= arr[left]
            left += 1
        if current_sum == target:
            return True
    return False
```

**Dry Run:**
```
Input: arr = [1, 4, 20, 3, 10, 5], target = 33
Step 1: right=0 → sum=1
Step 2: right=1 → sum=5
Step 3: right=2 → sum=25
Step 4: right=3 → sum=28
Step 5: right=4 → sum=38 > 33 → subtract arr[0]=1, sum=37, left=1
         → 37>33 → subtract arr[1]=4, sum=33, left=2
         → 33==33 → return True
Output: True
```

**Complexity:**
- Time: O(N)
- Space: O(1)

**Common Mistakes:**
- Not handling target = 0
- Infinite loop with while

**Edge Cases:**
- Single element equal to target → True
- All elements larger than target → False

**Variations:** Subarray sum with negative numbers (use prefix sum + hash map)

**Follow-up Questions:** Find length of smallest subarray with sum ≥ S

**Expected Output:**
```
Input: [1,4,20,3,10,5], target=33 → Output: True
Input: [1,4,20,3,10,5], target=18 → Output: False
```

**Quick Revision Notes:**
- Sliding window works for non-negative arrays
- Expand right, shrink left when sum exceeds
- O(N) time with O(1) space

---

### Q20: Wave Array / Rearrange Alternately

**Problem Statement:** Rearrange sorted array in wave form: arr[0] ≥ arr[1] ≤ arr[2] ≥ arr[3] ≤ ...

**Difficulty:** Medium

**Pattern:** Array rearrangement

**Companies Asked:** CTS, TCS, Accenture, Infosys

**Concepts Needed:** In-place swapping

**Constraints:**
- 1 ≤ N ≤ 10^5
- Sorted array

**Approach 1 (Brute Force):** Sort, swap adjacent pairs.

**Approach 2 (Optimized):** Since input is sorted, swap every adjacent pair at even indices.

**Python Solution:**
```python
from typing import List


def wave_array(arr: List[int]) -> List[int]:
    for i in range(0, len(arr) - 1, 2):
        arr[i], arr[i + 1] = arr[i + 1], arr[i]
    return arr
```

**Dry Run:**
```
Input: [1, 2, 3, 4, 5, 6]
Step 1: i=0 → swap(0,1) → [2, 1, 3, 4, 5, 6]
Step 2: i=2 → swap(2,3) → [2, 1, 4, 3, 5, 6]
Step 3: i=4 → swap(4,5) → [2, 1, 4, 3, 6, 5]
Output: [2, 1, 4, 3, 6, 5]
```

**Complexity:**
- Time: O(N)
- Space: O(1)

**Common Mistakes:**
- Off-by-one for odd-length arrays

**Edge Cases:**
- Single element → same array
- Two elements → swap

**Variations:** Wave array without sorting

**Follow-up Questions:** Alternate positive-negative arrangement

**Expected Output:**
```
Input: [1,2,3,4,5,6] → Output: [2,1,4,3,6,5]
Input: [1,2] → Output: [2,1]
```

**Quick Revision Notes:**
- Sorted input → swap adjacent pairs at even indices
- Guarantees wave condition
- O(N) time, in-place

---

### Pattern Summary (Q11-Q20)

| Pattern | Questions |
|---------|-----------|
| Number theory | Q11 |
| Two pointers / In-place | Q12, Q20 |
| Hashing / Frequency | Q13, Q18 |
| Matrix operations | Q14 |
| Searching | Q15, Q16 |
| Stack | Q17 |
| Sliding window | Q19 |

### Revision Table (Q11-Q20)

| Question | Pattern | Difficulty | Key Technique |
|----------|---------|------------|---------------|
| Q11: Perfect number | Number theory | Easy | Divisor sum √N |
| Q12: Remove duplicates | Two pointers | Easy | j pointer for unique |
| Q13: Frequency count | Hashing | Easy | dict.get() |
| Q14: Matrix addition | Matrix | Easy | Nested loops |
| Q15: Linear search | Searching | Easy | Enumerate |
| Q16: Binary search | Divide & conquer | Medium | Mid-point halving |
| Q17: Balanced parentheses | Stack | Medium | Mapping dict |
| Q18: Majority element | Voting | Medium | Boyer-Moore |
| Q19: Subarray sum | Sliding window | Medium | Expand/shrink |
| Q20: Wave array | Rearrangement | Medium | Adjacent swap |

### Important Observations (Q11-Q20)
- Sliding window is common for contiguous subarray problems
- Stack is default for parenthesis matching
- Boyer-Moore voting is a must-know O(1) space algorithm


### Q21: Chocolate Distribution Problem

**Problem Statement:** Minimize the difference between max and min chocolates among M students.

**Difficulty:** Medium

**Pattern:** Sliding window / Sorting + Greedy

**Companies Asked:** CTS, TCS, Accenture, Wipro

**Concepts Needed:** Sorting, sliding window

**Constraints:**
- 1 ≤ M ≤ N ≤ 10^5

**Approach 1 (Brute Force):** Generate all size-M combinations. Exponential.

**Approach 2 (Optimized):** Sort, then sliding window of size M. Min difference = min(arr[i+M-1] - arr[i]).

**Python Solution:**
```python
from typing import List


def chocolate_distribution(arr: List[int], m: int) -> int:
    if m > len(arr) or m == 0:
        return 0
    arr.sort()
    min_diff = float('inf')
    for i in range(len(arr) - m + 1):
        diff = arr[i + m - 1] - arr[i]
        min_diff = min(min_diff, diff)
    return min_diff
```

**Dry Run:**
```
Input: arr = [7, 3, 2, 4, 9, 12, 56], m = 3
Step 1: sort → [2, 3, 4, 7, 9, 12, 56]
Step 2: i=0 → window=[2,3,4], diff=2, min_diff=2
Step 3: i=1 → window=[3,4,7], diff=4, min_diff=2
Step 4: i=2 → window=[4,7,9], diff=5, min_diff=2
Step 5: i=3 → window=[7,9,12], diff=5, min_diff=2
Step 6: i=4 → window=[9,12,56], diff=47, min_diff=2
Output: 2
```

**Complexity:**
- Time: O(N log N)
- Space: O(1)

**Common Mistakes:**
- Not sorting first
- M > N or M = 0 not handled

**Edge Cases:**
- M = 1 → 0 (only one student)
- M = N → max - min

**Variations:** Fair distribution of tasks

**Expected Output:**
```
Input: [7,3,2,4,9,12,56], m=3 → Output: 2
```

**Quick Revision Notes:**
- Sort array first
- Sliding window of size M after sorting
- Minimize arr[i+M-1] - arr[i]

---

### Q22: Find Kth Smallest Element

**Problem Statement:** Find Kth smallest element (1-indexed) in unsorted array.

**Difficulty:** Medium

**Pattern:** QuickSelect / Heap

**Companies Asked:** CTS, TCS, Accenture, Infosys

**Concepts Needed:** Partitioning, priority queue

**Constraints:**
- 1 ≤ N ≤ 10^5, 1 ≤ K ≤ N

**Approach 1 (Sorting):** Sort and return arr[K-1]. O(N log N).

**Approach 2 (Min-Heap):** Build heap, pop K times. O(N + K log N).

**Approach 3 (QuickSelect):** Partition around pivot. O(N) average.

**Python Solution:**
```python
import heapq
from typing import List


def kth_smallest(arr: List[int], k: int) -> int:
    heapq.heapify(arr)
    for _ in range(k - 1):
        heapq.heappop(arr)
    return heapq.heappop(arr)
```

**Dry Run:**
```
Input: arr = [7, 10, 4, 3, 20, 15], k = 3
Step 1: heapify → [3, 7, 4, 10, 20, 15]
Step 2: pop 3 → heap = [4, 7, 15, 10, 20]
Step 3: pop 4 → heap = [7, 10, 15, 20]
Step 4: pop → 7
Output: 7
```

**Complexity:**
- Time: O(N + K log N)
- Space: O(1) (in-place heapify)

**Common Mistakes:**
- Confusing Kth smallest with Kth largest
- 0-indexed vs 1-indexed K

**Edge Cases:**
- K = 1 → minimum element
- K = N → maximum element

**Variations:** Kth largest using min-heap

**Expected Output:**
```
Input: [7,10,4,3,20,15], k=3 → Output: 7
Input: [7,10,4,3,20,15], k=4 → Output: 10
```

**Quick Revision Notes:**
- Kth smallest = index K-1 in sorted array
- Heap method: heapify + pop K times
- QuickSelect gives average O(N)

---

### Q23: Stock Buy and Sell (One Transaction)

**Problem Statement:** Find max profit from one buy-sell transaction.

**Difficulty:** Medium

**Pattern:** Array / Dynamic Programming

**Companies Asked:** CTS, TCS, Accenture, Infosys

**Concepts Needed:** Tracking minimum price

**Constraints:**
- 1 ≤ N ≤ 10^5
- 0 ≤ prices[i] ≤ 10^4

**Approach 1 (Brute Force):** Check all buy-sell pairs. O(N²).

**Approach 2 (Optimized):** Track minimum price seen. Profit = price - min_price.

**Python Solution:**
```python
from typing import List


def max_profit(prices: List[int]) -> int:
    min_price = float('inf')
    max_profit_val = 0
    for price in prices:
        min_price = min(min_price, price)
        profit = price - min_price
        max_profit_val = max(max_profit_val, profit)
    return max_profit_val
```

**Dry Run:**
```
Input: prices = [7, 1, 5, 3, 6, 4]
Step 1: price=7 → min=7, profit=0, max_profit=0
Step 2: price=1 → min=1, profit=0, max_profit=0
Step 3: price=5 → min=1, profit=4, max_profit=4
Step 4: price=3 → min=1, profit=2, max_profit=4
Step 5: price=6 → min=1, profit=5, max_profit=5
Step 6: price=4 → min=1, profit=3, max_profit=5
Output: 5
```

**Complexity:**
- Time: O(N)
- Space: O(1)

**Common Mistakes:**
- Allowing multiple transactions (single only)
- Buying after selling (violates order)

**Edge Cases:**
- Decreasing prices → 0
- Single day → 0

**Variations:** Multiple transactions, with cooldown

**Expected Output:**
```
Input: [7,1,5,3,6,4] → Output: 5
Input: [7,6,4,3,1] → Output: 0
```

**Quick Revision Notes:**
- Track minimum buy price seen so far
- Profit = current - min_price
- Return max profit (0 if only loss)

---

### Q24: Move All Zeroes to End

**Problem Statement:** Move all zeroes to end maintaining relative order of non-zero elements.

**Difficulty:** Medium

**Pattern:** Array / Two pointers

**Companies Asked:** CTS, TCS, Accenture, Wipro

**Concepts Needed:** In-place modification

**Constraints:**
- 1 ≤ N ≤ 10^5

**Approach 1 (Brute Force):** New array with non-zero first, then zeroes. O(N) space.

**Approach 2 (Optimized):** Swap non-zero element with position j.

**Python Solution:**
```python
from typing import List


def move_zeroes(arr: List[int]) -> List[int]:
    j = 0
    for i in range(len(arr)):
        if arr[i] != 0:
            arr[i], arr[j] = arr[j], arr[i]
            j += 1
    return arr
```

**Dry Run:**
```
Input: [0, 1, 0, 3, 12]
Step 1: i=0, arr[0]=0 → skip (j=0)
Step 2: i=1, arr[1]=1 → swap(1,0) → [1, 0, 0, 3, 12], j=1
Step 3: i=2, arr[2]=0 → skip (j=1)
Step 4: i=3, arr[3]=3 → swap(3,1) → [1, 3, 0, 0, 12], j=2
Step 5: i=4, arr[4]=12 → swap(12,2) → [1, 3, 12, 0, 0], j=3
Output: [1, 3, 12, 0, 0]
```

**Complexity:**
- Time: O(N)
- Space: O(1)

**Common Mistakes:**
- Not maintaining relative order
- Counting zeroes and overwriting

**Edge Cases:**
- No zeroes → same array
- All zeroes → same array

**Variations:** Move zeroes to front

**Expected Output:**
```
Input: [0,1,0,3,12] → Output: [1,3,12,0,0]
Input: [0,0,1] → Output: [1,0,0]
```

**Quick Revision Notes:**
- j tracks where next non-zero goes
- Swap non-zero with position j
- All zeroes naturally end at end

---

### Q25: Find Leaders in Array

**Problem Statement:** Find elements greater than all elements to their right.

**Difficulty:** Medium

**Pattern:** Array traversal

**Companies Asked:** CTS, TCS, Accenture, Wipro

**Concepts Needed:** Reverse traversal

**Constraints:**
- 1 ≤ N ≤ 10^5

**Approach 1 (Brute Force):** For each element, scan right side. O(N²).

**Approach 2 (Optimized):** Traverse right-to-left tracking max.

**Python Solution:**
```python
from typing import List


def find_leaders(arr: List[int]) -> List[int]:
    n = len(arr)
    leaders = []
    max_from_right = float('-inf')
    for i in range(n - 1, -1, -1):
        if arr[i] > max_from_right:
            leaders.append(arr[i])
            max_from_right = arr[i]
    return leaders[::-1]
```

**Dry Run:**
```
Input: [16, 17, 4, 3, 5, 2]
Step 1: i=5, arr[5]=2 > -∞ → leader=2, max=2
Step 2: i=4, arr[4]=5 > 2 → leader=5, max=5
Step 3: i=3, arr[3]=3 < 5 → skip
Step 4: i=2, arr[2]=4 < 5 → skip
Step 5: i=1, arr[1]=17 > 5 → leader=17, max=17
Step 6: i=0, arr[0]=16 < 17 → skip
leaders collected: [2, 5, 17] → reverse: [17, 5, 2]
Output: [17, 5, 2]
```

**Complexity:**
- Time: O(N)
- Space: O(N) for output

**Common Mistakes:**
- Not including rightmost element (always a leader)
- Using >= instead of >

**Edge Cases:**
- Single element → that element
- Decreasing array → all elements are leaders

**Variations:** Find elements greater than all to left

**Expected Output:**
```
Input: [16,17,4,3,5,2] → Output: [17,5,2]
Input: [1,2,3,4,5] → Output: [5]
```

**Quick Revision Notes:**
- Traverse right-to-left, track max seen
- Rightmost element is always a leader
- Strictly greater comparison

---

### Q26: Check if Two Strings are Anagrams

**Problem Statement:** Check if two strings have the same characters with same frequencies.

**Difficulty:** Medium

**Pattern:** Hashing / String

**Companies Asked:** CTS, TCS, Accenture, Infosys

**Concepts Needed:** Character counting

**Constraints:**
- 1 ≤ S1.length, S2.length ≤ 10^5
- Lowercase letters only

**Approach 1 (Sorting):** Sort both and compare. O(N log N).

**Approach 2 (Count Array):** Array of size 26. Increment for s1, decrement for s2.

**Python Solution:**
```python
def is_anagram(s1: str, s2: str) -> bool:
    if len(s1) != len(s2):
        return False
    count = [0] * 26
    for c1, c2 in zip(s1, s2):
        count[ord(c1) - ord('a')] += 1
        count[ord(c2) - ord('a')] -= 1
    return all(c == 0 for c in count)
```

**Dry Run:**
```
Input: s1 = "listen", s2 = "silent"
Step 1: len both = 6
Step 2: count array updates cancel out
Step 3: All counts are 0 → True
Output: True
```

**Complexity:**
- Time: O(N)
- Space: O(1) (array of size 26)

**Common Mistakes:**
- Not checking length equality first
- Forgetting uppercase handling

**Edge Cases:**
- Empty strings → True
- Different lengths → False

**Variations:** Group anagrams, find all anagram substrings

**Follow-up Questions:** Valid anagram with Unicode

**Expected Output:**
```
Input: s1="listen", s2="silent" → Output: True
Input: s1="hello", s2="world" → Output: False
```

**Quick Revision Notes:**
- Check length equality first
- Use array of size 26
- Increment for s1, decrement for s2

---

### Q27: Find First Non-Repeating Character

**Problem Statement:** Find first character that appears exactly once. Return '-' if none.

**Difficulty:** Medium

**Pattern:** Hashing / String

**Companies Asked:** CTS, TCS, Accenture, Infosys

**Concepts Needed:** Frequency counting, order preservation

**Constraints:**
- 1 ≤ S.length ≤ 10^5
- Lowercase letters

**Approach 1 (Brute Force):** For each char, scan string for duplicates. O(N²).

**Approach 2 (Two Pass):** Count frequencies first, then find first with count=1.

**Python Solution:**
```python
def first_non_repeating(s: str) -> str:
    count = [0] * 26
    for c in s:
        count[ord(c) - ord('a')] += 1
    for c in s:
        if count[ord(c) - ord('a')] == 1:
            return c
    return '-'
```

**Dry Run:**
```
Input: s = "geeksforgeeks"
Step 1: Count: g:2, e:4, k:2, s:2, f:1, o:1, r:1
Step 2: Traverse: 'g'→2, 'e'→4, 'e'→4, 'k'→2, 's'→2, 'f'→1 → return 'f'
Output: 'f'
```

**Complexity:**
- Time: O(N)
- Space: O(1)

**Common Mistakes:**
- Using dict without order preservation
- Returning from count dict (unordered)

**Edge Cases:**
- Empty string → '-'
- All characters repeat → '-'

**Variations:** First repeating character

**Expected Output:**
```
Input: "geeksforgeeks" → Output: 'f'
Input: "aabbcc" → Output: '-'
Input: "abcd" → Output: 'a'
```

**Quick Revision Notes:**
- Two passes: count first, then find
- Use ordered traversal for "first"
- O(N) time with O(1) space

---

### Q28: Count and Say Problem

**Problem Statement:** Generate Nth term of count-and-say sequence.

**Difficulty:** Medium

**Pattern:** String manipulation

**Companies Asked:** CTS, TCS, Accenture, Wipro

**Concepts Needed:** String building, run-length encoding

**Constraints:**
- 1 ≤ N ≤ 30

**Approach:** Iterative. Start with "1". For each iteration, build next term.

**Python Solution:**
```python
def count_and_say(n: int) -> str:
    result = "1"
    for _ in range(2, n + 1):
        current = ""
        i = 0
        while i < len(result):
            count = 1
            while i + 1 < len(result) and result[i] == result[i + 1]:
                count += 1
                i += 1
            current += str(count) + result[i]
            i += 1
        result = current
    return result
```

**Dry Run:**
```
Input: n = 5
Iter 1: "1"
Iter 2: one 1 = "11"
Iter 3: two 1s = "21"
Iter 4: one 2, one 1 = "1211"
Iter 5: one 1, one 2, two 1s = "111221"
Output: "111221"
```

**Complexity:**
- Time: O(2^N) (length grows exponentially)
- Space: O(2^N)

**Common Mistakes:**
- Off-by-one in iteration count
- Appending digit before count

**Edge Cases:**
- N = 1 → "1"

**Variations:** Look-and-say sequence

**Expected Output:**
```
Input: 4 → Output: "1211"
Input: 5 → Output: "111221"
```

**Quick Revision Notes:**
- Rule: count + digit for each run
- Sequence: "1", "11", "21", "1211", "111221"
- Nth term describes (N-1)th term

---

### Q29: Maximum Subarray Sum (Kadane's Algorithm)

**Problem Statement:** Find maximum sum of any contiguous subarray.

**Difficulty:** Medium

**Pattern:** Dynamic Programming / Kadane's Algorithm

**Companies Asked:** CTS, TCS, Accenture, Infosys, Wipro

**Concepts Needed:** Dynamic programming

**Constraints:**
- 1 ≤ N ≤ 10^5

**Approach 1 (Brute Force):** Check all subarrays. O(N²).

**Approach 2 (Kadane's):** max_ending_here = max(arr[i], max_ending_here + arr[i]).

**Python Solution:**
```python
from typing import List


def max_subarray_sum(arr: List[int]) -> int:
    max_ending_here = max_so_far = arr[0]
    for i in range(1, len(arr)):
        max_ending_here = max(arr[i], max_ending_here + arr[i])
        max_so_far = max(max_so_far, max_ending_here)
    return max_so_far
```

**Dry Run:**
```
Input: [-2, 1, -3, 4, -1, 2, 1, -5, 4]
Step 1: max_end=-2, max_sofar=-2
Step 2: i=1 → max(1, -2+1=-1)=1, max_sofar=1
Step 3: i=2 → max(-3, 1-3=-2)=-2, max_sofar=1
Step 4: i=3 → max(4, -2+4=2)=4, max_sofar=4
Step 5: i=4 → max(-1, 4-1=3)=3, max_sofar=4
Step 6: i=5 → max(2, 3+2=5)=5, max_sofar=5
Step 7: i=6 → max(1, 5+1=6)=6, max_sofar=6
Step 8: i=7 → max(-5, 6-5=1)=1, max_sofar=6
Step 9: i=8 → max(4, 1+4=5)=5, max_sofar=6
Output: 6 (subarray [4, -1, 2, 1])
```

**Complexity:**
- Time: O(N)
- Space: O(1)

**Common Mistakes:**
- Initializing max_so_far to 0 (fails for all-negative arrays)

**Edge Cases:**
- All negative → maximum element
- Single element → that element

**Variations:** Maximum circular subarray, max product subarray

**Expected Output:**
```
Input: [-2,1,-3,4,-1,2,1,-5,4] → Output: 6
Input: [-1,-2,-3] → Output: -1
```

**Quick Revision Notes:**
- max_ending_here = max(arr[i], max_ending_here + arr[i])
- Initialization matters for all-negative arrays
- O(N) time, O(1) space

---

### Q30: Find Missing Number in Array

**Problem Statement:** Find missing number from 1 to N in array of N-1 distinct integers.

**Difficulty:** Medium

**Pattern:** Array / Mathematics / XOR

**Companies Asked:** CTS, TCS, Accenture, Infosys, Wipro

**Concepts Needed:** Sum formula, XOR

**Constraints:**
- 2 ≤ N ≤ 10^5
- Array has N-1 elements

**Approach 1 (Brute Force):** For each number 1..N, check existence. O(N²).

**Approach 2 (Sum Formula):** Expected - actual = missing.

**Approach 3 (XOR):** XOR all numbers 1..N and all array elements.

**Python Solution:**
```python
from typing import List


def find_missing_number(arr: List[int], n: int) -> int:
    expected_sum = n * (n + 1) // 2
    actual_sum = sum(arr)
    return expected_sum - actual_sum
```

**Dry Run:**
```
Input: arr = [1, 2, 4, 5, 6], n = 6
Step 1: expected_sum = 6 * 7 / 2 = 21
Step 2: actual_sum = 1 + 2 + 4 + 5 + 6 = 18
Step 3: missing = 21 - 18 = 3
Output: 3
```

**Complexity:**
- Time: O(N)
- Space: O(1)

**Common Mistakes:**
- Using N as array length instead of N (array has N-1 elements)

**Edge Cases:**
- Missing number is 1
- Missing number is N

**Variations:** Find missing number with duplicates

**Expected Output:**
```
Input: [1,2,4,5,6], n=6 → Output: 3
Input: [2,3,4,5], n=5 → Output: 1
```

**Quick Revision Notes:**
- Sum(1 to N) = N*(N+1)/2
- Missing = expected - actual
- XOR approach also works

---

### Pattern Summary (Q21-Q30)

| Pattern | Questions |
|---------|-----------|
| Sliding window | Q21 |
| Heap / QuickSelect | Q22 |
| Dynamic Programming | Q23, Q29 |
| Two pointers | Q24 |
| Array traversal | Q25 |
| Hashing / Frequency | Q26, Q27 |
| String manipulation | Q28 |
| Mathematics | Q30 |

### Revision Table (Q21-Q30)

| Question | Pattern | Difficulty | Key Technique |
|----------|---------|------------|---------------|
| Q21: Chocolate distribution | Sliding window | Medium | Sort + window |
| Q22: Kth smallest | Heap | Medium | heapify + pop K |
| Q23: Stock buy/sell | DP | Medium | Track min price |
| Q24: Move zeroes | Two pointers | Medium | Swap non-zero |
| Q25: Leaders in array | Reverse traversal | Medium | Right-to-left max |
| Q26: Anagrams | Hashing | Medium | Count[26] array |
| Q27: First non-repeating | Hashing | Medium | Two pass |
| Q28: Count and say | String | Medium | Run-length encoding |
| Q29: Kadane's algorithm | DP | Medium | max_ending_here |
| Q30: Missing number | Math | Medium | Sum formula |

### Important Observations (Q21-Q30)
- Sorting + window is common for minimization problems
- Two-pass is useful for "first" + "frequency" problems
- Kadane's is a must-know DP pattern


### Q31: Intersection of Two Arrays

**Problem Statement:** Find unique elements common to both arrays.

**Difficulty:** Medium

**Pattern:** Hashing / Set operations

**Companies Asked:** CTS, TCS, Accenture, Infosys

**Concepts Needed:** Set, hash map

**Constraints:**
- 1 ≤ N, M ≤ 10^5

**Approach 1 (Brute Force):** Nested loops. O(N×M).

**Approach 2 (Using Set):** Convert first to set. Check second against set.

**Python Solution:**
```python
from typing import List


def intersection(arr1: List[int], arr2: List[int]) -> List[int]:
    set1 = set(arr1)
    result = []
    for num in arr2:
        if num in set1:
            result.append(num)
            set1.remove(num)
    return result
```

**Dry Run:**
```
Input: arr1 = [1, 2, 2, 1], arr2 = [2, 2]
Step 1: set1 = {1, 2}
Step 2: arr2[0]=2 → in set1 → result=[2], set1={1}
Step 3: arr2[1]=2 → not in set1 → skip
Output: [2]
```

**Complexity:**
- Time: O(N + M)
- Space: O(min(N, M))

**Common Mistakes:**
- Including duplicates in result

**Edge Cases:**
- No intersection → empty list

**Variations:** Union of two arrays

**Expected Output:**
```
Input: [1,2,2,1], [2,2] → Output: [2]
Input: [4,9,5], [9,4,9,8,4] → Output: [9,4]
```

**Quick Revision Notes:**
- Use set for O(1) lookup
- Remove from set to avoid duplicates

---

### Q32: Find Duplicate in Array

**Problem Statement:** Find one duplicate in N+1 array with values [1, N].

**Difficulty:** Medium

**Pattern:** Array / Floyd's Cycle Detection

**Companies Asked:** CTS, TCS, Accenture, Infosys

**Concepts Needed:** Cycle detection, hash set

**Constraints:**
- 1 ≤ N ≤ 10^5, array length N+1

**Approach 1 (Hash Set):** Track seen. O(N) time, O(N) space.

**Approach 2 (Floyd's Cycle):** Treat as linked list. Slow/fast pointers.

**Python Solution:**
```python
from typing import List


def find_duplicate(arr: List[int]) -> int:
    slow = fast = arr[0]
    while True:
        slow = arr[slow]
        fast = arr[arr[fast]]
        if slow == fast:
            break
    slow = arr[0]
    while slow != fast:
        slow = arr[slow]
        fast = arr[fast]
    return slow
```

**Dry Run:**
```
Input: arr = [1, 3, 4, 2, 2]
Phase 1: slow=1, fast=1
Step 1: slow=arr[1]=3, fast=arr[arr[1]]=arr[3]=2
Step 2: slow=arr[3]=2, fast=arr[arr[2]]=arr[4]=2 → slow==fast
Phase 2: slow=arr[0]=1, fast=2
Step 3: slow=arr[1]=3, fast=arr[2]=4
Step 4: slow=arr[3]=2, fast=arr[4]=2 → return 2
Output: 2
```

**Complexity:**
- Time: O(N)
- Space: O(1)

**Common Mistakes:**
- Not understanding cycle detection concept

**Edge Cases:**
- Duplicate at position 0
- N = 1, array = [1, 1]

**Variations:** Find all duplicates

**Expected Output:**
```
Input: [1,3,4,2,2] → Output: 2
Input: [3,1,3,4,2] → Output: 3
```

**Quick Revision Notes:**
- Hash set is simplest: O(N) space
- Floyd's cycle detection gives O(1) space

---

### Q33: Matrix Transpose

**Problem Statement:** Find transpose of a matrix (rows become columns).

**Difficulty:** Medium

**Pattern:** Matrix operation

**Companies Asked:** CTS, TCS, Accenture, Infosys

**Concepts Needed:** 2D array manipulation

**Constraints:**
- 1 ≤ M, N ≤ 100

**Approach 1 (New Matrix):** Create N×M matrix. result[j][i] = matrix[i][j].

**Approach 2 (In-place):** For square matrices only, swap across diagonal.

**Python Solution:**
```python
from typing import List


def transpose(matrix: List[List[int]]) -> List[List[int]]:
    m, n = len(matrix), len(matrix[0])
    result = [[0] * m for _ in range(n)]
    for i in range(m):
        for j in range(n):
            result[j][i] = matrix[i][j]
    return result
```

**Dry Run:**
```
Input: [[1, 2, 3], [4, 5, 6]]  (2×3)
Step 1: result = [[0,0], [0,0], [0,0]]  (3×2)
Step 2: i=0,j=0→result[0][0]=1; i=0,j=1→result[1][0]=2; i=0,j=2→result[2][0]=3
         i=1,j=0→result[0][1]=4; i=1,j=1→result[1][1]=5; i=1,j=2→result[2][1]=6
Output: [[1, 4], [2, 5], [3, 6]]
```

**Complexity:**
- Time: O(M × N)
- Space: O(M × N)

**Common Mistakes:**
- Trying to transpose non-square matrix in-place

**Edge Cases:**
- 1×1 matrix → same
- Row matrix (1×N) → column matrix (N×1)

**Variations:** Rotate matrix 90 degrees

**Expected Output:**
```
Input: [[1,2,3],[4,5,6]] → Output: [[1,4],[2,5],[3,6]]
```

**Quick Revision Notes:**
- Dimensions swap: M×N → N×M
- result[j][i] = matrix[i][j]
- In-place only for square matrices

---

### Q34: Check if Array is Sorted and Rotated

**Problem Statement:** Check if array was sorted and then rotated some positions.

**Difficulty:** Medium

**Pattern:** Array traversal

**Companies Asked:** CTS, TCS, Wipro

**Concepts Needed:** Counting inversions

**Constraints:**
- 1 ≤ N ≤ 10^5

**Approach 1 (Brute Force):** Find rotation point, verify sorted order.

**Approach 2 (Optimized):** Count descents (arr[i] > arr[(i+1)%N]). Must be ≤ 1.

**Python Solution:**
```python
from typing import List


def is_sorted_and_rotated(arr: List[int]) -> bool:
    n = len(arr)
    descents = 0
    for i in range(n):
        if arr[i] > arr[(i + 1) % n]:
            descents += 1
            if descents > 1:
                return False
    return True
```

**Dry Run:**
```
Input: [3, 4, 5, 1, 2]
Step 1: 3<4, 4<5, 5>1 → descents=1, 1<2, 2<3 (circular)
Output: True

Input: [1, 2, 3, 5, 4]
Step 1: 1<2, 2<3, 3<5, 5>4 → descents=1
Step 2: 4>1 (circular) → descents=2 → return False
Output: False
```

**Complexity:**
- Time: O(N)
- Space: O(1)

**Common Mistakes:**
- Forgetting circular wrap (last to first)

**Edge Cases:**
- Already sorted → True (0 descents)
- Single element → True

**Variations:** Find rotation count

**Expected Output:**
```
Input: [3,4,5,1,2] → Output: True
Input: [1,2,3,4,5] → Output: True
Input: [1,2,3,5,4] → Output: False
```

**Quick Revision Notes:**
- Count descents where arr[i] > arr[i+1]
- Valid arrays have ≤ 1 descent (circular)

---

### Q35: Find All Pairs with Given Sum

**Problem Statement:** Find all unique pairs that sum to target.

**Difficulty:** Medium

**Pattern:** Hashing / Two pointers

**Companies Asked:** CTS, TCS, Accenture, Infosys

**Concepts Needed:** Hash set

**Constraints:**
- 1 ≤ N ≤ 10^5

**Approach 1 (Brute Force):** Check all pairs. O(N²).

**Approach 2 (Hash Set):** For each num, check if target - num is in set.

**Python Solution:**
```python
from typing import List, Tuple


def find_pairs(arr: List[int], target: int) -> List[Tuple[int, int]]:
    seen = set()
    result = set()
    for num in arr:
        complement = target - num
        if complement in seen:
            pair = (min(num, complement), max(num, complement))
            result.add(pair)
        seen.add(num)
    return list(result)
```

**Dry Run:**
```
Input: arr = [1, 5, 7, -1, 5], target = 6
Step 1: num=1, complement=5 → seen={1}
Step 2: num=5, complement=1 → in seen → pair=(1,5) → seen={1,5}
Step 3: num=7, complement=-1 → seen={1,5,7}
Step 4: num=-1, complement=7 → in seen → pair=(-1,7) → seen={1,5,7,-1}
Step 5: num=5 → pair already in result
Output: [(1, 5), (-1, 7)]
```

**Complexity:**
- Time: O(N)
- Space: O(N)

**Common Mistakes:**
- Counting pairs twice ((a,b) and (b,a))

**Edge Cases:**
- No pairs → empty list

**Variations:** Count pairs with given sum

**Expected Output:**
```
Input: [1,5,7,-1,5], target=6 → Output: [(1,5), (-1,7)]
Input: [2,2,2], target=4 → Output: [(2,2)]
```

**Quick Revision Notes:**
- Use set for complement lookup
- Sort pair and use result set for uniqueness

---

### Q36: Find Longest Common Prefix

**Problem Statement:** Find longest common prefix among all strings.

**Difficulty:** Medium

**Pattern:** String manipulation

**Companies Asked:** CTS, TCS, Accenture, Infosys

**Concepts Needed:** String comparison

**Constraints:**
- 1 ≤ N ≤ 200, 0 ≤ S[i].length ≤ 200

**Approach 1 (Horizontal):** Start with first string as prefix, trim for each subsequent.

**Approach 2 (Vertical):** Compare characters at each position across all strings.

**Python Solution:**
```python
from typing import List


def longest_common_prefix(strs: List[str]) -> str:
    if not strs:
        return ""
    prefix = strs[0]
    for s in strs[1:]:
        while not s.startswith(prefix):
            prefix = prefix[:-1]
            if not prefix:
                return ""
    return prefix
```

**Dry Run:**
```
Input: ["flower", "flow", "flight"]
Step 1: prefix = "flower"
Step 2: s="flow" → trim: "flowe"→"flow" → matches
Step 3: s="flight" → trim: "flow"→"flo"→"fl" → matches
Output: "fl"
```

**Complexity:**
- Time: O(S) where S is sum of all characters
- Space: O(1)

**Common Mistakes:**
- Not handling empty strings in list

**Edge Cases:**
- Empty list → ""
- Single string → that string

**Variations:** Longest common prefix vertical scanning

**Expected Output:**
```
Input: ["flower","flow","flight"] → Output: "fl"
Input: ["dog","racecar","car"] → Output: ""
```

**Quick Revision Notes:**
- Start with first string as prefix
- Trim prefix character by character until it matches

---

### Q37: Valid Anagram Check

**Problem Statement:** Check if t is an anagram of s.

**Difficulty:** Medium

**Pattern:** Hashing / Sorting

**Companies Asked:** CTS, TCS, Accenture, Infosys

**Concepts Needed:** Character counting

**Constraints:**
- 1 ≤ S.length, T.length ≤ 10^5

**Approach 1 (Sorting):** Sort both strings. O(N log N).

**Approach 2 (Frequency Array):** Count[26] array. Increment/decrement.

**Python Solution:**
```python
def valid_anagram(s: str, t: str) -> bool:
    if len(s) != len(t):
        return False
    counts = [0] * 26
    for c in s:
        counts[ord(c) - ord('a')] += 1
    for c in t:
        idx = ord(c) - ord('a')
        counts[idx] -= 1
        if counts[idx] < 0:
            return False
    return True
```

**Dry Run:**
```
Input: s = "anagram", t = "nagaram"
Step 1: len both = 7
Step 2: Count s: a:3, n:1, g:1, r:1, m:1
Step 3: Process t → all decrement to zero, no negative
Output: True
```

**Complexity:**
- Time: O(N)
- Space: O(1)

**Common Mistakes:**
- Not checking length equality first

**Edge Cases:**
- Empty strings → True
- Different lengths → False

**Variations:** Group anagrams

**Expected Output:**
```
Input: s="anagram", t="nagaram" → Output: True
Input: s="rat", t="car" → Output: False
```

**Quick Revision Notes:**
- Length check first for early exit
- Array of size 26 for lowercase letters

---

### Q38: Remove Vowels from String

**Problem Statement:** Remove all vowels (a, e, i, o, u) from string.

**Difficulty:** Easy

**Pattern:** String manipulation

**Companies Asked:** CTS, TCS, Wipro, HCL

**Concepts Needed:** Character iteration, filtering

**Constraints:**
- 1 ≤ S.length ≤ 10^5

**Approach 1 (Iterative):** Build new string excluding vowels.

**Approach 2 (List Comprehension):** Join filtered characters.

**Python Solution:**
```python
def remove_vowels(s: str) -> str:
    vowels = set('aeiouAEIOU')
    return ''.join(c for c in s if c not in vowels)
```

**Dry Run:**
```
Input: "Hello World"
Step 1: 'H' → keep, 'e' → skip, 'l' → keep, 'l' → keep, 'o' → skip
         ' ' → keep, 'W' → keep, 'o' → skip, 'r' → keep, 'l' → keep, 'd' → keep
Output: "Hll Wrld"
```

**Complexity:**
- Time: O(N)
- Space: O(N)

**Common Mistakes:**
- Only handling lowercase vowels

**Edge Cases:**
- Empty string → ""
- String with only vowels → ""

**Variations:** Remove consonants

**Expected Output:**
```
Input: "Hello World" → Output: "Hll Wrld"
Input: "aeiou" → Output: ""
```

**Quick Revision Notes:**
- Use set for vowel lookup (both cases)
- ''.join(list comprehension) is efficient

---

### Q39: Find Sum of Digits Until Single Digit

**Problem Statement:** Digital root: repeatedly add digits until single digit.

**Difficulty:** Easy

**Pattern:** Number theory / Digit manipulation

**Companies Asked:** CTS, TCS, Wipro, HCL

**Concepts Needed:** Digital root formula

**Constraints:**
- 0 ≤ N ≤ 10^18

**Approach 1 (Iterative):** Keep summing digits until < 10.

**Approach 2 (Mathematical):** If N = 0 → 0. If N%9==0 → 9. Else → N%9.

**Python Solution:**
```python
def digital_root(n: int) -> int:
    if n == 0:
        return 0
    if n % 9 == 0:
        return 9
    return n % 9
```

**Dry Run:**
```
Input: n = 9875
Step 1: 9875 % 9 = 2
Step 2: 2 != 0 → return 2
Output: 2 (9+8+7+5=29, 2+9=11, 1+1=2)

Input: n = 999
Step 1: n % 9 == 0 → return 9
Output: 9 (9+9+9=27, 2+7=9)
```

**Complexity:**
- Time: O(1)
- Space: O(1)

**Common Mistakes:**
- Forgetting N=0 returns 0 (not 9)

**Edge Cases:**
- N = 0 → 0
- N divisible by 9 → 9

**Variations:** Sum of digits (not until single)

**Expected Output:**
```
Input: 9875 → Output: 2
Input: 999 → Output: 9
Input: 0 → Output: 0
```

**Quick Revision Notes:**
- Formula: N%9 (with N=0 and N%9=0 special cases)
- Also called digital root
- O(1) time with mathematical formula

---

### Q40: Number Pattern Printing

**Problem Statement:** Print pattern where row i has numbers 1 to i.

**Difficulty:** Easy

**Pattern:** Pattern printing

**Companies Asked:** CTS, TCS, Wipro, HCL

**Concepts Needed:** Nested loops

**Constraints:**
- 1 ≤ N ≤ 100

**Approach:** Outer loop for rows (1 to N), inner loop for columns (1 to row_num).

**Python Solution:**
```python
def print_number_pattern(n: int) -> None:
    for i in range(1, n + 1):
        for j in range(1, i + 1):
            print(j, end=' ')
        print()
```

**Dry Run:**
```
Input: n = 5
Row 1: 1
Row 2: 1 2
Row 3: 1 2 3
Row 4: 1 2 3 4
Row 5: 1 2 3 4 5
```

**Complexity:**
- Time: O(N²)
- Space: O(1)

**Common Mistakes:**
- Starting row from 0 instead of 1

**Edge Cases:**
- N = 0 → no output
- N = 1 → "1"

**Variations:** Floyd's triangle, Pascal's triangle

**Expected Output:**
```
N=4:
1
1 2
1 2 3
1 2 3 4
```

**Quick Revision Notes:**
- Outer loop: 1 to N (rows)
- Inner loop: 1 to row_number (columns)

---

### Pattern Summary (Q31-Q40)

| Pattern | Questions |
|---------|-----------|
| Hashing / Set | Q31, Q32, Q35, Q37 |
| Matrix operations | Q33 |
| Array traversal | Q34 |
| String manipulation | Q36, Q38 |
| Number theory | Q39 |
| Pattern printing | Q40 |

### Revision Table (Q31-Q40)

| Question | Pattern | Difficulty | Key Technique |
|----------|---------|------------|---------------|
| Q31: Intersection | Hashing | Medium | Set lookup |
| Q32: Find duplicate | Cycle detection | Medium | Floyd's algorithm |
| Q33: Matrix transpose | Matrix | Medium | Nested loops |
| Q34: Sorted & rotated | Array | Medium | Count descents |
| Q35: Pair with sum | Hashing | Medium | Complement in set |
| Q36: Longest prefix | String | Medium | Horizontal scanning |
| Q37: Valid anagram | Hashing | Medium | Count[26] array |
| Q38: Remove vowels | String | Easy | Set + join |
| Q39: Digital root | Math | Easy | N % 9 formula |
| Q40: Number pattern | Pattern | Easy | Nested loops |

### Important Observations (Q31-Q40)
- Hashing is the most common pattern in medium range
- String problems often reduce to character counting
- Mathematical formulas can optimize iterative problems


### Q41: Merge Two Sorted Arrays

**Problem Statement:** Merge two sorted arrays into one sorted array.

**Difficulty:** Hard

**Pattern:** Two pointers / Merge sort

**Companies Asked:** CTS, TCS, Accenture, Infosys, Wipro

**Concepts Needed:** Two-pointer technique

**Constraints:**
- 1 ≤ N, M ≤ 10^5

**Approach 1 (Brute Force):** Concatenate and sort. O((N+M) log(N+M)).

**Approach 2 (Two Pointers):** Compare smallest remaining, pick the smaller.

**Python Solution:**
```python
from typing import List


def merge_sorted(arr1: List[int], arr2: List[int]) -> List[int]:
    i = j = 0
    result = []
    while i < len(arr1) and j < len(arr2):
        if arr1[i] <= arr2[j]:
            result.append(arr1[i])
            i += 1
        else:
            result.append(arr2[j])
            j += 1
    result.extend(arr1[i:])
    result.extend(arr2[j:])
    return result
```

**Dry Run:**
```
Input: arr1 = [1, 3, 5, 7], arr2 = [2, 4, 6, 8]
Step 1: i=0,j=0 → 1≤2 → result=[1], i=1
Step 2: i=1,j=0 → 3>2 → result=[1,2], j=1
Step 3: i=1,j=1 → 3≤4 → result=[1,2,3], i=2
Step 4: i=2,j=1 → 5>4 → result=[1,2,3,4], j=2
Step 5: i=2,j=2 → 5≤6 → result=[1,2,3,4,5], i=3
Step 6: i=3,j=2 → 7>6 → result=[1,2,3,4,5,6], j=3
Step 7: i=3,j=3 → 7≤8 → result=[1,2,3,4,5,6,7], i=4
Step 8: extend arr2[3:] = [8]
Output: [1, 2, 3, 4, 5, 6, 7, 8]
```

**Complexity:**
- Time: O(N + M)
- Space: O(N + M)

**Common Mistakes:**
- Forgetting to extend remaining elements

**Edge Cases:**
- One array empty → return the other

**Variations:** Merge in-place, merge K sorted arrays

**Expected Output:**
```
Input: [1,3,5,7], [2,4,6,8] → Output: [1,2,3,4,5,6,7,8]
Input: [1,2], [3,4] → Output: [1,2,3,4]
```

**Quick Revision Notes:**
- Compare smallest remaining, pick smaller
- Append remaining after loop
- Core of merge sort algorithm

---

### Q42: Find Equilibrium Index

**Problem Statement:** Find index where left sum equals right sum.

**Difficulty:** Hard

**Pattern:** Prefix sum / Array

**Companies Asked:** CTS, TCS, Accenture, Infosys

**Concepts Needed:** Prefix sum, total sum

**Constraints:**
- 1 ≤ N ≤ 10^5

**Approach 1 (Brute Force):** For each index, calculate left and right sums. O(N²).

**Approach 2 (Optimized):** Total sum first. right_sum = total - left_sum - arr[i].

**Python Solution:**
```python
from typing import List


def equilibrium_index(arr: List[int]) -> int:
    total = sum(arr)
    left_sum = 0
    for i, num in enumerate(arr):
        right_sum = total - left_sum - num
        if left_sum == right_sum:
            return i
        left_sum += num
    return -1
```

**Dry Run:**
```
Input: arr = [1, 7, 3, 6, 5, 6]
Step 1: total = 28
Step 2: i=0, left=0, right=27 → 0≠27 → left=1
Step 3: i=1, left=1, right=20 → 1≠20 → left=8
Step 4: i=2, left=8, right=17 → 8≠17 → left=11
Step 5: i=3, left=11, right=11 → 11==11 → return 3
Output: 3
```

**Complexity:**
- Time: O(N)
- Space: O(1)

**Common Mistakes:**
- Including arr[i] in left or right sum incorrectly

**Edge Cases:**
- First index (left = 0) → check if right also 0
- Last index (right = 0)

**Variations:** Find pivot index

**Expected Output:**
```
Input: [1,7,3,6,5,6] → Output: 3
Input: [1,2,3] → Output: -1
```

**Quick Revision Notes:**
- right_sum = total - left_sum - arr[i]
- Return first matching index

---

### Q43: Maximum Product Subarray

**Problem Statement:** Find contiguous subarray with maximum product.

**Difficulty:** Hard

**Pattern:** Dynamic Programming

**Companies Asked:** CTS, TCS, Accenture, Infosys

**Concepts Needed:** DP, handling negatives

**Constraints:**
- 1 ≤ N ≤ 10^5
- -10 ≤ arr[i] ≤ 10

**Approach 1 (Brute Force):** Check all subarrays. O(N²).

**Approach 2 (DP):** Track both max_ending and min_ending (negative × min = max).

**Python Solution:**
```python
from typing import List


def max_product_subarray(arr: List[int]) -> int:
    if not arr:
        return 0
    max_ending = min_ending = max_so_far = arr[0]
    for i in range(1, len(arr)):
        num = arr[i]
        temp_max = max(num, max_ending * num, min_ending * num)
        min_ending = min(num, max_ending * num, min_ending * num)
        max_ending = temp_max
        max_so_far = max(max_so_far, max_ending)
    return max_so_far
```

**Dry Run:**
```
Input: arr = [2, 3, -2, 4]
Step 1: max_end=2, min_end=2, max_sofar=2
Step 2: i=1, num=3 → max_end=6, min_end=3, max_sofar=6
Step 3: i=2, num=-2 → max_end=-2, min_end=-12, max_sofar=6
Step 4: i=3, num=4 → max_end=4, min_end=-48, max_sofar=6
Output: 6 (subarray [2, 3])
```

**Complexity:**
- Time: O(N)
- Space: O(1)

**Common Mistakes:**
- Only tracking max (fails with negative numbers)

**Edge Cases:**
- All negative → largest or product of pairs
- Contains zero → product resets

**Variations:** Maximum sum subarray (Kadane's)

**Expected Output:**
```
Input: [2,3,-2,4] → Output: 6
Input: [-2,0,-1] → Output: 0
Input: [-2,3,4] → Output: 12
```

**Quick Revision Notes:**
- Track both max and min ending at each position
- Negative number can flip min to max

---

### Q44: Find Median of Two Sorted Arrays

**Problem Statement:** Find median of two sorted arrays.

**Difficulty:** Hard

**Pattern:** Binary Search / Divide and Conquer

**Companies Asked:** CTS, TCS, Accenture, Infosys

**Concepts Needed:** Binary search, partition

**Constraints:**
- 1 ≤ N, M ≤ 10^5
- Sorted arrays

**Approach 1 (Brute Force):** Merge and find median. O(N+M).

**Approach 2 (Binary Search):** Partition the smaller array. O(log min(N, M)).

**Python Solution:**
```python
from typing import List


def find_median_sorted_arrays(nums1: List[int], nums2: List[int]) -> float:
    if len(nums1) > len(nums2):
        nums1, nums2 = nums2, nums1
    m, n = len(nums1), len(nums2)
    low, high = 0, m
    while low <= high:
        partition1 = (low + high) // 2
        partition2 = (m + n + 1) // 2 - partition1
        max_left1 = float('-inf') if partition1 == 0 else nums1[partition1 - 1]
        min_right1 = float('inf') if partition1 == m else nums1[partition1]
        max_left2 = float('-inf') if partition2 == 0 else nums2[partition2 - 1]
        min_right2 = float('inf') if partition2 == n else nums2[partition2]
        if max_left1 <= min_right2 and max_left2 <= min_right1:
            if (m + n) % 2 == 0:
                return (max(max_left1, max_left2) + min(min_right1, min_right2)) / 2
            else:
                return max(max_left1, max_left2)
        elif max_left1 > min_right2:
            high = partition1 - 1
        else:
            low = partition1 + 1
    raise ValueError("Arrays not sorted")
```

**Dry Run:**
```
Input: nums1 = [1, 3], nums2 = [2]
m=2, n=1, low=0, high=2
Step 1: partition1=1, partition2=(3+1)//2 - 1 = 1
        max_left1=1, min_right1=3, max_left2=-∞, min_right2=2
        1≤2 ✓, -∞≤3 ✓ → total odd → max(1,-∞)=1
Output: 1.0
```

**Complexity:**
- Time: O(log min(N, M))
- Space: O(1)

**Common Mistakes:**
- Off-by-one in partition calculation

**Edge Cases:**
- One array empty
- Both arrays same size

**Variations:** Find kth element of two sorted arrays

**Expected Output:**
```
Input: [1,3], [2] → Output: 2.0
Input: [1,2], [3,4] → Output: 2.5
```

**Quick Revision Notes:**
- Partition the smaller array
- O(log min(N, M)) time
- Ensure all left elements ≤ all right elements

---

### Q45: Longest Substring Without Repeating Characters

**Problem Statement:** Find length of longest substring with all unique characters.

**Difficulty:** Hard

**Pattern:** Sliding window / Hashing

**Companies Asked:** CTS, TCS, Accenture, Infosys

**Concepts Needed:** Sliding window, hash map

**Constraints:**
- 0 ≤ S.length ≤ 10^5
- ASCII characters

**Approach 1 (Brute Force):** Check all substrings for uniqueness. O(N³).

**Approach 2 (Sliding Window):** Expand right, track last index of each char. Shrink window when duplicate found.

**Python Solution:**
```python
def length_of_longest_substring(s: str) -> int:
    char_index = {}
    max_len = left = 0
    for right, char in enumerate(s):
        if char in char_index and char_index[char] >= left:
            left = char_index[char] + 1
        char_index[char] = right
        max_len = max(max_len, right - left + 1)
    return max_len
```

**Dry Run:**
```
Input: s = "abcabcbb"
Step 1: right=0,'a' → {'a':0}, len=1, max=1
Step 2: right=1,'b' → {'a':0,'b':1}, len=2, max=2
Step 3: right=2,'c' → {'a':0,'b':1,'c':2}, len=3, max=3
Step 4: right=3,'a' → 'a' seen at 0 >= left=0 → left=1
         {'a':3}, len=3, max=3
Step 5: right=4,'b' → 'b' seen at 1 >= left=1 → left=2
         {'b':4}, len=3, max=3
Step 6: right=5,'c' → 'c' seen at 2 >= left=2 → left=3
         {'c':5}, len=3, max=3
Step 7: right=6,'b' → 'b' seen at 4 >= left=3 → left=5
         len=2, max=3
Step 8: right=7,'b' → 'b' seen at 6 >= left=5 → left=7
         len=1, max=3
Output: 3
```

**Complexity:**
- Time: O(N)
- Space: O(min(N, 26)) or O(1) for ASCII

**Common Mistakes:**
- Not checking if seen index >= left (out of window)

**Edge Cases:**
- Empty string → 0
- All unique → N
- All same → 1

**Variations:** Longest substring with at most K distinct chars

**Expected Output:**
```
Input: "abcabcbb" → Output: 3
Input: "bbbbb" → Output: 1
Input: "pwwkew" → Output: 3
```

**Quick Revision Notes:**
- Sliding window with hash map storing last index
- When duplicate found, move left past previous occurrence
- O(N) time

---

### Q46: Group Anagrams

**Problem Statement:** Group anagrams together from a list of strings.

**Difficulty:** Hard

**Pattern:** Hashing / String

**Companies Asked:** CTS, TCS, Accenture

**Concepts Needed:** Hash map, sorted string as key

**Constraints:**
- 1 ≤ N ≤ 10^4
- 0 ≤ S[i].length ≤ 100

**Approach 1 (Sorting Key):** Use sorted string as key in hash map.

**Approach 2 (Count Array Key):** Use tuple of 26 counts as key for O(L) per string.

**Python Solution:**
```python
from typing import List
from collections import defaultdict


def group_anagrams(strs: List[str]) -> List[List[str]]:
    groups = defaultdict(list)
    for s in strs:
        key = ''.join(sorted(s))
        groups[key].append(s)
    return list(groups.values())
```

**Dry Run:**
```
Input: strs = ["eat", "tea", "tan", "ate", "nat", "bat"]
Step 1: "eat" → sorted="aet" → groups["aet"] = ["eat"]
Step 2: "tea" → sorted="aet" → groups["aet"] = ["eat", "tea"]
Step 3: "tan" → sorted="ant" → groups["ant"] = ["tan"]
Step 4: "ate" → sorted="aet" → groups["aet"] = ["eat", "tea", "ate"]
Step 5: "nat" → sorted="ant" → groups["ant"] = ["tan", "nat"]
Step 6: "bat" → sorted="abt" → groups["abt"] = ["bat"]
Output: [["eat","tea","ate"], ["tan","nat"], ["bat"]]
```

**Complexity:**
- Time: O(N × K log K) where K is max string length
- Space: O(N × K)

**Common Mistakes:**
- Not using defaultdict for clean code

**Edge Cases:**
- Empty list → []
- Single string → [[string]]

**Variations:** Group anagrams using count array (O(N×K))

**Expected Output:**
```
Input: ["eat","tea","tan","ate","nat","bat"]
Output: [["eat","tea","ate"],["tan","nat"],["bat"]]
```

**Quick Revision Notes:**
- Sorted string as hash map key
- All anagrams produce same sorted string
- O(N × K log K) time

---

### Q47: Find All Duplicates in Array

**Problem Statement:** Find all elements that appear twice in an array of size N where values are in [1, N].

**Difficulty:** Hard

**Pattern:** Array / Hashing

**Companies Asked:** CTS, TCS, Accenture

**Concepts Needed:** Index marking, in-place

**Constraints:**
- 1 ≤ N ≤ 10^5
- Each element appears once or twice

**Approach 1 (Hash Map):** Count frequencies. O(N) space.

**Approach 2 (In-place):** Use value as index and mark by negating. If already negative, it's a duplicate.

**Python Solution:**
```python
from typing import List


def find_all_duplicates(arr: List[int]) -> List[int]:
    result = []
    for num in arr:
        idx = abs(num) - 1
        if arr[idx] < 0:
            result.append(abs(num))
        else:
            arr[idx] = -arr[idx]
    return result
```

**Dry Run:**
```
Input: arr = [4, 3, 2, 7, 8, 2, 3, 1]
Step 1: num=4 → idx=3, arr[3]=7 → mark arr[3]=-7
Step 2: num=3 → idx=2, arr[2]=2 → mark arr[2]=-2
Step 3: num=2 → idx=1, arr[1]=3 → mark arr[1]=-3
Step 4: num=7 → idx=6, arr[6]=3 → mark arr[6]=-3
Step 5: num=8 → idx=7, arr[7]=1 → mark arr[7]=-1
Step 6: num=2 → idx=1, arr[1]=-3 < 0 → duplicate 2
Step 7: num=3 → idx=2, arr[2]=-2 < 0 → duplicate 3
Step 8: num=1 → idx=0, arr[0]=4 → mark arr[0]=-4
Output: [2, 3]
```

**Complexity:**
- Time: O(N)
- Space: O(1) (excluding output)

**Common Mistakes:**
- Forgetting abs() when using value as index

**Edge Cases:**
- No duplicates → empty list

**Variations:** Find single duplicate, find missing numbers

**Expected Output:**
```
Input: [4,3,2,7,8,2,3,1] → Output: [2, 3]
Input: [1,1,2] → Output: [1]
```

**Quick Revision Notes:**
- Use value as index, mark by negating
- If already negative, it's duplicate
- O(1) extra space

---

### Q48: Next Permutation

**Problem Statement:** Find next lexicographically greater permutation of array.

**Difficulty:** Hard

**Pattern:** Array / Permutation

**Companies Asked:** CTS, TCS, Accenture

**Concepts Needed:** Rearrangement, reverse

**Constraints:**
- 1 ≤ N ≤ 10^5

**Approach 1 (Brute Force):** Generate all permutations, find next.

**Approach 2 (Optimized):** Find first decreasing element from right. Swap with next larger. Reverse suffix.

**Python Solution:**
```python
from typing import List


def next_permutation(arr: List[int]) -> List[int]:
    n = len(arr)
    i = n - 2
    while i >= 0 and arr[i] >= arr[i + 1]:
        i -= 1
    if i >= 0:
        j = n - 1
        while arr[j] <= arr[i]:
            j -= 1
        arr[i], arr[j] = arr[j], arr[i]
    arr[i + 1:] = reversed(arr[i + 1:])
    return arr
```

**Dry Run:**
```
Input: [1, 2, 3]
Step 1: i=1, arr[1]=2 < arr[2]=3 → don't decrement
Step 2: i>=0 → j=2, arr[2]=3 > arr[1]=2 → swap(1,2) → [1, 3, 2]
Step 3: reverse from index 2 (unchanged)
Output: [1, 3, 2]

Input: [3, 2, 1]
Step 1: i=1, arr[1]=2 >= arr[2]=1 → i=0, arr[0]=3 >= arr[1]=2 → i=-1
Step 2: i<0 → skip swap
Step 3: reverse entire array → [1, 2, 3]
Output: [1, 2, 3]
```

**Complexity:**
- Time: O(N)
- Space: O(1)

**Common Mistakes:**
- Using > instead of >= in first while (handles duplicates)

**Edge Cases:**
- Last permutation → reverse to first
- Single element → same array

**Variations:** Previous permutation

**Expected Output:**
```
Input: [1,2,3] → Output: [1,3,2]
Input: [3,2,1] → Output: [1,2,3]
Input: [1,1,5] → Output: [1,5,1]
```

**Quick Revision Notes:**
- Find first decreasing element from right
- Swap with next larger element from right
- Reverse the suffix

---

### Q49: Trapping Rain Water

**Problem Statement:** Calculate water trapped between bars of varying heights.

**Difficulty:** Hard

**Pattern:** Two pointers / Dynamic Programming

**Companies Asked:** CTS, TCS, Accenture, Infosys

**Concepts Needed:** Two pointers, prefix/suffix max

**Constraints:**
- 1 ≤ N ≤ 10^5
- 0 ≤ height[i] ≤ 10^5

**Approach 1 (Brute Force):** For each bar, find left max and right max. O(N²).

**Approach 2 (Two Pointers):** Track left_max and right_max. Process from both ends.

**Python Solution:**
```python
from typing import List


def trap_rain_water(height: List[int]) -> int:
    if not height:
        return 0
    left, right = 0, len(height) - 1
    left_max = right_max = water = 0
    while left < right:
        if height[left] < height[right]:
            if height[left] >= left_max:
                left_max = height[left]
            else:
                water += left_max - height[left]
            left += 1
        else:
            if height[right] >= right_max:
                right_max = height[right]
            else:
                water += right_max - height[right]
            right -= 1
    return water
```

**Dry Run:**
```
Input: [0, 1, 0, 2, 1, 0, 1, 3, 2, 1, 2, 1]
Step 1: left=0, right=11, left_max=0, right_max=0, water=0
Step 2: h[0]=0 < h[11]=1, 0>=0 → left_max=0, left=1
Step 3: h[1]=1 < h[11]=1, 1>=0 → left_max=1, left=2
Step 4: h[2]=0 < h[11]=1, 0<1 → water+=1-0=1, left=3
Step 5: h[3]=2 > h[11]=1, h[11]=1<2, 1<0→actually process right...
(continues)
Output: 6
```

**Complexity:**
- Time: O(N)
- Space: O(1)

**Common Mistakes:**
- Not tracking water when height[left/right] < respective max

**Edge Cases:**
- Empty array → 0
- Less than 3 bars → 0
- No valley → 0

**Variations:** Container with most water

**Expected Output:**
```
Input: [0,1,0,2,1,0,1,3,2,1,2,1] → Output: 6
Input: [4,2,0,3,2,5] → Output: 9
```

**Quick Revision Notes:**
- Two-pointer approach from both ends
- Water trapped = min(left_max, right_max) - height[current]
- O(N) time, O(1) space

---

### Q50: Find Celebrity in a Party

**Problem Statement:** Find the celebrity (known by everyone, knows no one) in minimum queries.

**Difficulty:** Hard

**Pattern:** Graph / Elimination

**Companies Asked:** CTS, TCS, Accenture

**Concepts Needed:** Elimination strategy

**Constraints:**
- N people labeled 0 to N-1
- knows(a, b) API returns True if a knows b

**Approach 1 (Brute Force):** For each person, check if they know anyone and if everyone knows them. O(N²) queries.

**Approach 2 (Elimination):** Use two-pointer elimination. Candidate starts at 0. For each i from 1 to N-1, if candidate knows i, update candidate to i. Then verify candidate.

**Python Solution:**
```python
from typing import List


def find_celebrity(n: int, knows: callable) -> int:
    candidate = 0
    for i in range(1, n):
        if knows(candidate, i):
            candidate = i
    for i in range(n):
        if i != candidate:
            if knows(candidate, i) or not knows(i, candidate):
                return -1
    return candidate
```

**Dry Run:**
```
Assume party of 4 people, celebrity is person 2:
knows(0,1)=T, knows(0,2)=T, knows(0,3)=T
knows(1,0)=F, knows(1,2)=T, knows(1,3)=T
knows(2,0)=F, knows(2,1)=F, knows(2,3)=F
knows(3,0)=F, knows(3,1)=F, knows(3,2)=T

Phase 1 - Elimination:
Step 1: candidate=0, i=1 → knows(0,1)=T → candidate=1
Step 2: candidate=1, i=2 → knows(1,2)=T → candidate=2
Step 3: candidate=2, i=3 → knows(2,3)=F → keep candidate=2

Phase 2 - Verification:
Check 2: knows(2,0)=F ✓, knows(2,1)=F ✓, knows(2,3)=F ✓
Check 0: knows(0,2)=T ✓
Check 1: knows(1,2)=T ✓
Check 3: knows(3,2)=T ✓
Output: 2
```

**Complexity:**
- Time: O(N) queries (elimination + verification)
- Space: O(1)

**Common Mistakes:**
- Not verifying candidate (elimination only narrows down to 1 candidate)

**Edge Cases:**
- No celebrity → -1
- Celebrity at index 0

**Variations:** Find celebrity with different knowledge representation

**Expected Output:**
```
If celebrity exists → celebrity index
If no celebrity → -1
```

**Quick Revision Notes:**
- Elimination phase: if candidate knows i, update to i
- Verification phase: check candidate knows no one, everyone knows candidate
- O(N) time, O(1) space

---

### Pattern Summary (Q41-Q50)

| Pattern | Questions |
|---------|-----------|
| Two pointers / Merge | Q41, Q49 |
| Prefix sum | Q42 |
| Dynamic Programming | Q43 |
| Binary Search | Q44 |
| Sliding window | Q45 |
| Hashing | Q46, Q47 |
| Array / Permutation | Q48 |
| Graph / Elimination | Q50 |

### Revision Table (Q41-Q50)

| Question | Pattern | Difficulty | Key Technique |
|----------|---------|------------|---------------|
| Q41: Merge sorted arrays | Two pointers | Hard | Compare & advance |
| Q42: Equilibrium index | Prefix sum | Hard | Total - left - current |
| Q43: Max product subarray | DP | Hard | Track min too |
| Q44: Median of two arrays | Binary search | Hard | Partition smaller |
| Q45: Longest substring | Sliding window | Hard | Char index map |
| Q46: Group anagrams | Hashing | Hard | Sorted string key |
| Q47: All duplicates | Marking | Hard | Negate index |
| Q48: Next permutation | Permutation | Hard | Find decreasing |
| Q49: Trapping rain water | Two pointers | Hard | Left/right max |
| Q50: Find celebrity | Elimination | Hard | Candidate elimination |

### Important Observations (Q41-Q50)
- Hard problems often combine multiple patterns
- Two-pointer is dominant in hard problems
- O(N) time with O(1) space is the gold standard
- Verification step is common after elimination phase


## Final Sections

### Complete Revision Table (All 50 Questions)

| Q# | Question | Pattern | Difficulty | Key Technique |
|----|----------|---------|------------|---------------|
| 1 | Reverse an array | Two pointers | Easy | Swap left/right |
| 2 | Check palindrome | Two pointers | Easy | Skip non-alnum |
| 3 | Factorial of a number | Iteration | Easy | Loop multiply |
| 4 | Generate Fibonacci | DP/Iteration | Easy | a, b = b, a+b |
| 5 | Check prime number | Number theory | Easy | √N optimization |
| 6 | Find GCD | Euclidean | Easy | a, b = b, a%b |
| 7 | Reverse words in string | String | Easy | split() + join() |
| 8 | Check Armstrong number | Digit manipulation | Easy | digit^power sum |
| 9 | Second largest in array | Array | Easy | Two variables |
| 10 | Print star pyramid | Pattern | Easy | Spaces + stars |
| 11 | Check perfect number | Number theory | Easy | Divisor sum √N |
| 12 | Remove duplicates | Two pointers | Easy | j pointer for unique |
| 13 | Find frequency of each | Hashing | Easy | dict.get() |
| 14 | Matrix addition | Matrix | Easy | Nested loops |
| 15 | Linear search | Searching | Easy | Enumerate |
| 16 | Binary search | Divide & conquer | Medium | Mid-point halving |
| 17 | Balanced parentheses | Stack | Medium | Mapping dict |
| 18 | Majority element | Voting | Medium | Boyer-Moore |
| 19 | Subarray with given sum | Sliding window | Medium | Expand/shrink |
| 20 | Wave array | Rearrangement | Medium | Adjacent swap |
| 21 | Chocolate distribution | Sliding window | Medium | Sort + window |
| 22 | Kth smallest element | Heap | Medium | heapify + pop K |
| 23 | Stock buy and sell | DP | Medium | Track min price |
| 24 | Move zeroes to end | Two pointers | Medium | Swap non-zero |
| 25 | Find leaders in array | Reverse traversal | Medium | Right-to-left max |
| 26 | Check two strings anagram | Hashing | Medium | Count[26] array |
| 27 | First non-repeating char | Hashing | Medium | Two pass |
| 28 | Count and say | String | Medium | Run-length encoding |
| 29 | Maximum subarray sum | DP | Medium | max_ending_here |
| 30 | Find missing number | Math | Medium | Sum formula |
| 31 | Intersection of arrays | Hashing | Medium | Set lookup |
| 32 | Find duplicate in array | Cycle detection | Medium | Floyd's algorithm |
| 33 | Matrix transpose | Matrix | Medium | Nested loops |
| 34 | Sorted and rotated array | Array | Medium | Count descents |
| 35 | Pairs with given sum | Hashing | Medium | Complement in set |
| 36 | Longest common prefix | String | Medium | Horizontal scanning |
| 37 | Valid anagram check | Hashing | Medium | Count[26] array |
| 38 | Remove vowels from string | String | Easy | Set + join |
| 39 | Sum of digits until single | Math | Easy | N % 9 formula |
| 40 | Number pattern printing | Pattern | Easy | Nested loops |
| 41 | Merge two sorted arrays | Two pointers | Hard | Compare & advance |
| 42 | Find equilibrium index | Prefix sum | Hard | Total - left - current |
| 43 | Maximum product subarray | DP | Hard | Track min too |
| 44 | Median of two arrays | Binary search | Hard | Partition smaller |
| 45 | Longest substring unique | Sliding window | Hard | Char index map |
| 46 | Group anagrams | Hashing | Hard | Sorted string key |
| 47 | All duplicates in array | Marking | Hard | Negate index |
| 48 | Next permutation | Permutation | Hard | Find decreasing |
| 49 | Trapping rain water | Two pointers | Hard | Left/right max |
| 50 | Find celebrity | Elimination | Hard | Candidate elimination |

---

### Frequently Repeated Questions for CTS (Top 10)

These questions have historically appeared most frequently in CTS coding assessments:

1. **Q1: Reverse an array** — Almost every CTS assessment tests basic array operations
2. **Q4: Generate Fibonacci series** — Classic recursion/iteration question
3. **Q5: Check prime number** — Number theory basics are frequently tested
4. **Q9: Second largest in array** — Tests array traversal without sorting
5. **Q2: Check palindrome** — String manipulation fundamentals
6. **Q18: Majority element** — Boyer-Moore algorithm impresses interviewers
7. **Q24: Move zeroes to end** — In-place array modification is commonly asked
8. **Q29: Maximum subarray sum (Kadane's)** — Dynamic programming classic
9. **Q16: Binary search** — Search algorithm fundamentals
10. **Q41: Merge two sorted arrays** — Merge sort core concept

---

### Must Practice Before Interview (Priority List)

**Priority 1 (Very High - Almost Always Asked):**
- String reversal, palindrome check, reverse words
- Array reverse, rotate, find second largest
- Fibonacci, factorial, prime check
- Binary search and linear search
- Frequency counting and anagrams

**Priority 2 (High - Frequently Asked):**
- Balanced parentheses using stack
- Kadane's algorithm (max subarray sum)
- Move zeroes to end
- Remove duplicates from sorted array
- Missing number in array
- Stock buy and sell (one transaction)
- Find leaders in array

**Priority 3 (Medium - Occasionally Asked):**
- Chocolate distribution problem
- Wave array
- Matrix operations (add, transpose)
- Pattern printing
- First non-repeating character
- Count and say sequence
- Longest common prefix

**Priority 4 (Low - Rarely Asked but Good to Know):**
- Trap rain water
- Next permutation
- Median of two sorted arrays
- Find celebrity
- Maximum product subarray

---

### Most Important Python Tricks for CTS

**1. Swapping Variables (Reversal, Sorting)**
```python
# Swap without temp variable
a, b = b, a
arr[left], arr[right] = arr[right], arr[left]
```

**2. List Comprehension (Filtering, Transformation)**
```python
# Filter vowels
''.join(c for c in s if c not in vowels)
# Square all numbers
squares = [x**2 for x in nums]
```

**3. Dictionary Shortcuts (Frequency Counting)**
```python
freq[num] = freq.get(num, 0) + 1
# Using default dict
from collections import defaultdict, Counter
counter = Counter(arr)  # Most frequent element: counter.most_common(1)
```

**4. String Split and Join (Word Reversal, Cleaning)**
```python
words = s.split()  # split on any whitespace
reversed_str = ' '.join(reversed(words))
```

**5. Two Pointer Technique (In-place Operations)**
```python
# Reverse array
left, right = 0, len(arr) - 1
while left < right:
    arr[left], arr[right] = arr[right], arr[left]
    left += 1
    right -= 1
```

**6. Enumerate (Index + Value)**
```python
for i, num in enumerate(arr):
    # i is index, num is value
```

**7. Zip for Parallel Iteration**
```python
for a, b in zip(arr1, arr2):
    # iterate two arrays simultaneously
```

**8. Math.isqrt for Integer Square Root**
```python
import math
limit = int(math.isqrt(n))  # more precise than int(n**0.5)
```

**9. Slicing (Reverse, Copy, etc.)**
```python
arr[::-1]  # reverse copy
arr[i:j]   # slice from i to j
arr[::2]   # every other element
```

**10. Set Operations (Intersection, Union, Uniqueness)**
```python
# Remove duplicates from list
list(set(arr))
# Intersection of two arrays
set(arr1) & set(arr2)
# Check if element exists in O(1)
if x in seen_set:
```

**11. Python One-Liner (Digital Root)**
```python
digital_root = 0 if n == 0 else (9 if n % 9 == 0 else n % 9)
```

**12. Efficient Loop for Prime Checking (6k±1)**
```python
for i in range(5, int(math.isqrt(n)) + 1, 6):
    if n % i == 0 or n % (i + 2) == 0:
```

---

### Interview Checklist

**Before the Interview (1 Week Prior):**
- [ ] Review all 50 questions in this guide
- [ ] Practice coding without IDE (use HackerRank editor)
- [ ] Time yourself: solve easy problems in 10 min, medium in 20 min, hard in 30 min
- [ ] Set up your HackerRank/AMCAT profile if needed
- [ ] Review Python syntax for type hints, list/dict/set operations

**Before the Interview (1 Day Prior):**
- [ ] Learn/revise all Python tricks listed above
- [ ] Practice dry runs — explain code line by line
- [ ] Prepare your introduction (1 min)
- [ ] Check internet connection and webcam
- [ ] Charge your laptop and keep charger handy

**During the Interview:**
- [ ] Read problem statement carefully (2-3 times)
- [ ] Ask clarifying questions to the interviewer
- [ ] Discuss brute force approach first, then optimize
- [ ] Write clean, readable code with proper variable names
- [ ] Comment your code in critical sections
- [ ] Run through a dry run with sample input
- [ ] Mention time and space complexity
- [ ] Test with edge cases (empty input, single element, large values)
- [ ] If stuck, communicate your thought process
- [ ] Don't panic — take deep breaths and think step by step

**Coding Round Tips Specific to CTS:**
- ✅ Focus on basic data structures (arrays, strings, hash maps)
- ✅ Write clean, PEP-8 compliant Python code
- ✅ Cover edge cases explicitly during dry run
- ✅ Use Python's built-in functions wisely (sorted(), reversed(), enumerate())
- ❌ Don't over-engineer solutions
- ❌ Don't use obscure libraries or advanced algorithms
- ❌ Don't forget to handle empty/null inputs

**Post-Interview:**
- [ ] Thank the interviewer
- [ ] Note down questions you struggled with for future prep
- [ ] Send a thank-you email within 24 hours

---

### Pattern Distribution Analysis

```
Pattern Distribution Across 50 Questions:
─────────────────────────────────────────────────
Array / Two Pointers    ████████████████  (16%)   8 questions
String Manipulation     ██████████████    (14%)   7 questions
Hashing / Frequency     ██████████████    (14%)   7 questions
Number Theory / Math    ██████████        (10%)   5 questions
Dynamic Programming     ████████          (8%)    4 questions
Searching              ██████            (6%)    3 questions
Sliding Window         ██████            (6%)    3 questions
Pattern Printing       ██████            (6%)    3 questions
Matrix Operations      ██████            (6%)    3 questions
Stack                  ████              (4%)    2 questions
Heap                   ████              (4%)    2 questions
Misc (Graph, etc.)     ██████            (6%)    3 questions
```

### Tips Based on Pattern Frequency
1. **Array + Two Pointer** is the most important pattern (16%) — master it
2. **String + Hashing** together cover 28% of questions
3. **Number Theory** basics (prime, GCD, Armstrong) are CTS favorites
4. **DP** questions in CTS are limited to Kadane's and stock problems
5. Matrix questions are usually limited to basic operations
6. All questions are solvable without advanced data structures

---

*This comprehensive guide covers all 50 Python coding questions commonly asked in CTS (Cognizant) coding interviews. Practice regularly, focus on understanding patterns, and always communicate your approach clearly during the interview. Good luck!*

---
**Author:** Tamilselvan S
**LinkedIn:** https://www.linkedin.com/in/tamilselvan-ai/
**GitHub:** `your-github-username`
---

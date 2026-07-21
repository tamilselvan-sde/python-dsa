# Wipro Coding Interview Preparation

## Company Overview

| Detail | Information |
|--------|------------|
| **Interview Process** | Online Assessment → Technical Round 1 → Technical Round 2 → HR |
| **Difficulty** | Easy to Medium (focus on fundamentals) |
| **Coding Round Pattern** | 2-3 coding questions in 60-90 minutes |
| **Duration** | 60-90 minutes for coding assessment |
| **Platform** | Wipro own platform / HackerRank / Mettl |
| **Tips** | Focus on array/string manipulation, basic DSA, Python-specific features |

---

## Question #1: Reverse a String

### Problem Statement
Write a program to reverse a given string without using built-in reverse functions.

### Difficulty
Easy

### Pattern
Strings, Two Pointers

### Companies Asked
Wipro, TCS, Infosys, Accenture

### Concepts Needed
- String indexing
- Two-pointer technique
- Mutable vs immutable strings

### Constraints
- 1 ≤ string length ≤ 10^5
- String contains only printable ASCII characters

### Approach 1: Brute Force
Convert string to list, create a new list by iterating backwards, then join.

**Time Complexity:** O(n)
**Space Complexity:** O(n)

### Approach 2: Optimized (Two Pointers)
Convert string to list. Use two pointers (left at start, right at end). Swap characters and move pointers toward center. Join the list at the end.

**Time Complexity:** O(n)
**Space Complexity:** O(n) — Python strings are immutable, so we need a list

### Python Solution
```python
def reverse_string(s: str) -> str:
    """
    Reverses a string using two-pointer technique.
    
    Approach: Convert to list, swap characters from both ends
    moving toward center.
    
    Args:
        s: Input string
        
    Returns:
        Reversed string
    """
    chars = list(s)
    left, right = 0, len(chars) - 1
    
    while left < right:
        chars[left], chars[right] = chars[right], chars[left]
        left += 1
        right -= 1
    
    return "".join(chars)
```

### Dry Run
```
Input: "WIPRO"

Initial: chars = ['W', 'I', 'P', 'R', 'O'], left=0, right=4

Step 1: swap chars[0] and chars[4] → ['O', 'I', 'P', 'R', 'W'], left=1, right=3
Step 2: swap chars[1] and chars[3] → ['O', 'R', 'P', 'I', 'W'], left=2, right=2
Step 3: left < right is False → exit loop

Result: "ORPIW"
```

### Complexity
- **Time:** O(n) — single pass through half the string
- **Space:** O(n) — for the list conversion

### Common Mistakes
- Trying to modify string in place (strings are immutable in Python)
- Off-by-one errors in loop condition
- Not handling empty string or single character input

### Edge Cases
- Empty string → returns empty string
- Single character → returns same string
- String with spaces → spaces should be preserved in correct positions
- Unicode characters → works correctly

### Variations
- Reverse words in a sentence
- Reverse only vowels in a string
- Reverse each word in a sentence individually

### Follow-up Questions
- Can you do it without using extra space for the list?
- How would you reverse a string recursively?

### Interview Tips
Wipro interviewers check for understanding of immutability. Explain why you convert to list first.

### Expected Output
```
Input: "WIPRO"
Output: "ORPIW"

Input: "hello"
Output: "olleh"
```

### Quick Revision Notes
- Strings are immutable → convert to list first
- Two-pointer technique: one from left, one from right
- Swap until pointers cross

---

## Question #2: Check Palindrome

### Problem Statement
Check if a given string is a palindrome. A palindrome reads the same forwards and backwards.

### Difficulty
Easy

### Pattern
Strings, Two Pointers

### Companies Asked
Wipro, TCS, Cognizant, Capgemini

### Concepts Needed
- String traversal
- Two-pointer technique
- Palindrome definition

### Constraints
- 1 ≤ string length ≤ 10^5
- String contains alphanumeric characters only

### Approach 1: Brute Force
Reverse the string and compare with original.

**Time Complexity:** O(n)
**Space Complexity:** O(n)

### Approach 2: Optimized (Two Pointers)
Use two pointers, one from start and one from end. Compare characters and move inward. If any mismatch occurs, return False.

**Time Complexity:** O(n)
**Space Complexity:** O(1)

### Python Solution
```python
def is_palindrome(s: str) -> bool:
    """
    Checks if a string is a palindrome using two-pointer technique.
    
    Args:
        s: Input string
        
    Returns:
        True if palindrome, False otherwise
    """
    left, right = 0, len(s) - 1
    
    while left < right:
        if s[left] != s[right]:
            return False
        left += 1
        right -= 1
    
    return True
```

### Dry Run
```
Input: "MADAM"

Initial: left=0, right=4

Step 1: s[0]='M' vs s[4]='M' → match, left=1, right=3
Step 2: s[1]='A' vs s[3]='A' → match, left=2, right=2
Step 3: left < right is False → exit loop

Result: True (palindrome)

Input: "WIPRO"

Step 1: s[0]='W' vs s[4]='O' → mismatch → return False

Result: False (not palindrome)
```

### Complexity
- **Time:** O(n)
- **Space:** O(1)

### Common Mistakes
- Not handling case sensitivity
- Not handling spaces or punctuation properly
- Off-by-one errors in loop condition

### Edge Cases
- Empty string → True
- Single character → True
- Odd length strings → middle character doesn't need a match

### Variations
- Check if integer is palindrome
- Check palindrome ignoring spaces/punctuation
- Find longest palindromic substring

### Interview Tips
Wipro often asks this as a warm-up. You can also solve with slicing `s == s[::-1]` but two-pointer is more efficient.

### Quick Revision Notes
- Palindrome reads same forward and backward
- Two-pointer comparison: left vs right
- Can use slicing for quick check

---

## Question #3: Factorial of a Number

### Problem Statement
Find the factorial of a given non-negative integer n.

### Difficulty
Easy

### Pattern
Math, Recursion

### Companies Asked
Wipro, TCS, Infosys, HCL

### Concepts Needed
- Recursion basics
- Iteration
- Factorial definition: 0! = 1

### Constraints
- 0 ≤ n ≤ 20

### Approach 1: Iterative
Multiply numbers from 1 to n sequentially.

**Time Complexity:** O(n)
**Space Complexity:** O(1)

### Approach 2: Recursive
Use recurrence: fact(n) = n × fact(n-1) with base case fact(0) = 1.

**Time Complexity:** O(n)
**Space Complexity:** O(n) — recursion stack

### Python Solution
```python
def factorial_iterative(n: int) -> int:
    """
    Computes factorial of n using iteration.
    
    Args:
        n: Non-negative integer
        
    Returns:
        Factorial of n
    """
    if n < 0:
        raise ValueError("Factorial not defined for negative numbers")
    result = 1
    for i in range(2, n + 1):
        result *= i
    return result


def factorial_recursive(n: int) -> int:
    """
    Computes factorial of n using recursion.
    """
    if n < 0:
        raise ValueError("Factorial not defined for negative numbers")
    if n == 0 or n == 1:
        return 1
    return n * factorial_recursive(n - 1)
```

### Dry Run
```
Input: n = 5

Iterative:
result = 1
i=2: result = 2
i=3: result = 6
i=4: result = 24
i=5: result = 120

Output: 120

Recursive:
factorial_recursive(5)
= 5 × factorial_recursive(4)
= 5 × 4 × factorial_recursive(3)
= 5 × 4 × 3 × factorial_recursive(2)
= 5 × 4 × 3 × 2 × factorial_recursive(1)
= 5 × 4 × 3 × 2 × 1 = 120
```

### Complexity
- **Time:** O(n)
- **Space:** O(1) iterative, O(n) recursive

### Common Mistakes
- Forgetting that 0! = 1
- Recursion without base case causes stack overflow

### Edge Cases
- n = 0 → returns 1
- n = 1 → returns 1
- Negative input → should raise error

### Variations
- Trailing zeroes in factorial
- nCr using factorial

### Interview Tips
Show both iterative and recursive. Mention Python handles arbitrarily large integers natively.

### Expected Output
```
Input: 5
Output: 120

Input: 0
Output: 1
```

### Quick Revision Notes
- 0! = 1
- Iterative is space efficient; recursive is elegant
- Python handles big integers

---

## Question #4: Fibonacci Series

### Problem Statement
Generate the Fibonacci series up to n terms.

### Difficulty
Easy

### Pattern
Math, Iteration

### Companies Asked
Wipro, TCS, Cognizant, Accenture

### Concepts Needed
- Fibonacci definition: F(0)=0, F(1)=1, F(n)=F(n-1)+F(n-2)
- Loop constructs

### Constraints
- 1 ≤ n ≤ 100

### Approach 1: Iterative
Start with a=0, b=1. For each step, compute next = a + b, then shift.

**Time Complexity:** O(n)
**Space Complexity:** O(n) for list, O(1) if just printing

### Approach 2: Recursive (with memoization)
**Time Complexity:** O(n)
**Space Complexity:** O(n)

### Python Solution
```python
from typing import List


def fibonacci_series(n: int) -> List[int]:
    """
    Generates first n terms of Fibonacci series.
    
    Args:
        n: Number of terms
        
    Returns:
        List containing first n Fibonacci numbers
    """
    if n <= 0:
        return []
    
    fib = [0] * n
    if n >= 1:
        fib[0] = 0
    if n >= 2:
        fib[1] = 1
    
    for i in range(2, n):
        fib[i] = fib[i - 1] + fib[i - 2]
    
    return fib
```

### Dry Run
```
Input: n = 7

fib = [0, 0, 0, 0, 0, 0, 0]
fib[0] = 0, fib[1] = 1

i=2: fib[2] = 0 + 1 = 1
i=3: fib[3] = 1 + 1 = 2
i=4: fib[4] = 1 + 2 = 3
i=5: fib[5] = 2 + 3 = 5
i=6: fib[6] = 3 + 5 = 8

Result: [0, 1, 1, 2, 3, 5, 8]
```

### Complexity
- **Time:** O(n)
- **Space:** O(n)

### Common Mistakes
- Starting with 1, 1 instead of 0, 1
- Off-by-one in number of terms

### Edge Cases
- n = 0 → empty list
- n = 1 → [0]
- n = 2 → [0, 1]

### Variations
- Find nth Fibonacci
- Sum of even Fibonacci numbers

### Interview Tips
Wipro expects both iterative and recursive. Iterative is preferred.

### Expected Output
```
Input: n = 7
Output: [0, 1, 1, 2, 3, 5, 8]
```

### Quick Revision Notes
- F(0)=0, F(1)=1
- Each term is sum of previous two
- Iterative is O(n) time, O(1) space

---

## Question #5: Prime Number Check

### Problem Statement
Check if a given number is prime.

### Difficulty
Easy

### Pattern
Math, Number Theory

### Companies Asked
Wipro, TCS, Infosys, HCL, Cognizant

### Concepts Needed
- Prime number definition
- Modulo operation
- Optimization: check up to √n

### Constraints
- 1 ≤ n ≤ 10^9

### Approach 1: Brute Force
Check divisibility from 2 to n-1.

**Time Complexity:** O(n)
**Space Complexity:** O(1)

### Approach 2: Optimized
Check up to √n. Handle 2 separately, then skip even numbers.

**Time Complexity:** O(√n)
**Space Complexity:** O(1)

### Python Solution
```python
import math


def is_prime(n: int) -> bool:
    """
    Checks if a number is prime using optimized trial division.
    
    Args:
        n: Integer to check
        
    Returns:
        True if prime, False otherwise
    """
    if n <= 1:
        return False
    if n <= 3:
        return True
    if n % 2 == 0 or n % 3 == 0:
        return False
    
    for i in range(5, int(math.sqrt(n)) + 1, 6):
        if n % i == 0 or n % (i + 2) == 0:
            return False
    return True
```

### Dry Run
```
Input: n = 29

Step 1: n > 1 → continue
Step 2: n > 3 → continue
Step 3: 29%2=1, 29%3=2 → continue
Step 4: sqrt(29) ≈ 5.38, loop i=5:
  i=5: 29%5=4, 29%7=1 → both non-zero
Step 5: loop ends → return True

Result: True (29 is prime)

Input: n = 15

15 > 1, 15 > 3, 15%2=1, 15%3=0 → return False

Result: False (15 is not prime)
```

### Complexity
- **Time:** O(√n)
- **Space:** O(1)

### Common Mistakes
- Forgetting that 1 is not prime
- Not handling 2 (the only even prime)

### Edge Cases
- n = 1 → False
- n = 2 → True (smallest prime)

### Variations
- Sieve of Eratosthenes
- Find nth prime

### Interview Tips
Wipro looks for the √n optimization. Always explain why checking up to sqrt(n) is sufficient.

### Expected Output
```
Input: 29
Output: True

Input: 15
Output: False
```

### Quick Revision Notes
- 1 is not prime
- 2 is the only even prime
- Check only up to √n

---

## Question #6: Armstrong Number

### Problem Statement
Check if a number is an Armstrong number (sum of each digit raised to power of number of digits equals the number).

### Difficulty
Easy

### Pattern
Math, Number Theory

### Companies Asked
Wipro, TCS, Cognizant

### Concepts Needed
- Armstrong number definition
- Digit extraction

### Constraints
- 0 ≤ n ≤ 10^6

### Approach
Extract each digit, count digits, compute sum of digit^k, compare.

**Time Complexity:** O(d) where d is number of digits
**Space Complexity:** O(1)

### Python Solution
```python
def is_armstrong(n: int) -> bool:
    """
    Checks if a number is an Armstrong number.
    
    A k-digit number is Armstrong if sum of each digit
    raised to power k equals the number.
    
    Args:
        n: Integer to check
        
    Returns:
        True if Armstrong number, False otherwise
    """
    if n < 0:
        return False
    
    num_str = str(n)
    k = len(num_str)
    total = sum(int(digit) ** k for digit in num_str)
    return total == n
```

### Dry Run
```
Input: n = 153

k = 3
digits: 1, 5, 3
total = 1³ + 5³ + 3³ = 1 + 125 + 27 = 153
153 == 153 → True

Result: True (Armstrong number)

Input: n = 123

total = 1³ + 2³ + 3³ = 1 + 8 + 27 = 36
36 != 123 → False
```

### Complexity
- **Time:** O(log₁₀ n)
- **Space:** O(1)

### Common Mistakes
- Not counting digits correctly
- Forgetting single-digit numbers (0-9 are all Armstrong)

### Edge Cases
- n = 0 → True
- n = 1 → True

### Variations
- Find all Armstrong numbers in a range

### Interview Tips
Wipro frequently asks this. Know that 153, 370, 371, 407 are the 3-digit Armstrong numbers.

### Expected Output
```
Input: 153
Output: True

Input: 123
Output: False
```

### Quick Revision Notes
- Armstrong: sum of each digit^(number of digits)
- 3-digit Armstrong numbers: 153, 370, 371, 407
- Single digits 0-9 are all Armstrong

---

## Question #7: Count Vowels and Consonants

### Problem Statement
Count vowels and consonants in a given string.

### Difficulty
Easy

### Pattern
Strings

### Companies Asked
Wipro, TCS, Cognizant, Accenture

### Concepts Needed
- String traversal
- Character sets

### Constraints
- 1 ≤ string length ≤ 10^5

### Approach
Traverse string, convert to lowercase, check if character is vowel.

**Time Complexity:** O(n)
**Space Complexity:** O(1)

### Python Solution
```python
def count_vowels_consonants(s: str) -> tuple:
    """
    Counts vowels and consonants in a string.
    
    Args:
        s: Input string
        
    Returns:
        Tuple of (vowel_count, consonant_count)
    """
    vowels = set("aeiou")
    vowel_count = 0
    consonant_count = 0
    
    for ch in s.lower():
        if ch.isalpha():
            if ch in vowels:
                vowel_count += 1
            else:
                consonant_count += 1
    
    return vowel_count, consonant_count
```

### Dry Run
```
Input: "Wipro"

Lowercase: "wipro"
w → consonant → 1
i → vowel → 1
p → consonant → 2
r → consonant → 3
o → vowel → 2

Result: vowels=2, consonants=3
```

### Complexity
- **Time:** O(n)
- **Space:** O(1)

### Common Mistakes
- Forgetting case sensitivity
- Counting non-alphabetic characters as consonants

### Edge Cases
- Empty string → (0, 0)
- Only vowels → (n, 0)

### Variations
- Count frequency of each vowel

### Quick Revision Notes
- Vowels: a, e, i, o, u
- Use .lower() for case-insensitive
- Use .isalpha() to filter non-letters

---

## Question #8: Find Largest Element in Array

### Problem Statement
Find the largest element in a given array.

### Difficulty
Easy

### Pattern
Arrays

### Companies Asked
Wipro, TCS, Infosys, HCL

### Concepts Needed
- Array traversal
- Comparison

### Constraints
- 1 ≤ array length ≤ 10^6

### Approach
Initialize max with first element. Traverse and update.

**Time Complexity:** O(n)
**Space Complexity:** O(1)

### Python Solution
```python
from typing import List


def find_largest(arr: List[int]) -> int:
    """
    Finds the largest element in an array.
    
    Args:
        arr: List of integers
        
    Returns:
        Largest element
    """
    if not arr:
        raise ValueError("Array cannot be empty")
    
    max_element = arr[0]
    for num in arr[1:]:
        if num > max_element:
            max_element = num
    return max_element
```

### Dry Run
```
Input: [3, 7, 2, 9, 1, 5]

max_element = 3

num=7: 7 > 3 → max_element = 7
num=2: 2 > 7 → no update
num=9: 9 > 7 → max_element = 9
num=1: 1 > 9 → no update
num=5: 5 > 9 → no update

Result: 9
```

### Complexity
- **Time:** O(n)
- **Space:** O(1)

### Common Mistakes
- Not handling empty arrays
- Initializing max to 0 instead of first element (fails for all negative)

### Edge Cases
- Single element → returns that element
- All negative → works correctly

### Variations
- Find second largest
- Find smallest element

### Expected Output
```
Input: [3, 7, 2, 9, 1, 5]
Output: 9
```

### Quick Revision Notes
- Initialize max with first element
- Single pass O(n)
- Python's max() does this internally

---

## Question #9: Remove Duplicates from Array

### Problem Statement
Remove duplicates from an array preserving original order.

### Difficulty
Easy

### Pattern
Arrays, Hashing

### Companies Asked
Wipro, TCS, Infosys, Cognizant

### Concepts Needed
- Set for tracking seen elements
- Order preservation

### Constraints
- 1 ≤ array length ≤ 10^5

### Approach 1: Brute Force
Check each element against result array.

**Time Complexity:** O(n²)
**Space Complexity:** O(n)

### Approach 2: Optimized (Using Set)
Use set to track seen elements.

**Time Complexity:** O(n)
**Space Complexity:** O(n)

### Python Solution
```python
from typing import List


def remove_duplicates(arr: List[int]) -> List[int]:
    """
    Removes duplicates from array while preserving order.
    
    Args:
        arr: Input list with possible duplicates
        
    Returns:
        List with unique elements in original order
    """
    seen = set()
    result = []
    
    for num in arr:
        if num not in seen:
            seen.add(num)
            result.append(num)
    
    return result
```

### Dry Run
```
Input: [4, 2, 4, 1, 2, 3, 1]

seen={}, result=[]

num=4: not seen → seen={4}, result=[4]
num=2: not seen → seen={4,2}, result=[4,2]
num=4: in seen → skip
num=1: not seen → seen={4,2,1}, result=[4,2,1]
num=2: in seen → skip
num=3: not seen → seen={4,2,1,3}, result=[4,2,1,3]
num=1: in seen → skip

Result: [4, 2, 1, 3]
```

### Complexity
- **Time:** O(n)
- **Space:** O(n)

### Common Mistakes
- Using a set directly loses order

### Edge Cases
- Empty → []
- All duplicates → [element]

### Variations
- Remove duplicates in-place (sorted array)
- Count frequency

### Quick Revision Notes
- Set for O(1) duplicate detection
- List preserves insertion order
- Trade-off: O(n) extra space

---

## Question #10: Pyramid Pattern

### Problem Statement
Print a pyramid pattern of stars for n rows.

### Difficulty
Easy

### Pattern
Pattern Printing

### Companies Asked
Wipro, TCS, Cognizant, HCL

### Concepts Needed
- Nested loops
- String multiplication

### Constraints
- 1 ≤ n ≤ 100

### Approach
For row i (0-indexed): print (n-i-1) spaces and (2i+1) stars.

**Time Complexity:** O(n²)
**Space Complexity:** O(1)

### Python Solution
```python
def print_pyramid(n: int) -> None:
    """
    Prints a pyramid pattern of stars.
    
    Pattern for n=5:
        *
       ***
      *****
     *******
    *********
    """
    for i in range(n):
        print(" " * (n - i - 1) + "*" * (2 * i + 1))
```

### Dry Run
```
Input: n = 4

i=0: spaces=3, stars=1 → "   *"
i=1: spaces=2, stars=3 → "  ***"
i=2: spaces=1, stars=5 → " *****"
i=3: spaces=0, stars=7 → "*******"
```

### Complexity
- **Time:** O(n²)
- **Space:** O(1)

### Common Mistakes
- Off-by-one in star count (2i+1 not 2i-1)
- Wrong number of spaces

### Edge Cases
- n = 1 → single star

### Variations
- Inverted pyramid
- Diamond pattern
- Number pyramid

### Quick Revision Notes
- Row i: spaces = n-i-1, stars = 2i+1
- Use string multiplication for clean code

---

## 📊 Pattern Summary (Questions 1-10)

### Arrays Covered
- Find largest element, remove duplicates

### Strings Covered
- Reverse string, palindrome check, count vowels/consonants

### Math/Number Theory
- Factorial, Fibonacci, prime check, Armstrong number

### Pattern Printing
- Pyramid pattern

### Revision Table
| Question | Pattern | Difficulty | Key Technique |
|----------|---------|-----------|--------------|
| 1. Reverse String | Strings | Easy | Two pointers |
| 2. Check Palindrome | Strings | Easy | Two pointers |
| 3. Factorial | Math | Easy | Iteration/Recursion |
| 4. Fibonacci | Math | Easy | DP-like iteration |
| 5. Prime Check | Number Theory | Easy | √n optimization |
| 6. Armstrong Number | Number Theory | Easy | Digit extraction |
| 7. Count Vowels/Consonants | Strings | Easy | Set for lookup |
| 8. Largest Element | Arrays | Easy | Single pass |
| 9. Remove Duplicates | Arrays/Hashing | Easy | Set tracking |
| 10. Pyramid Pattern | Pattern Printing | Easy | String multiplication |

---

## Question #11: Sum of Digits

### Problem Statement
Find sum of digits of a given number.

### Difficulty
Easy

### Pattern
Math

### Companies Asked
Wipro, TCS, Infosys, Accenture

### Concepts Needed
- Digit extraction using modulo 10

### Constraints
- 0 ≤ n ≤ 10^18

### Approach
Repeatedly extract last digit using n % 10, add to sum, divide by 10.

**Time Complexity:** O(log₁₀ n)
**Space Complexity:** O(1)

### Python Solution
```python
def sum_of_digits(n: int) -> int:
    """
    Computes sum of digits of a number.
    
    Args:
        n: Input integer
        
    Returns:
        Sum of its digits
    """
    n = abs(n)
    total = 0
    while n > 0:
        total += n % 10
        n //= 10
    return total
```

### Dry Run
```
Input: n = 1234

Iter 1: total += 1234 % 10 = 4 → total=4, n=123
Iter 2: total += 123 % 10 = 3 → total=7, n=12
Iter 3: total += 12 % 10 = 2 → total=9, n=1
Iter 4: total += 1 % 10 = 1 → total=10, n=0

Result: 10
```

### Complexity
- **Time:** O(log₁₀ n)
- **Space:** O(1)

### Common Mistakes
- Not handling n=0
- Forgetting absolute value for negative numbers

### Edge Cases
- n = 0 → 0
- n = 5 → 5

### Variations
- Digital root (repeated sum until single digit)
- Product of digits

### Expected Output
```
Input: 1234
Output: 10
```

### Quick Revision Notes
- Extract last digit with n % 10
- Remove last digit with n // 10
- O(log₁₀ n) time

---

## Question #12: Perfect Number

### Problem Statement
Check if a number is perfect (equals sum of its proper divisors).

### Difficulty
Easy

### Pattern
Math, Number Theory

### Companies Asked
Wipro, TCS

### Concepts Needed
- Proper divisors
- Optimization: check up to √n

### Constraints
- 1 ≤ n ≤ 10^8

### Approach
Find divisors by iterating from 1 to √n. Add both i and n/i.

**Time Complexity:** O(√n)
**Space Complexity:** O(1)

### Python Solution
```python
import math


def is_perfect_number(n: int) -> bool:
    """
    Checks if a number is perfect.
    
    A perfect number equals sum of its proper divisors (excluding itself).
    """
    if n <= 1:
        return False
    
    divisor_sum = 0
    for i in range(1, int(math.sqrt(n)) + 1):
        if n % i == 0:
            divisor_sum += i
            if i != 1 and i != n // i:
                divisor_sum += n // i
    
    return divisor_sum == n
```

### Dry Run
```
Input: n = 28

Divisors: 1, 2, 4, 7, 14
sqrt(28) ≈ 5.29

i=1: add 1 → sum=1
i=2: add 2, add 14 → sum=17
i=3: skip
i=4: add 4, add 7 → sum=28
i=5: skip

28 == 28 → True
```

### Complexity
- **Time:** O(√n)
- **Space:** O(1)

### Common Mistakes
- Including n itself in sum
- Not handling i = n/i case (perfect squares)

### Edge Cases
- n = 1 → False
- n = 6 → True

### Variations
- Abundant/deficient numbers

### Expected Output
```
Input: 28
Output: True
Input: 10
Output: False
```

### Quick Revision Notes
- Proper divisors exclude the number itself
- Check only up to √n
- Known perfect numbers: 6, 28, 496, 8128

---

## Question #13: GCD of Two Numbers

### Problem Statement
Find GCD of two numbers using Euclid's algorithm.

### Difficulty
Easy

### Pattern
Math, Recursion

### Companies Asked
Wipro, TCS, Infosys

### Concepts Needed
- Euclid's algorithm: GCD(a,b) = GCD(b, a%b)

### Constraints
- 1 ≤ a, b ≤ 10^9

### Approach
Replace (a,b) with (b, a%b) until b = 0.

**Time Complexity:** O(log min(a,b))
**Space Complexity:** O(1)

### Python Solution
```python
def gcd(a: int, b: int) -> int:
    """
    Computes GCD using Euclid's algorithm.
    """
    while b != 0:
        a, b = b, a % b
    return a


def gcd_recursive(a: int, b: int) -> int:
    """
    Computes GCD recursively.
    """
    return a if b == 0 else gcd_recursive(b, a % b)
```

### Dry Run
```
Input: a=48, b=18

Iter 1: a=48, b=18 → 48%18=12 → a=18, b=12
Iter 2: a=18, b=12 → 18%12=6 → a=12, b=6
Iter 3: a=12, b=6 → 12%6=0 → a=6, b=0

b=0 → return a=6

Result: 6
```

### Complexity
- **Time:** O(log min(a,b))
- **Space:** O(1)

### Common Mistakes
- Not handling zero correctly

### Edge Cases
- One number is 0 → GCD is the other
- Both equal → GCD is that number

### Variations
- LCM using GCD: LCM = a × b / GCD(a,b)
- GCD of an array

### Expected Output
```
Input: a=48, b=18
Output: 6
```

### Quick Revision Notes
- GCD(a,b) = GCD(b, a%b)
- Time: O(log min(a,b))
- LCM = a × b / GCD(a,b)

---

## Question #14: Check Anagram

### Problem Statement
Check if two strings are anagrams.

### Difficulty
Easy

### Pattern
Strings, Hashing

### Companies Asked
Wipro, TCS, Accenture, Cognizant

### Concepts Needed
- Character frequency counting

### Constraints
- 1 ≤ string length ≤ 10^5
- Lowercase English letters

### Approach 1: Sorting
Sort both and compare.

**Time Complexity:** O(n log n)
**Space Complexity:** O(n)

### Approach 2: Frequency Count
Count character frequencies using array of size 26.

**Time Complexity:** O(n)
**Space Complexity:** O(1)

### Python Solution
```python
from collections import Counter


def are_anagrams(s1: str, s2: str) -> bool:
    """
    Checks if two strings are anagrams.
    """
    if len(s1) != len(s2):
        return False
    return Counter(s1) == Counter(s2)
```

### Dry Run
```
Input: s1="listen", s2="silent"

Counter(s1) = {'l':1,'i':1,'s':1,'t':1,'e':1,'n':1}
Counter(s2) = {'s':1,'i':1,'l':1,'e':1,'n':1,'t':1}

Counters are equal → True
```

### Complexity
- **Time:** O(n)
- **Space:** O(1)

### Common Mistakes
- Not checking length first
- Case sensitivity issues

### Edge Cases
- Empty strings → True
- Different lengths → False

### Variations
- Group anagrams
- Find all anagrams in a string

### Quick Revision Notes
- Anagrams have same character frequencies
- Length check first for optimization
- Use Counter or frequency array of size 26

---

## Question #15: Diamond Pattern

### Problem Statement
Print a diamond pattern of stars for n rows (half of diamond).

### Difficulty
Easy

### Pattern
Pattern Printing

### Companies Asked
Wipro, TCS, Cognizant

### Concepts Needed
- Symmetric patterns
- Nested loops

### Constraints
- 1 ≤ n ≤ 100

### Approach
Upper half: pyramid (n rows). Lower half: inverted pyramid (n-1 rows).

**Time Complexity:** O(n²)
**Space Complexity:** O(1)

### Python Solution
```python
def print_diamond(n: int) -> None:
    """
    Prints a diamond pattern of stars.
    
    Pattern for n=4:
       *
      ***
     *****
    *******
     *****
      ***
       *
    """
    # Upper half
    for i in range(n):
        print(" " * (n - i - 1) + "*" * (2 * i + 1))
    
    # Lower half
    for i in range(n - 2, -1, -1):
        print(" " * (n - i - 1) + "*" * (2 * i + 1))
```

### Dry Run
```
Input: n = 3

Upper half:
i=0: "  *"
i=1: " ***"
i=2: "*****"

Lower half:
i=1: " ***"
i=0: "  *"
```

### Complexity
- **Time:** O(n²)
- **Space:** O(1)

### Variations
- Hollow diamond
- Number diamond

### Quick Revision Notes
- Diamond = pyramid + inverted pyramid
- Upper: i from 0 to n-1
- Lower: i from n-2 down to 0

---

## 📊 Pattern Summary (Questions 11-15)

| Question | Pattern | Difficulty | Key Technique |
|----------|---------|-----------|--------------|
| 11. Sum of Digits | Math | Easy | Modulo/division |
| 12. Perfect Number | Number Theory | Easy | √n divisor check |
| 13. GCD | Math | Easy | Euclid's algorithm |
| 14. Check Anagram | Strings | Easy | Frequency count |
| 15. Diamond Pattern | Pattern Printing | Easy | Symmetric printing |

---

## Question #16: Two Sum

### Problem Statement
Given an array nums and target, return indices of two numbers that add up to target.

### Difficulty
Easy

### Pattern
HashMap, Arrays

### Companies Asked
Wipro, TCS, Infosys, Amazon

### Concepts Needed
- Complement = target - current
- HashMap O(1) lookup

### Constraints
- 2 ≤ array length ≤ 10^5

### Approach 1: Brute Force
Nested loops checking all pairs.

**Time Complexity:** O(n²)
**Space Complexity:** O(1)

### Approach 2: HashMap
One pass: check complement in hashmap, store current.

**Time Complexity:** O(n)
**Space Complexity:** O(n)

### Python Solution
```python
from typing import List, Tuple


def two_sum(nums: List[int], target: int) -> Tuple[int, int]:
    """
    Finds indices of two numbers that sum to target.
    """
    seen = {}
    for i, num in enumerate(nums):
        complement = target - num
        if complement in seen:
            return seen[complement], i
        seen[num] = i
    return -1, -1
```

### Dry Run
```
Input: nums=[2, 7, 11, 15], target=9

seen = {}
i=0, num=2: complement=7, not in seen → seen={2:0}
i=1, num=7: complement=2, in seen → return (0, 1)

Result: (0, 1)
```

### Complexity
- **Time:** O(n)
- **Space:** O(n)

### Common Mistakes
- Using same element twice
- Not handling no-solution case

### Edge Cases
- Negative numbers → works
- Duplicates → works

### Variations
- Three sum
- Two sum in sorted array (two pointers)

### Quick Revision Notes
- Complement = target - current
- HashMap for O(1) lookup
- Single pass: check before storing

---

## Question #17: Find Missing Number

### Problem Statement
Find missing number in an array of n distinct numbers from range 0 to n.

### Difficulty
Easy

### Pattern
Arrays, Math, XOR

### Companies Asked
Wipro, TCS, Infosys

### Concepts Needed
- Sum formula: n(n+1)/2
- XOR properties

### Constraints
- 1 ≤ n ≤ 10^5

### Approach 1: Sum Formula
Expected sum - actual sum = missing number.

**Time Complexity:** O(n)
**Space Complexity:** O(1)

### Approach 2: XOR
XOR all indices and values → missing number.

**Time Complexity:** O(n)
**Space Complexity:** O(1)

### Python Solution
```python
from typing import List


def find_missing_number(nums: List[int]) -> int:
    """
    Finds missing number using sum formula.
    """
    n = len(nums)
    expected_sum = n * (n + 1) // 2
    actual_sum = sum(nums)
    return expected_sum - actual_sum
```

### Dry Run
```
Input: [3, 0, 1]

n = 3
expected_sum = 3 × 4 / 2 = 6
actual_sum = 3 + 0 + 1 = 4
missing = 6 - 4 = 2

Result: 2
```

### Complexity
- **Time:** O(n)
- **Space:** O(1)

### Common Mistakes
- Using n+1 as max incorrectly

### Variations
- Find all missing numbers
- Find duplicate and missing

### Quick Revision Notes
- Sum(0 to n) = n(n+1)/2
- Missing = expected - actual
- XOR avoids overflow

---

## Question #18: Move Zeroes to End

### Problem Statement
Move all zeroes to end while maintaining order of non-zero elements.

### Difficulty
Easy

### Pattern
Arrays, Two Pointers

### Companies Asked
Wipro, TCS, Infosys

### Concepts Needed
- In-place modification
- Two-pointer technique

### Constraints
- 1 ≤ array length ≤ 10^5

### Approach
Non-zero pointer tracks where next non-zero goes. Swap when non-zero found.

**Time Complexity:** O(n)
**Space Complexity:** O(1)

### Python Solution
```python
from typing import List


def move_zeroes(nums: List[int]) -> None:
    """
    Moves zeroes to end in-place.
    """
    position = 0
    for i in range(len(nums)):
        if nums[i] != 0:
            nums[position], nums[i] = nums[i], nums[position]
            position += 1
```

### Dry Run
```
Input: [0, 1, 0, 3, 12]

position=0
i=0: nums[0]=0 → skip
i=1: nums[1]=1 → swap(0,1) → [1,0,0,3,12], pos=1
i=2: nums[2]=0 → skip
i=3: nums[3]=3 → swap(1,3) → [1,3,0,0,12], pos=2
i=4: nums[4]=12 → swap(2,4) → [1,3,12,0,0], pos=3

Result: [1, 3, 12, 0, 0]
```

### Complexity
- **Time:** O(n)
- **Space:** O(1)

### Common Mistakes
- Overwriting instead of swapping

### Edge Cases
- All zeroes, no zeroes

### Variations
- Move zeroes to front
- Sort by parity

### Quick Revision Notes
- Use swap, not overwrite
- One pointer tracks non-zero position
- O(n) time, O(1) space

---

## Question #19: First Non-Repeating Character

### Problem Statement
Find first non-repeating character in a string. Return index or -1.

### Difficulty
Easy

### Pattern
Strings, Hashing

### Companies Asked
Wipro, TCS, Infosys, Amazon

### Concepts Needed
- Character frequency counting

### Constraints
- 1 ≤ string length ≤ 10^5
- Lowercase English letters

### Approach
First pass: count frequencies. Second pass: find first with count 1.

**Time Complexity:** O(n)
**Space Complexity:** O(1)

### Python Solution
```python
from collections import Counter


def first_non_repeating(s: str) -> int:
    """
    Finds index of first non-repeating character.
    """
    freq = Counter(s)
    for i, ch in enumerate(s):
        if freq[ch] == 1:
            return i
    return -1
```

### Dry Run
```
Input: "wiproprogram"

freq = {'w':1,'i':1,'p':2,'r':2,'o':2,'g':1,'a':1,'m':1}

i=0, ch='w': freq=1 → return 0

Result: 0
```

### Complexity
- **Time:** O(n)
- **Space:** O(1)

### Common Mistakes
- Using single pass (need two passes)

### Edge Cases
- Empty → -1
- All repeating → -1

### Variations
- First non-repeating in stream
- First repeating character

### Quick Revision Notes
- Two passes: count then find
- O(n) time, O(1) space
- Return -1 if none

---

## Question #20: Valid Parentheses

### Problem Statement
Check if string with '(',')','{','}','[',']' is valid.

### Difficulty
Easy

### Pattern
Stack

### Companies Asked
Wipro, TCS, Infosys, Amazon

### Concepts Needed
- Stack data structure
- Matching pairs

### Constraints
- 1 ≤ string length ≤ 10^5

### Approach
Stack for opening brackets. Match closing with top.

**Time Complexity:** O(n)
**Space Complexity:** O(n)

### Python Solution
```python
def is_valid_parentheses(s: str) -> bool:
    """
    Checks if parentheses string is valid using stack.
    """
    bracket_map = {")": "(", "}": "{", "]": "["}
    stack = []
    
    for ch in s:
        if ch in bracket_map:
            top = stack.pop() if stack else "#"
            if bracket_map[ch] != top:
                return False
        else:
            stack.append(ch)
    
    return not stack
```

### Dry Run
```
Input: "{[()]}"

{: push → ['{']
[: push → ['{', '[']
(: push → ['{', '[', '(']
): pop → '(' matches → ['{', '[']
]: pop → '[' matches → ['{']
}: pop → '{' matches → []

Result: True
```

### Complexity
- **Time:** O(n)
- **Space:** O(n)

### Common Mistakes
- Not checking stack emptiness before pop

### Edge Cases
- Empty → True
- Only opening → False

### Variations
- Generate valid parentheses
- Longest valid parentheses

### Quick Revision Notes
- Stack for opening brackets
- Check matching with hashmap
- Must be empty at end

---

## 📊 Pattern Summary (Questions 16-20)

| Question | Pattern | Difficulty | Key Technique |
|----------|---------|-----------|--------------|
| 16. Two Sum | HashMap | Easy | Complement lookup |
| 17. Missing Number | Arrays/Math | Easy | Sum formula |
| 18. Move Zeroes | Arrays/Two Pointers | Easy | In-place swapping |
| 19. First Non-Repeating | Strings | Easy | Frequency count |
| 20. Valid Parentheses | Stack | Easy | Stack matching |

---

## Question #21: Find Second Largest

### Problem Statement
Find second largest element without sorting.

### Difficulty
Easy

### Pattern
Arrays

### Companies Asked
Wipro, TCS, Infosys, Cognizant

### Concepts Needed
- Two-variable tracking

### Constraints
- 2 ≤ array length ≤ 10^5

### Approach
Track largest and second_largest in one pass.

**Time Complexity:** O(n)
**Space Complexity:** O(1)

### Python Solution
```python
from typing import List, Optional


def find_second_largest(arr: List[int]) -> Optional[int]:
    """
    Finds second largest in one pass.
    """
    if len(arr) < 2:
        return None
    
    largest = second_largest = float("-inf")
    
    for num in arr:
        if num > largest:
            second_largest = largest
            largest = num
        elif num > second_largest and num != largest:
            second_largest = num
    
    return second_largest if second_largest != float("-inf") else None
```

### Dry Run
```
Input: [3, 7, 2, 9, 1, 5]

largest=-inf, second=-inf
3: 3>-inf → second=-inf, largest=3
7: 7>3 → second=3, largest=7
2: skip
9: 9>7 → second=7, largest=9
1: skip
5: skip

Result: 7
```

### Complexity
- **Time:** O(n)
- **Space:** O(1)

### Common Mistakes
- Not handling duplicates
- Initializing with 0

### Variations
- Find kth largest

---

## Question #22: Reverse Words in String

### Problem Statement
Reverse words in a sentence.

### Difficulty
Easy

### Pattern
Strings

### Companies Asked
Wipro, TCS, Infosys

### Approach
Split, reverse list, join.

**Time Complexity:** O(n)
**Space Complexity:** O(n)

### Python Solution
```python
def reverse_words(s: str) -> str:
    """
    Reverses order of words.
    """
    return " ".join(reversed(s.split()))
```

### Dry Run
```
Input: "Wipro coding interview"
split → ["Wipro", "coding", "interview"]
reverse → ["interview", "coding", "Wipro"]
join → "interview coding Wipro"
```

### Complexity
- **Time:** O(n)
- **Space:** O(n)

### Variations
- Reverse each word individually

---

## Question #23: Remove Duplicates from Sorted Array

### Problem Statement
Remove duplicates from sorted array in-place, return new length.

### Difficulty
Easy

### Pattern
Arrays, Two Pointers

### Companies Asked
Wipro, TCS, Infosys, Amazon

### Concepts Needed
- In-place modification
- Slow/fast pointers

### Constraints
- 1 ≤ array length ≤ 10^5

### Approach
Slow pointer i tracks last unique. Fast pointer j traverses. When nums[j] != nums[i], increment i and set nums[i] = nums[j].

**Time Complexity:** O(n)
**Space Complexity:** O(1)

### Python Solution
```python
from typing import List


def remove_duplicates_sorted(nums: List[int]) -> int:
    """
    Removes duplicates from sorted array in-place.
    """
    if not nums:
        return 0
    
    i = 0
    for j in range(1, len(nums)):
        if nums[j] != nums[i]:
            i += 1
            nums[i] = nums[j]
    
    return i + 1
```

### Dry Run
```
Input: [1, 1, 2, 2, 3, 4, 4]

i=0
j=1: nums[1]=1 == nums[0]=1 → skip
j=2: nums[2]=2 != nums[0]=1 → i=1, nums[1]=2 → [1,2,2,2,3,4,4]
j=3: nums[3]=2 == nums[1]=2 → skip
j=4: nums[4]=3 != nums[1]=2 → i=2, nums[2]=3 → [1,2,3,2,3,4,4]
j=5: nums[5]=4 != nums[2]=3 → i=3, nums[3]=4 → [1,2,3,4,3,4,4]
j=6: nums[6]=4 == nums[3]=4 → skip

Return i+1 = 4
First 4 elements: [1, 2, 3, 4]
```

### Complexity
- **Time:** O(n)
- **Space:** O(1)

### Common Mistakes
- Returning array instead of length

### Variations
- Allow at most 2 duplicates

---

## Question #24: Intersection of Two Arrays

### Problem Statement
Find intersection of two arrays (distinct elements).

### Difficulty
Easy

### Pattern
Arrays, Hashing

### Companies Asked
Wipro, TCS, Cognizant

### Approach
Convert to sets and use intersection.

**Time Complexity:** O(m + n)
**Space Complexity:** O(m + n)

### Python Solution
```python
from typing import List, Set


def intersection(arr1: List[int], arr2: List[int]) -> List[int]:
    """
    Finds intersection of two arrays.
    """
    return list(set(arr1) & set(arr2))
```

### Dry Run
```
Input: arr1=[1,2,2,3,4], arr2=[2,4,6,8]
set1={1,2,3,4}, set2={2,4,6,8}
set1 & set2 = {2,4}
Result: [2,4]
```

### Complexity
- **Time:** O(m + n)
- **Space:** O(m + n)

### Variations
- Intersection with duplicates
- Union of arrays

---

## Question #25: Number Pyramid

### Problem Statement
Print number pyramid where each row has numbers 1 to row_number.

### Difficulty
Easy

### Pattern
Pattern Printing

### Companies Asked
Wipro, TCS, Cognizant

### Python Solution
```python
def print_number_pyramid(n: int) -> None:
    """
    Prints number pyramid.
    
    Pattern for n=5:
        1
       12
      123
     1234
    12345
    """
    for i in range(1, n + 1):
        spaces = " " * (n - i)
        numbers = "".join(str(j) for j in range(1, i + 1))
        print(spaces + numbers)
```

### Dry Run
```
Input: n=4

i=1: spaces=3, "1" → "   1"
i=2: spaces=2, "12" → "  12"
i=3: spaces=1, "123" → " 123"
i=4: spaces=0, "1234" → "1234"
```

### Complexity
- **Time:** O(n²)
- **Space:** O(1)

---

## 📊 Pattern Summary (Questions 21-25)

| Question | Pattern | Difficulty | Key Technique |
|----------|---------|-----------|--------------|
| 21. Second Largest | Arrays | Easy | Two-track variables |
| 22. Reverse Words | Strings | Easy | Split/reverse/join |
| 23. Remove Duplicates Sorted | Arrays/Two Pointers | Easy | Slow/fast pointers |
| 24. Intersection | Arrays/Hashing | Easy | Set intersection |
| 25. Number Pyramid | Pattern Printing | Easy | String join |

---

## Question #26: Balanced Brackets

### Problem Statement
Check if brackets are balanced. Same as Q20 with detailed explanation.

### Difficulty
Easy

### Pattern
Stack

### Python Solution
```python
def is_balanced(s: str) -> bool:
    """
    Checks if brackets are balanced.
    """
    matching = {")": "(", "}": "{", "]": "["}
    stack = []
    
    for ch in s:
        if ch in matching:
            if not stack or stack.pop() != matching[ch]:
                return False
        else:
            stack.append(ch)
    
    return not stack
```

### Quick Revision Notes
- Stack for opening, check closing against top
- Must check stack is empty at end

---

## Question #27: Longest Substring Without Repeating Characters

### Problem Statement
Find length of longest substring without repeating characters.

### Difficulty
Medium

### Pattern
Sliding Window, HashMap

### Companies Asked
Wipro, TCS, Infosys, Amazon, Google

### Concepts Needed
- Sliding window
- HashMap for character positions

### Constraints
- 1 ≤ string length ≤ 10^5

### Approach
Sliding window with left and right pointers. When char repeats, move left to last_index + 1.

**Time Complexity:** O(n)
**Space Complexity:** O(min(m, n))

### Python Solution
```python
def length_of_longest_substring(s: str) -> int:
    """
    Finds length of longest substring without repeating chars.
    """
    char_index = {}
    max_length = 0
    left = 0
    
    for right, ch in enumerate(s):
        if ch in char_index and char_index[ch] >= left:
            left = char_index[ch] + 1
        char_index[ch] = right
        max_length = max(max_length, right - left + 1)
    
    return max_length
```

### Dry Run
```
Input: "abcabcbb"

char_index={}, left=0, max_length=0

r=0, 'a': not in map → store a:0, max_len=1
r=1, 'b': not in map → store b:1, max_len=2
r=2, 'c': not in map → store c:2, max_len=3
r=3, 'a': in map, 0>=0 → left=1, store a:3, max_len=3
r=4, 'b': in map, 1>=1 → left=2, store b:4, max_len=3
r=5, 'c': in map, 2>=2 → left=3, store c:5, max_len=3
r=6, 'b': in map, 4>=3 → left=5, store b:6, max_len=3
r=7, 'b': in map, 6>=5 → left=7, store b:7, max_len=3

Result: 3
```

### Complexity
- **Time:** O(n)
- **Space:** O(min(m, n))

### Common Mistakes
- Not updating left to max(current, last_index+1)

### Edge Cases
- Empty → 0
- All unique → n
- All same → 1

### Variations
- Longest substring with at most k distinct characters

### Quick Revision Notes
- Sliding window with two pointers
- HashMap stores last seen index
- left jumps to last_index + 1 on repeat

---

## Question #28: Rotate Array

### Problem Statement
Rotate array to right by k steps.

### Difficulty
Medium

### Pattern
Arrays

### Companies Asked
Wipro, TCS, Infosys, Amazon

### Concepts Needed
- Reversal algorithm

### Constraints
- 1 ≤ array length ≤ 10^5

### Approach
Three reverses: entire array, first k, remaining n-k.

**Time Complexity:** O(n)
**Space Complexity:** O(1)

### Python Solution
```python
from typing import List


def rotate_array(nums: List[int], k: int) -> None:
    """
    Rotates array right by k steps in-place.
    """
    n = len(nums)
    if n == 0:
        return
    k = k % n
    
    def reverse(start: int, end: int) -> None:
        while start < end:
            nums[start], nums[end] = nums[end], nums[start]
            start += 1
            end -= 1
    
    reverse(0, n - 1)
    reverse(0, k - 1)
    reverse(k, n - 1)
```

### Dry Run
```
Input: [1,2,3,4,5,6,7], k=3

Step 1: Reverse all → [7,6,5,4,3,2,1]
Step 2: Reverse first 3 → [5,6,7,4,3,2,1]
Step 3: Reverse last 4 → [5,6,7,1,2,3,4]

Result: [5,6,7,1,2,3,4]
```

### Complexity
- **Time:** O(n)
- **Space:** O(1)

### Common Mistakes
- Not handling k > n (use k % n)

### Variations
- Rotate left
- Search in rotated sorted array

---

## Question #29: Array Subset Check

### Problem Statement
Check if arr2 is subset of arr1.

### Difficulty
Easy

### Pattern
Arrays, Hashing

### Companies Asked
Wipro, TCS, Cognizant

### Approach
Convert arr1 to set, check all elements of arr2.

**Time Complexity:** O(m + n)
**Space Complexity:** O(m)

### Python Solution
```python
from typing import List


def is_subset(arr1: List[int], arr2: List[int]) -> bool:
    """
    Checks if arr2 is subset of arr1.
    """
    set1 = set(arr1)
    return all(elem in set1 for elem in arr2)
```

### Dry Run
```
Input: arr1=[1,2,3,4,5], arr2=[2,4]
set1={1,2,3,4,5}
2 in set1 → True, 4 in set1 → True → True
```

### Complexity
- **Time:** O(m + n)
- **Space:** O(m)

---

## Question #30: Find Duplicate in Array

### Problem Statement
Find duplicate element where elements in range [1,n-1]. Exactly one duplicate.

### Difficulty
Easy

### Pattern
Arrays, Two Pointers, Negative Marking

### Companies Asked
Wipro, TCS, Infosys

### Approach
Negative marking: mark visited index as negative.

**Time Complexity:** O(n)
**Space Complexity:** O(1)

### Python Solution
```python
from typing import List


def find_duplicate(nums: List[int]) -> int:
    """
    Finds duplicate using negative marking.
    """
    for num in nums:
        index = abs(num)
        if nums[index] < 0:
            return index
        nums[index] = -nums[index]
    return -1
```

### Dry Run
```
Input: [1, 3, 4, 2, 2]

num=1 → index=1, nums[1]=3>0 → nums[1]=-3
num=-3 → index=3, nums[3]=2>0 → nums[3]=-2
num=4 → index=4, nums[4]=2>0 → nums[4]=-2
num=-2 → index=2, nums[2]=4>0 → nums[2]=-4
num=-2 → index=2, nums[2]=-4<0 → return 2

Result: 2
```

### Complexity
- **Time:** O(n)
- **Space:** O(1) — modifies array

### Edge Cases
- Duplicate at end → works

### Quick Revision Notes
- Negative marking: O(1) space
- Set approach: O(n) space, no modification

---

## 📊 Pattern Summary (Questions 26-30)

| Question | Pattern | Difficulty | Key Technique |
|----------|---------|-----------|--------------|
| 26. Balanced Brackets | Stack | Easy | Stack matching |
| 27. Longest Substring No Repeat | Sliding Window | Medium | HashMap + two pointers |
| 28. Rotate Array | Arrays | Medium | Reversal algorithm |
| 29. Array Subset | Hashing | Easy | Set operations |
| 30. Find Duplicate | Arrays | Medium | Negative marking |

---

## Question #31: Merge Two Sorted Arrays

### Problem Statement
Merge two sorted arrays into one sorted array.

### Difficulty
Easy

### Pattern
Arrays, Two Pointers

### Companies Asked
Wipro, TCS, Infosys, Amazon

### Concepts Needed
- Merge step of merge sort

### Constraints
- 1 ≤ array length ≤ 10^5

### Approach
Two pointers, compare and advance.

**Time Complexity:** O(m + n)
**Space Complexity:** O(m + n)

### Python Solution
```python
from typing import List


def merge_sorted_arrays(arr1: List[int], arr2: List[int]) -> List[int]:
    """
    Merges two sorted arrays.
    """
    result = []
    i = j = 0
    
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

### Dry Run
```
Input: arr1=[1,3,5], arr2=[2,4,6]

result=[], i=0, j=0
1<=2 → result=[1], i=1
3>2 → result=[1,2], j=1
3<=4 → result=[1,2,3], i=2
5>4 → result=[1,2,3,4], j=2
5<=6 → result=[1,2,3,4,5], i=3
i=3 done → add arr2[2:]=[6]

Result: [1,2,3,4,5,6]
```

### Complexity
- **Time:** O(m + n)
- **Space:** O(m + n)

### Common Mistakes
- Forgetting remaining elements

---

## Question #32: Maximum Subarray Sum (Kadane)

### Problem Statement
Find maximum subarray sum using Kadane's algorithm.

### Difficulty
Medium

### Pattern
Arrays, DP, Kadane's Algorithm

### Companies Asked
Wipro, TCS, Infosys, Amazon, Google

### Concepts Needed
- Dynamic programming
- Local vs global optimum

### Constraints
- 1 ≤ array length ≤ 10^5

### Approach
current_sum = max(num, current_sum + num). Track global max.

**Time Complexity:** O(n)
**Space Complexity:** O(1)

### Python Solution
```python
from typing import List


def max_subarray_sum(nums: List[int]) -> int:
    """
    Finds maximum subarray sum using Kadane's algorithm.
    """
    max_sum = float("-inf")
    current_sum = 0
    
    for num in nums:
        current_sum = max(num, current_sum + num)
        max_sum = max(max_sum, current_sum)
    
    return max_sum
```

### Dry Run
```
Input: [-2, 1, -3, 4, -1, 2, 1, -5, 4]

cur=0, max=-inf
-2: cur=max(-2,-2)=-2, max=-2
1: cur=max(1,-1)=1, max=1
-3: cur=max(-3,-2)=-2, max=1
4: cur=max(4,2)=4, max=4
-1: cur=max(-1,3)=3, max=4
2: cur=max(2,5)=5, max=5
1: cur=max(1,6)=6, max=6
-5: cur=max(-5,1)=1, max=6
4: cur=max(4,5)=5, max=6

Result: 6
```

### Complexity
- **Time:** O(n)
- **Space:** O(1)

### Common Mistakes
- Initializing max_sum to 0 (fails for all negative)

### Edge Cases
- All negative → returns largest (least negative)
- All positive → returns total sum

### Variations
- Maximum product subarray

### Quick Revision Notes
- current_sum = max(num, current_sum + num)
- Initialize max_sum to -inf
- O(n) time, O(1) space

---

## Question #33: Find Pairs with Given Sum

### Problem Statement
Find all distinct pairs summing to target.

### Difficulty
Easy

### Pattern
HashMap, Arrays

### Companies Asked
Wipro, TCS, Infosys

### Approach
Use seen set and pair set.

**Time Complexity:** O(n)
**Space Complexity:** O(n)

### Python Solution
```python
from typing import List, Set, Tuple


def find_pairs(nums: List[int], target: int) -> Set[Tuple[int, int]]:
    """
    Finds all distinct pairs summing to target.
    """
    seen = set()
    pairs = set()
    
    for num in nums:
        complement = target - num
        if complement in seen:
            pair = tuple(sorted((num, complement)))
            pairs.add(pair)
        seen.add(num)
    
    return pairs
```

### Dry Run
```
Input: [1,2,3,4,5], target=5

num=1: comp=4, not seen → seen={1}
num=2: comp=3, not seen → seen={1,2}
num=3: comp=2, in seen → add (2,3), seen={1,2,3}
num=4: comp=1, in seen → add (1,4), seen={1,2,3,4}
num=5: comp=0, not seen

Result: {(1,4), (2,3)}
```

---

## Question #34: Pascal's Triangle

### Problem Statement
Generate Pascal's triangle up to n rows.

### Difficulty
Medium

### Pattern
Pattern Printing, DP

### Companies Asked
Wipro, TCS, Infosys, Cognizant

### Approach
Each element is sum of two elements above.

**Time Complexity:** O(n²)
**Space Complexity:** O(n²)

### Python Solution
```python
from typing import List


def generate_pascals_triangle(n: int) -> List[List[int]]:
    """
    Generates Pascal's triangle.
    """
    triangle = []
    
    for i in range(n):
        row = [1] * (i + 1)
        for j in range(1, i):
            row[j] = triangle[i - 1][j - 1] + triangle[i - 1][j]
        triangle.append(row)
    
    return triangle
```

### Dry Run
```
Input: n = 5

i=0: [1]
i=1: [1,1]
i=2: j=1: tri[1][0]+tri[1][1]=2 → [1,2,1]
i=3: j=1: 1+2=3, j=2: 2+1=3 → [1,3,3,1]
i=4: j=1:1+3=4, j=2:3+3=6, j=3:3+1=4 → [1,4,6,4,1]

Result: [[1],[1,1],[1,2,1],[1,3,3,1],[1,4,6,4,1]]
```

### Complexity
- **Time:** O(n²)
- **Space:** O(n²)

### Quick Revision Notes
- Edges are always 1
- Interior: tri[i][j] = tri[i-1][j-1] + tri[i-1][j]

---

## Question #35: Palindrome Number (Integer)

### Problem Statement
Check if integer is palindrome without converting to string.

### Difficulty
Easy

### Pattern
Math

### Companies Asked
Wipro, TCS, Infosys, Amazon

### Approach
Reverse half the digits and compare.

**Time Complexity:** O(log₁₀ n)
**Space Complexity:** O(1)

### Python Solution
```python
def is_palindrome_number(n: int) -> bool:
    """
    Checks if integer is palindrome without string conversion.
    """
    if n < 0 or (n % 10 == 0 and n != 0):
        return False
    
    reversed_half = 0
    while n > reversed_half:
        reversed_half = reversed_half * 10 + n % 10
        n //= 10
    
    return n == reversed_half or n == reversed_half // 10
```

### Dry Run
```
Input: 1221

n=1221, rev=0
1: rev=1, n=122
2: rev=12, n=12
n=12 <= rev=12 → exit
12 == 12 → True

Result: True
```

### Complexity
- **Time:** O(log₁₀ n)
- **Space:** O(1)

### Common Mistakes
- Forgetting negative numbers not palindrome

### Edge Cases
- 0 → True
- Negative → False

---

## 📊 Pattern Summary (Questions 31-35)

| Question | Pattern | Difficulty | Key Technique |
|----------|---------|-----------|--------------|
| 31. Merge Sorted Arrays | Arrays | Easy | Two pointers |
| 32. Maximum Subarray Sum | Arrays/DP | Medium | Kadane's algorithm |
| 33. Find Pairs with Sum | HashMap | Easy | Complement lookup |
| 34. Pascal's Triangle | Pattern/DP | Medium | Previous row sum |
| 35. Palindrome Number | Math | Easy | Reverse half |

---

## Question #36: String Compression

### Problem Statement
Compress string using counts of consecutive characters.

### Difficulty
Easy

### Pattern
Strings

### Companies Asked
Wipro, TCS, Infosys

### Approach
Count consecutive occurrences.

**Time Complexity:** O(n)
**Space Complexity:** O(n)

### Python Solution
```python
def compress_string(s: str) -> str:
    """
    Compresses string using run-length encoding.
    Returns original if compressed isn't shorter.
    """
    if not s:
        return s
    
    compressed = []
    count = 1
    
    for i in range(1, len(s)):
        if s[i] == s[i - 1]:
            count += 1
        else:
            compressed.append(s[i - 1] + str(count))
            count = 1
    
    compressed.append(s[-1] + str(count))
    
    result = "".join(compressed)
    return result if len(result) < len(s) else s
```

### Dry Run
```
Input: "aabbbccc"

i=1: 'a'=='a' → count=2
i=2: 'b'!='a' → append "a2", count=1
i=3: 'b'=='b' → count=2
i=4: 'b'=='b' → count=3
i=5: 'c'!='b' → append "b3", count=1
i=6: 'c'=='c' → count=2
i=7: 'c'=='c' → count=3
End: append "c3"

Result: "a2b3c3"

Input: "abcd"
Result: "abcd" (compressed "a1b1c1d1" is longer)
```

### Complexity
- **Time:** O(n)
- **Space:** O(n)

### Common Mistakes
- Forgetting to handle last character
- Not comparing lengths

---

## Question #37: Count Element Frequencies

### Problem Statement
Count frequencies of each element in array.

### Difficulty
Easy

### Pattern
Hashing

### Companies Asked
Wipro, TCS, Cognizant

### Approach
Use dictionary.

**Time Complexity:** O(n)
**Space Complexity:** O(k) where k is distinct elements

### Python Solution
```python
from typing import List, Dict


def count_frequencies(nums: List[int]) -> Dict[int, int]:
    """
    Counts frequency of each element.
    """
    freq = {}
    for num in nums:
        freq[num] = freq.get(num, 0) + 1
    return freq
```

### Dry Run
```
Input: [1, 2, 2, 3, 3, 3]

1: freq[1]=1
2: freq[2]=1
2: freq[2]=2
3: freq[3]=1
3: freq[3]=2
3: freq[3]=3

Result: {1:1, 2:2, 3:3}
```

---

## Question #38: String Rotation Check

### Problem Statement
Check if one string is rotation of another.

### Difficulty
Easy

### Pattern
Strings

### Companies Asked
Wipro, TCS, Infosys

### Approach
Check if s2 is substring of s1 + s1.

**Time Complexity:** O(n)
**Space Complexity:** O(n)

### Python Solution
```python
def is_rotation(s1: str, s2: str) -> bool:
    """
    Checks if s2 is rotation of s1.
    """
    if len(s1) != len(s2) or not s1:
        return False
    return s2 in s1 + s1
```

### Dry Run
```
Input: s1="wipro", s2="prowi"

s1+s1 = "wiprowipro"
"prowi" in "wiprowipro" → True
```

---

## Question #39: Majority Element

### Problem Statement
Find element appearing more than n/2 times.

### Difficulty
Medium

### Pattern
Voting Algorithm

### Companies Asked
Wipro, TCS, Infosys, Amazon

### Approach
Boyer-Moore voting algorithm.

**Time Complexity:** O(n)
**Space Complexity:** O(1)

### Python Solution
```python
from typing import List, Optional


def find_majority_element(nums: List[int]) -> Optional[int]:
    """
    Finds majority element (> n/2) using Boyer-Moore.
    """
    candidate = None
    count = 0
    
    for num in nums:
        if count == 0:
            candidate = num
        count += 1 if num == candidate else -1
    
    # Verification
    verification_count = sum(1 for num in nums if num == candidate)
    return candidate if verification_count > len(nums) // 2 else None
```

### Dry Run
```
Input: [2, 2, 1, 1, 1, 2, 2]

candidate=None, count=0
2: candidate=2, count=1
2: count=2
1: count=1
1: count=0
1: candidate=1, count=1
2: count=0
2: candidate=2, count=1

candidate=2
verification: count of 2 = 4 > 3.5 → return 2
```

### Complexity
- **Time:** O(n)
- **Space:** O(1)

### Quick Revision Notes
- Boyer-Moore: cancel non-matching pairs
- Need verification phase

---

## Question #40: Spiral Matrix

### Problem Statement
Print matrix elements in spiral order.

### Difficulty
Medium

### Pattern
Matrix, Simulation

### Companies Asked
Wipro, TCS, Infosys, Amazon

### Approach
Four boundaries: top, bottom, left, right.

**Time Complexity:** O(m × n)
**Space Complexity:** O(1) excluding result

### Python Solution
```python
from typing import List


def spiral_order(matrix: List[List[int]]) -> List[int]:
    """
    Returns matrix in spiral order.
    """
    if not matrix or not matrix[0]:
        return []
    
    result = []
    top, bottom = 0, len(matrix) - 1
    left, right = 0, len(matrix[0]) - 1
    
    while top <= bottom and left <= right:
        # Right
        for col in range(left, right + 1):
            result.append(matrix[top][col])
        top += 1
        
        # Down
        for row in range(top, bottom + 1):
            result.append(matrix[row][right])
        right -= 1
        
        if top <= bottom:
            for col in range(right, left - 1, -1):
                result.append(matrix[bottom][col])
            bottom -= 1
        
        if left <= right:
            for row in range(bottom, top - 1, -1):
                result.append(matrix[row][left])
            left += 1
    
    return result
```

### Dry Run
```
Input: [[1,2,3],[4,5,6],[7,8,9]]

top=0, bottom=2, left=0, right=2

Iter 1:
→: [1,2,3], top=1
↓: [6,9], right=1
←: [8,7], bottom=1
↑: [4], left=1

Iter 2:
→: [5], top=2 → top>bottom → exit

Result: [1,2,3,6,9,8,7,4,5]
```

### Complexity
- **Time:** O(m × n)
- **Space:** O(1)

### Variations
- Generate spiral matrix

---

## 📊 Pattern Summary (Questions 36-40)

| Question | Pattern | Difficulty | Key Technique |
|----------|---------|-----------|--------------|
| 36. String Compression | Strings | Easy | Run-length encoding |
| 37. Count Frequencies | Hashing | Easy | Dictionary |
| 38. String Rotation | Strings | Easy | s1+s1 trick |
| 39. Majority Element | Voting | Medium | Boyer-Moore |
| 40. Spiral Matrix | Matrix | Medium | Boundary traversal |

---

## Question #41: Binary Search

### Problem Statement
Implement binary search to find target in sorted array.

### Difficulty
Medium

### Pattern
Binary Search, Divide and Conquer

### Companies Asked
Wipro, TCS, Infosys, Amazon

### Concepts Needed
- Divide and conquer
- Sorted array property

### Constraints
- 1 ≤ array length ≤ 10^5

### Approach
Maintain left and right pointers. Find mid. Compare with target, eliminate half.

**Time Complexity:** O(log n)
**Space Complexity:** O(1)

### Python Solution
```python
from typing import List


def binary_search(nums: List[int], target: int) -> int:
    """
    Binary search for target in sorted array.
    Returns index or -1.
    """
    left, right = 0, len(nums) - 1
    
    while left <= right:
        mid = left + (right - left) // 2
        
        if nums[mid] == target:
            return mid
        elif nums[mid] < target:
            left = mid + 1
        else:
            right = mid - 1
    
    return -1
```

### Dry Run
```
Input: nums=[1,3,5,7,9,11], target=7

left=0, right=5
mid=0+(5-0)//2=2, nums[2]=5 < 7 → left=3
mid=3+(5-3)//2=4, nums[4]=9 > 7 → right=3
mid=3+(3-3)//2=3, nums[3]=7 == 7 → return 3

Result: 3
```

### Complexity
- **Time:** O(log n)
- **Space:** O(1)

### Common Mistakes
- Not using mid = left + (right-left)//2 (overflow prevention)
- Off-by-one in while condition (left <= right vs left < right)
- Infinite loop if mid not updated properly

### Edge Cases
- Target at ends → works
- Target not found → returns -1
- Empty array → returns -1

### Variations
- Lower bound / upper bound
- Search in rotated sorted array
- Find peak element

### Interview Tips
Wipro expects binary search as a fundamental algorithm. Explain the divide-and-conquer concept.

### Expected Output
```
Input: nums=[1,3,5,7,9,11], target=7
Output: 3

Input: nums=[1,3,5,7,9,11], target=2
Output: -1
```

### Quick Revision Notes
- Eliminate half the search space each iteration
- O(log n) time
- Array must be sorted

---

## Question #42: Peak Element

### Problem Statement
Find a peak element in an array (element greater than its neighbors).

### Difficulty
Medium

### Pattern
Binary Search

### Companies Asked
Wipro, TCS, Infosys

### Approach
Binary search: check mid. If mid is peak, return. If left neighbor is greater, search left. Else search right.

**Time Complexity:** O(log n)
**Space Complexity:** O(1)

### Python Solution
```python
from typing import List


def find_peak_element(nums: List[int]) -> int:
    """
    Finds a peak element using binary search.
    """
    left, right = 0, len(nums) - 1
    
    while left < right:
        mid = left + (right - left) // 2
        if nums[mid] > nums[mid + 1]:
            right = mid
        else:
            left = mid + 1
    
    return left
```

### Dry Run
```
Input: [1, 2, 3, 1]

left=0, right=3
mid=1, nums[1]=2 < nums[2]=3 → left=2
left=2, right=3
mid=2, nums[2]=3 > nums[3]=1 → right=2
left=2, right=2 → exit

Result: 2 (nums[2]=3 is peak)
```

### Complexity
- **Time:** O(log n)
- **Space:** O(1)

### Quick Revision Notes
- Binary search variant
- Compare mid with mid+1
- Guaranteed to find a peak

---

## Question #43: Maximum Product Subarray

### Problem Statement
Find maximum product subarray.

### Difficulty
Medium

### Pattern
Arrays, DP

### Companies Asked
Wipro, TCS, Infosys, Amazon

### Approach
Track both max_product and min_product (because negative × negative = positive).

**Time Complexity:** O(n)
**Space Complexity:** O(1)

### Python Solution
```python
from typing import List


def max_product_subarray(nums: List[int]) -> int:
    """
    Finds maximum product subarray.
    """
    if not nums:
        return 0
    
    max_prod = min_prod = result = nums[0]
    
    for num in nums[1:]:
        if num < 0:
            max_prod, min_prod = min_prod, max_prod
        
        max_prod = max(num, max_prod * num)
        min_prod = min(num, min_prod * num)
        
        result = max(result, max_prod)
    
    return result
```

### Dry Run
```
Input: [2, 3, -2, 4]

max_p=min_p=result=2

num=3: max_p=max(3,6)=6, min_p=min(3,6)=3, result=6
num=-2: swap → max_p=3, min_p=6
        max_p=max(-2,-6)=-2, min_p=min(-2,-12)=-12, result=max(6,-2)=6
num=4: max_p=max(4,-8)=4, min_p=min(4,-48)=-48, result=max(6,4)=6

Result: 6
```

### Complexity
- **Time:** O(n)
- **Space:** O(1)

### Quick Revision Notes
- Track both max and min products
- Swap on negative numbers
- O(n) time, O(1) space

---

## Question #44: Longest Common Prefix

### Problem Statement
Find longest common prefix among an array of strings.

### Difficulty
Easy

### Pattern
Strings

### Companies Asked
Wipro, TCS, Infosys

### Approach
Sort strings and compare first and last.

**Time Complexity:** O(n log n + m)
**Space Complexity:** O(1)

### Python Solution
```python
from typing import List


def longest_common_prefix(strs: List[str]) -> str:
    """
    Finds longest common prefix among strings.
    """
    if not strs:
        return ""
    
    strs.sort()
    first, last = strs[0], strs[-1]
    i = 0
    
    while i < len(first) and i < len(last) and first[i] == last[i]:
        i += 1
    
    return first[:i]
```

### Dry Run
```
Input: ["flower", "flow", "flight"]

Sorted: ["flight", "flow", "flower"]
first="flight", last="flower"

i=0: 'f' == 'f' → i=1
i=1: 'l' == 'l' → i=2
i=2: 'i' != 'o' → stop

Result: "fl"
```

### Complexity
- **Time:** O(n log n + m)
- **Space:** O(1)

---

## Question #45: Group Anagrams

### Problem Statement
Group a list of strings into anagram groups.

### Difficulty
Medium

### Pattern
Strings, HashMap, Sorting

### Companies Asked
Wipro, TCS, Infosys, Amazon

### Approach
Sort each string, use as key in hashmap.

**Time Complexity:** O(n × k log k)
**Space Complexity:** O(n × k)

### Python Solution
```python
from typing import List
from collections import defaultdict


def group_anagrams(strs: List[str]) -> List[List[str]]:
    """
    Groups anagrams together.
    """
    groups = defaultdict(list)
    
    for s in strs:
        key = "".join(sorted(s))
        groups[key].append(s)
    
    return list(groups.values())
```

### Dry Run
```
Input: ["eat", "tea", "tan", "ate", "nat", "bat"]

"eat" sorted → "aet": ["eat"]
"tea" sorted → "aet": ["eat", "tea"]
"tan" sorted → "ant": ["tan"]
"ate" sorted → "aet": ["eat", "tea", "ate"]
"nat" sorted → "ant": ["tan", "nat"]
"bat" sorted → "abt": ["bat"]

Result: [["eat","tea","ate"], ["tan","nat"], ["bat"]]
```

### Complexity
- **Time:** O(n × k log k)
- **Space:** O(n × k)

### Quick Revision Notes
- Sorted string as hashmap key
- O(n × k log k) time

---

## Question #46: Tree Traversal (DFS)

### Problem Statement
Implement inorder, preorder, and postorder traversal of a binary tree.

### Difficulty
Medium

### Pattern
Trees, Recursion, DFS

### Companies Asked
Wipro, TCS, Infosys, Amazon

### Concepts Needed
- Tree data structure
- Recursion
- DFS traversal types

### Python Solution
```python
from typing import List, Optional


class TreeNode:
    def __init__(self, val: int = 0, left: Optional["TreeNode"] = None, right: Optional["TreeNode"] = None):
        self.val = val
        self.left = left
        self.right = right


def inorder_traversal(root: Optional[TreeNode]) -> List[int]:
    """Left → Root → Right"""
    result = []
    
    def dfs(node: Optional[TreeNode]) -> None:
        if not node:
            return
        dfs(node.left)
        result.append(node.val)
        dfs(node.right)
    
    dfs(root)
    return result


def preorder_traversal(root: Optional[TreeNode]) -> List[int]:
    """Root → Left → Right"""
    result = []
    
    def dfs(node: Optional[TreeNode]) -> None:
        if not node:
            return
        result.append(node.val)
        dfs(node.left)
        dfs(node.right)
    
    dfs(root)
    return result


def postorder_traversal(root: Optional[TreeNode]) -> List[int]:
    """Left → Right → Root"""
    result = []
    
    def dfs(node: Optional[TreeNode]) -> None:
        if not node:
            return
        dfs(node.left)
        dfs(node.right)
        result.append(node.val)
    
    dfs(root)
    return result
```

### Dry Run
```
Input Tree:
    1
     \
      2
     /
    3

Inorder (Left-Root-Right): [1, 3, 2]
Preorder (Root-Left-Right): [1, 2, 3]
Postorder (Left-Right-Root): [3, 2, 1]
```

### Complexity
- **Time:** O(n)
- **Space:** O(n) — recursion stack

### Quick Revision Notes
- Inorder: Left → Root → Right (sorted order for BST)
- Preorder: Root → Left → Right
- Postorder: Left → Right → Root

---

## Question #47: Maximum Depth of Binary Tree

### Problem Statement
Find maximum depth (height) of a binary tree.

### Difficulty
Medium

### Pattern
Trees, Recursion

### Companies Asked
Wipro, TCS, Infosys, Amazon

### Approach
Recursively compute max depth of left and right subtrees.

**Time Complexity:** O(n)
**Space Complexity:** O(n) — recursion stack

### Python Solution
```python
from typing import Optional


class TreeNode:
    def __init__(self, val: int = 0, left: Optional["TreeNode"] = None, right: Optional["TreeNode"] = None):
        self.val = val
        self.left = left
        self.right = right


def max_depth(root: Optional[TreeNode]) -> int:
    """
    Finds maximum depth of binary tree.
    """
    if not root:
        return 0
    
    left_depth = max_depth(root.left)
    right_depth = max_depth(root.right)
    
    return 1 + max(left_depth, right_depth)
```

### Dry Run
```
Input Tree:
      3
     / \
    9  20
       / \
      15  7

max_depth(3) = 1 + max(max_depth(9), max_depth(20))
max_depth(9) = 1 + max(0, 0) = 1
max_depth(20) = 1 + max(max_depth(15), max_depth(7))
max_depth(15) = 1
max_depth(7) = 1
max_depth(20) = 1 + max(1, 1) = 2
max_depth(3) = 1 + max(1, 2) = 3

Result: 3
```

### Complexity
- **Time:** O(n)
- **Space:** O(n)

### Quick Revision Notes
- Base case: empty node has depth 0
- Depth = 1 + max(left_depth, right_depth)

---

## Question #48: Coin Change

### Problem Statement
Find minimum number of coins needed to make a given amount using given denominations.

### Difficulty
Medium

### Pattern
DP (Dynamic Programming)

### Companies Asked
Wipro, TCS, Infosys, Amazon

### Approach
DP array where dp[i] = min coins for amount i.

**Time Complexity:** O(amount × n)
**Space Complexity:** O(amount)

### Python Solution
```python
from typing import List


def coin_change(coins: List[int], amount: int) -> int:
    """
    Minimum coins needed to make amount.
    """
    dp = [float("inf")] * (amount + 1)
    dp[0] = 0
    
    for i in range(1, amount + 1):
        for coin in coins:
            if coin <= i:
                dp[i] = min(dp[i], 1 + dp[i - coin])
    
    return dp[amount] if dp[amount] != float("inf") else -1
```

### Dry Run
```
Input: coins=[1,2,5], amount=11

dp = [0, inf, inf, inf, inf, inf, inf, inf, inf, inf, inf, inf]

i=1: coin 1 → dp[1]=min(inf, 1+dp[0])=1
i=2: coin 1 → dp[2]=1+1=2, coin 2 → dp[2]=min(2, 1+dp[0])=1
i=3: coin 1 → dp[3]=1+1=2, coin 2 → dp[3]=min(2, 1+dp[1])=2
i=4: coin 1 → dp[4]=1+2=3, coin 2 → dp[4]=min(3, 1+dp[2])=2
i=5: coin 1 → dp[5]=1+2=3, coin 2 → dp[5]=min(3, 1+dp[3])=3, coin 5 → dp[5]=min(3, 1+dp[0])=1
...
i=11: coin 1 → 1+dp[10]=3, coin 2 → 1+dp[9]=3, coin 5 → 1+dp[6]=3
dp[11] = 3

Result: 3 (5+5+1)
```

### Complexity
- **Time:** O(amount × n)
- **Space:** O(amount)

### Quick Revision Notes
- DP[j] = min(DP[j], 1 + DP[j - coin])
- Initialize DP[0] = 0, rest = inf

---

## Question #49: Longest Palindromic Substring

### Problem Statement
Find longest palindromic substring.

### Difficulty
Medium

### Pattern
Strings, DP, Two Pointers

### Companies Asked
Wipro, TCS, Infosys, Amazon

### Approach
Expand around center. Check both odd and even length palindromes.

**Time Complexity:** O(n²)
**Space Complexity:** O(1)

### Python Solution
```python
def longest_palindromic_substring(s: str) -> str:
    """
    Finds longest palindromic substring using expand around center.
    """
    if not s:
        return ""
    
    start, max_len = 0, 0
    
    def expand_around_center(left: int, right: int) -> int:
        while left >= 0 and right < len(s) and s[left] == s[right]:
            left -= 1
            right += 1
        return right - left - 1
    
    for i in range(len(s)):
        # Odd length palindrome
        len1 = expand_around_center(i, i)
        # Even length palindrome
        len2 = expand_around_center(i, i + 1)
        
        curr_len = max(len1, len2)
        if curr_len > max_len:
            max_len = curr_len
            start = i - (curr_len - 1) // 2
    
    return s[start:start + max_len]
```

### Dry Run
```
Input: "babad"

i=0: odd: len1=1 (b), even: len2=0 → curr=1, start=0
i=1: odd: "a" → "aba" → len1=3, even: "ba" → 0 → curr=3, start=0
i=2: odd: "b" → len1=1, even: "ab" → 0 → curr=1
i=3: odd: "a" → len1=1, even: "ad" → 0 → curr=1
i=4: odd: "d" → len1=1, even: 0 → curr=1

Result: "bab" or "aba"
```

### Complexity
- **Time:** O(n²)
- **Space:** O(1)

### Quick Revision Notes
- Expand around each center (2n-1 centers)
- Check odd and even length
- O(n²) time, O(1) space

---

## Question #50: Edit Distance (Levenshtein Distance)

### Problem Statement
Find minimum number of operations (insert, delete, replace) to convert str1 to str2.

### Difficulty
Hard

### Pattern
DP (Dynamic Programming)

### Companies Asked
Wipro, TCS, Infosys, Amazon

### Approach
2D DP where dp[i][j] = edit distance between first i chars of word1 and first j chars of word2.

**Time Complexity:** O(m × n)
**Space Complexity:** O(m × n)

### Python Solution
```python
def edit_distance(word1: str, word2: str) -> int:
    """
    Computes minimum edit distance between two strings.
    """
    m, n = len(word1), len(word2)
    dp = [[0] * (n + 1) for _ in range(m + 1)]
    
    for i in range(m + 1):
        dp[i][0] = i
    for j in range(n + 1):
        dp[0][j] = j
    
    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if word1[i - 1] == word2[j - 1]:
                dp[i][j] = dp[i - 1][j - 1]
            else:
                dp[i][j] = 1 + min(
                    dp[i - 1][j],     # Delete
                    dp[i][j - 1],     # Insert
                    dp[i - 1][j - 1]  # Replace
                )
    
    return dp[m][n]
```

### Dry Run
```
Input: word1="horse", word2="ros"

DP Table:
    ""  r   o   s
""  0   1   2   3
h   1   1   2   3
o   2   2   1   2
r   3   2   2   2
s   4   3   3   2
e   5   4   4   3

dp[5][3] = 3

Result: 3 operations (h→r, r→s, delete e)
```

### Complexity
- **Time:** O(m × n)
- **Space:** O(m × n)

### Quick Revision Notes
- DP[i][j] = min(delete, insert, replace) + 1 if chars differ
- DP[i][j] = DP[i-1][j-1] if chars same
- Base: DP[i][0] = i, DP[0][j] = j

---

## 📊 Complete Revision Table

| # | Question | Pattern | Difficulty | Key Technique |
|---|----------|---------|-----------|--------------|
| 1 | Reverse String | Strings | Easy | Two pointers |
| 2 | Check Palindrome | Strings | Easy | Two pointers |
| 3 | Factorial | Math | Easy | Iteration/Recursion |
| 4 | Fibonacci | Math | Easy | DP iteration |
| 5 | Prime Check | Number Theory | Easy | √n optimization |
| 6 | Armstrong Number | Number Theory | Easy | Digit extraction |
| 7 | Count Vowels/Consonants | Strings | Easy | Set lookup |
| 8 | Largest Element | Arrays | Easy | Single pass |
| 9 | Remove Duplicates | Arrays/Hashing | Easy | Set tracking |
| 10 | Pyramid Pattern | Pattern Printing | Easy | String multiplication |
| 11 | Sum of Digits | Math | Easy | Mod/division |
| 12 | Perfect Number | Number Theory | Easy | √n divisor check |
| 13 | GCD | Math | Easy | Euclid's algorithm |
| 14 | Check Anagram | Strings | Easy | Frequency count |
| 15 | Diamond Pattern | Pattern Printing | Easy | Symmetric printing |
| 16 | Two Sum | HashMap | Easy | Complement lookup |
| 17 | Missing Number | Arrays/Math | Easy | Sum formula |
| 18 | Move Zeroes | Arrays/Two Pointers | Easy | In-place swap |
| 19 | First Non-Repeating | Strings/Hashing | Easy | Frequency count |
| 20 | Valid Parentheses | Stack | Easy | Stack matching |
| 21 | Second Largest | Arrays | Easy | Two-track variables |
| 22 | Reverse Words | Strings | Easy | Split/reverse/join |
| 23 | Remove Duplicates Sorted | Arrays/Two Pointers | Easy | Slow/fast pointers |
| 24 | Intersection | Arrays/Hashing | Easy | Set intersection |
| 25 | Number Pyramid | Pattern Printing | Easy | String join |
| 26 | Balanced Brackets | Stack | Easy | Stack matching |
| 27 | Longest Substring No Repeat | Sliding Window | Medium | HashMap + two pointers |
| 28 | Rotate Array | Arrays | Medium | Reversal algorithm |
| 29 | Array Subset | Hashing | Easy | Set operations |
| 30 | Find Duplicate | Arrays | Medium | Negative marking |
| 31 | Merge Sorted Arrays | Arrays | Easy | Two pointers |
| 32 | Maximum Subarray Sum | Arrays/DP | Medium | Kadane's algorithm |
| 33 | Find Pairs with Sum | HashMap | Easy | Complement lookup |
| 34 | Pascal's Triangle | Pattern/DP | Medium | Previous row sum |
| 35 | Palindrome Number | Math | Easy | Reverse half |
| 36 | String Compression | Strings | Easy | Run-length encoding |
| 37 | Count Frequencies | Hashing | Easy | Dictionary |
| 38 | String Rotation | Strings | Easy | s1+s1 trick |
| 39 | Majority Element | Voting Algorithm | Medium | Boyer-Moore |
| 40 | Spiral Matrix | Matrix | Medium | Boundary traversal |
| 41 | Binary Search | Binary Search | Medium | Divide and conquer |
| 42 | Peak Element | Binary Search | Medium | Binary search variant |
| 43 | Max Product Subarray | Arrays/DP | Medium | Min/Max tracking |
| 44 | Longest Common Prefix | Strings | Easy | Sorting + compare |
| 45 | Group Anagrams | Strings/Hashing | Medium | Sorted key |
| 46 | Tree Traversal | Trees | Medium | DFS recursion |
| 47 | Max Depth of Tree | Trees | Medium | Recursive height |
| 48 | Coin Change | DP | Medium | DP array |
| 49 | Longest Palindromic Substring | Strings/DP | Medium | Expand around center |
| 50 | Edit Distance | DP | Hard | 2D DP table |

---

## 🔥 Frequently Repeated Questions (Top 10 for Wipro)

1. **Two Sum** (HashMap, Easy) — Most asked coding question
2. **Check Palindrome** (Strings, Easy) — Wipro favorite
3. **Armstrong Number** (Number Theory, Easy) — Wipro specific
4. **Pattern Printing** (Pyramid/Diamond) — Wipro specific
5. **Prime Number Check** (Math, Easy) — Very common
6. **Factorial/Fibonacci** (Math, Easy) — Warm-up questions
7. **Remove Duplicates** (Arrays/Hashing) — Core concept
8. **Valid Parentheses** (Stack) — Stack fundamentals
9. **Missing Number** (Arrays) — Wipro frequently asks
10. **Anagram Check** (Strings) — String manipulation

---

## 🎯 Must Practice Before Interview (Priority Order)

### Priority 1 (Must Know - 100% chance)
- Reverse string, Check palindrome, Factorial, Fibonacci
- Prime check, Armstrong number, Sum of digits
- Pyramid/diamond pattern printing
- Two sum, Missing number, Remove duplicates
- Valid parentheses, Anagram check

### Priority 2 (High Probability - 70% chance)
- Move zeroes, First non-repeating character
- Find second largest, Reverse words
- Merge sorted arrays, Max subarray sum (Kadane)
- Binary search, String compression
- Number pyramid, Count frequencies

### Priority 3 (Medium Probability - 40% chance)
- Rotate array, Pascal's triangle
- String rotation, Majority element
- Spiral matrix, Maximum product subarray
- Longest common prefix, Group anagrams
- Tree traversal (DFS), Max depth of tree
- Coin change, Longest palindromic substring

### Priority 4 (Low but Possible - 20% chance)
- Intersection of arrays, Array subset
- Find duplicate (negative marking)
- Edit distance, Peak element

---

## ⚡ Most Important Python Tricks for Wipro

### String Tricks
```python
# Reverse string using slicing
s[::-1]

# Check palindrome
s == s[::-1]

# Join list to string
"".join(list_of_chars)

# Split string
words = s.split()

# Character frequency
from collections import Counter
freq = Counter(s)

# Check for anagram
Counter(s1) == Counter(s2)

# String rotation check
s2 in s1 + s1
```

### List/Dict Comprehension
```python
# List comprehension
squares = [x**2 for x in range(10)]

# Dict comprehension
squares_dict = {x: x**2 for x in range(10)}

# Filter with comprehension
evens = [x for x in nums if x % 2 == 0]

# All/any with generator
all(x > 0 for x in nums)
any(x == target for x in nums)
```

### Common Patterns
```python
# Swap two variables (Pythonic)
a, b = b, a

# Swap elements in list
nums[i], nums[j] = nums[j], nums[i]

# Two-pointer initialization
left, right = 0, len(arr) - 1

# Get complement (Two Sum)
complement = target - num

# Frequency dictionary
freq[num] = freq.get(num, 0) + 1

# Default dict
from collections import defaultdict
groups = defaultdict(list)

# Max with key
max(arr, key=lambda x: x["key"])

# Sorted with key
sorted(arr, key=lambda x: x[1])
```

### Math Tricks
```python
# Sum of first n natural numbers
n * (n + 1) // 2

# Digit extraction
last_digit = n % 10
n //= 10

# GCD
import math
math.gcd(a, b)

# Prime check up to sqrt(n)
import math
int(math.sqrt(n)) + 1

# Palindrome number (reverse half)
reversed_half = reversed_half * 10 + n % 10
```

### Input/Output
```python
# Read integer
n = int(input())

# Read array
arr = list(map(int, input().split()))

# Read multiple integers
a, b = map(int, input().split())

# Read full input until EOF
import sys
data = sys.stdin.read().split()
```

---

## ✅ Interview Checklist

### Before Coding Round
- [ ] Review Python basics (lists, dicts, sets, tuples)
- [ ] Practice pattern printing (pyramid, diamond, number)
- [ ] Practice string manipulation methods
- [ ] Review basic math problems (prime, factorial, Fibonacci)
- [ ] Go through Two Sum and similar HashMap problems
- [ ] Practice stack-based problems (parentheses)
- [ ] Review binary search implementation
- [ ] Practice at least 5 problems on HackerRank/LeetCode
- [ ] Time yourself (2 problems in 60 minutes)
- [ ] Set up Python environment (know how to debug)

### During Coding Assessment
- [ ] Read problem statement carefully (multiple times if needed)
- [ ] Clarify input/output format in comments
- [ ] Start with brute force, then optimize
- [ ] Write clean, readable code with type hints
- [ ] Handle edge cases (empty input, single element, large values)
- [ ] Add comments for key logic
- [ ] Test with example inputs
- [ ] Check for off-by-one errors
- [ ] Ensure time complexity is acceptable
- [ ] If stuck, move to next problem and come back

### After Coding Round (Technical Interview)
- [ ] Explain your approach clearly
- [ ] Discuss time/space complexity
- [ ] Be ready to optimize further
- [ ] Ask clarifying questions
- [ ] Think out loud while coding
- [ ] Handle follow-up questions confidently
- [ ] Show problem-solving methodology

### General Preparation
- [ ] Be comfortable with Python idioms
- [ ] Know basic DSA (Arrays, Strings, Stacks, Queues)
- [ ] Understand recursion and when to use it
- [ ] Practice on Wipro's own platform if available
- [ ] Review previous Wipro coding questions
- [ ] Focus on accuracy over speed initially

---

## 💡 Pattern Distribution Analysis

```
    Pattern Distribution for Wipro Questions
    
    Arrays          ████████████████     16  (32%)
    Strings         ████████████         12  (24%)
    Math            ███████              7   (14%)
    Pattern Printing ████                4   (8%)
    Hashing         ████                 4   (8%)
    DP              ███                  3   (6%)
    Stack           ██                   2   (4%)
    Binary Search   ██                   2   (4%)
```

### Key Insights:
- **Arrays + Strings = 56%** of all questions (the most important categories)
- **Math problems** = 14% (factorial, prime, Armstrong, GCD)
- **Pattern printing** = 8% (unique to service-based company interviews)
- **Data Structures** = 22% (HashMap, Stack, Trees, Binary Search)
- **DP problems** = 6% (only basic DP at medium level)

### What Wipro Specifically Looks For:
1. **Fundamentals first** — They test core programming concepts
2. **Clean code** — Readable, well-structured Python code
3. **Multiple approaches** — Brute force first, then optimization
4. **Edge case handling** — Empty inputs, single elements, negatives
5. **Pattern printing** — Unique to service companies like Wipro
6. **Math/number problems** — Armstrong, prime, palindrome numbers
7. **String manipulation** — Reversal, palindrome, anagram, compression
8. **Basic data structures** — HashMap, Stack, Two Pointers, Sliding Window

---

## 👨‍💻 Author

**Tamilselvan S**
- LinkedIn: https://www.linkedin.com/in/tamilselvan-ai/
- GitHub: `your-github-username`

---

*Happy Coding! Best of luck for your Wipro interview! 🚀*

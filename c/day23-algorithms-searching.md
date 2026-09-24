# C Day 23 — Introduction to algorithms: complexity and searching

**Goal:** describe how an algorithm's cost grows with its input (Big-O), measure it, and implement
linear search, binary search and two classic array techniques.

## Concepts

### What's an algorithm, and how do we compare them?

An **algorithm** is a precise sequence of steps that solves a problem. Two algorithms can solve the
same problem with wildly different speed, and the difference matters more as the input grows.
Measuring seconds depends on the machine, so we count **how the number of steps grows with the
input size n**.

### Big-O notation

Big-O keeps only the fastest-growing term and drops constants: `3n² + 10n + 7` steps is **O(n²)**.

| Class | Name | Example | n = 1,000 | n = 1,000,000 |
|---|---|---|---|---|
| O(1) | constant | array index, push to stack | 1 | 1 |
| O(log n) | logarithmic | binary search | ~10 | ~20 |
| O(n) | linear | linear search, sum of an array | 1,000 | 1,000,000 |
| O(n log n) | linearithmic | merge sort, `qsort` | ~10,000 | ~20,000,000 |
| O(n²) | quadratic | two nested loops over the data; bubble sort | 1,000,000 | 10¹² |
| O(2ⁿ) | exponential | naive recursive Fibonacci; trying all subsets | astronomically many | — |

At about 10⁸–10⁹ simple steps per second, O(n²) on a million elements takes minutes to hours;
O(n log n) takes a fraction of a second. **Choosing the right algorithm beats any
micro-optimization.**

How to find the Big-O of code: count loops. One loop over n → O(n). A loop inside a loop, both
over n → O(n²). A loop that halves the remaining range each time → O(log n). Call a function in a
loop → multiply by that function's cost (a `strlen` in a loop condition turns an O(n) loop into
O(n²)!).

Big-O usually describes the **worst case**. We also talk about the average case, and about
**memory** used (space complexity) in the same notation.

### Linear search: O(n)

Check every element. Works on anything, sorted or not:

```c
long linear_search(const int *a, size_t n, int key)
{
    for (size_t i = 0; i < n; i++) {
        if (a[i] == key) return (long)i;
    }
    return -1;
}
```

### Binary search: O(log n) — on sorted data

Look at the middle element. If the key is smaller, it can only be in the left half; if bigger, the
right half. Each step discards half of what remains, so a million elements take at most 20 steps.

```c
#include <stdio.h>

// Returns the index of key in the sorted array a, or -1.
long binary_search(const int *a, size_t n, int key)
{
    size_t lo = 0, hi = n;                 // search the half-open range [lo, hi)
    while (lo < hi) {
        size_t mid = lo + (hi - lo) / 2;   // not (lo + hi) / 2, which can overflow
        if (a[mid] < key) {
            lo = mid + 1;
        } else if (a[mid] > key) {
            hi = mid;
        } else {
            return (long)mid;
        }
    }
    return -1;
}

int main(void)
{
    int a[] = {2, 3, 5, 7, 11, 13, 17, 19, 23, 29};
    size_t n = sizeof a / sizeof a[0];
    printf("%ld %ld %ld\n", binary_search(a, n, 2), binary_search(a, n, 23),
           binary_search(a, n, 4));   // 0 8 -1
    return 0;
}
```

Binary search is famously easy to get subtly wrong. The half-open range `[lo, hi)` (like
`begin`/`end` on day 9) keeps it simple: the range is empty exactly when `lo == hi`.

A more useful variant is **lower bound**: the index of the first element `>= key` (or `n`). It
tells you where the key *would* go, and handles duplicates. C++'s `std::lower_bound` does this.

### Measuring time

```c
#include <time.h>

clock_t start = clock();
/* ... work ... */
double seconds = (double)(clock() - start) / CLOCKS_PER_SEC;
```

Compile with `-O2` (and without sanitizers) when timing, and make sure the result of the work is
used (print it), or the optimizer may delete the work entirely.

### Technique: two pointers

On a **sorted** array, find two numbers that add up to a target in O(n) instead of O(n²):

```c
// Returns 1 and sets *i, *j if a[*i] + a[*j] == target.
int pair_with_sum(const int *a, size_t n, int target, size_t *i, size_t *j)
{
    if (n < 2) return 0;
    size_t lo = 0, hi = n - 1;
    while (lo < hi) {
        int sum = a[lo] + a[hi];
        if (sum == target) { *i = lo; *j = hi; return 1; }
        if (sum < target) lo++;     // need bigger: move the left pointer right
        else hi--;                  // need smaller: move the right pointer left
    }
    return 0;
}
```

### Technique: prefix sums

To answer many "sum of `a[l..r]`" questions, precompute `prefix[i] = a[0] + ... + a[i-1]` once in
O(n). Then each query is `prefix[r + 1] - prefix[l]` — O(1) instead of O(n).

## Common mistakes

- Binary search on unsorted data.
- `mid = (lo + hi) / 2` overflowing for huge arrays; off-by-one in the loop bounds.
- Timing with sanitizers on, or at `-O0`.
- Hidden O(n) calls inside loops (`strlen`, `list_length`).
- Confusing "fast on my 10-element test" with "fast".

## Exercises

1. What's the Big-O of each?

   ```c
   // (a)
   for (size_t i = 0; i < n; i++) for (size_t j = 0; j < 10; j++) x++;
   // (b)
   for (size_t i = 0; i < n; i++) for (size_t j = i; j < n; j++) x++;
   // (c)
   for (size_t i = 1; i < n; i *= 2) x++;
   // (d)
   for (size_t i = 0; i < strlen(s); i++) if (s[i] == ' ') x++;
   // (e)
   for (size_t i = 0; i < n; i++) for (size_t j = 1; j < n; j *= 2) x++;
   ```

   <details><summary>Answers</summary>

   (a) O(n) — the inner loop is constant. (b) O(n²) — about n²/2 steps. (c) O(log n).
   (d) O(n²) — `strlen` is O(n) and runs every iteration (unless the optimizer hoists it).
   (e) O(n log n).

   </details>

2. Type in `binary_search` and test it on: an empty array, one element (found and not found), the
   first and last elements, and a missing value smaller/larger than everything.
3. Write `size_t lower_bound(const int *a, size_t n, int key)`. Use it to count how many times a
   value appears in a sorted array with duplicates (`lower_bound(key + 1) - lower_bound(key)`).
4. **Timing experiment:** fill an array of 10⁷ sorted ints; search for 1,000 random keys with
   linear and with binary search; time both. Then do it for 10⁶ and 10⁵. Does the ratio match
   what Big-O predicts?
5. Type in `pair_with_sum`. Then solve the same problem with two nested loops and compare times on
   10⁵ elements.
6. Use prefix sums to answer 10⁶ random range-sum queries on an array of 10⁶ elements. Compare with
   summing each range directly (try smaller sizes for the slow version!).
7. **Integer square root** by binary search: the largest `r` with `r * r <= n`, for `n` up to 10¹⁸
   (use `unsigned long long` and watch for overflow in `r * r`).
8. ★ **Maximum subarray (Kadane's algorithm):** find the largest sum of a contiguous subarray in
   O(n). First write the O(n²) version, then think about "the best sum ending at position i".

## Check yourself

1. What does O(n log n) mean in words?
2. Why is binary search O(log n)?
3. What's the precondition for binary search?
4. How can a single `strlen` call make a loop O(n²)?
5. Why should you compile with `-O2` when timing?

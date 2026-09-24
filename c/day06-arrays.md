# C Day 6 — Arrays

**Goal:** store many values of the same type, loop over them, pass them to functions, and work
with 2D arrays.

## Concepts

### Declaring and initializing

```c
int scores[5];                        // 5 ints, uninitialized (garbage) if local
int primes[5] = {2, 3, 5, 7, 11};     // initialized
int zeros[100] = {0};                 // first element 0, the rest are zero-filled too
int partial[5] = {1, 2};              // {1, 2, 0, 0, 0}
int auto_sized[] = {4, 8, 15, 16, 23, 42};  // size deduced: 6
```

Elements are numbered from **0** to **size − 1**. In memory they sit side by side, with no gaps:

```text
primes:  [  2 ][  3 ][  5 ][  7 ][ 11 ]
index:      0     1     2     3     4
```

The size must be a constant for arrays you declare like this (C99 allows variable-length arrays,
`int a[n];`, but they became optional in C11 and MSVC doesn't support them — avoid them in this
course).

### Looping over an array

```c
#include <stdio.h>

int main(void)
{
    int data[] = {4, 8, 15, 16, 23, 42};
    size_t n = sizeof data / sizeof data[0];   // number of elements: 24 bytes / 4 bytes = 6

    int sum = 0;
    for (size_t i = 0; i < n; i++) {
        sum += data[i];
    }
    printf("%zu elements, sum %d, average %.2f\n", n, sum, (double)sum / n);
    return 0;
}
```

`sizeof data` is the size of the whole array in bytes. Dividing by the size of one element gives
the element count. **This only works where the array itself is in scope** — see "arrays and
functions" below.

### Out of bounds: C doesn't check

```c
int a[5];
a[5] = 1;     // BUG: valid indexes are 0..4
```

C does **not** check array indexes. Writing past the end silently overwrites whatever is next in
memory — another variable, or the function's return address. This is **undefined behavior**: the
program might crash, might seem to work, or might misbehave somewhere unrelated. It's the source
of countless security holes. From now on, compile exercises with `-fsanitize=address` (see
[setup](../00-setup.md)) — it catches these.

### Arrays and functions

You can't pass an array by value. When you pass an array to a function, the function receives the
**address of its first element** (day 9 explains exactly how). Two consequences:

1. The function can **modify** the caller's array. (Compare with day 5, where an `int` parameter
   was a copy.)
2. The function **doesn't know the size**, so you must pass it separately.

```c
#include <stdio.h>

void fill(int arr[], size_t n, int value)
{
    for (size_t i = 0; i < n; i++) {
        arr[i] = value;              // modifies the caller's array
    }
}

int sum(const int arr[], size_t n)   // const: promises not to modify it
{
    int total = 0;
    for (size_t i = 0; i < n; i++) {
        total += arr[i];
    }
    return total;
}

int main(void)
{
    int nums[10];
    fill(nums, 10, 3);
    printf("%d\n", sum(nums, 10));   // 30
    return 0;
}
```

Inside `fill`, `sizeof arr` is the size of an address (8 bytes), *not* of the array — GCC even
warns if you try. Always pass the length.

### 2D arrays

```c
#include <stdio.h>

#define ROWS 3
#define COLS 4

void print_matrix(int m[][COLS], int rows)   // all but the first size must be given
{
    for (int r = 0; r < rows; r++) {
        for (int c = 0; c < COLS; c++) {
            printf("%4d", m[r][c]);
        }
        printf("\n");
    }
}

int main(void)
{
    int m[ROWS][COLS] = {
        {1, 2, 3, 4},
        {5, 6, 7, 8},
        {9, 10, 11, 12},
    };
    print_matrix(m, ROWS);
    return 0;
}
```

`#define ROWS 3` creates a named constant (day 18). A 2D array is stored **row by row** in one
continuous block: `m[1][0]` comes right after `m[0][3]`.

Why no `const` on `m`, when `sum` above had one? A quirk: before C23, passing an `int[3][4]` to a
`const int[][4]` parameter isn't allowed without a cast (`-Wpedantic` warns). C23 fixed it. For 1D
arrays, `const` works fine.

### A first look at strings

A string in C is an array of `char` ending with the **null character** `'\0'`:

```c
char name[] = "Ada";    // 4 chars: 'A' 'd' 'a' '\0'
```

Day 10 covers strings properly.

### Random numbers (for exercises)

```c
#include <stdlib.h>
#include <time.h>

srand((unsigned)time(NULL));   // once, at the start of main
int die = rand() % 6 + 1;      // 1..6
```

## Common mistakes

- Looping with `i <= n` — reads one element past the end.
- Using `sizeof arr / sizeof arr[0]` inside a function that received the array as a parameter.
- Forgetting that arrays passed to functions are modified in place.
- Assigning arrays: `a = b;` doesn't compile. Copy element by element (or use `memcpy`, day 10).
- Uninitialized local arrays — use `= {0}`.

## Exercises

1. Read 10 integers into an array. Print the minimum, maximum and average.
2. Write `void reverse(int arr[], size_t n)` that reverses an array **in place** (swap first with
   last, second with second-to-last, ...). Test with even and odd lengths.
3. Write `int index_of(const int arr[], size_t n, int value)` that returns the index of the first
   occurrence, or −1 if not found.
4. Roll a die 6000 times and count how often each face appears, using an array `counts[6]`. Print
   a histogram with one `*` per 50 rolls.
5. Write `size_t count_greater(const int arr[], size_t n, int limit)`.
6. Write `void rotate_left(int arr[], size_t n)` that moves every element one place to the left,
   and the first to the end.
7. For a 3×4 matrix, print the sum of each row and each column.
8. Transpose a 3×3 matrix in place (swap `m[r][c]` with `m[c][r]` — careful to swap each pair once).
9. Compile this with and without `-fsanitize=address` and compare what happens:

   ```c
   // BUG: out of bounds on purpose
   #include <stdio.h>
   int main(void)
   {
       int a[5] = {0};
       for (int i = 0; i <= 5; i++) {
           a[i] = i;
       }
       printf("%d\n", a[0]);
       return 0;
   }
   ```

10. ★ **Sieve of Eratosthenes:** use an array `is_composite[1001]` to find all primes up to 1000.

## Check yourself

1. What are the valid indexes of `int a[8]`?
2. Why doesn't `sizeof arr / sizeof arr[0]` work inside a function parameter?
3. Why can a function modify an array you pass it, but not an `int` you pass it?
4. How is `int m[3][4]` laid out in memory?

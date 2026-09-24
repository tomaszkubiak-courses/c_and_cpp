# C Day 9 — Pointers II: pointer arithmetic and arrays

**Goal:** understand how arrays and pointers relate, move pointers through arrays, and read the
`*p++` family of expressions without guessing.

## Concepts

### Adding to a pointer moves it by whole elements

If `p` points to an element of an array, `p + 1` points to the **next element** — not the next
byte. The compiler multiplies by the element size for you:

```c
#include <stdio.h>

int main(void)
{
    int a[4] = {10, 20, 30, 40};
    int *p = &a[0];

    for (int i = 0; i < 4; i++) {
        printf("p + %d = %p, *(p + %d) = %d\n", i, (void *)(p + i), i, *(p + i));
    }
    return 0;
}
```

Look at the addresses: they go up by 4 each time (the size of an `int`). With a `double` array
they'd go up by 8. That's why pointer types matter.

```text
 a:    [ 10 ][ 20 ][ 30 ][ 40 ]
 addr   1000  1004  1008  1012
        p     p+1   p+2   p+3
```

### An array name "decays" to a pointer to its first element

In almost every expression, the name of an array turns into a pointer to element 0:

```c
int a[4] = {10, 20, 30, 40};
int *p = a;          // same as: int *p = &a[0];
```

The exceptions: `sizeof a` (size of the whole array) and `&a` (address of the whole array — day
11). This conversion is called **decay**.

### a[i] is *(a + i)

The subscript operator is defined in terms of pointers: `a[i]` means `*(a + i)`. That's why:

- indexing works on pointers too: if `p = a`, then `p[2]` is `a[2]`;
- arrays start at index 0: `a[0]` is `*(a + 0)`, the first element;
- (a curiosity you should never use) `2[a]` also compiles, since `*(2 + a)` is the same thing.

And now day 6's mystery is solved: when you pass an array to a function, it decays to a pointer.
These three declarations are **identical**:

```c
int sum(int arr[], size_t n);
int sum(int arr[10], size_t n);   // the 10 is ignored!
int sum(int *arr, size_t n);
```

That's why the function can modify the array (it has its address), and why `sizeof arr` inside
it gives the size of a pointer.

### Walking an array with a pointer

Two equivalent ways to sum an array:

```c
#include <stdio.h>

int sum_index(const int *a, size_t n)
{
    int total = 0;
    for (size_t i = 0; i < n; i++) {
        total += a[i];
    }
    return total;
}

int sum_pointer(const int *a, size_t n)
{
    int total = 0;
    for (const int *p = a; p < a + n; p++) {   // a + n is "one past the end"
        total += *p;
    }
    return total;
}

int main(void)
{
    int data[] = {1, 2, 3, 4, 5};
    printf("%d %d\n", sum_index(data, 5), sum_pointer(data, 5));   // 15 15
    return 0;
}
```

The **one-past-the-end** pointer `a + n` is special: you may *compute* it and *compare* with it,
but never dereference it. Any pointer outside `a[0] .. a + n` (like `a - 1`) is undefined behavior
even to compute.

Many C functions (and all C++ algorithms) use a `(begin, end)` pair instead of `(pointer, count)`:

```c
int sum_range(const int *begin, const int *end)
{
    int total = 0;
    while (begin != end) {
        total += *begin++;
    }
    return total;
}
/* call: sum_range(data, data + 5) */
```

### Subtracting pointers

Two pointers into the same array can be subtracted; the result is the number of **elements**
between them, of type `ptrdiff_t` (print with `%td`):

```c
int a[10];
int *first = &a[2];
int *last = &a[7];
printf("%td\n", last - first);   // 5
```

Comparing pointers with `<`, `>` is only meaningful within the same array. Adding two pointers is
meaningless and doesn't compile.

### *p++ and friends

These appear constantly in C code. Precedence rule: postfix `++` binds tighter than `*`, but it
increments *after* the value is used.

| Expression | Meaning | Changes |
|---|---|---|
| `*p++` | use `*p`, then move `p` forward | the pointer |
| `(*p)++` | use `*p`, then add 1 to the pointee | the value |
| `*++p` | move `p` forward, then use `*p` | the pointer |
| `++*p` | add 1 to the pointee, then use it | the value |

`*p++ = value;` is the classic "store and advance" idiom.

### Returning a pointer into an array

Returning a pointer to an element of the **caller's** array is safe (unlike day 8's local
variable), because the array lives in the caller:

```c
#include <stdio.h>

// Returns a pointer to the first negative number, or NULL if there is none.
int *find_negative(int *a, size_t n)
{
    for (size_t i = 0; i < n; i++) {
        if (a[i] < 0) {
            return &a[i];          // or: return a + i;
        }
    }
    return NULL;
}

int main(void)
{
    int data[] = {3, 8, -2, 7, -9};
    int *neg = find_negative(data, 5);
    if (neg != NULL) {
        printf("found %d at index %td\n", *neg, neg - data);   // found -2 at index 2
        *neg = 0;                                               // we can change it
    }
    return 0;
}
```

### Arrays are not pointers

Decay makes them look alike, but:

| | `int a[5];` | `int *p;` |
|---|---|---|
| What it is | 5 ints of storage | one variable holding an address |
| `sizeof` | 20 | 8 (on 64-bit) |
| Can you assign to it? | no (`a = ...` is an error) | yes (`p = ...`) |
| `&x` gives | address of the whole array (type `int (*)[5]`) | address of the pointer (type `int **`) |

## Common mistakes

- Dereferencing the one-past-the-end pointer (`*(a + n)`).
- Walking backwards with `p >= a` as the condition: the loop computes `a - 1` at the end, which is
  undefined. Use `p != a` and decrement first (see exercise D2).
- `sizeof` on an array parameter.
- Confusing `*p++` (advance the pointer) with `(*p)++` (increment the value).
- Subtracting pointers from different arrays.

## Exercises

### A. Predict the output

1.
   ```c
   int a[] = {10, 20, 30, 40};
   int *p = a;
   printf("%d ", *p++);
   printf("%d ", *p);
   printf("%d ", (*p)++);
   printf("%d ", *p);
   printf("%d ", *++p);
   printf("%td\n", p - a);
   ```

2.
   ```c
   int a[] = {1, 2, 3, 4, 5};
   int *p = a + 4;
   int *q = a + 1;
   printf("%d %d %td %d\n", *p, *q, p - q, p[-2]);
   ```

3.
   ```c
   int a[5] = {0};
   int *p = a;
   for (int i = 0; i < 5; i++) {
       *p++ = i * i;
   }
   printf("%d %d %td\n", a[2], a[4], p - a);
   ```

4.
   ```c
   char s[] = "hello";
   char *p = s + 1;
   *p = 'a';
   p += 3;
   *p = 'y';
   printf("%s\n", s);
   ```

5.
   ```c
   double d[3] = {1.5, 2.5, 3.5};
   double *p = d;
   printf("%d %d\n", (int)(sizeof d), (int)(sizeof p));
   printf("%d\n", (int)((char *)(p + 1) - (char *)p));
   ```

6.
   ```c
   int a[] = {5, 6, 7, 8};
   int *p = a;
   int x = *p++ + *p++;
   /* is this line OK? */
   ```

<details><summary>Answers</summary>

1. `10 20 20 21 30 2` — `*p++` gives 10 and moves to a[1]; `(*p)++` gives 20 and makes a[1] 21;
   `*++p` moves to a[2] and gives 30; `p - a` is 2.
2. `5 2 3 3` — `p[-2]` is `*(p - 2)` = a[2] = 3. Negative indexes are fine as long as you stay
   inside the array.
3. `4 16 5` — `p` ends at one past the end.
4. `hally` — `s + 1` is the `'e'`, which becomes `'a'` ("hallo"); `p += 3` moves to `s[4]`, the
   `'o'`, which becomes `'y'`.
5. `24 8` then `8` — `sizeof` of the array vs of a pointer (on 64-bit); moving a `double *` by one
   moves 8 bytes. (Casting to `char *` measures the difference in bytes.)
6. **Not OK** — `p` is modified twice without the order being defined (unsequenced). This is
   undefined behavior. `-Wall` warns: "operation on 'p' may be undefined". Split it into two
   statements.

</details>

### B. Write it (no `[]` allowed in these — use only pointers)

1. `void print_array(const int *a, size_t n)` using a pointer that walks the array.
2. `int *max_element(int *a, size_t n)` — return a pointer to the largest element (NULL if
   `n == 0`). Use it in `main` to set the largest element to 0.
3. `void reverse(int *begin, int *end)` — reverse the range `[begin, end)` by walking one pointer
   forward and one backward until they meet. Test with 0, 1, 4 and 5 elements.
4. `void copy_array(int *dest, const int *src, size_t n)` using `*dest++ = *src++;`.
5. `size_t count_if_even(const int *begin, const int *end)`.
6. `int *find(int *begin, int *end, int value)` — return a pointer to the first match, or `end` if
   not found (the C++ convention). Why return `end` rather than `NULL`?
7. Print the addresses of the elements of a `char[4]`, `int[4]` and `double[4]` array. Explain the
   spacing.
8. ★ `void rotate_right(int *a, size_t n, size_t k)` — rotate by `k` positions. Hint: reverse the
   whole array, then reverse the first `k` and the rest separately, using your `reverse`.

### C. Arrays vs pointers

Given `int a[6]; int *p = a;`, which of these compile, and what does each mean? `a++`, `p++`,
`a = p`, `p = a + 2`, `*(a + 5) = 1`, `sizeof a`, `sizeof p`, `&a[6]`, `*(a + 6)`.

<details><summary>Answers</summary>

`a++` and `a = p` don't compile (an array isn't assignable). `p++` moves p to `a[1]`.
`p = a + 2` makes p point to `a[2]`. `*(a + 5) = 1` sets the last element. `sizeof a` is 24,
`sizeof p` is 8. `&a[6]` is the one-past-the-end pointer — fine to compute. `*(a + 6)` compiles
but is undefined behavior: it reads past the end.

</details>

### D. Find the bug

1.
   ```c
   // BUG
   #include <stdio.h>
   int main(void)
   {
       int a[5] = {1, 2, 3, 4, 5};
       int *p = a;
       while (*p != 0) {
           printf("%d\n", *p++);
       }
       return 0;
   }
   ```

2.
   ```c
   // BUG
   #include <stdio.h>
   void print_backwards(const int *a, size_t n)
   {
       for (const int *p = a + n - 1; p >= a; p--) {
           printf("%d ", *p);
       }
       printf("\n");
   }
   int main(void)
   {
       int a[] = {1, 2, 3};
       print_backwards(a, 3);
       return 0;
   }
   ```

<details><summary>Answers</summary>

1. The array has no 0 in it, so the loop runs past the end (ASan reports "stack-buffer-overflow").
   A sentinel only works if the data really contains it. Loop over `n` elements instead.
2. On the last iteration `p--` computes `a - 1`, which is undefined behavior (and with `n == 0`,
   `a + n - 1` is already outside). It usually "works", which is what makes it dangerous. Correct
   pattern: `for (const int *p = a + n; p != a; ) { --p; printf("%d ", *p); }`.

</details>

## Check yourself

1. If `p` is a `double *` holding address 1000, what's `p + 3`?
2. What does "array decay" mean, and what are the two exceptions?
3. Why is `a[i]` the same as `*(a + i)`?
4. Which pointer may you compute but not dereference?
5. What's the difference between `*p++` and `(*p)++`?

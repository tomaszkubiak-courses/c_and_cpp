# C Day 13 — Function pointers and void *

**Goal:** pass functions to functions (callbacks), use `qsort` and `bsearch`, and write generic
code with `void *`.

## Concepts

### Functions have addresses too

A function's code lives in memory, so it has an address, and you can store that address in a
**function pointer**:

```c
#include <stdio.h>

int add(int a, int b) { return a + b; }
int mul(int a, int b) { return a * b; }

int main(void)
{
    int (*op)(int, int);      // op: pointer to a function taking (int, int) and returning int

    op = add;                 // a function name decays to its address (like arrays)
    printf("%d\n", op(3, 4)); // 7 — call through the pointer
    op = mul;
    printf("%d\n", op(3, 4)); // 12
    return 0;
}
```

The parentheses in `int (*op)(int, int)` are essential: `int *op(int, int)` would declare a
*function* returning `int *`. Reading it right to left from the name: "op is a pointer to a
function (int, int) returning int".

`typedef` makes it readable:

```c
typedef int (*BinaryOp)(int, int);
BinaryOp op = add;
```

### Callbacks: let the caller decide what to do

```c
#include <stdio.h>

void for_each(int *a, size_t n, void (*action)(int *))
{
    for (size_t i = 0; i < n; i++) {
        action(&a[i]);
    }
}

void double_it(int *x) { *x *= 2; }
void print_it(int *x) { printf("%d ", *x); }

int main(void)
{
    int a[] = {1, 2, 3, 4};
    for_each(a, 4, double_it);
    for_each(a, 4, print_it);    // 2 4 6 8
    printf("\n");
    return 0;
}
```

`for_each` knows *how* to walk the array; the callback decides *what* to do with each element.
This is the idea behind C++ algorithms and lambdas.

### Dispatch tables

An array of function pointers replaces a long `switch`:

```c
#include <stdio.h>

double add(double a, double b) { return a + b; }
double sub(double a, double b) { return a - b; }
double mul(double a, double b) { return a * b; }
double dvd(double a, double b) { return a / b; }

int main(void)
{
    const char symbols[] = "+-*/";
    double (*const ops[])(double, double) = {add, sub, mul, dvd};

    for (int i = 0; i < 4; i++) {
        printf("8 %c 2 = %g\n", symbols[i], ops[i](8, 2));
    }
    return 0;
}
```

### void *: a pointer to "something"

A `void *` holds an address without saying what's there. Any object pointer converts to `void *`
and back automatically in C. You **can't** dereference a `void *` or do arithmetic on it — you
must convert it to a real pointer type first. `malloc` returns `void *`; `memcpy` takes `void *`.

A generic swap that works for any type — it swaps raw bytes:

```c
#include <stdio.h>
#include <string.h>

void swap_any(void *a, void *b, size_t size)
{
    unsigned char tmp[64];            // fixed buffer: handles elements up to 64 bytes
    if (size > sizeof tmp) {
        return;
    }
    memcpy(tmp, a, size);
    memcpy(a, b, size);
    memcpy(b, tmp, size);
}

int main(void)
{
    int x = 1, y = 2;
    double d = 1.5, e = 2.5;
    swap_any(&x, &y, sizeof x);
    swap_any(&d, &e, sizeof d);
    printf("%d %d %g %g\n", x, y, d, e);   // 2 1 2.5 1.5
    return 0;
}
```

The price of `void *`: the compiler can no longer check types. Pass the wrong `size` and you
corrupt memory silently.

### qsort: the standard library's generic sort

```c
void qsort(void *base, size_t count, size_t size,
           int (*compare)(const void *, const void *));
```

`qsort` doesn't know your element type, so you give it the size of each element and a comparison
function. The comparator receives **pointers to two elements** (as `const void *`) and returns
negative, zero or positive:

```c
#include <stdio.h>
#include <stdlib.h>

int compare_ints(const void *pa, const void *pb)
{
    int a = *(const int *)pa;          // convert back to the real type, then dereference
    int b = *(const int *)pb;
    return (a > b) - (a < b);          // -1, 0 or 1; avoids overflow of "a - b"
}

int main(void)
{
    int a[] = {42, -7, 19, 0, 3};
    size_t n = sizeof a / sizeof a[0];
    qsort(a, n, sizeof a[0], compare_ints);
    for (size_t i = 0; i < n; i++) {
        printf("%d ", a[i]);           // -7 0 3 19 42
    }
    printf("\n");
    return 0;
}
```

Don't write `return a - b;` — for large values of opposite sign it overflows.

### Sorting strings with qsort — the classic pointer puzzle

For an array of `const char *`, each **element** is a `const char *`, so the comparator receives
a **pointer to a `const char *`** — a `const char *const *` — not the string itself:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int compare_strings(const void *pa, const void *pb)
{
    const char *a = *(const char *const *)pa;   // pa points to an array element (a char *)
    const char *b = *(const char *const *)pb;
    return strcmp(a, b);
}

int main(void)
{
    const char *names[] = {"Linus", "Ada", "Grace", "Dennis"};
    size_t n = sizeof names / sizeof names[0];
    qsort(names, n, sizeof names[0], compare_strings);
    for (size_t i = 0; i < n; i++) {
        printf("%s\n", names[i]);
    }
    return 0;
}
```

```text
 names: [ ●-]--> "Linus"        compare_strings receives pa = &names[i]
        [ ●-]--> "Ada"          *pa is names[i], the char * to pass to strcmp
        [ ●-]--> ...
```

Passing `strcmp` itself as the comparator (`qsort(names, n, sizeof names[0], strcmp)`) is a very
common bug: `strcmp` would receive pointers to the pointers and compare their bytes as text.

### bsearch

`bsearch` finds an element in a **sorted** array in O(log n) (day 23 explains why), using the
same kind of comparator. It returns a pointer to the found element, or `NULL`:

```c
int key = 19;
int *found = bsearch(&key, a, n, sizeof a[0], compare_ints);
```

## Common mistakes

- `int *op(int, int)` instead of `int (*op)(int, int)`.
- Dereferencing a `void *` without converting it.
- In a `qsort` comparator for strings, casting to `const char *` instead of `const char *const *`.
- Comparators that return `a - b` (overflow) or that aren't consistent (sorting may misbehave).
- `bsearch` on an unsorted array.

## Exercises

1. Write `void map(int *a, size_t n, int (*f)(int))` that replaces each element with `f(element)`,
   and use it with `square`, `negate` and `abs_value` functions.
2. Write `size_t count_if(const int *a, size_t n, int (*pred)(int))`. Count evens, negatives and
   primes in an array with three predicate functions.
3. Turn day 4's calculator into a dispatch table indexed by operator: find the operator's position
   in `"+-*/%"` with `strchr` and call the matching function.
4. Sort an array of `int` in **descending** order with `qsort`.
5. Sort an array of `const char *` by **length**, then alphabetically for equal lengths.
6. Sort an array of `double` with `qsort`. Then use `bsearch` to look up a value.
7. Write your own generic `void my_sort(void *base, size_t n, size_t size, int (*cmp)(const void *, const void *))`
   using insertion sort and `swap_any`. You need `char *` arithmetic to find element `i`:
   `(char *)base + i * size`. Check that it gives the same results as `qsort` on ints and strings.
8. Write `void *find_if(void *base, size_t n, size_t size, int (*pred)(const void *))` returning a
   pointer to the first matching element, or NULL.
9. ★ Write a tiny event system: an array of up to 8 callbacks `void (*)(const char *msg)`,
   a `subscribe` function that adds one, and `publish` that calls all of them.

## Check yourself

1. Declare a pointer `f` to a function taking a `double` and returning a `double`.
2. Why can't you dereference a `void *`?
3. What exactly does a `qsort` comparator receive when sorting an array of `char *`?
4. Why is `return a - b;` a bad comparator?
5. What does `bsearch` require of the array?

# C Day 8 — Pointers I: addresses and indirection

**Goal:** understand what a pointer *is*, use `&` and `*` confidently, draw memory diagrams, and
write functions that change the caller's variables.

**Time:** ~2.5 h. This is the most important lesson of the C course. Go slowly, draw everything on
paper, and paste examples into [Python Tutor](https://pythontutor.com/c.html) to watch them run.

## Concepts

### Memory is a row of numbered boxes

Your program's memory is a long row of bytes, and every byte has a number — its **address**.
A variable occupies some bytes (an `int` usually 4) and its address is the address of its first
byte.

```c
int x = 42;
```

```text
 name   address   value
 ----   -------   -----
 x      1000      42
```

(Real addresses look like `0x7ffd5e8c3a4c`. In diagrams we use small made-up numbers like 1000.)

A variable has three things: a **name** (`x`), a **value** (`42`) and an **address** (`1000`).
Until today you've only used names and values.

### & — "address of"

`&x` gives you the address of `x`:

```c
#include <stdio.h>

int main(void)
{
    int x = 42;
    printf("value of x:   %d\n", x);
    printf("address of x: %p\n", (void *)&x);   // %p prints an address
    return 0;
}
```

(`%p` expects a `void *` — the cast keeps the compiler happy. Day 13 explains `void *`.)

### A pointer is a variable that holds an address

```c
int x = 42;
int *p = &x;      // p is a "pointer to int"; its value is the address of x
```

```text
 name   address   value
 ----   -------   -----
 x      1000      42      <--+
 p      1008      1000    ---+   p "points to" x
```

`p` is an ordinary variable. It has its own address (1008) and its own value (1000). What makes it
special is its **type**, `int *`: its value is meant to be the address of an `int`.

### * — "the thing pointed to" (dereference)

Given a pointer, `*p` means **the variable at the address stored in `p`**. Since `p` holds `&x`,
`*p` *is* `x` — you can read it and write it:

```c
#include <stdio.h>

int main(void)
{
    int x = 42;
    int *p = &x;

    printf("%d\n", *p);   // 42 — read x through p
    *p = 7;               // write x through p
    printf("%d\n", x);    // 7
    return 0;
}
```

Say it out loud while you read code:

| Code | Read it as |
|---|---|
| `int *p;` | "p is a pointer to int" |
| `p = &x;` | "p gets the address of x" / "p now points to x" |
| `*p = 7;` | "the int that p points to gets 7" |
| `y = *p;` | "y gets the int that p points to" |

### The two meanings of `*`

The same symbol does two different jobs, which is confusing at first:

- In a **declaration**, `int *p` says what type `p` has. Read it as "`*p` is an `int`" — so `p`
  is a pointer to an int.
- In an **expression**, `*p` follows the pointer.

And a trap: in `int *p, q;` only `p` is a pointer; `q` is a plain `int`. The `*` belongs to the
name, not the type. Declare one pointer per line.

### Changing a pointer vs changing what it points to

```c
#include <stdio.h>

int main(void)
{
    int a = 1, b = 2;
    int *p = &a;

    *p = 10;     // changes a (p still points to a)
    p = &b;      // changes p: now it points to b; a is untouched
    *p = 20;     // changes b

    printf("a=%d b=%d\n", a, b);   // a=10 b=20
    return 0;
}
```

`*p = ...` changes the **pointee**. `p = ...` changes the **pointer**. Keep them apart in your
head — most pointer bugs come from mixing them up.

### Pointers have types

An `int *` points to an `int`, a `double *` to a `double`. The type tells `*` how many bytes to
read and how to interpret them, and (tomorrow) how far `p + 1` moves. Don't mix them:
`double *d = &some_int;` is an error (or at least a warning you must not ignore).

### NULL: pointing at nothing

`NULL` (from `<stddef.h>`, `<stdio.h>`, `<stdlib.h>`) is a special pointer value meaning "points to
nothing". Initialize pointers that don't have a target yet to `NULL`, and check before
dereferencing:

```c
int *p = NULL;
/* ... */
if (p != NULL) {
    *p = 5;
}
```

Dereferencing `NULL` is undefined behavior; in practice it crashes ("segmentation fault"). An
**uninitialized** pointer is worse: it holds a random address, and writing through it corrupts
random memory.

### Why pointers? Functions that change the caller's variables

Remember day 5's broken `swap`? The function got copies. Now give it **addresses** instead:

```c
#include <stdio.h>

void swap(int *a, int *b)
{
    int tmp = *a;   // the int a points to
    *a = *b;
    *b = tmp;
}

int main(void)
{
    int x = 1, y = 2;
    swap(&x, &y);
    printf("x=%d y=%d\n", x, y);   // x=2 y=1
    return 0;
}
```

The function still receives copies — but copies of **addresses**. Through a copy of an address you
reach the original variable. Diagram at the moment `swap` starts:

```text
 main's frame                 swap's frame
 x  (1000)  1   <-----------  a  (2000)  1000
 y  (1004)  2   <-----------  b  (2008)  1004
```

**This is also why `scanf` needs `&`**: `scanf("%d", &n)` gives `scanf` the address of `n`, so it
can store the number there.

### Output parameters: returning more than one value

```c
#include <stdio.h>

void divide(int dividend, int divisor, int *quotient, int *remainder)
{
    *quotient = dividend / divisor;
    *remainder = dividend % divisor;
}

int main(void)
{
    int q, r;
    divide(17, 5, &q, &r);
    printf("17 = 5 * %d + %d\n", q, r);   // 17 = 5 * 3 + 2
    return 0;
}
```

A common C pattern: **return a success flag, and write the result through a pointer**:

```c
#include <stdio.h>

// Returns 1 and stores the value in *out on success; returns 0 on bad input.
int read_int(const char *prompt, int *out)
{
    printf("%s", prompt);
    return scanf("%d", out) == 1;   // out is already an address: no & here
}

int main(void)
{
    int age;
    if (!read_int("Age: ", &age)) {
        printf("Not a number\n");
        return 1;
    }
    printf("Next year: %d\n", age + 1);
    return 0;
}
```

(`const char *prompt` is a pointer to characters — a string. Day 10.)

### Never return the address of a local variable

```c
// BUG: returns a dangling pointer
int *make_number(void)
{
    int n = 42;
    return &n;      // n dies when the function returns!
}
```

When `make_number` returns, its stack frame (day 5) is gone, and `n` with it. The caller gets an
address of dead memory — a **dangling pointer**. Using it is undefined behavior: it may "work" in a
test and fail later. GCC warns ("function returns address of local variable"). Day 12 shows the
correct way to create memory that outlives a function.

## A notation for your diagrams

Use this in every pointer exercise from now on:

```text
 x [ 42 ]          a box per variable, value inside
 p [ ●--]--> x     a pointer: an arrow to the box it points to
 q [NULL]          a null pointer
 r [ ?? ]          uninitialized
```

When code runs, update the diagram line by line. When you're confused, it's almost always because
you skipped drawing.

## Common mistakes

- `int *p; *p = 5;` — writing through an uninitialized pointer.
- `int *p, q;` — `q` isn't a pointer.
- `p = 5;` when you meant `*p = 5;` (the compiler warns: "makes pointer from integer").
- `scanf("%d", &out)` where `out` is already an `int *` — that passes a pointer to the pointer.
- Returning the address of a local variable.

## Exercises

### A. Predict the output

Decide on the output of each snippet **before** opening the answer. Draw a diagram for each.

1.
   ```c
   int x = 5;
   int *p = &x;
   *p = 10;
   printf("%d\n", x);
   ```

2.
   ```c
   int a = 1, b = 2;
   int *p = &a;
   int *q = &b;
   *p = *q;
   p = q;
   *p = 7;
   printf("%d %d\n", a, b);
   ```

3.
   ```c
   int x = 10, y = 20;
   int *p = &x, *q = &y;
   int *tmp = p;
   p = q;
   q = tmp;
   printf("%d %d %d %d\n", x, y, *p, *q);
   ```

4.
   ```c
   void f(int n) { n = 100; }
   void g(int *n) { *n = 100; }
   /* in main: */
   int a = 1, b = 1;
   f(a);
   g(&b);
   printf("%d %d\n", a, b);
   ```

5.
   ```c
   int x = 4;
   int *p = &x;
   int *q = p;
   (*q)++;
   *p *= 10;
   printf("%d %d %d\n", x, *p, *q);
   ```

6.
   ```c
   void point_elsewhere(int *p) { static int other = 50; p = &other; }
   /* in main: */
   int x = 1;
   int *p = &x;
   point_elsewhere(p);
   printf("%d\n", *p);
   ```

7.
   ```c
   int a = 3, b = 4;
   int *p = &a, *q = &b;
   *p = *p + *q;
   *q = *p - *q;
   *p = *p - *q;
   printf("%d %d\n", a, b);
   ```

8.
   ```c
   int x = 1;
   int *p = &x;
   int *q = &x;
   printf("%d %d\n", p == q, *p == *q);
   int y = 1;
   q = &y;
   printf("%d %d\n", p == q, *p == *q);
   ```

<details><summary>Answers</summary>

1. `10` — `*p` is `x`.
2. `2 7` — `*p = *q` copies b's value (2) into a. Then `p` is redirected to b, and `*p = 7`
   changes b.
3. `10 20 20 10` — only the pointers were swapped; `x` and `y` never changed.
4. `1 100` — `f` changed its copy; `g` changed `b` through its address.
5. `50 50 50` — `p` and `q` both point to `x`: 4 → 5 → 50.
6. `1` — the pointer itself was passed by value. `point_elsewhere` changed its own copy of `p`;
   `main`'s `p` still points to `x`. (To change the caller's *pointer*, you need a pointer to a
   pointer — day 11.)
7. `4 3` — the swap-without-a-temporary trick, done through pointers.
8. `1 1` then `0 1` — `p == q` compares **addresses**; `*p == *q` compares **values**.

</details>

### B. Draw it

Draw the memory diagram after each line (variables, addresses you make up, arrows):

```c
int a = 5;
int b = 6;
int *p = &a;
int *q = &b;
*q = *p;
q = p;
*q = 9;
```

Final values: `a` = ?, `b` = ?, and where do `p` and `q` point? Check with Python Tutor.

### C. Write it

1. Fix your `swap` from day 5 using pointers. Test it with `x = 3, y = 8`.
2. Write `void sort3(int *a, int *b, int *c)` that rearranges three variables so that
   `*a <= *b <= *c`. Use your `swap`.
3. Write `void min_max(int a, int b, int c, int *min, int *max)`.
4. Write `void to_upper_char(char *ch)` that changes a lowercase letter to uppercase in place
   (`'a'`–`'z'` → subtract 32, or use `toupper` from `<ctype.h>`). Leave other characters alone.
5. Write `int safe_divide(int a, int b, int *result)` that returns 0 without touching `*result`
   when `b == 0`, else stores `a / b` and returns 1.
6. Write `void split_time(int total_seconds, int *h, int *m, int *s)`.
7. Write `void increment_all(int *a, int *b, int *c)` that adds 1 to each. Then call it as
   `increment_all(&x, &x, &x)`. What happens to `x`, and why?
8. Rewrite the day 7 grade book's `add_scores` as `void add_scores(int scores[], size_t *count, size_t capacity)`
   so it updates the caller's count directly.

### D. Find the bug

Each program has one pointer bug. Find it, explain it, fix it. Compile with
`-Wall -Wextra -fsanitize=address,undefined` and read what the tools say.

1.
   ```c
   // BUG
   #include <stdio.h>
   int main(void)
   {
       int *p;
       *p = 10;
       printf("%d\n", *p);
       return 0;
   }
   ```

2.
   ```c
   // BUG
   #include <stdio.h>
   int main(void)
   {
       int n;
       printf("Number: ");
       scanf("%d", n);
       printf("%d\n", n * 2);
       return 0;
   }
   ```

3.
   ```c
   // BUG
   #include <stdio.h>
   void set_to_zero(int *p)
   {
       p = 0;
   }
   int main(void)
   {
       int x = 5;
       set_to_zero(&x);
       printf("%d\n", x);   // expected 0
       return 0;
   }
   ```

4.
   ```c
   // BUG
   #include <stdio.h>
   int *bigger(int a, int b)
   {
       return a > b ? &a : &b;
   }
   int main(void)
   {
       int *m = bigger(3, 9);
       printf("%d\n", *m);
       return 0;
   }
   ```

<details><summary>Answers</summary>

1. `p` is uninitialized — it points to a random address. Point it at a real variable first
   (`int x; int *p = &x;`).
2. `scanf` needs the address: `&n`.
3. `p = 0` sets the local pointer to null; it should be `*p = 0`.
4. `a` and `b` are parameters — local variables of `bigger` — and die when it returns. Return the
   value (`int bigger(int a, int b)`), or take pointers to the caller's variables:
   `int *bigger(int *a, int *b) { return *a > *b ? a : b; }` — safe because those variables live
   in the caller.

</details>

## Check yourself

1. What are the name, value and address of a variable?
2. In `int *p = &x;`, what's the type of `p`? Of `*p`? Of `&x`?
3. What's the difference between `p = q;` and `*p = *q;`?
4. Why does `scanf` need `&`?
5. Why is returning `&local` a bug?
6. What's the difference between a `NULL` pointer and an uninitialized pointer?

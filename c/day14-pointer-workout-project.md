# C Day 14 — Pointer workout and project: a dynamic array

**Goal:** consolidate the pointer week. Morning: puzzles, diagrams and bug hunts mixing everything
from days 8–13. Afternoon: build a growable integer array library using only pointers.

## Pointer cheat sheet

| Syntax | Meaning |
|---|---|
| `T *p` | p holds the address of a T |
| `&x` | address of x |
| `*p` | the T that p points to |
| `p + i`, `p[i]` | i elements further; `p[i]` is `*(p + i)` |
| `q - p` | elements between two pointers into the same array (`ptrdiff_t`) |
| `const T *p` | can't modify `*p` through p |
| `T *const p` | can't change p itself |
| `T **pp` | pointer to a pointer; use to change the caller's `T *` |
| `T (*p)[N]` | pointer to an array of N T |
| `T *a[N]` | array of N pointers |
| `R (*f)(A)` | pointer to a function taking A and returning R |
| `void *` | untyped address; convert before use |
| `NULL` | points nowhere |

Rules to remember:

1. Everything in C is passed by value — including pointers. To change something in the caller,
   pass its address.
2. A pointer is only valid while the thing it points to is alive: stack variables die with their
   function, heap blocks die at `free`, and `realloc` may move them.
3. One-past-the-end may be computed and compared, never dereferenced.
4. Every `malloc` has exactly one owner and exactly one `free`.

## Part 1: Predict the output

Draw a diagram for each. Answers at the end of the section.

1.
   ```c
   int a[] = {1, 2, 3, 4, 5};
   int *p = a + 1, *q = a + 3;
   *p++ = *q--;
   printf("%d %d %d %td\n", a[1], *p, *q, q - p);
   ```

2.
   ```c
   char s[] = "pointer";
   char *p = s;
   while (*p) p++;
   printf("%td %c\n", p - s, *(p - 1));
   ```

3.
   ```c
   int x = 1, y = 2, z = 3;
   int *arr[] = {&x, &y, &z};
   int **pp = arr;
   **pp += 10;
   *(*(pp + 2)) *= 2;
   pp++;
   **pp = 0;
   printf("%d %d %d\n", x, y, z);
   ```

4.
   ```c
   void f(int *p) { p[1] = 99; p = NULL; }
   /* in main: */
   int a[3] = {1, 2, 3};
   int *p = a;
   f(p);
   printf("%d %d\n", a[1], p == a);
   ```

5.
   ```c
   const char *s = "abcdef";
   const char *t = s + 2;
   printf("%s %c %c %td\n", t, t[1], *s + 1, (s + 5) - t);
   ```

6.
   ```c
   int m[2][2] = {{1, 2}, {3, 4}};
   int *flat = &m[0][0];
   printf("%d %d\n", flat[2], *(flat + 3));
   ```

7.
   ```c
   int apply(int (*f)(int), int x) { return f(f(x)); }
   int inc(int x) { return x + 1; }
   int dbl(int x) { return x * 2; }
   /* in main: */
   printf("%d %d\n", apply(inc, 5), apply(dbl, apply(inc, 1)));
   ```

8.
   ```c
   void grow(int **p) { *p = malloc(2 * sizeof **p); (*p)[0] = 7; (*p)[1] = 8; }
   /* in main: */
   int *q = NULL;
   grow(&q);
   printf("%d %d\n", q[0], *(q + 1));
   free(q);
   ```

<details><summary>Answers</summary>

1. `4 3 3 0` — `*p++ = *q--` copies a[3] (4) into a[1], then p moves to a[2] and q to a[2].
   So `*p` and `*q` are both a[2] (3), and `q - p` is 0.
2. `7 r`
3. `11 0 6` — arr[0] is &x (x += 10), arr[2] is &z (z *= 2), then pp moves to arr[1] (&y) and y = 0.
4. `99 1` — `f` wrote through its copy of the pointer, then only nulled its copy.
5. `cdef d b 3` — `*s + 1` is `'a' + 1`, which is `'b'` (it adds to the character, not the pointer).
6. `3 4` — a 2D array is one continuous block. (Strictly, walking past the end of row 0 with a
   pointer to `m[0][0]` is a gray area in the standard; in practice every compiler lays it out like
   this, and the flat-index idiom from day 12 avoids the question.)
7. `7 12` — inc(inc(5)) = 7; apply(inc, 1) = 3, then dbl(dbl(3)) = 12.
8. `7 8` — `grow` allocated memory for the caller through the `int **`.

</details>

## Part 2: Find the bugs

Each program has at least one bug. Find them by reading first, then confirm with
`-Wall -Wextra -g -fsanitize=address,undefined`.

1.
   ```c
   // BUG
   #include <stdio.h>
   #include <stdlib.h>
   #include <string.h>
   char *shout(const char *s)
   {
       char buf[64];
       snprintf(buf, sizeof buf, "%s!", s);
       return buf;
   }
   int main(void)
   {
       printf("%s\n", shout("hello"));
       return 0;
   }
   ```

2.
   ```c
   // BUG
   #include <stdlib.h>
   int main(void)
   {
       int *a = malloc(4 * sizeof *a);
       if (a == NULL) return 1;
       int *keep = a + 1;
       a = realloc(a, 1000 * sizeof *a);
       *keep = 5;
       free(a);
       return 0;
   }
   ```

3.
   ```c
   // BUG
   #include <stdio.h>
   #include <stdlib.h>
   #include <string.h>
   int main(void)
   {
       const char *names[] = {"b", "a", "c"};
       qsort(names, 3, sizeof names[0], (int (*)(const void *, const void *))strcmp);
       printf("%s %s %s\n", names[0], names[1], names[2]);
       return 0;
   }
   ```

4.
   ```c
   // BUG
   #include <stdlib.h>
   #include <string.h>
   int main(void)
   {
       char *words[3];
       for (int i = 0; i < 3; i++) {
           words[i] = malloc(8);
           strcpy(words[i], "word");
       }
       free(words);
       return 0;
   }
   ```

5.
   ```c
   // BUG
   #include <stdio.h>
   void read_two(int *a, int *b)
   {
       scanf("%d %d", &a, &b);
   }
   int main(void)
   {
       int x, y;
       read_two(&x, &y);
       printf("%d\n", x + y);
       return 0;
   }
   ```

<details><summary>Answers</summary>

1. Returns the address of a local array (dangling). Allocate with `malloc` (and have the caller
   free), or let the caller pass in a buffer: `void shout(const char *s, char *out, size_t size)`.
2. `keep` points into the old block, which `realloc` may have freed (use after free). Also
   `a = realloc(a, ...)` loses the block if `realloc` fails. Recompute pointers from the new block
   after `realloc`, and use a temporary for the result.
3. `strcmp` gets pointers to the array elements (`const char **`), not the strings. The cast hides
   the problem from the compiler. Write a proper comparator (day 13).
4. `words` is a stack array — freeing it is an invalid free; the three heap strings leak. Free
   each `words[i]` in a loop instead.
5. `a` and `b` are already addresses; `&a` is the address of the local pointer. Use
   `scanf("%d %d", a, b)` (and check its return value).

</details>

## Part 3: Diagram drill

Draw the final state of stack and heap after this code, including which blocks are leaked:

```c
char *a = malloc(4);
char *b = malloc(4);
strcpy(a, "one");
strcpy(b, "two");
char *c = a;
a = b;
b = malloc(6);
strcpy(b, "three");
free(c);
```

<details><summary>Answer</summary>

`a` → "two", `b` → "three", `c` → freed block (dangling). Nothing is leaked yet: "one" was freed
through `c`, and "two" and "three" are still reachable through `a` and `b` — they must be freed
before the program ends.

</details>

## Part 4: Project — IntVec, a growable array

Build a small library of functions that manage a growable array of `int`. You haven't learned
`struct`s yet (tomorrow), so the array is described by **three variables** that you pass around
by pointer. This is deliberately pointer-heavy: every function takes `int **data`,
`size_t *size` and/or `size_t *capacity`.

```c
// All functions return 0 on success, -1 on failure (out of memory or bad index).

int  vec_init(int **data, size_t *size, size_t *capacity, size_t initial_capacity);
void vec_free(int **data, size_t *size, size_t *capacity);          // frees, sets *data = NULL
int  vec_push(int **data, size_t *size, size_t *capacity, int value); // grows ×2 when full
int  vec_pop(int *data, size_t *size, int *out);                      // removes the last element
int  vec_insert(int **data, size_t *size, size_t *capacity, size_t index, int value);
int  vec_remove(int *data, size_t *size, size_t index);               // shifts the rest left
int *vec_find(int *data, size_t size, int value);                     // pointer to element or NULL
void vec_print(const int *data, size_t size);                         // [1, 2, 3]
```

Requirements:

1. `vec_push` and `vec_insert` grow the capacity by doubling with `realloc`, using a temporary
   pointer. Why is doubling better than adding 1 each time? (Count the copies for 1000 pushes.)
2. `vec_insert` and `vec_remove` use `memmove` to shift elements (the regions overlap).
3. Write `main` as a test: push 1..20, print; insert 100 at index 0 and at the end; remove index 5;
   find 7 and change it through the returned pointer to 70; pop everything, printing each value;
   free. Assert the expected values with `assert` from `<assert.h>`:

   ```c
   assert(size == 21);
   assert(data[0] == 100);
   ```

4. It must run clean under `-fsanitize=address,undefined`, with no leaks.
5. ★ Add `int vec_shrink_to_fit(int **data, size_t size, size_t *capacity)`.
6. ★ Add `void vec_sort(int *data, size_t size)` using `qsort`.

Keep this code. Tomorrow you'll see how much simpler a `struct` makes it — and in the C++ course
you'll see `std::vector`, which is this exact data structure.

## Check yourself

If you can explain these to someone else, you've got pointers:

1. Why does `swap(&x, &y)` work when `swap(x, y)` can't?
2. Why is `a[i]` the same as `*(a + i)`, and what does it imply about passing arrays?
3. When does a function parameter need to be a `T **`?
4. Why is returning a pointer to a local variable wrong, but returning a pointer from `malloc`
   fine?
5. What exactly does a `qsort` comparator receive?

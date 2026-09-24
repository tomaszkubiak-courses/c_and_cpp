# C Day 12 — Dynamic memory: malloc, free and ownership

**Goal:** allocate memory whose size is known only at run time, or that must outlive a function;
free it correctly; and find memory bugs with the sanitizer.

## Concepts

### Stack and heap

So far every variable lived on the **stack**: created when its block starts, gone when it ends,
with a size fixed at compile time. Two things the stack can't do:

1. hold data whose size you only learn at run time ("read as many numbers as the user types");
2. keep data alive after the function that created it returns (day 8's dangling pointer).

The **heap** is a big pool of memory you manage by hand: you ask for a block with `malloc`, get
back a pointer to it, and give it back with `free` when you're done. It lives until you free it.

### malloc and free

```c
#include <stdio.h>
#include <stdlib.h>

int main(void)
{
    size_t n = 5;
    int *a = malloc(n * sizeof *a);     // room for n ints
    if (a == NULL) {                    // malloc returns NULL when it fails
        fprintf(stderr, "out of memory\n");
        return 1;
    }

    for (size_t i = 0; i < n; i++) {
        a[i] = (int)(i * 10);           // use it like an array
    }
    printf("%d\n", a[4]);               // 40

    free(a);                            // give it back
    a = NULL;                           // optional: avoids accidental reuse
    return 0;
}
```

```text
 stack                     heap
 a [ ●--]------------->   [ 0 ][ 10 ][ 20 ][ 30 ][ 40 ]
```

- `malloc(bytes)` returns a `void *` to an **uninitialized** block, or `NULL`. In C the `void *`
  converts to `int *` automatically — no cast needed.
- **Idiom:** `p = malloc(n * sizeof *p);` — `sizeof *p` is the size of what `p` points to, so it
  stays correct if you change `p`'s type later.
- `calloc(n, sizeof *p)` allocates **and zero-fills**, and checks `n * size` for overflow.
- `free(p)` releases the block. `free(NULL)` is allowed and does nothing.
- `a` itself is a normal stack variable holding the address. Freeing releases the heap block, not
  the pointer variable.

### Returning heap memory from a function

This is the fix for day 8's dangling pointer — the caller receives memory that outlives the
function, and becomes its **owner**:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

// Returns a newly allocated copy of s. The caller must free() it.
char *my_strdup(const char *s)
{
    size_t len = strlen(s);
    char *copy = malloc(len + 1);          // +1 for the '\0'!
    if (copy == NULL) {
        return NULL;
    }
    memcpy(copy, s, len + 1);
    return copy;
}

int main(void)
{
    char *name = my_strdup("Dennis");
    if (name == NULL) {
        return 1;
    }
    name[0] = 'd';
    printf("%s\n", name);   // dennis
    free(name);
    return 0;
}
```

### Ownership: the rule that keeps you sane

For every `malloc` there must be **exactly one** `free`, and at every moment exactly one piece of
code is responsible for it — the **owner**. Write it down in a comment on every function that
allocates ("The caller must free() the result") or takes ownership. C++ turns this rule into code
(`std::unique_ptr`, C++ day 8); in C, it lives in your comments and discipline.

### Growing an array with realloc

`realloc(p, new_size)` resizes a block, moving it if necessary (and copying the contents). Because
it can fail and return `NULL` — leaving the old block untouched — **never** assign the result
directly to the same pointer:

```c
#include <stdio.h>
#include <stdlib.h>

int main(void)
{
    size_t count = 0, capacity = 4;
    int *nums = malloc(capacity * sizeof *nums);
    if (nums == NULL) {
        return 1;
    }

    int value;
    printf("Numbers (end with Ctrl+D, or Ctrl+Z then Enter on Windows): ");
    while (scanf("%d", &value) == 1) {
        if (count == capacity) {
            capacity *= 2;                                   // double: amortized O(1) appends
            int *bigger = realloc(nums, capacity * sizeof *nums);
            if (bigger == NULL) {                            // old block is still valid here
                free(nums);
                return 1;
            }
            nums = bigger;
        }
        nums[count++] = value;
    }

    printf("read %zu numbers:", count);
    for (size_t i = 0; i < count; i++) {
        printf(" %d", nums[i]);
    }
    printf("\n");
    free(nums);
    return 0;
}
```

After a successful `realloc`, **the old pointer is invalid** — the block may have moved. Any other
pointers into the old block are now dangling too.

### Allocating through an output parameter (**)

Sometimes a function returns a status code and hands back allocated memory through a parameter.
That parameter must be a `T **` (day 11):

```c
#include <stdio.h>
#include <stdlib.h>

// On success, *out points to n ints set to 1..n; returns 0. The caller must free(*out).
int make_sequence(size_t n, int **out)
{
    int *a = malloc(n * sizeof *a);
    if (a == NULL) {
        return -1;
    }
    for (size_t i = 0; i < n; i++) {
        a[i] = (int)i + 1;
    }
    *out = a;           // write the address into the caller's pointer
    return 0;
}

int main(void)
{
    int *seq = NULL;
    if (make_sequence(5, &seq) != 0) {
        return 1;
    }
    printf("%d %d\n", seq[0], seq[4]);   // 1 5
    free(seq);
    return 0;
}
```

### Dynamic 2D arrays

Easiest and fastest: one block, index by hand:

```c
double *m = malloc(rows * cols * sizeof *m);
m[r * cols + c] = 1.0;   // element (r, c)
free(m);
```

Or an array of row pointers, if you want `m[r][c]` syntax — which needs one `free` per row plus one
for the row array (exercise 6).

### The memory bugs, and how to catch them

| Bug | What it is |
|---|---|
| **Leak** | never freeing a block (the pointer to it is lost) |
| **Use after free** | reading/writing a block after `free` |
| **Double free** | freeing the same block twice |
| **Invalid free** | freeing something that didn't come from `malloc` (a stack array, a literal, `p + 1`) |
| **Heap overflow** | writing past the end of a block (often the forgotten `+1` for `'\0'`) |
| **Uninitialized read** | reading `malloc`'d memory before writing it |

Compile with `-fsanitize=address` and ASan reports all of these except the last one, with the
line where the memory was allocated, freed and misused. Leaks are reported when the program exits.
On Linux, `valgrind --leak-check=full ./prog` (without sanitizers) is an alternative that also
catches uninitialized reads.

## Common mistakes

- `malloc(n)` instead of `malloc(n * sizeof *p)`.
- Forgetting `+ 1` for a string's terminator.
- `p = realloc(p, ...)` — leaks the old block when `realloc` fails.
- Keeping pointers into a block across a `realloc`.
- Not checking for `NULL`.
- Unclear ownership: two parts of the program both think they must free the same block.

## Exercises

1. **my_strdup**: type it in from above, then write `char *concat(const char *a, const char *b)`
   that returns a new string with both joined. Free everything in `main`.
2. **Read a line of any length:** `char *read_line(FILE *in)` — start with 16 bytes, read with
   `fgetc`, double with `realloc` when full, stop at `'\n'` or `EOF`, terminate, return the string
   (or NULL at end of input). Test with a very long line.
3. `int *filter_even(const int *a, size_t n, size_t *out_count)` — return a new array containing
   only the even numbers, and its length through `out_count`.
4. `char **split(const char *s, char sep, size_t *count)` — return a heap array of heap strings
   (`"a,b,c"` → `{"a", "b", "c"}`). Write `void free_split(char **parts, size_t count)` too.
   Draw the memory before coding: there are `count + 1` blocks.
5. Write `int resize(int **arr, size_t *capacity, size_t new_capacity)` that grows the caller's
   array with `realloc`, updating both the pointer and the capacity.
6. Allocate a `rows × cols` matrix as an array of row pointers (`int **m`), fill it with
   `r * cols + c`, print it, and free it correctly. Then do the same with a single block. Which is
   simpler?
7. **Bug hunt:** compile each with `-g -fsanitize=address` and read the report. Identify the bug
   type from the table above.

   ```c
   // BUG
   #include <stdlib.h>
   #include <string.h>
   int main(void)
   {
       char *s = malloc(strlen("hello"));
       strcpy(s, "hello");
       free(s);
       return 0;
   }
   ```

   ```c
   // BUG
   #include <stdio.h>
   #include <stdlib.h>
   int main(void)
   {
       int *a = malloc(3 * sizeof *a);
       if (a == NULL) return 1;
       a[0] = 1;
       free(a);
       printf("%d\n", a[0]);
       return 0;
   }
   ```

   ```c
   // BUG
   #include <stdlib.h>
   int main(void)
   {
       for (int i = 0; i < 10; i++) {
           int *p = malloc(100 * sizeof *p);
           if (p == NULL) return 1;
           p[0] = i;
       }
       return 0;
   }
   ```

   ```c
   // BUG
   #include <stdlib.h>
   int main(void)
   {
       int *a = malloc(10 * sizeof *a);
       int *b = a;
       free(a);
       free(b);
       return 0;
   }
   ```

   <details><summary>Answers</summary>

   Heap overflow (no room for `'\0'`) · use after free · leak (10 blocks, pointers lost each
   iteration) · double free (`a` and `b` are the same block).

   </details>

8. ★ Build a small "string builder": a struct-less set of functions `sb_append(char **buf,
   size_t *len, size_t *cap, const char *text)` that grows the buffer as needed. Use it to build a
   comma-separated list of the numbers 1–1000. (On day 15 you'll turn these three variables into a
   struct.)

## Check yourself

1. When do you need the heap instead of the stack?
2. Why write `malloc(n * sizeof *p)` rather than `malloc(n * 4)`?
3. Why must you not write `p = realloc(p, size)`?
4. What happens to other pointers into a block after `realloc` moves it?
5. Name four heap bugs that AddressSanitizer catches.
6. What does "owner" mean for heap memory?

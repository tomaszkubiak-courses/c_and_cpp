# C Day 11 — Pointers IV: const, pointers to pointers, arrays of pointers

**Goal:** read and write `const` pointer declarations, use `**` to change a caller's pointer,
work with arrays of strings and `argv`, and tell 2D arrays from arrays of pointers.

## Concepts

### const and pointers: two different things can be constant

```c
int x = 1, y = 2;

const int *a = &x;       // pointer to const int: can't change *a, can change a
int *const b = &x;       // const pointer to int: can change *b, can't change b
const int *const c = &x; // both fixed
```

| Declaration | `*p = 5` | `p = &y` |
|---|---|---|
| `const int *p` (same as `int const *p`) | no | yes |
| `int *const p` | yes | no |
| `const int *const p` | no | no |

**Read declarations right to left**, starting from the name:
`int *const b` → "b is a const pointer to int". `const int *a` → "a is a pointer to int that is
const". If a declaration looks scary, paste it into <https://cdecl.org>.

The one you'll use constantly is `const T *` in function parameters: it's a **promise** that the
function only reads through the pointer. `size_t strlen(const char *s)` tells you `strlen` won't
modify your string, and lets you pass string literals. Add `const` to every pointer parameter you
don't write through.

### Pointers to pointers

A pointer is a variable, so it has an address, so you can point to it:

```c
#include <stdio.h>

int main(void)
{
    int x = 5;
    int *p = &x;
    int **pp = &p;

    **pp = 9;                   // follows two arrows: changes x
    printf("%d %d %d\n", x, *p, **pp);   // 9 9 9
    return 0;
}
```

```text
 pp [ ●--]--> p [ ●--]--> x [ 9 ]
```

`*pp` is `p`. `**pp` is `x`.

### Why you need `**`: changing the caller's pointer

Day 8, puzzle 6: a function that received a pointer couldn't make the caller's pointer point
elsewhere, because the pointer was passed by value. It's the same rule as day 5, one level up:

| To change the caller's... | pass a... |
|---|---|
| `int` | `int *` |
| `int *` | `int **` |
| `char *` | `char **` |

A real use: a function that **advances** the caller's position in a string, like a tokenizer:

```c
#include <ctype.h>
#include <stdio.h>

// Moves *cursor past any spaces.
void skip_spaces(const char **cursor)
{
    while (isspace((unsigned char)**cursor)) {
        (*cursor)++;              // move the caller's pointer forward
    }
}

// Reads an unsigned integer at *cursor, advances *cursor past it.
// Returns 1 on success, 0 if there was no digit.
int read_number(const char **cursor, int *out)
{
    skip_spaces(cursor);
    if (!isdigit((unsigned char)**cursor)) {
        return 0;
    }
    int value = 0;
    while (isdigit((unsigned char)**cursor)) {
        value = value * 10 + (**cursor - '0');
        (*cursor)++;
    }
    *out = value;
    return 1;
}

int main(void)
{
    const char *text = "  12 7   300 x";
    const char *pos = text;
    int n;
    while (read_number(&pos, &n)) {
        printf("got %d, rest: \"%s\"\n", n, pos);
    }
    return 0;
}
```

Output:

```text
got 12, rest: " 7   300 x"
got 7, rest: "   300 x"
got 300, rest: " x"
```

Note `(*cursor)++` — the parentheses matter. `*cursor++` would move the local `cursor`, not the
caller's pointer. Tomorrow you'll use `**` again for functions that allocate memory for the caller.

### Arrays of pointers

```c
const char *days[] = {"Mon", "Tue", "Wed"};   // 3 pointers, each to a literal
```

```text
 days: [ ●-]--> "Mon\0"
       [ ●-]--> "Tue\0"
       [ ●-]--> "Wed\0"
```

`days[1]` is a `const char *` (the string "Tue"); `days[1][0]` is `'T'`. Sorting such an array
means **swapping pointers** — the strings themselves never move. That's fast, and it's exactly
what you'll do with `qsort` on day 13.

Compare with a 2D char array:

```c
char grid[3][4] = {"Mon", "Tue", "Wed"};   // 12 chars in one block, each row a copy
```

```text
 grid: [M][o][n][\0][T][u][e][\0][W][e][d][\0]
```

Every row must be the same length, and swapping two rows means copying characters.

### Command-line arguments: argc and argv

`main` can receive the command-line arguments:

```c
#include <stdio.h>

int main(int argc, char *argv[])      // also written: char **argv
{
    printf("%d argument(s)\n", argc);
    for (int i = 0; i < argc; i++) {
        printf("argv[%d] = \"%s\"\n", i, argv[i]);
    }
    return 0;
}
```

Run `./args one "two words" 3`:

```text
4 argument(s)
argv[0] = "./args"
argv[1] = "one"
argv[2] = "two words"
argv[3] = "3"
```

`argv` is an array of `char *`; `argv[0]` is the program name, and `argv[argc]` is `NULL`. All
arguments are strings — convert numbers with `strtol` (day 20) or `atoi` (quick but no error
checking).

### Pointers to arrays, and 2D arrays in functions

`int m[3][4]` is an array of 3 arrays of 4 ints. It decays to a pointer to its first element —
which is an **array of 4 ints** — so the type is `int (*)[4]`, "pointer to array of 4 int":

```c
#include <stdio.h>

void print(int (*rows)[4], int nrows)   // same as: int rows[][4]  (no const: see day 6)
{
    for (int r = 0; r < nrows; r++) {
        for (int c = 0; c < 4; c++) {
            printf("%3d", rows[r][c]);
        }
        printf("\n");
    }
}

int main(void)
{
    int m[3][4] = {{1, 2, 3, 4}, {5, 6, 7, 8}, {9, 10, 11, 12}};
    print(m, 3);
    return 0;
}
```

The parentheses matter:

| Declaration | Meaning |
|---|---|
| `int *a[4]` | array of 4 pointers to int |
| `int (*a)[4]` | pointer to an array of 4 ints |

A `int m[3][4]` is **not** an `int **`. There are no pointers stored in it — just 12 ints in a
row. Passing it to a function expecting `int **` is a bug (the compiler warns).

A common, simple alternative: store a matrix in a flat 1D array and compute the index yourself:
`a[r * cols + c]`. You'll do that tomorrow with dynamic memory.

## Common mistakes

- `*cursor++` when you meant `(*cursor)++`.
- Passing `int m[3][4]` where `int **` is expected.
- Forgetting `const` on read-only pointer parameters, which stops callers from passing literals.
- Swapping the strings in an array of pointers character by character, instead of swapping the
  pointers.

## Exercises

### A. Predict the output

1.
   ```c
   int x = 1, y = 2;
   int *p = &x;
   int **pp = &p;
   *pp = &y;
   **pp = 20;
   printf("%d %d %d\n", x, y, *p);
   ```

2.
   ```c
   const char *words[] = {"alpha", "beta", "gamma"};
   const char **w = words;
   printf("%s ", *w);
   printf("%s ", *(w + 2));
   printf("%c ", **(w + 1));
   printf("%c\n", *(*(w + 2) + 1));
   ```

3.
   ```c
   void redirect(int **pp, int *target) { *pp = target; }
   /* in main: */
   int a = 1, b = 2;
   int *p = &a;
   redirect(&p, &b);
   *p = 5;
   printf("%d %d\n", a, b);
   ```

4.
   ```c
   int m[2][3] = {{1, 2, 3}, {4, 5, 6}};
   int (*row)[3] = m;
   printf("%d %d %d\n", row[1][0], (*row)[2], *(*(row + 1) + 2));
   printf("%zu %zu\n", sizeof m, sizeof *row);
   ```

<details><summary>Answers</summary>

1. `1 20 20` — `*pp = &y` changes `p` to point to `y`.
2. `alpha gamma b a` — `**(w + 1)` is the first char of "beta"; `*(*(w + 2) + 1)` is the second
   char of "gamma".
3. `1 5` — `redirect` changed `main`'s `p` through `pp`.
4. `4 3 6` then `24 12` — `*row` is the first row (an array of 3 ints, 12 bytes).

</details>

### B. Which lines compile?

```c
int x = 1, y = 2;
const int *p = &x;
int *const q = &x;
*p = 5;       // (1)
p = &y;       // (2)
*q = 5;       // (3)
q = &y;       // (4)
```

<details><summary>Answer</summary>

(1) no — the pointee is const. (2) yes. (3) yes. (4) no — the pointer is const.

</details>

### C. Write it

1. **echo**: print all command-line arguments (without `argv[0]`) separated by spaces.
2. **sum**: `./sum 4 8 15` prints 27. Use `atoi` for now.
3. **calc**: `./calc 7 x 6` prints 42. Support `+ - x /` (why `x` and not `*`? Try `*` and see
   what your shell does with it).
4. Write `void sort_strings(const char *arr[], size_t n)` using insertion sort (day 7) and `strcmp`,
   swapping **pointers**. Test with an array of names.
5. Write `const char *longest(const char *arr[], size_t n)`.
6. Write `int next_word(const char **cursor, char *buf, size_t bufsize)` that copies the next
   whitespace-separated word into `buf`, advances `*cursor`, and returns 0 when there are no more
   words. Use it to print every word of a sentence on its own line.
7. Write `void split_path(const char *path, const char **dir_end, const char **ext)` that sets
   `*dir_end` to the last `'/'` and `*ext` to the last `'.'` (or NULL). Two output pointers.
8. ★ Decode these with cdecl.org, then write a line of code that uses each:
   `char **argv`, `int *(*f)[3]`, `const char *const names[]`.

## Check yourself

1. What's the difference between `const int *p` and `int *const p`?
2. When does a function need a `T **` parameter?
3. Why is `(*cursor)++` different from `*cursor++`?
4. Why can't you pass `int m[3][4]` as an `int **`?
5. What's `argv[argc]`?

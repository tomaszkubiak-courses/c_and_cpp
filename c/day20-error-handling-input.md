# C Day 20 — Error handling and robust input

**Goal:** report and handle errors the C way, clean up resources on every path, and read numbers
from users without crashing or accepting garbage.

## Concepts

### C has no exceptions

Errors are values. Common conventions:

| Convention | Example |
|---|---|
| return a status (0 = OK, negative = error) and pass results through pointers | `int parse_int(const char *s, int *out)` |
| return a pointer, `NULL` on failure | `fopen`, `malloc` |
| return a special value | `fgetc` returns `EOF`; `strchr` returns `NULL` |
| set `errno` | many library functions set the global `errno` to a code on failure |

Whatever you choose, **be consistent within a project** and document it in the header.

### errno, perror, strerror

```c
#include <errno.h>
#include <stdio.h>
#include <string.h>

int main(void)
{
    FILE *f = fopen("/no/such/file", "r");
    if (f == NULL) {
        int err = errno;                                // save it before calling anything else
        fprintf(stderr, "open failed: %s (errno %d)\n", strerror(err), err);
        return 1;
    }
    fclose(f);
    return 0;
}
```

`errno` is only meaningful right after a function that documents it has failed; successful calls
may leave any value in it. `perror("prefix")` prints `prefix: message` in one step.

### Cleanup on every path: the goto pattern

A function that acquires several resources must release them whether it succeeds or fails at any
step. Nested `if`s get deep quickly. The standard C idiom is a single cleanup section at the end
reached with `goto`:

```c
#include <stdio.h>
#include <stdlib.h>

// Copies the first n bytes of src to dst. Returns 0 on success, -1 on error.
int copy_prefix(const char *src, const char *dst, size_t n)
{
    int result = -1;
    FILE *in = NULL, *out = NULL;
    char *buf = NULL;

    in = fopen(src, "rb");
    if (in == NULL) goto cleanup;
    out = fopen(dst, "wb");
    if (out == NULL) goto cleanup;
    buf = malloc(n);
    if (buf == NULL) goto cleanup;

    size_t got = fread(buf, 1, n, in);
    if (fwrite(buf, 1, got, out) != got) goto cleanup;

    result = 0;                // everything worked
cleanup:
    free(buf);                 // free(NULL) is fine
    if (out != NULL && fclose(out) != 0) result = -1;
    if (in != NULL) fclose(in);
    return result;
}

int main(int argc, char *argv[])
{
    if (argc != 3) {
        fprintf(stderr, "usage: %s SRC DST\n", argv[0]);
        return 1;
    }
    if (copy_prefix(argv[1], argv[2], 64) != 0) {
        perror("copy_prefix");
        return 1;
    }
    return 0;
}
```

Rules that make it work: initialize every resource to "empty" (`NULL`) at the top, and make the
cleanup code safe for resources that were never acquired. This is the one use of `goto` that's
widely considered good style (the Linux kernel is full of it). C++ replaces it with destructors
(RAII, C++ day 6).

### assert vs error handling

- **User input, files, network, memory allocation** can fail in a correct program → handle with
  error returns.
- **"This can't happen unless my code is wrong"** (a NULL argument that the docs forbid, an index
  out of range computed by your own code) → `assert`.

### Robust number input: fgets + strtol

`scanf("%d", &n)` has problems: on "abc" it fails and leaves "abc" in the input, so a retry loop
spins forever; on "12abc" it accepts 12; on a huge number the behavior is undefined. The robust way
is to read a whole line and parse it with `strtol`, which reports exactly what happened:

```c
#include <errno.h>
#include <limits.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

// Parses all of s as a base-10 int (surrounding whitespace allowed).
// Returns 0 on success and stores the value in *out; returns -1 on any problem.
int parse_int(const char *s, int *out)
{
    char *end;
    errno = 0;
    long value = strtol(s, &end, 10);   // end: where parsing stopped

    if (end == s) return -1;                          // no digits at all
    while (*end == ' ' || *end == '\t' || *end == '\n') end++;
    if (*end != '\0') return -1;                      // trailing garbage like "12abc"
    if (errno == ERANGE || value < INT_MIN || value > INT_MAX) return -1;   // out of range

    *out = (int)value;
    return 0;
}

// Keeps asking until the user enters an int in [min, max]. Returns 0, or -1 at end of input.
int ask_int(const char *prompt, int min, int max, int *out)
{
    char line[128];
    for (;;) {
        printf("%s", prompt);
        if (fgets(line, sizeof line, stdin) == NULL) {
            return -1;                                // EOF or read error
        }
        if (strchr(line, '\n') == NULL && !feof(stdin)) {
            int c;
            while ((c = getchar()) != '\n' && c != EOF) {}   // discard the rest of a long line
            printf("Too long.\n");
            continue;
        }
        int value;
        if (parse_int(line, &value) == 0 && value >= min && value <= max) {
            *out = value;
            return 0;
        }
        printf("Please enter a whole number from %d to %d.\n", min, max);
    }
}

int main(void)
{
    int age;
    if (ask_int("Age: ", 0, 150, &age) != 0) {
        return 1;
    }
    printf("OK: %d\n", age);
    return 0;
}
```

`strtod` does the same for `double`, and `strtoul`/`strtoll` for other integer types.

### Exit codes

`return` from `main` or `exit(code)` from anywhere. Use `EXIT_SUCCESS` and `EXIT_FAILURE` from
`<stdlib.h>`. Scripts and build tools check these codes.

## Common mistakes

- Ignoring return values (`fclose`, `fwrite`, `scanf`, `malloc`...). GCC can warn for your own
  functions if you mark them `__attribute__((warn_unused_result))` (C23: `[[nodiscard]]`).
- Checking `errno` without a failure first, or after another call overwrote it.
- Leaking resources on early `return`s — use the cleanup pattern.
- Using `atoi`, which can't report errors ("abc" and "0" both give 0).
- `assert` for things that can legitimately fail.

## Exercises

1. Type in `parse_int` and test it with: `"42"`, `" -7 "`, `""`, `"abc"`, `"12abc"`,
   `"99999999999"`, `"2147483647"`, `"-2147483648"`. Write the tests as `assert`s in `main`.
2. Write `int parse_double(const char *s, double *out)` with `strtod`, and test it the same way.
3. Replace every `scanf` in your day 7 grade book with `ask_int`.
4. Rewrite your day 19 file-copy program with the `goto cleanup` pattern, checking every call
   (`fopen`, `fread`, `fwrite`, `fclose`). Test by copying to a directory that doesn't exist.
5. Write a config-file reader: lines are `key = value`, `#` starts a comment, blank lines are
   ignored. Store up to 32 pairs in an array of structs. Report errors with the line number
   (`config.txt:7: missing '='`).
6. Define an error enum for your IntVec library (`VEC_OK`, `VEC_NO_MEMORY`, `VEC_BAD_INDEX`) and a
   `const char *vec_strerror(VecError e)`. Update the functions to return it.
7. ★ Write `int read_matrix(const char *path, double **data, size_t *rows, size_t *cols)` that reads
   a whitespace-separated matrix file, validates that every row has the same number of columns, and
   cleans up correctly on every error path.

## Check yourself

1. Name three ways a C function can report an error.
2. Why save `errno` into a local variable immediately?
3. What problems does `scanf("%d")` have that `fgets` + `strtol` solves?
4. How does the `goto cleanup` pattern make sure everything is released?
5. When is `assert` the right tool, and when isn't it?

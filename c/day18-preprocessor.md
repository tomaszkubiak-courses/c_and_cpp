# C Day 18 — The preprocessor

**Goal:** know what the preprocessor does, write macros that don't bite, and use conditional
compilation and assertions.

## Concepts

### Text substitution before compilation

The preprocessor runs before the compiler and works on **text**: it pastes in headers, replaces
macros and removes code excluded by `#if`. It knows nothing about C types or scopes. See its output
with `gcc -E file.c`.

### Object-like macros

```c
#define BUFFER_SIZE 256
#define PI 3.14159265358979
```

Every later `BUFFER_SIZE` token becomes `256`. Conventions: UPPER_CASE names, no `;` at the end
(`#define N 10;` makes `int a[N];` into `int a[10;];`).

Alternatives that respect types and scope: `const double pi = 3.14...;` for values, and
`enum { BUFFER_SIZE = 256 };` for integer constants usable as array sizes. In C, macros or enums
are still the usual way to name array sizes.

### Function-like macros and their traps

```c
#define SQUARE(x) x * x          // broken
#define SQUARE_OK(x) ((x) * (x)) // parenthesize every parameter and the whole body
```

`SQUARE(1 + 2)` expands to `1 + 2 * 1 + 2` = 5. `SQUARE_OK(1 + 2)` is `((1 + 2) * (1 + 2))` = 9.

But even the "OK" version evaluates its argument **twice**: `SQUARE_OK(i++)` increments `i` twice
(undefined behavior). A `static inline` function has none of these problems and is just as fast:

```c
static inline int square(int x) { return x * x; }
```

**Prefer functions; use macros only for what functions can't do**: stringizing, token pasting,
`__FILE__`/`__LINE__`, conditional compilation, and generic tricks.

Multi-statement macros are wrapped in `do { ... } while (0)` so they behave like one statement,
even after an `if` without braces:

```c
#define SWAP_INT(a, b) do { int tmp_ = (a); (a) = (b); (b) = tmp_; } while (0)
```

### Macros that functions can't replace

```c
#include <stdio.h>

#define ARRAY_SIZE(a) (sizeof(a) / sizeof((a)[0]))
#define STR(x) #x                          // # turns the argument into a string literal
#define LOG(fmt, ...) fprintf(stderr, "[%s:%d] " fmt "\n", __FILE__, __LINE__, __VA_ARGS__)

int main(void)
{
    int data[] = {3, 1, 4, 1, 5, 9};
    printf("%zu elements\n", ARRAY_SIZE(data));
    printf("%s\n", STR(hello world));      // hello world
    LOG("value is %d", data[2]);           // [main.c:12] value is 4
    return 0;
}
```

- `ARRAY_SIZE` only works on real arrays — not on pointers or array parameters (day 9).
- `__FILE__`, `__LINE__`, `__DATE__` are predefined macros. `__func__` (the current function's
  name) is a predefined *variable* that works the same way.
- `...` and `__VA_ARGS__` make a variadic macro. (This `LOG` needs at least one argument after
  the format; C23 fixes that with `__VA_OPT__`.)
- `##` pastes tokens together: `#define MAKE_GETTER(name) int get_##name(void)`.

### Conditional compilation

```c
#ifdef DEBUG
    printf("debug: x = %d\n", x);
#endif

#if defined(_WIN32)
    const char *os = "Windows";
#elif defined(__linux__)
    const char *os = "Linux";
#else
    const char *os = "something else";
#endif
```

Define macros from the command line with `-D`: `gcc -DDEBUG prog.c` or `gcc -DLEVEL=3 prog.c`.
In CMake: `target_compile_definitions(app PRIVATE DEBUG)`.

`#error "message"` stops compilation — useful to reject unsupported configurations.

### X-macros: one list, many uses

Keep an enum and its names in sync by writing the list once:

```c
#include <stdio.h>

#define COLOR_LIST(X) \
    X(RED)            \
    X(GREEN)          \
    X(BLUE)

#define AS_ENUM(name) COLOR_##name,
#define AS_STRING(name) #name,

typedef enum { COLOR_LIST(AS_ENUM) COLOR_COUNT } Color;
static const char *const color_names[] = { COLOR_LIST(AS_STRING) };

int main(void)
{
    for (int c = 0; c < COLOR_COUNT; c++) {
        printf("%d = %s\n", c, color_names[c]);
    }
    return 0;
}
```

Add a color to `COLOR_LIST` and both the enum and the names update. A trailing `\` continues a
macro onto the next line.

### assert and static_assert

```c
#include <assert.h>

assert(index < size);          // run-time check: aborts with file/line if false
static_assert(sizeof(int) == 4, "this code assumes 32-bit int");   // compile-time check
```

- `assert` is for **programmer errors** — conditions that can only be false if the code has a bug.
  Don't use it for user input or I/O failures: compiling with `-DNDEBUG` (typical for release
  builds) removes all asserts.
- `static_assert` is checked by the compiler and costs nothing at run time. (In C11/C17 it's a
  macro from `<assert.h>` for the keyword `_Static_assert`; in C23 it's a keyword.)

## Common mistakes

- Missing parentheses in macro bodies and around parameters.
- Arguments with side effects (`MAX(i++, j)`).
- A `;` at the end of a `#define`.
- Side effects inside `assert(...)` — they disappear with `NDEBUG`.
- Using a macro where `const`, `enum` or an inline function would do.

## Exercises

1. Write `#define MAX(a, b) ((a) > (b) ? (a) : (b))`. Show with a small program that `MAX(i++, 5)`
   misbehaves. Replace it with a `static inline int max_int(int a, int b)`.
2. Look at `gcc -E` output for a file that uses `ARRAY_SIZE`, `STR` and `LOG`. Find your macros
   expanded.
3. Write a `DEBUG_PRINT(fmt, ...)` macro that prints file, line and function name, and compiles to
   nothing unless `DEBUG` is defined. Build with and without `-DDEBUG`.
4. Use an X-macro to define an enum of HTTP-like status codes with values
   (`X(OK, 200) X(NOT_FOUND, 404) ...`) and generate both the enum and a function
   `const char *status_text(int code)` with a `switch`.
5. Add `assert`s to your IntVec functions (e.g. `assert(v != NULL)`, `assert(index < v->size)`).
   Trigger one on purpose. Then compile with `-DNDEBUG` and see it vanish.
6. Add `static_assert(sizeof(IntVec) == 24, "...")` and explain why it holds on 64-bit Linux.
   ★ Would it hold on a 32-bit system?
7. Print which compiler and OS you're on using predefined macros: `__GNUC__`, `__clang__`,
   `_MSC_VER`, `__linux__`, `_WIN32`, `__STDC_VERSION__`. Compile with `-std=c17` and `-std=c23`
   and compare `__STDC_VERSION__`.

## Check yourself

1. When does the preprocessor run, and what does it know about types?
2. Why must macro parameters be parenthesized?
3. Why is `do { ... } while (0)` used in macros?
4. What's the difference between `assert` and `static_assert`?
5. Name three things macros can do that functions can't.

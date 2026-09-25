# C Day 2 — Types, variables and basic I/O

**Goal:** declare variables of the right type, print them with the right `printf` specifier,
and read numbers from the keyboard safely.

## Concepts

### Variables

A variable is a named piece of memory with a **type**. The type decides how many bytes it uses
and how those bytes are interpreted.

```c
int count = 10;        // declare and initialize
double price;          // declared but NOT initialized: its value is garbage
price = 9.99;          // assign later
```

**Always initialize.** A local variable that you read before assigning holds whatever was in that
memory — reading it is undefined behavior (day 28), and the compiler may do surprising things.

### The basic types

| Type | Typical size | Holds |
|---|---|---|
| `char` | 1 byte | a character or small integer |
| `short` | 2 bytes | small integer |
| `int` | 4 bytes | the "normal" integer |
| `long` | 8 bytes on Linux, 4 on Windows | larger integer |
| `long long` | 8 bytes | large integer |
| `float` | 4 bytes | floating point, ~7 significant digits |
| `double` | 8 bytes | floating point, ~15 significant digits — use this by default |
| `_Bool` / `bool` | 1 byte | true/false (`#include <stdbool.h>` for `bool`, `true`, `false`) |

Integer types come in `signed` (default) and `unsigned` versions: an `unsigned int` can't be
negative, so it holds 0 to about 4.29 billion instead of about ±2.1 billion.

### Why only a "typical" size?

The sizes in the table are what you'll see on today's common computers, but the C standard does not
promise them. It only promises **minimums**: an `int` must be at least 16 bits (2 bytes), a `long`
at least 32 bits (4 bytes), and so on. A compiler is allowed to make them bigger. That's why the
table says `long` is 8 bytes on Linux but 4 on Windows — both follow the rules.

Most of the time this doesn't matter: an `int` is plenty for a loop counter or someone's age. It
matters when the exact number of bits is part of the job — for example, reading a file format that
says "the next 4 bytes are a number". For those cases, `<stdint.h>` gives you types whose size is
written in their name:

```c
#include <stdint.h>

int8_t   a = -5;       // exactly 8 bits, signed:   -128 to 127
uint8_t  b = 200;      // exactly 8 bits, unsigned:  0 to 255
int32_t  c = 100000;   // exactly 32 bits, signed
uint64_t d = 0;        // exactly 64 bits, unsigned
```

Read the name piece by piece: `u` means unsigned (no negatives), the number is how many bits, and
`_t` just marks it as a type. So `uint8_t` is "unsigned, 8 bits" on every machine.

### size_t: the type for sizes and counts

`size_t` is a type with one job: holding the size of something in memory, or a count of things —
"how many bytes", "how many elements". It is unsigned, because a size can never be negative, and it
is guaranteed to be big enough for the largest object your program could have (on a 64-bit machine
it is usually 8 bytes).

You'll see it all the time because the standard library uses it. The `sizeof` operator (next
section) gives its answer as a `size_t`:

```c
size_t n = sizeof(int);   // n is 4 on most machines
printf("%zu\n", n);       // %zu is the printf specifier for size_t
```

It is defined in `<stddef.h>`, but `<stdio.h>` and `<stdlib.h>` provide it too, so you rarely need
to include `<stddef.h>` just for `size_t`.

### sizeof and limits

```c
#include <limits.h>
#include <stdio.h>

int main(void)
{
    printf("char:      %zu byte(s)\n", sizeof(char));
    printf("int:       %zu byte(s)\n", sizeof(int));
    printf("long:      %zu byte(s)\n", sizeof(long));
    printf("long long: %zu byte(s)\n", sizeof(long long));
    printf("double:    %zu byte(s)\n", sizeof(double));
    printf("int range: %d to %d\n", INT_MIN, INT_MAX);
    printf("unsigned int max: %u\n", UINT_MAX);
    return 0;
}
```

`sizeof(char)` is 1 by definition. Sizes are measured in bytes (8 bits on every machine you'll use).

### printf specifiers

| Specifier | Type |
|---|---|
| `%d` or `%i` | `int` |
| `%u` | `unsigned int` |
| `%ld`, `%lld` | `long`, `long long` |
| `%lu`, `%llu` | `unsigned long`, `unsigned long long` |
| `%zu` | `size_t` |
| `%f` | `double` (and `float`, which is converted to `double`) |
| `%e`, `%g` | `double` in scientific / shortest form |
| `%c` | a character |
| `%s` | a string |
| `%x`, `%X`, `%o` | `unsigned int` in hex / octal |
| `%p` | a pointer (day 8) |

Width and precision: `%5d` pads to 5 characters, `%-5d` left-aligns, `%05d` pads with zeros,
`%.3f` shows 3 decimals, `%8.2f` both.

For `<stdint.h>` types use the macros from `<inttypes.h>`:
`printf("%" PRId64 "\n", value);` — the string pieces are glued together by the compiler.

**A wrong specifier is a bug**, not a style issue: `printf("%d", 3.5)` prints garbage.
`-Wall` warns about it — another reason to always use it.

### Characters are small integers

```c
#include <stdio.h>

int main(void)
{
    char c = 'A';
    printf("%c has code %d\n", c, c);      // A has code 65
    printf("next letter: %c\n", c + 1);    // B
    printf("'7' - '0' = %d\n", '7' - '0'); // 7 — converting a digit character to its value
    return 0;
}
```

Single quotes `'A'` are one character; double quotes `"A"` are a string (day 10).

### Constants

```c
const double PI = 3.14159265358979;   // can't be changed after initialization
```

Also useful: integer literal suffixes `10u` (unsigned), `10L` (long), `10LL`; `3.0f` (float);
`0x1F` (hex), `017` (octal — beware, a leading zero means octal!).

### Reading input with scanf

```c
#include <stdio.h>

int main(void)
{
    int age;
    double height;

    printf("Age and height in meters: ");
    if (scanf("%d %lf", &age, &height) != 2) {
        printf("That wasn't two numbers.\n");
        return 1;
    }
    printf("In 10 years you'll be %d, and still %.2f m tall.\n", age + 10, height);
    return 0;
}
```

- `scanf` needs the **address** of each variable — that's the `&`. You'll understand exactly why
  on day 8. For now: `&` in front of each variable, except strings.
- For `double`, `scanf` uses `%lf` (but `printf` uses `%f`). Easy to forget.
- `scanf` returns how many values it read successfully. **Always check it** — if the user types
  "abc", nothing is read and your variables keep their garbage values.

`scanf` is fine for exercises. Day 20 shows the robust way to read input (`fgets` + `strtol`).

## Common mistakes

- Reading an uninitialized variable.
- `scanf("%d", age)` — missing `&`. Usually a crash. `-Wall` warns about it.
- `%d` for a `long`, `size_t` or `double`. Match specifiers to types.
- Integer overflow: `int big = 2147483647 + 1;` — **signed** overflow is undefined behavior in C.
  Unsigned arithmetic wraps around instead (`UINT_MAX + 1u == 0`), which is well-defined.
- Assuming `long` is 8 bytes. It's 4 on Windows.

## Exercises

1. Extend the `sizeof` program to print the size of every type in the table, plus `size_t`,
   `int8_t`, `int64_t` and `float`.
2. **Temperature converter:** read a temperature in Celsius (a `double`) and print it in
   Fahrenheit (`F = C * 9 / 5 + 32`) with one decimal.
3. Read two integers and print their sum, difference, product, quotient and remainder. What
   happens if the second number is 0? (Don't fix it yet — day 4 adds `if`.)
4. Print `UINT_MAX`, then `UINT_MAX + 1u`. Explain the result.
5. Read a single character with `scanf(" %c", &c)` (note the space: it skips whitespace) and
   print its numeric code, and the next three characters after it.
6. Print the number 255 in decimal, hex, octal, and zero-padded to width 8.
7. **Area calculator:** read the radius of a circle and print its area and circumference, each
   with 3 decimals, right-aligned in a 12-character column.
8. ★ Read a number of seconds (like 100000) into a `long long` and print it as
   days, hours, minutes and seconds.

## Check yourself

1. Why should you prefer `double` over `float`?
2. What's the difference between `'A'` and `"A"`?
3. What does `scanf` return, and why should you check it?
4. Which is well-defined, signed or unsigned overflow?
5. When would you use `int32_t` instead of `int`?

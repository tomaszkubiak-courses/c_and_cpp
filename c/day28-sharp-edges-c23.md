# C Day 28 — C's sharp edges, the tools that catch them, and what C23 adds

**Goal:** recognize undefined behavior, use every tool that finds bugs automatically, write safe
integer and bit code, and know what's new in C23.

## Concepts

### Undefined, unspecified, implementation-defined

The C standard classifies non-portable behavior:

| Kind | Meaning | Example |
|---|---|---|
| **Implementation-defined** | each compiler picks and documents a behavior | `sizeof(long)`; right-shifting a negative number |
| **Unspecified** | one of several outcomes, not documented | order in which function arguments are evaluated |
| **Undefined behavior (UB)** | the standard places **no requirements** at all | signed overflow, out-of-bounds access |

UB doesn't mean "crashes" or "gives a weird number". The optimizer is allowed to **assume UB never
happens**, and reasons from that. Classic example:

```c
int will_overflow(int x)
{
    return x + 1 < x;     // "overflow check" — but signed overflow is UB
}
```

GCC at `-O2` compiles this to `return 0;`: since `x + 1` can't overflow in a valid program, it can
never be less than `x`. Your safety check silently disappears.

### The common UB list

You've met most of these already:

1. Signed integer overflow (`INT_MAX + 1`).
2. Out-of-bounds array access, including reading one past the end.
3. Dereferencing NULL, dangling or uninitialized pointers.
4. Reading an uninitialized variable.
5. Use after free, double free.
6. Modifying a string literal.
7. Modifying a variable twice without sequencing (`i = i++ + 1;`, `a[i] = i++;`).
8. Shifting by a negative amount or by ≥ the width of the type (`1 << 32` for a 32-bit int), or
   shifting a 1 into/past the sign bit of a signed int.
9. Division by zero; `INT_MIN / -1`.
10. Accessing an object through a pointer of an incompatible type (**strict aliasing**): reading a
    `float` through an `int *`. Use `memcpy` (or `unsigned char *`) to reinterpret bytes.
11. Returning from a non-`void` function without a value, when the caller uses it.
12. Computing a pointer outside `[array, array + n]`.

### Safe integer arithmetic

- Check **before** operating: `if (a > INT_MAX - b) { /* would overflow */ }` (for positive `b`).
- Use unsigned types for bit manipulation and for values that wrap by design (hashes, checksums).
- Beware of mixing signed and unsigned (day 3) — `-Wsign-compare` and `-Wconversion` help.
- GCC/Clang built-ins: `__builtin_add_overflow(a, b, &result)` returns true on overflow. C23
  standardizes this as `ckd_add` in `<stdckdint.h>` (below).

### Bit manipulation recipes

```c
x & (1u << k)          // test bit k
x | (1u << k)          // set bit k
x & ~(1u << k)         // clear bit k
x ^ (1u << k)          // toggle bit k
x & (x - 1)            // clear the lowest set bit
x & -x                 // isolate the lowest set bit (x unsigned)
(x & (x - 1)) == 0     // x is a power of two (or zero)
```

### The tools, all together

| Tool | Catches | How |
|---|---|---|
| Warnings | many bugs at compile time | `-Wall -Wextra -Wpedantic -Wshadow -Wconversion` |
| AddressSanitizer | out-of-bounds, use-after-free, leaks | `-fsanitize=address` |
| UndefinedBehaviorSanitizer | overflow, bad shifts, misaligned access, NULL | `-fsanitize=undefined` |
| GCC static analyzer | leaks, double free, NULL deref — *without running* | `-fanalyzer` |
| clang-tidy | bug patterns, style, modernization | `clang-tidy file.c -- -std=c17` |
| Valgrind | memory errors, uninitialized reads | `valgrind ./prog` (no sanitizers) |
| cppcheck | static analysis | `cppcheck --enable=all .` |

A good habit for any C project: a **debug build** with warnings as errors plus ASan and UBSan, used
for all testing, and a **release build** with `-O2`.

## What C23 adds

C23 (ISO/IEC 9899:2024) is the newest standard. GCC 15 uses it by default; try these with
`-std=c23`:

```c
#include <stdbit.h>
#include <stdckdint.h>
#include <stdio.h>

constexpr int SIZE = 4;                      // real compile-time constants

int main(void)
{
    bool ok = true;                          // bool, true, false are keywords: no <stdbool.h>
    int *p = nullptr;                        // a proper null pointer constant
    auto x = 0b1010'1010;                    // auto type inference, binary literal, digit separators
    typeof(x) y = 3;                         // typeof: the type of an expression

    int arr[SIZE] = {};                      // empty initializer: zero everything
    static_assert(sizeof arr == SIZE * sizeof(int));   // keyword; message optional

    int sum;
    if (ckd_add(&sum, 2147483647, 1)) {      // checked arithmetic: true on overflow
        printf("overflow detected\n");
    }

    printf("%d %d %d %d %u\n", ok, p == nullptr, x, y, stdc_count_ones(255u));   // popcount
    return 0;
}
```

Other C23 additions worth knowing:

- **Attributes:** `[[nodiscard]]` (warn when a return value is ignored — great for error codes),
  `[[maybe_unused]]`, `[[deprecated]]`, `[[fallthrough]]`.
- `#embed "file.bin"` — include a binary file's bytes as an initializer.
- `strdup` and `strndup` are now standard; `memset_explicit` for wiping secrets.
- `__VA_OPT__` in variadic macros; `#elifdef` / `#elifndef`.
- `unreachable()` in `<stddef.h>`.
- Function declarations `f()` now mean "no parameters", like C++. Old K&R-style function
  definitions are gone.
- Enums can have a fixed underlying type: `enum Color : unsigned char { ... };`.

Many of these came from C++ — you'll meet `constexpr`, `auto`, `nullptr`, attributes and
`static_assert` again in the C++ course.

## Exercises

1. **UB hunt:** each snippet has undefined behavior. Name the rule it breaks, then run it with
   `-fsanitize=undefined` or `address` and read the report.

   ```c
   int a = INT_MAX; a++;
   int s = 1 << 31;
   int arr[4]; int v = arr[4];
   int i = 0; int b[3]; b[i] = i++;
   float f = 1.0f; int bits = *(int *)&f;
   ```

   <details><summary>Answers</summary>

   Signed overflow · shifting into the sign bit of a signed int (use `1u << 31`) · out of bounds ·
   unsequenced modification and use of `i` · strict aliasing (use
   `memcpy(&bits, &f, sizeof bits)`).

   </details>

2. Compile `will_overflow` at `-O0` and `-O2`, and look at the assembly on Compiler Explorer. Then
   write a correct version using `__builtin_add_overflow` (or `ckd_add` with `-std=c23`).
3. Write `int safe_multiply(int a, int b, int *out)` without built-ins — check all sign
   combinations against `INT_MAX / b` and `INT_MIN / b`. Test with edge values.
4. Implement with bit operations only: `is_power_of_two`, `next_power_of_two` (smallest power of
   two ≥ x), `reverse_bits` (32-bit), `count_set_bits`. Test each on 0, 1, and `UINT_MAX`.
5. Run your hash map (day 25) and linked list (day 21) code through `-fanalyzer` and `clang-tidy`.
   Fix or understand every finding.
6. Compile one of your programs with `-Wconversion`. How many warnings? Fix them properly (don't
   just cast them away blindly — think about each).
7. Rewrite a small earlier program in C23 style: `bool` without the header, `nullptr`,
   `constexpr`, `[[nodiscard]]` on your error-returning functions, `auto` where the type is obvious.
   Compile with `-std=c23`.

## Check yourself

1. What's the difference between undefined and implementation-defined behavior?
2. Why can the compiler delete `if (x + 1 < x)`?
3. Name five kinds of undefined behavior.
4. Which tools catch memory errors at run time, and which without running the program?
5. Name five features C23 added.

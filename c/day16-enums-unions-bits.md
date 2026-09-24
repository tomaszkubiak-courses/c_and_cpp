# C Day 16 — Enums, unions, bit flags and bit fields

**Goal:** name sets of values with enums, store "one of several types" with tagged unions, and
pack yes/no options into bits.

## Concepts

### enum: named integer constants

```c
#include <stdio.h>

typedef enum { RED, YELLOW, GREEN } Light;   // RED = 0, YELLOW = 1, GREEN = 2

Light next(Light l)
{
    switch (l) {
    case RED:    return GREEN;
    case GREEN:  return YELLOW;
    case YELLOW: return RED;
    }
    return RED;   // unreachable, but keeps the compiler happy
}

const char *light_name(Light l)
{
    static const char *const names[] = {"red", "yellow", "green"};
    return names[l];
}

int main(void)
{
    Light l = RED;
    for (int i = 0; i < 5; i++) {
        printf("%s ", light_name(l));
        l = next(l);
    }
    printf("\n");   // red green yellow red green
    return 0;
}
```

- Values start at 0 and count up; you can set them: `enum { OK = 0, NOT_FOUND = 404 };`.
- In C an enum is just an `int` — nothing stops `Light l = 42;`. (C++'s `enum class` fixes that.)
- A `switch` on an enum without a `default:` gets a warning (`-Wswitch`, in `-Wall`) when you
  forget a case. That's a reason to *omit* `default` for enums.

### Bit flags

Many yes/no options fit in one integer, one bit each:

```c
#include <stdio.h>

enum {
    PERM_READ  = 1u << 0,   // 0b001
    PERM_WRITE = 1u << 1,   // 0b010
    PERM_EXEC  = 1u << 2,   // 0b100
};

int main(void)
{
    unsigned perms = PERM_READ | PERM_WRITE;   // set two flags

    if (perms & PERM_WRITE)  printf("can write\n");   // test
    perms &= ~PERM_WRITE;                              // clear
    perms |= PERM_EXEC;                                // set
    perms ^= PERM_READ;                                // toggle
    printf("perms = %u\n", perms);                     // 4 (only EXEC)
    return 0;
}
```

The four operations to know by heart:

| Action | Code |
|---|---|
| set bit | `x \|= FLAG` |
| clear bit | `x &= ~FLAG` |
| toggle bit | `x ^= FLAG` |
| test bit | `if (x & FLAG)` |

This is how Unix file permissions, `open()` flags and countless APIs work.

### union: one of several types in the same memory

A `union` looks like a struct, but all members **share the same memory**; it's as big as its
biggest member, and only the member written last holds a meaningful value.

```c
union Number {
    int i;
    double d;
};
```

On its own a union doesn't remember which member is valid. So you pair it with an enum — a
**tagged union**, one of the most useful patterns in C:

```c
#include <stdio.h>

typedef enum { SHAPE_CIRCLE, SHAPE_RECT, SHAPE_TRIANGLE } ShapeKind;

typedef struct {
    ShapeKind kind;           // the tag: which union member is valid
    union {
        struct { double radius; } circle;
        struct { double w, h; } rect;
        struct { double base, height; } triangle;
    };                        // anonymous union (C11): access members directly
} Shape;

double area(const Shape *s)
{
    switch (s->kind) {
    case SHAPE_CIRCLE:   return 3.14159265358979 * s->circle.radius * s->circle.radius;
    case SHAPE_RECT:     return s->rect.w * s->rect.h;
    case SHAPE_TRIANGLE: return 0.5 * s->triangle.base * s->triangle.height;
    }
    return 0;
}

int main(void)
{
    Shape shapes[] = {
        {.kind = SHAPE_CIRCLE, .circle = {1.0}},
        {.kind = SHAPE_RECT, .rect = {2.0, 3.0}},
        {.kind = SHAPE_TRIANGLE, .triangle = {4.0, 5.0}},
    };
    for (size_t i = 0; i < 3; i++) {
        printf("area %.2f\n", area(&shapes[i]));
    }
    return 0;
}
```

This is how JSON values, tokens in a compiler, and messages in network protocols are usually
represented in C. C++ has `std::variant` for it (C++ day 12).

### Looking at raw bytes

Any object can be inspected byte by byte through an `unsigned char *` — always allowed:

```c
#include <stdio.h>

int main(void)
{
    unsigned int x = 0x11223344;
    const unsigned char *bytes = (const unsigned char *)&x;
    for (size_t i = 0; i < sizeof x; i++) {
        printf("%02x ", bytes[i]);
    }
    printf("\n");   // 44 33 22 11 on x86/ARM: little-endian, lowest byte first
    return 0;
}
```

### Bit fields

Struct members can have a width in bits:

```c
struct Flags {
    unsigned visible : 1;
    unsigned color   : 3;   // 0..7
    unsigned layer   : 4;   // 0..15
};
```

Convenient, but the exact layout is implementation-defined, so don't use bit fields to match a
file format or hardware register across compilers — use explicit masks and shifts for that.

## Common mistakes

- Reading a union member other than the last one written (except via `unsigned char`). In C it's
  allowed for "type punning" but easy to misuse; with a tag you never need it.
- Forgetting to update the tag when changing the union's active member.
- Using `&&` or `||` instead of `&` or `|` with flags.
- Flags that aren't powers of two (`FLAG_C = 3` overlaps A and B).
- Shifting signed values; use `1u << n`.

## Exercises

1. Write `enum Weekday` and `const char *weekday_name(enum Weekday d)` using a lookup array. Print
   the days of a week starting from any given day.
2. Write `void print_binary(unsigned x)` printing all 32 bits, with a space every 8. Use it to
   show what each flag operation above does.
3. Implement file permissions: flags for read/write/execute for owner/group/others (9 bits). Write
   `void perm_to_string(unsigned perms, char out[10])` producing `"rwxr-x---"`, and the reverse,
   `unsigned perm_from_string(const char *s)`.
4. Extend the tagged-union `Shape` with `perimeter` and a `SHAPE_SQUARE` kind. Compile with
   `-Wall` and see the warning in `switch` statements you didn't update.
5. Model a JSON-like value: a tagged union of `null`, `bool`, `number` (double) and `string`
   (`char *`, heap). Write `value_print` and `value_free`.
6. Write `int count_bits(unsigned x)` that counts the 1 bits (loop with `x & 1` and `x >>= 1`).
   ★ Then the faster version using `x &= x - 1`, which clears the lowest set bit each step.
7. Use the byte-inspection program on a `double` (1.0) and a negative `int` (-1). Explain the
   output.
8. ★ **Traffic light state machine:** use an enum for states and a table of
   `{state, duration_seconds, next_state}` structs; simulate 60 seconds, printing each change.

## Check yourself

1. What's the value of the third constant in `enum { A, B = 10, C };`?
2. How do you set, clear, toggle and test a bit flag?
3. What's a tagged union, and why is the tag needed?
4. Why shouldn't bit fields be used to describe a file format?

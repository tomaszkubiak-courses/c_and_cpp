# C Day 3 — Operators, expressions and conversions

**Goal:** use C's operators correctly and predict what happens when types mix.

## Concepts

### Arithmetic

`+ - * / %`. The one to watch is `/`:

```c
#include <stdio.h>

int main(void)
{
    printf("%d\n", 7 / 2);        // 3   — integer division truncates toward zero
    printf("%d\n", -7 / 2);       // -3
    printf("%d\n", 7 % 3);        // 1   — remainder
    printf("%d\n", -7 % 3);       // -1  — sign follows the left operand
    printf("%f\n", 7 / 2.0);      // 3.500000 — one double operand makes it double division
    printf("%f\n", (double)7 / 2);// 3.500000 — a cast does the same
    return 0;
}
```

If **both** operands are integers, you get integer division — even if you store the result in a
`double`: `double avg = sum / count;` is wrong when both are `int`. Write
`(double)sum / count`.

### Assignment and increment

`x += 3` means `x = x + 3`; likewise `-=`, `*=`, `/=`, `%=`, `&=`, `|=`, `<<=` ...

`x++` and `++x` both add 1. The difference is the value of the expression:

```c
int a = 5;
int b = a++;   // b = 5, a = 6  (post-increment: use, then increment)
int c = ++a;   // c = 7, a = 7  (pre-increment: increment, then use)
```

Don't modify the same variable twice in one expression: `i = i++ + 1;` is undefined behavior.
Keep increments on their own or in simple positions like `for (...; ...; i++)`.

### Comparison and logic

`== != < > <= >=` produce `1` (true) or `0` (false). In C, **any nonzero value is true**.

`&&` (and), `||` (or), `!` (not). They **short-circuit**: `a && b` doesn't evaluate `b` if `a` is
false. This is useful: `if (count != 0 && total / count > 10)` never divides by zero.

The classic mistake: `if (x = 5)` assigns 5 and is always true. You meant `==`. `-Wall` warns.

### Bitwise operators

They work on the individual bits of integers:

| Operator | Meaning | Example (on 4-bit values) |
|---|---|---|
| `&` | AND | `1100 & 1010 = 1000` |
| `\|` | OR | `1100 \| 1010 = 1110` |
| `^` | XOR | `1100 ^ 1010 = 0110` |
| `~` | NOT (flip all bits) | `~1100 = 0011` |
| `<<` | shift left (×2 per step) | `0011 << 1 = 0110` |
| `>>` | shift right (÷2 per step) | `0110 >> 1 = 0011` |

Common uses: `n & 1` is 1 if `n` is odd; `1u << k` is a number with only bit `k` set. Use
**unsigned** types for bit work — shifting negative numbers has pitfalls. Day 16 and day 28 go
further.

### Conditional operator

`condition ? value_if_true : value_if_false`:

```c
int max = (a > b) ? a : b;
```

### Precedence

C has 15 precedence levels; nobody remembers them all. Rules of thumb: `* / %` before `+ -`,
arithmetic before comparisons, comparisons before `&&`, `&&` before `||`, assignment last. The
famous trap: `x & 1 == 0` means `x & (1 == 0)`. **When in doubt, add parentheses.**

### Implicit conversions — where C bites

When you mix types in an expression, C converts them first:

1. **Integer promotion:** `char` and `short` become `int` before arithmetic.
2. **Usual arithmetic conversions:** the "smaller" operand is converted to the "larger" type —
   `int` + `double` → `double`; `int` + `long` → `long`.
3. **Signed + unsigned of the same size → unsigned.** This is the trap:

```c
#include <stdio.h>

int main(void)
{
    int a = -1;
    unsigned int b = 1;
    if (a < b)
        printf("-1 < 1, as expected\n");
    else
        printf("surprise: -1 is not less than 1u\n");  // this one prints!
    return 0;
}
```

`a` is converted to `unsigned`, and -1 becomes 4294967295. `-Wextra` warns about
signed/unsigned comparisons — don't ignore it.

Converting **double → int** truncates toward zero: `(int)3.99` is 3, `(int)-3.99` is -3.
To round, use `lround` from `<math.h>` (link with `-lm` on Linux).

### Floating point isn't exact

```c
#include <stdio.h>

int main(void)
{
    double x = 0.1 + 0.2;
    printf("%.17f\n", x);          // 0.30000000000000004
    printf("%d\n", x == 0.3);      // 0
    return 0;
}
```

0.1 has no exact binary representation, like 1/3 has no exact decimal one. Never compare doubles
with `==`; compare `fabs(a - b) < 1e-9` instead, and never store money in floating point (use
integer cents).

## Common mistakes

- `sum / count` with two ints when you wanted a fractional average.
- `=` instead of `==` in a condition.
- Mixing signed and unsigned in comparisons (especially `int i` vs `size_t n`).
- Relying on precedence instead of parentheses with `&`, `|`, `<<`.
- `i = i++;` and similar — undefined behavior.

## Exercises

1. Predict, then check, the output of each line:

   ```c
   printf("%d %d %d\n", 17 / 5, 17 % 5, -17 / 5);
   printf("%f %f\n", 17 / 5.0, (double)(17 / 5));
   int x = 3; int y = x++ * 2; printf("%d %d\n", x, y);
   printf("%d %d %d\n", 5 > 3, 5 == 3, !0);
   printf("%d %d\n", 6 & 3, 6 | 3);
   printf("%d %d\n", 1 << 4, 100 >> 2);
   ```

   <details><summary>Answers</summary>

   `3 2 -3` · `3.400000 3.000000` · `4 6` · `1 0 1` · `2 7` · `16 25`

   </details>

2. Read a number of seconds and print it as `h:mm:ss` (e.g. 3725 → `1:02:05`). Use `/` and `%`,
   and `%02d` for padding.
3. Read an integer and print whether it's even or odd twice: once using `%`, once using `&`.
4. Read three integers and print their average **as a double** with 2 decimals. Make sure
   `1 2 2` gives `1.67`, not `1.00`.
5. Swap two variables using a third temporary variable. ★ Then do it without a temporary using
   XOR (`a ^= b; b ^= a; a ^= b;`) and explain why it works — and why you shouldn't do it in real
   code.
6. Read a positive integer `n` and print 1 if it's a power of two, 0 otherwise, using the
   expression `n > 0 && (n & (n - 1)) == 0`. Work out by hand on paper why it works for 8 and
   fails for 12.
7. Write a program that shows the signed/unsigned trap with `int` and `size_t`, and compile it
   with `-Wall -Wextra`. Read the warning.
8. ★ Compute `0.1 + 0.2 == 0.3` and a fixed version that uses a tolerance.

## Check yourself

1. What is `9 / 2` in C? And `9 / 2.0`?
2. What does short-circuit evaluation mean, and how can it prevent a division by zero?
3. Why does `-1 < 1u` evaluate to false?
4. What does `x & 1` tell you about `x`?
5. Why shouldn't you use `double` for money?

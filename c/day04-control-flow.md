# C Day 4 — Control flow

**Goal:** make decisions and loops, and choose the right loop for the job.

## Concepts

### if / else

```c
if (score >= 90) {
    printf("A\n");
} else if (score >= 75) {
    printf("B\n");
} else {
    printf("C\n");
}
```

Braces are optional for a single statement, but **always use them**. Without them, adding a
second line to the "body" silently puts it outside the `if`.

### switch

`switch` compares one integer value against constants:

```c
#include <stdio.h>

int main(void)
{
    char op;
    double a, b;

    printf("Expression (e.g. 3 * 4): ");
    if (scanf("%lf %c %lf", &a, &op, &b) != 3) {
        printf("Bad input\n");
        return 1;
    }

    switch (op) {
    case '+':
        printf("%g\n", a + b);
        break;
    case '-':
        printf("%g\n", a - b);
        break;
    case '*':
    case 'x':               // two labels, same code
        printf("%g\n", a * b);
        break;
    case '/':
        if (b == 0) {
            printf("Division by zero\n");
        } else {
            printf("%g\n", a / b);
        }
        break;
    default:
        printf("Unknown operator '%c'\n", op);
    }
    return 0;
}
```

Without `break`, execution **falls through** into the next case. Sometimes that's what you want
(the `'*'`/`'x'` pair above), usually it's a bug. GCC's `-Wimplicit-fallthrough` (part of
`-Wextra`) warns about it.

### while, do-while, for

```c
// while: test first; body may run zero times
int n = 10;
while (n > 0) {
    n /= 2;
}

// do-while: body runs at least once — good for "ask until valid" input loops
int choice;
do {
    printf("Pick 1-3: ");
    if (scanf("%d", &choice) != 1) return 1;
} while (choice < 1 || choice > 3);

// for: initialization; condition; step — best when counting
for (int i = 0; i < 5; i++) {
    printf("%d ", i);
}
```

A `for` loop is a `while` loop with the counting bits gathered in one line. The variable `i`
declared in the `for` exists only inside the loop.

Conventional form: count from 0, test with `<`: `for (int i = 0; i < n; i++)` runs exactly `n`
times. Mixing up `<` and `<=` gives **off-by-one** errors, the most common loop bug.

### break and continue

`break` leaves the innermost loop (or `switch`). `continue` skips to the next iteration.

```c
#include <stdio.h>

int main(void)
{
    // print the first 5 odd numbers not divisible by 3
    int found = 0;
    for (int i = 1; ; i += 2) {        // no condition: loops until break
        if (i % 3 == 0) {
            continue;
        }
        printf("%d ", i);
        if (++found == 5) {
            break;
        }
    }
    printf("\n");                      // 1 5 7 11 13
    return 0;
}
```

### Nested loops

```c
#include <stdio.h>

int main(void)
{
    for (int row = 1; row <= 4; row++) {
        for (int col = 1; col <= row; col++) {
            printf("*");
        }
        printf("\n");
    }
    return 0;
}
```

Prints a triangle. To break out of *both* loops, use a flag variable, a `return` from a function
(day 5), or — acceptable in C — a `goto` to a label after the loops (day 20 shows the one common,
good use of `goto`).

## Common mistakes

- `;` right after the condition: `for (i = 0; i < n; i++);` or `if (x);` — the loop/`if` body is
  the empty statement. `-Wextra` often warns.
- Forgetting `break` in a `switch`.
- Off-by-one: `<=` where you meant `<`.
- Infinite loops because the loop variable never changes. Press **Ctrl+C** to stop a running
  program.
- Loops that compare `double` values with `==` or `!=` — they may never be exactly equal.

## Exercises

1. **FizzBuzz:** print 1 to 100, but "Fizz" for multiples of 3, "Buzz" for multiples of 5,
   "FizzBuzz" for multiples of both.
2. Print a 10×10 multiplication table with each number right-aligned in 4 characters.
3. Read `n` and print these shapes of height `n` (here `n = 4`):

   ```text
   *        ****       *
   **       ***       ***
   ***      **       *****
   ****     *       *******
   ```

4. Read a non-negative integer and print the sum of its digits (e.g. 4096 → 19). Use `% 10` and
   `/ 10` in a loop.
5. Read `n` and say whether it's prime. Only test divisors up to √n (loop while `d * d <= n`).
6. **Collatz:** starting from `n`, if it's even halve it, otherwise make it `3n + 1`; repeat until
   1. Print the sequence and the number of steps. Which starting number below 100 takes the most
   steps?
7. Write a menu loop: show "1) Add 2) Subtract 3) Quit", read the choice, perform the action on
   two numbers you read, and repeat until the user chooses 3. Use `do`/`while` and `switch`.
8. **Guessing game:** pick a secret number with `srand(time(NULL)); int secret = rand() % 100 + 1;`
   (`#include <stdlib.h>` and `<time.h>`). Let the user guess, answer "higher"/"lower", and count
   guesses.
9. ★ Print all prime numbers below 1000 and how many there are (168).

## Check yourself

1. When does a `do`/`while` body run at least once, and why is that useful for input?
2. What happens in a `switch` if you forget `break`?
3. How many times does `for (int i = 1; i <= 10; i += 3)` run? What are the values of `i`?
4. What does `continue` do in a `for` loop — is the step (`i++`) still executed?

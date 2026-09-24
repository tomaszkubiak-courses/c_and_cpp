# C Day 5 — Functions, scope and lifetime

**Goal:** split programs into functions, understand that C passes arguments **by value**, and
know where each variable lives and for how long.

## Concepts

### Defining and declaring functions

```c
#include <stdio.h>

int square(int x);                  // declaration (prototype): name, parameters, return type

int main(void)
{
    printf("%d\n", square(7));      // 49
    return 0;
}

int square(int x)                   // definition: the actual code
{
    return x * x;
}
```

The compiler reads top to bottom, so a function must be **declared before it's called**. Either
define it above `main`, or put a prototype above and the definition anywhere. On day 17,
prototypes move into header files.

- A function that returns nothing has return type `void`.
- A function with no parameters is written `f(void)` in C. (Empty parentheses `f()` meant
  "unspecified parameters" in C17 — avoid it.)
- `return` ends the function immediately. A non-`void` function must return a value on every path.

### Pass by value: the function gets a copy

This is the most important idea of today, and it's the reason pointers exist:

```c
#include <stdio.h>

void try_to_change(int n)
{
    n = 100;                           // changes the local copy only
    printf("inside:  n = %d\n", n);    // 100
}

int main(void)
{
    int value = 5;
    try_to_change(value);
    printf("outside: value = %d\n", value);   // still 5
    return 0;
}
```

When you call `try_to_change(value)`, a **new** variable `n` is created and initialized with a
copy of `value`. Changing `n` has no effect on `value`. So how would you write a function that
swaps two variables, or that "returns" two results? You can't — until day 8 (pointers).

### Scope: where a name is visible

- **Block scope:** a variable declared inside `{ }` is visible from its declaration to the
  closing `}`. Function parameters have block scope in the function body.
- **File scope:** a variable declared outside all functions is visible from there to the end of
  the file (a **global** variable).

An inner declaration **shadows** an outer one with the same name — legal, but confusing. GCC's
`-Wshadow` warns about it.

### Lifetime (storage duration): how long a variable exists

- **Automatic** — ordinary local variables. Created when the block is entered, destroyed when it
  ends. Each call to a function gets fresh ones.
- **Static** — globals, and locals declared `static`. Created once when the program starts,
  live until it ends, and are initialized to zero if you don't initialize them.
- **Allocated** — memory you request with `malloc` and release with `free` (day 12).

```c
#include <stdio.h>

int next_id(void)
{
    static int counter = 0;   // initialized once; keeps its value between calls
    counter++;
    return counter;
}

int main(void)
{
    printf("%d ", next_id());
    printf("%d ", next_id());
    printf("%d\n", next_id());  // 1 2 3
    return 0;
}
```

**Avoid global variables** for passing data between functions. They make it hard to see which
function changes what. Pass parameters and return values instead.

### The call stack

Each function call gets a **stack frame**: a block of memory holding its parameters and local
variables. Calling a function pushes a frame; returning pops it. Draw it like this for
`main` calling `square(7)`:

```text
 +------------------+
 | square: x = 7    |   <- top of stack (current function)
 +------------------+
 | main:            |
 +------------------+
```

When `square` returns, its frame — including `x` — is gone. Keep this picture in mind; on day 8
it explains why returning the address of a local variable is a bug.

### Recursion

A function may call itself. Every recursive function needs a **base case** that stops it:

```c
#include <stdio.h>

unsigned long long factorial(unsigned int n)
{
    if (n <= 1) {
        return 1;                    // base case
    }
    return n * factorial(n - 1);     // recursive case, on a smaller problem
}

int main(void)
{
    for (unsigned int i = 0; i <= 20; i++) {
        printf("%2u! = %llu\n", i, factorial(i));
    }
    return 0;
}
```

Each call has its own frame with its own `n`. Recursion that never reaches the base case fills the
stack and crashes ("stack overflow"). Anything recursive can be written with a loop; use recursion
when it makes the code clearer (trees, day 26; divide-and-conquer sorting, day 24).

### Useful library functions

`<math.h>`: `sqrt`, `pow`, `fabs`, `floor`, `ceil`, `round`, `sin`... On Linux, link the math
library with `-lm` at the **end** of the command: `gcc ... prog.c -o prog -lm`.
`<stdlib.h>`: `abs`, `rand`, `srand`, `exit`.

## Common mistakes

- Calling a function before it's declared.
- Forgetting to `return` a value on some path (`-Wall` warns: "control reaches end of non-void
  function").
- Expecting a function to change its argument (pass by value!).
- Recursion without a correct base case.
- Using a global where a parameter would do.

## Exercises

1. Write `int is_prime(int n)` returning 1 or 0, and use it to print primes up to 100.
2. Write `int gcd(int a, int b)` twice: iteratively (loop) and recursively, using
   `gcd(a, b) = gcd(b, a % b)`, `gcd(a, 0) = a`. Then `int lcm(int a, int b)` using `gcd`.
3. Write `double power(double base, int exp)` with a loop, handling negative exponents. Compare with
   `pow` from `<math.h>`.
4. Write `void print_reversed_digits(int n)` recursively (1234 prints `4321`), then
   `void print_digits(int n)` that prints them in order — just move the `printf` relative to the
   recursive call. Why does that work?
5. Write `void swap(int a, int b)` that tries to swap two variables and call it from `main`. It
   doesn't work. Write two sentences explaining why, using the words "copy" and "stack frame".
   (Keep this file — you'll fix it on day 8.)
6. Use a `static` local variable to write a function that prints how many times it has been
   called.
7. **Towers of Hanoi:** write `void hanoi(int n, char from, char to, char via)` that prints the
   moves to transfer `n` disks. How many moves does it print for `n = 10`? (2ⁿ − 1.)
8. ★ Write `long long fib(int n)` recursively and time `fib(40)` (just count seconds). Then write
   it with a loop. Why is the recursive one so slow? (Draw the calls for `fib(5)`.)

## Check yourself

1. What does "pass by value" mean?
2. What's the difference between scope and lifetime?
3. What does `static` do to a local variable?
4. What happens to a function's local variables when it returns?
5. What must every recursive function have?

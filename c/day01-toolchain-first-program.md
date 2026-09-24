# C Day 1 — The toolchain and your first program

**Goal:** write, compile and run a C program, and know what happens between the source file and
the running program.

**Time:** ~1.5 h. Make sure you've done the [setup](../00-setup.md) first.

## Concepts

### What C is

C is a small, compiled language from the early 1970s. It sits close to the hardware: you manage
memory yourself, and a C program does almost nothing you didn't ask for. Operating systems
(Linux, the Windows kernel), databases (SQLite, PostgreSQL), language runtimes (Python, Lua),
embedded firmware — all mostly C. Learning it teaches you how computers actually run programs.

The language is standardized by ISO. Versions are named by year: C89, C99, C11, **C17** (what
this course uses; it's C11 with fixes), and C23 (the newest; covered on day 28).

### Your first program

Create `hello.c`:

```c
#include <stdio.h>

int main(void)
{
    printf("Hello, world!\n");
    return 0;
}
```

Compile and run it:

```bash
gcc -std=c17 -Wall -Wextra -Wpedantic -g hello.c -o hello
./hello
```

Line by line:

- `#include <stdio.h>` — paste in the declarations from the standard I/O header, so the compiler
  knows what `printf` is. Lines starting with `#` are for the **preprocessor** (day 18).
- `int main(void)` — every program starts in `main`. It returns an `int` to the operating system;
  `void` means it takes no arguments. (Day 11 shows the version that takes command-line
  arguments.)
- `printf("...\n");` — print formatted text. `\n` is a newline. Every statement ends with `;`.
- `return 0;` — tell the OS the program succeeded. Any nonzero value means failure.

### From source to program: four stages

`gcc hello.c -o hello` actually runs four steps. You can stop after each one:

```bash
gcc -E hello.c -o hello.i    # 1. preprocess: expand #include and #define -> C text
gcc -S hello.i -o hello.s    # 2. compile: C -> assembly text
gcc -c hello.s -o hello.o    # 3. assemble: assembly -> machine code (object file)
gcc hello.o -o hello         # 4. link: object files + libraries -> executable
```

1. **Preprocessor** — handles `#` lines. Look at `hello.i`: your 6 lines became hundreds, because
   `stdio.h` was pasted in.
2. **Compiler** — checks your code and translates it to assembly for your CPU. Syntax and type
   errors come from here.
3. **Assembler** — turns assembly into an **object file** (`.o`): machine code with "holes" where
   it calls functions defined elsewhere (like `printf`).
4. **Linker** — fills those holes by combining object files and libraries (the C standard library
   contains `printf`). Errors like `undefined reference to 'foo'` come from the linker.

Why care? Because error messages tell you which stage failed, and on day 17 you'll build programs
from many `.c` files — that's all about stages 3 and 4.

### printf basics

`printf` takes a **format string**; each `%` placeholder is filled by the next argument:

```c
#include <stdio.h>

int main(void)
{
    printf("I am %d years old.\n", 30);
    printf("Pi is roughly %f\n", 3.14159);
    printf("Pi to 2 places: %.2f\n", 3.14159);
    printf("My initial is %c and my name is %s.\n", 'T', "Tom");
    printf("A literal percent sign: 100%%\n");
    return 0;
}
```

`%d` integer, `%f` floating point, `%c` one character, `%s` string, `%%` a `%` sign. Day 2 has
the full list. Escape sequences: `\n` newline, `\t` tab, `\\` backslash, `\"` quote.

### Comments

```c
// A line comment (since C99).
/* A block comment,
   which can span lines. */
```

### Reading compiler errors

Remove the `;` after `printf(...)` and compile. You'll get something like:

```text
hello.c: In function 'main':
hello.c:5:30: error: expected ';' before 'return'
```

`hello.c:5:30` is *file:line:column*. The compiler often notices a problem one line late — a
missing `;` is reported at the start of the *next* statement. **Always fix the first error first**;
later errors are often consequences of it.

## Common mistakes

- Forgetting `-std=c17` and the warning flags. Warnings catch real bugs; turn them on every time.
- Writing `void main()` — not standard C. Use `int main(void)`.
- Running `hello` instead of `./hello` on Linux (the current directory isn't searched by default).
- Editing the file but forgetting to recompile before running.

## Exercises

1. Write a program that prints your name, and on the next line your favorite programming
   language, using two `printf` calls.
2. Print this box, using `\t` for the gaps inside it:

   ```text
   +--------+
   |  C     |
   |  day 1 |
   +--------+
   ```

3. Break your program on purpose, one change at a time, and read each error: remove a `;`, remove
   the `#include`, misspell `printf` as `prinft`, remove the closing `}`. All of these are caught
   by the compiler. Now make a *linker* error: add the line `void greet(void);` above `main` (a
   promise that a function `greet` exists) and call `greet();` inside `main`, without ever writing
   `greet`. Compile and find the words "undefined reference" — that's the linker complaining that
   the promise was never kept.
4. Run the four stages by hand (the commands above). How many lines is `hello.i`
   (`wc -l hello.i`)? Open `hello.s` and find the string `"Hello, world!"`.
5. Change `return 0;` to `return 3;`. Run the program, then type `echo $?` — the shell prints the
   exit code of the last program.
6. Paste your program into [Compiler Explorer](https://godbolt.org), choose x86-64 gcc, and find
   the call to `puts` or `printf` in the assembly. ★ Why might the compiler call `puts` instead of
   `printf`?
7. Print a table of three items with prices, using `%.2f` so every price has two decimals.

## Check yourself

1. What are the four stages of building a C program, and which one reports "undefined reference"?
2. What does `#include <stdio.h>` do, and why is it needed for `printf`?
3. What does the return value of `main` mean?
4. Why is `-Wall -Wextra` worth typing every time?

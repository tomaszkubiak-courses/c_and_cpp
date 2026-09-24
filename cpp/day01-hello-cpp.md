# C++ Day 1 — From C to C++

**Goal:** compile C++20 programs, and use the everyday C++ tools that replace their C
equivalents: streams, `std::string`, `std::format`, namespaces, `auto` and range-based `for`.

This course assumes you've done the [C course](../c/) or know C. If you haven't, C++ is still
learnable from here, but read the C pointer lessons (C days 8–12) when C++ day 8 comes around.

## Concepts

### What C++ is

C++ started in 1979 as "C with classes" and grew into a large multi-paradigm language: you can
write C-style code, object-oriented code, generic code with templates, and functional-style code
with lambdas and ranges. Its core promise is **zero-overhead abstraction**: high-level features
that compile down to code as fast as what you'd write by hand in C.

Versions: C++98, C++11 (the big modernization — "modern C++" starts here), C++14, C++17, **C++20**
(this course — concepts, ranges, `std::format`, modules, coroutines), C++23, and C++26.

### Hello, C++

`hello.cpp`:

```cpp
#include <iostream>
#include <string>

int main()
{
    std::string name;
    std::cout << "Your name: ";
    std::getline(std::cin, name);
    std::cout << "Hello, " << name << "!\n";
}
```

```bash
g++ -std=c++20 -Wall -Wextra -Wpedantic -g hello.cpp -o hello
./hello
```

What's different from C:

- `.cpp` files, compiled with `g++` (or `clang++`, or MSVC's `cl /std:c++20 /EHsc`).
- Headers of the C++ standard library have no `.h`: `<iostream>`, `<string>`, `<vector>`.
- `int main()` — in C++, empty parentheses mean "no parameters". And `main` (only `main`) may
  omit `return 0;`.
- `std::cout << x` prints `x`; `<<` can be chained. `std::cin >> x` reads (like `scanf`, it stops
  at whitespace). `std::getline(std::cin, s)` reads a whole line into a `std::string`.
- `std::` is the **namespace** of the standard library.

### Namespaces

A namespace groups names to avoid clashes: your `sort` and `std::sort` can coexist.

```cpp
namespace geometry {
    double area(double w, double h) { return w * h; }
}

double a = geometry::area(2, 3);
```

You'll see `using namespace std;` in many tutorials. It dumps every standard name into your scope
and can cause confusing clashes. Avoid it at file scope, and never in headers. `using std::cout;`
for a single name is fine, and so is writing `std::` — you'll stop noticing it within a week.

### std::string: strings that manage themselves

```cpp
#include <iostream>
#include <string>

int main()
{
    std::string s = "Hello";
    s += ", world";                         // append: memory grows automatically
    std::cout << s << " has " << s.size() << " characters\n";
    std::cout << "first: " << s[0] << ", last: " << s.back() << '\n';
    std::cout << "sub: " << s.substr(7, 5) << '\n';        // "world"

    if (s.find("world") != std::string::npos) {           // npos: "not found"
        std::cout << "found it\n";
    }
    std::string t = s;                      // a real copy, not a shared pointer
    t[0] = 'J';
    std::cout << s << " / " << t << '\n';   // Hello, world / Jello, world
    std::cout << (s == "Hello, world") << '\n';            // 1: == compares contents
}
```

Everything you did by hand in C day 10 — `malloc`, `+ 1` for the terminator, `strcmp`, `strcpy`,
`free` — the `std::string` class does for you, safely. When the string goes out of scope its memory
is freed automatically (day 6 explains how).

### std::format (C++20): printf done right

```cpp
#include <format>
#include <iostream>
#include <string>

int main()
{
    int count = 3;
    double price = 9.5;
    std::string item = "coffee";
    std::cout << std::format("{} x {} at {:.2f} = {:>8.2f}\n", count, item, price, count * price);
    std::cout << std::format("{:<10}|{:^10}|{:>10}|\n", "left", "center", "right");
    std::cout << std::format("{:#x} {:08b} {:+}\n", 255, 5, 42);   // 0xff 00000101 +42
}
```

`{}` placeholders take any type — no `%d` vs `%ld` vs `%zu`. The type is checked at compile time.
Format specs after `:` work like `printf`'s: width, precision, alignment (`<` `^` `>`), `x` hex,
`b` binary. C++23 adds `std::print` and `std::println`, which print directly
(`std::println("{} items", n);`).

### auto and uniform initialization

```cpp
auto n = 42;              // int
auto pi = 3.14;           // double
auto name = std::string{"Ada"};   // std::string

int a{5};                 // brace initialization
int b{3.7};               // ERROR: narrowing conversion from double to int is refused
int c = 3.7;              // compiles, silently truncates to 3 (C behavior)
```

Braces `{}` catch narrowing conversions that C would silently accept. `auto` deduces the type from
the initializer; use it when the type is obvious or long, and spell the type out when it helps the
reader.

### Range-based for

```cpp
#include <iostream>
#include <string>
#include <vector>

int main()
{
    std::vector<int> nums = {3, 1, 4, 1, 5};   // a growable array — day 4
    for (int n : nums) {
        std::cout << n << ' ';
    }
    std::cout << '\n';

    for (char c : std::string{"hey"}) {
        std::cout << c << '-';
    }
    std::cout << '\n';                          // h-e-y-
}
```

`for (x : container)` walks every element — no index, no off-by-one.

### bool, nullptr, and other small differences

- `bool`, `true`, `false` are built in.
- Use `nullptr`, never `NULL` or `0`, for null pointers.
- C++ casts are explicit about what they do: `static_cast<int>(3.7)` instead of `(int)3.7`
  (day 2).
- `const` variables are real compile-time constants when initialized with one, and `constexpr`
  (day 2) guarantees it.
- C++ is stricter: `void *` doesn't convert implicitly to other pointer types, so
  `int *p = malloc(...)` doesn't compile — and you won't need `malloc` anyway.

## Common mistakes

- Compiling C++ with `gcc` instead of `g++` (link errors about `std::...`).
- `using namespace std;` in headers.
- Mixing `std::cin >> x` and `std::getline`: `>>` leaves the newline in the input, so a following
  `getline` reads an empty line. Call `std::cin.ignore()` in between, or use only `getline`.
- Forgetting `-std=c++20` (older defaults don't have `std::format`).

## Exercises

1. Type in the hello program. Then ask for the user's birth year too (`std::cin >> year`) and print
   their age with `std::format`. Put the year question *before* the name question and observe the
   `getline` problem; fix it.
2. Rewrite C day 2's temperature converter in C++ with `std::cin` and `std::format`.
3. Read a line and print: its length, the number of vowels, the line reversed (build a new string
   with a loop, or look up `std::string`'s constructor that takes two iterators), and whether it's a
   palindrome.
4. Read words until end of input (`while (std::cin >> word)`) and print the longest one and the
   total number of words.
5. Print a multiplication table 1–10 with `std::format` and a field width, like C day 4.
6. Put two functions named `area` in namespaces `circle` and `square` and call both.
7. Try `int x{2.5};` and `int y = 2.5;` and read the compiler's messages.
8. ★ Print the numbers 0–15 in decimal, hex and binary in aligned columns with `std::format`.

## Check yourself

1. What does `std::` mean, and why avoid `using namespace std;` in headers?
2. Name three things `std::string` does that C strings make you do by hand.
3. What does `{}` initialization prevent?
4. How is `std::format` safer than `printf`?
5. Why does `getline` sometimes read an empty line after `std::cin >> x`?

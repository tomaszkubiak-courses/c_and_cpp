# C++ Day 3 — Functions: overloading, defaults, parameter passing, lambdas

**Goal:** write C++ functions idiomatically: choose the right parameter type, overload, use
default arguments, and write your first lambdas.

## Concepts

### Overloading

C++ lets several functions share a name if their parameter types differ. The compiler picks the
best match:

```cpp
#include <iostream>
#include <string>

void print(int x) { std::cout << "int " << x << '\n'; }
void print(double x) { std::cout << "double " << x << '\n'; }
void print(const std::string &s) { std::cout << "string " << s << '\n'; }

int main()
{
    print(42);                    // int 42
    print(3.14);                  // double 3.14
    print(std::string{"hi"});     // string hi
}
```

The return type alone doesn't distinguish overloads. To make overloading possible, C++ encodes the
parameter types into each function's symbol name (**name mangling**) — which is why calling C code
from C++ needs special care (C++ day 21).

### Default arguments

```cpp
#include <iostream>
#include <string>

std::string repeat(const std::string &s, int times = 2, const std::string &sep = " ")
{
    std::string result;
    for (int i = 0; i < times; ++i) {
        if (i > 0) result += sep;
        result += s;
    }
    return result;
}

int main()
{
    std::cout << repeat("ha") << '\n';           // ha ha
    std::cout << repeat("ha", 3) << '\n';        // ha ha ha
    std::cout << repeat("ha", 3, "-") << '\n';   // ha-ha-ha
}
```

Defaults go at the end of the parameter list, and in the declaration (the header), not repeated in
the definition.

### How to pass parameters: the guideline

From the C++ Core Guidelines, simplified:

| You want to... | Parameter type | Example |
|---|---|---|
| read a cheap value (int, double, pointer, small struct) | `T` | `int square(int x)` |
| read an expensive object (string, vector, big struct) | `const T &` | `void print(const std::string &s)` |
| modify the caller's object | `T &` | `void normalize(std::string &s)` |
| express "optional / may be absent" | `T *` (can be `nullptr`) or `std::optional` (day 13) | `void draw(const Style *style)` |
| take ownership / keep a copy | `T` (by value, then `std::move` — day 7) | `void set_name(std::string name)` |

And for results: **return by value**. `std::string make_greeting()` doesn't copy the string on
return — the compiler constructs it directly in the caller (*copy elision*, guaranteed in many
cases since C++17). Out-parameters (C's `int *out`) are rarely needed; return a struct, a
`std::pair`, or a `std::optional` instead.

### [[nodiscard]]

```cpp
[[nodiscard]] bool save(const std::string &path);
```

The compiler warns if the caller ignores the result — perfect for error returns.

### inline and headers

Functions defined in a header that's included in several `.cpp` files must be marked `inline`
(or be templates, `constexpr`, or class members defined in the class), otherwise the linker sees
multiple definitions — the same rule as C day 17. `inline` in C++ means "may be defined in several
translation units", not "please inline this call" (the optimizer decides that on its own).

### Lambdas: functions you write inline

A **lambda** is an unnamed function object you can define right where you need it:

```cpp
#include <algorithm>
#include <iostream>
#include <vector>

int main()
{
    std::vector<int> v = {5, -3, 8, -1, 2};

    auto is_negative = [](int x) { return x < 0; };   // [captures](params) { body }
    std::cout << std::count_if(v.begin(), v.end(), is_negative) << '\n';   // 2

    int limit = 4;
    auto above = [limit](int x) { return x > limit; };   // captures limit by value
    std::cout << std::count_if(v.begin(), v.end(), above) << '\n';         // 2

    std::sort(v.begin(), v.end(), [](int a, int b) { return a > b; });     // descending
    for (int x : v) std::cout << x << ' ';
    std::cout << '\n';   // 8 5 2 -1 -3
}
```

Compare with C day 13: instead of writing a separate named comparator with `const void *`
parameters, you write the comparison right at the call — and it's type-safe. The `[limit]` part is
the **capture list**: which surrounding variables the lambda can use. Day 18 covers captures in
detail; for now, `[x]` copies `x`, `[&x]` refers to it.

### Recursion and the call stack

Same as C: each call gets a stack frame. Nothing new — except that local objects with destructors
(like `std::string`) are cleaned up automatically when the frame is popped.

### Organizing code: header and source

`math_utils.h`:

```cpp
#pragma once

namespace mu {
    [[nodiscard]] int gcd(int a, int b);
    [[nodiscard]] inline int square(int x) { return x * x; }   // defined in header: inline
}
```

`math_utils.cpp`:

```cpp
#include "math_utils.h"

namespace mu {
    int gcd(int a, int b) { return b == 0 ? a : gcd(b, a % b); }
}
```

Same structure as C day 17, with `#pragma once` (universally supported) and a namespace per
module.

## Common mistakes

- Overloads that are ambiguous: `f(long)` and `f(double)` called with an `int`.
- Passing `std::string` or `std::vector` by value when you only read them.
- Default arguments repeated in the definition (compile error).
- Non-`inline` function definitions in headers (linker error: multiple definition).
- Returning a reference or pointer to a local.

## Exercises

1. Write overloads `double area(double radius)` and `double area(double w, double h)`. Can you add
   `int area(double radius)`? Why not?
2. Write `std::string pad_left(const std::string &s, std::size_t width, char fill = ' ')`.
3. Write `std::vector<int> primes_up_to(int n)` that **returns** a vector, and print the result
   with a range-based `for`. Why is returning a vector by value fine?
4. Write `void normalize_spaces(std::string &s)` that collapses runs of spaces into one space.
5. Split a program into `stats.h` / `stats.cpp` / `main.cpp` with `double mean(const std::vector<double> &)`
   and `double median(std::vector<double> v)` — why might `median` take its parameter **by value**?
   Compile with `g++ -std=c++20 main.cpp stats.cpp -o stats`.
6. Using lambdas and `std::count_if`, count the words in a vector of strings that are longer than
   a length read from the user.
7. Sort a `std::vector<std::string>` by length using `std::sort` and a lambda, then by
   length descending and alphabetically for ties.
8. ★ Mark a function `[[nodiscard]]`, call it without using the result, and read the warning.

## Check yourself

1. How does the compiler choose between overloads?
2. What parameter type would you use to read a `std::vector<int>`? To modify it? To keep a copy?
3. Why is returning big objects by value usually fine in C++?
4. What does `inline` mean for a function defined in a header?
5. What's in a lambda's `[]`?

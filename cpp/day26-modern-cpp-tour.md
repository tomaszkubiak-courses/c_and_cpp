# C++ Day 26 — A tour of modern C++: C++20 features, and a look at C++23 and C++26

**Goal:** know the rest of what C++20 offers, and what the newer standards add, so you can read
modern code and pick the right tool.

## Concepts

### C++20 features you've already used

Concepts (day 16), ranges (day 19), `std::format` (day 1), `std::span` (day 4), `operator<=>`
(day 10), `contains` on maps (day 17), `std::erase_if` (day 9). Here's the rest of the useful part.

### Small but handy

```cpp
#include <algorithm>
#include <bit>
#include <cmath>
#include <iostream>
#include <numbers>
#include <numeric>
#include <source_location>
#include <string>
#include <vector>

struct Config {
    std::string host = "localhost";
    int port = 80;
    bool verbose = false;
};

void log(const std::string &msg, std::source_location loc = std::source_location::current())
{
    std::cout << loc.file_name() << ':' << loc.line() << " [" << loc.function_name() << "] "
              << msg << '\n';
}

consteval int square(int x) { return x * x; }       // must run at compile time

int main()
{
    Config c{.port = 8080, .verbose = true};       // designated initializers (in declaration order)
    std::cout << c.host << ':' << c.port << '\n';

    std::cout << std::numbers::pi << ' ' << std::numbers::e << '\n';   // math constants
    std::cout << std::popcount(255u) << ' ' << std::bit_width(1000u) << ' '
              << std::has_single_bit(64u) << '\n';                    // <bit>: 8 10 1

    std::string s = "main.cpp";
    std::cout << s.starts_with("main") << s.ends_with(".cpp") << '\n'; // 11

    std::vector<int> v = {1, 2, 3};
    for (auto i = std::ssize(v) - 1; i >= 0; --i) std::cout << v[i];  // signed size: no unsigned wrap
    std::cout << '\n';

    std::cout << std::midpoint(1, 9) << ' ' << std::lerp(0.0, 10.0, 0.25) << '\n';   // 5 2.5

    constexpr int nine = square(3);
    static_assert(nine == 9);

    log("hello");                                  // prints file, line and function
}
```

Also in C++20:

- `constinit` — a variable must be initialized at compile time (avoids the "static initialization
  order fiasco"), but can change later.
- `using enum Color;` — bring enumerators into scope inside a `switch`.
- `[[likely]]`, `[[unlikely]]` — hints on branches.
- Template lambdas: `[]<typename T>(const std::vector<T> &v) { ... }`.
- `std::bit_cast<To>(from)` — reinterpret bytes safely (the correct replacement for the
  strict-aliasing-violating pointer cast from C day 28).
- `<chrono>` calendars and time zones: `std::chrono::year_month_day`, `std::chrono::weekday`,
  `std::chrono::zoned_time` — date arithmetic without third-party libraries.
- `std::jthread`, `std::stop_token`, `std::latch`, `std::barrier`, `std::counting_semaphore` —
  tomorrow.

### Modules

Modules replace `#include` for your own code: a module is compiled once, exports only what it
declares `export`, and isn't affected by macros from the files that import it.

```cpp
// math.cppm
export module math;
export int add(int a, int b) { return a + b; }

// main.cpp
import math;
import std;          // C++23: the whole standard library as a module
int main() { std::println("{}", add(2, 3)); }
```

The benefits are real (faster builds, no include-order problems), but tool support is still
maturing: you need CMake 3.28+, a recent compiler, and the Ninja generator, and `import std;`
needs extra setup. For now, know what modules are and read about them; headers remain the norm in
most codebases.

### Coroutines

Functions that can **suspend** and **resume** (`co_await`, `co_yield`, `co_return`), for
generators and asynchronous code. C++20 provides only the low-level machinery — you need a library
type to use them comfortably. C++23 adds `std::generator`:

```cpp
#include <generator>   // C++23
#include <iostream>
#include <ranges>
#include <utility>

std::generator<int> fibonacci()
{
    int a = 0, b = 1;
    while (true) {
        co_yield a;                   // suspend and hand out a value
        a = std::exchange(b, a + b);
    }
}

int main()
{
    for (int x : fibonacci() | std::views::take(10)) std::cout << x << ' ';
}
```

### C++23 highlights

- `std::print` / `std::println` — `std::format` straight to the console.
- `std::expected<T, E>` — day 13.
- `std::ranges::to<Container>()`, `views::zip`, `views::enumerate`, `views::chunk` — day 19.
- `std::generator` — above.
- **Deducing `this`** — explicit object parameters: `void f(this Self &&self)`, which removes a lot
  of duplicated `const`/non-`const` overloads.
- `std::flat_map` / `std::flat_set` — sorted-vector-based maps, faster to iterate than `std::map`.
- `std::mdspan` — multidimensional views (a `span` for matrices).
- `import std;` — the standard library as a module.
- `std::stacktrace`, `std::unreachable()`, `if consteval`, `auto(x)` decay-copy, the `z` suffix for
  `size_t` literals (`10uz`).

### C++26

C++26 is the next standard; its feature set was fixed in 2025. Its biggest additions:

- **Static reflection** — programs can inspect types at compile time (members of a struct, names of
  enumerators), which will replace much hand-written boilerplate like serialization and
  enum-to-string.
- **Contracts** — preconditions and postconditions on functions (`pre(x > 0)`), checked according
  to a build setting.
- **`std::execution`** (senders/receivers) — a standard framework for asynchronous and parallel
  work.
- Plus `std::inplace_vector`, `#embed` (from C23), linear algebra, and more.

Compiler support arrives over the following years. Check <https://en.cppreference.com/w/cpp/compiler_support>
to see what your compiler supports.

## Exercises

1. Type in the tour program. Change the designated initializer order (`.verbose` before `.port`)
   and read the error. Why does C++ require declaration order when C doesn't?
2. Use `std::source_location` to write a `CHECK(condition)` function that prints where a failed
   check was called from — without macros.
3. Use `std::chrono` calendar types: print the day of the week you were born, and the number of days
   until your next birthday.
4. Replace the strict-aliasing float-to-bits cast from C day 28 with `std::bit_cast<std::uint32_t>(1.0f)`
   and print the bits.
5. With `-std=c++23`, rewrite a few earlier exercises using `std::println`, `std::ranges::to` and
   `views::enumerate`.
6. ★ With `-std=c++23`, write `std::generator<int> primes()` and print the first 20 primes.
7. ★ Try modules: follow the CMake documentation for "C++20 modules" (CMake 3.28+, Ninja) to build
   a two-module project.

## Check yourself

1. What does `consteval` guarantee, compared to `constexpr`?
2. What does `std::source_location::current()` do as a default argument?
3. What problems do modules solve, and why aren't they everywhere yet?
4. Name three C++23 additions.
5. What is static reflection, and what boilerplate will it remove?

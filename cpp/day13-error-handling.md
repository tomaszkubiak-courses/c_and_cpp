# C++ Day 13 — Error handling: exceptions, optional, expected

**Goal:** throw and catch exceptions correctly, write exception-safe code with RAII, and choose
between exceptions, `std::optional` and `std::expected`.

## Concepts

### Exceptions

```cpp
#include <iostream>
#include <stdexcept>
#include <string>
#include <vector>

double average(const std::vector<double> &v)
{
    if (v.empty()) {
        throw std::invalid_argument{"average of an empty vector"};
    }
    double sum = 0;
    for (double x : v) sum += x;
    return sum / static_cast<double>(v.size());
}

int main()
{
    try {
        std::vector<double> data;
        std::cout << average(data) << '\n';       // throws; the next line never runs
        std::cout << "not printed\n";
    } catch (const std::invalid_argument &e) {    // catch by const reference
        std::cerr << "invalid argument: " << e.what() << '\n';
    } catch (const std::exception &e) {           // any standard exception
        std::cerr << "error: " << e.what() << '\n';
    }
}
```

When an exception is thrown, the program leaves the current function and every caller up the
stack until a matching `catch` is found (**stack unwinding**). On the way, **every local object's
destructor runs** — this is why RAII (day 6) matters: files close, memory is freed, locks are
released, automatically, on the error path too. If nothing catches it, the program terminates.

### The standard exception hierarchy

```text
std::exception
├── std::logic_error          (bugs: the caller broke a precondition)
│   ├── std::invalid_argument
│   ├── std::out_of_range     (vector::at, string::substr)
│   └── std::length_error
├── std::runtime_error        (things that go wrong at run time)
│   ├── std::range_error, std::overflow_error
│   └── std::system_error     (OS errors, with an error code)
└── std::bad_alloc            (new failed)
```

Throw one of these (or your own class derived from them), always **catch by `const &`**, and order
`catch` blocks from most to least specific.

```cpp
class ParseError : public std::runtime_error {
public:
    ParseError(const std::string &msg, int line)
        : std::runtime_error{msg + " (line " + std::to_string(line) + ")"}, line_{line} {}
    int line() const { return line_; }
private:
    int line_;
};
```

`throw;` inside a `catch` rethrows the current exception unchanged.

### Exception safety guarantees

What state is an object in after an operation throws? Three levels:

1. **Basic guarantee** — no leaks, and objects are still valid (but may have changed).
2. **Strong guarantee** — the operation either fully succeeds or has no effect (commit or
   rollback). Copy-and-swap (day 6) gives this: do everything that can throw on a copy, then swap
   with no-throw operations.
3. **No-throw guarantee** — it never throws. Mark such functions `noexcept`: destructors (implicitly
   `noexcept`), move operations, swap.

RAII gives you the basic guarantee almost for free. Never let an exception escape a destructor.

### When to use exceptions

Use them for errors that the **immediate caller usually can't handle**, and that are exceptional:
a file that should exist doesn't, a network connection drops, a config file is malformed, a
precondition is violated. Don't use them for normal control flow ("not found" in a search is not an
exception). Exceptions are cheap when not thrown, and expensive when thrown.

Some codebases (games, embedded) disable exceptions (`-fno-exceptions`); then you need the value-based
tools below.

### std::optional: "a value or nothing"

```cpp
#include <iostream>
#include <optional>
#include <string>
#include <vector>

struct User { int id; std::string name; };

std::optional<User> find_user(const std::vector<User> &users, int id)
{
    for (const auto &u : users) {
        if (u.id == id) return u;
    }
    return std::nullopt;                       // "nothing"
}

int main()
{
    std::vector<User> users = {{1, "Ada"}, {2, "Alan"}};

    if (auto u = find_user(users, 2)) {        // contextually converts to bool
        std::cout << u->name << '\n';           // Alan: -> and * access the value
    }
    auto missing = find_user(users, 9);
    std::cout << missing.has_value() << '\n';  // 0
    std::cout << find_user(users, 9).value_or(User{0, "guest"}).name << '\n';   // guest
}
```

Replaces C's "return a pointer, NULL means not found" and "return -1 on failure" — without
pointers or magic values. Use it when absence is a normal outcome and there's nothing more to say
about *why*.

### std::expected (C++23): "a value or an error"

When the caller needs to know *why* it failed, and failure is expected (parsing user input,
validating data), `std::expected<T, E>` holds either a value or an error:

```cpp
#include <charconv>
#include <expected>
#include <iostream>
#include <string>
#include <string_view>

enum class ParseErr { empty, not_a_number, out_of_range };

std::expected<int, ParseErr> parse_int(std::string_view s)
{
    if (s.empty()) return std::unexpected{ParseErr::empty};
    int value{};
    auto [ptr, ec] = std::from_chars(s.data(), s.data() + s.size(), value);
    if (ec == std::errc::result_out_of_range) return std::unexpected{ParseErr::out_of_range};
    if (ec != std::errc{} || ptr != s.data() + s.size()) return std::unexpected{ParseErr::not_a_number};
    return value;
}

int main()
{
    for (std::string_view s : {"42", "", "12abc", "99999999999"}) {
        auto r = parse_int(s);
        if (r) std::cout << s << " -> " << *r << '\n';
        else   std::cout << '"' << s << "\" -> error " << static_cast<int>(r.error()) << '\n';
    }
}
```

`std::expected` is C++23 (compile with `-std=c++23`; GCC 12+, Clang 16+, MSVC 17.3+). It's C's
"return a status + write the value through a pointer" (C day 20) as one type. `std::from_chars`
(C++17, `<charconv>`) is the fast, non-throwing, locale-independent parser — C++'s `strtol`.

### Choosing

| Situation | Tool |
|---|---|
| Programmer error, "can't happen" | `assert` (or a `logic_error` exception in library code) |
| Rare failure the caller can't fix locally (I/O, resource exhaustion, corrupt data) | exception |
| Absence is a normal answer | `std::optional<T>` |
| Failure is normal and the reason matters | `std::expected<T, E>` (C++23), or an error code |
| Compile-time checkable | `static_assert` |

## Common mistakes

- Catching by value (`catch (std::exception e)`) — slices the exception and copies it.
- `catch (...) {}` that silently swallows everything.
- Throwing from destructors.
- Using exceptions for normal control flow.
- Calling `.value()` or `*` on an empty `optional` without checking (`.value()` throws; `*` is
  undefined behavior).
- Resources without RAII, leaked when an exception passes through.

## Exercises

1. Type in the `average` example and add a `catch` for `std::out_of_range` triggered by
   `std::vector::at`. Which `catch` block runs, and why does the order of the blocks matter?
2. Write `class ParseError` above and a function that parses `"key=value"` lines, throwing
   `ParseError` with the line number. Catch it in `main` and print `e.what()` and `e.line()`.
3. Put a `Noisy` object (day 6) in each of three nested functions, throw from the innermost, and
   catch in `main`. Watch the destructors run during unwinding.
4. Write `std::optional<std::size_t> index_of(std::span<const int> v, int x)`.
5. Write `parse_int` with `std::expected` (compile with `-std=c++23`), and a `parse_point`
   (`"3,4"` → `{3, 4}`) that uses it and propagates errors. Look up the monadic operations
   `and_then` and `transform` and use them.
6. Make your day 7 `Matrix::multiply` throw `std::invalid_argument` on a size mismatch, and give the
   strong guarantee for a `resize` member (build the new data first, then swap).
7. ★ Measure: time a loop that throws and catches 10⁶ exceptions, vs one that returns 10⁶ error
   codes. Conclude when exceptions are appropriate.

## Check yourself

1. What happens to local objects when an exception propagates?
2. Why catch by `const &`?
3. What are the basic, strong and no-throw guarantees?
4. When would you return `std::optional` rather than throw?
5. What does `std::expected` add over `std::optional`?

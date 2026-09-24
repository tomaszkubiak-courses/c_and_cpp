# C++ Day 22 — Testing, debugging and code quality tools

**Goal:** write unit tests with a real framework (Catch2), run them with CTest, and make sanitizers,
the debugger, clang-tidy and clang-format part of your routine.

## Concepts

### Why a test framework

`assert`-based tests (C day 25, C++ day 20) stop at the first failure, say little about what went
wrong, and vanish in Release builds. A framework gives you named test cases, clear failure
messages showing both values, running a subset of tests, and integration with CMake and IDEs.

The two most popular: **Catch2** (simple, header-style syntax) and **GoogleTest** (widely used in
industry). They're similar; this lesson uses Catch2 v3.

### Adding Catch2 with FetchContent

`tests/CMakeLists.txt` (in the day 20 project layout):

```cmake
include(FetchContent)
FetchContent_Declare(
    Catch2
    GIT_REPOSITORY https://github.com/catchorg/Catch2.git
    GIT_TAG v3.8.1
)
FetchContent_MakeAvailable(Catch2)

add_executable(unit_tests test_fraction.cpp)
target_link_libraries(unit_tests PRIVATE fraction Catch2::Catch2WithMain)   # Catch2 provides main()

include(Catch)                       # a Catch2 CMake module (found automatically after MakeAvailable)
catch_discover_tests(unit_tests)     # registers each TEST_CASE with CTest individually
```

### Writing tests

Say you have a `Fraction` class (days 5 and 10) in a library target `fraction`:

`include/fraction.h`:

```cpp
#pragma once

#include <compare>
#include <stdexcept>

class Fraction {
public:
    Fraction(long num, long den = 1);
    long num() const { return num_; }
    long den() const { return den_; }
    Fraction operator+(const Fraction &o) const;
    bool operator==(const Fraction &) const = default;
private:
    long num_, den_;
};
```

`src/fraction.cpp`:

```cpp
#include "fraction.h"

#include <numeric>

Fraction::Fraction(long num, long den)
{
    if (den == 0) throw std::invalid_argument{"zero denominator"};
    if (den < 0) { num = -num; den = -den; }
    long g = std::gcd(num, den);
    num_ = num / g;
    den_ = den / g;
}

Fraction Fraction::operator+(const Fraction &o) const
{
    return Fraction{num_ * o.den_ + o.num_ * den_, den_ * o.den_};
}
```

`tests/test_fraction.cpp`:

```cpp
#include <catch2/catch_test_macros.hpp>

#include "fraction.h"

TEST_CASE("fractions are normalized", "[fraction]")
{
    Fraction f{6, -8};
    CHECK(f.num() == -3);          // CHECK: record failure, continue
    CHECK(f.den() == 4);
}

TEST_CASE("zero denominator is rejected", "[fraction]")
{
    REQUIRE_THROWS_AS(Fraction(1, 0), std::invalid_argument);   // REQUIRE: stop this test on failure
}

TEST_CASE("addition", "[fraction][math]")
{
    SECTION("simple") {            // each SECTION runs separately, with fresh setup above it
        CHECK(Fraction{1, 2} + Fraction{1, 3} == Fraction{5, 6});
    }
    SECTION("result is reduced") {
        CHECK(Fraction{1, 4} + Fraction{1, 4} == Fraction{1, 2});
    }
    SECTION("negative") {
        CHECK(Fraction{1, 2} + Fraction{-1, 2} == Fraction{0});
    }
}
```

Build and run:

```bash
cmake -S . -B build && cmake --build build
./build/tests/unit_tests                 # all tests
./build/tests/unit_tests "[math]"        # only tests tagged [math]
ctest --test-dir build --output-on-failure
```

A failing `CHECK` prints the expression **and the values**:

```text
test_fraction.cpp:8: FAILED:
  CHECK( f.num() == -3 )
with expansion:
  3 == -3
```

`CHECK_THAT` with matchers, `Approx`/`WithinRel` for floating point
(`#include <catch2/matchers/catch_matchers_floating_point.hpp>`), `GENERATE` for data-driven tests,
and benchmarks are all worth looking up in the Catch2 docs.

### What to test

- Each function's normal behavior, **edge cases** (empty, one element, maximum values, negative),
  and **error cases** (invalid input throws or returns an error).
- One behavior per test case, with a name that says what should happen.
- Tests must be deterministic: no dependence on time, random seeds (fix the seed) or test order.
- Test through the public interface; tests that know about private details break when you
  refactor.

**Test-driven development (TDD)** is worth trying: write a failing test first, make it pass with the
simplest code, then clean up. It forces you to design the interface from the caller's side.

### Sanitizers in CMake

Add an option once, use it for every test run:

```cmake
option(ENABLE_SANITIZERS "Build with ASan and UBSan" OFF)
if(ENABLE_SANITIZERS AND NOT MSVC)
    add_compile_options(-fsanitize=address,undefined -fno-omit-frame-pointer)
    add_link_options(-fsanitize=address,undefined)
endif()
```

(Setting them globally in the top-level file is a reasonable exception to "targets, not
variables" — sanitizers must be applied to everything linked together.) For data races in
multithreaded code (day 27), use `-fsanitize=thread` instead (it can't be combined with `address`).

### Debugging

- **gdb** (setup guide) works for C++ too. Useful extras: `catch throw` stops when any exception is
  thrown; `info locals`; `finish` runs until the current function returns. With libstdc++'s pretty
  printers, `print v` shows a `std::vector`'s elements.
- **In the IDE:** VS Code with CMake Tools: pick the target, set breakpoints, press the debug
  button. Visual Studio: same, with a superb watch window.
- **Debug builds of the standard library:** `-D_GLIBCXX_ASSERTIONS` makes `operator[]` on
  containers bounds-checked (cheap; good for all debug builds). `-D_GLIBCXX_DEBUG` checks iterator
  validity too (slower; changes the ABI, so everything must be compiled with it).

A debugging method that works: **reproduce** reliably (write a failing test), **isolate** (shrink
the input, bisect the code), **understand** the cause before changing anything, **fix**, and keep
the test so it never comes back.

### Static analysis and formatting

- **clang-tidy** checks for bug patterns and suggests modernizations. With
  `CMAKE_EXPORT_COMPILE_COMMANDS ON` (day 20): `clang-tidy -p build src/*.cpp`. Configure checks in
  a `.clang-tidy` file, e.g. `Checks: 'bugprone-*,modernize-*,performance-*,readability-*'`.
- **clang-format** formats code consistently: a `.clang-format` file (`BasedOnStyle: LLVM`) and
  `clang-format -i src/*.cpp include/*.h`. Many editors format on save.
- **Compiler warnings** remain the cheapest tool of all: `-Wall -Wextra -Wpedantic -Wshadow
  -Wconversion`, and `-Werror` in CI.

## Common mistakes

- Tests that only check the happy path.
- Tests that depend on each other or on order.
- `REQUIRE` where `CHECK` would show more failures at once (or the reverse: continuing after a
  failure that makes later checks meaningless).
- Comparing floating point with `==` in tests.
- Never running tests under sanitizers.

## Exercises

1. Set up the `Fraction` project with the day 20 layout and Catch2. Get all tests passing.
2. Add tests for `operator-`, `operator*`, `operator/` (and division by a zero fraction), and
   `operator<=>` — write the tests **first** (TDD), watch them fail, then implement.
3. Write Catch2 tests for your day 19 exercises (ranges) or day 18 algorithms functions — make each
   function testable by giving it inputs and outputs instead of reading `std::cin`.
4. Use `GENERATE` to test `is_prime` against a table of known primes and non-primes.
5. Introduce a bug (an off-by-one in a loop), and find it with gdb or the IDE debugger: set a
   breakpoint, step, watch variables. Then write a test that catches it.
6. Build with `ENABLE_SANITIZERS=ON` and run the tests. Plant a use-after-free and check that the
   test run reports it.
7. Run clang-tidy on your day 14 library project. Fix or understand every warning.
8. ★ Set up the same tests with GoogleTest (`FetchContent` from `https://github.com/google/googletest.git`,
   link `GTest::gtest_main`, `include(GoogleTest)`, `gtest_discover_tests`) and compare.

## Check yourself

1. What's the difference between `CHECK` and `REQUIRE`?
2. What does `catch_discover_tests` do?
3. What makes a good unit test?
4. Which sanitizer finds data races, and why can't it be combined with ASan?
5. What does `-D_GLIBCXX_ASSERTIONS` do?

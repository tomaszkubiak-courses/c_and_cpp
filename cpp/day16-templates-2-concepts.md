# C++ Day 16 — Templates II: concepts, constraints, and variadic templates

**Goal:** state template requirements with C++20 concepts, write your own concepts, and use
`if constexpr`, variadic templates and fold expressions.

## Concepts

### The problem: implicit requirements

Yesterday's `max_of` needed `T` to have `<`, but nothing said so. Pass a type without `<` and the
error appears inside the template body, often dozens of lines long. **Concepts** (C++20) name the
requirements and check them at the call site.

### Using standard concepts

The `<concepts>` header defines many: `std::integral`, `std::floating_point`,
`std::totally_ordered`, `std::equality_comparable`, `std::copyable`, `std::movable`,
`std::invocable<F, Args...>`, `std::predicate<F, Args...>`, `std::convertible_to<From, To>`,
`std::same_as<A, B>`...

Three equivalent ways to constrain a template:

```cpp
#include <concepts>
#include <iostream>
#include <string>

// 1. A concept instead of typename
template <std::totally_ordered T>
T max_of(const T &a, const T &b) { return (a < b) ? b : a; }

// 2. A requires-clause
template <typename T>
    requires std::integral<T>
T gcd(T a, T b) { return b == 0 ? a : gcd(b, a % b); }

// 3. Abbreviated function template: "auto" parameter with a concept
void print_twice(const std::floating_point auto &x) { std::cout << x * 2 << '\n'; }

struct NoCompare {};

int main()
{
    std::cout << max_of(3, 7) << ' ' << max_of(std::string{"b"}, std::string{"a"}) << '\n';   // 7 b
    std::cout << gcd(12, 18) << '\n';                                                         // 6
    print_twice(1.25);                                                                        // 2.5
    // max_of(NoCompare{}, NoCompare{});   // error: constraints not satisfied — a short, clear message
    // gcd(1.5, 2.5);                      // error: double does not satisfy std::integral
    // print_twice(3);                     // error: int does not satisfy std::floating_point
}
```

Try uncommenting the error lines one at a time and compare the messages with yesterday's
unconstrained `max_of`.

### Writing your own concepts

A concept is a named compile-time predicate on types. A `requires` expression lists what must
compile:

```cpp
#include <concepts>
#include <iostream>
#include <string>
#include <vector>

template <typename T>
concept Shape = requires(const T &s) {
    { s.area() } -> std::convertible_to<double>;    // must have area() returning something double-like
    { s.name() } -> std::convertible_to<std::string>;
};

template <typename T>
concept Container = requires(T c) {
    c.begin();
    c.end();
    c.size();
};

struct Square {
    double side;
    double area() const { return side * side; }
    std::string name() const { return "square"; }
};

void report(const Shape auto &s)
{
    std::cout << s.name() << " with area " << s.area() << '\n';
}

double total_area(const Container auto &shapes)
{
    double sum = 0;
    for (const auto &s : shapes) sum += s.area();
    return sum;
}

int main()
{
    Square sq{3};
    report(sq);                                           // square with area 9
    std::vector<Square> many{{1}, {2}};
    std::cout << total_area(many) << '\n';                // 5
    static_assert(Shape<Square>);
    static_assert(!Shape<int>);
}
```

This is **static polymorphism**: `report` works with any type that has `area()` and `name()`,
without a base class or virtual functions — resolved at compile time, with no run-time cost. Compare
with day 11's `virtual` version (dynamic polymorphism): concepts need the type known at compile
time, so you can't put different shapes in one vector without `std::variant` or a base class.

### Overloading on concepts

The compiler picks the most constrained overload that matches:

```cpp
#include <concepts>
#include <iostream>

void describe(const std::integral auto &x) { std::cout << x << " is an integer\n"; }
void describe(const std::floating_point auto &x) { std::cout << x << " is floating point\n"; }
void describe(const auto &) { std::cout << "something else\n"; }

int main()
{
    describe(42);        // is an integer
    describe(2.5);       // is floating point
    describe("text");    // something else
}
```

The overloads must have the same parameter form (here all `const ... &`) for "most constrained"
to decide. Mix `std::integral auto x` (by value) with `const auto &` and the call is simply
ambiguous — try it and read the error.

### if constexpr: compile-time branches

Inside a template, `if constexpr` discards the branch that doesn't apply, so it doesn't even need
to compile for that type:

```cpp
#include <iostream>
#include <string>
#include <type_traits>

template <typename T>
std::string to_text(const T &value)
{
    if constexpr (std::is_same_v<T, std::string>) {
        return '"' + value + '"';
    } else if constexpr (std::is_arithmetic_v<T>) {
        return std::to_string(value);          // wouldn't compile for std::string
    } else {
        return "<object>";
    }
}

int main()
{
    std::cout << to_text(42) << ' ' << to_text(std::string{"hi"}) << ' ' << to_text(3.5) << '\n';
}
```

`<type_traits>` provides compile-time questions about types: `std::is_integral_v<T>`,
`std::is_pointer_v<T>`, `std::remove_reference_t<T>` and many more. Concepts are mostly built on
them.

### Variadic templates and fold expressions

A **parameter pack** accepts any number of arguments of any types:

```cpp
#include <iostream>

template <typename... Args>
auto sum_all(const Args &...args)
{
    return (args + ...);                         // fold expression: a1 + (a2 + (a3 + ...))
}

template <typename... Args>
void print_line(const Args &...args)
{
    ((std::cout << args << ' '), ...);           // comma fold: print each
    std::cout << '\n';
}

template <typename... Args>
constexpr std::size_t count_args(const Args &...)
{
    return sizeof...(Args);                      // number of elements in the pack
}

int main()
{
    std::cout << sum_all(1, 2, 3, 4.5) << '\n';  // 10.5
    print_line("x =", 42, "and y =", 3.14);      // x = 42 and y = 3.14
    std::cout << count_args(1, "a", 2.0) << '\n';   // 3
}
```

`std::make_unique<T>(args...)`, `emplace_back(args...)` and `std::format` are all variadic
templates. They pass the arguments on with **perfect forwarding**: `std::forward<Args>(args)...`
preserves whether each argument was an lvalue or rvalue, so moves stay moves.

Now you can read day 12's helper:

```cpp
template <class... Fs> struct overloaded : Fs... { using Fs::operator()...; };
```

"A struct that inherits from every lambda type passed in, and brings all their `operator()`s into
scope."

## Common mistakes

- Constraining too tightly (requiring `std::integral` when any number would do).
- Concepts that check syntax but not meaning (a type with `area()` returning something unrelated).
  Concepts check that code compiles, not that it's correct.
- Forgetting `constexpr` in `if constexpr` — then both branches must compile.
- Deep template metaprogramming when a plain function would do.

## Exercises

1. Go back to yesterday's exercise 7 and constrain `max_of` with `std::totally_ordered`. Compare
   the error messages.
2. Write `template <std::integral T> bool is_prime(T n)` and check it rejects `double` at compile
   time.
3. Write a concept `Printable` that requires `std::cout << x` to compile, and
   `template <Printable T> void print_all(const std::vector<T> &v)`. Check that a vector of a
   struct without `operator<<` is rejected with a readable message.
4. Write a concept `Stack` requiring `push`, `pop`, `empty`, and a function `drain` that pops and
   prints every element of any `Stack`. Test it with your `Stack<T>` from yesterday.
5. Overload `describe` for `std::integral`, `std::floating_point`, and a concept `StringLike`
   (convertible to `std::string_view`).
6. Write `template <typename... Ts> bool all_positive(Ts... xs)` with a fold over `&&`.
7. Write `to_text` with `if constexpr`, adding a branch for `std::vector<T>` of anything
   (hint: a concept `requires { typename T::value_type; }` or a trait).
8. ★ Write `template <typename F, typename... Args> auto timed(F &&f, Args &&...args)` that calls
   `f(std::forward<Args>(args)...)`, prints how long it took, and returns the result.

## Check yourself

1. What problem do concepts solve?
2. Name three ways to constrain a template with a concept.
3. What's the difference between static and dynamic polymorphism?
4. What does `if constexpr` do that a normal `if` can't?
5. What does `(args + ...)` expand to?

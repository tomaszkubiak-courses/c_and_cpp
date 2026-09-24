# C++ Day 2 — References, const, constexpr and the type system

**Goal:** use references instead of pointers where possible, know when each is appropriate, and
use `const`, `constexpr`, `enum class`, structured bindings and C++ casts.

## Concepts

### References: another name for an existing object

```cpp
#include <iostream>

int main()
{
    int x = 10;
    int &r = x;        // r is a reference to x: another name for the same int
    r = 20;            // changes x
    std::cout << x << '\n';   // 20
    std::cout << (&r == &x) << '\n';   // 1: same address, same object
}
```

A reference is like a pointer that is **automatically dereferenced**, **can't be null**, and
**can't be re-seated** (it refers to the same object for its whole life). It must be initialized.

### References as function parameters

The C `swap` with pointers becomes:

```cpp
#include <iostream>

void swap_values(int &a, int &b)    // a and b ARE the caller's variables
{
    int tmp = a;
    a = b;
    b = tmp;
}

int main()
{
    int x = 1, y = 2;
    swap_values(x, y);              // no & at the call site
    std::cout << x << ' ' << y << '\n';   // 2 1
}
```

(The standard library already has `std::swap` in `<utility>`.)

### Pointers vs references

| | Pointer `T *p` | Reference `T &r` |
|---|---|---|
| Can be null | yes (`nullptr`) | no |
| Can change what it refers to | yes | no — bound once |
| Syntax to use the target | `*p`, `p->m` | just `r`, `r.m` |
| Must be initialized | no (but should be) | yes |
| Arithmetic | yes | no |
| Typical use | optional things, arrays, ownership (with smart pointers) | parameters, return of existing objects, aliases |

Rule of thumb: **use a reference unless you need "nothing" (null) or need to re-point** — then use a
pointer.

### const references: the default way to pass big objects

Passing a `std::string` by value copies it. Passing by `const &` avoids the copy and promises not
to modify it:

```cpp
#include <iostream>
#include <string>

std::size_t count_vowels(const std::string &s)   // no copy, read-only
{
    std::size_t n = 0;
    for (char c : s) {
        if (std::string{"aeiouAEIOU"}.find(c) != std::string::npos) ++n;
    }
    return n;
}

int main()
{
    std::string text = "Programming in C++";
    std::cout << count_vowels(text) << '\n';   // 4
    std::cout << count_vowels("literal") << '\n';   // 3: const & can bind to a temporary
}
```

A `const &` can bind to a temporary (like the `std::string` made from `"literal"`); a plain `&`
can't. The full rules for choosing parameter types come tomorrow.

### const everywhere

```cpp
const int max_users = 100;         // can't change
const std::string greeting = "hi"; // can't call modifying functions on it
```

Make things `const` unless they need to change. It documents intent and lets the compiler catch
mistakes. Day 5 adds `const` member functions.

### constexpr: computed at compile time

```cpp
#include <array>
#include <iostream>

constexpr int square(int x) { return x * x; }

int main()
{
    constexpr int size = square(4);          // evaluated by the compiler: 16
    std::array<int, size> data{};            // usable where a compile-time constant is required
    static_assert(size == 16);
    std::cout << data.size() << '\n';

    int runtime = 5;
    std::cout << square(runtime) << '\n';    // a constexpr function can also run at run time
}
```

`constexpr` variables must be computable at compile time; `constexpr` functions *can* be. C++20 adds
`consteval` for functions that **must** run at compile time. Prefer `constexpr` over `#define` for
constants — it has a type and a scope.

### enum class: scoped, strongly typed enums

```cpp
enum class Color { Red, Green, Blue };
enum class Light { Red, Yellow, Green };   // no clash with Color::Red

Color c = Color::Red;
// int n = c;                     // error: no implicit conversion to int
int n = static_cast<int>(c);      // explicit is fine
```

Always prefer `enum class` over plain `enum` in C++.

### Type aliases

```cpp
using Score = int;                                 // modern form of typedef
using Callback = void (*)(int);                    // much easier to read than the typedef
```

### Structured bindings

Unpack a struct, pair or array into named variables:

```cpp
#include <iostream>
#include <map>
#include <string>

struct Point { int x, y; };

int main()
{
    Point p{3, 4};
    auto [x, y] = p;                        // x = 3, y = 4
    std::cout << x + y << '\n';

    std::map<std::string, int> ages{{"Ada", 36}, {"Alan", 41}};
    for (const auto &[name, age] : ages) {  // each element is a pair (key, value)
        std::cout << name << " is " << age << '\n';
    }
}
```

### C++ casts

C's `(type)value` can do anything, which hides mistakes. C++ has named casts, each for one job:

| Cast | Use |
|---|---|
| `static_cast<T>(x)` | ordinary conversions: `double` → `int`, `int` → `enum class`, `void *` → `T *` |
| `const_cast<T>(x)` | add/remove `const` — almost never needed; a smell |
| `reinterpret_cast<T>(x)` | reinterpret bits: pointer ↔ integer, unrelated pointer types — low-level only |
| `dynamic_cast<T>(x)` | checked downcast in class hierarchies (day 11) |

They're long and ugly on purpose: easy to search for, and they make you think.

### auto and references

`auto` drops references and top-level `const`. To avoid copies, say so:

```cpp
std::vector<std::string> names = {"a", "b"};
for (auto n : names) {}         // copies each string
for (const auto &n : names) {}  // no copies, read-only  <- the usual choice
for (auto &n : names) {}        // no copies, can modify
```

## Common mistakes

- Returning a reference to a local variable — the same dangling problem as in C (day 9 covers
  lifetime bugs).
- `for (auto x : big_objects)` copying everything.
- Expecting `int &r = x; r = y;` to re-point `r` — it assigns `y`'s value to `x`.
- Using plain `enum` or `#define` constants out of C habit.

## Exercises

1. Write `void min_max(const std::vector<int> &v, int &min, int &max)` (assume `v` isn't empty).
2. Write `void to_upper(std::string &s)` that modifies in place, and
   `std::string to_upper_copy(const std::string &s)` that returns a new string. When would you use
   each?
3. Predict the output, then check:

   ```cpp
   int a = 1, b = 2;
   int &r = a;
   r = b;
   r = 10;
   std::cout << a << ' ' << b << '\n';
   int *p = &a;
   p = &b;
   *p = 30;
   std::cout << a << ' ' << b << '\n';
   ```

   <details><summary>Answer</summary>

   `10 2` then `10 30` — `r = b` copied 2 into `a` (a reference can't be re-seated), then
   `r = 10` set `a` to 10. The pointer, by contrast, *was* re-pointed to `b`.

   </details>

4. Write a `constexpr` function `factorial` and use `static_assert` to check `factorial(5) == 120`
   at compile time. What happens if the assertion is wrong?
5. Define `enum class Weekday` and a function `std::string to_string(Weekday d)` with a `switch`.
   Compile with `-Wall` and leave out a case.
6. Use structured bindings to return two values: write `std::pair<int, int> divide(int a, int b)`
   (from `<utility>`) and call it as `auto [q, r] = divide(17, 5);`.
7. Replace every C-style cast in a C day 23/24 program with the right C++ cast.
8. ★ Try binding a non-`const` `int &` to a literal (`int &r = 5;`) and a `const int &` to it. Read
   the error and explain the difference.

## Check yourself

1. Name three differences between a reference and a pointer.
2. Why pass `const std::string &` instead of `std::string`?
3. What does `constexpr` guarantee for a variable?
4. Why is `enum class` better than `enum`?
5. Which C++ cast would you use to convert a `double` to an `int`?

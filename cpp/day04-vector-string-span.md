# C++ Day 4 — vector, array, string_view and span

**Goal:** use the core containers for everyday work, and understand the non-owning "view" types
that replace C's pointer-plus-length pairs.

## Concepts

### std::vector: the default container

`std::vector<T>` is the IntVec you built in C day 14 — a growable heap array — for any type, and
with automatic memory management.

```cpp
#include <iostream>
#include <vector>

int main()
{
    std::vector<int> v;                 // empty
    for (int i = 1; i <= 5; ++i) {
        v.push_back(i * 10);            // grows as needed (doubling, like your IntVec)
    }
    std::cout << "size " << v.size() << ", capacity " << v.capacity() << '\n';

    v[0] = 1;                           // unchecked access, like a C array
    v.at(1) = 2;                        // checked: throws std::out_of_range if invalid
    std::cout << v.front() << ' ' << v.back() << '\n';   // 1 50

    v.pop_back();                       // remove last
    v.insert(v.begin() + 1, 99);        // insert at index 1 (shifts the rest: O(n))
    v.erase(v.begin());                 // remove index 0

    for (int x : v) std::cout << x << ' ';
    std::cout << '\n';                  // 99 2 30 40

    std::vector<int> zeros(10);         // 10 zeros   <- parentheses: a count
    std::vector<int> two{10, 20};       // {10, 20}   <- braces: the elements
    std::vector<std::vector<int>> grid(3, std::vector<int>(4, 0));   // 3x4 of zeros
    std::cout << zeros.size() << ' ' << two.size() << ' ' << grid[2].size() << '\n';   // 10 2 4
}
```

- `reserve(n)` pre-allocates capacity when you know the final size, avoiding repeated
  reallocations.
- `emplace_back(args...)` constructs an element in place from constructor arguments (useful for
  objects — day 5).
- `v.data()` gives a `T *` to the elements — for passing to C functions (C++ day 21).
- A vector frees its memory automatically when it goes out of scope. No `free`, no leaks.

**Use `std::vector` unless you have a reason not to.** Its elements are contiguous in memory, which
makes it faster than linked structures for almost everything (day 17 explains why).

### Iterator invalidation: the pointer lesson returns

When a vector grows, it moves its elements to a new block — exactly like `realloc` in C day 12.
Any pointer, reference or iterator into the old block **dangles**:

```cpp
std::vector<int> v = {1, 2, 3};
int &first = v[0];
v.push_back(4);          // may reallocate
first = 10;              // UNDEFINED BEHAVIOR if it did
```

Day 9 is a whole lesson on these bugs.

### std::array: fixed size, no heap

```cpp
#include <array>

std::array<int, 5> a = {1, 2, 3, 4, 5};   // size is part of the type, on the stack
a.size();                                 // 5 — knows its size, unlike a C array
```

Use `std::array` instead of C arrays: same performance, but it has `.size()`, can be copied and
returned, and doesn't decay to a pointer.

### std::string, revisited

Useful members beyond day 1: `append`, `insert`, `erase`, `replace`, `find`, `rfind`, `substr`,
`starts_with`/`ends_with` (C++20), `c_str()` (a `const char *` for C functions), and conversions
`std::to_string(42)`, `std::stoi("42")`, `std::stod("3.5")` (which throw on bad input).

### std::string_view: a non-owning view of characters

`std::string_view` (from `<string_view>`) is just a **pointer and a length** — exactly the
`(const char *, size_t)` pair you passed around in C — wrapped in a type with string-like member
functions. It doesn't own or copy anything.

```cpp
#include <iostream>
#include <string>
#include <string_view>

bool is_keyword(std::string_view word)       // accepts std::string, literals, substrings — no copy
{
    return word == "if" || word == "for" || word == "while";
}

std::string_view first_word(std::string_view s)
{
    auto end = s.find(' ');
    return s.substr(0, end);                 // substr of a view is a view: no allocation
}

int main()
{
    std::string line = "while true";
    std::cout << is_keyword(first_word(line)) << '\n';   // 1
    std::cout << is_keyword("return") << '\n';          // 0
}
```

Use `std::string_view` for **read-only string parameters**. But since it doesn't own the
characters, it must never outlive them:

```cpp
std::string_view bad()
{
    std::string s = "temporary";
    return s;            // BUG: the view points into s, which is destroyed here
}
```

### std::span (C++20): a non-owning view of a contiguous sequence

`std::span<T>` (from `<span>`) is the same idea for any element type: pointer + length. One function
accepts a `std::vector`, a `std::array`, or a C array:

```cpp
#include <array>
#include <iostream>
#include <span>
#include <vector>

double average(std::span<const int> values)     // const int: read-only view
{
    if (values.empty()) return 0.0;
    long long sum = 0;
    for (int x : values) sum += x;
    return static_cast<double>(sum) / values.size();
}

void fill(std::span<int> values, int x)         // non-const: may modify the elements
{
    for (int &v : values) v = x;
}

int main()
{
    std::vector<int> v = {1, 2, 3, 4};
    std::array<int, 3> a = {10, 20, 30};
    int c[] = {5, 5, 5, 5, 5};

    std::cout << average(v) << ' ' << average(a) << ' ' << average(c) << '\n';   // 2.5 20 5
    fill(std::span{v}.subspan(1, 2), 0);        // only elements 1 and 2
    std::cout << v[0] << v[1] << v[2] << v[3] << '\n';   // 1004
}
```

This replaces C's `const int *a, size_t n` parameter pair — the size can no longer get out of sync
with the pointer.

### Owning vs viewing

| Owns its data (frees it) | Non-owning view (just looks) |
|---|---|
| `std::string` | `std::string_view` |
| `std::vector<T>`, `std::array<T, N>` | `std::span<T>` |

Function parameters are usually views (or `const &`); data members and return values are usually
owners.

## Common mistakes

- `std::vector<int> v(10)` vs `v{10}`: ten zeros vs one element with value 10.
- Keeping references/iterators/pointers into a vector across `push_back`/`insert`.
- Returning a `string_view` or `span` to a local container.
- `v[i]` with an invalid `i` — undefined behavior, as in C. Use `at` while learning, or compile
  with `-D_GLIBCXX_ASSERTIONS` to make `[]` checked in libstdc++.
- Comparing a signed `int i` with `v.size()` (unsigned) — use `std::size_t`, or `std::ssize(v)`
  (C++20) which returns a signed size.

## Exercises

1. Rewrite the C day 7 grade book core with `std::vector<int>`: read scores until -1, then print
   min, max, mean and median. No manual memory management at all.
2. Write `std::vector<int> merge_sorted(std::span<const int> a, std::span<const int> b)`.
3. Write `std::vector<std::string_view> split(std::string_view s, char sep)` — the views point into
   `s`. Then show the bug: call it on a temporary string and use the result.
4. Write `bool is_palindrome(std::string_view s)` with two indexes moving toward each other.
5. Print `size()` and `capacity()` after each of 20 `push_back`s. What growth factor does your
   standard library use? Then call `reserve(20)` first and repeat.
6. Make a 2D grid with `std::vector<std::vector<char>>` for a 10×20 board and draw a rectangle in
   it; print it.
7. Write `void rotate_left(std::span<int> s)` and test it with a vector, an array, and a subspan.
8. ★ Implement the Sieve of Eratosthenes with `std::vector<bool>` for 10⁷ numbers and time it.

## Check yourself

1. What happens to references into a vector when it reallocates?
2. When would you choose `std::array` over `std::vector`?
3. What's inside a `std::string_view`?
4. What C pattern does `std::span` replace?
5. Why is returning a `string_view` from a function risky?

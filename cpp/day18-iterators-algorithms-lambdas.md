# C++ Day 18 — Iterators, algorithms and lambdas in depth

**Goal:** use the standard algorithms instead of hand-written loops, and master lambda captures.

## Concepts

### Iterators: generalized pointers

An iterator is an object that points into a container and moves like a pointer: `*it` gives the
element, `++it` moves to the next one, `it == end` compares. `begin()` points to the first element,
`end()` to **one past the last** — exactly C day 9's `(begin, end)` pointer pair. For a `vector`,
iterators are basically pointers; for a `map`, `++it` walks the tree.

```cpp
std::vector<int> v = {3, 1, 4};
for (auto it = v.begin(); it != v.end(); ++it) {
    *it *= 2;
}
```

Categories, from weakest to strongest: **input** (read once, forward), **forward** (re-readable),
**bidirectional** (`--it`: `list`, `map`), **random access** (`it + n`, `it[n]`: `vector`,
`deque`), **contiguous** (elements adjacent in memory: `vector`, `array`, `string`). Algorithms
state what they need: `std::sort` needs random access, which is why `std::list` has its own
`.sort()`.

### The algorithms

`<algorithm>` and `<numeric>` hold ~100 algorithms that work on any iterator range. Learn them —
each replaces a loop you'd otherwise write and debug:

```cpp
#include <algorithm>
#include <iostream>
#include <numeric>
#include <string>
#include <vector>

int main()
{
    std::vector<int> v = {5, 3, 8, 1, 9, 2, 8};

    int total = std::accumulate(v.begin(), v.end(), 0);                     // 36
    auto max_it = std::max_element(v.begin(), v.end());                     // iterator to 9
    long evens = std::count_if(v.begin(), v.end(), [](int x) { return x % 2 == 0; });   // 3
    bool any_big = std::any_of(v.begin(), v.end(), [](int x) { return x > 8; });        // true
    auto first8 = std::find(v.begin(), v.end(), 8);                         // iterator to the first 8

    std::cout << total << ' ' << *max_it << ' ' << evens << ' ' << any_big << ' '
              << (first8 - v.begin()) << '\n';                              // 36 9 3 1 2

    std::vector<int> squares(v.size());
    std::transform(v.begin(), v.end(), squares.begin(), [](int x) { return x * x; });

    std::sort(v.begin(), v.end());                                          // 1 2 3 5 8 8 9
    v.erase(std::unique(v.begin(), v.end()), v.end());                      // 1 2 3 5 8 9
    bool has5 = std::binary_search(v.begin(), v.end(), 5);                  // on sorted data
    auto pos = std::lower_bound(v.begin(), v.end(), 4);                     // first >= 4: the 5

    std::vector<int> seq(10);
    std::iota(seq.begin(), seq.end(), 1);                                   // 1..10
    std::reverse(seq.begin(), seq.end());

    std::cout << has5 << ' ' << *pos << ' ' << seq.front() << ' ' << squares[0] << '\n';   // 1 5 10 25
}
```

A short tour, by job:

| Job | Algorithms |
|---|---|
| Search | `find`, `find_if`, `search`, `binary_search`, `lower_bound`, `upper_bound`, `equal_range` |
| Count / test | `count`, `count_if`, `all_of`, `any_of`, `none_of`, `equal`, `mismatch` |
| Min / max | `min_element`, `max_element`, `minmax_element`, `clamp` |
| Transform | `transform`, `for_each`, `replace_if`, `fill`, `generate`, `iota` |
| Reorder | `sort`, `stable_sort`, `partial_sort`, `nth_element`, `reverse`, `rotate`, `shuffle`, `partition` |
| Remove | `remove`, `remove_if`, `unique` (+ `erase`), or C++20 `std::erase` / `std::erase_if` |
| Sets (sorted ranges) | `set_union`, `set_intersection`, `set_difference`, `merge`, `includes` |
| Numeric | `accumulate`, `reduce`, `inner_product`, `partial_sum`, `adjacent_difference` |

`remove`/`remove_if` don't erase — they move the kept elements to the front and return the new
logical end. That's why the "erase-remove idiom" `v.erase(std::remove_if(...), v.end())` exists;
in C++20 just write `std::erase_if(v, pred)`.

Tomorrow's **ranges** let you write `std::ranges::sort(v)` instead of `std::sort(v.begin(), v.end())`.

### Lambdas in depth

```cpp
[captures](parameters) -> return_type { body }
```

The return type is usually deduced. Captures decide how the lambda sees surrounding local
variables:

| Capture | Meaning |
|---|---|
| `[]` | nothing |
| `[x]` | copy of `x` (made when the lambda is created) |
| `[&x]` | reference to `x` |
| `[=]` | copy of everything used (avoid: hides what's captured, and captures `this` implicitly in older code) |
| `[&]` | reference to everything used — fine for lambdas used immediately, dangerous if stored (day 9) |
| `[this]` | the current object, by pointer |
| `[p = std::move(ptr)]` | init-capture: a new variable initialized however you like — e.g. move a `unique_ptr` in |

```cpp
#include <functional>
#include <iostream>
#include <memory>

int main()
{
    int calls = 0;
    auto counted_square = [&calls](int x) { ++calls; return x * x; };   // modifies calls via reference
    counted_square(3);
    counted_square(4);
    std::cout << calls << '\n';                                          // 2

    auto counter = [n = 0]() mutable { return ++n; };                    // owns its own state
    counter();
    std::cout << counter() << '\n';                                      // 2

    auto data = std::make_unique<int>(42);
    auto reader = [p = std::move(data)] { return *p; };                  // lambda now owns the int
    std::cout << reader() << '\n';                                       // 42

    auto add = [](auto a, auto b) { return a + b; };                     // generic lambda (a template)
    std::cout << add(1, 2) << ' ' << add(1.5, 2.25) << '\n';             // 3 3.75

    auto make_multiplier = [](int k) { return [k](int x) { return x * k; }; };   // returns a lambda
    auto triple = make_multiplier(3);
    std::cout << triple(7) << '\n';                                      // 21

    std::function<int(int)> f = triple;                                  // type-erased holder
    std::cout << f(2) << '\n';                                           // 6
}
```

- By default a lambda's `operator()` is `const`: copies captured by value can't be modified unless
  the lambda is `mutable`.
- Each lambda has its own unique type. Store it with `auto`, pass it to templates, or — when you
  need one type for different lambdas (a vector of callbacks, a class member) — wrap it in
  `std::function<R(Args...)>`, which costs a heap allocation and an indirect call.

### What a lambda really is

The compiler turns a lambda into a class with the captures as members and an `operator()`:

```cpp
// [k](int x) { return x * k; } becomes roughly:
struct __lambda {
    int k;
    int operator()(int x) const { return x * k; }
};
```

C day 13's "function pointer + `void *ctx`" pattern, generated and type-checked. And since the call
is direct, `std::sort` with a lambda can inline the comparison — faster than `qsort`.

## Common mistakes

- Writing a raw loop where an algorithm says what you mean (`any_of`, `count_if`, `find_if`).
- `std::remove_if` without `erase`.
- `std::sort` with a comparator using `<=` (must be a strict weak ordering).
- Storing lambdas that capture locals by reference.
- `std::accumulate(v.begin(), v.end(), 0)` on doubles — the `0` makes the accumulator an `int`!
  Use `0.0`.

## Exercises

Solve each **without writing a loop**, using only algorithms and lambdas:

1. Given `std::vector<int>`: the sum of squares of the odd numbers (`transform` + `accumulate`, or
   `std::transform_reduce`).
2. Remove all negative numbers from a vector, then sort the rest descending.
3. Given `std::vector<std::string>`: the longest word; the number of words starting with a capital;
   whether all words are shorter than 10 letters.
4. Find the median of a vector using `std::nth_element` (O(n) on average — why is that better than
   sorting?).
5. Given a vector of `struct Employee { std::string name, dept; double salary; }`: sort by
   department, then by salary descending (`std::sort` with a comparator using `std::tie`, or
   `std::stable_sort` twice); the total salary per department (a `std::map` and `std::for_each`).
6. Check whether two strings are anagrams (sort copies and compare with `==`, or
   `std::is_permutation`).
7. Generate 20 random numbers in [1, 100] (`<random>`: `std::mt19937`,
   `std::uniform_int_distribution`, `std::generate`), then compute the running totals
   (`std::partial_sum`).
8. Write `make_counter()` returning a lambda that counts its calls, and `make_accumulator(start)`
   returning a lambda that adds its argument to a running total. Store three different lambdas in a
   `std::vector<std::function<int(int)>>`.
9. ★ Implement your own `my_find_if(It first, It last, Pred p)` template and `my_transform`, and
   check them against the standard ones.

## Check yourself

1. What does `end()` point to?
2. Why does `std::sort` not work on `std::list`?
3. Why do you need `erase` after `remove_if`?
4. What's the difference between `[x]` and `[&x]`, and when is `[&]` dangerous?
5. What does `mutable` do on a lambda?
6. When would you use `std::function` instead of `auto`?

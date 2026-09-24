# C++ Day 19 — Ranges (C++20)

**Goal:** use range algorithms, projections and lazy views to write data-processing code as
readable pipelines.

## Concepts

### Range algorithms: pass the container, not two iterators

Every classic algorithm has a version in `std::ranges` that takes a whole range:

```cpp
#include <algorithm>
#include <iostream>
#include <string>
#include <vector>

struct Employee {
    std::string name;
    std::string dept;
    double salary;
};

int main()
{
    std::vector<int> v = {5, 3, 8, 1};
    std::ranges::sort(v);                                   // instead of std::sort(v.begin(), v.end())
    std::cout << std::ranges::binary_search(v, 8) << '\n';  // 1

    std::vector<Employee> staff = {
        {"Ann", "IT", 5200}, {"Bob", "Sales", 4100}, {"Cid", "IT", 6100},
    };

    // Projections: sort by a member, no hand-written comparator
    std::ranges::sort(staff, {}, &Employee::salary);                     // ascending by salary
    std::ranges::sort(staff, std::ranges::greater{}, &Employee::name);   // descending by name

    auto richest = std::ranges::max_element(staff, {}, &Employee::salary);
    auto it = std::ranges::find(staff, "Bob", &Employee::name);          // find by member value
    std::cout << richest->name << ' ' << it->dept << '\n';               // Cid Sales
}
```

- `{}` is the default comparator (`std::ranges::less`).
- A **projection** transforms each element before comparing — a member pointer
  `&Employee::salary`, or any lambda.
- Range algorithms are **constrained with concepts**, so misuse gives clear errors.
- They refuse to return dangling iterators: calling `std::ranges::find` on a temporary vector
  returns `std::ranges::dangling` instead of an iterator into a dead object, and using it doesn't
  compile. Day 9 would approve.

### Views: lazy, composable ranges

A **view** is a lightweight range that doesn't own elements; it computes them on demand from
another range. Views are combined with `|`, like a Unix pipeline:

```cpp
#include <iostream>
#include <ranges>
#include <vector>

int main()
{
    std::vector<int> v = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

    auto result = v
        | std::views::filter([](int x) { return x % 2 == 0; })   // keep evens
        | std::views::transform([](int x) { return x * x; })     // square them
        | std::views::take(3);                                   // first three

    for (int x : result) std::cout << x << ' ';                  // 4 16 36
    std::cout << '\n';

    for (int i : std::views::iota(1, 6)) std::cout << i;         // 12345 — numbers, no container
    std::cout << '\n';

    for (int x : v | std::views::reverse | std::views::drop(7)) std::cout << x;   // 321
    std::cout << '\n';
}
```

**Nothing is computed when `result` is created.** Each element is produced when the `for` loop asks
for it: filter checks `1` (rejected), `2` (kept) → transform gives `4` → take counts 1... After the
third element, `take` stops, so elements 7–10 are never even looked at. No temporary vectors are
allocated.

### Useful views

| View | Produces |
|---|---|
| `views::iota(a, b)` / `iota(a)` | a, a+1, ..., b-1 / infinite |
| `views::filter(pred)` | elements satisfying pred |
| `views::transform(f)` | f(element) |
| `views::take(n)`, `views::drop(n)` | first n / all but the first n |
| `views::take_while(p)`, `views::drop_while(p)` | prefix while p holds / the rest |
| `views::reverse` | reversed (needs bidirectional) |
| `views::keys`, `views::values` | first / second of pairs (e.g. a `map`) |
| `views::elements<N>` | the N-th member of tuples |
| `views::split(delim)` | subranges between delimiters |
| `views::join` | flattens a range of ranges |
| `views::common` | adapts a view for older iterator-pair APIs |

C++23 adds `views::enumerate` (index + element), `views::zip`, `views::chunk`, `views::slide`,
`views::join_with`, and `std::ranges::to<Container>()` to collect a view into a container. In C++20
you collect manually:

```cpp
auto evens = v | std::views::filter([](int x) { return x % 2 == 0; });
std::vector<int> collected(evens.begin(), evens.end());
```

### Splitting a string

```cpp
#include <iostream>
#include <ranges>
#include <string>
#include <string_view>
#include <vector>

int main()
{
    std::string_view csv = "alpha,beta,,gamma";
    std::vector<std::string> fields;
    for (auto part : csv | std::views::split(',')) {
        fields.emplace_back(part.begin(), part.end());   // each part is a subrange of chars
    }
    for (const auto &f : fields) std::cout << '[' << f << ']';   // [alpha][beta][][gamma]
    std::cout << '\n';
}
```

### Rules for views

- Views **refer to** their source. The source must outlive the view (day 9 again).
- Some views (like `filter`) cache things on first iteration, so iterate a `filter` view through a
  non-`const` variable.
- Keep pipelines readable: name intermediate lambdas, and break long pipelines onto several lines.
- Views are great for transforming data; for simple loops, a plain `for` is still fine.

## Common mistakes

- Expecting a view to hold results — it holds instructions; a changed source changes the output.
- A view over a temporary container that has died (a view returned from a function that built a
  local vector).
- Forgetting that `views::transform`'s lambda runs every time an element is accessed — expensive
  lambdas in pipelines iterated twice run twice.
- Passing a `const` filter view to a function expecting a range (some views need non-const
  iteration).

## Exercises

Solve with ranges and views, no raw loops except the final printing `for`:

1. Print the squares of the first 10 odd numbers, using `iota`, `filter`, `transform`, `take`.
2. Sort a `std::vector<Employee>` by department, then by salary descending — using
   `std::ranges::sort` with a lambda comparator that uses `std::tie`, or two stable sorts with
   projections (`std::ranges::stable_sort`).
3. From a `std::map<std::string, int>` of word counts, print the keys of entries with count > 2
   (`views::filter` + `views::keys`).
4. Split a sentence into words with `views::split(' ')`, keep words longer than 3 letters, and
   convert them to uppercase into a `std::vector<std::string>`.
5. Find the first number > 1000 whose square ends in `...444` using an infinite `iota` and
   `std::ranges::find_if`. (Laziness makes the infinite range fine.)
6. Rewrite three exercises from day 18 with range algorithms and projections. Which versions read
   better?
7. ★ Compile a pipeline with `-std=c++23` and use `views::enumerate`, `views::zip` and
   `std::ranges::to<std::vector>()`.

## Check yourself

1. What's a projection? Give an example.
2. What does "lazy" mean for a view? When is the lambda in `views::transform` called?
3. Why can a view dangle?
4. What does `std::ranges::dangling` protect you from?
5. How do you collect a view into a vector in C++20? In C++23?

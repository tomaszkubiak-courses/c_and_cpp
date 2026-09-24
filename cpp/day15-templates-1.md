# C++ Day 15 — Templates I: generic functions and classes

**Goal:** write function and class templates, understand how the compiler instantiates them, and
know why template code lives in headers.

## Concepts

### The problem templates solve

In C you wrote generic code with `void *` and element sizes (C day 13): flexible, but no type
checking and easy to get wrong. Or you wrote the same function once per type. Templates let you
write the code once, and the compiler generates a type-checked version for each type you use.

### Function templates

```cpp
#include <iostream>
#include <string>

template <typename T>
T max_of(const T &a, const T &b)
{
    return (a < b) ? b : a;
}

int main()
{
    std::cout << max_of(3, 7) << '\n';                     // T = int
    std::cout << max_of(2.5, 1.5) << '\n';                 // T = double
    std::cout << max_of(std::string{"pear"}, std::string{"apple"}) << '\n';   // T = std::string
    std::cout << max_of<double>(3, 4.5) << '\n';           // explicit T: 3 converts to double
    // max_of(3, 4.5);                                     // error: T can't be both int and double
}
```

`template <typename T>` declares a type parameter (`class T` means the same). When you call
`max_of(3, 7)`, the compiler **deduces** `T = int` and **instantiates** a real function
`int max_of(const int &, const int &)`. The template itself isn't code; each instantiation is.

The requirements on `T` are implicit: `max_of` works for any type with `<`. Call it with a type that
has no `<` and you get an error — often a long one, pointing deep into the template. Tomorrow's
**concepts** make those requirements explicit.

### Class templates

`std::vector<int>` is a class template instantiated with `int`. Your own:

```cpp
#include <cstddef>
#include <iostream>
#include <stdexcept>
#include <string>
#include <utility>
#include <vector>

template <typename T>
class Stack {
public:
    void push(const T &value) { items_.push_back(value); }
    void push(T &&value) { items_.push_back(std::move(value)); }

    T pop()
    {
        if (items_.empty()) throw std::out_of_range{"pop on empty stack"};
        T top = std::move(items_.back());
        items_.pop_back();
        return top;
    }

    const T &top() const { return items_.back(); }
    bool empty() const { return items_.empty(); }
    std::size_t size() const { return items_.size(); }

private:
    std::vector<T> items_;
};

int main()
{
    Stack<int> numbers;
    numbers.push(1);
    numbers.push(2);
    std::cout << numbers.pop() << numbers.pop() << '\n';   // 21

    Stack<std::string> words;
    words.push("hello");
    std::cout << words.top() << ' ' << words.size() << '\n';   // hello 1
}
```

Compare with C day 22's `CharStack`: one definition now works for every type — and there's no
`realloc` or `free` either.

### Non-type template parameters

Template parameters can also be values known at compile time:

```cpp
template <typename T, std::size_t N>
class RingBuffer {
public:
    bool push(const T &v)
    {
        if (count_ == N) return false;
        items_[(head_ + count_++) % N] = v;
        return true;
    }
    // ...
private:
    std::array<T, N> items_{};        // size fixed at compile time, no heap
    std::size_t head_ = 0, count_ = 0;
};

RingBuffer<int, 8> rb;               // C day 22's ring buffer, generic
```

`std::array<T, N>` itself is defined this way.

### Class template argument deduction (CTAD)

Since C++17 the compiler can often deduce class template arguments from the constructor:

```cpp
std::vector v = {1, 2, 3};          // std::vector<int>
std::pair p{1, 2.5};                // std::pair<int, double>
```

### Why templates live in headers

The compiler needs the template's **full definition** at each point of use to generate code for
the specific `T`. So class and function templates are defined entirely in headers, not split into
`.h` and `.cpp` like ordinary code (member functions defined outside the class need a
`template <typename T>` prefix and `Stack<T>::`). Defining them in a header doesn't cause
"multiple definition" link errors — templates are exempt, like `inline` functions.

### Specialization (briefly)

You can provide a different implementation for a specific type:

```cpp
template <typename T>
std::string describe(const T &) { return "something"; }

template <>
std::string describe<bool>(const bool &b) { return b ? "yes" : "no"; }
```

Overloading plain functions is usually simpler than specializing function templates; class template
specialization is more common (`std::vector<bool>` is a famous — and infamous — one).

### How templates compare to C's void *

| | C: `void *` + size + function pointer | C++: templates |
|---|---|---|
| Type checking | none | full |
| Speed | indirect calls, `memcpy` | direct, inlinable — often faster than hand-written C |
| Binary size | one copy | one copy per type used ("code bloat" if overused) |
| Error messages | none (silent bugs) | at compile time (long, but concepts help) |

`std::sort` with a lambda is typically faster than `qsort`, because the comparison is inlined.

## Common mistakes

- Putting template definitions in a `.cpp` file → "undefined reference" at link time.
- Expecting deduction to convert: `max_of(3, 4.5)` fails.
- Unclear requirements on `T`, discovered through huge error messages (tomorrow: concepts).
- Templating things that don't need to be generic (YAGNI).

## Exercises

1. Write `template <typename T> void print_all(const std::vector<T> &v)`. Call it with `int`,
   `double` and `std::string` vectors. What happens with a `std::vector<std::vector<int>>`, and why?
2. Write `template <typename T> T sum(std::span<const T> values)` and call it with a vector and an
   array. (You may need to write `sum<int>(v)` or `sum(std::span<const int>{v})` — why?)
3. Write your C day 24 insertion sort as `template <typename T, typename Compare> void insertion_sort(std::vector<T> &v, Compare less)`
   and call it with lambdas. Compare the code with the `void *` version from C day 13.
4. Type in `Stack<T>` and add `std::optional<T> try_pop()`.
5. Finish `RingBuffer<T, N>` with `pop`, `size`, `full`, `empty`, and test with `int` and
   `std::string`.
6. Write a class template `Pair<A, B>` with `first`, `second`, a `swap()` returning `Pair<B, A>`,
   and `operator<<`.
7. Try `max_of` with a struct that has no `operator<`. Read the error message. Keep it for tomorrow.
8. ★ Look at `nm -C` output for a program using `max_of<int>` and `max_of<double>`: find both
   instantiations.

## Check yourself

1. What does "instantiation" mean?
2. Why must templates be defined in headers?
3. What's a non-type template parameter? Give an example from the standard library.
4. How do templates compare to `void *` generic code?
5. What's CTAD?

# C++ Day 10 — Operator overloading and comparisons

**Goal:** make your types behave like built-in ones — `+`, `==`, `<`, `<<`, `[]` — and use C++20's
`<=>` and defaulted comparisons.

## Concepts

### Operators are functions with special names

`a + b` for class types calls `operator+(a, b)` or `a.operator+(b)`. You can define them for your
types, which makes code read naturally — **if** the meaning is obvious. Overload operators only
when they mean what they mean for numbers and containers; don't make `+` do something surprising.

### A complete example: Vec2

```cpp
#include <cmath>
#include <iostream>

struct Vec2 {
    double x = 0, y = 0;

    Vec2 &operator+=(const Vec2 &o) { x += o.x; y += o.y; return *this; }   // member: modifies *this
    Vec2 &operator*=(double k) { x *= k; y *= k; return *this; }
    Vec2 operator-() const { return {-x, -y}; }                               // unary minus

    double length() const { return std::hypot(x, y); }

    bool operator==(const Vec2 &) const = default;   // C++20: memberwise ==, and != for free
};

// Binary operators as free functions, implemented with the compound ones
Vec2 operator+(Vec2 a, const Vec2 &b) { return a += b; }   // a is a copy: modify and return it
Vec2 operator*(Vec2 v, double k) { return v *= k; }
Vec2 operator*(double k, Vec2 v) { return v *= k; }        // so 2 * v works as well as v * 2

std::ostream &operator<<(std::ostream &os, const Vec2 &v)   // printing
{
    return os << '(' << v.x << ", " << v.y << ')';
}

int main()
{
    Vec2 a{1, 2}, b{3, 4};
    Vec2 c = a + b * 2;
    std::cout << c << ' ' << -c << ' ' << (2 * a == a + a) << '\n';   // (7, 10) (-7, -10) 1
    std::cout << b.length() << '\n';                                  // 5
}
```

Conventions worth following:

- **Compound assignment** (`+=`, `*=`) as members returning `*this` by reference.
- **Binary operators** (`+`, `*`) as free functions, built on the compound ones. Free functions
  allow conversions on both sides and symmetric forms like `2 * v`.
- **`operator<<`** for printing is always a free function taking and returning `std::ostream &`, so
  calls chain.

### C++20 comparisons: == and <=>

Before C++20 you wrote six comparison operators by hand. Now:

- `bool operator==(const T &) const = default;` gives `==` and `!=` comparing all members in order.
- `auto operator<=>(const T &) const = default;` — the **three-way comparison** ("spaceship")
  operator — gives `<`, `<=`, `>`, `>=` comparing members lexicographically. (Defaulting `<=>`
  also defaults `==`.)

```cpp
#include <compare>
#include <iostream>
#include <set>
#include <string>

struct Version {
    int major = 0, minor = 0, patch = 0;
    auto operator<=>(const Version &) const = default;   // compares major, then minor, then patch
};

struct Person {
    std::string last, first;
    int age = 0;
    // custom ordering: by last name, then first name; ignore age
    std::weak_ordering operator<=>(const Person &o) const
    {
        if (auto c = last <=> o.last; c != 0) return c;
        return first <=> o.first;
    }
    bool operator==(const Person &o) const { return last == o.last && first == o.first; }
};

int main()
{
    Version a{1, 4, 2}, b{1, 10, 0};
    std::cout << (a < b) << (a == b) << (a >= Version{1, 4, 2}) << '\n';   // 101

    std::set<Person> people{{"Hopper", "Grace", 85}, {"Lovelace", "Ada", 36}, {"Hopper", "Alan", 1}};
    for (const auto &p : people) std::cout << p.first << ' ' << p.last << '\n';
}
```

`a <=> b` returns an ordering value you compare with 0: `< 0` means `a < b`. The result type says
what kind of ordering it is: `std::strong_ordering` (equal values are identical — ints),
`std::weak_ordering` (equivalent but distinguishable — case-insensitive strings, or `Person`
above ignoring age), `std::partial_ordering` (some values are incomparable — doubles, because of
NaN).

Once a type has `<`, it works with `std::sort`, `std::set`, `std::map` keys, `std::max` and so on.

### Other operators you'll meet

```cpp
class Matrix {
public:
    double &operator()(std::size_t r, std::size_t c) { return data_[r * cols_ + c]; }         // m(1, 2)
    double operator()(std::size_t r, std::size_t c) const { return data_[r * cols_ + c]; }
    // ...
};

class IntArray {
public:
    int &operator[](std::size_t i) { return data_[i]; }                // a[3] = 5
    const int &operator[](std::size_t i) const { return data_[i]; }    // for const objects
    // ...
};
```

Provide `const` and non-`const` versions of access operators. Others:

- `++`/`--`: prefix `T &operator++()` and postfix `T operator++(int)` (the `int` is a dummy
  parameter that marks the postfix form).
- `explicit operator bool() const` — lets objects be tested in `if (obj)`, like smart pointers and
  streams.
- `operator()` with no fixed meaning makes **function objects** — that's what lambdas are compiled
  into (day 18).

### friend

A `friend` function isn't a member but can access private members. Useful for `operator<<` when it
needs private data:

```cpp
class Money {
public:
    explicit Money(long cents) : cents_{cents} {}
    friend std::ostream &operator<<(std::ostream &os, const Money &m)
    {
        return os << m.cents_ / 100 << '.' << (m.cents_ % 100 < 10 ? "0" : "") << m.cents_ % 100;
    }
private:
    long cents_;
};
```

Use sparingly; a public getter is often enough.

## Common mistakes

- Operators with surprising meanings (`+` that modifies its left operand, `<<` for anything but
  streams/shifts).
- `operator+` that returns a reference (to a local — dangling).
- Missing `const` versions of `[]`.
- Inconsistent `==` and `<=>` (two objects "equivalent" by `<=>` but not `==`, or vice versa) —
  default both when possible.
- Defaulting `<=>` when members are in the wrong order for the ordering you want (it compares in
  declaration order).

## Exercises

1. Type in `Vec2` and add `operator-` (binary), `operator/`, `dot(const Vec2 &)`, and a
   `normalized()` member.
2. Give yesterday's (day 5) `Fraction` `+ - * /`, compound versions, `==` and `<=>` (compare
   `a/b` with `c/d` via `a*d <=> c*b` — denominators are positive), and `<<`. Sort a
   `std::vector<Fraction>`.
3. Write a `Date` struct (year, month, day) with a defaulted `<=>`, and sort a vector of dates.
   Then add `operator<<` printing `YYYY-MM-DD` with `std::format`.
4. Write a `Money` class (cents in a `long long`) with `+`, `-`, `*` by an integer, comparisons and
   `<<` that prints `12.05`. Why shouldn't it have `operator*(Money, Money)`?
5. Give the day 7 `Matrix` `operator()`, `operator*` (matrix product), `operator==` and `<<`.
6. Write a case-insensitive string wrapper `CIString` with `operator<=>` returning
   `std::weak_ordering`, and put some in a `std::set` — `"Apple"` and `"apple"` should collide.
7. ★ Write a `Counter` iterator-like class with prefix and postfix `++` and `operator*`, and use it
   to understand why prefix `++it` is preferred for class types.

## Check yourself

1. Why implement `+` in terms of `+=`?
2. Why is `operator<<` a free function?
3. What does `= default` on `<=>` compare, and in what order?
4. What's the difference between `strong_ordering` and `weak_ordering`?
5. Why provide both `const` and non-`const` `operator[]`?

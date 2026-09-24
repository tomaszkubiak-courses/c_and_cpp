# C++ Day 5 — Classes I: members, constructors, invariants

**Goal:** define classes with private data and public member functions, write constructors that
establish invariants, and use `const` member functions.

## Concepts

### From C's opaque struct to a class

On C day 27 you protected a bank account's invariant with an opaque struct and functions taking a
`self` pointer. A C++ class does the same, built into the language:

```cpp
#include <iostream>
#include <stdexcept>
#include <string>
#include <utility>

class Account {
public:                                             // accessible to everyone
    Account(std::string owner, long initial_cents)  // constructor: runs when an Account is created
        : owner_{std::move(owner)}, balance_cents_{initial_cents}   // member initializer list
    {
        if (balance_cents_ < 0) {
            throw std::invalid_argument{"negative initial balance"};   // exceptions: day 13
        }
    }

    void deposit(long cents)
    {
        if (cents <= 0) throw std::invalid_argument{"deposit must be positive"};
        balance_cents_ += cents;
    }

    bool withdraw(long cents)
    {
        if (cents <= 0 || cents > balance_cents_) return false;   // protect the invariant
        balance_cents_ -= cents;
        return true;
    }

    long balance() const { return balance_cents_; }       // const: doesn't modify the object
    const std::string &owner() const { return owner_; }

private:                                            // only member functions can touch these
    std::string owner_;
    long balance_cents_;                            // invariant: never negative
};

int main()
{
    Account acc{"Grace", 1000};
    acc.deposit(500);
    if (!acc.withdraw(5000)) {
        std::cout << "insufficient funds\n";
    }
    std::cout << acc.owner() << ": " << acc.balance() << '\n';   // Grace: 1500
    // acc.balance_cents_ = -1;   // error: 'long Account::balance_cents_' is private
}
```

What corresponds to what:

| C (day 27) | C++ |
|---|---|
| `Account *account_create(...)` | constructor `Account(...)` |
| `account_deposit(acc, 500)` | `acc.deposit(500)` |
| the `self` parameter | the hidden `this` pointer |
| opaque struct hides fields | `private:` |
| `account_destroy` | destructor — tomorrow |

### class vs struct

The only difference: members of a `struct` are public by default, members of a `class` private.
Convention: `struct` for plain data bundles with no invariants (`struct Point { double x, y; };`),
`class` when there are invariants to protect.

### Constructors

A constructor has the class's name and no return type. Several can be overloaded:

```cpp
class Rect {
public:
    Rect() = default;                                 // default constructor: uses member defaults
    Rect(double w, double h) : w_{w}, h_{h} {}
    explicit Rect(double side) : Rect{side, side} {}  // delegates to another constructor

    double area() const { return w_ * h_; }

private:
    double w_ = 1.0;                                  // default member initializers
    double h_ = 1.0;
};

Rect a;            // 1 x 1
Rect b{2, 3};      // 2 x 3
Rect c{4};         // 4 x 4
```

- **Initialize members in the initializer list** (`: w_{w}, h_{h}`), not by assignment in the
  body. Members are initialized in the order they're **declared** in the class, whatever order you
  write the list in (`-Wall` warns about mismatches).
- **Default member initializers** (`double w_ = 1.0;`) give every constructor a sensible starting
  point.
- **`explicit`** on single-argument constructors prevents surprising implicit conversions: without
  it, a function `double area_of(const Rect &r)` could be called as `area_of(5.0)`, silently
  turning `5.0` into a 5×5 `Rect`. Make single-argument constructors `explicit` by default.
- If you declare no constructor at all, the compiler provides a default one.

### const member functions

`double area() const` promises not to change the object. Only `const` member functions can be
called on a `const` object or through a `const &`:

```cpp
void print_area(const Rect &r)
{
    std::cout << r.area();   // OK only because area() is const
}
```

Mark every member function that doesn't modify the object `const`. Forgetting it is the most
common beginner error with classes — you'll notice when you can't call the function from a
function taking `const &`.

### this

Inside a member function, `this` is a pointer to the object the function was called on — exactly
the `self` parameter from C. You rarely need to write it: `balance_cents_` means
`this->balance_cents_`. A common use is returning `*this` to allow chaining.

### static members

A `static` data member is shared by all objects of the class (one copy, like a global in the
class's namespace); a `static` member function has no `this`:

```cpp
class Widget {
public:
    Widget() { ++count_; }
    static int count() { return count_; }
private:
    inline static int count_ = 0;   // C++17: can be initialized in the class with inline
};
```

### Header / source split

`rect.h`:

```cpp
#pragma once

class Rect {
public:
    Rect(double w, double h);
    double area() const;
private:
    double w_, h_;
};
```

`rect.cpp`:

```cpp
#include "rect.h"

Rect::Rect(double w, double h) : w_{w}, h_{h} {}

double Rect::area() const { return w_ * h_; }
```

`Rect::` says "this definition belongs to class `Rect`". Short functions are often defined directly
in the class body (implicitly `inline`).

### Designing a class: invariants first

Before writing a class, write down its **invariant** — what's always true about a valid object:
"denominator is never zero and the fraction is in lowest terms"; "the time is 00:00–23:59". Then:
the constructor establishes it (or refuses to create the object), and every public member function
preserves it. That's the whole idea of encapsulation.

## Common mistakes

- Forgetting `const` on getters.
- Assigning members in the constructor body instead of initializing them in the list (and for
  `const` or reference members, assignment doesn't even compile).
- Initializer list order different from declaration order.
- Public data members in a class that has invariants.
- Getters and setters for every field: `set_balance()` destroys the invariant as surely as a public
  field. Offer meaningful operations (`deposit`, `withdraw`) instead.

## Exercises

1. Type in `Account` and add a `transfer_to(Account &other, long cents)` member that returns
   `false` if it can't be done.
2. Write a `Fraction` class with invariant "denominator > 0, lowest terms". Constructor
   `Fraction(long num, long den = 1)` normalizes (reduce with `std::gcd` from `<numeric>`, move the
   sign to the numerator) and throws on zero denominator. Members: `numerator()`, `denominator()`,
   `to_double()`, `plus(const Fraction &)`, `times(const Fraction &)`.
3. Write a `Time` class (hours and minutes, invariant 00:00–23:59) with `add_minutes(int)` that
   wraps around midnight and `std::string to_string() const` producing `"09:05"`.
4. Split `Fraction` into `fraction.h` and `fraction.cpp`.
5. Write a `Counter` class with a `static` count of how many counters exist. (Tomorrow you'll
   decrement it in the destructor.)
6. Try calling a non-`const` member function on a `const` object and read the error.
7. ★ Write a `Stack` class of `int` using a `std::vector<int>` member, with `push`, `pop` (throws
   `std::out_of_range` on empty), `top`, `empty`, `size`. Compare its length with your C stack.

## Check yourself

1. What's the difference between `struct` and `class`?
2. Why use a member initializer list?
3. What does `const` after a member function's parameter list mean?
4. What is `this`?
5. What's an invariant, and who is responsible for it?
6. Why make single-argument constructors `explicit`?

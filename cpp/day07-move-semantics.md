# C++ Day 7 — Move semantics, and week 1 review

**Goal:** understand lvalues, rvalues and moves — how C++ transfers resources instead of copying
them — and write move-aware classes.

## Concepts

### The problem: copies of things about to die

```cpp
std::vector<std::string> make_names();   // returns a vector of 1,000,000 strings

std::vector<std::string> names;
names = make_names();   // copy a million strings, then destroy the originals?
```

The temporary returned by `make_names()` is about to be destroyed anyway. Copying its contents is
wasteful — we should just **steal** its internal pointer. That's a **move**: transfer ownership of
the resources and leave the source empty but valid. For a vector, a move copies three pointers
instead of a million strings.

### lvalues and rvalues

- An **lvalue** has a name and an address you can take: a variable `x`, `v[3]`, `*p`.
- An **rvalue** is a temporary without a name: `42`, `a + b`, `make_names()`,
  `std::string{"hi"}`.

Moving from rvalues is always safe — nobody else can see them. Moving from an lvalue is only safe
if you promise not to use its value again.

### Rvalue references and std::move

`T &&` is an **rvalue reference**: it binds to temporaries. Overloading on it lets a class tell
"copy from something that stays alive" from "move from something that's going away":

```cpp
class Buffer {
public:
    Buffer(const Buffer &other);   // copy: other stays alive, must duplicate
    Buffer(Buffer &&other) noexcept;   // move: other is going away, steal its data
};
```

`std::move(x)` (from `<utility>`) doesn't move anything. It's a cast that says "treat `x` as an
rvalue — I'm done with it", which makes overload resolution pick the move constructor:

```cpp
#include <iostream>
#include <string>
#include <utility>
#include <vector>

int main()
{
    std::string a = "a long string that doesn't fit in the small-string buffer";
    std::string b = a;                // copy: a unchanged
    std::string c = std::move(a);     // move: c takes a's buffer

    std::cout << "a: \"" << a << "\"\n";   // a: "" — valid but unspecified; here empty
    std::cout << "c: \"" << c << "\"\n";

    std::vector<std::string> v;
    std::string line = "some data";
    v.push_back(line);                // copies line
    v.push_back(std::move(line));     // moves line into the vector
    std::cout << v.size() << '\n';    // 2
}
```

After a move, the source is in a **valid but unspecified state**: you may assign a new value to it
or destroy it, but don't rely on its contents.

### Writing move operations

Adding moves to yesterday's rule-of-three `Buffer` makes it rule of five:

```cpp
#include <algorithm>
#include <cstddef>
#include <iostream>
#include <utility>

class Buffer {
public:
    explicit Buffer(std::size_t n) : data_{new int[n]{}}, size_{n} {}
    ~Buffer() { delete[] data_; }

    Buffer(const Buffer &other) : data_{new int[other.size_]}, size_{other.size_}
    {
        std::copy(other.data_, other.data_ + size_, data_);
        std::cout << "copy\n";
    }

    Buffer(Buffer &&other) noexcept                        // steal, leave other empty
        : data_{std::exchange(other.data_, nullptr)},
          size_{std::exchange(other.size_, 0)}
    {
        std::cout << "move\n";
    }

    Buffer &operator=(Buffer other) noexcept               // by value: copy OR move happens here
    {
        std::swap(data_, other.data_);
        std::swap(size_, other.size_);
        return *this;
    }

    std::size_t size() const { return size_; }

private:
    int *data_;
    std::size_t size_;
};

Buffer make_buffer() { return Buffer{1000}; }

int main()
{
    Buffer a{10};
    Buffer b = a;                   // copy
    Buffer c = std::move(a);        // move
    Buffer d = make_buffer();       // neither! copy elision constructs d directly
    b = Buffer{5};                  // neither: the temporary is built directly in the parameter
    std::cout << a.size() << ' ' << c.size() << ' ' << d.size() << ' ' << b.size() << '\n';   // 0 10 1000 5
}
```

- `std::exchange(x, new_value)` sets `x` to `new_value` and returns the old value — perfect for
  "take the pointer and null out the source".
- The moved-from `Buffer` holds `nullptr`, and `delete[] nullptr` is a no-op, so its destructor is
  still safe.
- A single assignment operator taking its parameter **by value** handles both copy and move
  assignment: an lvalue argument is copied into the parameter, a `std::move`d one is moved, and a
  temporary is constructed right in it. Then the parameter is swapped in.

### noexcept moves matter

When `std::vector` grows, it must transfer elements to the new block. If an element type's move
constructor is `noexcept`, it moves them; otherwise it **copies** them (to keep its strong exception
guarantee — day 13). So always mark move operations `noexcept`. The compiler-generated ones are,
when possible.

### When do moves happen automatically?

- Returning a local variable: `return result;` moves (or better, elides) — **never** write
  `return std::move(result);`, it prevents elision.
- Passing a temporary: `f(std::string{"x"})`.
- Otherwise, you ask with `std::move`.

### The rule of zero, again

With the rule of zero, you get correct moves for free: a class whose members are `std::vector`,
`std::string`, `std::unique_ptr` gets compiler-generated move operations that move each member.
But declaring a destructor, or copy operations, **suppresses** the implicit moves — another reason
to not write special members you don't need.

### "Sink" parameters

A function that stores its argument (a constructor, a setter) can take it **by value and move
it in**:

```cpp
class Person {
public:
    explicit Person(std::string name) : name_{std::move(name)} {}
private:
    std::string name_;
};

std::string n = "Ada";
Person p1{n};                 // one copy (into the parameter), then a move
Person p2{"Grace"};           // a temporary string, then a move: no copy at all
Person p3{std::move(n)};      // two moves: no copy
```

That's the "take ownership / keep a copy → `T` by value" row from day 3's table.

## Common mistakes

- Using an object after `std::move`-ing from it.
- `return std::move(local);` (pessimization).
- Forgetting `noexcept` on move operations.
- `std::move` on a `const` object — it silently copies (you can't steal from a `const`).
- Writing a destructor in an otherwise rule-of-zero class, losing implicit moves.

## Exercises

1. Type in the rule-of-five `Buffer`. Before running, predict every line of output.
2. Put `Buffer`s in a `std::vector` with `push_back(Buffer{10})` five times. Count copies and
   moves. Now remove `noexcept` from the move constructor and run again — what changes?
3. Give yesterday's `Noisy` class copy and move constructors that print, and experiment: pass by
   value an lvalue, a `std::move`d lvalue, and a temporary.
4. Write a `Person` class with a sink constructor as above, and add a `set_name(std::string)`
   sink setter.
5. Make your `File` class (day 6) **movable but not copyable**: a move constructor and move
   assignment that transfer the `FILE *` and null out the source; the destructor must skip
   `fclose` on `nullptr`. Return a `File` from a function `File open_log()`.
6. Explain why `const std::string s = "x"; std::string t = std::move(s);` copies.

## Week 1 review

1. Why is `std::format("{}", x)` safer than `printf("%d", x)`? (Day 1)
2. When should a parameter be `T`, `const T &`, `T &`, `T *`? (Days 2–3)
3. What's the difference between `std::vector<int> v(5)` and `v{5}`? (Day 4)
4. What's the danger of `std::string_view` and `std::span`? (Day 4)
5. What does a constructor's initializer list do, and in what order are members initialized?
   (Day 5)
6. What is RAII, and what C pattern does it replace? (Day 6)
7. State the rule of zero. (Days 6–7)
8. What does `std::move` actually do? (Day 7)

**Mini project (★):** write a `Matrix` class (rows × cols of `double`, stored in a
`std::vector<double>`, rule of zero) with `operator()(r, c)` access (day 10 covers operators — try
it: `double &operator()(std::size_t r, std::size_t c)`), `transpose()`, `multiply(const Matrix &)`
that throws on a size mismatch, and a `print()` function. Time multiplying two 300×300 matrices.

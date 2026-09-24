# C++ Day 6 — Classes II: destructors, RAII, and the rule of 0/3/5

**Goal:** understand destructors and RAII — the single most important idea in C++ — and know what
happens when objects are copied.

## Concepts

### Destructors

A **destructor** runs automatically when an object's lifetime ends: when a local variable goes out
of scope, when a heap object is deleted, when the containing object is destroyed. It's named `~`
plus the class name:

```cpp
#include <iostream>
#include <string>
#include <utility>

class Noisy {
public:
    explicit Noisy(std::string name) : name_{std::move(name)} { std::cout << "construct " << name_ << '\n'; }
    ~Noisy() { std::cout << "destroy " << name_ << '\n'; }
private:
    std::string name_;
};

int main()
{
    Noisy a{"a"};
    {
        Noisy b{"b"};
        Noisy c{"c"};
    }                           // c destroyed, then b (reverse order of construction)
    std::cout << "end of main\n";
}                               // a destroyed
```

Output:

```text
construct a
construct b
construct c
destroy c
destroy b
end of main
destroy a
```

It happens on **every** exit path: normal end of scope, `return`, `break`, and exceptions (day 13).

### RAII: Resource Acquisition Is Initialization

The idea: **tie every resource to an object's lifetime**. The constructor acquires it; the
destructor releases it. Then releasing can't be forgotten — the compiler does it.

Remember C day 20's `goto cleanup` for files and buffers? With RAII:

```cpp
#include <cstdio>
#include <stdexcept>
#include <string>

class File {
public:
    File(const std::string &path, const char *mode)
        : f_{std::fopen(path.c_str(), mode)}
    {
        if (f_ == nullptr) throw std::runtime_error{"cannot open " + path};
    }
    ~File() { std::fclose(f_); }                  // always runs

    File(const File &) = delete;                  // copying a File makes no sense: see below
    File &operator=(const File &) = delete;

    std::FILE *get() const { return f_; }

private:
    std::FILE *f_;
};

void copy_prefix(const std::string &src, const std::string &dst)
{
    File in{src, "rb"};                           // if this throws, nothing to clean up
    File out{dst, "wb"};                          // if this throws, `in` is closed automatically
    char buf[64];
    std::size_t n = std::fread(buf, 1, sizeof buf, in.get());
    std::fwrite(buf, 1, n, out.get());
}                                                 // both files closed here, in reverse order

int main(int argc, char *argv[])
{
    if (argc == 3) copy_prefix(argv[1], argv[2]);
}
```

No `goto`, no cleanup section, no way to leak. `std::string`, `std::vector`, `std::fstream`,
smart pointers (day 8) and locks (day 27) are all RAII classes. In modern C++ you almost never
write `delete`, `free` or `fclose` in ordinary code — a class does it.

### Copying: what happens by default

Copying an object copies each member ("memberwise copy"). For members like `std::string` and
`std::vector`, that's a proper deep copy — they know how to copy themselves. But for a **raw
pointer member that owns memory**, it copies the pointer: the same shallow-copy trap as C day 15.

```cpp
class Buffer {                    // BROKEN: owns memory, but copies shallowly
public:
    explicit Buffer(std::size_t n) : data_{new int[n]{}}, size_{n} {}
    ~Buffer() { delete[] data_; }
private:
    int *data_;
    std::size_t size_;
};

Buffer a{10};
Buffer b = a;     // b.data_ == a.data_
                  // at the end of scope both destructors delete[] the same memory: double free
```

(`new int[n]{}` allocates n zeroed ints on the heap; `delete[]` frees them. They're C++'s
`calloc`/`free`, and after today you'll rarely use them directly.)

### The rule of three / five / zero

If a class manages a resource directly, the compiler-generated copy operations are wrong, and you
must write (or delete) them. The special member functions are:

| Function | Signature | Called when |
|---|---|---|
| destructor | `~T()` | lifetime ends |
| copy constructor | `T(const T &other)` | `T b = a;`, pass/return by value |
| copy assignment | `T &operator=(const T &other)` | `b = a;` for an existing `b` |
| move constructor | `T(T &&other) noexcept` | tomorrow |
| move assignment | `T &operator=(T &&other) noexcept` | tomorrow |

- **Rule of three:** if you write any of destructor, copy constructor, copy assignment, you
  probably need all three.
- **Rule of five:** with move semantics (tomorrow), add the two move operations.
- **Rule of zero:** best of all — **don't manage resources directly in your classes**. Use members
  that already manage themselves (`std::vector`, `std::string`, `std::unique_ptr`), and write none
  of the five. Almost all your classes should follow the rule of zero.

Here's `Buffer` done right, the hard way (rule of three) — you should write this once to understand
it:

```cpp
#include <algorithm>
#include <cstddef>
#include <iostream>
#include <utility>

class Buffer {
public:
    explicit Buffer(std::size_t n) : data_{new int[n]{}}, size_{n} {}

    ~Buffer() { delete[] data_; }

    Buffer(const Buffer &other)                        // deep copy
        : data_{new int[other.size_]}, size_{other.size_}
    {
        std::copy(other.data_, other.data_ + size_, data_);
    }

    Buffer &operator=(const Buffer &other)             // copy-and-swap: safe even for a = a
    {
        Buffer tmp{other};                             // copy first (may throw: *this untouched)
        std::swap(data_, tmp.data_);                   // then swap in the new data
        std::swap(size_, tmp.size_);
        return *this;                                  // tmp's destructor frees our old data
    }

    int &operator[](std::size_t i) { return data_[i]; }
    std::size_t size() const { return size_; }

private:
    int *data_;
    std::size_t size_;
};

int main()
{
    Buffer a{3};
    a[0] = 42;
    Buffer b = a;          // copy constructor
    b[0] = 7;
    Buffer c{1};
    c = a;                 // copy assignment
    std::cout << a[0] << ' ' << b[0] << ' ' << c[0] << ' ' << c.size() << '\n';   // 42 7 42 3
}
```

And the rule-of-zero version, which is what you'd actually write:

```cpp
class Buffer {
public:
    explicit Buffer(std::size_t n) : data_(n) {}
    int &operator[](std::size_t i) { return data_[i]; }
    std::size_t size() const { return data_.size(); }
private:
    std::vector<int> data_;     // copies, moves and frees itself correctly
};
```

### = default and = delete

- `= default` asks for the compiler-generated version explicitly.
- `= delete` forbids an operation. Deleting the copy operations makes a class **non-copyable**
  (like `File` above — two objects closing the same `FILE *` would be a bug). Tomorrow you'll make
  it **movable** instead.

## Common mistakes

- Owning raw pointers in classes without the rule of three/five.
- `delete` vs `delete[]` mismatch (`new[]` needs `delete[]`).
- Writing a destructor "just in case" — it disables the implicit move operations (day 7).
  Follow the rule of zero.
- Self-assignment bugs in hand-written `operator=` (copy-and-swap avoids them).
- Forgetting that pass-by-value calls the copy constructor.

## Exercises

1. Type in `Noisy`. Predict and check the output when you: pass a `Noisy` by value to a function;
   by `const &`; store three in a `std::vector<Noisy>` with `emplace_back` (watch what happens when
   the vector grows — you'll understand it tomorrow).
2. Type in the broken `Buffer`, compile with `-fsanitize=address` and see the double free. Then fix
   it with the rule of three, and finally with the rule of zero.
3. Write an RAII class `Timer` that records the start time in its constructor
   (`std::chrono::steady_clock::now()`) and prints the elapsed milliseconds in its destructor. Use
   it to time a block: `{ Timer t{"sorting"}; std::sort(...); }`.
4. Finish `File`: add `write_line(std::string_view)` and `std::optional<std::string> read_line()`
   (look up `std::optional` — or return an empty string at EOF for now).
5. Write an RAII class `IndentGuard` that increases a global indentation level in its constructor
   and decreases it in its destructor; use it in a recursive function that prints a tree of calls.
6. Make `Counter` from yesterday decrement its static count in the destructor, and print the count
   at several points in a program with nested scopes.
7. ★ Write a `String` class that owns a `char *` (rule of three, with `new[]`/`delete[]`), with a
   constructor from `const char *`, `size()`, `c_str()`, and `operator+=`. Test copies with ASan.
   Then appreciate `std::string`.

## Check yourself

1. When exactly does a destructor run?
2. What is RAII, in one sentence?
3. What does the default copy constructor do with a pointer member?
4. State the rule of three and the rule of zero.
5. Why is `File`'s copy constructor deleted?

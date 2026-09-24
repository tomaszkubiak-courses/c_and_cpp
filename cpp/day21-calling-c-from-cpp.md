# C++ Day 21 — Calling C code from C++ (and C++ from C)

**Goal:** use an existing C library — including your own code from the C course — in a C++
program: headers that work in both languages, building C and C++ together with CMake, RAII wrappers
around C APIs, and callbacks across the boundary.

## Concepts

### Why this needs care: name mangling

A C compiler emits a function `int rb_push(RingBuffer *, int)` under the symbol name `rb_push`.
A C++ compiler must support overloading (day 3), so it encodes the parameter types into the symbol:
something like `_Z7rb_pushP10RingBufferi`. If C++ code calls a function compiled by a C compiler
using a plain declaration, the linker looks for the mangled name, doesn't find it, and you get:

```text
undefined reference to `rb_push(RingBuffer*, int)'
```

See it yourself: compile a C file and a C++ file with the same function, and compare `nm file.o`.
Pipe through `c++filt` to decode mangled names.

### extern "C"

`extern "C"` tells the C++ compiler: "these functions use C linkage — don't mangle their names":

```cpp
extern "C" {
    int rb_push(RingBuffer *rb, int value);
}
```

Functions declared `extern "C"` can't be overloaded (C has no overloading), but otherwise behave
normally in C++.

### A header that works for both languages

A C compiler doesn't understand `extern "C"`. The standard trick uses the `__cplusplus` macro, which
only C++ compilers define:

```c
#ifndef RINGBUF_H
#define RINGBUF_H

#include <stddef.h>

#ifdef __cplusplus
extern "C" {
#endif

/* ... ordinary C declarations ... */

#ifdef __cplusplus
}
#endif

#endif
```

Every C library meant to be usable from C++ does this — look inside `/usr/include/stdio.h`.
The standard C headers are also available as `<cstdio>`, `<cstring>`, `<cstdlib>`, etc., which put
the names in namespace `std` (`std::printf`, `std::strlen`). Prefer those in C++ code.

### Worked example: a C ring buffer library used from C++

This builds a complete project: a C library compiled **as C**, and a C++ program using it through an
RAII wrapper.

```text
ringbuf_demo/
├── CMakeLists.txt
├── clib/
│   ├── ringbuf.h
│   └── ringbuf.c
└── app/
    └── main.cpp
```

`clib/ringbuf.h` — an opaque C type (C day 27) with create/destroy and a callback:

```c
#ifndef RINGBUF_H
#define RINGBUF_H

#include <stddef.h>

#ifdef __cplusplus
extern "C" {
#endif

typedef struct RingBuffer RingBuffer;

RingBuffer *rb_create(size_t capacity);             /* NULL on failure; free with rb_destroy */
void        rb_destroy(RingBuffer *rb);
int         rb_push(RingBuffer *rb, int value);     /* 0 on success, -1 if full */
int         rb_pop(RingBuffer *rb, int *out);       /* 0 on success, -1 if empty */
size_t      rb_size(const RingBuffer *rb);

/* Calls fn(value, user) for each element, oldest first. */
void rb_foreach(const RingBuffer *rb, void (*fn)(int value, void *user), void *user);

#ifdef __cplusplus
}
#endif

#endif
```

`clib/ringbuf.c` — plain C17:

```c
#include "ringbuf.h"

#include <stdlib.h>

struct RingBuffer {
    size_t head, count, capacity;
    int items[];                    /* flexible array member (C99) — not valid C++! */
};

RingBuffer *rb_create(size_t capacity)
{
    RingBuffer *rb = malloc(sizeof *rb + capacity * sizeof rb->items[0]);
    if (rb == NULL) return NULL;
    rb->head = rb->count = 0;
    rb->capacity = capacity;
    return rb;
}

void rb_destroy(RingBuffer *rb) { free(rb); }

int rb_push(RingBuffer *rb, int value)
{
    if (rb->count == rb->capacity) return -1;
    rb->items[(rb->head + rb->count) % rb->capacity] = value;
    rb->count++;
    return 0;
}

int rb_pop(RingBuffer *rb, int *out)
{
    if (rb->count == 0) return -1;
    *out = rb->items[rb->head];
    rb->head = (rb->head + 1) % rb->capacity;
    rb->count--;
    return 0;
}

size_t rb_size(const RingBuffer *rb) { return rb->count; }

void rb_foreach(const RingBuffer *rb, void (*fn)(int value, void *user), void *user)
{
    for (size_t i = 0; i < rb->count; i++) {
        fn(rb->items[(rb->head + i) % rb->capacity], user);
    }
}
```

The file uses a flexible array member, which is valid C but not C++ — a reminder that the `.c` file
**must be compiled by a C compiler**, and only the header needs to be C++-compatible.

`app/main.cpp` — a C++ wrapper class with RAII, exceptions, and lambdas as callbacks:

```cpp
#include <iostream>
#include <memory>
#include <optional>
#include <stdexcept>
#include <type_traits>
#include <vector>

#include "ringbuf.h"

class IntRing {
public:
    explicit IntRing(std::size_t capacity) : rb_{rb_create(capacity)}
    {
        if (!rb_) throw std::bad_alloc{};
    }

    bool push(int value) { return rb_push(rb_.get(), value) == 0; }

    std::optional<int> pop()
    {
        int out;
        if (rb_pop(rb_.get(), &out) != 0) return std::nullopt;
        return out;
    }

    std::size_t size() const { return rb_size(rb_.get()); }

    // Accepts any callable, including capturing lambdas.
    template <typename F>
    void for_each(F &&f) const
    {
        // A C function pointer can't carry state, so pass the callable through void *user
        // and use a captureless lambda (which converts to a function pointer) as a trampoline.
        auto trampoline = [](int value, void *user) {
            (*static_cast<std::remove_reference_t<F> *>(user))(value);
        };
        rb_foreach(rb_.get(), trampoline, &f);
    }

private:
    struct Deleter {
        void operator()(RingBuffer *rb) const noexcept { rb_destroy(rb); }
    };
    std::unique_ptr<RingBuffer, Deleter> rb_;   // rule of zero: movable, not copyable, auto-freed
};

int main()
{
    IntRing ring{4};
    for (int i = 1; i <= 5; ++i) {
        if (!ring.push(i * 10)) std::cout << "full, dropped " << i * 10 << '\n';
    }
    ring.pop();                                 // drop the oldest (10)
    ring.push(99);

    std::vector<int> seen;
    int sum = 0;
    ring.for_each([&](int v) { seen.push_back(v); sum += v; });   // capturing lambda via trampoline

    for (int v : seen) std::cout << v << ' ';
    std::cout << "| sum " << sum << ", size " << ring.size() << '\n';   // 20 30 40 99 | sum 189, size 4
}
```

`CMakeLists.txt` — the key line is `LANGUAGES C CXX`: CMake compiles `.c` files with the C compiler
and `.cpp` files with the C++ compiler:

```cmake
cmake_minimum_required(VERSION 3.20)
project(ringbuf_demo LANGUAGES C CXX)

add_library(ringbuf STATIC clib/ringbuf.c)
target_include_directories(ringbuf PUBLIC clib)
target_compile_features(ringbuf PUBLIC c_std_17)

add_executable(app app/main.cpp)
target_link_libraries(app PRIVATE ringbuf)
target_compile_features(app PRIVATE cxx_std_20)
```

```bash
cmake -S . -B build && cmake --build build && ./build/app
```

The pieces that made it work:

1. `extern "C"` in the header, guarded by `__cplusplus`.
2. The C file compiled as C (`project(... LANGUAGES C CXX)`, `.c` extension).
3. An RAII class owning the C handle via `std::unique_ptr` with a custom deleter (day 8).
4. Errors translated at the boundary: C return codes become exceptions or `std::optional`.
5. Callbacks: the C API takes a function pointer plus `void *user`; the wrapper passes the lambda's
   address as `user` and a captureless trampoline as the function pointer.

### Rules at the boundary

- **Exceptions must not cross into C code.** If a C library calls your callback and the callback
  throws, the exception unwinds through C stack frames that know nothing about it — undefined
  behavior. Catch everything inside callbacks, or mark them `noexcept` so the program terminates
  cleanly instead.
- **Memory:** free with the library's own function (`rb_destroy`), never with `delete`. If a C
  function returns `malloc`'d memory, free it with `std::free` (a `unique_ptr` with
  `decltype(&std::free)` as the deleter does it).
- **Types:** use only C-compatible types in the shared header: no references, classes, templates,
  `std::string`, overloads, or default arguments. `bool` is fine (C has `<stdbool.h>`; C23 has it
  built in).
- **Struct layout:** a plain struct with only C types has the same layout in both languages, so you
  can share it (this is what "standard layout" means).

### Code that compiles as C but differs in C++

If you ever include C *implementation* code in a C++ file (e.g. a header-only C library), these
differences bite:

| C | C++ |
|---|---|
| `void *` converts implicitly: `int *p = malloc(n);` | needs a cast: `static_cast<int *>(std::malloc(n))` |
| `'a'` has type `int` (`sizeof 'a'` is 4) | `'a'` has type `char` (`sizeof 'a'` is 1) |
| Variable-length arrays `int a[n];` (optional) | not allowed |
| Flexible array members `int items[];` | not allowed (compilers accept as an extension) |
| Designated initializers in any order, nested, array ones | C++20: in declaration order only, no array designators |
| `restrict` keyword | doesn't exist |
| `struct S` needs `struct` or a typedef | `S` is a type name directly |
| `f()` means unspecified params (before C23) | `f()` means no params |
| `new`, `class`, `this`, `template` are valid names | keywords |
| `_Generic`, compound literals `(int[]){1,2}` | not available |

### Calling C++ from C

The other direction: write `extern "C"` wrapper functions in a `.cpp` file that expose an opaque
handle and C types only:

```cpp
// counter_api.h  (C-compatible)
#ifdef __cplusplus
extern "C" {
#endif
typedef struct Counter Counter;
Counter *counter_new(void);
void counter_add(Counter *c, const char *word);
int counter_get(const Counter *c, const char *word);
void counter_free(Counter *c);
#ifdef __cplusplus
}
#endif
```

```cpp
// counter_api.cpp
#include "counter_api.h"

#include <string>
#include <unordered_map>

struct Counter {                                  // a C++ type behind the C handle
    std::unordered_map<std::string, int> counts;
};

extern "C" {
Counter *counter_new(void) { try { return new Counter{}; } catch (...) { return nullptr; } }
void counter_add(Counter *c, const char *word) { try { ++c->counts[word]; } catch (...) {} }
int counter_get(const Counter *c, const char *word)
{
    auto it = c->counts.find(word);
    return it == c->counts.end() ? 0 : it->second;
}
void counter_free(Counter *c) { delete c; }
}
```

Every function catches exceptions so none escape into C. The executable must be linked by the C++
compiler (or with the C++ runtime library), which CMake does automatically when any source in the
link is C++.

## Common mistakes

- Missing `extern "C"`: "undefined reference" to a mangled name.
- Compiling the `.c` file as C++ (renaming it `.cpp`, or MSVC's `/TP`): it may not compile, or
  compiles with different semantics — and then the C++ side's `extern "C"` declarations don't match.
- Letting exceptions escape through C callbacks.
- Freeing library memory with `delete`/`free` instead of the library's own function.
- C++ types (references, `std::string`) in a header meant for C.

## Exercises

1. Build the ring buffer example with CMake. Then remove the `extern "C"` block from the header,
   rebuild, and read the linker error; run `nm build/libringbuf.a` and `nm build/CMakeFiles/app.dir/app/main.cpp.o | c++filt`
   to see the two different symbol names.
2. Add `int rb_peek(const RingBuffer *rb, int *out)` to the C library and a `std::optional<int> peek() const`
   to the wrapper.
3. **Wrap your C day 25 hash map**: copy `hashmap.h`/`hashmap.c` into a new project, add the
   `extern "C"` guards, and write a C++ class `StringIntMap` with RAII, `put`, `std::optional<int> get`,
   `erase`, and a `for_each` taking a lambda (add `hm_foreach` from C day 25, exercise 2, if you
   haven't). Use it to count words, and compare the speed with `std::unordered_map`.
4. Make `IntRing` iterable with a range-based `for`... without changing the C library: copy the
   elements into a `std::vector` in a `to_vector()` member. ★ Then think about what a real iterator
   would need.
5. Write a callback that throws inside `for_each`, run it, and observe what happens. Then make the
   trampoline catch exceptions and store them (`std::exception_ptr`), and rethrow after `rb_foreach`
   returns — the robust pattern.
6. Implement the "C++ from C" `Counter` example and call it from a `main.c` compiled as C.
7. Use a real C library from C++: `sqlite3` (install `libsqlite3-dev`, `find_package(SQLite3)`,
   `SQLite::SQLite3` target). Wrap `sqlite3 *` in a `unique_ptr` with `sqlite3_close` as the deleter,
   create a table, insert rows, query them with a callback lambda.

## Check yourself

1. What's name mangling, and why does C++ need it?
2. What does `extern "C"` change?
3. Why wrap the `extern "C" {` in `#ifdef __cplusplus`?
4. Why must `ringbuf.c` be compiled as C?
5. How can a capturing lambda be used as a C callback?
6. Why must exceptions not propagate through C code?

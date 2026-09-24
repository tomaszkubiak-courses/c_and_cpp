# C++ Day 8 — Pointers in C++: new/delete and smart pointers

**Goal:** know how raw pointers, `new`/`delete` and smart pointers relate; use `unique_ptr` and
`shared_ptr` to express ownership; and know when a raw pointer or reference is still the right
choice.

## Concepts

### Everything from C still applies

Addresses, `&`, `*`, `->`, pointer arithmetic, `nullptr` instead of `NULL` — all the same. What
changes in C++ is **who frees heap memory**: ideally, never you.

### new and delete (and why you'll rarely write them)

`new` allocates heap memory **and runs a constructor**; `delete` runs the destructor **and frees**:

```cpp
auto *p = new std::string{"hello"};    // like malloc + constructor
delete p;                              // like destructor + free
int *a = new int[10]{};                // array form
delete[] a;                            // must match: new[] -> delete[]
```

They have every problem `malloc`/`free` had — leaks, double deletes, dangling pointers — plus a new
one: if an exception is thrown between `new` and `delete`, the `delete` never runs. **In modern
C++, don't write `new`/`delete` in application code.** Use containers, or smart pointers.

### std::unique_ptr: exactly one owner

`std::unique_ptr<T>` (from `<memory>`) is an RAII class holding a pointer. Its destructor deletes
the object. It **can't be copied** (two owners would delete twice) but **can be moved** (ownership
transfers):

```cpp
#include <iostream>
#include <memory>
#include <string>
#include <utility>

struct Monster {
    std::string name;
    int hp;
    Monster(std::string n, int h) : name{std::move(n)}, hp{h} { std::cout << "spawn " << name << '\n'; }
    ~Monster() { std::cout << "despawn " << name << '\n'; }
};

std::unique_ptr<Monster> spawn_boss()
{
    return std::make_unique<Monster>("Dragon", 500);   // allocate + construct
}

int main()
{
    auto boss = spawn_boss();                 // ownership moves out of the function
    boss->hp -= 50;                           // use like a pointer
    std::cout << boss->name << " has " << boss->hp << " hp\n";

    // auto copy = boss;                      // error: unique_ptr is not copyable
    auto new_owner = std::move(boss);         // ownership moves; boss is now nullptr
    std::cout << (boss == nullptr) << '\n';   // 1

    {
        auto minion = std::make_unique<Monster>("Goblin", 20);
    }                                         // "despawn Goblin" — end of scope
    std::cout << "end of main\n";
}                                             // "despawn Dragon"
```

- Create with **`std::make_unique<T>(constructor args...)`**, never with `new`.
- A `unique_ptr` has **no overhead** compared to a raw pointer — same size, same speed.
- `.get()` returns the raw pointer (without giving up ownership); `.reset()` deletes the object
  now; `.release()` gives up ownership and returns the raw pointer (rarely needed).
- `std::unique_ptr<T[]>` exists for arrays, but `std::vector<T>` is almost always better.

**This is the C ownership rule from C day 12 enforced by the compiler.** In C you wrote "the
caller must free the result" in a comment; `std::unique_ptr<Monster> spawn_boss()` says it in the
type, and frees it automatically.

### When do you need heap objects at all?

Less often than you think. Local variables and members are best; containers hold their elements on
the heap for you. Use `unique_ptr` when:

1. an object must be **polymorphic** — you hold a `Base` pointer to a `Derived` object (day 11);
2. an object is **big** and must move around cheaply, or is non-movable;
3. the object is **optional** or created later, and `std::optional` doesn't fit;
4. you're building **linked structures** (lists, trees) with ownership.

### std::shared_ptr: shared ownership

Sometimes there's no single owner: several parts of the program use an object, and it should die
when the last one lets go. `std::shared_ptr<T>` keeps a **reference count**; copying increments
it, destruction decrements it, and the object is deleted when it reaches zero.

```cpp
#include <iostream>
#include <memory>
#include <vector>

struct Texture {
    ~Texture() { std::cout << "texture freed\n"; }
};

int main()
{
    std::vector<std::shared_ptr<Texture>> sprites;
    {
        auto tex = std::make_shared<Texture>();
        sprites.push_back(tex);              // count 2
        sprites.push_back(tex);              // count 3
        std::cout << "count " << tex.use_count() << '\n';   // 3
    }                                        // tex gone: count 2
    sprites.pop_back();                      // count 1
    std::cout << "one left\n";
    sprites.clear();                         // count 0: "texture freed"
    std::cout << "done\n";
}
```

`shared_ptr` has costs: a separate control block, atomic count updates (it's thread-safe), and —
more importantly — it makes ownership vague. **Prefer `unique_ptr`**; reach for `shared_ptr` only
when ownership is genuinely shared.

### std::weak_ptr: breaking cycles

If A owns a `shared_ptr` to B and B owns one to A, the counts never reach zero: a leak. A
`std::weak_ptr` observes a `shared_ptr`-managed object without owning it. To use it, `lock()` it —
you get a `shared_ptr` that's empty if the object is already gone:

```cpp
std::weak_ptr<Texture> watcher = some_shared_ptr;
if (auto sp = watcher.lock()) {
    // object still alive, sp keeps it alive during this block
}
```

Typical use: parent owns children (`shared_ptr` or `unique_ptr`), children point back to the parent
with a `weak_ptr` or raw pointer.

### Raw pointers and references still have a job: non-owning access

Smart pointers are about **ownership**. For simply *using* an object that someone else owns, pass a
reference or a raw pointer:

```cpp
void heal(Monster &m, int amount);          // must exist: reference
void target(const Monster *m);              // may be null: pointer
void keep(std::unique_ptr<Monster> m);      // takes ownership: unique_ptr by value
void share(std::shared_ptr<Monster> m);     // shares ownership: shared_ptr by value
```

Don't pass `const std::unique_ptr<T> &` or `std::shared_ptr<T>` to a function that just uses the
object — pass `T &` or `T *` (with `.get()` or `*ptr` at the call). A **raw pointer in modern C++
means "non-owning"**. If you see `delete p` on a raw pointer in modern code, that's a smell.

### Custom deleters: RAII for C resources

`unique_ptr` can call any cleanup function — a one-line RAII wrapper for C APIs:

```cpp
#include <cstdio>
#include <memory>

struct FileCloser {
    void operator()(std::FILE *f) const { if (f) std::fclose(f); }
};
using FilePtr = std::unique_ptr<std::FILE, FileCloser>;

int main()
{
    FilePtr f{std::fopen("out.txt", "w")};
    if (!f) return 1;
    std::fputs("closed automatically\n", f.get());
}
```

That's day 6's `File` class in three lines. You'll use this on C++ day 21 to wrap your own C
library.

## Common mistakes

- `new` without a smart pointer; `delete` anywhere in application code.
- `std::unique_ptr<T>(new T)` instead of `std::make_unique<T>()`.
- Two smart pointers created from the same raw pointer (`std::shared_ptr<T> a{p}, b{p};`) — double
  delete.
- `shared_ptr` everywhere "to be safe" — it hides ownership and costs performance.
- Cycles of `shared_ptr`s.
- Using a `unique_ptr` after moving from it (it's `nullptr`).

## Exercises

1. Type in the `Monster` program and predict every line of output first.
2. Write a `Node` for a singly linked list with `int value; std::unique_ptr<Node> next;`. Implement
   `push_front`, `print` and `size` for a list represented by `std::unique_ptr<Node> head`. Notice
   there's no `free_list` — why? (★ With a very long list — 10⁶ nodes — the recursive destruction
   may overflow the stack. Write an iterative `clear()` that fixes it.)
3. Write `std::unique_ptr<int[]> make_squares(std::size_t n)` and then rewrite it returning
   `std::vector<int>`. Which is nicer?
4. Write a `Library` that owns `Book`s in a `std::vector<std::unique_ptr<Book>>`, and a `Member`
   class that holds a non-owning `std::vector<const Book *>` of borrowed books. What happens if the
   library removes a borrowed book? (Think about it — the bug here is the subject of tomorrow.)
5. Demonstrate a `shared_ptr` cycle: two `struct Person { std::shared_ptr<Person> friend_; ~Person(){...} }`
   pointing at each other — the destructors never print. Fix it with `weak_ptr`.
6. Wrap `std::FILE *` with a custom deleter (above), and write `FilePtr open_or_throw(const char *path, const char *mode)`.
7. ★ Wrap a C `malloc`'d buffer: `std::unique_ptr<char, decltype(&std::free)> p{static_cast<char *>(std::malloc(100)), &std::free};`.
   Why might you need this when calling C libraries that allocate?

## Check yourself

1. What's wrong with `new`/`delete` in application code?
2. Why can't a `unique_ptr` be copied, and how do you transfer it?
3. When is `shared_ptr` the right choice, and what does it cost?
4. What problem does `weak_ptr` solve?
5. In a modern C++ function signature, what does a raw `T *` parameter mean?

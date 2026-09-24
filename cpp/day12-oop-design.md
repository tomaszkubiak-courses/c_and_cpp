# C++ Day 12 — OOP design: composition, interfaces, and alternatives to inheritance

**Goal:** design classes that are easy to change: prefer composition, program to interfaces,
inject dependencies, and know when `std::variant` beats a class hierarchy.

## Concepts

### Composition over inheritance

Inheritance is the tightest coupling in C++: a derived class depends on every detail of its base.
Use it for genuine "is a" relationships with polymorphic behavior. For "has a" or "uses a", make
the other object a **member**:

```cpp
class Engine {
public:
    void start() { running_ = true; }
    bool running() const { return running_; }
private:
    bool running_ = false;
};

class Car {                         // a Car HAS an Engine — not "Car : public Engine"
public:
    void start() { engine_.start(); }
private:
    Engine engine_;
};
```

A test: if a derived class would inherit functions that make no sense for it (a `Stack` inheriting
`std::vector`'s `insert`-in-the-middle), it's not "is a". Wrap instead.

### Interfaces: abstract classes with only pure virtuals

```cpp
class Logger {
public:
    virtual ~Logger() = default;
    virtual void log(std::string_view message) = 0;
};
```

Code that depends on `Logger &` works with a console logger, a file logger, or a test logger that
records messages — without knowing which. That's C day 27's `Logger` with the function pointer and
`void *ctx`, now type-safe.

### Dependency injection

Pass collaborators in, rather than creating them inside. Then tests can pass fakes:

```cpp
#include <iostream>
#include <string>
#include <string_view>
#include <vector>

class Logger {
public:
    virtual ~Logger() = default;
    virtual void log(std::string_view message) = 0;
};

class ConsoleLogger : public Logger {
public:
    void log(std::string_view message) override { std::cout << "[log] " << message << '\n'; }
};

class MemoryLogger : public Logger {                // for tests
public:
    void log(std::string_view message) override { lines.emplace_back(message); }
    std::vector<std::string> lines;
};

class OrderService {
public:
    explicit OrderService(Logger &log) : log_{log} {}   // injected, not created here
    bool place(int item_id, int quantity)
    {
        if (quantity <= 0) {
            log_.log("rejected order: bad quantity");
            return false;
        }
        log_.log("order placed for item " + std::to_string(item_id));
        return true;
    }
private:
    Logger &log_;                                  // non-owning: the logger must outlive the service
};

int main()
{
    ConsoleLogger console;
    OrderService service{console};
    service.place(42, 3);

    MemoryLogger mem;                              // in a test:
    OrderService tested{mem};
    tested.place(1, 0);
    std::cout << (mem.lines.size() == 1 && mem.lines[0].find("rejected") != std::string::npos) << '\n';   // 1
}
```

### SOLID, briefly

Five guidelines often quoted for OO design. Treat them as questions to ask, not laws:

1. **Single responsibility** — does this class have one reason to change? (An `Invoice` that also
   formats PDFs and sends email has three.)
2. **Open/closed** — can you add a new kind of thing (a new shape) without editing existing code?
3. **Liskov substitution** — can every derived object be used wherever the base is expected,
   without surprises? (The classic violation: `Square` deriving from a *mutable* `Rectangle` —
   `set_width` breaks the square's invariant. Notice that yesterday's `Square` worked only
   because `Rect` was immutable.)
4. **Interface segregation** — small, focused interfaces instead of one huge one.
5. **Dependency inversion** — depend on abstractions (`Logger &`), not concrete classes
   (`ConsoleLogger`).

### Closed hierarchies: std::variant

When the set of types is **fixed** and known up front, a `std::variant` (from `<variant>`) is often
simpler than inheritance: no heap allocation, no pointers, value semantics. It's C day 16's tagged
union, type-safe:

```cpp
#include <iostream>
#include <numbers>
#include <variant>
#include <vector>

struct Circle { double r; };
struct Rect { double w, h; };
struct Triangle { double base, height; };

using Shape = std::variant<Circle, Rect, Triangle>;   // holds exactly one of these

template <class... Fs> struct overloaded : Fs... { using Fs::operator()...; };   // a common helper

double area(const Shape &s)
{
    return std::visit(overloaded{
        [](const Circle &c) { return std::numbers::pi * c.r * c.r; },
        [](const Rect &r) { return r.w * r.h; },
        [](const Triangle &t) { return 0.5 * t.base * t.height; },
    }, s);                                            // forgetting a type is a compile error
}

int main()
{
    std::vector<Shape> shapes = {Circle{1}, Rect{2, 3}, Triangle{4, 5}};   // stored by value
    for (const auto &s : shapes) std::cout << area(s) << '\n';
    std::cout << std::holds_alternative<Rect>(shapes[1]) << '\n';          // 1
}
```

(The `overloaded` helper merges several lambdas into one object with several `operator()`s — you'll
understand the template syntax on day 16. Just copy it for now.)

The trade-off is the one from C day 27, restated:

| | Virtual functions | `std::variant` + `std::visit` |
|---|---|---|
| Add a new **type** | easy: new class, nothing else changes | edit the variant and every visitor |
| Add a new **operation** | edit every class | easy: one new function |
| Storage | heap, via pointers | inline, by value |
| Set of types | open (plugins, libraries) | closed, known at compile time |

## Common mistakes

- Inheriting to reuse code rather than to express "is a".
- God classes: one class that does everything.
- Creating dependencies inside constructors (`logger_ = std::make_unique<FileLogger>("x.log")`),
  which makes testing hard.
- Deep hierarchies. Two levels is usually plenty.
- A `Logger &` member whose logger dies first (day 9!).

## Exercises

1. Type in the `OrderService` example. Add a `FileLogger` using `std::ofstream`, and a
   `TimestampLogger` that **wraps another `Logger &`** and prefixes the time (this is the
   *decorator* pattern — composition + interface).
2. Refactor this design, explaining each change:

   ```cpp
   class Report {
   public:
       void load_from_database(const std::string &conn);
       void compute_statistics();
       void render_html(const std::string &path);
       void email_to(const std::string &address);
   };
   ```

3. Write the shapes example with `std::variant` and add a `perimeter` visitor. Then add a new shape
   type and see what the compiler tells you.
4. Model a small **bank**: `Account` interface with `deposit`/`withdraw`/`balance`, implementations
   `CheckingAccount` (overdraft limit) and `SavingsAccount` (no overdraft, monthly interest), and a
   `Bank` owning a `std::vector<std::unique_ptr<Account>>`. Write a function that applies monthly
   processing to all accounts.
5. Model a JSON value with `std::variant<std::nullptr_t, bool, double, std::string>` (arrays and
   objects need recursion — ★ look up how to do it with `std::vector<Value>`), and write a
   `to_string` visitor.
6. ★ Design (on paper, with class names and responsibilities) a text adventure game: rooms, items,
   player, commands. Which relationships are "is a", which "has a"? Where do interfaces help?

## Check yourself

1. When should you use inheritance, and when composition?
2. What's dependency injection, and why does it make testing easier?
3. Why is "Square derives from a mutable Rectangle" a Liskov violation?
4. When is `std::variant` a better fit than a class hierarchy?

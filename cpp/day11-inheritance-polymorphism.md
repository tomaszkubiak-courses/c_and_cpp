# C++ Day 11 — Inheritance and polymorphism

**Goal:** write class hierarchies with virtual functions, understand what the compiler generates,
and avoid slicing and the missing-virtual-destructor bug.

## Concepts

### Inheritance: "is a"

```cpp
class Animal {
public:
    explicit Animal(std::string name) : name_{std::move(name)} {}
    const std::string &name() const { return name_; }
private:
    std::string name_;
};

class Dog : public Animal {              // a Dog is an Animal
public:
    explicit Dog(std::string name) : Animal{std::move(name)} {}   // construct the base part first
    void fetch() const { std::cout << name() << " fetches\n"; }
};
```

A `Dog` object contains an `Animal` sub-object (exactly like C day 27's `Circle` embedding a
`Shape` as its first member), and gets all its public members. Use `public` inheritance; the other
kinds are rare.

Access: `public` members are visible to everyone, `private` only to the class itself (not to
derived classes!), `protected` to the class and its derived classes. Prefer private data plus
protected or public functions.

### Virtual functions: polymorphism

```cpp
#include <iostream>
#include <memory>
#include <numbers>
#include <string>
#include <vector>

class Shape {
public:
    virtual ~Shape() = default;                   // ALWAYS: virtual destructor in a polymorphic base
    virtual double area() const = 0;              // pure virtual: derived classes must implement
    virtual std::string name() const { return "shape"; }   // virtual with a default
};

class Circle : public Shape {
public:
    explicit Circle(double r) : r_{r} {}
    double area() const override { return std::numbers::pi * r_ * r_; }
    std::string name() const override { return "circle"; }
private:
    double r_;
};

class Rect : public Shape {
public:
    Rect(double w, double h) : w_{w}, h_{h} {}
    double area() const override { return w_ * h_; }
    std::string name() const override { return "rectangle"; }
private:
    double w_, h_;
};

class Square final : public Rect {                // final: no further derivation
public:
    explicit Square(double s) : Rect{s, s} {}
    std::string name() const override { return "square"; }
};

int main()
{
    std::vector<std::unique_ptr<Shape>> shapes;
    shapes.push_back(std::make_unique<Circle>(1.0));
    shapes.push_back(std::make_unique<Rect>(2.0, 3.0));
    shapes.push_back(std::make_unique<Square>(2.0));

    double total = 0;
    for (const auto &s : shapes) {
        std::cout << s->name() << ": " << s->area() << '\n';   // calls the right override
        total += s->area();
    }
    std::cout << "total: " << total << '\n';
}                                                  // unique_ptrs delete through Shape*: needs the virtual destructor
```

- **`virtual`** in the base makes calls through a base pointer or reference go to the most-derived
  override at run time (**dynamic dispatch**).
- **`= 0`** makes a function **pure virtual** and the class **abstract**: you can't create a
  `Shape` itself, only derived classes that implement all pure virtuals.
- **`override`** on every overriding function. It's optional, but without it a typo (a missing
  `const`, a different parameter type) silently creates a *new* function instead of overriding,
  and your override is never called. With it, that's a compile error.
- **`final`** forbids further overriding (on a function) or derivation (on a class).

### Under the hood: it's your C vtable

For each class with virtual functions, the compiler builds one **vtable** — an array of function
pointers — and puts a hidden **vptr** at the start of every object. `s->area()` becomes
`s->vptr->area(s)`. This is exactly what you built by hand on C day 27, generated and type-checked
for you. The cost: one pointer per object and an indirect call. Compare the C and C++ versions of
your zoo program (C day 27, exercise 7) — same structure, a third of the code.

### The virtual destructor rule

```cpp
std::unique_ptr<Shape> s = std::make_unique<Circle>(1.0);
// when s is destroyed, it calls delete on a Shape*
```

If `~Shape()` isn't virtual, deleting through a `Shape *` calls only `Shape`'s destructor —
`Circle`'s part (and its members) is never destroyed. That's undefined behavior. **A class meant to
be used polymorphically must have a public virtual destructor.** (`-Wall` includes
`-Wdelete-non-virtual-dtor`, which warns.)

### Slicing

Polymorphism only works through **pointers and references**. Copying a derived object into a base
**object** copies only the base part:

```cpp
void print_area(Shape s);        // doesn't even compile: Shape is abstract. Good!

Rect r{2, 3};
Rect copy_of_base = Square{4};   // compiles: sliced to a plain Rect; name() now says "rectangle"
```

Pass polymorphic objects by `const Base &` or `Base *`, store them as `std::unique_ptr<Base>`.
A good defense: make polymorphic base classes non-copyable, or abstract.

### Calling the base version

```cpp
std::string name() const override { return "special " + Rect::name(); }
```

### dynamic_cast: asking "what are you really?"

```cpp
if (auto *c = dynamic_cast<const Circle *>(shape_ptr)) {
    // shape_ptr points to a Circle (or something derived from it); c is non-null
}
```

`dynamic_cast` checks at run time and returns `nullptr` on failure (for pointers). Needing it often
is a design smell — usually a virtual function should do the work — but it's occasionally the right
tool.

### Constructors and virtual calls

During a base class's constructor or destructor, the object *is* only a base object, so virtual
calls go to the base version. Don't call virtual functions from constructors or destructors
expecting the derived behavior.

## Common mistakes

- No virtual destructor in a polymorphic base.
- Forgetting `override`, and a signature mismatch silently hides the base function.
- Slicing: storing derived objects in a `std::vector<Base>`.
- Deep hierarchies (more than 2–3 levels) — hard to understand and change. See tomorrow.
- Making everything `virtual` "just in case" — only functions meant to be overridden.

## Exercises

1. Type in the shapes program. Add a `Triangle`. Add a pure virtual `perimeter()` and implement it
   everywhere.
2. Remove `override` from `Circle::area` and change its signature to non-`const`. What happens?
   Put `override` back and read the error.
3. Remove `virtual` from `~Shape()`, give `Circle` a member of type `std::string` with a long value,
   and run with ASan (you may see a leak or a "new-delete-type-mismatch" report).
4. **The zoo:** an abstract `Animal` with `name()` and pure virtual `speak()`; `Dog`, `Cat` and
   `Cow`; a `std::vector<std::unique_ptr<Animal>>`; make them all speak. Compare with your C
   version from C day 27.
5. Demonstrate slicing: `std::vector<Rect>` holding a `Square`. Print `name()` of each element.
6. Add a `virtual std::unique_ptr<Shape> clone() const = 0;` to `Shape` and implement it in every
   class (`return std::make_unique<Circle>(*this);`). Use it to deep-copy a vector of shapes. This
   is the "virtual copy constructor" pattern.
7. Print `sizeof` of a class with no virtual functions and the same class with one. Explain the
   difference.
8. ★ Use `dynamic_cast` to count how many shapes in a vector are `Rect`s (including squares).

## Check yourself

1. What does `virtual` change about a function call?
2. Why should you always write `override`?
3. Why must a polymorphic base class have a virtual destructor?
4. What is slicing, and how do you avoid it?
5. What is a vtable, and how does it relate to C day 27?

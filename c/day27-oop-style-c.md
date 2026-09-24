# C Day 27 — Object-oriented programming in C

**Goal:** apply the three core OOP ideas — encapsulation, polymorphism and (a form of) inheritance
— in plain C. This shows you what C++ classes and virtual functions do under the hood, and it's how
large C projects (the Linux kernel, GTK, SQLite) are organized.

## Concepts

### What OOP is really about

Object-oriented programming isn't about a `class` keyword. It's three ideas:

1. **Encapsulation** — an object's data can only be changed through its functions, which protect
   its rules (invariants). "A bank account balance never goes negative."
2. **Polymorphism** — code works with many types through a common interface. "Draw every shape",
   without knowing which shapes exist.
3. **Inheritance / composition** — build new types from existing ones.

C supports all three with patterns.

### Encapsulation: opaque types

Declare the struct in the header **without its members** — an *incomplete type*. Users can hold
pointers to it but can't see or touch the fields; only the `.c` file can. You've already used
this: `FILE *`, and your `HashMap` from day 25.

`account.h`:

```c
#ifndef ACCOUNT_H
#define ACCOUNT_H

typedef struct Account Account;                 // opaque

Account    *account_create(const char *owner);  // "constructor"
void        account_destroy(Account *a);        // "destructor"
int         account_deposit(Account *a, long cents);
int         account_withdraw(Account *a, long cents);   // fails if it would go negative
long        account_balance(const Account *a);
const char *account_owner(const Account *a);

#endif
```

`account.c`:

```c
#include "account.h"

#include <stdlib.h>
#include <string.h>

struct Account {             // the real definition, visible only in this file
    char owner[64];
    long balance_cents;      // invariant: never negative
};

Account *account_create(const char *owner)
{
    Account *a = calloc(1, sizeof *a);
    if (a != NULL) {
        strncat(a->owner, owner, sizeof a->owner - 1);   // copies and always terminates
    }
    return a;
}

void account_destroy(Account *a) { free(a); }

int account_deposit(Account *a, long cents)
{
    if (cents <= 0) return -1;
    a->balance_cents += cents;
    return 0;
}

int account_withdraw(Account *a, long cents)
{
    if (cents <= 0 || cents > a->balance_cents) return -1;   // protect the invariant
    a->balance_cents -= cents;
    return 0;
}

long account_balance(const Account *a) { return a->balance_cents; }
const char *account_owner(const Account *a) { return a->owner; }
```

Because `main.c` can't write `acc->balance_cents = -500`, the invariant holds everywhere. You can
also change the struct's layout later without recompiling users — only `account.c` knows it.
The naming convention `type_verb(Type *self, ...)` stands in for methods; the first parameter is
what C++ calls `this`.

### Polymorphism: a table of function pointers (a vtable)

Define the interface as a struct of function pointers. Each "class" provides one static, constant
table, and each object starts with a pointer to its class's table:

```c
#include <stdio.h>
#include <stdlib.h>

typedef struct Shape Shape;

typedef struct {
    const char *name;
    double (*area)(const Shape *self);
    void (*destroy)(Shape *self);
} ShapeVTable;

struct Shape {
    const ShapeVTable *vtable;     // every shape starts with this
};

// "Virtual" calls: dispatch through the table
double shape_area(const Shape *s) { return s->vtable->area(s); }
void shape_destroy(Shape *s) { s->vtable->destroy(s); }

/* ---- Circle "derives from" Shape: its first member IS a Shape ---- */
typedef struct {
    Shape base;
    double radius;
} Circle;

static double circle_area(const Shape *self)
{
    const Circle *c = (const Circle *)self;   // OK: base is the first member
    return 3.14159265358979 * c->radius * c->radius;
}

static void generic_destroy(Shape *self) { free(self); }

static const ShapeVTable circle_vtable = {"circle", circle_area, generic_destroy};

Shape *circle_create(double radius)
{
    Circle *c = malloc(sizeof *c);
    if (c == NULL) return NULL;
    c->base.vtable = &circle_vtable;
    c->radius = radius;
    return &c->base;
}

/* ---- Rectangle ---- */
typedef struct {
    Shape base;
    double w, h;
} Rect;

static double rect_area(const Shape *self)
{
    const Rect *r = (const Rect *)self;
    return r->w * r->h;
}

static const ShapeVTable rect_vtable = {"rectangle", rect_area, generic_destroy};

Shape *rect_create(double w, double h)
{
    Rect *r = malloc(sizeof *r);
    if (r == NULL) return NULL;
    r->base.vtable = &rect_vtable;
    r->w = w;
    r->h = h;
    return &r->base;
}

int main(void)
{
    Shape *shapes[] = {circle_create(1.0), rect_create(2.0, 3.0), circle_create(0.5)};
    double total = 0;
    for (size_t i = 0; i < 3; i++) {
        if (shapes[i] == NULL) return 1;
        double a = shape_area(shapes[i]);      // calls the right function for each type
        printf("%-9s area %.3f\n", shapes[i]->vtable->name, a);
        total += a;
    }
    printf("total %.3f\n", total);
    for (size_t i = 0; i < 3; i++) {
        shape_destroy(shapes[i]);
    }
    return 0;
}
```

```text
 Circle object (heap)              circle_vtable (static, one for all circles)
 +----------------------+          +-----------------------+
 | base.vtable  ●-------+--------> | name    "circle"      |
 | radius       1.0     |          | area    circle_area   |
 +----------------------+          | destroy generic_destroy|
                                   +-----------------------+
```

This is **exactly** how C++ implements `virtual` functions: every object of a class with virtual
functions carries a hidden pointer to its class's vtable. On C++ day 11 you'll write `virtual double
area() const = 0;` and the compiler will generate all of this for you.

The cast from `Shape *` to `Circle *` is valid because a pointer to a struct can be converted to a
pointer to its first member and back. That's "inheritance by embedding": a `Circle` **is a**
`Shape` at the same address.

Compare with day 16's tagged union: there, adding a new *operation* is easy (write one function
with a `switch`), but adding a new *type* means editing every `switch`. With vtables it's the
opposite: new types are easy (a new file, nothing else changes), new operations need every type
updated. Neither is always better.

### Interfaces with a context pointer

Often you want polymorphic behavior without allocating objects. A common pattern: a struct with a
function pointer plus a `void *` for its state:

```c
typedef struct {
    void (*write)(void *ctx, const char *msg);
    void *ctx;
} Logger;

void log_msg(const Logger *log, const char *msg) { log->write(log->ctx, msg); }
```

A console logger ignores `ctx`; a file logger stores its `FILE *` there; a test logger stores a
buffer to check later. The code that logs doesn't care which one it has — this is **dependency
injection**, and it's how you make C code testable.

## Common mistakes

- Forgetting to set the vtable pointer in a constructor (crash on the first virtual call).
- Casting a `Shape *` to the wrong derived type.
- Putting the base struct anywhere but first.
- Exposing struct members in the header "just for now" — then everyone uses them and you can never
  change them.

## Exercises

1. Type in the `Account` module with a `main.c` that tries to break the invariant through the
   public functions. Then try `acc->balance_cents = -1;` in `main.c` and read the compiler error.
2. Add a `Triangle` to the shapes (base and height). Notice that no existing code changes except
   `main`.
3. Add `perimeter` to the vtable and implement it for all shapes. Notice that *every* shape
   changes.
4. Add a `describe` "method" with a default implementation: shapes whose vtable entry is `NULL` fall
   back to printing `"<name> with area X"`.
5. Implement the `Logger` interface with three implementations: console, file (ctx is a `FILE *`),
   and memory (ctx is a struct with a buffer and length). Write a function
   `process_order(const Logger *log, int id)` and test it with the memory logger, checking the
   logged text with `assert` and `strstr`.
6. Convert your day 22 stack into an opaque type `Stack` with `stack_create`, `stack_destroy`,
   `stack_push`, `stack_pop`, `stack_size` — nothing but a `typedef` in the header.
7. ★ **Animal zoo:** a vtable with `speak` and `name`; `Dog` and `Cat` "classes", each with an
   extra data field; an array of `Animal *`; call `speak` on each. You'll write the same program
   in C++ on C++ day 11 — keep this one to compare.

## Check yourself

1. What's an opaque type, and how does it enforce encapsulation?
2. What's in a vtable, and where does each object keep it?
3. Why must the base struct be the first member for the cast to work?
4. When is a tagged union a better choice than a vtable, and vice versa?
5. What's the `void *ctx` in the `Logger` for?

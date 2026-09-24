# C Day 15 — Structs

**Goal:** group related data into your own types, use pointers to structs, and understand what a
copy of a struct really copies.

## Concepts

### Defining and using a struct

```c
#include <stdio.h>

struct Point {
    double x;
    double y;
};

int main(void)
{
    struct Point a = {1.0, 2.0};              // positional initialization
    struct Point b = {.y = 5.0, .x = 3.0};    // designated initializers (C99) — clearer
    struct Point origin = {0};                // all members zero

    a.x += 10;                                // member access with .
    printf("a = (%g, %g), b = (%g, %g), origin = (%g, %g)\n",
           a.x, a.y, b.x, b.y, origin.x, origin.y);
    return 0;
}
```

### typedef: drop the `struct` keyword

```c
typedef struct {
    char name[32];
    int age;
    double gpa;
} Student;

Student s = {.name = "Ada", .age = 20, .gpa = 3.9};
```

Both styles are common. If the struct needs to refer to itself (linked lists, day 21), give it a
tag too: `typedef struct Node { int value; struct Node *next; } Node;`.

### Structs are values: assignment copies everything

```c
struct Point a = {1, 2};
struct Point b = a;     // copies all members
b.x = 100;              // a.x is still 1
```

Structs can be passed to and returned from functions by value — the whole struct is copied:

```c
struct Point midpoint(struct Point p, struct Point q)
{
    struct Point m = {(p.x + q.x) / 2, (p.y + q.y) / 2};
    return m;
}
```

That's fine for small structs. For larger ones, or when the function must modify the struct,
pass a **pointer**.

### Pointers to structs and ->

```c
#include <stdio.h>
#include <string.h>

typedef struct {
    char owner[32];
    long balance_cents;
} Account;

void deposit(Account *acc, long cents)
{
    acc->balance_cents += cents;          // acc->x is shorthand for (*acc).x
}

void print_account(const Account *acc)    // const: read-only, and no copy is made
{
    printf("%s: %ld.%02ld\n", acc->owner, acc->balance_cents / 100, acc->balance_cents % 100);
}

int main(void)
{
    Account a = {.owner = "Grace", .balance_cents = 1050};
    deposit(&a, 250);
    print_account(&a);                    // Grace: 13.00
    return 0;
}
```

**`p->member` means `(*p).member`.** The parentheses are needed in the long form because `.` binds
tighter than `*`. Everyone uses `->`.

Rule of thumb for parameters: small struct you only read → by value or `const T *`; anything you
modify → `T *`; big struct → `const T *` to avoid the copy.

### Arrays of structs, and sorting them

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct {
    char name[32];
    int score;
} Player;

int by_score_desc(const void *pa, const void *pb)
{
    const Player *a = pa;           // void * converts to Player * implicitly in C
    const Player *b = pb;
    return (b->score > a->score) - (b->score < a->score);
}

int main(void)
{
    Player players[] = {
        {"Ann", 30}, {"Bob", 55}, {"Cid", 42},
    };
    size_t n = sizeof players / sizeof players[0];

    qsort(players, n, sizeof players[0], by_score_desc);
    for (size_t i = 0; i < n; i++) {
        printf("%zu. %-5s %d\n", i + 1, players[i].name, players[i].score);
    }
    return 0;
}
```

### Structs containing pointers: the shallow copy trap

Copying a struct copies its members — including **pointers**, but not what they point to:

```c
typedef struct {
    char *name;       // heap-allocated
    int age;
} Person;

Person a = {my_strdup("Ann"), 30};
Person b = a;          // b.name and a.name point to the SAME heap string
free(a.name);          // now b.name dangles, and free(b.name) would be a double free
```

```text
 a: [name ●-]---+
               +--> "Ann"      both point to one block
 b: [name ●-]---+
```

If a struct owns heap memory, write functions for it: `person_create`, `person_copy` (which
duplicates the string — a **deep copy**), `person_destroy`. C++ automates exactly this with copy
constructors and destructors (C++ day 6).

### Padding and alignment

```c
#include <stddef.h>
#include <stdio.h>

struct Mixed {
    char c;      // 1 byte
    int i;       // 4 bytes, must start at a multiple of 4
    char d;      // 1 byte
};

int main(void)
{
    printf("sizeof = %zu\n", sizeof(struct Mixed));                       // 12, not 6
    printf("offsets: c=%zu i=%zu d=%zu\n", offsetof(struct Mixed, c),
           offsetof(struct Mixed, i), offsetof(struct Mixed, d));         // 0 4 8
    return 0;
}
```

The CPU reads values fastest (on some machines, only) at addresses that are multiples of their
size, so the compiler inserts **padding** bytes. Ordering members from largest to smallest usually
minimizes it. Consequences: never assume `sizeof` a struct is the sum of its members, and don't
compare structs with `memcmp` (padding bytes have unspecified values). C has no `==` for structs;
compare member by member.

### Refactoring IntVec

Yesterday's functions took three pointers. With a struct:

```c
typedef struct {
    int *data;
    size_t size;
    size_t capacity;
} IntVec;

int vec_push(IntVec *v, int value);
```

One pointer instead of three, and you can't mix up one vector's size with another's data.

## Common mistakes

- `p.member` when `p` is a pointer (use `->`), or `s->member` when `s` isn't.
- Shallow-copying a struct that owns heap memory, then freeing twice.
- Passing big structs by value in hot loops.
- Assuming no padding; comparing structs with `memcmp`.
- Forgetting that a `char name[32]` member must be copied with `snprintf`/`strcpy` — you can't
  assign an array: `s.name = "Bob";` doesn't compile.

## Exercises

1. Define `Rect` with a top-left `Point` and `width`/`height`. Write `double rect_area(const Rect *r)`,
   `int rect_contains(const Rect *r, Point p)`, and `Rect rect_intersection(const Rect *a, const Rect *b)`
   (width/height 0 if they don't overlap).
2. Read 5 students (name, age, GPA) into an array of `Student`. Sort them with `qsort` by GPA
   descending, then by name for ties. Print a table.
3. **Refactor your day 14 IntVec** into the struct version above. Note how the signatures shrink.
4. Write `Person *person_create(const char *name, int age)`, `Person *person_copy(const Person *p)`
   (deep copy) and `void person_destroy(Person *p)`, where `Person` holds a heap `char *name`.
   Verify with ASan that no copy shares memory with the original.
5. Write a `Fraction` struct with `fraction_add`, `fraction_mul` and `fraction_reduce` (use your
   `gcd` from day 5). Print `1/2 + 1/3 = 5/6`.
6. Print `sizeof` and `offsetof` of each member for a struct with a `char`, a `double`, an `int`
   and a `short`. Reorder the members to minimize the size.
7. ★ **Dynamic array of structs:** keep a heap array of `Player` that grows with `realloc` as you
   read players until EOF. Then print the top 3.

## Check yourself

1. What's the difference between `.` and `->`?
2. When should a function take `const T *` instead of `T`?
3. What does "shallow copy" mean, and when is it a bug?
4. Why can `sizeof` of a struct be bigger than the sum of its members?
5. How do you compare two structs for equality in C?

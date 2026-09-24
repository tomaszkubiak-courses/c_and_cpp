# C++ Day 14 — Week 2 project: a library management system

**Goal:** design and build an object-oriented program of 300–500 lines that uses classes, RAII,
smart pointers, polymorphism, operators, exceptions and `std::optional`.

## The project

A console application for a small lending library.

```text
library> add book "The C Programming Language" "Kernighan, Ritchie" 1978
Added item #1.
library> add dvd "Hackers" 105
Added item #2.
library> member "Ada Lovelace"
Registered member #1.
library> lend 1 1
Ada Lovelace borrowed #1 "The C Programming Language" (due in 21 days).
library> lend 1 1
Error: item #1 is already on loan.
library> list
  #1  [book] The C Programming Language — Kernighan, Ritchie (1978)   on loan to #1
  #2  [dvd]  Hackers (105 min)                                          available
library> return 1
library> save library.txt
library> quit
```

### Requirements

1. **Item hierarchy:** an abstract `Item` (id, title, virtual `describe()`, virtual
   `loan_days()`), with `Book` (authors, year; 21 days) and `Dvd` (minutes; 7 days). ★ Add
   `Magazine` (issue number; cannot be lent: `loan_days()` returns 0 and lending throws).
2. **Ownership:** `Library` owns items in `std::vector<std::unique_ptr<Item>>` and members in
   `std::vector<Member>`. Loans refer to items and members **by id**, not by pointer — remember day 9.
3. **Lookups** return `std::optional` or a pointer that may be null:
   `Item *find_item(int id)`, `const Member *find_member(int id) const`.
4. **Errors:** lending an unknown item, to an unknown member, or an item already on loan throws a
   `LibraryError` (derived from `std::runtime_error`). The command loop catches it and prints the
   message; the program keeps running.
5. **Operators:** `operator<<` for `Item` (calls the virtual `describe()`) and `Member`;
   `operator<=>` on `Member` by name, used by a `members` command that lists them sorted.
6. **Persistence:** `save FILE` / `load FILE` in a simple line-based text format of your design
   (use `std::ofstream`/`std::ifstream` and `std::getline` — day 28 has details, and this much is
   enough: `std::ofstream out{path}; out << ...;` and `std::ifstream in{path}; while (std::getline(in, line))`).
   Loading a malformed file throws, and the library is unchanged (strong guarantee: load into a new
   `Library`, then swap it in).
7. **Command parsing:** split input lines into words, supporting double-quoted arguments. Put this
   in its own function with its own tests.
8. **Rule of zero** everywhere: no `new`, no `delete`, no destructor you wrote yourself.

### Suggested structure

```text
library/
├── item.h / item.cpp          Item, Book, Dvd
├── member.h / member.cpp      Member
├── library.h / library.cpp    Library: add, find, lend, return, save, load
├── commands.h / commands.cpp  tokenizer + command dispatch
├── main.cpp                   the loop
└── tests.cpp                  assert-based tests of Library and the tokenizer
```

Build by hand for now (tomorrow's topics don't need CMake; day 20 will convert this project):

```bash
g++ -std=c++20 -Wall -Wextra -Wpedantic -g -fsanitize=address,undefined \
    item.cpp member.cpp library.cpp commands.cpp main.cpp -o library
```

### Design questions to answer first (write them down)

1. Why hold items as `unique_ptr<Item>` but members as plain `Member` values?
2. Why do loans store ids instead of `Item *`? What would go wrong with pointers if you add an
   "remove item" command?
3. Where does the "already on loan" rule live — in `Item`, in `Library`, or in a `Loan` class?
4. How will `load` know which derived class to create for each line?

### Extensions (★)

- Due dates with `std::chrono` (`std::chrono::year_month_day`, C++20) and an `overdue` command.
- Search: `find TEXT` over titles and authors, case-insensitive.
- Reservations: a queue of member ids per item; returning an item notifies the next member.
- Undo for the last command (store an inverse action).

## Week 2 review

1. What does `std::make_unique` return, and who deletes the object? (Day 8)
2. What does `weak_ptr::lock()` return when the object is gone? (Day 8)
3. Name four ways a reference or pointer can dangle in C++. (Day 9)
4. Why is `operator+` usually a free function built on `operator+=`? (Day 10)
5. What does `= default` on `operator<=>` generate? (Day 10)
6. What happens when you delete a derived object through a base pointer without a virtual
   destructor? (Day 11)
7. When is `std::variant` better than inheritance? (Day 12)
8. What's the strong exception guarantee, and how does copy-and-swap provide it? (Day 13)

# C Day 29 — Capstone project, part 1: design and core

**Goal:** build a complete, multi-module C program from scratch over two days, using everything
from the course: structs, dynamic memory, a hash table, sorting, file I/O, robust input, error
handling and CMake.

## The project: `contacts`, a command-line address book

```text
$ ./contacts
contacts> add
  Name: Ada Lovelace
  Phone: +44 20 7946 0000
  Email: ada@example.com
  Birth year: 1815
Added #1.
contacts> list
  #  Name              Phone              Email               Born
  1  Ada Lovelace      +44 20 7946 0000   ada@example.com     1815
  2  Grace Hopper      +1 212 555 0100    grace@example.com   1906
contacts> find grace
  2  Grace Hopper      +1 212 555 0100    grace@example.com   1906
contacts> sort born desc
contacts> delete 1
Deleted #1.
contacts> save
Saved 1 contact to contacts.csv.
contacts> quit
```

### Requirements

1. **Commands:** `add`, `list`, `find TEXT` (case-insensitive substring in name or email),
   `show ID`, `edit ID`, `delete ID`, `sort FIELD [asc|desc]` (name, born, id), `save [FILE]`,
   `load [FILE]`, `help`, `quit`.
2. **Storage:** a growable array of `Contact` structs (your IntVec pattern), with **heap-allocated
   strings** for name, phone and email — not fixed arrays.
3. **Index:** a hash map from ID to position is overkill here; instead use your day 25 hash map
   (adapted if needed) to prevent **duplicate emails**: `add` and `edit` reject an email that's
   already used.
4. **Persistence:** CSV file, one contact per line: `id,name,phone,email,born`. Handle fields that
   contain commas by quoting them (`"Doe, John"`) — or, simpler, reject commas in input and say so.
   The file loads automatically on start if it exists and is saved on `quit` if there are unsaved
   changes (ask first).
5. **Robust input:** everything read through `fgets`; numbers parsed with your `parse_int`; empty
   names rejected; birth year in a sensible range.
6. **Errors:** no crash on any input — try to break it. Every function that can fail returns a
   status. The `goto cleanup` pattern for file functions.
7. **Build:** CMake, with the warning flags. A `debug` build with sanitizers must run clean — no
   leaks when you `quit`.
8. **Tests:** a separate test executable (a second `add_executable` in CMake) that tests the
   contact list and CSV functions with `assert`.

### Suggested module layout

```text
contacts/
├── CMakeLists.txt
├── src/
│   ├── contact.h / contact.c        Contact struct: create, destroy, copy, validate
│   ├── contact_list.h / .c          growable array: add, remove, find_by_id, sort, foreach
│   ├── hashmap.h / hashmap.c        from day 25 (email -> id)
│   ├── storage.h / storage.c        save_csv, load_csv
│   ├── input.h / input.c            read_line, parse_int, ask_int, ask_string
│   ├── commands.h / commands.c      one function per command
│   └── main.c                       the read-dispatch loop
└── tests/
    └── test_main.c
```

CMake sketch — the modules become a library shared by the app and the tests:

```cmake
cmake_minimum_required(VERSION 3.20)
project(contacts C)

set(CMAKE_C_STANDARD 17)
set(CMAKE_C_STANDARD_REQUIRED ON)

add_library(contacts_core STATIC
    src/contact.c src/contact_list.c src/hashmap.c src/storage.c src/input.c src/commands.c)
target_include_directories(contacts_core PUBLIC src)
target_compile_options(contacts_core PRIVATE -Wall -Wextra -Wpedantic)

add_executable(contacts src/main.c)
target_link_libraries(contacts PRIVATE contacts_core)

add_executable(contacts_tests tests/test_main.c)
target_link_libraries(contacts_tests PRIVATE contacts_core)
```

### Design decisions to make (write them down)

Before coding, write a short `DESIGN.md` answering:

1. Who **owns** each string? When `contact_list_add` receives a `Contact`, does it copy it or take
   ownership?
2. How are IDs assigned? (Hint: a counter that never reuses IDs, stored in the file as the max ID.)
3. What does each module's header expose? What's `static`?
4. What happens to the email index when a contact is edited or deleted?
5. Command dispatch: a `switch` on the first letter, `if`/`strcmp` chain, or a table of
   `{ "name", handler_function }` structs (day 13)? (The table is the nicest.)

## Today's plan

1. Write `DESIGN.md` (30 min).
2. `contact.c`: `contact_create(name, phone, email, born)`, `contact_destroy`, `contact_copy`,
   and a validation function. Test them.
3. `contact_list.c`: the growable array with add, remove by id, find by id, sort (with `qsort` and
   one comparator per field and direction), and `foreach`. Test them.
4. `input.c`: reuse day 20's functions.
5. `main.c` with a command table and the `add`, `list`, `quit` commands working end to end.

Commit to a working program at the end of the day, even if it only has three commands. (If you
use git — you should — commit after each step.)

## Checklist for today

- [ ] `DESIGN.md` written
- [ ] `contact` module with tests
- [ ] `contact_list` module with tests
- [ ] `add`, `list`, `quit` working
- [ ] builds with no warnings; test binary passes under ASan

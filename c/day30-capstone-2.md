# C Day 30 — Capstone project, part 2: finish, harden, review

**Goal:** complete the contacts program, try hard to break it, review it against a checklist,
and plan what comes next.

## Today's plan

1. **Remaining commands:** `find`, `show`, `edit`, `delete`, `sort`, `help`.
2. **Email index** with the hash map: build it at load, update it on add/edit/delete.
3. **Storage:** `save_csv` and `load_csv` with full error handling (`goto cleanup`), a
   "modified" flag, and the "save changes?" question on quit.
4. **Tests** for storage: save a list, load it into a new list, compare every field; load a
   malformed file and check you get an error and no leak.
5. **Break it** (below), fix what breaks.
6. **Review** (below).

## Try to break it

Run the debug (sanitizer) build and try all of these:

- Empty input, just Enter, at every prompt.
- Very long input (paste 5,000 characters).
- Letters where numbers are expected; negative IDs; IDs that don't exist; `99999999999999`.
- Commands with extra spaces or wrong case.
- End of input (Ctrl+D / Ctrl+Z) in the middle of `add`.
- A CSV file with missing fields, extra fields, empty lines, a missing final newline, and a line of
  garbage.
- A read-only or missing save location.
- 10,000 contacts (generate them with a loop in the test program): is `list` fast? `find`? `load`?
- `delete` every contact, then `list`, `sort`, `save`.

Every case must produce a sensible message, never a crash, a sanitizer report or a leak.

## Code review checklist

Go through your code with this list. It's also a summary of the C course.

**Memory and pointers**
- [ ] Every `malloc`/`calloc`/`realloc` result is checked.
- [ ] `realloc` results go into a temporary.
- [ ] Every allocation has one clear owner, documented in the header.
- [ ] No pointer is used after the memory it points to is freed or reallocated.
- [ ] Strings get `+ 1` for the terminator; no `strcpy`/`strcat`/`sprintf` into buffers of unknown
      size.

**Functions and modules**
- [ ] Functions are short and do one thing.
- [ ] Read-only pointer parameters are `const`.
- [ ] Everything not in a header is `static`.
- [ ] Headers have include guards and contain no definitions.
- [ ] Error-returning functions have their return values checked by every caller.

**Input and errors**
- [ ] All input goes through `fgets` + parsing; no bare `scanf`.
- [ ] File functions clean up on every path.
- [ ] Error messages go to `stderr` and say what went wrong and where.
- [ ] `assert` is used only for programmer errors.

**Build**
- [ ] No warnings with `-Wall -Wextra -Wpedantic`.
- [ ] Tests pass under `-fsanitize=address,undefined`.
- [ ] `-fanalyzer` finds nothing you haven't understood.

## Extensions (★)

- Store birthdays as full dates and add `upcoming` (birthdays in the next 30 days) using
  `<time.h>`.
- Groups/tags per contact, stored as a list of strings, with `find tag:work`.
- Replace the growable array with your BST (day 26) keyed by name, and compare.
- Export to JSON; import from JSON (write a tiny parser — a serious but rewarding challenge).
- Command-line mode for scripting: `./contacts add "Ada" "+44..." "ada@..." 1815`.

## You've finished the C course

Look back at where you started: `printf("Hello, world!\n");`. You now know how memory, pointers,
the heap and the build process work at the level most programmers never see.

### Where to go next in C

- **Read good C code:** [SQLite](https://sqlite.org/src/doc/trunk/README.md) (superbly commented),
  [Redis](https://github.com/redis/redis) (the early versions are small and readable), the
  [Lua interpreter](https://www.lua.org/source/) (small, elegant).
- **Books:** *Effective C* (Seacord) for professional practice; *Modern C* (Gustedt) for depth;
  *Computer Systems: A Programmer's Perspective* (Bryant & O'Hallaron) for how C maps to hardware.
- **Projects:** write a shell, an HTTP server with sockets, an interpreter for a tiny language
  (the free book *Crafting Interpreters* by Robert Nystrom builds one in C), or firmware for a
  microcontroller (Arduino, Raspberry Pi Pico).
- **Practice:** the C track on [Exercism](https://exercism.org/tracks/c).

### Up next: C++

The [C++ course](../cpp/) starts from here. Watch for the moments where C++ automates something you
did by hand in C:

| In C you... | In C++ you'll use... |
|---|---|
| wrote `create`/`destroy` and freed on every path | constructors, destructors, RAII (C++ day 6) |
| wrote IntVec | `std::vector` (C++ day 4) |
| tracked ownership in comments | `std::unique_ptr` (C++ day 8) |
| built vtables by hand | `virtual` functions (C++ day 11) |
| wrote the hash map | `std::unordered_map` (C++ day 17) |
| used `void *` + element size for generic code | templates (C++ day 15) |
| passed function pointers + `void *ctx` | lambdas (C++ day 18) |
| returned error codes and used `goto cleanup` | exceptions, `std::optional`, `std::expected` (C++ day 13) |

And on C++ day 21 you'll call your own C code — like the day 25 hash map — from C++.

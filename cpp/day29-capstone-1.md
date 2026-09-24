# C++ Day 29 — Capstone project, part 1: design and core

**Goal:** build a complete, well-structured C++20 application over two days, using the whole
course: classes and RAII, smart pointers, the standard library, ranges, `std::format`, error
handling, file I/O, CMake, Catch2 tests — and, optionally, your own C code.

## The project: `taskr`, a command-line task manager

```text
$ taskr add "Write CMake lesson" --due 2026-10-01 --priority high --tag course
Added task 7.
$ taskr list
  ID  PRI   DUE         TAGS      TITLE
   3  high  2026-09-28  work      Review pull request
   7  high  2026-10-01  course    Write CMake lesson
   5  low   —           home      Fix the bike
$ taskr list --tag course --sort due
$ taskr done 3
Completed task 3.
$ taskr undo
Undid: done 3.
$ taskr stats
  7 tasks: 5 open, 2 done. 1 overdue. Most used tag: course (3).
```

Commands are passed as command-line arguments (`argv`), so each run loads the task file, performs
one command, and saves. (★ Add an interactive mode later.)

### Requirements

1. **Commands:** `add TITLE [--due DATE] [--priority low|normal|high] [--tag TAG]...`,
   `list [--all] [--tag TAG] [--sort id|due|priority]`, `done ID`, `edit ID [options]`,
   `remove ID`, `undo`, `stats`, `help`.
2. **Model:** a `Task` value type (id, title, priority as `enum class`, optional due date as
   `std::optional<std::chrono::year_month_day>`, tags as `std::vector<std::string>`, done flag).
   Defaulted `operator==`. Invariants enforced in the constructor (non-empty title).
3. **Repository:** a `TaskList` class owning the tasks (rule of zero), with `add`, `find` (returns
   `Task *` or `std::optional`), `remove`, `complete`, and a `query` function that takes a filter
   and a sort key and returns the matching tasks — implemented with ranges and projections.
4. **Undo:** each modifying command is a `Command` object with `execute(TaskList &)` and
   `undo(TaskList &)` (a polymorphic interface, `std::unique_ptr<Command>` in a history stack). The
   history is saved alongside the tasks so `undo` works across runs. (★ Alternatively store
   snapshots, and compare the two designs.)
5. **Storage:** a text format you design, in a file whose location comes from an environment
   variable (`TASKR_FILE`) or defaults to `tasks.txt` in the current directory. Use
   `std::filesystem` and write safely: write to a temporary file, then `fs::rename` over the old
   one, so a crash never leaves a half-written file.
6. **Errors:** bad arguments, unknown IDs, malformed files → a clear message on `stderr` and exit
   code 1. Internally: exceptions for I/O and corrupt data, `std::optional`/`std::expected` for
   lookups and parsing.
7. **Output:** aligned table with `std::format`; overdue tasks marked.
8. **Build:** the day 20 layout: a `taskr_core` library, the `taskr` executable, a Catch2 test
   executable, presets for debug (with sanitizers) and release.
9. **Tests:** parsing of arguments and dates, the text format round trip (save → load → compare),
   queries, every command's execute/undo pair.

### Suggested layout

```text
taskr/
├── CMakeLists.txt
├── CMakePresets.json
├── include/taskr/
│   ├── task.h            Task, Priority, date helpers
│   ├── task_list.h       TaskList, queries
│   ├── commands.h        Command interface and implementations
│   ├── storage.h         load/save
│   └── cli.h             argument parsing -> command objects
├── src/                  the .cpp files for the above
├── app/main.cpp          tiny: parse, load, execute, save, print
└── tests/                Catch2 tests, one file per module
```

### Optional: use your C code

Reuse your C hash map (C day 25) through a C++ wrapper (C++ day 21) as a tag index: tag → count,
used by `stats`. Build it as a C library target in the same CMake project
(`project(taskr LANGUAGES C CXX)`). It's not the simplest choice — `std::unordered_map` would do —
but it's excellent practice at the boundary, and a nice way to close the loop between the two
courses.

### Design questions to answer first

Write a short `DESIGN.md`:

1. What's `Task`'s invariant, and where is it enforced?
2. Who owns the tasks? Which functions get `Task &`, `const Task &`, `Task *`?
3. How are IDs assigned so they're never reused, even after `remove` and `undo`?
4. How does `RemoveCommand::undo` restore a task exactly (including its ID)?
5. What does `main` look like in 20 lines or fewer?
6. What's the file format? How will you escape a title that contains your separator character?

## Today's plan

1. `DESIGN.md` (30 min).
2. CMake skeleton with the library, app and test targets, building an empty `main` and one passing
   test.
3. `Task` with its constructor, date parsing (`"2026-10-01"` → `year_month_day`, returning
   `std::expected` or `std::optional`), and tests.
4. `TaskList` with add/find/remove/complete and `query`, with tests.
5. `storage` save/load round trip, with a test.
6. `main` wired for `add` and `list`.

## Checklist for today

- [ ] `DESIGN.md` written
- [ ] builds from a clean checkout with `cmake --preset debug && cmake --build --preset debug`
- [ ] `Task`, `TaskList`, storage implemented and tested
- [ ] `taskr add` and `taskr list` work end to end
- [ ] no warnings; tests pass under ASan/UBSan

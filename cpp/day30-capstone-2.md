# C++ Day 30 — Capstone project, part 2: finish, review, and what's next

**Goal:** complete `taskr`, review it against modern C++ practice, and plan where to go from here.

## Today's plan

1. **Commands:** `done`, `edit`, `remove` as `Command` classes with `execute`/`undo`, plus the
   history stack and `undo`.
2. **Queries:** `list` with `--tag`, `--all` and `--sort`, using range views and projections.
3. **Stats:** counts, overdue tasks (compare with today's date:
   `std::chrono::floor<std::chrono::days>(std::chrono::system_clock::now())`), most used tag.
4. **Atomic save** with a temporary file and `fs::rename`.
5. **Tests** for every command and its undo.
6. **Break it** (below), fix what breaks, and **review** (below).

## Try to break it

- No arguments; unknown command; missing values (`--due` at the end); unknown options.
- Invalid dates (`2026-02-30`, `tomorrow`, empty), invalid priorities, empty titles, a title with
  your file format's separator character and with non-ASCII characters (`"Zażółć gęślą jaźń"`).
- IDs: `0`, `-1`, `abc`, an ID that was removed, a huge number.
- `undo` with no history; `undo` twice after `remove`.
- A corrupt task file (truncated, garbage, duplicate IDs), a missing file (first run), a read-only
  directory.
- 100,000 tasks (generate them in a test): are `list` and `save` still fast? Time it in a Release
  build.

## Code review checklist

This is also a summary of the C++ course.

**Resources and ownership**
- [ ] No `new`, `delete`, `malloc` or `free` outside of a wrapper class.
- [ ] Every class follows the rule of zero, or has all five special members deliberately.
- [ ] Owners are `std::unique_ptr` / values / containers; raw pointers and references never own.
- [ ] No reference, pointer, iterator, `string_view` or `span` outlives what it refers to (day 9).

**Interfaces**
- [ ] Parameters follow day 3's table (`const T &` for reading, `T` for sinks, and so on).
- [ ] Member functions that don't modify are `const`; single-argument constructors are `explicit`.
- [ ] Polymorphic bases have virtual destructors; overrides say `override`.
- [ ] Headers are self-contained, use `#pragma once`, and have no `using namespace`.

**Error handling**
- [ ] Exceptions for exceptional failures, caught at the top level with a clear message.
- [ ] `std::optional`/`std::expected` where failure is a normal outcome.
- [ ] Destructors and move operations are `noexcept`.

**Standard library**
- [ ] Algorithms and ranges instead of hand-written loops where they're clearer.
- [ ] `std::format` for output; `std::filesystem` for paths.
- [ ] The right container for each job (and `std::vector` by default).

**Build and tests**
- [ ] Clean build with `-Wall -Wextra -Wpedantic -Wshadow -Wconversion`.
- [ ] Tests pass in Debug with sanitizers and in Release.
- [ ] `clang-tidy` findings reviewed.

## Extensions (★)

- Interactive mode (a REPL), with the same commands.
- Recurring tasks ("every Monday") using `std::chrono::weekday`.
- JSON storage with a library pulled in by `FetchContent` (for example nlohmann/json).
- Colored output with ANSI escape codes, disabled when output isn't a terminal.
- A `watch` command that checks for due tasks every minute on a `std::jthread` with a `stop_token`.
- Package it: `install()` rules in CMake, and a GitHub Actions workflow that builds and tests on
  Linux and Windows.

## You've finished both courses

Sixty days ago you wrote `printf("Hello, world!\n");`. You've since built linked lists, trees and
hash tables by hand, understood pointers down to the byte, written generic and object-oriented C++,
built projects with CMake, tested them, and called C from C++.

### Where to go next

**Deepen C++**

- Read *A Tour of C++* (3rd ed.) cover to cover — it will now read as a review with extras.
- *Effective Modern C++* (Meyers) and *C++ Software Design* (Klaus Iglberger) for design.
- Watch CppCon talks beyond "Back to Basics": pick topics you used here (ranges, concurrency,
  CMake).
- Follow the [C++ Core Guidelines](https://isocpp.github.io/CppCoreGuidelines/) and run
  `clang-tidy` with the `cppcoreguidelines-*` checks on your projects.

**Pick a direction and build something real**

| Direction | Starting points |
|---|---|
| Games | SFML or raylib (both easy to add with CMake), then a small 2D game |
| Graphics | *Learn OpenGL* (learnopengl.com), or *Ray Tracing in One Weekend* (a free, short book, in C++) |
| Systems | a shell, an HTTP server with sockets, a key-value store with a write-ahead log |
| Embedded | Raspberry Pi Pico or Arduino (C and C++), or Zephyr/FreeRTOS |
| Performance | profile with `perf` and a benchmark library (Google Benchmark); study cache effects |
| Compilers/interpreters | *Crafting Interpreters* (free online) — implement its C interpreter in C++ |
| Open source | find "good first issue" labels in C++ projects you use; read their code first |

**Keep practicing**

- Exercism's C++ track with mentoring.
- Advent of Code every December (and past years any time).
- Rewrite one of your C course projects in idiomatic C++ and compare size, safety and speed.

The most important habit: **keep writing code**, a little every week, on things you care about.
Come back to these lessons whenever a topic needs a refresh — especially the pointer days.

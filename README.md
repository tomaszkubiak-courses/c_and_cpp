# C & C++ in 60 Days

Repository: <https://github.com/tomaszkubiak-courses/c_and_cpp>

```bash
git clone https://github.com/tomaszkubiak-courses/c_and_cpp.git
```

Two 30-day courses you take one after the other:

1. **[C course](c/)** — C17 (the current mainstream standard), with notes on what C23 adds.
2. **[C++ course](cpp/)** — C++20, with a short look at C++23 and C++26.

Take C first. C shows you memory, pointers and how a program is built with nothing hidden, and
the C++ course builds on that: C++ smart pointers, references, RAII and vtables all make sense
once you've done them by hand in C. If you already know some C, skim C days 1–7 in two or three
days, then slow down for the pointer block (C days 8–14).

## Before you start

- **[Setup](00-setup.md)** — compiler, editor, debugger, sanitizers. Do this before C day 1.
- **[Resources](resources.md)** — books, websites, videos and practice sites, and which course
  days each one fits.

## How each day works

Plan on **about 2 hours a day** (some days are longer; see "Pace" below). Each lesson has:

- **Goal** — what you can do by the end of the day.
- **Concepts** — explanation with complete example programs. *Type them in* rather than
  copy-pasting them; typing is how the syntax sinks in.
- **Common mistakes** — the bugs everyone writes at least once.
- **Exercises** — do them in order; they get harder. Exercises marked ★ are optional stretch goals.
- **Check yourself** — short questions. If you can't answer one, reread that section.
- Answers to "predict the output" puzzles are hidden in collapsible `<details>` blocks. Decide on
  your answer *before* opening them.

Keep your work in a separate folder, for example `work/c/day08/`, one folder per day. Always
compile with warnings on (the setup guide shows how) and treat every warning as an error until
you understand it.

## Pace

Thirty days each is brisk. Some advice:

- **Pointers get more time on purpose.** C days 8–14 are all pointer work, and pointers come back
  in C days 21, 22 and 26 (linked lists, trees) and C++ days 8–9. If a pointer day takes you two
  days, take two days. It pays back for the rest of both courses.
- **Use the "project" days (C 7, 14, 29–30; C++ 7, 14, 29–30) to catch up** if you fall behind.
- Take one day off per week if you need it. Consistency beats cramming.
- **Draw memory diagrams on paper.** Every pointer lesson shows the notation; use it.

## Schedule

### C (C17)

| Week | Days | Topics |
|---|---|---|
| 1 | [1](c/day01-toolchain-first-program.md) · [2](c/day02-types-variables-io.md) · [3](c/day03-operators-conversions.md) · [4](c/day04-control-flow.md) · [5](c/day05-functions-scope.md) · [6](c/day06-arrays.md) · [7](c/day07-project-gradebook.md) | Toolchain, types, operators, control flow, functions, arrays, first project |
| 2 | [8](c/day08-pointers-1-addresses.md) · [9](c/day09-pointers-2-arithmetic-arrays.md) · [10](c/day10-pointers-3-strings.md) · [11](c/day11-pointers-4-const-double-pointers.md) · [12](c/day12-dynamic-memory.md) · [13](c/day13-function-pointers-void.md) · [14](c/day14-pointer-workout-project.md) | **The pointer block**: addresses, arithmetic, strings, `const`, `**`, heap, function pointers, workout |
| 3 | [15](c/day15-structs.md) · [16](c/day16-enums-unions-bits.md) · [17](c/day17-multi-file-builds.md) · [18](c/day18-preprocessor.md) · [19](c/day19-file-io.md) · [20](c/day20-error-handling-input.md) · [21](c/day21-linked-lists.md) | Structs, enums/unions, multi-file programs, preprocessor, files, errors, linked lists |
| 4 | [22](c/day22-stacks-queues.md) · [23](c/day23-algorithms-searching.md) · [24](c/day24-sorting.md) · [25](c/day25-hash-tables.md) · [26](c/day26-trees.md) · [27](c/day27-oop-style-c.md) · [28](c/day28-sharp-edges-c23.md) | Data structures, algorithms, OOP-style C, undefined behavior, C23 |
| 5 | [29](c/day29-capstone-1.md) · [30](c/day30-capstone-2.md) | Capstone project |

### C++ (C++20)

| Week | Days | Topics |
|---|---|---|
| 1 | [1](cpp/day01-hello-cpp.md) · [2](cpp/day02-references-const-types.md) · [3](cpp/day03-functions.md) · [4](cpp/day04-vector-string-span.md) · [5](cpp/day05-classes-1.md) · [6](cpp/day06-classes-2-raii.md) · [7](cpp/day07-move-semantics.md) | C to C++, references, functions, core containers, classes, RAII, move semantics |
| 2 | [8](cpp/day08-smart-pointers.md) · [9](cpp/day09-pointer-lifetime-workout.md) · [10](cpp/day10-operators-comparisons.md) · [11](cpp/day11-inheritance-polymorphism.md) · [12](cpp/day12-oop-design.md) · [13](cpp/day13-error-handling.md) · [14](cpp/day14-project-library.md) | Smart pointers, **pointer & lifetime workout**, operators, OOP, exceptions, project |
| 3 | [15](cpp/day15-templates-1.md) · [16](cpp/day16-templates-2-concepts.md) · [17](cpp/day17-stl-containers.md) · [18](cpp/day18-iterators-algorithms-lambdas.md) · [19](cpp/day19-ranges.md) · [20](cpp/day20-cmake.md) · [21](cpp/day21-calling-c-from-cpp.md) | Templates, concepts, STL, lambdas, ranges, **CMake**, **calling C from C++** |
| 4 | [22](cpp/day22-testing-debugging.md) · [23](cpp/day23-algorithms-1.md) · [24](cpp/day24-algorithms-2.md) · [25](cpp/day25-graphs.md) · [26](cpp/day26-modern-cpp-tour.md) · [27](cpp/day27-concurrency.md) · [28](cpp/day28-files-filesystem.md) | Testing, algorithms, graphs, C++20 features, threads, files |
| 5 | [29](cpp/day29-capstone-1.md) · [30](cpp/day30-capstone-2.md) | Capstone project |

## Progress

Tick days off as you finish them (edit this file, change `[ ]` to `[x]`).

**C:** [ ] 1 [ ] 2 [ ] 3 [ ] 4 [ ] 5 [ ] 6 [ ] 7 [ ] 8 [ ] 9 [ ] 10 [ ] 11 [ ] 12 [ ] 13 [ ] 14 [ ] 15
[ ] 16 [ ] 17 [ ] 18 [ ] 19 [ ] 20 [ ] 21 [ ] 22 [ ] 23 [ ] 24 [ ] 25 [ ] 26 [ ] 27 [ ] 28 [ ] 29 [ ] 30

**C++:** [ ] 1 [ ] 2 [ ] 3 [ ] 4 [ ] 5 [ ] 6 [ ] 7 [ ] 8 [ ] 9 [ ] 10 [ ] 11 [ ] 12 [ ] 13 [ ] 14 [ ] 15
[ ] 16 [ ] 17 [ ] 18 [ ] 19 [ ] 20 [ ] 21 [ ] 22 [ ] 23 [ ] 24 [ ] 25 [ ] 26 [ ] 27 [ ] 28 [ ] 29 [ ] 30

## Getting help while you learn

- When the compiler gives you an error, read the **first** error first; later ones are often
  caused by it.
- Paste your solution to an AI assistant or a forum and ask for a *review*, not a rewrite:
  "What's wrong with this, and why?" teaches you more than getting a fixed version.
- Use [Compiler Explorer](https://godbolt.org) to try tiny snippets and to see what the compiler
  does with them, and [Python Tutor](https://pythontutor.com/c.html) to *watch* pointers move.

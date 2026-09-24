# Resources — the "extra lesson"

The courses are self-contained, but these are the best outside resources I know of for each
topic. Free ones are marked **(free)**. You don't need all of them — pick one reference, one
book per language and one practice site, and use the rest when a topic doesn't click.

## References — keep these open while you code

- **cppreference.com** **(free)** — <https://en.cppreference.com>. The reference for both
  languages: every standard library function and header, with the version that added it.
  The C part is at <https://en.cppreference.com/w/c>. When a lesson names a function, look it up here.
- **Compiler Explorer** **(free)** — <https://godbolt.org>. Paste code, pick a compiler, see
  the output (and assembly) instantly. Great for "does this compile?" and "what does this do?".
- **cdecl** **(free)** — <https://cdecl.org>. Translates C declarations into English:
  `int (*(*f)(int))[5]` becomes readable. Use it on C day 11 and C day 13.
- **C++ Core Guidelines** **(free)** — <https://isocpp.github.io/CppCoreGuidelines/>. How to
  write modern C++, from Stroustrup and Sutter. Read the sections on resource management (R) and
  functions (F) after C++ day 8.
- **hacking C++** **(free)** — <https://hackingcpp.com>. Excellent visual cheat sheets for C++
  containers, algorithms and more.

## Pointers — for when pointers don't click

- **Python Tutor for C and C++** **(free)** — <https://pythontutor.com/c.html>. Runs your code
  step by step and *draws* every variable and pointer as boxes and arrows. Paste the pointer
  exercises from C days 8–14 into it.
- **Stanford CS Education Library** **(free)** — Nick Parlante's short classics:
  - *Pointers and Memory* — <http://cslibrary.stanford.edu/102/>
  - *Linked List Basics* and *Linked List Problems* — <http://cslibrary.stanford.edu/103/> and
    <http://cslibrary.stanford.edu/105/> (18 graded linked-list problems with solutions — ideal
    after C day 21)
  - *Binky Pointer Fun* video — <http://cslibrary.stanford.edu/104/> (3 minutes, silly, and it
    works)
  - *Binary Trees* — <http://cslibrary.stanford.edu/110/> (after C day 26)
- **Book: *Pointers on C*** by Kenneth Reek — old but the most thorough treatment of pointers in
  any book.

## Learning C

- **Beej's Guide to C Programming** **(free)** — <https://beej.us/guide/bgc/>. Friendly, modern,
  complete. Good second explanation for any C day.
- **CS50** (Harvard) **(free)** — <https://cs50.harvard.edu/x/>. The first weeks teach C with
  great lectures and problem sets. The memory lecture pairs well with C days 8–12.
- **Book: *Modern C*** by Jens Gustedt — covers C17 and C23, written by a member of the C
  standards committee. The author publishes it freely online: <https://gustedt.gitlabpages.inria.fr/modern-c/>.
- **Book: *Effective C*** (2nd edition) by Robert Seacord — practical and security-minded,
  covers C23. Good after this course.
- **Book: *The C Programming Language*** by Kernighan & Ritchie ("K&R") — the classic. Short and
  beautifully written, but predates modern C; read it for the style and the exercises.

## Learning C++

- **learncpp.com** **(free)** — <https://www.learncpp.com>. The best free C++ tutorial; it goes
  deeper than this course on every C++ topic. If a C++ day feels too fast, read the matching
  chapter there.
- **Book: *A Tour of C++*** (3rd edition) by Bjarne Stroustrup — C++20 in about 300 pages, by
  the language's creator. Perfect companion to this course.
- **Book: *Programming: Principles and Practice Using C++*** (3rd edition) by Stroustrup — a
  full beginner textbook, C++20.
- **Book: *Effective Modern C++*** by Scott Meyers — C++11/14, but still the best explanation of
  move semantics, smart pointers and lambdas. Read after C++ day 9.
- **CppCon "Back to Basics" talks** **(free)** — on the CppCon YouTube channel
  (<https://www.youtube.com/@CppCon>), search "Back to Basics". One-hour talks on exactly the
  topics in this course: pointers, RAII, move semantics, templates, concepts, CMake.
- **C++ Weekly** by Jason Turner **(free)** — <https://www.youtube.com/@cppweekly>. Short videos
  on one feature each.

## CMake

- **Official CMake tutorial** **(free)** — <https://cmake.org/cmake/help/latest/guide/tutorial/>
- **An Introduction to Modern CMake** **(free)** — <https://cliutils.gitlab.io/modern-cmake/>.
  Short and opinionated in the right way ("targets, not variables").
- **Book: *Professional CMake*** by Craig Scott (a CMake maintainer) — the complete reference,
  updated with each CMake release.

## Algorithms and data structures

- **VisuAlgo** **(free)** — <https://visualgo.net>. Animations of sorting, trees, hash tables and
  graph algorithms. Watch the animation before implementing on C days 23–26 and C++ days 23–25.
- **Book: *Grokking Algorithms*** by Aditya Bhargava — the gentlest introduction, with pictures.
- **cp-algorithms** **(free)** — <https://cp-algorithms.com>. Clear write-ups of classic
  algorithms, mostly with C++ code.
- **Book: *Introduction to Algorithms*** (Cormen et al., "CLRS") — the standard university
  textbook, for later.

## Practice

- **Exercism** **(free)** — <https://exercism.org/tracks/c> and <https://exercism.org/tracks/cpp>.
  Small exercises with automatic tests, and optional human mentoring on your solutions. The best
  fit for this course: do 2–3 exercises per week alongside it.
- **Advent of Code** **(free)** — <https://adventofcode.com>. 25 puzzles a year (all past years
  available). Great practice for C++ days 17–25 — parsing input, containers, graphs.
- **LeetCode** — <https://leetcode.com>. Interview-style algorithm problems; the "Easy" ones
  match C++ days 23–25.
- **Codewars** — <https://www.codewars.com>. Short "kata" in both languages, ranked by difficulty.

## Which resource goes with which day

| Course days | Best companion |
|---|---|
| C 1–7 | Beej's Guide chapters 1–6, CS50 weeks 1–2 |
| C 8–14 (pointers) | Python Tutor, Parlante's *Pointers and Memory*, Binky video, CS50 week 4 (memory) |
| C 15–20 | Beej's Guide (structs, files, preprocessor), *Modern C* |
| C 21–26 | Parlante's linked list and tree problems, VisuAlgo |
| C 27–28 | *Effective C*, cppreference's "undefined behavior" page |
| C++ 1–7 | learncpp.com, *A Tour of C++* chapters 1–6 |
| C++ 8–9 | Back to Basics: "Smart Pointers", "Move Semantics"; *Effective Modern C++* |
| C++ 10–16 | learncpp.com (OOP, templates), Back to Basics: "Concepts" |
| C++ 17–19 | hacking C++ cheat sheets, cppreference algorithm pages |
| C++ 20–22 | *An Introduction to Modern CMake*, official CMake tutorial |
| C++ 23–25 | VisuAlgo, cp-algorithms, LeetCode easy problems, Advent of Code |
| C++ 26–28 | cppreference "compiler support" page, C++ Weekly |

# Setup — compiler, editor, debugger

Do this before C day 1. It takes 30–60 minutes.

You need four things:

1. A **compiler** — turns `.c`/`.cpp` files into programs. We use **GCC** (`gcc` for C, `g++` for
   C++). Clang and Microsoft's MSVC also work; notes for them are below.
2. An **editor** — VS Code is used in the examples; any editor is fine.
3. A **debugger** — `gdb` (with GCC) or the Visual Studio debugger.
4. **CMake** — the build tool used from C day 17 and in depth on C++ day 20.

## Option A (recommended on Windows): WSL with Ubuntu

WSL runs a real Linux inside Windows. Most C/C++ tutorials, and every tool in this course
(including sanitizers and Valgrind, which find memory bugs), work there without changes.

1. In PowerShell as administrator: `wsl --install` (skip if you already have a distribution;
   `wsl -l -v` lists them). Reboot if asked, then open "Ubuntu" from the Start menu.
2. Install the tools:

   ```bash
   sudo apt update
   sudo apt install build-essential gdb cmake ninja-build valgrind clang clang-format clang-tidy
   ```

3. Check they work:

   ```bash
   gcc --version
   g++ --version
   cmake --version
   ```

4. Install **VS Code** on Windows, then these extensions: *WSL*, *C/C++* (Microsoft),
   *CMake Tools*. In the Ubuntu terminal, go to your work folder and type `code .` — VS Code opens
   connected to Linux.

**Keep your work inside the Linux file system** (for example `~/learn/`), not under `/mnt/c/...`
or `/mnt/d/...`. Building across the Windows/Linux boundary is many times slower, and some tools
misbehave there. You can still open the Linux folder from Windows Explorer at `\\wsl$\`.

## Option B: Visual Studio (MSVC)

If you prefer a full IDE: install **Visual Studio Community** with the workload *Desktop
development with C++*. It includes the MSVC compiler, CMake, a very good debugger and
AddressSanitizer. Use *File → Open → Folder* on a folder with a `CMakeLists.txt`, or compile
single files from the **Developer PowerShell for VS**:

```powershell
cl /std:c17 /W4 hello.c
cl /std:c++20 /W4 /EHsc hello.cpp
```

Notes: MSVC doesn't support variable-length arrays (C day 6 mentions them; you won't need them),
and `scanf` triggers "unsafe function" warnings you can silence with
`#define _CRT_SECURE_NO_WARNINGS` at the top of the file.

## Option C: MSYS2 (native GCC on Windows)

Install MSYS2 from <https://www.msys2.org>, open the *UCRT64* shell and run
`pacman -S mingw-w64-ucrt-x86_64-toolchain mingw-w64-ucrt-x86_64-cmake mingw-w64-ucrt-x86_64-ninja`.
It works well, but sanitizers don't — so for the memory-debugging exercises you'll want WSL or MSVC.

## Folder names

Put your work in a folder whose path has **no spaces and no special characters** like `&`, `(`,
`#`. Build tools pass paths through shells, and a character like `&` means "run another
command" to a shell. Something like `~/learn/c/day01/` is ideal.

## Compiling a single file

You'll compile most C-course exercises by hand. Always turn on warnings:

```bash
# C
gcc -std=c17 -Wall -Wextra -Wpedantic -g hello.c -o hello
./hello

# C++
g++ -std=c++20 -Wall -Wextra -Wpedantic -g hello.cpp -o hello
./hello
```

What the flags mean:

| Flag | Meaning |
|---|---|
| `-std=c17` / `-std=c++20` | Which language version. **Always set it.** GCC 15 defaults to C23 for C, so without the flag you'd silently get a newer language than the course teaches. |
| `-Wall -Wextra -Wpedantic` | Turn on the useful warnings. Treat each warning as a bug until you understand it. |
| `-g` | Include debug information, so the debugger and sanitizers can show source lines. |
| `-o hello` | Name of the output program (otherwise it's `a.out`). |
| `-O2` | Optimize. Use it when timing things (algorithm days), not while debugging. |
| `-Werror` | Turn warnings into errors. Good discipline once you're comfortable. |

### Memory-bug detectors (sanitizers)

From C day 8 on, compile while you practice with:

```bash
gcc -std=c17 -Wall -Wextra -Wpedantic -g -fsanitize=address,undefined prog.c -o prog
```

`-fsanitize=address` (ASan) stops the program with a clear report on out-of-bounds access,
use-after-free, double free and leaks. `-fsanitize=undefined` (UBSan) reports undefined behavior
such as signed overflow. They make the program slower, so leave them off when timing.

Save typing with aliases — add these to `~/.bashrc`, then open a new terminal:

```bash
alias c17='gcc -std=c17 -Wall -Wextra -Wpedantic -g -fsanitize=address,undefined'
alias cpp20='g++ -std=c++20 -Wall -Wextra -Wpedantic -g -fsanitize=address,undefined'
```

Then: `c17 prog.c -o prog && ./prog`.

**Valgrind** is another leak/memory checker; run a program *without* `-fsanitize` like this:
`valgrind --leak-check=full ./prog`.

## Debugger basics (gdb)

Compile with `-g`, then `gdb ./prog`:

| Command | What it does |
|---|---|
| `break main` or `b file.c:42` | Stop at a function or line |
| `run` (`r`) | Start the program (add arguments after it: `run a b c`) |
| `next` (`n`) | Run the next line, stepping *over* function calls |
| `step` (`s`) | Run the next line, stepping *into* function calls |
| `print x` (`p x`) | Show a variable; `p *ptr`, `p arr[3]`, `p *arr@5` (5 elements) also work |
| `display x` | Print `x` after every step |
| `backtrace` (`bt`) | Show the chain of function calls — the first thing to type after a crash |
| `continue` (`c`) | Run until the next breakpoint |
| `quit` (`q`) | Exit |

In VS Code you can do the same with breakpoints (click left of a line number) and F5, once the
C/C++ extension has created a `launch.json`. Learn the debugger early: by C day 12 it will save
you hours.

## Formatting

`clang-format -i file.c` reformats a file. Pick a style once, put it in a `.clang-format` file in
your work folder (for example `BasedOnStyle: LLVM` with `IndentWidth: 4`), and stop thinking about
formatting.

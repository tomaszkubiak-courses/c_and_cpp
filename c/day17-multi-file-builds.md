# C Day 17 — Multi-file programs, headers, linking, and a first CMake

**Goal:** split a program into modules with headers, understand what the compiler and linker each
do, build a static library, and build with a minimal CMake project.

## Concepts

### Why split?

A single 3000-line file is slow to navigate and slow to rebuild. Real C programs are split into
**modules**: each is a `.c` file with the implementation and a `.h` header with its public
interface. Each `.c` file is compiled separately into an object file (a **translation unit**), and
the linker joins them.

### A module: header + source

`intvec.h` — the **interface**: what other files may use.

```c
#ifndef INTVEC_H
#define INTVEC_H

#include <stddef.h>

typedef struct {
    int *data;
    size_t size;
    size_t capacity;
} IntVec;

int  intvec_init(IntVec *v, size_t capacity);
void intvec_free(IntVec *v);
int  intvec_push(IntVec *v, int value);

#endif
```

`intvec.c` — the **implementation**:

```c
#include "intvec.h"

#include <stdlib.h>

static int grow(IntVec *v)                 // static: private to this file
{
    size_t new_cap = v->capacity ? v->capacity * 2 : 4;
    int *p = realloc(v->data, new_cap * sizeof *p);
    if (p == NULL) {
        return -1;
    }
    v->data = p;
    v->capacity = new_cap;
    return 0;
}

int intvec_init(IntVec *v, size_t capacity)
{
    v->size = 0;
    v->capacity = capacity;
    v->data = capacity ? malloc(capacity * sizeof *v->data) : NULL;
    return (capacity && v->data == NULL) ? -1 : 0;
}

void intvec_free(IntVec *v)
{
    free(v->data);
    v->data = NULL;
    v->size = v->capacity = 0;
}

int intvec_push(IntVec *v, int value)
{
    if (v->size == v->capacity && grow(v) != 0) {
        return -1;
    }
    v->data[v->size++] = value;
    return 0;
}
```

`main.c` — a user of the module:

```c
#include <stdio.h>

#include "intvec.h"

int main(void)
{
    IntVec v;
    if (intvec_init(&v, 0) != 0) {
        return 1;
    }
    for (int i = 1; i <= 10; i++) {
        intvec_push(&v, i * i);
    }
    printf("%zu elements, last = %d\n", v.size, v.data[v.size - 1]);   // 10 elements, last = 100
    intvec_free(&v);
    return 0;
}
```

Details that matter:

- `#include "file.h"` (quotes) searches your project first; `<file.h>` searches system headers.
- **Include guards** (`#ifndef INTVEC_H` / `#define` / `#endif`) stop a header being pasted twice
  into the same translation unit. `#pragma once` does the same and is supported everywhere in
  practice, though it's not standard C.
- The `.c` file includes its own header first — that checks the header is self-contained.
- **`static` on a function or global variable** makes it private to its file (*internal linkage*).
  Make everything `static` that isn't in the header.
- Headers contain **declarations** (types, prototypes, `extern` variable declarations), not
  **definitions** of functions or variables — a definition in a header ends up in every file that
  includes it, and the linker complains about duplicates.

### Building by hand

```bash
gcc -std=c17 -Wall -Wextra -c intvec.c -o intvec.o   # compile only (-c)
gcc -std=c17 -Wall -Wextra -c main.c -o main.o
gcc intvec.o main.o -o app                            # link
```

Change `main.c` and only `main.o` needs rebuilding. That's what build tools automate.

### Linker errors you'll meet

| Message | Cause |
|---|---|
| `undefined reference to 'intvec_push'` | declared (header) but no object file defines it — you forgot to compile/link `intvec.c`, or misspelled the definition |
| `multiple definition of 'counter'` | the same function/global is defined in two files — typically a definition in a header |
| `undefined reference to 'sqrt'` | the math library isn't linked: add `-lm` |

### Global variables across files: extern

If you really need a global shared between files: **define** it in exactly one `.c` file
(`int log_level = 1;`) and **declare** it in the header with `extern int log_level;`. Usually a
pair of functions (`get_log_level`/`set_log_level`) with a `static` variable is the better design.

### Static libraries

Bundle object files into a library other programs can link:

```bash
ar rcs libintvec.a intvec.o
gcc main.c -L. -lintvec -o app     # -L: where to look, -lintvec: libintvec.a
```

### A minimal CMake project

Typing compiler commands doesn't scale. **CMake** describes *what* to build; it then generates the
real build files (Makefiles, Ninja files, a Visual Studio solution) for your platform. The C++
course covers it in depth (C++ day 20); this is all you need for the rest of the C course.

`CMakeLists.txt` next to your sources:

```cmake
cmake_minimum_required(VERSION 3.20)
project(intvec_demo C)

set(CMAKE_C_STANDARD 17)
set(CMAKE_C_STANDARD_REQUIRED ON)

add_library(intvec STATIC intvec.c)
target_include_directories(intvec PUBLIC .)
target_compile_options(intvec PRIVATE -Wall -Wextra -Wpedantic)

add_executable(app main.c)
target_link_libraries(app PRIVATE intvec)
target_compile_options(app PRIVATE -Wall -Wextra -Wpedantic)
```

Build it:

```bash
cmake -S . -B build                         # configure: generate build files into build/
cmake --build build                         # compile and link
./build/app
```

After editing a source file, only run `cmake --build build` again. `build/` is disposable —
delete it any time. (The `-Wall` options are GCC/Clang syntax; C++ day 20 shows how to make them
portable to MSVC.)

To build with sanitizers:

```bash
cmake -S . -B build-asan -DCMAKE_BUILD_TYPE=Debug -DCMAKE_C_FLAGS="-fsanitize=address,undefined"
cmake --build build-asan
```

## Common mistakes

- Defining functions or variables in a header.
- Missing include guards.
- Forgetting to add a new `.c` file to the build (→ "undefined reference").
- Putting `-lm` before the source files (the linker reads left to right).
- Running CMake inside the source folder without `-B build`, which scatters generated files among
  your sources.

## Exercises

1. Split your day 15 IntVec (with all functions from day 14) into `intvec.h`, `intvec.c` and a
   `main.c` test. Build by hand with `-c` and link.
2. Break it on purpose and read each error: remove `intvec.o` from the link command; define
   `int counter = 0;` in the header and include it from two `.c` files; remove the include guard
   and include the header twice.
3. Make the helper `grow` non-`static` and define another `grow` in `main.c`. What happens? Then
   restore `static`.
4. Build `libintvec.a` with `ar` and link against it.
5. Write the `CMakeLists.txt` above and build with CMake. Change `main.c`, rebuild, and notice that
   only `main.c` is recompiled (`cmake --build build -- -v` shows the commands with Makefiles).
6. Create a second module `stats.h/stats.c` with `double mean(const IntVec *v)` and
   `int max(const IntVec *v)`, add it to the library in CMake, and use it from `main.c`.
7. ★ Use `nm libintvec.a` (or `nm intvec.o`) to list the symbols. Which ones are marked `T`
   (global) and which `t` (local/static)?

## Check yourself

1. What goes into a header, and what doesn't?
2. What do include guards prevent?
3. What does `static` mean on a function at file scope?
4. Which tool reports "undefined reference", and what are two common causes?
5. What are the two CMake commands to configure and build?

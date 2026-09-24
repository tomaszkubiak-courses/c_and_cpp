# C++ Day 20 — Managing projects with CMake

**Goal:** structure a real project into libraries, executables and tests; write modern,
target-based CMake; build on any platform; and pull in third-party libraries.

You wrote a minimal `CMakeLists.txt` on C day 17. Today is the full picture.

## Concepts

### What CMake is (and isn't)

CMake is a **build system generator**. You describe *what* to build — targets and their
relationships — and CMake generates the files for an actual build tool: Makefiles or Ninja on Linux,
a Visual Studio solution on Windows, Xcode on macOS. The same `CMakeLists.txt` works on all of them.
It's the de-facto standard for C and C++: most libraries ship a CMake build, and IDEs (VS Code,
Visual Studio, CLion) open CMake projects directly.

### The workflow: configure, build, run

```bash
cmake -S . -B build                  # configure: read CMakeLists.txt, generate build files in build/
cmake --build build                  # build (calls make/ninja/msbuild for you)
cmake --build build --parallel       # use all CPU cores
ctest --test-dir build               # run the tests
```

- Always build **out of source** (`-B build`). The `build/` directory is disposable; add it to
  `.gitignore`.
- Choose a generator with `-G`: `cmake -S . -B build -G Ninja` (fast; install `ninja-build`).
- Pass options with `-D`: `-DCMAKE_BUILD_TYPE=Release`.

### Build types

| `CMAKE_BUILD_TYPE` | Flags (GCC/Clang) | Use |
|---|---|---|
| `Debug` | `-g` | development, debugging |
| `Release` | `-O3 -DNDEBUG` | shipping, benchmarking |
| `RelWithDebInfo` | `-O2 -g -DNDEBUG` | profiling |
| `MinSizeRel` | `-Os -DNDEBUG` | small binaries |

Single-config generators (Makefiles, Ninja) pick it at configure time. Multi-config generators
(Visual Studio, `Ninja Multi-Config`) pick it at build time: `cmake --build build --config Release`.

### Modern CMake: targets, not variables

Old CMake set global variables (`include_directories(...)`, `set(CMAKE_CXX_FLAGS ...)`) that leaked
into everything. **Modern CMake is about targets**: each library or executable declares its own
sources, include directories, compile options and dependencies, and dependents inherit what they
need automatically.

The key commands:

| Command | Does |
|---|---|
| `add_executable(name src...)` | a program |
| `add_library(name STATIC\|SHARED src...)` | a library (`.a`/`.lib` static, `.so`/`.dll` shared) |
| `target_include_directories(t PUBLIC\|PRIVATE dir)` | where `#include` looks |
| `target_link_libraries(t PUBLIC\|PRIVATE other)` | link against another target — and inherit its PUBLIC properties |
| `target_compile_features(t PUBLIC cxx_std_20)` | require a language standard |
| `target_compile_options(t PRIVATE ...)` | compiler flags |
| `target_compile_definitions(t PRIVATE NAME=VALUE)` | `-D` macros |

### PUBLIC, PRIVATE, INTERFACE

The visibility keyword says who gets the property:

- **PRIVATE** — only this target uses it (e.g. warning flags, internal include dirs, a library used
  only inside the `.cpp` files).
- **INTERFACE** — only targets that link to this one use it (header-only libraries).
- **PUBLIC** — both (e.g. the library's own `include/` directory, because its headers are included
  by users; a dependency whose types appear in your public headers).

If `app` links `mylib` and `mylib` has `target_include_directories(mylib PUBLIC include)`, then
`app` automatically gets `-I.../include`. No global state, and no copy-pasting flags.

### A real project layout

```text
geometry/
├── CMakeLists.txt              top level
├── CMakePresets.json           (optional) named configurations
├── cmake/
│   └── warnings.cmake          reusable helper
├── include/geometry/           public headers — note the extra folder level
│   ├── shapes.h
│   └── vec2.h
├── src/                        library implementation
│   ├── CMakeLists.txt
│   └── shapes.cpp
├── app/                        the program
│   ├── CMakeLists.txt
│   └── main.cpp
└── tests/
    ├── CMakeLists.txt
    └── test_shapes.cpp
```

Users write `#include "geometry/shapes.h"` — the folder prefix avoids clashes with other libraries'
`shapes.h`.

Top-level `CMakeLists.txt`:

```cmake
cmake_minimum_required(VERSION 3.25)
project(geometry VERSION 1.0 LANGUAGES CXX)

set(CMAKE_CXX_EXTENSIONS OFF)                 # -std=c++20, not -std=gnu++20
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)         # compile_commands.json for clangd / clang-tidy

include(cmake/warnings.cmake)

add_subdirectory(src)
add_subdirectory(app)

option(GEOMETRY_BUILD_TESTS "Build the tests" ON)
if(GEOMETRY_BUILD_TESTS)
    enable_testing()
    add_subdirectory(tests)
endif()
```

`cmake/warnings.cmake` — portable warning flags using a **generator expression** (`$<...>`), which
CMake evaluates per compiler at generate time:

```cmake
function(set_project_warnings target)
    target_compile_options(${target} PRIVATE
        $<$<CXX_COMPILER_ID:MSVC>:/W4 /permissive->
        $<$<NOT:$<CXX_COMPILER_ID:MSVC>>:-Wall -Wextra -Wpedantic -Wshadow -Wconversion>
    )
endfunction()
```

`src/CMakeLists.txt`:

```cmake
add_library(geometry STATIC shapes.cpp)
target_include_directories(geometry PUBLIC ${PROJECT_SOURCE_DIR}/include)
target_compile_features(geometry PUBLIC cxx_std_20)
set_project_warnings(geometry)
```

`app/CMakeLists.txt`:

```cmake
add_executable(geometry_app main.cpp)
target_link_libraries(geometry_app PRIVATE geometry)    # gets the include dir and C++20 too
set_project_warnings(geometry_app)
```

`tests/CMakeLists.txt` (plain `assert`-based tests; day 22 swaps in a test framework):

```cmake
add_executable(test_shapes test_shapes.cpp)
target_link_libraries(test_shapes PRIVATE geometry)
add_test(NAME shapes COMMAND test_shapes)               # ctest runs it; exit code 0 = pass
```

Code for the example:

`include/geometry/shapes.h`:

```cpp
#pragma once

namespace geometry {
    double circle_area(double r);
    double rect_area(double w, double h);
}
```

`src/shapes.cpp`:

```cpp
#include "geometry/shapes.h"

#include <numbers>

namespace geometry {
    double circle_area(double r) { return std::numbers::pi * r * r; }
    double rect_area(double w, double h) { return w * h; }
}
```

`app/main.cpp`:

```cpp
#include <format>
#include <iostream>

#include "geometry/shapes.h"

int main()
{
    std::cout << std::format("circle: {:.3f}, rect: {}\n", geometry::circle_area(1.0),
                             geometry::rect_area(2, 3));
}
```

`tests/test_shapes.cpp`:

```cpp
#include <cassert>
#include <cmath>

#include "geometry/shapes.h"

int main()
{
    assert(geometry::rect_area(2, 3) == 6);
    assert(std::abs(geometry::circle_area(1.0) - 3.14159265) < 1e-6);
}
```

Build and test:

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build
./build/app/geometry_app
ctest --test-dir build --output-on-failure
```

(Note: `assert` does nothing in Release builds, because of `-DNDEBUG` — another reason to use a
real test framework on day 22.)

### Using third-party libraries

**1. Installed libraries: `find_package`.** If a library is installed on the system (via `apt`,
`vcpkg`, `conan`...) and provides a CMake package:

```cmake
find_package(fmt REQUIRED)
target_link_libraries(app PRIVATE fmt::fmt)     # namespaced targets: Name::Name
```

**2. Download at configure time: `FetchContent`.** CMake downloads the source and builds it as part
of your project — no installation needed:

```cmake
include(FetchContent)
FetchContent_Declare(
    fmt
    GIT_REPOSITORY https://github.com/fmtlib/fmt.git
    GIT_TAG 11.1.4                               # always pin a version (a tag or commit)
)
FetchContent_MakeAvailable(fmt)

target_link_libraries(app PRIVATE fmt::fmt)
```

Day 22 uses this to fetch the Catch2 test framework. For larger projects, a **package manager**
like vcpkg or Conan manages many dependencies and caches builds.

### CMakePresets.json

Instead of remembering long configure commands, name them:

```json
{
  "version": 6,
  "configurePresets": [
    {
      "name": "debug",
      "binaryDir": "${sourceDir}/build/debug",
      "cacheVariables": {
        "CMAKE_BUILD_TYPE": "Debug",
        "CMAKE_CXX_FLAGS": "-fsanitize=address,undefined"
      }
    },
    {
      "name": "release",
      "binaryDir": "${sourceDir}/build/release",
      "cacheVariables": { "CMAKE_BUILD_TYPE": "Release" }
    }
  ],
  "buildPresets": [
    { "name": "debug", "configurePreset": "debug" },
    { "name": "release", "configurePreset": "release" }
  ]
}
```

```bash
cmake --preset debug
cmake --build --preset debug
```

VS Code's CMake Tools and Visual Studio show presets in a drop-down.

### Installing (briefly)

`install(TARGETS geometry_app)` and `cmake --install build --prefix <dir>` copy the built program
(and, for libraries, headers and a CMake package so others can `find_package` it). You'll need this
when you publish a library; skip it for now.

### Things to avoid

- Global commands: `include_directories`, `link_libraries`, `add_definitions`,
  `set(CMAKE_CXX_FLAGS ...)` in `CMakeLists.txt` — use the `target_*` versions.
- `file(GLOB ...)` to collect sources — CMake doesn't notice new files until you re-configure. List
  sources explicitly.
- Hardcoding GCC flags without checking the compiler: MSVC ignores `-Wextra` with a warning, and
  reads `-Wall` as "every warning there is", flooding you with noise from system headers. Use a
  generator expression as above.
- Building in the source directory.
- Absolute paths in `CMakeLists.txt` — use `${PROJECT_SOURCE_DIR}`, `${CMAKE_CURRENT_SOURCE_DIR}`.

## Exercises

1. Create the `geometry` project above exactly, build it, run the app and `ctest`.
2. Add `vec2.h` (header-only: make a separate `INTERFACE` library target, `add_library(vec2 INTERFACE)`,
   and link `geometry` to it). Which keyword does `geometry` need so that `geometry_app` can also
   include `vec2.h`?
3. Break the build on purpose: change `PUBLIC` to `PRIVATE` in `target_include_directories` for
   `geometry`. What error do you get, and in which target?
4. Add the `CMakePresets.json` above; configure and build both presets. Check that the debug build
   catches a deliberate out-of-bounds write in `main.cpp`.
5. Use `FetchContent` to add the `fmt` library and use `fmt::print` in the app.
6. **Convert your day 14 library project** to this layout: a `library_core` static library with
   everything except `main.cpp`, an executable, and your `tests.cpp` registered with `add_test`.
7. Open the project in VS Code (CMake Tools) or Visual Studio (*Open Folder*), select a preset,
   build and debug from the IDE.
8. ★ Add an `option(ENABLE_SANITIZERS ...)` that adds the sanitizer flags to all targets via a
   helper function, instead of the preset's `CMAKE_CXX_FLAGS`.

## Check yourself

1. What does "build system generator" mean?
2. What's the difference between `PUBLIC`, `PRIVATE` and `INTERFACE`?
3. Why prefer `target_include_directories` to `include_directories`?
4. How do you choose Release vs Debug with Makefiles? With Visual Studio?
5. What does `FetchContent` do, and why pin a `GIT_TAG`?
6. Why avoid `file(GLOB)` for sources?

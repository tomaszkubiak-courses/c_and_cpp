# C++ Day 28 — Files, streams and the filesystem

**Goal:** read and write text and binary files with streams, parse text with string streams and
`std::from_chars`, and work with paths and directories using `std::filesystem`.

## Concepts

### File streams

`<fstream>` provides `std::ifstream` (read), `std::ofstream` (write) and `std::fstream` (both).
They're RAII: the file closes when the stream is destroyed.

```cpp
#include <fstream>
#include <iostream>
#include <string>

int main()
{
    {
        std::ofstream out{"notes.txt"};                  // truncates; std::ios::app to append
        if (!out) {
            std::cerr << "cannot open notes.txt for writing\n";
            return 1;
        }
        out << "first line\n" << 42 << ' ' << 3.5 << '\n';
    }                                                     // closed (and flushed) here

    std::ifstream in{"notes.txt"};
    if (!in) {
        std::cerr << "cannot open notes.txt\n";
        return 1;
    }
    std::string line;
    std::getline(in, line);                               // "first line"
    int n;
    double d;
    in >> n >> d;                                         // formatted extraction
    std::cout << line << " | " << n + 1 << " | " << d * 2 << '\n';   // first line | 43 | 7
}
```

A stream converts to `false` when it's in a failed state (couldn't open, bad input, end of file).
Check it after opening and after reading. The idiomatic read loop tests the read itself — never
`while (!in.eof())` (same bug as C day 19's `feof`):

```cpp
std::string line;
while (std::getline(in, line)) {
    // process line
}
```

### Parsing lines: std::istringstream

A string stream reads from a string as if it were a file — handy for splitting a line into fields:

```cpp
#include <iostream>
#include <sstream>
#include <string>
#include <vector>

struct Record {
    std::string name;
    int age = 0;
    double score = 0;
};

// Parses "name,age,score". Returns false on malformed input.
bool parse_record(const std::string &line, Record &r)
{
    std::istringstream ss{line};
    std::string age, score;
    if (!std::getline(ss, r.name, ',') || !std::getline(ss, age, ',') || !std::getline(ss, score)) {
        return false;
    }
    try {
        std::size_t pos;
        r.age = std::stoi(age, &pos);
        if (pos != age.size()) return false;
        r.score = std::stod(score, &pos);
        return pos == score.size();
    } catch (const std::exception &) {            // stoi/stod throw on garbage or overflow
        return false;
    }
}

int main()
{
    std::vector<Record> records;
    for (std::string line : {"Ada,36,9.5", "Alan,41,8.75", "bad line", "Grace,x,1"}) {
        Record r;
        if (parse_record(line, r)) records.push_back(r);
        else std::cerr << "skipping: " << line << '\n';
    }
    for (const auto &r : records) std::cout << r.name << ' ' << r.age << ' ' << r.score << '\n';
}
```

`std::ostringstream` builds strings with `<<` (though `std::format` is usually nicer).

For speed and strictness, `std::from_chars` (day 13) parses numbers without exceptions, locales
or allocations — the best choice for large files.

### Reading a whole file

```cpp
#include <fstream>
#include <sstream>
#include <stdexcept>
#include <string>

std::string read_file(const std::string &path)
{
    std::ifstream in{path, std::ios::binary};
    if (!in) throw std::runtime_error{"cannot open " + path};
    std::ostringstream ss;
    ss << in.rdbuf();                          // copy the whole stream buffer
    return ss.str();
}
```

### Binary I/O

```cpp
std::ofstream out{"data.bin", std::ios::binary};
std::vector<double> values = {1.5, 2.5, 3.5};
out.write(reinterpret_cast<const char *>(values.data()), values.size() * sizeof(double));

std::ifstream in{"data.bin", std::ios::binary};
std::vector<double> loaded(3);
in.read(reinterpret_cast<char *>(loaded.data()), loaded.size() * sizeof(double));
```

The same portability caveats as C day 19 apply: byte order, type sizes and padding. Only write
**trivially copyable** types this way (no `std::string`, no pointers); for anything else, write a
text format or serialize field by field.

### std::filesystem

`<filesystem>` (C++17) handles paths and directories portably:

```cpp
#include <cstdint>
#include <filesystem>
#include <iostream>
#include <map>
#include <string>

namespace fs = std::filesystem;

int main(int argc, char *argv[])
{
    fs::path root = argc > 1 ? argv[1] : ".";
    if (!fs::is_directory(root)) {
        std::cerr << root << " is not a directory\n";
        return 1;
    }

    std::map<std::string, std::uintmax_t> bytes_by_ext;
    std::error_code ec;
    for (const auto &entry : fs::recursive_directory_iterator(
             root, fs::directory_options::skip_permission_denied, ec)) {
        if (entry.is_regular_file(ec)) {
            auto size = entry.file_size(ec);
            if (ec) continue;                  // unreadable: skip it rather than abort
            auto ext = entry.path().extension().string();
            bytes_by_ext[ext.empty() ? "(none)" : ext] += size;
        }
    }
    for (const auto &[ext, bytes] : bytes_by_ext) {
        std::cout << ext << ": " << bytes << " bytes\n";
    }
}
```

Useful pieces:

| Code | Does |
|---|---|
| `fs::path p = dir / "file.txt";` | join paths with `/` (uses the right separator on each OS) |
| `p.filename()`, `p.stem()`, `p.extension()`, `p.parent_path()` | parts of a path |
| `fs::exists(p)`, `fs::is_regular_file(p)`, `fs::is_directory(p)` | queries |
| `fs::file_size(p)`, `fs::last_write_time(p)` | metadata |
| `fs::create_directories(p)` | `mkdir -p` |
| `fs::copy`, `fs::rename`, `fs::remove`, `fs::remove_all` | file operations |
| `fs::directory_iterator`, `fs::recursive_directory_iterator` | list directories |
| `fs::current_path()`, `fs::temp_directory_path()` | well-known locations |

Most functions come in two versions: one that **throws** `fs::filesystem_error`, and one taking a
`std::error_code &` that doesn't. Use the non-throwing version in loops over many files, where one
unreadable file shouldn't abort everything.

## Common mistakes

- Not checking whether the stream opened.
- `while (!in.eof())` loops.
- Mixing `>>` and `getline` (day 1's leftover-newline problem).
- Text mode for binary data on Windows (always `std::ios::binary`).
- Writing non-trivially-copyable objects with `write`.
- Building paths with string concatenation and `"\\"` or `"/"` instead of `fs::path` and `/`.

## Exercises

1. **CSV report:** read a CSV of `name,department,salary` (write one with 10 rows), skip and report
   malformed lines with their line numbers, and print the average salary per department with
   `std::format`.
2. **Word count tool:** `wc`-like counts for every file given on the command line, plus a total.
3. **Config parser:** `key = value` lines with `#` comments and `[section]` headers, into a
   `std::map<std::string, std::map<std::string, std::string>>`. Report errors with line numbers.
4. **Directory report:** the program above, extended to print the 10 largest files, sorted, with
   sizes formatted as KB/MB.
5. **Find duplicates:** find files with identical contents in a directory tree (group by size first,
   then compare contents or a hash of them).
6. **Save/load:** give your day 14 library project `save`/`load` in a robust text format using what
   you learned today, with a test that saves and reloads.
7. ★ **Binary format with a header:** write a file with a magic number, a version, a record count,
   then records written field by field in little-endian; read it back and validate every part.

## Check yourself

1. How do you check that a file stream opened successfully?
2. What's wrong with `while (!in.eof())`?
3. What's `std::istringstream` good for?
4. Which objects is it safe to write with `ostream::write`?
5. When would you use the `std::error_code` overloads of filesystem functions?

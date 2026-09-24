# C Day 19 — File I/O

**Goal:** read and write text and binary files, handle errors, and use the standard streams.

## Concepts

### Opening and closing

```c
#include <stdio.h>

int main(void)
{
    FILE *f = fopen("notes.txt", "w");      // open for writing (creates or truncates)
    if (f == NULL) {
        perror("notes.txt");                // prints e.g. "notes.txt: Permission denied"
        return 1;
    }
    fprintf(f, "line %d\n", 1);
    fputs("line 2\n", f);
    if (fclose(f) != 0) {                   // closing flushes buffered data; it can fail too
        perror("fclose");
        return 1;
    }
    return 0;
}
```

A `FILE *` is a pointer to an opaque structure the library manages (you'll build your own opaque
types on day 27). **Always check `fopen` for `NULL`** and always `fclose` what you open.

| Mode | Meaning |
|---|---|
| `"r"` | read; file must exist |
| `"w"` | write; creates, or **truncates** an existing file to zero length |
| `"a"` | append; writes go to the end |
| `"r+"` | read and write; file must exist |
| `"w+"`, `"a+"` | read and write, with `w`/`a` behavior |
| add `"b"` | binary mode: `"rb"`, `"wb"` |

On Linux, text and binary mode are identical. On Windows, text mode translates `\n` ↔ `\r\n` and
treats byte 26 as end-of-file — so **always use `"b"` for binary data**.

### The three standard streams

`stdin`, `stdout` and `stderr` are `FILE *`s opened for you. `printf(...)` is
`fprintf(stdout, ...)`. Print errors to `stderr` so they don't mix with real output when output is
redirected:

```bash
./prog > out.txt          # stdout to a file, errors still on screen
./prog < in.txt           # stdin from a file
./prog 2> errors.txt      # stderr to a file
```

### Reading text line by line

```c
#include <stdio.h>
#include <string.h>

int main(int argc, char *argv[])
{
    if (argc != 2) {
        fprintf(stderr, "usage: %s FILE\n", argv[0]);
        return 1;
    }
    FILE *f = fopen(argv[1], "r");
    if (f == NULL) {
        perror(argv[1]);
        return 1;
    }

    char line[1024];
    int number = 0;
    while (fgets(line, sizeof line, f) != NULL) {     // NULL at end of file or on error
        line[strcspn(line, "\n")] = '\0';
        printf("%4d: %s\n", ++number, line);
    }
    if (ferror(f)) {
        perror("read");
    }
    fclose(f);
    return 0;
}
```

(Lines longer than the buffer arrive in pieces; your `read_line` from day 12 handles any length.)

### Reading character by character

```c
int c;                                // int, not char!
while ((c = fgetc(f)) != EOF) {
    putchar(c);
}
```

`fgetc` returns an `int`: either a byte value 0–255, or `EOF` (a negative value). Storing it in a
`char` makes the byte 255 indistinguishable from `EOF` on some systems, or never matches `EOF` on
others. Always use `int`.

### Formatted reading: fscanf

`fscanf(f, "%d %lf", &n, &x)` works like `scanf`. It's fine for simple, trusted formats; for
anything a user edits, read lines with `fgets` and parse them (`sscanf` or `strtol` — day 20).

### Binary files: fread and fwrite

```c
#include <stdio.h>

typedef struct {
    int id;
    double price;
} Item;

int main(void)
{
    Item items[3] = {{1, 9.99}, {2, 19.5}, {3, 0.25}};

    FILE *out = fopen("items.bin", "wb");
    if (out == NULL) { perror("items.bin"); return 1; }
    size_t written = fwrite(items, sizeof items[0], 3, out);   // returns elements written
    fclose(out);
    if (written != 3) { fprintf(stderr, "write failed\n"); return 1; }

    Item loaded[3];
    FILE *in = fopen("items.bin", "rb");
    if (in == NULL) { perror("items.bin"); return 1; }
    size_t read = fread(loaded, sizeof loaded[0], 3, in);
    fclose(in);

    for (size_t i = 0; i < read; i++) {
        printf("%d %.2f\n", loaded[i].id, loaded[i].price);
    }
    return 0;
}
```

Writing structs directly is quick but **not portable**: padding, byte order and type sizes can
differ between compilers and machines, and pointer members are meaningless once saved. Text
formats (CSV, JSON) avoid those problems.

### Moving around: fseek, ftell, rewind

```c
fseek(f, 0, SEEK_END);     // go to the end
long size = ftell(f);      // current position = file size in bytes (binary mode)
rewind(f);                 // back to the start
```

## Common mistakes

- Not checking `fopen`'s result.
- `char c = fgetc(f)` instead of `int c`.
- `while (!feof(f))` loops — `feof` only becomes true *after* a read fails, so the last iteration
  processes garbage. Loop on the read function's return value instead, as above.
- Opening binary files without `"b"` on Windows.
- Forgetting `fclose` (data may stay in the buffer and never reach the disk).
- Writing structs containing pointers to a file.

## Exercises

1. **cat:** print the contents of each file named on the command line; report errors for files
   that can't be opened but continue with the rest.
2. **wc:** count lines, words and characters of a file (or of `stdin` if no file is given).
   Compare with the real `wc`.
3. **copy:** `./copy SRC DST` copies any file, in binary mode, using a 4096-byte buffer with
   `fread`/`fwrite`. Verify with `cmp` or `diff` on an image file.
4. **CSV:** write a file `people.csv` with lines like `Ada,36,London` by hand. Read it into an
   array of structs (use `strtok` or `sscanf("%31[^,],%d,%31[^\n]", ...)`), then print them sorted
   by age.
5. **Save/load:** give your day 15 `Player` list `save_players(const char *path, ...)` and
   `load_players(...)` functions in text format, one player per line.
6. **Hex dump:** print a file as 16 bytes per line, in hex, followed by the printable characters
   (`isprint`), like `xxd`.
7. Write the binary `items.bin` example, then open the file in your hex dump. Find the padding
   bytes between `id` and `price`.
8. ★ **tail:** print the last N lines of a file (`./tail 5 file.txt`). Hint: keep the last N lines
   in a ring of N heap strings.

## Check yourself

1. What happens when you open an existing file with `"w"`?
2. Why must the result of `fgetc` be stored in an `int`?
3. What's wrong with `while (!feof(f))`?
4. Why should errors go to `stderr`?
5. Why is `fwrite` of a struct not a portable file format?

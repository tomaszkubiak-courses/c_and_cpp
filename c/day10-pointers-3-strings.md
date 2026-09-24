# C Day 10 — Pointers III: strings

**Goal:** understand C strings as `char` arrays ending in `'\0'`, use `<string.h>` safely, and
write the classic string functions yourself with pointers.

## Concepts

### A string is a char array with a terminator

C has no string type. A string is a sequence of `char` ending with the **null character** `'\0'`
(the byte with value 0). Functions find the end of a string by looking for it.

```c
char word[] = "cat";
```

```text
 word: [ 'c' ][ 'a' ][ 't' ][ '\0' ]
         0      1      2      3        -> 4 bytes for a 3-letter word
```

- `sizeof word` is 4 (the array, including the terminator).
- `strlen(word)` is 3 (characters before the terminator).
- **Always leave room for the `'\0'`.** A buffer for up to N characters needs N + 1 bytes.

### Array vs pointer to a literal

```c
char a[] = "hello";        // an array of 6 chars, initialized with a COPY of the literal
const char *p = "hello";   // a pointer to the literal itself, which is read-only
```

```text
 a: [h][e][l][l][o][\0]          your own modifiable array (on the stack)

 p: [ ●-]--> [h][e][l][l][o][\0] the literal, in read-only memory
```

`a[0] = 'J';` is fine. Changing the literal through `p` is **undefined behavior** (it usually
crashes). C allows `char *p = "hello";` without `const` for historical reasons — always write
`const char *` for pointers to literals, so the compiler stops you from modifying them.

### Printing and reading

```c
#include <stdio.h>
#include <string.h>

int main(void)
{
    char name[32];

    printf("Your name: ");
    if (fgets(name, sizeof name, stdin) == NULL) {   // reads a line, at most 31 chars + '\0'
        return 1;
    }
    name[strcspn(name, "\n")] = '\0';   // fgets keeps the newline; cut it off
    printf("Hello, %s! Your name has %zu letters.\n", name, strlen(name));
    return 0;
}
```

- **Use `fgets`, not `scanf("%s")`**, for text. `scanf("%s", buf)` stops at the first space and
  has no idea how big `buf` is — typing a long word overflows it. (If you must use it, give a
  width: `scanf("%31s", buf)`.) Never use `gets` — it was removed from the language because it
  can't be used safely.
- `strcspn(s, "\n")` returns the index of the first `'\n'` (or the length if there's none) —
  a neat one-liner to strip the newline.

### The <string.h> essentials

| Function | Does | Watch out |
|---|---|---|
| `strlen(s)` | length, not counting `'\0'` | walks the whole string each call — don't put it in a loop condition on long strings |
| `strcmp(a, b)` | compare: `<0`, `0`, `>0` | returns **0 when equal**. Never compare strings with `==` (that compares addresses) |
| `strncmp(a, b, n)` | compare at most n chars | |
| `strcpy(dst, src)` | copy including `'\0'` | no size check — overflow if `dst` is too small |
| `strcat(dst, src)` | append | no size check |
| `strchr(s, c)` | pointer to first `c`, or NULL | |
| `strrchr(s, c)` | pointer to last `c`, or NULL | |
| `strstr(s, sub)` | pointer to first occurrence of `sub`, or NULL | |
| `memcpy(dst, src, n)` | copy n bytes (any data) | regions must not overlap |
| `memmove(dst, src, n)` | copy n bytes, overlap allowed | |
| `memset(p, byte, n)` | fill n bytes | |

**Safe copying and joining:** use `snprintf`, which always stays within the size you give it and
always adds `'\0'`:

```c
#include <stdio.h>

int main(void)
{
    char buf[16];
    const char *first = "Grace", *last = "Hopper";
    int needed = snprintf(buf, sizeof buf, "%s %s", first, last);
    if (needed >= (int)sizeof buf) {
        printf("(truncated)\n");
    }
    printf("%s\n", buf);
    return 0;
}
```

`strncpy` sounds safe but isn't: if the source is too long it does **not** add `'\0'`. Avoid it.

### Character tests: <ctype.h>

`isalpha`, `isdigit`, `isspace`, `isupper`, `islower`, `isalnum`, `ispunct`, `toupper`,
`tolower`. Cast the argument to `unsigned char` when it comes from a `char`, because passing a
negative value (possible for non-ASCII bytes) is undefined:
`isalpha((unsigned char)*p)`.

### Writing string functions with pointers

This is where pointer arithmetic pays off. Study these — they're the core of today:

```c
#include <stdio.h>

size_t my_strlen(const char *s)
{
    const char *p = s;
    while (*p != '\0') {    // often written as: while (*p)
        p++;
    }
    return (size_t)(p - s);   // pointer difference = number of chars
}

char *my_strcpy(char *dst, const char *src)
{
    char *start = dst;
    while ((*dst++ = *src++) != '\0') {
        // copy each char, including the final '\0', then stop
    }
    return start;
}

int my_strcmp(const char *a, const char *b)
{
    while (*a != '\0' && *a == *b) {
        a++;
        b++;
    }
    return (unsigned char)*a - (unsigned char)*b;
}

int main(void)
{
    char buf[20];
    my_strcpy(buf, "pointer");
    printf("%s %zu\n", buf, my_strlen(buf));                        // pointer 7
    printf("%d %d %d\n", my_strcmp("abc", "abd") < 0,
           my_strcmp("abc", "abc") == 0, my_strcmp("b", "a") > 0);  // 1 1 1
    return 0;
}
```

Trace `my_strcpy` for `src = "hi"` with a diagram: each loop iteration copies one char and moves
both pointers; the loop's test is the value just copied, so it stops right after copying `'\0'`.

## Common mistakes

- A buffer one byte too small: `char s[5] = "hello";` compiles in C (the `'\0'` is silently
  dropped!) and then every string function runs off the end. Use `char s[] = "hello";`.
- Comparing strings with `==`.
- Modifying a string literal.
- Forgetting that `fgets` keeps the `'\n'`.
- Using `strcpy`/`strcat` without knowing the destination is big enough.
- Forgetting to write the terminator when building a string by hand.

## Exercises

### A. Predict the output

1.
   ```c
   char s[] = "hello";
   const char *p = "hello";
   printf("%zu %zu %zu\n", sizeof s, strlen(s), sizeof p);
   ```

2.
   ```c
   char s[10] = "abc";
   printf("%zu %zu\n", sizeof s, strlen(s));
   s[1] = '\0';
   printf("%s %zu\n", s, strlen(s));
   ```

3.
   ```c
   const char *s = "programming";
   printf("%s\n", s + 3);
   printf("%c\n", *(s + 3));
   printf("%c\n", s[strlen(s) - 1]);
   ```

4.
   ```c
   char a[] = "abc", b[] = "abc";
   printf("%d %d\n", a == b, strcmp(a, b) == 0);
   ```

5.
   ```c
   const char *path = "docs/notes/today.txt";
   const char *slash = strrchr(path, '/');
   const char *dot = strchr(path, '.');
   printf("%s | %s | %td\n", slash + 1, dot, dot - path);
   ```

<details><summary>Answers</summary>

1. `6 5 8` — the array holds 6 bytes; `sizeof p` is the size of a pointer.
2. `10 3` then `a 1` — `sizeof` is the array size, `strlen` stops at the first `'\0'`.
3. `gramming`, `g`, `g` — `s + 3` is a pointer into the middle of the string, and `%s` prints from
   there.
4. `0 1` — `a` and `b` are different arrays at different addresses; their contents are equal.
5. `today.txt | .txt | 16`

</details>

### B. Write it — with pointers, not indexes

1. `size_t count_char(const char *s, char c)`.
2. `void reverse_string(char *s)` — in place, using two pointers moving toward each other.
3. `int is_palindrome(const char *s)` — ignore case and non-letters: "A man, a plan, a canal:
   Panama" is a palindrome.
4. `void to_upper_str(char *s)`.
5. `char *my_strcat(char *dst, const char *src)` — find the end of `dst` first, then copy.
6. `const char *my_strchr(const char *s, int c)` — return a pointer to the first `c`, or NULL.
7. `size_t count_words(const char *s)` — words are separated by any amount of whitespace.
8. `void trim(char *s)` — remove leading and trailing spaces in place (hint: find the first
   non-space, find the last non-space, `memmove` the middle part to the front, and terminate).
9. `char *my_strstr(const char *haystack, const char *needle)` — return a pointer to the first
   occurrence or NULL. (Return type is `char *` like the standard one; you'll need a cast.)
10. ★ `int my_atoi(const char *s)` — convert `"-123"` to -123: optional sign, then digits
    (`*p - '0'`), stop at the first non-digit.

### C. Find the bug

1.
   ```c
   // BUG
   #include <stdio.h>
   #include <string.h>
   int main(void)
   {
       char greeting[5];
       strcpy(greeting, "hello");
       printf("%s\n", greeting);
       return 0;
   }
   ```

2.
   ```c
   // BUG
   #include <stdio.h>
   int main(void)
   {
       char *s = "hello";
       s[0] = 'H';
       printf("%s\n", s);
       return 0;
   }
   ```

3.
   ```c
   // BUG
   #include <stdio.h>
   int main(void)
   {
       char answer[8];
       printf("Continue? ");
       fgets(answer, sizeof answer, stdin);
       if (answer == "yes") {
           printf("Continuing\n");
       }
       return 0;
   }
   ```

4.
   ```c
   // BUG
   #include <stdio.h>
   int main(void)
   {
       char letters[3];
       for (int i = 0; i < 3; i++) {
           letters[i] = 'a' + i;
       }
       printf("%s\n", letters);
       return 0;
   }
   ```

<details><summary>Answers</summary>

1. "hello" needs 6 bytes. `char greeting[6]` — or better, `snprintf(greeting, sizeof greeting, "hello")`.
2. Modifying a string literal. Use `char s[] = "hello";`.
3. `==` compares addresses. Strip the newline, then `strcmp(answer, "yes") == 0`.
4. No `'\0'`, so `printf` keeps reading past the array. Make it `char letters[4]` and set
   `letters[3] = '\0'`.

</details>

## Check yourself

1. How many bytes does the string `"C99"` occupy?
2. Why is `char *s = "text"; s[0] = 'T';` wrong, while `char s[] = "text"; s[0] = 'T';` is fine?
3. What does `strcmp` return for equal strings?
4. Why prefer `fgets` over `scanf("%s")` and `snprintf` over `strcpy`?
5. What does `p - s` give you when both point into the same string?

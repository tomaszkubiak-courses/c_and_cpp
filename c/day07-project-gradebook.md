# C Day 7 — Week 1 project: grade book

**Goal:** combine everything from days 1–6 in one program of about 150 lines, organized into
functions. Then review the week.

## The project

Write a console grade book that stores up to 50 scores (integers 0–100) and offers a menu:

```text
=== Grade book (12 scores) ===
1) Add scores
2) Show all scores
3) Statistics
4) Histogram
5) Remove a score
0) Quit
>
```

### Requirements

1. **Add scores:** read scores until the user enters −1. Reject values outside 0–100 with a
   message. Refuse to add when the book is full (50).
2. **Show all:** print the scores 10 per line, each right-aligned in 4 characters, with the
   index in front of each line.
3. **Statistics:** count, minimum, maximum, mean (2 decimals) and **median**. For the median,
   copy the scores to a second array, sort the copy (write a simple sort — see below), and take
   the middle element (or the average of the two middle elements when the count is even).
4. **Histogram:** buckets 0–9, 10–19, ..., 90–100, with one `*` per score:

   ```text
    0- 9 |
   10-19 | *
   ...
   80-89 | *****
   90-100| ***
   ```

5. **Remove:** ask for an index and remove that score, shifting the later ones left.
6. Every menu action is its own function. `main` only holds the array, the count and the menu
   loop.
7. Handle bad input: if `scanf` doesn't return 1, print a message and quit.

### Suggested function signatures

```c
size_t add_scores(int scores[], size_t count, size_t capacity);   // returns the new count
void show_scores(const int scores[], size_t count);
void print_statistics(const int scores[], size_t count);
void print_histogram(const int scores[], size_t count);
size_t remove_score(int scores[], size_t count, size_t index);     // returns the new count
void sort_ascending(int arr[], size_t n);
```

Notice that `add_scores` has to *return* the new count, because it can't change the caller's
`count` variable (day 5: pass by value). On day 8 you'll learn the other way to do it.

### A simple sort (insertion sort)

You'll learn sorting properly on day 24. Here's one to use now — read it and trace it by hand on
`{5, 2, 4, 1}`:

```c
void sort_ascending(int arr[], size_t n)
{
    for (size_t i = 1; i < n; i++) {
        int key = arr[i];
        size_t j = i;
        while (j > 0 && arr[j - 1] > key) {   // shift bigger elements right
            arr[j] = arr[j - 1];
            j--;
        }
        arr[j] = key;                          // drop key into the gap
    }
}
```

### Extensions (★)

- Store a letter grade next to each score (A ≥ 90, B ≥ 75, C ≥ 50, else F) and print the count of
  each letter.
- Add "5) Load demo data" that fills the book with 30 random scores.
- Print the standard deviation (`sqrt` from `<math.h>`, link with `-lm`).

## Week 1 review

Answer these without looking back. If you can't, reread that day's lesson.

1. What does the linker do? Give an example of a linker error. (Day 1)
2. Which `printf` specifiers print a `size_t`, a `double`, a `long long`? (Day 2)
3. Why is `(double)sum / count` different from `(double)(sum / count)`? (Day 3)
4. What does `5 & 3` evaluate to? (Day 3)
5. Write a loop that runs exactly `n` times. (Day 4)
6. Why can't `void swap(int a, int b)` swap its arguments? (Day 5)
7. What's the lifetime of a `static` local variable? (Day 5)
8. What happens when you write to `a[10]` in an array of 10 elements? (Day 6)
9. Why must you pass an array's length to a function separately? (Day 6)

Before moving on, make sure your grade book compiles **without any warnings** using
`-Wall -Wextra -Wpedantic`, and runs cleanly with `-fsanitize=address,undefined`.

**Next week is pointers** — the most important week of this course. Get a good night's sleep.

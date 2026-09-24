# C Day 24 — Sorting algorithms

**Goal:** implement the classic sorting algorithms, understand why some are O(n²) and others
O(n log n), and know what "stable" means.

Watch each algorithm animate on <https://visualgo.net/en/sorting> before implementing it.

## Concepts

### Why learn sorting when qsort exists?

You'll almost always use the library sort (`qsort` in C, `std::sort` in C++). But sorting
algorithms are the best introduction to algorithm design: the simple ones show loops and
invariants, merge sort shows **divide and conquer** and recursion, quicksort shows partitioning
and average-case analysis.

### Selection sort — O(n²)

Find the smallest element, swap it to the front; repeat on the rest.

```c
void selection_sort(int *a, size_t n)
{
    for (size_t i = 0; i + 1 < n; i++) {
        size_t min = i;
        for (size_t j = i + 1; j < n; j++) {
            if (a[j] < a[min]) min = j;
        }
        int tmp = a[i]; a[i] = a[min]; a[min] = tmp;
    }
}
```

Always n²/2 comparisons, but at most n swaps.

### Insertion sort — O(n²), but O(n) on nearly sorted data

You met it on day 7. Take each element and slide it left into its place among the already-sorted
prefix. It's very fast for small or nearly sorted arrays, which is why real library sorts switch to
it for small ranges.

### Bubble sort — O(n²)

Repeatedly swap adjacent elements that are out of order. It's the famous one and the worst in
practice; implement it once (exercise 1) and then forget it.

### Merge sort — O(n log n), stable, needs extra memory

**Divide and conquer:** split the array in half, sort each half (recursively), then **merge** the
two sorted halves by repeatedly taking the smaller front element.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

// Merges sorted a[lo..mid) and a[mid..hi) using tmp as scratch space.
static void merge(int *a, int *tmp, size_t lo, size_t mid, size_t hi)
{
    size_t i = lo, j = mid, k = lo;
    while (i < mid && j < hi) {
        tmp[k++] = (a[j] < a[i]) ? a[j++] : a[i++];   // on ties take the left one: stable
    }
    while (i < mid) tmp[k++] = a[i++];
    while (j < hi) tmp[k++] = a[j++];
    memcpy(a + lo, tmp + lo, (hi - lo) * sizeof *a);
}

static void merge_sort_rec(int *a, int *tmp, size_t lo, size_t hi)
{
    if (hi - lo < 2) return;                   // 0 or 1 element: already sorted
    size_t mid = lo + (hi - lo) / 2;
    merge_sort_rec(a, tmp, lo, mid);
    merge_sort_rec(a, tmp, mid, hi);
    merge(a, tmp, lo, mid, hi);
}

int merge_sort(int *a, size_t n)
{
    int *tmp = malloc(n * sizeof *tmp);
    if (n > 0 && tmp == NULL) return -1;
    merge_sort_rec(a, tmp, 0, n);
    free(tmp);
    return 0;
}

int main(void)
{
    int a[] = {38, 27, 43, 3, 9, 82, 10};
    size_t n = sizeof a / sizeof a[0];
    merge_sort(a, n);
    for (size_t i = 0; i < n; i++) printf("%d ", a[i]);
    printf("\n");   // 3 9 10 27 38 43 82
    return 0;
}
```

Why O(n log n)? The array is halved log₂ n times, and at each level of the recursion, all merges
together touch every element once — O(n) per level.

### Quicksort — O(n log n) on average, O(n²) worst case, in place

Pick a **pivot**, **partition** the array so everything smaller is left of it and everything larger
is right of it, then recursively sort both sides.

```c
static void swap_int(int *x, int *y) { int t = *x; *x = *y; *y = t; }

// Lomuto partition of a[lo..hi] (inclusive) around the last element. Returns the pivot's index.
static size_t partition(int *a, size_t lo, size_t hi)
{
    int pivot = a[hi];
    size_t store = lo;
    for (size_t i = lo; i < hi; i++) {
        if (a[i] < pivot) {
            swap_int(&a[i], &a[store]);
            store++;
        }
    }
    swap_int(&a[store], &a[hi]);
    return store;
}

void quick_sort(int *a, size_t lo, size_t hi)   // sorts a[lo..hi] inclusive
{
    if (lo >= hi) return;
    size_t p = partition(a, lo, hi);
    if (p > lo) quick_sort(a, lo, p - 1);       // careful: p - 1 underflows if p == 0
    quick_sort(a, p + 1, hi);
}
/* call: if (n > 0) quick_sort(a, 0, n - 1); */
```

If the pivot is always the smallest or largest element (e.g. already-sorted input with this
last-element pivot), each partition removes only one element → O(n²). Real implementations pick
the pivot more cleverly (median of three, or random) and fall back to heap sort if recursion gets
too deep ("introsort" — what `std::sort` uses).

### Stability

A sort is **stable** if equal elements keep their original order. It matters when you sort records
by one key after another: sort people by name, then *stably* by age, and people with the same age
stay alphabetical. Merge sort and insertion sort are stable; selection sort and quicksort (as
above) aren't. The C standard doesn't promise that `qsort` is stable.

### Counting sort — O(n + k) for small integer ranges

If values are integers in a small range `[0, k)`, count how many of each there are and write them
back in order. No comparisons at all — which is why it can beat O(n log n).

## Summary

| Algorithm | Best | Average | Worst | Extra memory | Stable |
|---|---|---|---|---|---|
| Selection | n² | n² | n² | O(1) | no |
| Insertion | n | n² | n² | O(1) | yes |
| Bubble | n | n² | n² | O(1) | yes |
| Merge | n log n | n log n | n log n | O(n) | yes |
| Quick | n log n | n log n | n² | O(log n) stack | no |
| Counting | n + k | n + k | n + k | O(k) | yes (if done carefully) |

## Exercises

1. Implement bubble sort, with the improvement that it stops early when a pass makes no swaps.
2. Type in selection, insertion, merge and quick sort. Write a test function that fills an array
   with random numbers, sorts it, and checks it's sorted (`a[i-1] <= a[i]`). Test sizes 0, 1, 2,
   10 and 1000, plus arrays that are already sorted, reversed, and all equal.
3. **Benchmark:** time every algorithm and `qsort` on random arrays of 1,000, 10,000 and 100,000
   elements (compile with `-O2`). Make a table. Where does O(n²) become painful?
4. Time your quicksort on an **already sorted** array of 50,000 elements. Then change the pivot to
   a random element (swap it to the end first) and time again.
5. Make your insertion sort generic, with the same signature as `qsort` (day 13, exercise 7), and
   use it to sort an array of `Player` structs by score. Verify it's stable: sort by name first,
   then by score.
6. Implement counting sort for exam scores 0–100, and compare its speed with `qsort` on 10⁷
   scores.
7. ★ Write merge sort for a **linked list** (day 21): split with slow/fast pointers, sort both
   halves, merge by relinking. It needs no extra array.
8. ★ Implement **heap sort** (read about binary heaps first; VisuAlgo has an animation).

## Check yourself

1. Why is merge sort O(n log n)?
2. What input makes a last-element-pivot quicksort O(n²)?
3. What does "stable" mean, and why does it matter?
4. Why do real sort implementations switch to insertion sort for small ranges?
5. How can counting sort beat O(n log n)?

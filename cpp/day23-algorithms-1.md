# C++ Day 23 — Algorithms I: complexity, searching, and classic techniques

**Goal:** solve algorithmic problems in C++ using the right standard tools, and learn the core
techniques that most array and string problems reduce to: sorting first, binary search, two
pointers, sliding window, prefix sums and hashing.

If you did the C course, days 23–24 there covered Big-O, binary search and sorting from scratch.
Today builds on that with C++'s library and a problem-solving mindset. If you didn't, read C day 23
first.

## Concepts

### Complexity of the standard library

Know these costs — they decide whether a solution is O(n) or accidentally O(n²):

| Operation | Cost |
|---|---|
| `std::sort`, `std::stable_sort` | O(n log n) |
| `std::nth_element` | O(n) average |
| `std::lower_bound` / `binary_search` on sorted random-access range | O(log n) |
| `std::find`, `std::count`, `accumulate` | O(n) |
| `vector::push_back` | O(1) amortized |
| `vector::insert`/`erase` in the middle | O(n) |
| `set`/`map` insert, find, erase | O(log n) |
| `unordered_set`/`unordered_map` insert, find, erase | O(1) average, O(n) worst |
| `priority_queue` push/pop | O(log n) |
| `std::string::find` | O(n·m) worst case |

A practical limit: about 10⁸ simple operations per second. With n = 10⁵, O(n²) = 10¹⁰ is too slow;
O(n log n) ≈ 1.7 × 10⁶ is instant. **Read the input size first; it tells you which complexity you
need.**

### Technique 1: sort first

Many problems get easy on sorted data: duplicates are adjacent, the closest pair is adjacent, and
binary search works.

```cpp
// Does the vector contain duplicates? O(n log n)
bool has_duplicates(std::vector<int> v)          // by value: we sort our copy
{
    std::ranges::sort(v);
    return std::ranges::adjacent_find(v) != v.end();
}
```

### Technique 2: hashing for O(1) lookups

The classic "two sum": find two numbers that add up to a target. Nested loops are O(n²). With a hash
map from value to index, one pass is O(n):

```cpp
#include <iostream>
#include <optional>
#include <unordered_map>
#include <utility>
#include <vector>

std::optional<std::pair<std::size_t, std::size_t>> two_sum(const std::vector<int> &v, int target)
{
    std::unordered_map<int, std::size_t> seen;          // value -> index
    for (std::size_t i = 0; i < v.size(); ++i) {
        if (auto it = seen.find(target - v[i]); it != seen.end()) {
            return std::pair{it->second, i};
        }
        seen[v[i]] = i;
    }
    return std::nullopt;
}

int main()
{
    if (auto r = two_sum({2, 7, 11, 15}, 26)) {
        std::cout << r->first << ' ' << r->second << '\n';   // 2 3
    }
}
```

Pattern: "for each element, have I already seen its complement / partner?" → store what you've seen
in a hash set or map.

### Technique 3: two pointers

On a sorted array (or for in-place rearranging), move two indexes toward each other or in the same
direction. C day 23 did pair-with-sum; another classic — remove duplicates from a sorted array in
place:

```cpp
std::size_t dedupe_sorted(std::vector<int> &v)   // returns the new length
{
    if (v.empty()) return 0;
    std::size_t write = 1;
    for (std::size_t read = 1; read < v.size(); ++read) {
        if (v[read] != v[write - 1]) v[write++] = v[read];
    }
    v.resize(write);
    return write;
}
```

(That's exactly what `std::unique` does.)

### Technique 4: sliding window

For "best contiguous subarray/substring with some property", keep a window `[left, right)` and
slide it, updating a running summary instead of recomputing:

```cpp
#include <algorithm>
#include <iostream>
#include <string>
#include <unordered_map>

// Length of the longest substring without repeating characters. O(n)
std::size_t longest_unique(const std::string &s)
{
    std::unordered_map<char, std::size_t> last_pos;
    std::size_t best = 0, left = 0;
    for (std::size_t right = 0; right < s.size(); ++right) {
        if (auto it = last_pos.find(s[right]); it != last_pos.end() && it->second >= left) {
            left = it->second + 1;                     // shrink past the previous occurrence
        }
        last_pos[s[right]] = right;
        best = std::max(best, right - left + 1);
    }
    return best;
}

int main()
{
    std::cout << longest_unique("abcabcbb") << ' ' << longest_unique("pwwkew") << '\n';   // 3 3
}
```

### Technique 5: prefix sums

`prefix[i]` = sum of the first `i` elements; any range sum is a subtraction (C day 23).
`std::partial_sum` or `std::inclusive_scan` compute them. Combined with a hash map it solves
"count subarrays with sum k" in O(n) (exercise 6).

### Technique 6: binary search on the answer

If you can check "is X achievable?" and achievability is monotonic (if X works, anything bigger
works too), binary-search for the smallest X. Example: the minimum capacity to ship packages within
D days — check a capacity in O(n), binary-search over capacities.

### Problem-solving routine

1. Restate the problem; work out 2–3 examples by hand, including edge cases.
2. Look at the constraints → target complexity.
3. Brute force first (it's also your test oracle).
4. Look for a pattern: sorted? → binary search / two pointers. "Seen before?" → hash. Contiguous? →
   sliding window / prefix sums. "Minimum X such that..." → binary search on the answer.
5. Code it, test against brute force on random inputs.

## Exercises

Write each as a function with Catch2 tests (day 22), and compare against a brute-force version on
random inputs.

1. `bool has_duplicates(std::vector<int>)` three ways: nested loops, sort, `unordered_set`. Time
   them for n = 10⁴ and 10⁶.
2. **Two sum** as above; then **three sum**: all unique triplets summing to 0 (sort + two pointers,
   O(n²)).
3. **Valid anagram**, **group anagrams** (hash map keyed by sorted string).
4. **Maximum sum of k consecutive elements** with a sliding window.
5. **Longest substring with at most 2 distinct characters** (sliding window + counts).
6. **Count subarrays with sum k** (prefix sums + `unordered_map<long long, int>` of prefix counts).
7. **Merge overlapping intervals**: sort by start, then sweep.
8. **Kth largest element** three ways: sort, `std::nth_element`, and a min-heap of size k
   (`priority_queue` with `std::greater`). Which is best when k is small and n huge?
9. **Minimum capacity to ship within D days** (binary search on the answer).
10. ★ Solve 5 problems tagged "Easy" and 2 tagged "Medium" on LeetCode (arrays, hashing, two
    pointers, sliding window), in C++.

## Check yourself

1. With n = 10⁶, which complexities are acceptable?
2. Which technique fits "have I seen the complement before"?
3. When does the sliding window technique apply?
4. What property must hold to binary-search on the answer?
5. Why write a brute-force solution first?

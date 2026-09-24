# C++ Day 17 — The standard containers

**Goal:** know every standard container, what each operation costs, and how to choose — and see
that the C data structures you built (C days 21–26) are all here, ready-made.

## Concepts

### The families

| Kind | Containers | Built like (C course) |
|---|---|---|
| Sequence | `vector`, `deque`, `list`, `forward_list`, `array` | IntVec (C 14), linked lists (C 21–22) |
| Ordered associative | `set`, `map`, `multiset`, `multimap` | balanced BST (C 26) — a red-black tree |
| Unordered associative | `unordered_set`, `unordered_map`, and multi- versions | hash table with chaining (C 25) |
| Adapters | `stack`, `queue`, `priority_queue` | a restricted interface over another container (C 22) |

### Costs

| Container | Access by index | Find | Insert/erase at end | Insert/erase in middle | Notes |
|---|---|---|---|---|---|
| `vector` | O(1) | O(n) (O(log n) if sorted) | O(1) amortized | O(n) | contiguous; the default |
| `deque` | O(1) | O(n) | O(1) at both ends | O(n) | chunks; no reallocation of existing elements on push |
| `list` | — | O(n) | O(1) | O(1) given an iterator | doubly linked; stable iterators |
| `set`/`map` | — | O(log n) | O(log n) | O(log n) | sorted; stable iterators |
| `unordered_set`/`_map` | — | O(1) average | O(1) average | O(1) average | unordered; rehashing invalidates iterators |
| `priority_queue` | top only O(1) | — | push/pop O(log n) | — | a binary heap on a vector |

### Why vector wins more often than Big-O suggests

Modern CPUs read memory in 64-byte **cache lines** and prefetch sequential data. Walking a
`vector` touches memory in order; walking a `list` jumps to a random heap address per node, missing
the cache each time. So even for "insert in the middle", a `vector` often beats a `list` up to
thousands of elements. **Default to `vector`; switch only when measurements say so**, or when you
need a guarantee only another container gives (stable addresses, sorted order).

### map and unordered_map

```cpp
#include <iostream>
#include <map>
#include <string>
#include <unordered_map>

int main()
{
    std::map<std::string, int> ages;           // sorted by key
    ages["Ada"] = 36;                          // operator[] inserts if missing (value-initialized)
    ages["Alan"] = 41;
    ages.insert({"Grace", 85});                // doesn't overwrite an existing key
    ages.insert_or_assign("Ada", 37);          // does overwrite
    ages.try_emplace("Linus", 28);             // constructs in place only if missing

    if (ages.contains("Grace")) {              // C++20
        std::cout << "Grace is " << ages.at("Grace") << '\n';   // at() throws if missing
    }
    if (auto it = ages.find("Nobody"); it == ages.end()) {
        std::cout << "not found\n";
    }
    for (const auto &[name, age] : ages) {     // in key order: Ada, Alan, Grace, Linus
        std::cout << name << ": " << age << '\n';
    }

    std::unordered_map<std::string, int> counts;   // hash table: faster, unordered
    for (std::string w : {"a", "b", "a", "c", "a"}) {
        ++counts[w];                           // the classic counting idiom
    }
    std::cout << counts["a"] << '\n';          // 3
    std::erase_if(ages, [](const auto &kv) { return kv.second > 40; });   // C++20
    std::cout << ages.size() << '\n';          // 2
}
```

**Careful with `operator[]` on maps:** reading `m["missing"]` *inserts* a default value. Use
`find`, `contains` or `at` when you only want to look.

`map` when you need sorted order, range queries (`lower_bound`, "all keys between A and C"), or
stable iterators; `unordered_map` for pure lookups (usually 2–5× faster).

### set

A set of unique keys: `std::set<int> s{3, 1, 3, 2};` holds `{1, 2, 3}`. `insert` returns a pair
`{iterator, bool inserted}`.

### Custom keys

- For `map`/`set`: the key needs `<` (or pass a comparator type). A defaulted `operator<=>` (day 10)
  is enough.
- For `unordered_map`/`unordered_set`: the key needs `==` and a hash:

```cpp
struct Point {
    int x, y;
    bool operator==(const Point &) const = default;
};

struct PointHash {
    std::size_t operator()(const Point &p) const noexcept
    {
        return std::hash<int>{}(p.x) * 31 + std::hash<int>{}(p.y);
    }
};

std::unordered_set<Point, PointHash> visited;
```

### Adapters

```cpp
#include <iostream>
#include <queue>
#include <stack>
#include <vector>

int main()
{
    std::stack<int> st;             // LIFO; push, pop, top
    std::queue<int> q;              // FIFO; push, pop, front, back
    std::priority_queue<int> pq;    // largest on top; push, pop, top

    for (int x : {5, 1, 8, 3}) { st.push(x); q.push(x); pq.push(x); }
    std::cout << st.top() << ' ' << q.front() << ' ' << pq.top() << '\n';   // 3 5 8

    // smallest on top: pass the underlying container and std::greater
    std::priority_queue<int, std::vector<int>, std::greater<>> min_heap;
    for (int x : {5, 1, 8, 3}) min_heap.push(x);
    std::cout << min_heap.top() << '\n';   // 1
}
```

Note `pop()` returns nothing; read `top()` first. The priority queue is the key to Dijkstra's
algorithm (day 25).

### Iterator invalidation, per container

(Day 9 showed the bugs; this is the reference.)

| Container | Insert invalidates | Erase invalidates |
|---|---|---|
| `vector` | all, if it reallocates; otherwise those after the insertion point | the erased element and everything after |
| `deque` | all iterators (references stay valid when inserting at the ends) | depends; assume all |
| `list`, `set`, `map` | nothing | only the erased element |
| `unordered_*` | all iterators if it rehashes (references stay valid) | only the erased element |

## Common mistakes

- `std::list` "because we insert in the middle", without measuring.
- `m[key]` to check for existence (it inserts).
- A custom `<` that isn't a strict weak ordering (e.g. `<=`) — `map` and `sort` misbehave.
- Forgetting that `unordered_map` iteration order is unspecified and changes after rehashing.
- Holding iterators across modifications.

## Exercises

1. **Word frequency, again:** read a text file, count words with `std::unordered_map<std::string, int>`,
   then copy to a `std::vector<std::pair<std::string, int>>` and sort by count to print the top 10.
   Compare the amount of code with your C day 25 version.
2. Same problem with `std::map`. Print words alphabetically with their counts. Time both versions
   on a large text (compile with `-O2`).
3. **Anagram groups:** read words, group them by their sorted letters in a
   `std::map<std::string, std::vector<std::string>>`, print groups of size > 1.
4. **Benchmark:** insert 10⁵ random ints at random positions in a `vector` and a `list` (for the
   list, walk to the position first). Then sum all elements. Which wins?
5. Use a `std::set<int>` to find the distinct values of an array, and `lower_bound`/`upper_bound`
   to count values in a range [a, b].
6. **Task scheduler:** a `std::priority_queue` of `struct Task { int priority; std::string name; }`
   with a custom comparator; pop and print tasks in priority order, ties in insertion order (hint:
   add a sequence number).
7. Store `Point`s in an `unordered_set` with the custom hash above; use it to detect when a random
   walk on a grid revisits a cell.
8. ★ Implement an LRU cache of capacity N: `std::list<std::pair<Key, Value>>` for recency order
   plus an `std::unordered_map<Key, list::iterator>` for lookup. Why is `list` the right choice
   here? (Hint: which iterators stay valid?)

## Check yourself

1. What data structures underlie `map` and `unordered_map`?
2. Why is `vector` often faster than `list` even for middle insertions?
3. What's the trap with `map::operator[]`?
4. What does an `unordered_map` key type need?
5. Which containers keep iterators valid on insertion?

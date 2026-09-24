# C++ Day 24 — Algorithms II: recursion, backtracking, dynamic programming, greedy

**Goal:** solve problems by breaking them into smaller ones: recursion and backtracking for
exploring choices, dynamic programming for overlapping subproblems, and greedy algorithms when a
local choice is provably safe.

## Concepts

### Recursion and divide and conquer

A recursive solution: solve a smaller instance of the same problem, and combine. You've done merge
sort (C day 24) and tree traversals (C day 26). Pattern for divide and conquer:

1. **Divide** the input (usually in halves).
2. **Conquer** each part recursively.
3. **Combine** the results.

Fast exponentiation — compute xⁿ in O(log n) multiplications:

```cpp
long long power(long long x, unsigned n)
{
    if (n == 0) return 1;
    long long half = power(x, n / 2);
    return (n % 2 == 0) ? half * half : half * half * x;
}
```

### Backtracking: try, recurse, undo

For "find all / any combination satisfying constraints": build a candidate step by step; at each
step try every option, recurse, then **undo** the choice. Abandon a branch as soon as it can't lead
to a solution (pruning).

```cpp
#include <iostream>
#include <vector>

// All subsets of nums.
void subsets(const std::vector<int> &nums, std::size_t i, std::vector<int> &current,
             std::vector<std::vector<int>> &out)
{
    if (i == nums.size()) {
        out.push_back(current);
        return;
    }
    subsets(nums, i + 1, current, out);          // choice 1: skip nums[i]
    current.push_back(nums[i]);                  // choice 2: take nums[i]
    subsets(nums, i + 1, current, out);
    current.pop_back();                          // undo
}

int main()
{
    std::vector<std::vector<int>> out;
    std::vector<int> current;
    subsets({1, 2, 3}, 0, current, out);
    for (const auto &s : out) {
        std::cout << '{';
        for (int x : s) std::cout << x;
        std::cout << "} ";
    }
    std::cout << '\n';   // {} {3} {2} {23} {1} {13} {12} {123}
}
```

There are 2ⁿ subsets and n! permutations, so backtracking is exponential; pruning is what makes it
usable. Classic problems: permutations (`std::next_permutation` also generates them), N-Queens,
Sudoku, combination sum, word search in a grid.

### Dynamic programming: remember subproblem answers

C day 5's recursive Fibonacci was exponential because it recomputed the same values over and over.
If a recursive solution has **overlapping subproblems** and **optimal substructure** (the best
answer is built from best answers to subproblems), store each subproblem's answer:

- **Top-down (memoization):** keep the recursion, cache results.
- **Bottom-up (tabulation):** fill a table from the smallest subproblems up, with loops.

```cpp
#include <cstdint>
#include <iostream>
#include <unordered_map>
#include <vector>

std::uint64_t fib_memo(int n, std::unordered_map<int, std::uint64_t> &memo)
{
    if (n <= 1) return static_cast<std::uint64_t>(n);
    if (auto it = memo.find(n); it != memo.end()) return it->second;
    return memo[n] = fib_memo(n - 1, memo) + fib_memo(n - 2, memo);
}

// Fewest coins to make `amount`, or -1. Bottom-up: best[a] = 1 + min(best[a - coin]).
int min_coins(const std::vector<int> &coins, int amount)
{
    const int INF = amount + 1;
    std::vector<int> best(amount + 1, INF);
    best[0] = 0;
    for (int a = 1; a <= amount; ++a) {
        for (int c : coins) {
            if (c <= a && best[a - c] + 1 < best[a]) best[a] = best[a - c] + 1;
        }
    }
    return best[amount] >= INF ? -1 : best[amount];
}

int main()
{
    std::unordered_map<int, std::uint64_t> memo;
    std::cout << fib_memo(90, memo) << '\n';               // 2880067194370816120, instantly
    std::cout << min_coins({1, 5, 10, 25}, 63) << '\n';     // 6 (25+25+10+1+1+1)
    std::cout << min_coins({2}, 3) << '\n';                 // -1
}
```

How to design a DP:

1. Define the **state**: what does `dp[i]` (or `dp[i][j]`) mean, in words? ("the fewest coins to
   make amount i").
2. Write the **transition**: how `dp[i]` follows from smaller states.
3. Set the **base cases**.
4. Decide the **order** of computation, and where the answer is.

Classic DP problems: climbing stairs, coin change, longest increasing subsequence, longest common
subsequence (`dp[i][j]` over two strings), edit distance, 0/1 knapsack, unique paths in a grid.

### Greedy algorithms

A greedy algorithm makes the locally best choice at each step and never reconsiders. It's fast and
simple — but only correct when you can argue that the local choice is always safe.

**Interval scheduling:** choose the maximum number of non-overlapping meetings. Greedy rule: always
take the meeting that **ends first**. (Taking the shortest, or the one that starts first, fails —
find counterexamples!)

```cpp
#include <algorithm>
#include <iostream>
#include <vector>

struct Meeting { int start, end; };

int max_meetings(std::vector<Meeting> m)
{
    std::ranges::sort(m, {}, &Meeting::end);
    int count = 0, free_at = -1;
    for (const auto &x : m) {
        if (x.start >= free_at) {
            ++count;
            free_at = x.end;
        }
    }
    return count;
}

int main()
{
    std::cout << max_meetings({{1, 4}, {3, 5}, {0, 6}, {5, 7}, {3, 9}, {5, 9}, {6, 10}, {8, 11}}) << '\n';   // 3
}
```

Coin change with US coins is greedy-solvable (take the biggest coin that fits), but with coins
{1, 3, 4} and amount 6, greedy gives 4+1+1 (3 coins) while the optimum is 3+3 — which is why the
general version needs DP.

## Exercises

1. Generate all **permutations** of a string with backtracking; compare with `std::next_permutation`.
2. **N-Queens:** count the solutions for n = 1..10 (answers: 1, 0, 0, 2, 10, 4, 40, 92, 352, 724).
   Prune using sets of used columns and diagonals.
3. **Sudoku solver** with backtracking. Read a puzzle from a file of 9 lines.
4. **Climbing stairs:** ways to climb n steps taking 1 or 2 at a time — recursive, memoized, and
   bottom-up with O(1) memory.
5. **Longest common subsequence** of two strings with a 2D table; then reconstruct the subsequence
   itself by walking back through the table.
6. **Edit distance** (Levenshtein) between two words; use it to suggest the closest word from a
   dictionary for a misspelled input.
7. **0/1 knapsack:** items with weights and values, capacity W — the maximum value.
8. Find a counterexample for the "shortest meeting first" greedy rule; then prove to yourself (in
   words) why "earliest end first" works.
9. ★ **Longest increasing subsequence** in O(n²) with DP, then in O(n log n) with
   `std::lower_bound` on a "tails" array.

## Check yourself

1. What are the three steps of divide and conquer?
2. What does "undo" mean in backtracking, and why is it needed?
3. What two properties make a problem suitable for dynamic programming?
4. What's the difference between memoization and tabulation?
5. When is a greedy algorithm correct?

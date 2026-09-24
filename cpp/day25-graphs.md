# C++ Day 25 — Graph algorithms

**Goal:** represent graphs, traverse them with BFS and DFS, and find shortest paths with BFS and
Dijkstra's algorithm.

## Concepts

### Graphs

A **graph** is a set of **vertices** (nodes) connected by **edges**. Edges may be **directed**
(one-way: web links, dependencies) or **undirected** (roads, friendships), and may carry
**weights** (distances, costs). Trees and linked lists are special graphs. So are maps, social
networks, task dependencies, state machines, and mazes.

### Representations

With vertices numbered 0..n-1:

- **Adjacency list** — for each vertex, the list of its neighbors:
  `std::vector<std::vector<int>> adj(n);`. Memory O(V + E). The usual choice.
- **Adjacency matrix** — `matrix[u][v]` is true (or the weight) if there's an edge. Memory O(V²);
  fine for small dense graphs, O(1) edge lookup.
- **Implicit graphs** — a grid or puzzle, where neighbors are computed (up/down/left/right) rather
  than stored.

For weighted graphs, store pairs: `std::vector<std::vector<std::pair<int, int>>> adj;` with
`{neighbor, weight}`.

### Breadth-first search (BFS)

BFS visits vertices in order of distance from the start: all neighbors first, then their neighbors,
and so on — using a **queue**. In an unweighted graph, it finds **shortest paths** (fewest edges).

```cpp
#include <algorithm>
#include <iostream>
#include <queue>
#include <vector>

// Shortest path from `start` to `goal` in an unweighted graph, or an empty vector.
std::vector<int> shortest_path(const std::vector<std::vector<int>> &adj, int start, int goal)
{
    std::vector<int> parent(adj.size(), -1);
    std::vector<bool> visited(adj.size(), false);
    std::queue<int> q;
    q.push(start);
    visited[start] = true;

    while (!q.empty()) {
        int u = q.front();
        q.pop();
        if (u == goal) break;
        for (int v : adj[u]) {
            if (!visited[v]) {
                visited[v] = true;           // mark when enqueued, not when dequeued
                parent[v] = u;
                q.push(v);
            }
        }
    }
    if (!visited[goal]) return {};
    std::vector<int> path;
    for (int v = goal; v != -1; v = parent[v]) path.push_back(v);
    std::ranges::reverse(path);
    return path;
}

int main()
{
    //  0 - 1 - 3 - 5
    //  |   |       |
    //  2 - 4 ------+
    std::vector<std::vector<int>> adj(6);
    auto edge = [&](int a, int b) { adj[a].push_back(b); adj[b].push_back(a); };
    edge(0, 1); edge(0, 2); edge(1, 3); edge(1, 4); edge(2, 4); edge(3, 5); edge(4, 5);

    for (int v : shortest_path(adj, 0, 5)) std::cout << v << ' ';   // 0 1 3 5
    std::cout << '\n';
}
```

Cost: O(V + E) — each vertex and edge is processed once. Recording `parent` lets you rebuild the
path; recording `dist[v] = dist[u] + 1` gives distances.

### Depth-first search (DFS)

DFS goes as deep as possible before backtracking — naturally recursive (or iterative with a
**stack**). Uses: detecting cycles, connected components, topological sort, maze generation, solving
puzzles.

```cpp
void dfs(const std::vector<std::vector<int>> &adj, int u, std::vector<bool> &visited)
{
    visited[u] = true;
    for (int v : adj[u]) {
        if (!visited[v]) dfs(adj, v, visited);
    }
}

// Number of connected components in an undirected graph
int count_components(const std::vector<std::vector<int>> &adj)
{
    std::vector<bool> visited(adj.size(), false);
    int components = 0;
    for (int u = 0; u < static_cast<int>(adj.size()); ++u) {
        if (!visited[u]) {
            ++components;
            dfs(adj, u, visited);
        }
    }
    return components;
}
```

Deep recursion on huge graphs (10⁶ vertices in a line) can overflow the stack; convert to an
explicit `std::stack` if that's a risk.

### Topological sort

For a **directed acyclic graph** (DAG) — tasks with dependencies, course prerequisites, build
steps — a topological order lists every vertex before those that depend on it. **Kahn's algorithm**:
repeatedly take a vertex with no remaining incoming edges.

```cpp
#include <optional>
#include <queue>
#include <vector>

std::optional<std::vector<int>> topo_sort(const std::vector<std::vector<int>> &adj)
{
    std::vector<int> indegree(adj.size(), 0);
    for (const auto &edges : adj)
        for (int v : edges) ++indegree[v];

    std::queue<int> ready;
    for (int u = 0; u < static_cast<int>(adj.size()); ++u)
        if (indegree[u] == 0) ready.push(u);

    std::vector<int> order;
    while (!ready.empty()) {
        int u = ready.front();
        ready.pop();
        order.push_back(u);
        for (int v : adj[u])
            if (--indegree[v] == 0) ready.push(v);
    }
    if (order.size() != adj.size()) return std::nullopt;   // a cycle: no valid order
    return order;
}
```

### Dijkstra's algorithm: shortest paths with weights

With non-negative edge weights, Dijkstra finds the shortest distance from a source to every vertex.
Like BFS, but the "queue" is a **min-priority queue** ordered by current distance (day 17):

```cpp
#include <functional>
#include <iostream>
#include <limits>
#include <queue>
#include <utility>
#include <vector>

using Graph = std::vector<std::vector<std::pair<int, int>>>;   // adj[u] = {(v, weight), ...}

std::vector<long long> dijkstra(const Graph &g, int source)
{
    const long long INF = std::numeric_limits<long long>::max();
    std::vector<long long> dist(g.size(), INF);
    using Item = std::pair<long long, int>;                     // (distance, vertex)
    std::priority_queue<Item, std::vector<Item>, std::greater<>> pq;

    dist[source] = 0;
    pq.push({0, source});
    while (!pq.empty()) {
        auto [d, u] = pq.top();
        pq.pop();
        if (d > dist[u]) continue;                              // stale entry: already improved
        for (auto [v, w] : g[u]) {
            if (dist[u] + w < dist[v]) {
                dist[v] = dist[u] + w;
                pq.push({dist[v], v});
            }
        }
    }
    return dist;
}

int main()
{
    Graph g(5);
    auto road = [&](int a, int b, int w) { g[a].push_back({b, w}); g[b].push_back({a, w}); };
    road(0, 1, 4); road(0, 2, 1); road(2, 1, 2); road(1, 3, 1); road(2, 3, 5); road(3, 4, 3);

    auto dist = dijkstra(g, 0);
    for (std::size_t v = 0; v < dist.size(); ++v) {
        std::cout << "0 -> " << v << ": " << dist[v] << '\n';   // 0 3 1 4 7
    }
}
```

Cost: O((V + E) log V). With negative weights Dijkstra is wrong — use Bellman-Ford. For all pairs on
small graphs, Floyd–Warshall (three nested loops) is simple.

### Grids as graphs

Mazes and maps are graphs where each cell's neighbors are the adjacent cells:

```cpp
const int dr[] = {-1, 1, 0, 0};
const int dc[] = {0, 0, -1, 1};
for (int k = 0; k < 4; ++k) {
    int nr = r + dr[k], nc = c + dc[k];
    if (nr >= 0 && nr < rows && nc >= 0 && nc < cols && grid[nr][nc] != '#') {
        // (nr, nc) is a neighbor
    }
}
```

## Exercises

1. Type in BFS and Dijkstra. Draw the example graphs and verify the outputs by hand.
2. **Maze solver:** read a maze from a text file (`#` walls, `.` open, `S` start, `E` end), find the
   shortest path with BFS, and print the maze with the path marked `*`.
3. **Number of islands:** count connected regions of `#` in a grid, with DFS.
4. **Course schedule:** given prerequisites as pairs `(course, required)`, print a valid order with
   topological sort, or report a cycle.
5. **Cycle detection** in a directed graph with DFS and three colors (white = unvisited, gray = on
   the current path, black = done).
6. Extend Dijkstra to also return the **path** (a `parent` array), and use it on a small road map of
   cities with names (map names to indexes with an `unordered_map<std::string, int>`).
7. **Word ladder:** transform one word into another, changing one letter at a time, through words in
   a dictionary — the shortest sequence (BFS where neighbors differ by one letter).
8. ★ Solve an Advent of Code puzzle that involves a grid or graph (most years have several, e.g.
   2022 day 12, "Hill Climbing Algorithm").

## Check yourself

1. When is an adjacency list better than a matrix?
2. Why does BFS find shortest paths in unweighted graphs, and which data structure does it use?
3. Why mark vertices visited when they're enqueued rather than dequeued?
4. What does topological sort require of the graph?
5. Why does Dijkstra need a priority queue, and why does it fail with negative weights?

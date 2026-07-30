# Container Adapters — `stack`, `queue`, `priority_queue`

Thin wrappers that restrict a container to one access discipline. Small APIs, huge presence in interview solutions: every DFS-iterative, every BFS, every monotonic stack, every heap problem.

---

## `stack<T>` — LIFO

```cpp
#include <stack>
stack<int> st;
st.push(x);        st.emplace(args...);
st.pop();          // removes top, returns VOID
st.top();          // peek — UB if empty
st.size();         st.empty();
```

**`pop()` returns nothing.** The two-step read-and-remove is mandatory:

```cpp
int x = st.top(); st.pop();
```

No iteration, no random access. If you need to inspect the middle, use a `vector<int>` as your stack (`push_back` / `back()` / `pop_back()`) — same complexity, plus indexing.

### Canonical Uses

```cpp
// Valid parentheses (LC 20)
stack<char> st;
for (char c : s) {
    if (c=='('||c=='['||c=='{') st.push(c);
    else {
        if (st.empty()) return false;
        char t = st.top(); st.pop();
        if ((c==')'&&t!='(') || (c==']'&&t!='[') || (c=='}'&&t!='{')) return false;
    }
}
return st.empty();

// Monotonic stack — next greater element (LC 496 / 739)
vector<int> res(n, -1);
stack<int> st;                                  // holds INDICES, values decreasing
for (int i = 0; i < n; i++) {
    while (!st.empty() && nums[st.top()] < nums[i]) {
        res[st.top()] = nums[i];                // nums[i] is the next greater for st.top()
        st.pop();
    }
    st.push(i);
}

// Iterative DFS
stack<int> st; st.push(start);
vector<bool> vis(n);
while (!st.empty()) {
    int u = st.top(); st.pop();
    if (vis[u]) continue;
    vis[u] = true;
    for (int v : adj[u]) if (!vis[v]) st.push(v);
}
```

Monotonic stacks store **indices**, not values — you almost always need the position to write the answer. See [stack-monotonic-stack-patterns.md](../stack-monotonic-stack-patterns.md).

---

## `queue<T>` — FIFO

```cpp
#include <queue>
queue<int> q;
q.push(x);      q.emplace(args...);
q.pop();        // removes FRONT, returns void
q.front();      q.back();
q.size();       q.empty();
```

### Canonical Use — BFS with Level Tracking

```cpp
queue<int> q; q.push(start);
vector<bool> vis(n); vis[start] = true;
int level = 0;
while (!q.empty()) {
    int sz = q.size();                 // SNAPSHOT the size before the inner loop
    for (int i = 0; i < sz; i++) {
        int u = q.front(); q.pop();
        for (int v : adj[u]) if (!vis[v]) { vis[v] = true; q.push(v); }
    }
    level++;
}
```

**`int sz = q.size();` must be captured before the loop** — `q.size()` changes as you push children, so `for (int i = 0; i < q.size(); i++)` processes the wrong nodes. This is the #1 BFS bug.

**Mark visited on push, not on pop.** Marking on pop lets the same node enter the queue many times and can turn BFS quadratic.

Grid BFS:

```cpp
int dr[] = {-1, 1, 0, 0}, dc[] = {0, 0, -1, 1};
queue<pair<int,int>> q; q.push({sr, sc});
while (!q.empty()) {
    auto [r, c] = q.front(); q.pop();
    for (int d = 0; d < 4; d++) {
        int nr = r + dr[d], nc = c + dc[d];
        if (nr < 0 || nr >= R || nc < 0 || nc >= C) continue;
        if (vis[nr][nc] || grid[nr][nc] == '0') continue;
        vis[nr][nc] = true;
        q.push({nr, nc});
    }
}
```

---

## `priority_queue<T>` — Binary Heap

**Max-heap by default.** This surprises people coming from Python's `heapq` (min-heap) and Java's `PriorityQueue` (min-heap).

```cpp
#include <queue>
priority_queue<int> maxHeap;                                   // top() = largest
priority_queue<int, vector<int>, greater<int>> minHeap;        // top() = smallest

pq.push(x);     pq.emplace(args...);
pq.pop();       // removes top, returns void
pq.top();       // peek — UB if empty
pq.size();      pq.empty();
```

Complexities: `push` O(log n), `pop` O(log n), `top` O(1). **No iteration, no search, no arbitrary delete, no decrease-key.**

### The Three-Argument Template

`priority_queue<T, Container, Compare>` — you must spell out the container to reach the comparator.

```cpp
priority_queue<int, vector<int>, greater<int>> minHeap;
priority_queue<pair<int,int>, vector<pair<int,int>>, greater<>> pq;   // C++14 greater<>
```

### The Comparator Is Inverted

`Compare` answers "does a come *before* b in priority order", and the element that comes **last** is on top. So `less<T>` (the default) gives a max-heap and `greater<T>` gives a min-heap. When writing a custom comparator, **write it as if for `sort`, then expect the reverse.**

```cpp
// Min-heap on pair.second
auto cmp = [](const pair<int,int>& a, const pair<int,int>& b) {
    return a.second > b.second;         // ">" gives the SMALLEST second on top
};
priority_queue<pair<int,int>, vector<pair<int,int>>, decltype(cmp)> pq(cmp);
// C++20: decltype(cmp) is default-constructible for captureless lambdas, so `pq;` alone works
```

Struct comparator (works in every standard version):

```cpp
struct Cmp {
    bool operator()(const Node& a, const Node& b) const { return a.cost > b.cost; }  // min-heap
};
priority_queue<Node, vector<Node>, Cmp> pq;
```

### The Negation Trick

Instead of fighting comparators, negate the values and use the default max-heap:

```cpp
priority_queue<int> pq;
pq.push(-cost);
int best = -pq.top();
```

Fast to type, easy to get wrong late at night. Prefer `greater<>` for anything you'll reread.

### Canonical Uses

```cpp
// Kth largest — keep a min-heap of size k, top() is the answer
priority_queue<int, vector<int>, greater<int>> pq;
for (int x : nums) { pq.push(x); if ((int)pq.size() > k) pq.pop(); }
return pq.top();

// Dijkstra — {distance, node}, distance first so pair ordering does the work
priority_queue<pair<long long,int>, vector<pair<long long,int>>, greater<>> pq;
vector<long long> dist(n, LLONG_MAX);
dist[src] = 0; pq.push({0, src});
while (!pq.empty()) {
    auto [d, u] = pq.top(); pq.pop();
    if (d > dist[u]) continue;                       // stale entry — lazy deletion
    for (auto [v, w] : adj[u])
        if (d + w < dist[v]) { dist[v] = d + w; pq.push({dist[v], v}); }
}

// Merge k sorted lists — heap of (value, listIdx, elemIdx)
priority_queue<tuple<int,int,int>, vector<tuple<int,int,int>>, greater<>> pq;

// Running median — max-heap for the low half, min-heap for the high half
priority_queue<int> lo;                                  // largest of the small half
priority_queue<int, vector<int>, greater<int>> hi;       // smallest of the large half
```

**Lazy deletion** (`if (d > dist[u]) continue;`) is the standard workaround for the missing decrease-key. Push the improved entry and skip outdated ones when they surface. Same idea powers heap-based sliding windows: push `{value, index}` and pop the top while its index has left the window.

### Heap From a Vector — O(n)

```cpp
priority_queue<int> pq(nums.begin(), nums.end());        // heapify in O(n), not O(n log n)
```

### Raw Heap Algorithms

When you need to iterate over the heap contents or peek inside, use `<algorithm>`'s heap functions on a plain vector:

```cpp
make_heap(v.begin(), v.end());              // O(n)
push_heap(v.begin(), v.end());              // call AFTER v.push_back(x)
pop_heap(v.begin(), v.end());               // moves max to v.back(); THEN v.pop_back()
sort_heap(v.begin(), v.end());              // O(n log n), consumes the heap property
is_heap(v.begin(), v.end());
```

---

## Underlying Containers

| Adapter | Default backing | Alternatives |
|---|---|---|
| `stack` | `deque` | `vector`, `list` |
| `queue` | `deque` | `list` (not `vector` — no `pop_front`) |
| `priority_queue` | `vector` | `deque` |

```cpp
stack<int, vector<int>> st;         // marginally faster than the deque default
```

Rarely worth changing, but `stack<int, vector<int>>` is a free micro-optimization if a stack-heavy solution is borderline on time.

---

## Adapter Selection

| Need | Use |
|---|---|
| LIFO, undo, matching brackets, iterative DFS | `stack` |
| Next greater/smaller, histogram areas, span problems | monotonic `stack` of indices |
| FIFO, level-order, shortest path in an unweighted graph | `queue` |
| Access both ends, sliding-window max in O(n) | `deque` (not an adapter — see [containers-sequence.md](containers-sequence.md)) |
| Repeated min/max extraction, top-k, Dijkstra, scheduling | `priority_queue` |
| Repeated min/max **plus** arbitrary deletion | `multiset` — the heap can't do it |

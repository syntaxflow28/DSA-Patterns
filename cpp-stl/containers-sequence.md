# Sequence Containers — `vector`, `array`, `deque`, `list`, `pair`, `tuple`

These hold elements in a defined order and are the backbone of nearly every LeetCode solution. `vector` alone appears in more than half of them.

---

## `vector<T>` — The Default Choice

A dynamic array: contiguous memory, O(1) indexing, O(1) amortized `push_back`.

### Construction

```cpp
vector<int> a;                        // empty
vector<int> b(5);                     // 5 elements, value-initialized to 0
vector<int> c(5, -1);                 // 5 elements, all -1
vector<int> d = {1, 2, 3};            // initializer list
vector<int> e(d);                     // copy
vector<int> f(d.begin(), d.end());    // from a range
vector<int> g(arr, arr + n);          // from a C array

vector<vector<int>> grid(r, vector<int>(c, 0));       // r×c filled with 0
vector<vector<int>> adj(n);                            // n empty adjacency lists
vector<vector<vector<int>>> dp(n, vector<vector<int>>(m, vector<int>(k, -1)));  // 3D memo
```

**The 2D trap:** `vector<vector<int>> grid(r, vector<int>(c))` — the *inner* vector is the row. Index as `grid[row][col]`. Getting these swapped is the single most common grid bug.

### Element Access

| Call | Meaning | Bounds-checked? |
|---|---|---|
| `v[i]` | element at index i | no — UB if out of range |
| `v.at(i)` | element at index i | yes — throws `out_of_range` |
| `v.front()` | first element | no — UB if empty |
| `v.back()` | last element | no — UB if empty |
| `v.data()` | raw pointer to the buffer | — |

`v.back()` on an empty vector is undefined behaviour, not an exception. Guard with `if (!v.empty())`.

### Capacity & Size

```cpp
v.size();        // number of elements  (returns size_t — unsigned!)
v.empty();       // size() == 0, preferred over size() == 0
v.capacity();    // allocated slots
v.reserve(n);    // pre-allocate n slots — avoids reallocation, does NOT change size()
v.resize(n);     // change size() to n; new elements are 0
v.resize(n, x);  // ...new elements are x
v.shrink_to_fit();
```

**`reserve` vs `resize`:** `reserve(10)` on an empty vector leaves `size() == 0` — `v[0]` is still UB. `resize(10)` makes ten real elements. Use `reserve` for speed, `resize` to actually create elements.

**`size()` is unsigned.** `for (int i = 0; i < v.size() - 1; i++)` on an *empty* vector loops ~4 billion times, because `0u - 1` wraps around. Write `(int)v.size() - 1` or guard the empty case.

### Modifiers

```cpp
v.push_back(x);            // append (copy/move)
v.emplace_back(args...);   // construct in place — preferred for structs/pairs
v.pop_back();              // remove last, returns nothing
v.insert(v.begin() + i, x);            // O(n)
v.insert(v.end(), other.begin(), other.end());   // append another range
v.erase(v.begin() + i);                // O(n) — erase one
v.erase(v.begin() + l, v.begin() + r); // erase [l, r)
v.clear();                 // size becomes 0
v.assign(n, x);            // replace contents with n copies of x
swap(a, b);                // O(1) — swaps internals, not elements
```

**`pop_back()` returns void.** To take the last element: `int x = v.back(); v.pop_back();`

### The Erase-Remove Idiom

`std::remove` does **not** remove anything — it shuffles unwanted elements to the end and returns the new logical end. You must call `erase`.

```cpp
v.erase(remove(v.begin(), v.end(), val), v.end());                       // remove all == val
v.erase(remove_if(v.begin(), v.end(), [](int x){ return x % 2; }), v.end());  // remove odds
v.erase(unique(v.begin(), v.end()), v.end());                            // dedupe — SORT FIRST
```

C++20 gives you the sane version: `erase(v, val)` and `erase_if(v, pred)`.

### Iteration

```cpp
for (int x : v)          { }   // copy — fine for int, wasteful for strings/vectors
for (int& x : v)         { }   // reference — use this to modify
for (const auto& x : v)  { }   // read-only, no copy — the safe default
for (int i = 0; i < (int)v.size(); i++) { }   // when you need the index

for (auto it = v.begin(); it != v.end(); ++it) cout << *it;
for (auto it = v.rbegin(); it != v.rend(); ++it) cout << *it;  // reverse
```

### `vector<bool>` Is Not a Vector

`vector<bool>` is a bit-packed specialization. `v[i]` returns a proxy object, not a `bool&`. That means:

```cpp
vector<bool> vb(10);
bool& r = vb[0];        // COMPILE ERROR
auto r = vb[0];         // r is a proxy — assigning to it modifies the vector
```

Use `vector<char>` or `deque<bool>` when you need real references or want to avoid surprises. For plain visited arrays `vector<bool>` is fine and memory-efficient.

### Comparison

Vectors compare lexicographically out of the box — extremely handy.

```cpp
vector<int> a{1,2,3}, b{1,3};
a < b;      // true
a == b;     // false
sort(vv.begin(), vv.end());   // sorts vector<vector<int>> lexicographically, no comparator needed
```

---

## `array<T, N>` — Fixed Size, Stack Allocated

```cpp
#include <array>
array<int, 4> a = {1, 2, 3, 4};
array<int, 4> z{};        // all zeros
a.size();  a[i];  a.front();  a.back();  a.fill(0);
sort(a.begin(), a.end());
```

Size is part of the type, so it can't grow. Useful as a hash-map key (`unordered_map` needs a hash, but `map<array<int,3>, int>` works immediately) and for small fixed state like `array<int,26>` letter counts.

Raw C arrays still work and are faster to type for DP tables:

```cpp
int dp[1001][1001];          // global — zero-initialized, no stack overflow
memset(dp, 0, sizeof(dp));   // set every BYTE to 0
memset(dp, -1, sizeof(dp));  // -1 works because 0xFF...FF == -1
memset(dp, 0x3f, sizeof(dp));// ≈ 1e9 "infinity" that survives one addition
```

`memset` sets bytes, so only `0`, `-1`, and `0x3f` are meaningful for `int`. `memset(dp, 1, ...)` gives `16843009`, not `1`.

---

## `deque<T>` — Double-Ended Queue

O(1) insert/erase at **both** ends, plus O(1) random access. The container behind the monotonic-deque sliding-window-maximum pattern.

```cpp
#include <deque>
deque<int> dq;
dq.push_back(x);   dq.push_front(x);
dq.pop_back();     dq.pop_front();
dq.front();        dq.back();        dq[i];
dq.size();         dq.empty();       dq.clear();
```

**Canonical use — sliding window maximum (LC 239):** the deque holds *indices*, values decreasing front-to-back.

```cpp
deque<int> dq;                        // indices, values decreasing
vector<int> res;
for (int i = 0; i < n; i++) {
    while (!dq.empty() && dq.front() <= i - k) dq.pop_front();       // out of window
    while (!dq.empty() && nums[dq.back()] <= nums[i]) dq.pop_back(); // dominated
    dq.push_back(i);
    if (i >= k - 1) res.push_back(nums[dq.front()]);
}
```

Store **indices, not values** — you need the index to know when an element leaves the window.

Not contiguous in memory, so `&dq[0]` is not a valid buffer pointer and it's slower than `vector` for pure iteration.

---

## `list<T>` — Doubly Linked List

Rarely needed on LeetCode, because most linked-list problems hand you a custom `ListNode`. Its one real use: **LRU Cache (LC 146)**, where you need O(1) splice-to-front with a stable iterator.

```cpp
#include <list>
list<pair<int,int>> lru;                       // (key, value), most-recent at front
unordered_map<int, list<pair<int,int>>::iterator> pos;

lru.push_front({k, v});
pos[k] = lru.begin();
lru.splice(lru.begin(), lru, pos[k]);          // O(1) move-to-front, iterator stays valid
lru.pop_back();
```

`splice` is the whole reason to use `list` — it relinks nodes without copying and without invalidating iterators.

---

## `pair<A, B>`

```cpp
#include <utility>
pair<int, string> p = {1, "a"};
auto q = make_pair(1, "a");
p.first;  p.second;

auto [num, name] = p;              // C++17 structured binding
vector<pair<int,int>> edges;
edges.push_back({u, v});
edges.emplace_back(u, v);          // no temporary pair constructed
```

**Pairs compare lexicographically** — `first` first, then `second`. This is why `sort` on `vector<pair<int,int>>` "just works", and why `priority_queue<pair<int,int>>` orders by `first` — the foundation of Dijkstra:

```cpp
priority_queue<pair<int,int>, vector<pair<int,int>>, greater<>> pq;  // min-heap by distance
pq.push({0, src});                 // ALWAYS {distance, node} — distance must be first
```

`pair` has no hash in the standard library, so `unordered_map<pair<int,int>, T>` won't compile. See the custom hash in [snippets.md](snippets.md), or use `map<pair<int,int>, T>` (which works, at a log factor).

---

## `tuple<...>`

```cpp
#include <tuple>
tuple<int, int, string> t = {1, 2, "x"};
get<0>(t);                              // index must be a compile-time constant
auto [a, b, c] = t;                     // C++17 — much cleaner
tie(a, b, c) = t;                       // pre-C++17
```

Main uses: three-way heap entries (`{cost, u, v}` for Kruskal / weighted Dijkstra) and returning multiple values. Like `pair`, it compares lexicographically.

```cpp
priority_queue<tuple<int,int,int>, vector<tuple<int,int,int>>, greater<>> pq;
pq.emplace(w, u, v);
auto [w, u, v] = pq.top(); pq.pop();
```

---

## Performance Notes That Matter

- **`emplace_back` over `push_back`** for anything non-trivial: `v.emplace_back(a, b)` constructs in place; `v.push_back({a, b})` builds a temporary then moves it.
- **`reserve` when the size is known.** Without it, `push_back` reallocates ~log n times, copying everything each time.
- **Pass by `const&` or `&`.** `void solve(vector<int> v)` copies the entire array on every call — the classic reason a correct backtracking solution TLEs.
- **`swap` is O(1)**, so `vector<int>().swap(v)` genuinely frees memory (rarely needed).
- **Iterator invalidation:** any `push_back` that triggers reallocation invalidates *all* iterators, pointers, and references into the vector. Never hold `&v[0]` across a `push_back`.

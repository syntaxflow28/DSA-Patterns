# Lambdas, Comparators & Custom Ordering

Where most C++ interview code goes wrong syntactically. The rules are short; the failure modes are nasty.

---

## Lambda Syntax

```cpp
[capture](params) -> returnType { body };
```

```cpp
auto add   = [](int a, int b) { return a + b; };            // return type deduced
auto isEven= [](int x) { return x % 2 == 0; };
auto rec   = [](int n) -> long long { return n; };          // explicit return type
```

### Capture Modes

| Capture | Meaning |
|---|---|
| `[]` | capture nothing |
| `[&]` | capture everything by **reference** |
| `[=]` | capture everything by **value** (copies) |
| `[&x]` | capture `x` by reference |
| `[x]` | capture `x` by value |
| `[&, x]` | everything by reference except `x` |
| `[this]` | capture the enclosing object (needed for member access in a LeetCode `Solution` class) |

For interview code, **`[&]` is the right default.** It gives access to every local and avoids copying vectors.

```cpp
vector<int> nums = {...};
sort(idx.begin(), idx.end(), [&](int i, int j){ return nums[i] < nums[j]; });   // by reference
sort(idx.begin(), idx.end(), [=](int i, int j){ return nums[i] < nums[j]; });   // COPIES nums per lambda
```

**Danger with `[&]`:** never return or store a lambda that outlives the captured variables. Inside a single function call, which is all LeetCode ever needs, it's safe.

---

## Recursive Lambdas

Backtracking and DFS in a single function, without a helper method.

```cpp
// The idiomatic version — works in C++14 and up
function<void(int,int)> dfs = [&](int node, int parent) {
    for (int nxt : adj[node]) if (nxt != parent) dfs(nxt, node);
};
dfs(0, -1);
```

`function` is in `<functional>`. It has real call overhead (type erasure + heap allocation) — measurable in tight recursions.

Faster, no overhead, C++14 and up:

```cpp
auto dfs = [&](auto&& self, int node, int parent) -> void {
    for (int nxt : adj[node]) if (nxt != parent) self(self, nxt, node);
};
dfs(dfs, 0, -1);
```

C++23 gives the clean form: `auto dfs = [&](this auto&& self, int node) { ... };`

**The explicit return type is mandatory** on recursive lambdas — the compiler can't deduce a return type from a body that calls itself.

```cpp
// Memoized DP as a lambda
vector<vector<int>> memo(n, vector<int>(m, -1));
function<int(int,int)> solve = [&](int i, int j) -> int {
    if (i >= n) return 0;
    int& res = memo[i][j];                 // reference to the cell — write once, read once
    if (res != -1) return res;
    return res = max(solve(i+1, j), 1 + solve(i+1, j+1));
};
```

---

## Comparators — The Rules

### Rule 1: A Comparator Answers "Does `a` Come Before `b`?"

`return true` means `a` is placed earlier.

```cpp
sort(v.begin(), v.end(), [](int a, int b){ return a < b; });   // ascending
sort(v.begin(), v.end(), [](int a, int b){ return a > b; });   // descending
```

### Rule 2: Strict Weak Ordering — Never `<=`

`cmp(a, a)` **must** be `false`. Writing `<=` or `>=` violates this and causes `std::sort` to read past the end of the array — a segfault or garbage result, and it only shows up on large inputs.

```cpp
return a <= b;                    // UNDEFINED BEHAVIOUR — will crash
return a < b;                     // correct
```

Correct tie-breaking never needs `<=`:

```cpp
// WRONG — a.first <= b.first is not a strict weak ordering
[](auto& a, auto& b){ return a.first <= b.first; }

// RIGHT
[](auto& a, auto& b){
    if (a.first != b.first) return a.first < b.first;
    return a.second < b.second;
}
```

### Rule 3: `priority_queue` Inverts Your Comparator

The comparator defines "lower priority", and the element that would sort **last** ends up on `top()`. So:

- `less<T>` (default) → **max**-heap
- `greater<T>` → **min**-heap
- A comparator you'd use for ascending `sort` gives you a **max**-heap.

```cpp
// I want the smallest cost on top:
auto cmp = [](const Node& a, const Node& b){ return a.cost > b.cost; };   // note the >
priority_queue<Node, vector<Node>, decltype(cmp)> pq(cmp);
```

Mnemonic: **write it backwards for a heap.**

---

## The Four Ways to Write a Comparator

### 1. Standard functor

```cpp
sort(v.begin(), v.end(), greater<int>());
priority_queue<int, vector<int>, greater<int>> minHeap;
greater<>{}                             // C++14 transparent version, deduces types
```

### 2. Lambda (best for `sort`)

```cpp
sort(v.begin(), v.end(), [](const auto& a, const auto& b){ return a.second < b.second; });
```

Take parameters by `const&` — sorting `vector<string>` or `vector<vector<int>>` by value copies on every comparison.

### 3. Struct with `operator()` (best for `priority_queue` and containers)

```cpp
struct Cmp {
    bool operator()(const Node& a, const Node& b) const { return a.cost > b.cost; }
};
priority_queue<Node, vector<Node>, Cmp> pq;
set<Node, Cmp> s;
map<int, int, greater<int>> descendingMap;      // a map iterated largest-key-first
```

The struct form is the only one that works cleanly as a *type* parameter with no constructor argument.

### 4. `operator<` on your own type

```cpp
struct Node {
    int cost, id;
    bool operator<(const Node& o) const {                 // used by sort, set, map, and PQ
        return cost != o.cost ? cost < o.cost : id < o.id;
    }
    bool operator>(const Node& o) const { return o < *this; }
};
sort(v.begin(), v.end());
priority_queue<Node> maxHeapByCost;
priority_queue<Node, vector<Node>, greater<Node>> minHeapByCost;   // needs operator>
```

C++20's spaceship gives you all six comparisons for free:

```cpp
struct Node {
    int cost, id;
    auto operator<=>(const Node&) const = default;   // lexicographic on members, in order
};
```

---

## Comparator Recipes

```cpp
// Sort by frequency descending, then value ascending  (LC 451, 347)
sort(v.begin(), v.end(), [&](int a, int b){
    return freq[a] != freq[b] ? freq[a] > freq[b] : a < b;
});

// Sort intervals by start (pairs/vectors already do this)
sort(intervals.begin(), intervals.end());

// Sort intervals by END — the greedy activity-selection order
sort(intervals.begin(), intervals.end(), [](auto& a, auto& b){ return a[1] < b[1]; });

// Sort strings by length, then lexicographically
sort(v.begin(), v.end(), [](const string& a, const string& b){
    return a.size() != b.size() ? a.size() < b.size() : a < b;
});

// Largest Number (LC 179) — a custom concatenation order
sort(v.begin(), v.end(), [](const string& a, const string& b){ return a + b > b + a; });

// Sort points by distance from origin
sort(pts.begin(), pts.end(), [](auto& a, auto& b){
    return a[0]*a[0] + a[1]*a[1] < b[0]*b[0] + b[1]*b[1];
});

// Sort indices, leaving the data untouched
vector<int> idx(n); iota(idx.begin(), idx.end(), 0);
sort(idx.begin(), idx.end(), [&](int i, int j){ return nums[i] < nums[j]; });

// Descending map / set
map<int,int, greater<int>> m;
set<int, greater<int>> s;                 // *s.begin() is now the LARGEST
```

---

## Hashing Custom Types

`unordered_map` needs `std::hash<K>` and `operator==`. Neither exists for `pair`, `tuple`, or `vector`.

```cpp
struct PairHash {
    size_t operator()(const pair<int,int>& p) const noexcept {
        return ((size_t)p.first << 32) ^ (unsigned)p.second;
    }
};
unordered_map<pair<int,int>, int, PairHash> m;
unordered_set<pair<int,int>, PairHash> s;
```

For a user-defined struct, specialize `std::hash` and define `operator==`:

```cpp
struct P { int x, y; bool operator==(const P& o) const { return x==o.x && y==o.y; } };
namespace std {
    template<> struct hash<P> {
        size_t operator()(const P& p) const noexcept {
            return hash<int>()(p.x) ^ (hash<int>()(p.y) << 1);
        }
    };
}
```

XOR-combining hashes is weak (`hash(a,b) == hash(b,a)`). See [snippets.md](snippets.md) for the collision-resistant `splitmix64` version, or sidestep the whole thing: encode into a `long long` key, or use `map<pair<int,int>, T>`.

---

## `std::function` vs `auto`

```cpp
auto f = [](int x){ return x * 2; };          // zero overhead — prefer this
function<int(int)> g = [](int x){ return x * 2; };   // type-erased, heap alloc, virtual call
```

Use `function` only when you need recursion (pre-C++23) or to store lambdas of different types in a container. For everything else, `auto`.

---

## Comparator Checklist

- Comparator returns `true` when `a` should come **first**.
- Never `<=` or `>=` — strict weak ordering only.
- `priority_queue` **reverses** it: `greater` → min-heap.
- Tie-breaks go in an `if` chain, not in one chained expression.
- Take parameters by `const&` for anything larger than an `int`.
- `sort` isn't stable — encode the tie-break explicitly, or use `stable_sort`.
- A `set`/`map` comparator defines *equality* too: if `!cmp(a,b) && !cmp(b,a)`, the container treats `a` and `b` as the **same key** and silently drops one.

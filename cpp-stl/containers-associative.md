# Associative Containers — `map`, `set`, `unordered_map`, `unordered_set`, `multi*`

Two families, and choosing wrongly is the most common avoidable slowdown in interview code.

| | Ordered (`map`, `set`) | Unordered (`unordered_map`, `unordered_set`) |
|---|---|---|
| Structure | red-black tree | hash table |
| Complexity | O(log n) guaranteed | O(1) average, **O(n) worst** |
| Iteration order | **sorted by key** | arbitrary, changes on rehash |
| `lower_bound` / range queries | yes | **no** |
| Key requirement | `operator<` | `std::hash` + `operator==` |

**Default to `unordered_map`.** Reach for `map` only when you need sorted order, `lower_bound`, or a key type with no hash (like `pair`).

---

## `unordered_map<K, V>` — The Workhorse

```cpp
#include <unordered_map>
unordered_map<int, int> freq;
unordered_map<string, vector<string>> groups;
```

### The `[]` Trap — Read This Twice

```cpp
freq[x]++;                  // if x is missing, creates freq[x] = 0, then increments. Correct and idiomatic.
if (freq[x] > 0) { }        // BUG: this INSERTS x with value 0 if it was missing
if (freq.count(x)) { }      // correct existence check
if (freq.contains(x)) { }   // C++20, clearer
auto it = freq.find(x);
if (it != freq.end()) use(it->second);   // best: one lookup instead of two
```

`operator[]` default-constructs a missing value and inserts it. That silently changes `size()`, and inside a loop over the same map it's undefined behaviour. Use `at()` when you want a throw instead:

```cpp
freq.at(x);        // throws out_of_range if missing, never inserts
```

### Full API

```cpp
m.insert({k, v});                 // does NOT overwrite an existing key
m.emplace(k, v);                  // same, constructs in place
m[k] = v;                         // DOES overwrite
m.insert_or_assign(k, v);         // C++17 — explicit overwrite
m.try_emplace(k, v);              // C++17 — insert only if absent, no temp value built

m.erase(k);                       // by key, returns count erased (0 or 1)
m.erase(it);                      // by iterator, returns iterator to next
m.count(k);                       // 0 or 1
m.find(k);                        // iterator or m.end()
m.size();  m.empty();  m.clear();
m.reserve(n);                     // pre-size the table — real speedup for big inputs
```

### Iteration

```cpp
for (auto& [key, val] : m) { ... }        // C++17 structured binding — use this
for (auto& p : m) cout << p.first << p.second;
for (const auto& [k, v] : m) { ... }      // read-only
```

Order is unspecified and can differ between runs. Never rely on it. If you need results sorted, dump into a `vector` and sort.

### Common Patterns

```cpp
// Frequency count
unordered_map<int,int> cnt;
for (int x : nums) cnt[x]++;

// Two Sum
unordered_map<int,int> seen;                       // value -> index
for (int i = 0; i < n; i++) {
    auto it = seen.find(target - nums[i]);
    if (it != seen.end()) return {it->second, i};
    seen[nums[i]] = i;
}

// Group anagrams — key is the canonical sorted form
unordered_map<string, vector<string>> groups;
for (auto& s : strs) { string k = s; sort(k.begin(), k.end()); groups[k].push_back(s); }

// Prefix sum -> count (subarray sum equals K)
unordered_map<long long,int> pref{{0, 1}};         // seed with the empty prefix
long long sum = 0; int ans = 0;
for (int x : nums) { sum += x; ans += pref[sum - k]; pref[sum]++; }
```

Note the seed `{0, 1}` — forgetting it breaks every prefix-sum-counting problem where the answer starts at index 0.

### Erasing While Iterating

```cpp
for (auto it = m.begin(); it != m.end(); ) {
    if (shouldDelete(it->first)) it = m.erase(it);   // erase returns the NEXT iterator
    else ++it;
}
// C++20:
erase_if(m, [](auto& p){ return p.second == 0; });
```

Writing `m.erase(it); ++it;` uses a dangling iterator. Always take the return value.

### Worst-Case O(n) and Anti-Hash Tests

`std::hash<int>` is often the identity function. On Codeforces, adversarial inputs collide every key into one bucket and turn your map into a linked list. LeetCode does not do this, but if you ever hit an inexplicable TLE, either switch to `map` or add a randomized hash (see [snippets.md](snippets.md)).

---

## `map<K, V>` — Sorted, and the Only One With `lower_bound`

Everything in `unordered_map` applies, plus:

```cpp
map<int,int> m;
m.begin();                 // smallest key
m.rbegin();                // largest key
auto it = m.lower_bound(x);// first key >= x
auto it = m.upper_bound(x);// first key >  x
m.erase(m.begin());        // pop the smallest
```

This is what makes `map` irreplaceable for:

- **Sweep line / calendar problems** (LC 253 Meeting Rooms II, 731 My Calendar II): `map<int,int> diff; diff[start]++; diff[end]--;` then iterate in key order to get a running count.
- **Interval merging by key** (LC 715 Range Module, 352 Data Stream as Disjoint Intervals): `lower_bound` finds the neighbouring interval in O(log n).
- **TreeMap-style "closest key"** queries.

```cpp
// Closest key to x
auto it = m.lower_bound(x);
if (it != m.end())    consider(it->first);          // first >= x
if (it != m.begin())  consider(prev(it)->first);    // last < x
```

`prev(it)` and `next(it)` (from `<iterator>`) are the safe way to step; `it - 1` does not compile for tree iterators.

---

## `set<T>` and `unordered_set<T>`

```cpp
set<int> s;
s.insert(x);              // returns pair<iterator,bool>; .second is false if already present
s.count(x);  s.find(x);  s.erase(x);  s.erase(it);
s.size();    s.empty();  s.clear();
*s.begin();               // minimum
*s.rbegin();              // maximum
s.lower_bound(x);         // set only — NOT unordered_set
```

```cpp
if (s.insert(x).second) { /* x was new */ }        // insert-and-test in one lookup
```

**Critical:** use `s.lower_bound(x)`, the *member* function. The free function `std::lower_bound(s.begin(), s.end(), x)` compiles but is **O(n)** because tree iterators aren't random-access. Same trap for `map`, `multiset`, and `list`.

Common uses: dedupe (`set<int> s(v.begin(), v.end())`), visited sets, "does a cycle repeat" detection, and sorted-order maintenance with deletion (LC 220 Contains Duplicate III).

---

## `multiset<T>` / `multimap<K,V>` — Duplicates Allowed

The go-to for "sorted collection where elements come and go", e.g. sliding-window median or max.

```cpp
multiset<int> ms;
ms.insert(x);
ms.count(x);              // O(log n + count) — can be slow with many duplicates
*ms.begin();              // minimum
*ms.rbegin();             // maximum
ms.erase(ms.find(x));     // erase ONE copy
ms.erase(x);              // erases ALL copies — almost never what you want
```

**The `erase` distinction is the classic multiset bug.** `ms.erase(x)` removes every occurrence. To remove a single instance, always `ms.erase(ms.find(x))`.

```cpp
// Sliding window maximum with a multiset
multiset<int> win;
for (int i = 0; i < n; i++) {
    win.insert(nums[i]);
    if (i >= k) win.erase(win.find(nums[i - k]));
    if (i >= k - 1) res.push_back(*win.rbegin());
}
```

O(n log k) — slower than the deque version but far easier to get right, and it supports arbitrary removals.

---

## Custom Key Types

### For `map` / `set` — provide `operator<`

```cpp
struct P { int x, y; };
bool operator<(const P& a, const P& b) {
    return a.x != b.x ? a.x < b.x : a.y < b.y;    // strict weak ordering
}
map<P, int> m;
```

Or just use `pair` / `tuple`, which already compare lexicographically. `map<pair<int,int>,int>` and `set<vector<int>>` work with zero extra code.

### For `unordered_map` / `unordered_set` — provide a hash

`std::hash` exists for the built-in types and `string`, but **not** for `pair`, `tuple`, or `vector`. Options:

```cpp
// 1. Encode into a single integer (fastest, when the ranges are small)
unordered_map<long long,int> m;
m[(long long)r * COLS + c] = v;                   // 2D cell -> one key
m[(long long)a * 1000000 + b] = v;

// 2. Encode into a string (slow but instant to write)
unordered_set<string> seen;
seen.insert(to_string(r) + "," + to_string(c));

// 3. A real hash functor (see snippets.md for the collision-resistant version)
struct PairHash {
    size_t operator()(const pair<int,int>& p) const {
        return hash<long long>()((long long)p.first << 32 | (unsigned)p.second);
    }
};
unordered_map<pair<int,int>, int, PairHash> m;

// 4. Just use map — a log factor is usually fine
map<pair<int,int>, int> m;
```

Option 1 is what experienced competitors reach for. Watch the encoding range: `r * COLS + c` needs `COLS` to be an upper bound on the column count, and the multiplication must be done in `long long`.

---

## Quick Decision Guide

| Situation | Container |
|---|---|
| Count occurrences | `unordered_map<T,int>` |
| Count chars `a–z` | `vector<int>(26)` — not a map |
| Membership test | `unordered_set<T>` |
| Need min/max repeatedly, with deletions | `set` / `multiset` |
| Need "first ≥ x" | `map` / `set` + member `lower_bound` |
| Sweep line / difference array over sparse coordinates | `map<int,int>` |
| Key is `pair` or `vector` | `map`, or `unordered_map` + encoded key |
| Iteration must be sorted | `map` / `set` |
| Nothing special, just speed | `unordered_map` + `reserve` |

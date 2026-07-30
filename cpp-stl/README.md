# C++ STL for LeetCode — The Complete Reference

The pattern guides tell you *what algorithm to write*. This folder tells you *how to write it in C++ without fighting the language*. Every container, every algorithm, every idiom that actually appears in interview code — organized so you can revise the whole thing in an hour before a contest.

**Design rule for this folder:** if it never shows up in a LeetCode solution, it isn't here. No allocators, no `std::valarray`, no locale. What *is* here is complete for competitive and interview use.

---

## The Files

| File | What it covers | Revise when |
|---|---|---|
| [containers-sequence.md](containers-sequence.md) | `vector`, `array`, `deque`, `list`, `pair`, `tuple` | Arrays, 2D grids, sliding window buffers |
| [containers-associative.md](containers-associative.md) | `map`, `set`, `unordered_map`, `unordered_set`, `multi*` | Hashing, frequency counts, ordered queries |
| [adapters.md](adapters.md) | `stack`, `queue`, `priority_queue` | Monotonic stack, BFS, heaps, Dijkstra |
| [algorithms.md](algorithms.md) | `<algorithm>` — sort, binary search, permutations, rotate | Sorting, greedy, "find the boundary" |
| [strings.md](strings.md) | `std::string`, conversions, `stringstream`, char utilities | String parsing, tokenizing, palindromes |
| [pointers-nodes.md](pointers-nodes.md) | `ListNode` / `TreeNode`, pointer syntax, all traversals, recursion signatures | Linked lists, binary trees, BST |
| [numeric-bitwise.md](numeric-bitwise.md) | `<numeric>`, `<cmath>`, bit ops, overflow, limits | Bitmask DP, prefix sums, math problems |
| [comparators-lambdas.md](comparators-lambdas.md) | Lambdas, custom comparators, sorting rules, hashing pairs | Any time `sort` or a heap needs custom order |
| [snippets.md](snippets.md) | Copy-paste idioms: grid dirs, custom hash, fast I/O, debug | Right before you start typing |
| [gotchas.md](gotchas.md) | The bugs that cost you the interview | After every failed submission |

---

## The 60-Second Cheat Sheet

```cpp
#include <bits/stdc++.h>   // LeetCode/Codeforces only — never in real code
using namespace std;

vector<int> v(n, 0);                 // n zeros
vector<vector<int>> g(r, vector<int>(c, 0));   // r×c grid
unordered_map<int,int> freq;         // O(1) avg lookup
map<int,int> ordered;                // O(log n), sorted, supports lower_bound
set<int> s;  multiset<int> ms;       // sorted unique / sorted with duplicates
priority_queue<int> maxHeap;         // top() = largest
priority_queue<int, vector<int>, greater<int>> minHeap;
deque<int> dq;                       // O(1) both ends — monotonic deque
stack<int> st;  queue<int> q;

sort(v.begin(), v.end());
sort(v.begin(), v.end(), greater<int>());          // descending
sort(v.begin(), v.end(), [](int a, int b){ return a > b; });
auto it = lower_bound(v.begin(), v.end(), x);      // first >= x
int idx = it - v.begin();
reverse(v.begin(), v.end());
v.erase(unique(v.begin(), v.end()), v.end());      // dedupe (must be sorted)
int mx = *max_element(v.begin(), v.end());
long long sum = accumulate(v.begin(), v.end(), 0LL);   // 0LL, not 0!
```

---

## Complexity Master Table

The single most useful table in this folder. Memorize the shape, not the exact wording.

| Container | Insert | Erase | Find/Access | Ordered? | Backing structure |
|---|---|---|---|---|---|
| `vector` | O(1) amortized at back, O(n) middle | O(n) except back | O(1) by index | insertion order | dynamic array |
| `deque` | O(1) both ends | O(1) both ends | O(1) by index | insertion order | array of blocks |
| `list` | O(1) with iterator | O(1) with iterator | O(n) | insertion order | doubly linked list |
| `map` / `set` | O(log n) | O(log n) | O(log n) | **sorted by key** | red-black tree |
| `unordered_map` / `unordered_set` | O(1) avg, O(n) worst | O(1) avg | O(1) avg | **no order** | hash table |
| `multiset` / `multimap` | O(log n) | O(log n) | O(log n) | sorted, duplicates kept | red-black tree |
| `priority_queue` | O(log n) push | O(log n) pop | O(1) `top()` only | heap order | binary heap over `vector` |
| `stack` / `queue` | O(1) | O(1) | O(1) at the open end | LIFO / FIFO | `deque` by default |

| Algorithm | Complexity | Note |
|---|---|---|
| `sort` | O(n log n) | introsort; **not stable** |
| `stable_sort` | O(n log n) | stable, uses extra memory |
| `nth_element` | O(n) average | partial sort — the Quickselect you don't write yourself |
| `partial_sort` | O(n log k) | top-k without a heap |
| `lower_bound` / `upper_bound` / `binary_search` | O(log n) | **input must be sorted** |
| `next_permutation` | O(n) per call | full enumeration is O(n! · n) |
| `accumulate` | O(n) | watch the init-value type |
| `reverse` / `rotate` | O(n) | |
| `unique` | O(n) | only removes *adjacent* duplicates |

---

## Choosing a Container — Decision Table

| You need… | Use | Why not the obvious alternative |
|---|---|---|
| Frequency counting | `unordered_map<T,int>` | `map` costs a needless log factor |
| Frequency of chars `a–z` | `vector<int> cnt(26)` or `int cnt[26]{}` | A hash map for 26 keys is pure overhead |
| Keys in sorted order / range queries | `map` or `set` | `unordered_*` gives no ordering at all |
| "Smallest element ≥ x" | `set::lower_bound` | `std::lower_bound` on a `set` iterator is **O(n)** |
| Top-k / streaming min or max | `priority_queue` | Re-sorting each step is O(n log n) per step |
| Running median | two `priority_queue`s, or `multiset` | |
| Sliding-window max | `deque` (monotonic) | A heap needs lazy deletion; the deque is cleaner |
| Duplicates + sorted + delete-one | `multiset` | `set` collapses duplicates; `erase(value)` on a multiset erases **all** copies — use `erase(find(x))` |
| Fixed-size, known at compile time | `array<int,N>` | Stack-allocated, no heap traffic |
| Push/pop at both ends | `deque` | `vector::insert(begin())` is O(n) |

---

## The Six Mistakes That Actually Cost Points

Detail in [gotchas.md](gotchas.md), but these are the repeat offenders:

1. **`int` overflow.** `mid = (l + r) / 2` overflows; use `l + (r - l) / 2`. `accumulate(..., 0)` returns `int` even into a `long long`.
2. **`map[key]` inserts.** Reading a missing key with `[]` *creates* it with value 0. Use `.count()` / `.find()` / `.contains()` to test.
3. **Iterator invalidation.** Erasing while looping. Use `it = m.erase(it)`, and remember `vector::push_back` invalidates everything on reallocation.
4. **Modifying the container you're iterating** (especially `unordered_map` rehash).
5. **Comparator must be strict weak ordering.** `return a <= b;` in a sort comparator is undefined behaviour and *will* segfault on large inputs.
6. **Copying by value in recursion.** `void dfs(vector<int> path)` copies the whole vector per call — take `vector<int>&`.

---

## How to Use This Folder

- **First pass:** read [containers-sequence.md](containers-sequence.md), [containers-associative.md](containers-associative.md), [adapters.md](adapters.md), [algorithms.md](algorithms.md) end to end. That's ~90% of everything you'll type.
- **Pointer-based structures:** [pointers-nodes.md](pointers-nodes.md) is self-contained — linked lists and trees use almost no STL container, so they get their own page.
- **Before a contest:** the cheat sheet above + [snippets.md](snippets.md).
- **After a wrong answer:** [gotchas.md](gotchas.md). The bug is almost certainly on that page.
- **When a solution feels verbose:** [comparators-lambdas.md](comparators-lambdas.md) — most bloat is a hand-written comparator or loop that a one-liner replaces.

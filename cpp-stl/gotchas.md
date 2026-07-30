# Gotchas — The Bugs That Cost You the Interview

Ordered roughly by how often they actually happen. When a submission fails and the logic looks right, scan this page first.

---

## 1. Integer Overflow

```cpp
int mid = (lo + hi) / 2;              // overflows when lo + hi > 2.1e9
int mid = lo + (hi - lo) / 2;         // fix

long long s = a + b;                  // a + b is computed as int FIRST
long long s = (long long)a + b;       // fix
long long p = 1LL * a * b;            // fix for products

long long sum = accumulate(all(v), 0);    // returns int
long long sum = accumulate(all(v), 0LL);  // fix
```

**The expression's type comes from its operands, never from the variable you assign to.** Cast before the operation.

Signed overflow is UB — the compiler is allowed to assume it never happens, so results can look impossible.

---

## 2. `map[key]` Silently Inserts

```cpp
if (m[k] > 0) { }             // INSERTS k = 0 if absent, changing size() and iteration
if (m.count(k)) { }           // fix
if (m.contains(k)) { }        // C++20
auto it = m.find(k); if (it != m.end()) use(it->second);   // best — one lookup
```

Worse: doing this inside a loop over the same map invalidates iterators and can crash.

`m[k]++` is fine and idiomatic — you *want* the default-0 insert there.

---

## 3. Unsigned `size()`

```cpp
for (int i = 0; i < v.size() - 1; i++) { }    // empty vector -> 0u - 1 -> ~4e9 iterations
for (int i = 0; i + 1 < (int)v.size(); i++) { }   // fix

if (v.size() - k >= 0) { }                    // ALWAYS true — unsigned can't be negative
if ((int)v.size() >= (int)k) { }              // fix

int diff = a.size() - b.size();               // wraps if b is longer
int diff = (int)a.size() - (int)b.size();     // fix
```

Mixing `int` and `size_t` in a comparison promotes the `int` to unsigned. Cast to `int` whenever subtraction is involved.

---

## 4. Iterator Invalidation

```cpp
for (auto it = m.begin(); it != m.end(); ++it)
    if (bad(it)) m.erase(it);                 // it is now dangling; ++it is UB

for (auto it = m.begin(); it != m.end(); )
    if (bad(it)) it = m.erase(it);            // fix — erase returns the next
    else ++it;
```

```cpp
for (int x : v) if (x == 0) v.push_back(1);   // reallocation invalidates the range-for iterators
```

| Operation | Invalidates |
|---|---|
| `vector::push_back` (on reallocation) | **all** iterators, pointers, references |
| `vector::erase/insert` | everything from the modification point onward |
| `deque` insert/erase in the middle | all iterators |
| `map`/`set` erase | only the erased element's iterator |
| `unordered_*` insert (on rehash) | all **iterators**; references stay valid |

`map` and `set` iterators are stable — that's why LRU Cache stores `list` iterators in a map.

---

## 5. `pop()` Returns Void

```cpp
int x = st.pop();                 // COMPILE ERROR
int x = st.top(); st.pop();       // correct
int x = q.front(); q.pop();       // queue
int x = pq.top(); pq.pop();       // priority_queue
```

And `top()`/`front()` on an empty container is UB, not an exception. Guard with `!empty()`.

---

## 6. Comparator Isn't a Strict Weak Ordering

```cpp
sort(all(v), [](int a, int b){ return a <= b; });   // UB — segfault on large inputs
sort(all(v), [](int a, int b){ return a < b; });    // fix
```

`std::sort` uses `cmp(a, b) == false && cmp(b, a) == false` to mean "equal", and relies on that to stop its inner loop. `<=` breaks the invariant and the loop runs past the end of the array. It usually passes on small tests and crashes on the big one.

Same rule for `priority_queue`, `set`, and `map`.

---

## 7. `priority_queue` Is a MAX-Heap

```cpp
priority_queue<int> pq;                                  // top() is the LARGEST
priority_queue<int, vector<int>, greater<int>> pq;       // min-heap
```

Coming from Python (`heapq` is a min-heap) or Java (`PriorityQueue` is a min-heap), this is the default assumption that quietly reverses your answer. And the custom comparator is inverted relative to `sort` — see [comparators-lambdas.md](comparators-lambdas.md).

---

## 8. `multiset::erase(value)` Removes Everything

```cpp
ms.erase(x);              // erases ALL copies of x
ms.erase(ms.find(x));     // erases exactly one — almost always what you want
```

Guard `find` if `x` might be absent: `auto it = ms.find(x); if (it != ms.end()) ms.erase(it);`

---

## 9. `lower_bound` on a `set` — The O(n) Trap

```cpp
lower_bound(s.begin(), s.end(), x);     // compiles, but O(n) — tree iterators aren't random access
s.lower_bound(x);                       // fix — the member function, O(log n)
```

Same for `map`, `multiset`, and `list`. This turns an O(n log n) solution into O(n²) with no compile error.

---

## 10. Passing Containers by Value

```cpp
void dfs(vector<int> path, int i) { }        // copies the whole vector every call
void dfs(vector<int>& path, int i) { }       // fix
void solve(const vector<vector<int>>& g) { } // read-only input: const&
```

Same for lambda captures: `[=]` copies every captured container per invocation. Use `[&]`.

The usual symptom is "my backtracking is correct but TLEs" — this is why.

---

## 11. `substr` in a Loop

```cpp
for (int i = 0; i < n; i++)
    for (int j = i; j < n; j++)
        check(s.substr(i, j - i + 1));      // O(n) copy inside O(n²) loops = O(n³)
```

Pass indices instead, or use `string_view` (C++17) for a zero-copy slice.

---

## 12. `-7 % 3 == -1`

```cpp
int idx = (i - 1) % n;                       // negative when i == 0
int idx = ((i - 1) % n + n) % n;             // fix
```

C++'s `%` takes the sign of the dividend. Circular arrays and modular arithmetic hit this constantly. In modular sums: `(a - b + MOD) % MOD`.

---

## 13. Floating-Point Comparison and `pow`

```cpp
if (a == b) { }                    // unreliable for doubles
if (fabs(a - b) < 1e-9) { }        // fix

int x = (int)pow(10, 2);           // can be 99
int x = 100;                       // or write your own integer power
int r = (int)sqrt(n);              // can be off by one — adjust with a while loop
```

Prefer integer arithmetic whenever it's possible: compare `a * d` vs `c * b` instead of `a/b` vs `c/d`.

---

## 14. `vector<bool>` Isn't a Container of `bool`

```cpp
vector<bool> v(10);
bool& r = v[0];                    // COMPILE ERROR — v[0] is a proxy, not a bool&
for (auto& b : v) b = true;        // works, but `auto&` binds to the proxy
```

Use `vector<char>` when you need real references or pointer arithmetic.

---

## 15. 2D Vector Dimensions Swapped

```cpp
vector<vector<int>> g(R, vector<int>(C));   // g[row][col], R rows of C columns
vector<vector<int>> g(C, vector<int>(R));   // silently wrong; only crashes if R != C
```

Test with a non-square grid. Square test cases hide this bug completely.

---

## 16. BFS Level Loop Reads a Changing Size

```cpp
while (!q.empty()) {
    for (int i = 0; i < q.size(); i++) { ... q.push(child); }   // q.size() grows mid-loop
}

while (!q.empty()) {
    int sz = q.size();                                          // fix — snapshot first
    for (int i = 0; i < sz; i++) { ... }
}
```

Also: **mark visited on push, not on pop.** Marking on pop lets a node be enqueued many times.

---

## 17. `unique` Without Sorting

```cpp
v.erase(unique(all(v)), v.end());              // only removes ADJACENT duplicates
sort(all(v)); v.erase(unique(all(v)), v.end());// fix
```

And `unique` doesn't shrink the container by itself — the `erase` is mandatory.

---

## 18. `string::npos` Comparisons

```cpp
if (s.find(t) >= 0) { }             // ALWAYS true — npos is a huge unsigned value
if (s.find(t) != string::npos) { }  // fix
```

---

## 19. Uninitialized Locals

```cpp
int sum;                            // garbage value
for (int x : v) sum += x;           // garbage result

int arr[10];                        // garbage; `int arr[10] = {}` zero-fills
vector<int> v(10);                  // vectors ARE value-initialized to 0
```

Globals are zero-initialized; locals are not.

---

## 20. Stack Overflow from Deep Recursion

Recursion depth beyond roughly 10⁴–10⁵ frames blows the stack. A linked list of 10⁵ nodes recursed one node per frame, or a DFS on a path graph, will crash — often reported as a wrong answer rather than a clear error.

Fix: convert to an iterative loop with an explicit `stack`, or shrink the per-frame footprint (don't pass containers by value).

---

## 21. `max`/`min` Type Mismatch

```cpp
max(0, someLongLong);          // COMPILE ERROR — no matching function
max(0LL, someLongLong);        // fix
max<long long>(0, x);          // or force the template argument
max(a, (int)v.size());         // int vs size_t needs a cast too
```

---

## 22. Modifying a Container While Range-For'ing It

```cpp
for (auto& [k, v] : m) m[k * 2] = v;      // rehash invalidates the loop's iterators
```

Collect the changes into a separate vector, apply them after the loop.

---

## 23. Missing Reset Between Test Cases

LeetCode reuses your `Solution` object across test cases in some problem types. Member variables that hold state from a previous run cause a wrong answer that reproduces only in the full submission, never in a single custom test.

Fix: keep state local to the method, or reset every member at the top of the entry function.

---

## Pre-Submit Checklist

- [ ] Any sum, product, or prefix sum that could exceed 2.1e9 → `long long`.
- [ ] Binary search midpoint is `lo + (hi - lo) / 2`.
- [ ] No `map[k]` used purely as an existence test.
- [ ] Every `size()` subtraction is cast to `int`.
- [ ] Every comparator uses `<`, never `<=`.
- [ ] `priority_queue` direction is the one you intended.
- [ ] Empty input, single element, and all-equal elements are handled.
- [ ] Nothing large is passed or captured by value in a recursion.
- [ ] `unique` is preceded by `sort`.
- [ ] 2D dimensions verified on a non-square grid.

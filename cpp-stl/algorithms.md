# `<algorithm>` — The Functions Worth Knowing

Every algorithm here takes an iterator range `[first, last)` — the end is **exclusive**. `v.begin() + i` to `v.begin() + j` covers indices `i..j-1`.

---

## Sorting

```cpp
sort(v.begin(), v.end());                                   // ascending, O(n log n), NOT stable
sort(v.begin(), v.end(), greater<int>());                   // descending
sort(v.begin(), v.end(), [](int a, int b){ return a > b; });// same, with a lambda
sort(v.begin() + l, v.begin() + r);                         // sort a subrange [l, r)
stable_sort(v.begin(), v.end());                            // preserves the order of equals
```

`sort` is introsort (quicksort + heapsort fallback + insertion sort), so O(n log n) worst case is guaranteed — unlike a naive quicksort.

### Partial Sorting — Often Better Than a Full Sort

```cpp
nth_element(v.begin(), v.begin() + k, v.end());
// v[k] is now the element that would be at index k if sorted.
// Everything before it is <= it, everything after is >= it. Order within halves is arbitrary.
// O(n) average — this is Quickselect.

partial_sort(v.begin(), v.begin() + k, v.end());
// the first k elements are the k smallest, in sorted order. O(n log k).
```

For "kth largest element" (LC 215), `nth_element(v.begin(), v.begin()+k-1, v.end(), greater<int>())` is the O(n) answer. For "top k frequent", `partial_sort` or a heap beats sorting everything.

### Sorting Rules You'll Actually Write

```cpp
// By second, tie-break by first
sort(v.begin(), v.end(), [](auto& a, auto& b){
    return a.second != b.second ? a.second < b.second : a.first < b.first;
});

// Descending by value, ascending by index
sort(v.begin(), v.end(), [](auto& a, auto& b){
    return a.first != b.first ? a.first > b.first : a.second < b.second;
});

// Sort indices instead of data
vector<int> idx(n);
iota(idx.begin(), idx.end(), 0);
sort(idx.begin(), idx.end(), [&](int i, int j){ return nums[i] < nums[j]; });

// Sort intervals by start — the opening move of nearly every interval problem
sort(intervals.begin(), intervals.end());          // pairs/vectors compare lexicographically
```

**The comparator must be a strict weak ordering:** `cmp(a, a)` must be `false`. Writing `return a <= b;` causes out-of-bounds reads inside `std::sort` and crashes on large inputs. Always `<`, never `<=`. Full detail in [comparators-lambdas.md](comparators-lambdas.md).

---

## Binary Search — Requires a Sorted Range

| Function | Returns |
|---|---|
| `binary_search(b, e, x)` | `bool` — is `x` present |
| `lower_bound(b, e, x)` | iterator to the **first element ≥ x** |
| `upper_bound(b, e, x)` | iterator to the **first element > x** |
| `equal_range(b, e, x)` | `pair{lower_bound, upper_bound}` |

```cpp
auto it = lower_bound(v.begin(), v.end(), x);
int i = it - v.begin();                       // convert iterator to index
if (it == v.end()) { /* x is greater than everything */ }
if (it != v.end() && *it == x) { /* found */ }

int cntLess     = lower_bound(v.begin(), v.end(), x) - v.begin();
int cntEqual    = upper_bound(v.begin(), v.end(), x) - lower_bound(v.begin(), v.end(), x);
int cntLessEq   = upper_bound(v.begin(), v.end(), x) - v.begin();
```

**Insertion position for a sorted vector** is exactly `lower_bound`'s index — that's the LC 35 Search Insert Position answer in one line.

Descending arrays need the comparator too:

```cpp
lower_bound(v.begin(), v.end(), x, greater<int>());   // first element <= x, in a descending array
```

**On `set`/`map`, use the member function.** `std::lower_bound` on tree iterators is O(n). See [containers-associative.md](containers-associative.md).

### Hand-Rolled Binary Search on the Answer

The STL versions only search a container. "Binary search on the answer" needs the manual loop:

```cpp
int lo = 1, hi = maxPossible, ans = hi;
while (lo <= hi) {
    int mid = lo + (hi - lo) / 2;          // NOT (lo + hi) / 2 — that overflows
    if (feasible(mid)) { ans = mid; hi = mid - 1; }   // minimize
    else lo = mid + 1;
}
```

See [binary-search-patterns.md](../binary-search-patterns.md) for the full family.

---

## Min / Max

```cpp
max(a, b);   min(a, b);
max({a, b, c, d});                    // initializer list — any number of args
min({a, b, c});

*max_element(v.begin(), v.end());     // returns an ITERATOR — dereference it
*min_element(v.begin(), v.end());
int pos = max_element(v.begin(), v.end()) - v.begin();     // index of the max
max_element(v.begin(), v.end(), [](auto& a, auto& b){ return a.second < b.second; });

auto [mn, mx] = minmax_element(v.begin(), v.end());        // one pass for both
clamp(x, lo, hi);                     // C++17 — max(lo, min(x, hi))
```

`max(a, b)` requires both arguments to be the **same type**. `max(0, someLongLong)` fails to compile — write `max(0LL, x)` or `max<long long>(0, x)`.

---

## Searching & Counting

```cpp
find(v.begin(), v.end(), x);                       // iterator, or end() — O(n)
find_if(v.begin(), v.end(), [](int a){ return a > 5; });
count(v.begin(), v.end(), x);
count_if(v.begin(), v.end(), [](int a){ return a % 2 == 0; });

all_of(v.begin(), v.end(), pred);
any_of(v.begin(), v.end(), pred);
none_of(v.begin(), v.end(), pred);

search(a.begin(), a.end(), b.begin(), b.end());    // find subsequence b inside a
adjacent_find(v.begin(), v.end());                 // first pair of equal neighbours
```

`all_of` / `any_of` replace three-line validation loops and read far better in an interview.

---

## Modifying a Range

```cpp
reverse(v.begin(), v.end());
reverse(v.begin() + l, v.begin() + r + 1);         // reverse the inclusive range [l, r]

rotate(v.begin(), v.begin() + k, v.end());         // left-rotate by k — LC 189 in one line
// to right-rotate by k: rotate(v.begin(), v.end() - k, v.end());

fill(v.begin(), v.end(), 0);
iota(v.begin(), v.end(), 0);                       // <numeric> — 0,1,2,3,... (DSU parent init)

swap(a, b);
iter_swap(v.begin() + i, v.begin() + j);

copy(a.begin(), a.end(), back_inserter(b));        // append a to b
transform(v.begin(), v.end(), v.begin(), [](int x){ return x * 2; });

replace(v.begin(), v.end(), oldVal, newVal);
unique(v.begin(), v.end());                        // collapses ADJACENT duplicates only
```

`unique` needs a sorted range to actually deduplicate, and it doesn't shrink the container:

```cpp
sort(v.begin(), v.end());
v.erase(unique(v.begin(), v.end()), v.end());
```

**Coordinate compression** — the single most common use of this trio:

```cpp
vector<int> vals = nums;
sort(vals.begin(), vals.end());
vals.erase(unique(vals.begin(), vals.end()), vals.end());
int id = lower_bound(vals.begin(), vals.end(), x) - vals.begin();   // x -> 0..m-1
```

---

## Permutations

```cpp
sort(v.begin(), v.end());                          // start from the smallest permutation
do {
    process(v);
} while (next_permutation(v.begin(), v.end()));     // returns false after the largest
```

`next_permutation` handles duplicates correctly — it generates each *distinct* permutation exactly once, which makes LC 47 Permutations II trivial. It also solves LC 31 Next Permutation directly. `prev_permutation` goes the other way.

O(n) amortized per call, so full enumeration of n elements is O(n! · n) — fine up to n ≈ 9.

---

## Set Operations — Sorted Ranges Only

```cpp
vector<int> out;
set_intersection(a.begin(), a.end(), b.begin(), b.end(), back_inserter(out));
set_union       (a.begin(), a.end(), b.begin(), b.end(), back_inserter(out));
set_difference  (a.begin(), a.end(), b.begin(), b.end(), back_inserter(out));   // in a, not in b
includes        (a.begin(), a.end(), b.begin(), b.end());                       // is b a subset of a
```

Both inputs must be sorted. `back_inserter` is in `<iterator>`.

---

## Partitioning

```cpp
partition(v.begin(), v.end(), pred);         // pred-true elements first, order not preserved
stable_partition(v.begin(), v.end(), pred);  // ...order preserved
partition_point(v.begin(), v.end(), pred);   // first element where pred is false — binary search on a predicate
is_sorted(v.begin(), v.end());
```

`partition_point` is the general binary search: it works on any range partitioned by a monotone predicate, not just sorted values.

---

## Comparison

```cpp
equal(a.begin(), a.end(), b.begin());
lexicographical_compare(a.begin(), a.end(), b.begin(), b.end());
mismatch(a.begin(), a.end(), b.begin());     // first position where they differ
```

For `vector` and `string`, `a == b` and `a < b` already do the right thing — reach for these only with mixed container types.

---

## Ranges (C++20) — Shorter, When Available

LeetCode supports C++20, so these compile:

```cpp
#include <ranges>
ranges::sort(v);
ranges::sort(v, greater<>{});
ranges::reverse(v);
int mx = ranges::max(v);
auto it = ranges::find(v, x);
ranges::sort(v, {}, &Point::x);                  // projection: sort by member x
bool ok = ranges::all_of(v, [](int x){ return x > 0; });
```

The projection parameter (`&Point::x`) replaces most custom comparators. Nice to know, but plain iterator versions are still what most interviewers expect to see.

---

## Fast Reference

| Task | One-liner |
|---|---|
| Sort ascending | `sort(v.begin(), v.end())` |
| Sort descending | `sort(v.begin(), v.end(), greater<int>())` |
| Reverse | `reverse(v.begin(), v.end())` |
| Sum | `accumulate(v.begin(), v.end(), 0LL)` |
| Max / min value | `*max_element(...)` / `*min_element(...)` |
| Index of max | `max_element(...) - v.begin()` |
| Deduplicate | `sort(...); v.erase(unique(...), v.end())` |
| Count value | `count(v.begin(), v.end(), x)` |
| First index ≥ x | `lower_bound(...) - v.begin()` |
| Is x present (sorted) | `binary_search(v.begin(), v.end(), x)` |
| Kth smallest, O(n) | `nth_element(v.begin(), v.begin()+k, v.end())` |
| Rotate left by k | `rotate(v.begin(), v.begin()+k, v.end())` |
| 0,1,2,... fill | `iota(v.begin(), v.end(), 0)` |
| All permutations | `sort(...); do{}while(next_permutation(...))` |
| Remove all x | `v.erase(remove(v.begin(),v.end(),x), v.end())` |

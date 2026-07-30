# Divide & Conquer — The Complete Interview Pattern Guide (C++)

Divide and conquer is the pattern people think they know because they can recite merge sort, and then fail to recognize when it matters. Its real interview value is not sorting — it's that **splitting a problem in half creates information that doesn't exist in either half**, and that "cross-boundary" case is where the algorithm lives. Counting inversions, the maximum subarray, the skyline, closest pair: in every one of them the two recursive calls are trivial and the *merge* is the entire solution.

This guide covers the split strategies, the merge techniques, the recurrence math that tells you the cost, and the boundary where D&C stops being right and DP takes over.

---

## How to Recognize a Divide & Conquer Problem

Ask yourself:
- Can the input be **split into independent pieces** whose answers combine into the whole answer?
- Is there a **cross-boundary case** — pairs, subarrays, or ranges that straddle the split — that neither half sees?
- Do the subproblems **fail to overlap**? (If they overlap, you want DP with memoization, not plain recursion.)
- Is the structure **already recursive** — a tree, an expression, a sorted array, an exponent?
- Are the constraints shaped like `n log n` (n ≈ 10⁵–10⁶) rather than `n²`?

**The universal shape:**

```
solve(range):
    if range is trivial: return base answer
    mid = split point
    L = solve(left half)
    R = solve(right half)
    return combine(L, R, cross-boundary work)
```

Everything below is a different answer to two questions: **where do I split**, and **what does `combine` have to compute that neither half knew?**

---

## The Three Split Strategies

| Split | Recurrence | Cost (with O(n) combine) | Examples |
|---|---|---|---|
| **Halve the range** — `mid = (l+r)/2` | T(n) = 2T(n/2) + O(n) | O(n log n) | merge sort, inversions, max subarray, skyline |
| **Partition around a pivot** — sizes unknown | T(n) = T(n/2) + O(n) *expected* | **O(n) expected** | Quickselect, kth largest, top-k |
| **Halve the *value*, not the range** | T(n) = T(n/2) + O(1) | O(log n) | fast exponentiation, matrix power |

The second row is the one candidates forget. **Recursing into only one half turns O(n log n) into O(n)** — that's the entire difference between "sort then index" and Quickselect, and it's a routine follow-up question.

### Master Theorem, The Version You Actually Need

For `T(n) = a·T(n/b) + O(n^d)`:

| Condition | Result | Reading |
|---|---|---|
| `d > log_b(a)` | **O(n^d)** | the combine dominates |
| `d = log_b(a)` | **O(n^d log n)** | balanced — every level costs the same |
| `d < log_b(a)` | **O(n^(log_b a))** | the leaves dominate |

Three instances worth memorizing:
- `T(n) = 2T(n/2) + O(n)` → **O(n log n)** — merge sort. Balanced: log n levels, O(n) each.
- `T(n) = T(n/2) + O(n)` → **O(n)** — Quickselect. Geometric: n + n/2 + n/4 + … = 2n.
- `T(n) = T(n/2) + O(1)` → **O(log n)** — binary search, fast power.

Say the recurrence out loud in the interview before you write code. It converts "I think this is n log n" into a derivation.

---

## Pattern 1: Merge Sort — And the Merge as a Counting Device

**Logic:** Sort each half recursively, then merge two sorted halves in one linear pass. The recursion is boilerplate; the interview content is that **during the merge, both halves are sorted**, which lets you count or aggregate cross-pair relationships in O(n) that would cost O(n²) to check directly.

**Core insight — why it works:** When the left half is sorted and the right half is sorted, and you're comparing `left[i]` against `right[j]`, a single comparison settles a whole *block* of pairs at once. If `left[i] > right[j]`, then every remaining element of the left half is also `> right[j]` — because the left half is sorted. One comparison, `mid - i` pairs counted. That block-at-a-time settlement is what collapses the quadratic pair scan into a linear pass, and it is the reason inversion counting, "count of smaller after self", and range-sum counting are all merge sort with three extra lines.

**Template (sort + count inversions):**
```cpp
long long inversions = 0;

void mergeSort(vector<int>& a, int l, int r, vector<int>& buf) {
    if (r - l <= 1) return;                       // 0 or 1 element is sorted
    int mid = l + (r - l) / 2;
    mergeSort(a, l, mid, buf);
    mergeSort(a, mid, r, buf);

    int i = l, j = mid, k = l;
    while (i < mid && j < r) {
        if (a[i] <= a[j]) buf[k++] = a[i++];      // <= keeps the sort STABLE
        else {
            inversions += mid - i;                // a[i..mid) all exceed a[j]
            buf[k++] = a[j++];
        }
    }
    while (i < mid) buf[k++] = a[i++];
    while (j < r)   buf[k++] = a[j++];
    copy(buf.begin() + l, buf.begin() + r, a.begin() + l);
}
```

<details>
<summary>Java</summary>

```java
long inversions = 0;

void mergeSort(int[] a, int l, int r, int[] buf) {
    if (r - l <= 1) return;
    int mid = l + (r - l) / 2;                    // or (l + r) >>> 1
    mergeSort(a, l, mid, buf);
    mergeSort(a, mid, r, buf);

    int i = l, j = mid, k = l;
    while (i < mid && j < r) {
        if (a[i] <= a[j]) buf[k++] = a[i++];
        else { inversions += mid - i; buf[k++] = a[j++]; }
    }
    while (i < mid) buf[k++] = a[i++];
    while (j < r)   buf[k++] = a[j++];
    System.arraycopy(buf, l, a, l, r - l);        // fastest bulk copy
}
```

</details>

The half-open range `[l, r)` is deliberate: `mid` belongs to the right half, the base case is `r - l <= 1`, and there are no `+1`/`-1` adjustments anywhere. Mixing half-open and closed conventions inside one recursion is the most common source of infinite recursion here.

**When the count needs a separate pass** (LC 493, 327), do the counting *before* merging, with two independent pointer sweeps — trying to count and merge in the same loop with different comparison predicates is where these problems get botched:

```cpp
// LC 493: count pairs i < j with a[i] > 2*a[j], both halves sorted
int j = mid;
for (int i = l; i < mid; i++) {
    while (j < r && (long long)a[i] > 2LL * a[j]) j++;
    count += j - mid;
}
// ...then merge normally
```

Both pointers only move forward across the whole loop, so the counting sweep is O(n), not O(n²).

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 912. Sort an Array | Medium | The drill. Write merge sort *and* quicksort here; this is also where you learn that `sort()` is off-limits and the interviewer wants the buffer allocated once, outside the recursion, not per call. |
| 88. Merge Sorted Array | Easy | Just the merge step, in place, filling from the **back** so you never overwrite unread data. The reverse-fill trick is the whole problem. |
| 148. Sort List | Medium | Merge sort on a linked list — the one sort that's genuinely natural there: O(1) merge space, and the split is slow/fast pointers. Cutting the list at the middle (`prev->next = nullptr`) before recursing is the step people forget, and it loops forever without it. |
| 21. Merge Two Sorted Lists | Easy | The merge primitive alone. Dummy head, then attach the remainder wholesale. |
| 23. Merge k Sorted Lists | Hard | Pairwise D&C merging: merge lists 0&1, 2&3, … then repeat. O(N log k), same as the heap, and worth knowing because it needs no comparator. See [heap-priority-queue-patterns.md](heap-priority-queue-patterns.md) for the heap version. |
| 315. Count of Smaller Numbers After Self | Hard | Merge sort over **(value, original index)** pairs, accumulating into an answer array indexed by the original position. Carrying the index through the sort is the mechanical difficulty; the counting is one line. BIT + coordinate compression is the alternative. |
| 493. Reverse Pairs | Hard | The `2*a[j]` predicate breaks the merge's own comparison, so count in a separate sweep first. Use `long long` — `2 * a[j]` overflows on INT-range inputs, and that test case exists. |
| 327. Count of Range Sum | Hard | Merge sort over the **prefix sum array**, counting `j` with `pre[j] - pre[i] ∈ [lower, upper]` via two moving bounds. The reframe from "subarrays" to "pairs of prefix sums" is the actual unlock — see [tricks/algebraic-reformulation.md](tricks/algebraic-reformulation.md). |
| 剑指 51 / Count Inversions | Medium | The canonical statement. If you can derive `inversions += mid - i` from the sortedness argument, you own this entire family. |

---

## Pattern 2: Quickselect — Recurse Into One Side Only

**Logic:** Partition the array around a pivot so that everything left of it is smaller and everything right is larger. The pivot lands at its final sorted index. If that index is the one you want, you're done; otherwise recurse into **only the side that contains your target**.

**Core insight — why it works:** Sorting computes the position of all n elements. Selection needs the position of exactly one. Partitioning is the smallest amount of work that reveals a single element's final index — and it simultaneously tells you which half is irrelevant. Discarding half the input at each step gives `T(n) = T(n/2) + O(n)`, and that geometric series sums to **2n**, not n log n. The lesson generalizes far beyond selection: *whenever the recursion can discard a branch instead of recursing into both, the log factor disappears.*

**Template:**
```cpp
int partition(vector<int>& a, int l, int r) {     // Lomuto, pivot = a[r]
    int pivot = a[r], i = l;
    for (int j = l; j < r; j++)
        if (a[j] < pivot) swap(a[i++], a[j]);
    swap(a[i], a[r]);
    return i;                                     // a[i] is now in its final position
}

int quickselect(vector<int>& a, int l, int r, int k) {   // k = target INDEX, 0-based
    while (l < r) {
        swap(a[l + rand() % (r - l + 1)], a[r]);  // RANDOM pivot — the anti-worst-case guard
        int p = partition(a, l, r);
        if (p == k) return a[k];
        if (p < k) l = p + 1;                     // discard the left side
        else       r = p - 1;                     // discard the right side
    }
    return a[k];
}
```

<details>
<summary>Java</summary>

```java
private final Random rnd = new Random();

int partition(int[] a, int l, int r) {
    int pivot = a[r], i = l;
    for (int j = l; j < r; j++)
        if (a[j] < pivot) { int t = a[i]; a[i++] = a[j]; a[j] = t; }
    int t = a[i]; a[i] = a[r]; a[r] = t;
    return i;
}

int quickselect(int[] a, int l, int r, int k) {
    while (l < r) {
        int p = l + rnd.nextInt(r - l + 1);
        int t = a[p]; a[p] = a[r]; a[r] = t;      // random pivot to the end
        p = partition(a, l, r);
        if (p == k) return a[k];
        if (p < k) l = p + 1;
        else       r = p - 1;
    }
    return a[k];
}
```

C++ has `nth_element` and Java has nothing equivalent — `Arrays.sort` then index is O(n log n). Writing Quickselect by hand is the only way to hit O(n) in Java.

</details>

**The loop, not recursion.** The recursive call is in tail position, so the `while` form is equivalent and costs O(1) stack. Interviewers notice.

**Random pivot is mandatory.** With a fixed pivot, a sorted or adversarial input degrades every partition to size n−1 and the algorithm becomes O(n²). LeetCode ships those test cases for LC 215. Randomizing makes the expected cost O(n) regardless of input; the median-of-medians pivot makes it *worst-case* O(n) but is never worth writing in an interview — mention it exists and move on.

**In C++, `std::nth_element` is this algorithm** (introselect) and is O(n) average:

```cpp
nth_element(a.begin(), a.begin() + k, a.end());        // a[k] is now correct
nth_element(a.begin(), a.begin() + k - 1, a.end(), greater<int>());   // kth largest
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 215. Kth Largest Element in an Array | Medium | The canonical instance. Give the heap answer (O(n log k)) *and* the Quickselect answer (O(n) expected), then say which you'd ship and why. Note "kth largest" is index `n - k` in ascending order — that off-by-one is graded. |
| 973. K Closest Points to Origin | Medium | Quickselect on squared distance — never take the square root, it costs time and introduces float comparison. Partial output order doesn't matter, which is exactly why selection beats sorting here. |
| 347. Top K Frequent Elements | Medium | Count into a map, then Quickselect on the (value, frequency) pairs. Bucket sort by frequency is the O(n) alternative and is easier to write; know both. |
| 692. Top K Frequent Words | Medium | Selection alone is insufficient — ties break lexicographically, so the top k must then be *sorted*. A good test of whether you notice when selection is not enough. |
| 324. Wiggle Sort II | Hard | Quickselect the median, three-way partition around it, then interleave via virtual indexing. The hardest routine use of selection; the index mapping `(1 + 2*i) % (n | 1)` is the part to memorize. |
| 75. Sort Colors | Medium | Three-way (Dutch national flag) partitioning — the partition step alone, no recursion. It is also the fix for Quickselect on arrays with many duplicate values. See [two-pointers-patterns.md](two-pointers-patterns.md). |
| 462. Minimum Moves to Equal Array Elements II | Medium | The answer is the sum of distances to the median, so `nth_element` gives O(n) where the obvious solution sorts. Small problem, clean demonstration that selection ≠ sorting. |

**Duplicates kill Lomuto.** An array of all-equal values partitions into sizes 0 and n−1 every time — O(n²). Use three-way partitioning (`< pivot`, `== pivot`, `> pivot`) when duplicates are likely, and check whether `k` falls inside the equal block.

---

## Pattern 3: Combine-the-Halves Aggregation

**Logic:** The answer for a range is the best of three candidates: entirely in the left half, entirely in the right half, or **straddling the midpoint**. The two recursive calls handle the first two; the crossing case is computed directly, usually in O(n) by expanding outward from the middle.

**Core insight — why it works:** The split point creates a clean trichotomy: every candidate solution either avoids the boundary (and is therefore fully visible to one recursive call) or crosses it. Crossing candidates are constrained — they must include the midpoint — and that constraint is what makes them cheap to enumerate. A general subarray has O(n²) choices; a subarray *forced to contain index mid* has only O(n), because the best left extension and the best right extension are independent and each is a single linear scan. **Adding a constraint shrinks the search space; the split is what supplies the constraint.**

**Template (maximum subarray, LC 53):**
```cpp
int maxCross(vector<int>& a, int l, int mid, int r) {
    int sum = 0, bestLeft = INT_MIN;
    for (int i = mid; i >= l; i--) { sum += a[i]; bestLeft = max(bestLeft, sum); }
    sum = 0;
    int bestRight = INT_MIN;
    for (int i = mid + 1; i <= r; i++) { sum += a[i]; bestRight = max(bestRight, sum); }
    return bestLeft + bestRight;                  // both halves are non-empty by construction
}

int maxSub(vector<int>& a, int l, int r) {
    if (l == r) return a[l];
    int mid = l + (r - l) / 2;
    return max({maxSub(a, l, mid), maxSub(a, mid + 1, r), maxCross(a, l, mid, r)});
}
```

<details>
<summary>Java</summary>

```java
int maxCross(int[] a, int l, int mid, int r) {
    int sum = 0, bestLeft = Integer.MIN_VALUE;
    for (int i = mid; i >= l; i--) { sum += a[i]; bestLeft = Math.max(bestLeft, sum); }
    sum = 0;
    int bestRight = Integer.MIN_VALUE;
    for (int i = mid + 1; i <= r; i++) { sum += a[i]; bestRight = Math.max(bestRight, sum); }
    return bestLeft + bestRight;
}

int maxSub(int[] a, int l, int r) {
    if (l == r) return a[l];
    int mid = l + (r - l) / 2;
    return Math.max(Math.max(maxSub(a, l, mid), maxSub(a, mid + 1, r)),
                    maxCross(a, l, mid, r));
}
```

</details>

Note that `bestLeft` starts at the midpoint and `bestRight` at `mid+1`, so the crossing candidate always contains at least one element from each side — that's what makes it genuinely "crossing" and prevents double-counting with the recursive results.

**Say this out loud on LC 53:** Kadane solves it in O(n) and D&C in O(n log n), so D&C is the *worse* answer — but it's the one that generalizes to segment trees, where each node stores `(total, prefixMax, suffixMax, best)` and merges in O(1). That upgrade is the actual interview target (LC 53 follow-up, and it's how you'd handle updates).

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 53. Maximum Subarray | Medium | Kadane is the answer; D&C is the follow-up. Being able to write the four-field merge `(sum, pre, suf, best)` is what separates "I memorized Kadane" from "I understand the decomposition". |
| 4. Median of Two Sorted Arrays | Hard | Not a merge — a binary search on the **partition point** of the shorter array, so that left and right halves have equal size and the max-left ≤ min-right. O(log min(m,n)). The hardest boundary conditions on LeetCode; write the `INT_MIN`/`INT_MAX` sentinels for empty sides explicitly. |
| 218. The Skyline Problem | Hard | Split the buildings, recursively produce two skylines, merge them by sweeping both key-point lists and taking the running max of the two current heights. The merge is the entire problem and it is genuinely hard to get right; the sweep-line + heap solution is the alternative. |
| 240. Search a 2D Matrix II | Medium | Quadrant D&C: the midpoint splits into four quadrants, one of which is always eliminable. The O(m+n) staircase walk from the top-right corner is simpler and strictly better — present the staircase, mention the D&C. |
| 427. Construct Quad Tree | Medium | The purest "split into four, recurse, merge if uniform" problem. If all four children are identical leaves, collapse them into one leaf — that collapse *is* the combine step. |
| 558. Logical OR of Two Binary Grids | Medium | Quad-tree merge as a recursion over two trees simultaneously: if either node is a `true` leaf, the result is a `true` leaf. Clean practice at recursing over two structures at once. |
| 1274. Number of Ships in a Rectangle | Hard | Interactive quad-tree search with a query budget. The pruning rule — stop when the API reports no ships in the rectangle — is what keeps it inside the limit. |

---

## Pattern 4: Divide by the Exponent — Fast Power

**Logic:** To compute `x^n`, compute `x^(n/2)` **once**, then square it. Each step halves the exponent, so the cost is O(log n) multiplications rather than n.

**Core insight — why it works:** `x^n = (x^(n/2))² · x^(n mod 2)`. The naive loop recomputes the same partial product n times; the D&C form computes each distinct partial power exactly once and reuses it. This is "divide the *value*, not the collection" — the split isn't over data at all, it's over the binary representation of n. Every set bit of n contributes one multiply, which is why the iterative form is literally a walk over n's bits.

**Template:**
```cpp
double myPow(double x, long long n) {
    if (n < 0) { x = 1 / x; n = -n; }              // careful: -INT_MIN overflows, take n as long long
    double result = 1;
    while (n > 0) {
        if (n & 1) result *= x;                    // this bit of the exponent is set
        x *= x;                                    // x^(2^k) for the next bit
        n >>= 1;
    }
    return result;
}

long long powMod(long long b, long long e, long long mod) {
    long long r = 1; b %= mod;
    while (e > 0) {
        if (e & 1) r = r * b % mod;
        b = b * b % mod;
        e >>= 1;
    }
    return r;
}
```

<details>
<summary>Java</summary>

```java
double myPow(double x, int n) {
    long e = n;                                    // widen BEFORE negating — -Integer.MIN_VALUE overflows
    if (e < 0) { x = 1 / x; e = -e; }
    double result = 1;
    while (e > 0) {
        if ((e & 1) == 1) result *= x;
        x *= x;
        e >>= 1;
    }
    return result;
}

long powMod(long b, long e, long mod) {
    long r = 1; b %= mod;
    while (e > 0) {
        if ((e & 1) == 1) r = r * b % mod;
        b = b * b % mod;
        e >>= 1;
    }
    return r;
}
```

</details>

**`n = INT_MIN` is the graded edge case.** `-INT_MIN` overflows in both languages. Widen to `long long`/`long` *before* negating — this is the single test that fails most LC 50 submissions.

**The same halving applies to matrices**, which is how you get the O(log n) Fibonacci:

```cpp
// [[1,1],[1,0]]^n gives F(n+1) in the top-left
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 50. Pow(x, n) | Medium | The canonical instance. Handle negative n by inverting the base, and widen the exponent before negating. Interviewers ask for the recursive form and then the iterative one. |
| 372. Super Pow | Medium | Exponent given as a digit array: `a^[d1..dk] = (a^[d1..dk-1])^10 · a^dk`. D&C on the digit list rather than on bits — a good check that you understand the decomposition rather than the bit trick. |
| 29. Divide Two Integers | Medium | Division by repeated doubling — the additive mirror of fast power. Subtract the largest `divisor << k` that still fits and add `1 << k` to the quotient. Watch `INT_MIN / -1`. |
| 509. Fibonacci Number | Easy | Trivial by DP; the interview upgrade is matrix exponentiation in O(log n). Worth writing once so you recognize linear recurrences later. |
| 1922. Count Good Numbers | Medium | A pure modular fast-power application — the whole problem is recognizing the closed form, then `powMod` it. |
| 1969. Minimum Non-Zero Product | Medium | Closed form plus modular exponentiation, with a nasty `MOD - 1` subtlety in the exponent. Shows why you keep `powMod` in your head. |

---

## Pattern 5: Split at Every Position — Expression Partitioning

**Logic:** When there's no single natural split, **try every split point** and combine the results of both sides. For an expression, each operator is a candidate "last operation"; recurse on the sub-expressions to its left and right, then combine every left result with every right result.

**Core insight — why it works:** The question "how many ways can this be parenthesized" has no answer until you fix **which operation is performed last**. Fixing it makes the left and right sub-expressions completely independent — they can't interact, because the last operation is the only thing joining them. This is the same reframing move that unlocks Burst Balloons, and it's worth recognizing as a family: *when subproblems seem entangled, ask which step happens last rather than first.* See [tricks/reversal-and-time.md](tricks/reversal-and-time.md).

**Template (LC 241, Different Ways to Add Parentheses):**
```cpp
vector<int> ways(const string& s) {
    vector<int> res;
    for (int i = 0; i < (int)s.size(); i++) {
        char c = s[i];
        if (c == '+' || c == '-' || c == '*') {
            vector<int> left  = ways(s.substr(0, i));       // this operator is LAST
            vector<int> right = ways(s.substr(i + 1));
            for (int a : left)
                for (int b : right)
                    res.push_back(c == '+' ? a + b : c == '-' ? a - b : a * b);
        }
    }
    if (res.empty()) res.push_back(stoi(s));                // no operator: a leaf number
    return res;
}
```

<details>
<summary>Java</summary>

```java
List<Integer> ways(String s) {
    List<Integer> res = new ArrayList<>();
    for (int i = 0; i < s.length(); i++) {
        char c = s.charAt(i);
        if (c == '+' || c == '-' || c == '*') {
            List<Integer> left  = ways(s.substring(0, i));
            List<Integer> right = ways(s.substring(i + 1));
            for (int a : left)
                for (int b : right)
                    res.add(c == '+' ? a + b : c == '-' ? a - b : a * b);
        }
    }
    if (res.isEmpty()) res.add(Integer.parseInt(s));
    return res;
}
```

</details>

**This is where D&C meets DP.** Different split points produce **overlapping** subproblems — `ways("2*3")` gets recomputed from several parents. Memoizing on the substring (or on `(l, r)` index pairs) converts this into **interval DP**, which is the same recursion with a cache. The rule:

> Subproblems don't overlap → plain D&C. Subproblems overlap → memoize, and now it's DP.

That's the entire boundary between this guide and [dynamic-programming-patterns.md](dynamic-programming-patterns.md).

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 241. Different Ways to Add Parentheses | Medium | The canonical instance. Add a `unordered_map<string, vector<int>>` memo and you've written interval DP; the interviewer will usually ask for exactly that. |
| 95. Unique Binary Search Trees II | Medium | Every value in `[lo, hi]` is a candidate root; the left subtrees come from `[lo, root-1]` and the right from `[root+1, hi]`. Cartesian product of both lists — structurally identical to 241. |
| 96. Unique Binary Search Trees | Medium | The counting version: Catalan numbers. Shows the pattern's arithmetic skeleton without the tree building. |
| 312. Burst Balloons | Hard | The "which balloon bursts **last**" reframe. Choosing "first" gives ragged, dependent subproblems and no recurrence exists; choosing "last" makes both sides independent. The flagship problem of this idea. |
| 1039. Minimum Score Triangulation of Polygon | Medium | Fix the triangle containing edge `(i, j)`; its third vertex `k` splits the polygon into two independent sub-polygons. Same shape, geometric dressing. |
| 546. Remove Boxes | Hard | The extreme case: the state needs a third dimension (how many same-colored boxes are glued to the left) before the split is even legal. Attempt only after 312. |
| 282. Expression Add Operators | Hard | Backtracking rather than D&C, but the split-at-every-position instinct is what generates the candidates. The multiplication precedence bookkeeping is the real work. |

---

## Pattern 6: Divide by Structure — Trees and Traversals

**Logic:** A tree is a recursive structure by definition, so "divide and conquer" is often just *the natural recursion*. Building a tree from traversals is the interesting direction: locate the root, use it to split the remaining elements into left and right groups, recurse on each.

**Core insight — why it works:** Preorder tells you **which element is the root** (it's first). Inorder tells you **how the rest splits** (everything before the root belongs to the left subtree, everything after to the right). Neither traversal alone is enough — preorder has no split information, inorder has no root information. The reconstruction works precisely because the two orderings supply complementary halves of the same fact, and each recursive call needs exactly one of each.

**Template (LC 105, build from preorder + inorder):**
```cpp
unordered_map<int,int> pos;                 // value -> index in inorder, for O(1) splitting
int preIdx = 0;

TreeNode* build(vector<int>& pre, int inL, int inR) {
    if (inL > inR) return nullptr;
    int rootVal = pre[preIdx++];            // preorder hands out roots in exactly the right order
    TreeNode* root = new TreeNode(rootVal);
    int mid = pos[rootVal];
    root->left  = build(pre, inL, mid - 1); // LEFT FIRST — preorder demands it
    root->right = build(pre, mid + 1, inR);
    return root;
}
```

<details>
<summary>Java</summary>

```java
private Map<Integer,Integer> pos = new HashMap<>();
private int preIdx = 0;

TreeNode build(int[] pre, int inL, int inR) {
    if (inL > inR) return null;
    int rootVal = pre[preIdx++];
    TreeNode root = new TreeNode(rootVal);
    int mid = pos.get(rootVal);
    root.left  = build(pre, inL, mid - 1);
    root.right = build(pre, mid + 1, inR);
    return root;
}
```

</details>

Two things carry this template: the **hashmap from value to inorder index** (a linear scan instead makes it O(n²)), and **recursing left before right** (the shared `preIdx` counter is only correct in that order). For postorder + inorder, walk the postorder array *backwards* and recurse **right before left** — the mirror.

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 105. Construct Binary Tree from Preorder and Inorder | Medium | The canonical instance. Building the value→index map and consuming preorder with a shared counter are both required to hit O(n). |
| 106. Construct from Inorder and Postorder | Medium | The mirror: consume postorder from the end, build **right subtree first**. Doing both back-to-back is what makes the symmetry stick. |
| 889. Construct from Preorder and Postorder | Medium | Ambiguous — multiple valid trees exist, because neither traversal separates left from right for single-child nodes. Being able to *say why* it's ambiguous is the point of the problem. |
| 108. Convert Sorted Array to BST | Easy | Pick the middle as root, recurse on both halves. Choosing the middle is what makes the result height-balanced — the whole content of the problem in one line. |
| 109. Convert Sorted List to BST | Medium | Same idea with slow/fast to find the middle, O(n log n) — or the O(n) inorder-simulation trick that builds bottom-up while walking the list once. Know that the O(n) version exists. |
| 654. Maximum Binary Tree | Medium | Root = maximum of the range, recurse on both sides. O(n²) naive; the monotonic-stack construction is O(n) and is the intended follow-up. |
| 1008. Construct BST from Preorder | Medium | Split on the first element exceeding the root — bounds-passing turns it into O(n). |
| 427. Construct Quad Tree | Medium | The 2D analogue: split into four, collapse when uniform. |
| 617. Merge Two Binary Trees | Easy | Recursing over two trees simultaneously — the simplest instance of "combine two recursive structures". |

---

## Pattern 7: Split Where the Answer Cannot Cross

**Logic:** Find a position that **provably cannot be part of any valid answer**, and split there. Both sides become independent subproblems, and no crossing case exists at all — the combine step is just `max` or `sum`.

**Core insight — why it works:** This is the mirror image of Pattern 3. There, the combine step was expensive because candidates could straddle the split. Here you choose the split point *specifically so that nothing straddles it* — an element that disqualifies every window containing it acts as a wall. The recursion then has no cross-boundary work, which is why it's the escape hatch for problems where sliding window fails: a window needs a monotone shrink rule, and when none exists, a forced wall gives you the decomposition instead.

**Template (LC 395, longest substring with every char appearing ≥ k times):**
```cpp
int longestSubstring(string s, int k) {
    if ((int)s.size() < k) return 0;
    int cnt[26] = {};
    for (char c : s) cnt[c - 'a']++;
    for (int i = 0; i < (int)s.size(); i++) {
        if (cnt[s[i] - 'a'] < k) {                       // s[i] can never be in a valid answer
            int j = i;
            while (j < (int)s.size() && cnt[s[j] - 'a'] < k) j++;   // skip the whole bad run
            return max(longestSubstring(s.substr(0, i), k),
                       longestSubstring(s.substr(j), k));
        }
    }
    return s.size();                                     // no invalid char: the whole string qualifies
}
```

<details>
<summary>Java</summary>

```java
int longestSubstring(String s, int k) {
    if (s.length() < k) return 0;
    int[] cnt = new int[26];
    for (char c : s.toCharArray()) cnt[c - 'a']++;
    for (int i = 0; i < s.length(); i++) {
        if (cnt[s.charAt(i) - 'a'] < k) {
            int j = i;
            while (j < s.length() && cnt[s.charAt(j) - 'a'] < k) j++;
            return Math.max(longestSubstring(s.substring(0, i), k),
                            longestSubstring(s.substring(j), k));
        }
    }
    return s.length();
}
```

</details>

Skipping the entire *run* of invalid characters (the inner `while`) rather than one at a time is what keeps the recursion shallow — without it, a string of all-invalid characters recurses n deep.

**LC 84 (Largest Rectangle in Histogram)** is the same idea with the minimum bar as the wall: the optimal rectangle either spans the full width at the minimum height, or lies entirely left or right of the minimum bar — it can never *partially* include it. That gives an O(n log n) D&C with a sparse table for range minimum, and, more usefully, it is the cleanest explanation of *why* the monotonic stack solution is correct. See [tricks/invariants-and-proofs.md](tricks/invariants-and-proofs.md).

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 395. Longest Substring with At Least K Repeating Characters | Medium | The canonical instance, and the reason it's on this page: candidates try to slide a window, find no monotone shrink rule, and stall. The "Medium" tag is a lie. |
| 84. Largest Rectangle in Histogram | Hard | D&C on the minimum bar is O(n log n) with a sparse table and O(n²) without. Ship the monotonic stack, but use the minimum-bar argument to *justify* it. |
| 1102. Path With Maximum Minimum Value | Medium | Binary search on the answer plus connectivity — a different decomposition, but the same "eliminate by threshold" instinct. |
| 768. Max Chunks To Make Sorted II | Hard | A split point is legal exactly where `max(prefix) <= min(suffix)`. Finding all such walls in one pass is the whole problem — it's D&C thinking without recursion. |
| 769. Max Chunks To Make Sorted | Medium | The permutation special case: split wherever `max(prefix) == index`. Do this before 768. |

---

## Pattern 8: Binary Search — D&C With a Free Half

Binary search is divide and conquer where one half is discarded for free — `T(n) = T(n/2) + O(1)` → O(log n). It's large enough to warrant its own guide: see **[binary-search-patterns.md](binary-search-patterns.md)** for the full treatment (sorted arrays, rotated arrays, binary search on the answer, on 2D matrices, and on real numbers).

Two things to carry back here:

- **Recursing into one branch is the optimization.** Whenever your D&C recursion can *prove* the answer isn't in one half, drop it. That's the same move that turns merge sort into Quickselect and linear scan into binary search.
- **The midpoint is always `l + (r - l) / 2`** (C++) or `(l + r) >>> 1` (Java). `(l + r) / 2` overflows, and it does so in a way that only appears on large inputs.

---

## Pattern 9: Offline D&C Over Time (Advanced)

**Logic:** When queries and updates interleave and no online structure exists, process all operations offline: recursively split the *timeline*, apply the left half's updates to the right half's queries, and recurse.

**Core insight — why it works:** An update at time `t` affects exactly the queries at times `> t`. Splitting the timeline at `mid` means every (update, query) interaction is either entirely inside the left half, entirely inside the right half, or **left-update → right-query**. The two recursive calls handle the first two, and the third is a single batched pass — the same trichotomy as Pattern 3, applied to time instead of position.

This is CDQ divide-and-conquer, and the related "D&C with rollback DSU" handles offline dynamic connectivity. Rare on LeetCode, occasionally decisive on Codeforces. Related idea: [tricks/reversal-and-time.md](tricks/reversal-and-time.md), where a destructive process is simply run backwards so that removals become additions.

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 1157. Online Majority Element In Subarray | Hard | Segment-tree-of-candidates using the Boyer-Moore merge — a D&C structure where the *combine* is the clever part. |
| 699. Falling Squares | Hard | Coordinate compression plus a segment tree with lazy assignment; the offline reformulation is what makes it tractable. |
| 315 / 327 / 493 | Hard | Already listed under merge sort — they *are* the beginner's CDQ: a merge over sorted halves counting cross-boundary pairs. |

---

## Common Pitfalls

- **Mixing range conventions.** Pick `[l, r)` or `[l, r]` and never mix them in one recursion. Half-open makes merge sort cleanest (`mid` goes right, base case `r - l <= 1`); closed makes tree and interval problems cleanest (`l > r` is empty). Mixing produces infinite recursion or silently skipped elements.
- **`mid = (l + r) / 2` overflow.** Use `l + (r - l) / 2` in C++, `(l + r) >>> 1` in Java. It only fails on large indices, which is exactly the test you can't run locally.
- **A base case that doesn't shrink.** If `mid == l` and you recurse on `[l, mid]`, you recurse forever. With closed ranges and `mid = l + (r-l)/2`, the right call must be `[mid+1, r]`.
- **Allocating the merge buffer inside the recursion.** `vector<int> buf(n)` per call turns O(n log n) into O(n log n) allocations. Allocate once, pass by reference.
- **Forgetting to cut the list in 148.** Merge sort on a linked list requires severing `prev->next = nullptr` at the midpoint; otherwise both halves are the same list and it never terminates.
- **Counting and merging in one loop with different predicates** (LC 493). The `2*a[j]` comparison and the `a[i] <= a[j]` merge comparison advance pointers differently — do two separate sweeps.
- **Overflow in the counting predicate.** `2 * a[j]` and `pre[j] - pre[i]` need `long long`. LC 493 and 327 both ship inputs that break `int`.
- **Fixed pivot in Quickselect.** Sorted input degrades to O(n²). Randomize. And with many duplicates, use three-way partitioning.
- **Off-by-one on "kth largest".** In an ascending array, the kth largest sits at index `n - k`. Write it down before coding.
- **`-INT_MIN` in fast power.** Widen the exponent to 64 bits *before* negating (LC 50).
- **Assuming subproblems don't overlap.** If your D&C recursion is exponential, it's DP in disguise — memoize on the range. That's the 241/312 lesson.
- **Recursion depth.** Balanced splits give O(log n) depth and are always safe; a degenerate split (Quickselect worst case, Pattern 7 on adversarial input) can reach O(n). Convert tail recursion to a loop where possible.
- **Stability.** Merge sort is stable only if you use `a[i] <= a[j]` (take from the left on ties). `<` silently breaks stability, which matters when you're sorting `(value, index)` pairs.

---

## Cheat Sheet: Problem Phrase → Split Strategy

| Phrase in the problem | Strategy |
|---|---|
| "count pairs `i < j` with some relation" | Merge sort, count during the merge (P1) |
| "count inversions / smaller after self / range sums" | Merge sort over values or prefix sums (P1) |
| "kth largest / smallest / closest", order not required | Quickselect — O(n), not O(n log n) (P2) |
| "top k" where the k elements needn't be sorted | Quickselect or bucket sort (P2) |
| "maximum/minimum over all subarrays", updates expected | Combine halves; upgrade to a segment tree (P3) |
| "merge two structures into one" (skylines, quad trees) | Combine halves (P3) |
| "x^n", "a^b mod m", huge exponents | Halve the exponent (P4) |
| "all possible ways to parenthesize / build / split" | Split at every position, then memoize (P5) |
| "which element is processed **last**" | Split at every position (P5) — see [tricks](tricks/reversal-and-time.md) |
| "construct a tree from traversals / sorted data" | Root splits the rest into two groups (P6) |
| Sliding window has no valid shrink rule | Split at a character that can never qualify (P7) |
| "search in a sorted/rotated/monotone space" | Binary search (P8) |
| Interleaved updates and queries, offline allowed | D&C over the timeline (P9) |

---

## Complexity Summary

- Merge sort: **O(n log n)** time, **O(n)** auxiliary space (O(1) for linked lists).
- Counting variants (315, 493, 327): **O(n log n)**, same recursion plus a linear counting sweep per level.
- Quickselect: **O(n) expected**, O(n²) worst case with a fixed pivot, O(1) extra space with the loop form.
- Combine-the-halves (53, 218): **O(n log n)** with an O(n) merge; O(n) if the merge is O(1) (segment-tree node merges).
- Fast power: **O(log n)** multiplications; matrix power **O(k³ log n)** for a k×k matrix.
- Split-at-every-position: **exponential (Catalan)** without memoization, **O(n³)** as interval DP with it.
- Tree construction from traversals: **O(n)** with a value→index hashmap, O(n²) without.
- Split-at-a-wall (395): **O(n · 26)** in practice — depth is bounded by the alphabet size, since each level removes at least one distinct character.
- Recursion depth: **O(log n)** for balanced splits, **O(n)** worst case for pivot-based ones.

---

## Interview Tips

1. **State the recurrence before you code.** "Two subproblems of half the size plus a linear merge, so T(n) = 2T(n/2) + O(n) = O(n log n)." That one sentence demonstrates you're deriving the complexity rather than recalling it.
2. **Name the cross-boundary case explicitly.** For any halving D&C, say "left-only, right-only, or crossing" and then explain what crossing costs. If crossing is free, say *why* nothing can straddle the split — that's the Pattern 7 insight.
3. **Volunteer the one-branch optimization.** When you write a D&C that recurses into both halves, mention whether one half could be discarded. If it can, the log factor disappears; if it can't, saying so shows you checked.
4. **On selection problems, give both answers.** Heap O(n log k) and Quickselect O(n) expected, then state which you'd ship (usually the heap — it's shorter, and the constant factors are close for small k).
5. **Flag the memoization boundary.** When split points overlap, say "these subproblems repeat, so this becomes interval DP" and add the cache. Interviewers are explicitly testing whether you notice.
6. **Randomize your pivot without being asked**, and say the word "adversarial". It's a one-line change that shows you know the worst case exists.
7. **For merge sort variants, write the merge first.** The recursion is boilerplate; the counting logic is the graded part. Getting `inversions += mid - i` right and being able to justify it in one sentence is the whole problem.

---

## Suggested Practice Order

**Week 1 — the merge:** 88 → 21 → 912 → 148 → 23
**Week 2 — selection:** 215 → 973 → 347 → 692 → 75 → 324
**Week 3 — combine and split:** 53 → 108 → 105 → 106 → 654 → 427
**Week 4 — value splits and parenthesization:** 50 → 372 → 29 → 241 → 95 → 96
**Week 5 — boss fights:** 315 → 493 → 327 → 395 → 84 → 4 → 218 → 312

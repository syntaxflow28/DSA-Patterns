# Dynamic Programming — The Complete Interview Pattern Guide (C++)

DP has a reputation as the hardest interview topic, but it's really one idea industrialized: **solve each distinct subproblem once, store the answer, and build bigger answers from smaller ones.** The difficulty is never the loops — it's *defining the state*. Once the state is right, the recurrence is usually one honest sentence and the code writes itself. This guide covers every interview-relevant DP family: the logic, the **core insight** (why it's correct), C++ templates, strong problem sets with descriptive notes, and the pitfalls.

---

## What DP Actually Is

> **DP applies when a problem has (1) optimal substructure — the answer is composable from answers to smaller subproblems — and (2) overlapping subproblems — the same subproblems recur, so caching pays.**

Both conditions matter, and each rules out a different impostor:
- Optimal substructure without overlap → plain **divide & conquer** (merge sort: subproblems never repeat; caching buys nothing).
- Overlap without needing every option → maybe **greedy** (if a local rule provably always leads to the optimum, you don't need the table at all — and proving that exchange argument is how you justify *skipping* DP).
- Neither → search/backtracking territory.

**DP = recursion + memory.** Top-down (memoized DFS — Pattern 5 of the DFS guide) and bottom-up (tables and loops) are the *same algorithm* with different bookkeeping: the recursion tree collapsed into a DAG of distinct states, each computed once. Complexity for both is **O(states × transition cost)** — count states *first*; it's simultaneously your feasibility check and your complexity analysis.

**The anatomy every DP shares — four questions, in order:**
1. **State:** what does `dp[i]` (or `dp[i][j]`…) *mean, in one English sentence*? This sentence is the whole game. "dp[i] = the maximum money robbable from the first i houses" — precise, complete, unambiguous.
2. **Transition:** how does a state follow from strictly smaller ones? Usually a max/min/sum over the **last decision**: "either rob house i (add dp[i−2]) or skip it (take dp[i−1])."
3. **Base cases:** the smallest states answered directly — and they must match the state sentence *exactly* (empty prefix = 0? one house = its value?).
4. **Answer location + order:** which cell holds the final answer, and what fill order guarantees dependencies are ready (left-to-right? by interval length? topological?).

If you can say all four out loud, the implementation is transcription. If you can't say #1, no amount of coding will save you.

**The universal discovery method — "what was the last move?"** Nearly every recurrence is found the same way: stand at the final answer and ask what the **last decision** could have been. Last step onto stair n came from n−1 or n−2. Last character of the LCS either matches or one string drops its tail. The last balloon to burst in an interval splits it into two independent halves. Enumerating the last move partitions all solutions into disjoint classes, each reducible to a smaller state — that disjoint-and-exhaustive partition is *why* the recurrence is correct, and it's the argument to narrate in interviews.

---

## How to Recognize a DP Problem

**1. Phrasing signals.**
- "**Number of ways** to …" — counting over sequential choices.
- "**Minimum/maximum** cost / value / length to reach …" — optimization with composable structure.
- "**Can it be done**" (partitioned, segmented, matched) — feasibility over choices.
- "Longest/shortest **subsequence**" (not subarray!) — almost always DP (subarrays often yield to windows/Kadane; subsequences rarely do).

**2. Structural signals.** Decisions made left-to-right where the future depends only on a *summary* of the past (position, capacity left, last element taken, holding a stock or not); two sequences compared; intervals merged/split; "adjacent elements interact" rules (can't rob neighbors, can't take same letter twice in a row).

**3. The elimination route.** Greedy fails (you can construct a counterexample to every local rule) and brute force is exponential — but the brute-force recursion's *argument list is small*. That last observation is the tell: exponential tree, polynomial distinct states → DP.

**4. Constraint arithmetic.** n ≤ 20 → bitmask DP (2ⁿ states) or backtracking. n ≤ 500 → O(n³) interval DP is fine. n ≤ 10⁴–10⁵ → O(n²) borderline-to-dead; expect O(n log n) (LIS patience) or O(n) (Kadane). Two strings ≤ 1000 each → O(n·m) table intended. Constraints are the setter telling you the state count.

**When DP FAILS — check before committing:**

| Situation | Why DP breaks | Use instead |
|---|---|---|
| A greedy exchange argument actually holds | Table is correct but wasteful | Greedy (prove it!) |
| Subproblems don't overlap | Memo never hits | Divide & conquer |
| State needs the full history (no compact summary suffices) | State space explodes | Search/backtracking, or find a better state |
| "Subarray" problems with monotone structure | DP works but O(n²) | Sliding window / Kadane / monotonic stack |
| Need the optimal *and* adversary moves alternate | Plain max/min misses the opponent | Minimax DP (still DP — flip max/min per turn) |

---

## Top-Down vs Bottom-Up, and Space Optimization

**Top-down (memoized recursion):** write the brute-force recursion, add a cache. Pros: states define themselves, unreachable states never computed, order handled by recursion. Cons: stack depth, constant-factor overhead. **Start here when the transition is complex** (intervals, bitmasks, games).

**Bottom-up (tables):** explicit loops in dependency order. Pros: fast, no stack risk, enables space optimization. Cons: you must *derive* the fill order. **Convert to this when you need performance or rolling arrays.**

The conversion is mechanical: memo dimensions → table dimensions; recursion base → table init; recursive calls → reads of already-filled cells; fill order = any topological order of the dependency DAG (usually "smaller index first" or "shorter interval first").

**Space optimization — the rolling array:** when `dp[i][*]` depends only on `dp[i−1][*]`, keep two rows (or one, iterated cleverly): O(n·m) → O(m). The signature trap: **0/1 knapsack's single-array form must iterate capacity right-to-left** — left-to-right lets item i be counted twice (you'd read a cell already updated *with item i*), silently turning 0/1 knapsack into unbounded knapsack. The direction *is* the semantics; more in Pattern 4.

---

## Pattern 1: 1-D Linear DP — One Decision Per Position

**Logic:** Process elements left to right; `dp[i]` summarizes the best/count for the prefix ending at (or covering) position i; the transition looks back a constant number of steps or consults a small set of prior states.

**Core insight — why it works:** These problems have a **constant-width frontier**: the future interacts with the past only through the last one or two positions (you can't rob adjacent houses; a step covers 1 or 2 stairs; a digit pairs with at most its predecessor). So the entire history compresses into O(1) numbers per position, the last-move partition is tiny ("end with a 1-step or a 2-step"), and the whole computation is one O(n) sweep — often with O(1) space since only the last few dp values are ever read. Kadane's algorithm is exactly this with the state "best subarray *ending here*": the ending-here framing is what makes extension-vs-restart a legal local decision.

**Template (House Robber — the decide-per-element shape):**
```cpp
// dp[i] = max loot from the first i houses (house i-1 considered last)
int prev2 = 0, prev1 = 0;                  // dp[i-2], dp[i-1] — O(1) space
for (int x : nums) {
    int cur = max(prev1, prev2 + x);       // skip house i  vs  rob it
    prev2 = prev1; prev1 = cur;
}
return prev1;
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 70. Climbing Stairs | Easy | Fibonacci in costume: ways(n) = ways(n−1) + ways(n−2), partitioned by the last step's size. The "what was the last move" method in its purest form. |
| 198. House Robber | Medium | The decide-per-element archetype (template above). Say the state sentence before coding — it disambiguates "first i houses" vs "ending at house i", which differ in base cases. |
| 213. House Robber II | Medium | Circular constraint resolved by case analysis: run 198 twice — houses [0, n−2] and [1, n−1] — because house 0 and house n−1 can't both be robbed. Reducing new constraints to solved problems is a core DP move. |
| 746. Min Cost Climbing Stairs | Easy | Same shape, min instead of count. The off-by-one in "pay when you leave a step" is the entire problem — pin down the state sentence. |
| 91. Decode Ways | Medium | dp[i] = decodings of the first i chars; last move = decode 1 digit (if nonzero) or 2 digits (if "10"–"26"). Zeros are the minefield: '0' alone contributes nothing, "30" kills the branch. |
| 53. Maximum Subarray (Kadane) | Medium | dp[i] = best subarray **ending at** i = max(nums[i], dp[i−1] + nums[i]); answer = max over all i. The ending-here state is reused in a dozen problems — internalize it as a primitive. |
| 152. Maximum Product Subarray | Medium | Kadane with a twist: negatives flip extremes, so track **both** max and min ending here; a negative element swaps their roles. "Carry both extremes" is a reusable repair. |
| 139. Word Break | Medium | dp[i] = "first i chars segmentable"; last move = the final word. Cross-listed from the DFS guide — same DP, two skins (memoized vs table). |
| 740. Delete and Earn | Medium | Bucket by value, then it *is* House Robber on the value axis (taking value v kills v±1). Recognizing reductions to known DPs is worth more than new recurrences. |
| 983. Minimum Cost for Tickets | Medium | dp[day] = min cost covering travel through that day; last move = which pass ends today (1/7/30-day). Look-back over windows instead of fixed steps. |

**Pitfalls:**
- "First i elements" vs "ending at i" are *different states* with different base cases and different answer locations (last cell vs max over cells). Most 1-D bugs are an unacknowledged mismatch between the two.
- Index-shift discipline: dp sized n+1 with dp[0] = empty prefix avoids most off-by-ones; dp[i] then refers to element i−1. Pick the convention and announce it.
- O(1)-space rolling kills your ability to *reconstruct* the choices — if the problem asks for the actual selection, keep the table (or parent pointers).

---

## Pattern 2: LIS Family — Subsequence DP

**Logic:** `dp[i]` = best subsequence statistic **ending exactly at element i**; transition scans all j < i with a compatibility test (`nums[j] < nums[i]`): `dp[i] = 1 + max(dp[j])`. O(n²) — with an O(n log n) upgrade via patience sorting when only the length is needed.

**Core insight — why it works:** Subsequences (unlike subarrays) skip elements, so no window or frontier works — but a subsequence is still fully *extendable* knowing only its **last element**: whether x can be appended depends on nothing else. So "ending at i" is a sufficient state and the O(n²) scan enumerates the last-but-one element. The O(n log n) insight is a dominance argument: among all increasing subsequences of length L, only the one with the **smallest tail** matters (anything extendable from a larger tail is extendable from a smaller one). Keep `tails[L] = smallest tail of any length-(L+1) IS` — provably sorted — and each new element binary-searches its slot (first tail ≥ x) and replaces it. Length of `tails` = LIS length. The array's *contents* are not an actual subsequence — a famous misconception worth disclaiming out loud.

**Template (both versions):**
```cpp
// O(n^2): dp[i] = LIS ending at i
vector<int> dp(n, 1);
for (int i = 0; i < n; i++)
    for (int j = 0; j < i; j++)
        if (nums[j] < nums[i]) dp[i] = max(dp[i], dp[j] + 1);
// answer = *max_element(dp.begin(), dp.end());

// O(n log n): patience tails
vector<int> tails;
for (int x : nums) {
    auto it = lower_bound(tails.begin(), tails.end(), x);  // first tail >= x
    if (it == tails.end()) tails.push_back(x);             // extends longest
    else *it = x;                                          // dominates: smaller tail, same length
}
// answer = tails.size();
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 300. Longest Increasing Subsequence | Medium | Both versions above; interviews routinely demand the O(n log n) follow-up *and* the explanation of why replacing a tail is safe (dominance, not reconstruction). |
| 673. Number of Longest Increasing Subsequences | Medium | Carry a second array `cnt[i]` alongside `dp[i]`: on a strictly longer extension reset the count, on a tie add. The "augment the DP with counts" pattern, broadly reusable. |
| 354. Russian Doll Envelopes | Hard | 2-D LIS: sort by width ascending, **height descending within equal widths** (so equal widths can't chain), then LIS on heights. The tie-break trick is the entire problem. |
| 646. Maximum Length of Pair Chain | Medium | LIS-shaped, but intervals admit a greedy (sort by end, take compatible) — the rare family member where greedy beats DP. Knowing *which* relatives have greedy shortcuts is the skill. |
| 368. Largest Divisible Subset | Medium | Sort, then LIS with divisibility as the compatibility test — plus parent pointers to reconstruct the subset. Reconstruction drill. |
| 1048. Longest String Chain | Medium | LIS over words with "predecessor by one deletion" as compatibility; hashmap over sorted-by-length words. The compatibility test is pluggable — that's the family's signature. |
| 1218. Longest Arithmetic Subsequence II (fixed diff) | Medium | dp as a hashmap: `dp[v] = dp[v − diff] + 1` — when "the previous element" is *computable*, the O(n²) scan collapses to O(1) lookups. |
| 1964. Longest Valid Obstacle Course | Hard | Per-prefix LIS lengths (non-strict) — pure patience-tails with `upper_bound`. Strict vs non-strict ⇔ lower vs upper bound: a two-character decision that flips correctness. |

**Pitfalls:**
- `lower_bound` vs `upper_bound` encodes strict vs non-decreasing — derive it from a tiny example each time rather than trusting memory.
- The tails array is **not** the LIS itself; if reconstruction is required, keep the O(n²) dp with parents (or tails-with-indices + per-element predecessor records).
- In 354-style multi-key LIS, the descending tie-break on the second key is what prevents illegal same-width chains — skipping it passes small tests and fails hidden ones.

---

## Pattern 3: Grid DP — Paths Through a Lattice

**Logic:** `dp[r][c]` = best/count for reaching cell (r, c), with movement restricted (usually right/down). Transition = combine the predecessors: `dp[r][c] = grid[r][c] + min(dp[r−1][c], dp[r][c−1])`. Fill row-major; first row/column are the base cases.

**Core insight — why it works:** Movement restrictions make the grid a **DAG** (you can never return to a cell), and the position alone is a sufficient state — *how* you reached (r, c) doesn't change what's ahead. The last-move partition is geometric: every path arrives from above or from the left, disjointly and exhaustively. Row-major order is just a topological order of this DAG. Two power-ups recur: **rolling rows** (each row depends only on the previous → O(cols) space), and **reverse-direction DP** when the constraint propagates backward — Dungeon Game's "minimum health needed *from here on*" must be computed from the princess back to the start, because forward DP can't express "health must never dip below 1 in the future."

**Template (min path sum, rolled to one row):**
```cpp
vector<int> dp(n, INT_MAX);
dp[0] = 0;
for (int r = 0; r < m; r++)
    for (int c = 0; c < n; c++) {
        if (c > 0) dp[c] = min(dp[c], dp[c-1]);   // dp[c] still holds "from above"
        dp[c] += grid[r][c];                       // (r=0 handled by INT_MAX guard)
    }
return dp[n-1];
```
*(Interview-safe version: keep the 2-D table first; roll only when asked.)*

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 62. Unique Paths | Easy | Counting flavor: dp = up + left. Also C(m+n−2, m−1) by combinatorics — offering both shows range. |
| 64. Minimum Path Sum | Medium | The template. Initialize the first row/column as running sums — they have only one predecessor. |
| 63. Unique Paths II | Medium | Obstacles zero out cells: `dp = 0` at obstacles *before* combining. The obstacle-at-start edge case is the hidden test. |
| 120. Triangle | Medium | Jagged grid; the slick version DPs **bottom-up the triangle** so no edge cases exist at row ends and the answer lands in one cell. Direction choice as elegance. |
| 931. Minimum Falling Path Sum | Medium | Three predecessors (up-left, up, up-right) with edge clamping. Predecessor-set generalization drill. |
| 221. Maximal Square | Medium | A different state genus: dp[r][c] = **side of the largest square whose bottom-right corner is here** = 1 + min(up, left, up-left). The min-of-three is a bottleneck argument — all three sub-squares must support the expansion. Memorize the *argument*, not the formula. |
| 174. Dungeon Game | Hard | The reverse-DP flagship: dp[r][c] = min health needed entering (r,c) = max(1, min(right, down) − value). Forward DP provably can't work here (the binding constraint is in the future) — articulating *why* is the interview content. |
| 1289. Minimum Falling Path Sum II | Hard | Must switch columns each row: track the previous row's best and **second-best** (with column), giving O(n²) instead of O(n³). The top-two trick from tree-diameter reappears. |
| 2304. Minimum Rounds / grid-cost variants | Medium | Generic "grid + per-row transition cost" shape — recognize the lattice DAG and the rest is bookkeeping. |

**Pitfalls:**
- First row/column initialization: they have one predecessor, not two — folding them into the main loop without guards reads garbage memory or INT_MAX overflow (`INT_MAX + grid[r][c]` wraps; guard before adding).
- Rolling-array read-order: within a row, `dp[c−1]` must already be *this* row's value while `dp[c]` still holds *last* row's — the in-place update order encodes this; scrambling it is silent corruption.
- Dungeon-style problems: if your forward DP needs "and also never drop below X along the way," stop — that's a future-constraint, flip the direction.

---

## Pattern 4: Knapsack Family — Capacity as a Dimension

**Logic:** Items + a capacity/target; `dp[i][c]` = best/count using the first i items with capacity c. **0/1** (each item once): take it (`dp[i−1][c−w] + v`) or skip (`dp[i−1][c]`). **Unbounded** (unlimited copies): take from `dp[i][c−w]` — same row, allowing repeats. Subset-sum, coin change, and partition are all reskins.

**Core insight — why it works:** The frontier here isn't positional — it's **resource-shaped**: after deciding items 1..i, everything the future needs to know is *how much capacity remains*. That single number is the sufficient summary, which is why capacity becomes a state dimension. The 0/1-vs-unbounded distinction lives in one index: reading row i−1 means "item i no longer available" (0/1); reading row i means "still available" (unbounded). Compressed to one array, this becomes the famous **loop direction rule** — capacity right-to-left preserves 0/1 (you read pre-item-i values), left-to-right *deliberately* enables unbounded. And for counting problems, the **loop nesting order** is semantic, not stylistic: items-outer counts each multiset once (combinations — 518); capacity-outer counts every ordering (permutations — 377). Two interchangeable-looking loops, two different questions answered.

**Template (0/1 subset-sum, one array — note the direction):**
```cpp
vector<char> dp(target + 1, false);
dp[0] = true;                                   // empty subset reaches 0
for (int x : nums)
    for (int c = target; c >= x; c--)           // RIGHT-TO-LEFT: each item once
        dp[c] = dp[c] || dp[c - x];
return dp[target];

// Unbounded (coin change min): for c LEFT-TO-RIGHT, dp[c] = min(dp[c], dp[c-coin] + 1)
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 416. Partition Equal Subset Sum | Medium | Subset-sum to total/2 (odd total → instant no). The cleanest 0/1 specimen and the place to *demonstrate* the right-to-left rule. |
| 494. Target Sum | Medium | Algebra first: positives P with P − (S−P) = T ⇒ P = (S+T)/2 — reduces ±-assignment to subset-sum *counting*. The transform is the problem; parity/negativity checks are the edge cases. |
| 322. Coin Change | Medium | Unbounded min-coins; init dp[>0] = INF and guard `dp[c−coin] != INF` before +1. The single most-asked DP question in interviews — make it reflexive. |
| 518. Coin Change II | Medium | Counting **combinations**: coins outer, amount inner. Swap the loops and you've silently solved 377 instead — the canonical demonstration of loop-order semantics. |
| 377. Combination Sum IV | Medium | Counting **permutations**: amount outer, coins inner. Pair with 518 and explain the difference in one sentence: "outer loop = what's fixed across the count." |
| 279. Perfect Squares | Medium | Unbounded knapsack with items = squares. (BFS on remainders also works in O(same) — naming the equivalence is a nice flex.) |
| 474. Ones and Zeroes | Medium | Two capacities (zeros, ones) → 2-D capacity, both iterated right-to-left. Multi-resource knapsack. |
| 1049. Last Stone Weight II | Medium | Disguise: smashing stones ⇔ partition into two groups minimizing difference ⇔ subset-sum closest to total/2. Seeing through the story is the test. |
| 879. Profitable Schemes | Hard | Knapsack with a "≥ threshold" dimension: clamp profit at the threshold to keep states finite. The clamp-a-dimension trick generalizes. |
| 2218. Maximum Value of K Coins from Piles | Hard | Grouped knapsack: per pile, choose how many top coins to take. dp[pile][k] with an inner choice loop — knapsack where "items" are bundles. |

**Pitfalls:**
- The loop-direction bug is *the* knapsack bug: 0/1 with left-to-right capacity quietly becomes unbounded, passing examples where items happen not to repeat. Say the rule aloud as you write the loop.
- INF arithmetic: `INF + 1` overflows; either guard reads or use INF = a safe sentinel like `amount + 1`.
- Counting problems overflow `int` routinely — and some demand answers mod 1e9+7; check before the off-by-overflow hunt.
- 518 vs 377: if your combination counter returns suspiciously large numbers, your loops are inverted.

---

## Pattern 5: Two-Sequence DP — The Alignment Table

**Logic:** Two strings/arrays; `dp[i][j]` = answer for *prefixes* A[0..i) and B[0..j). The transition cases on the **last characters**: if they match, extend the diagonal; otherwise take the best of dropping one tail or the other (or paying an edit). Fill row-major; row 0/column 0 = one sequence empty.

**Core insight — why it works:** Any alignment/matching of two sequences ends in one of a constant number of ways — last chars matched, A's last char deleted, B's last char inserted, one substituted — and each ending reduces the problem to a *pair of shorter prefixes*. Pairs of prefixes are only (n+1)(m+1) distinct states, so the exponential space of alignments collapses into an O(nm) table where each cell is a constant-case decision. The diagonal/left/up reads have stable *meanings* (match/substitute, delete, insert) across the whole family — LCS, edit distance, distinct-subsequence counting, and even regex matching are one table with different case logic. Space rolls to two rows whenever reconstruction isn't needed.

**Template (edit distance):**
```cpp
vector<vector<int>> dp(n + 1, vector<int>(m + 1));
for (int i = 0; i <= n; i++) dp[i][0] = i;        // delete all of A's prefix
for (int j = 0; j <= m; j++) dp[0][j] = j;        // insert all of B's prefix
for (int i = 1; i <= n; i++)
    for (int j = 1; j <= m; j++)
        if (a[i-1] == b[j-1]) dp[i][j] = dp[i-1][j-1];          // free match
        else dp[i][j] = 1 + min({ dp[i-1][j-1],                  // substitute
                                  dp[i-1][j],                    // delete from A
                                  dp[i][j-1] });                 // insert into A
return dp[n][m];
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 1143. Longest Common Subsequence | Medium | The family's reference table: match → diagonal+1; else max(up, left). Everything else in this pattern is a case-logic variation on it. |
| 72. Edit Distance | Hard | The template above. Interviewers love asking what each of the three reads *means* — have the insert/delete/substitute mapping ready, not just the formula. |
| 583. Delete Operation for Two Strings | Medium | n + m − 2·LCS, or direct deletes-only DP. Reductions to LCS are a recurring shortcut — check for one before building a new table. |
| 97. Interleaving String | Medium | dp[i][j] = "can A's first i and B's first j interleave into C's first i+j" — the third index is *implied* (i+j), shrinking a 3-D-looking problem to 2-D. Implied-dimension spotting is a power move. |
| 115. Distinct Subsequences | Hard | Counting flavor: match → diagonal + up (use the char or don't); mismatch → up. Counts explode — `unsigned long long` or the mod, immediately. |
| 712. Minimum ASCII Delete Sum | Medium | Edit distance with weighted deletes only. Cost-function generality drill: same table, different arithmetic. |
| 10. Regular Expression Matching | Hard | The boss: `*` cases into "zero of the preceding element" (dp[i][j−2]) or "one more" (dp[i−1][j] if heads match). Write the two cases as English sentences first — the code is unforgiving of fuzzy case logic. |
| 44. Wildcard Matching | Hard | Simpler sibling: `*` = empty (left) or eats one char (up). Do 44 before 10 — the case structure transfers. |
| 1035. Uncrossed Lines | Medium | "Non-crossing connections" *is* LCS verbatim — geometry as costume. Recognizing isomorphic problems is most of senior-level DP. |
| 718. Maximum Length of Repeated Subarray | Medium | The sub**array** cousin: dp[i][j] = common suffix length ending exactly at (i, j); mismatch resets to 0; answer = max cell. Contrast with LCS to feel the subsequence/substring state difference. |

**Pitfalls:**
- The off-by-one convention: dp[i][j] covers prefixes of length i and j, so the characters compared are `a[i-1]`, `b[j-1]`. Mixing 0-indexed strings with 1-indexed tables without announcing it = guaranteed bug.
- Base row/column are real answers (empty-string cases), not zeros-by-default — edit distance's bases are i and j, not 0.
- In 10/12-style matching, enumerate the `*` semantics in comments *before* coding; patching cases mid-flight under pressure is how this problem eats 40 minutes.

---

## Pattern 6: Interval DP — Solve by Span Length

**Logic:** `dp[i][j]` = answer for the contiguous range [i, j]; transitions split the interval at some k or peel its endpoints; the table is filled **by increasing interval length**, since any split produces strictly shorter intervals.

**Core insight — why it works:** The discovery move is choosing the *right* last decision — the one that makes the pieces **independent**. Burst Balloons is the canonical lesson: recursing on the *first* balloon burst leaves two halves that still interact through their shared boundary (neighbors change as bursts happen); recursing on the **last** balloon burst means, at that final moment, it neighbors exactly the interval's fixed walls i−1 and j+1 — and the two sides, fully burst before it, never interacted across it. Independence restored, recurrence legal. This "pick the decision that decouples" is interval DP's repeating trick (last burst, the matrix-chain's outermost multiplication, the triangle containing a fixed edge). Fill-by-length is just the topological order of "longer depends on shorter." Costs run O(n²) states × O(n) splits = O(n³) — fine for n ≤ ~500, which the constraints will confirm.

**Template (longest palindromic subsequence — endpoint peeling):**
```cpp
// dp[i][j] = LPS length within s[i..j]
vector<vector<int>> dp(n, vector<int>(n, 0));
for (int i = n - 1; i >= 0; i--) {          // i descending ⇔ length ascending
    dp[i][i] = 1;
    for (int j = i + 1; j < n; j++)
        dp[i][j] = (s[i] == s[j]) ? dp[i+1][j-1] + 2
                                  : max(dp[i+1][j], dp[i][j-1]);
}
return dp[0][n-1];
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 516. Longest Palindromic Subsequence | Medium | The endpoint-peel reference (template above). Also = LCS(s, reverse(s)) — another reduction worth naming. |
| 5 / 647. Palindromic Substrings | Medium | The substring cousins: dp[i][j] = "is s[i..j] a palindrome" (ends match AND inner is) — though expand-from-center (two-pointers guide) beats the table on space. Choosing *against* DP knowingly scores points. |
| 312. Burst Balloons | Hard | The last-burst reframe described above — the single most instructive interval problem. dp[i][j] over *open* intervals with virtual 1-padded walls; get the boundary convention right first. |
| 1039. Minimum Score Triangulation | Medium | Fix the edge (i, j); choose the triangle's third vertex k: dp[i][j] = min over k of dp[i][k] + dp[k][j] + cost. Burst Balloons' geometry twin. |
| 1547. Minimum Cost to Cut a Stick | Hard | Sort cut positions; dp over cut-index intervals; cost of a cut = current segment length. "Add virtual endpoints, DP on the sorted cuts" — a mini-genre. |
| 664. Strange Printer | Hard | dp[i][j] with the merge insight: if s[i] == s[k], printing them can share a stroke. Interval DP where the transition needs a non-obvious observation — read editorials *after* a real attempt. |
| 546. Remove Boxes | Hard | Needs an extra state dimension (i, j, k = same-colored boxes attached to the left). The lesson: when intervals alone aren't a sufficient state, *extend* the state rather than abandon the pattern. Final-boss tier. |
| 87. Scramble String | Hard | Intervals on two strings simultaneously: (i, j, length). Cross-listed from DFS+memo — top-down is far saner here than tables. |
| 1000. Minimum Cost to Merge Stones | Hard | Merge k-at-a-time: feasibility check `(n−1) % (k−1) == 0` plus dp[i][j][piles]. The generalization stress-test of the whole pattern. |

**Pitfalls:**
- Fill order: any loop where dp[i][j] reads a not-yet-computed longer/equal interval is garbage-in. "i descending, j ascending" or an explicit `for len` loop — verify the reads before trusting output.
- Boundary conventions in Burst Balloons (open vs closed intervals, the virtual 1s) cause more failures than the recurrence — fix the convention in a comment line one.
- O(n³) is the budget: if n is 2000, interval DP as-is is dead; look for monotonicity optimizations or a different pattern — the constraints warned you.

---

## Pattern 7: State-Machine DP — The Stock Problems

**Logic:** At each timestep you occupy one of a few **named states** (holding a stock / not holding; cooldown; j transactions used), and the input moves you between them. `dp[day][state]` = best value in that state after that day; transitions follow the machine's arrows.

**Core insight — why it works:** When "what you can do next" depends on a small categorical mode — not just position — the fix is to make the mode part of the state, and the cleanest mental model is an explicit **finite-state machine**: draw circles (hold, empty, cooldown), label arrows (buy: −price, sell: +price, rest: 0), and the recurrence *is* the diagram — each state's new value = best over its incoming arrows. This dissolves the entire stock series into one method: 122 is the 2-state machine; 309 adds a cooldown node; 714 puts the fee on an arrow; 123/188 stack k copies of the machine. Drawing the diagram before coding converts a "hard DP" into transcription, and presenting it that way is exactly what interviewers reward.

**Template (best time with cooldown — 3-state machine):**
```cpp
// hold: own a stock; sold: sold today (cooldown next); rest: free to buy
long long hold = LLONG_MIN, sold = LLONG_MIN, rest = 0;
for (int p : prices) {
    long long prevSold = sold;
    sold = hold + p;                       // arrow: sell
    hold = max(hold, rest - p);            // arrow: buy (from rest only!)
    rest = max(rest, prevSold);            // arrow: cooldown expires
}
return max(sold, rest);
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 121. Best Time to Buy and Sell Stock I | Easy | One transaction: track min-so-far, max profit — technically the degenerate machine; also solvable as Kadane on daily deltas. |
| 122. Stock II | Medium | Unlimited transactions: the 2-state machine (hold/empty). The greedy "sum all positive deltas" is equivalent — show the machine, mention the greedy. |
| 714. Stock with Transaction Fee | Medium | 122 with the fee subtracted on the sell arrow. One-token diff from 122 — diff them consciously. |
| 309. Stock with Cooldown | Medium | The 3-state machine above; the bug to avoid is buying directly from "sold" (skipping cooldown) — the `prevSold` temp enforces ordering. |
| 123. Stock III | Hard | At most 2 transactions: four rolling variables (buy1, sell1, buy2, sell2), each fed by the previous. Two machine copies chained. |
| 188. Stock IV | Hard | At most k transactions: dp[k][2]; clamp k ≥ n/2 to "unlimited" (122) or the table wastes itself. The clamp is the hidden test. |
| 1911. Maximum Alternating Subsequence Sum | Medium | Two states (last taken at even/odd position) — the machine pattern outside stocks. |
| 376. Wiggle Subsequence | Medium | States = last move was up / was down. Machine framing makes the O(n) one-pass obvious where raw DP looks O(n²). |
| 935. Knight Dialer | Medium | States = keypad digits, arrows = knight moves, count paths of length n. The machine generalizes to *any* small transition graph — counting walks in a graph is dp[step][node]. |

**Pitfalls:**
- Update-order aliasing: computing `hold` from an already-updated `rest` (or `sold`) silently lets two actions happen in one day. Snapshot previous-day values (the `prevSold` temp) or compute all news from all olds.
- Initialization: impossible states start at −∞ (`LLONG_MIN`-ish; beware adding to it — use a large-negative sentinel that survives addition), not 0.
- 188's missing clamp turns an O(nk) solution into TLE-by-pointless-k — read the constraints' relationship between k and n.

---

## Pattern 8: Bitmask DP — Sets as States

**Logic:** When the state must record *which elements of a small set are used* (n ≤ ~20), encode the subset as an integer bitmask: `dp[mask]` (or `dp[mask][i]` = best way to have used set `mask`, currently at element i). Transition: pick the next unused element, set its bit.

**Core insight — why it works:** Order-dependent problems over n items naively have n! histories — but often the future depends only on **which** items are used (and maybe the last one), not the order they were used in. All orderings reaching the same mask collapse into one state: n! → 2ⁿ, which is the entire trick (20! ≈ 2.4×10¹⁸ hopeless; 2²⁰ ≈ 10⁶ trivial). Iterating masks in increasing numeric order is automatically a valid fill order, since a state's predecessors have strictly fewer bits — every submask is numerically smaller. The canonical-position optimization (assign "the lowest unused index" next, when slots are interchangeable) cuts another factor by killing symmetric orderings. Constraint tell: **n ≤ 20 with assignment/ordering/partition flavor ⇒ think bitmask before anything else.**

**Template (min-cost assignment — dp over masks):**
```cpp
int full = 1 << n;
vector<int> dp(full, INT_MAX);
dp[0] = 0;
for (int mask = 0; mask < full; mask++) {
    if (dp[mask] == INT_MAX) continue;
    int i = __builtin_popcount(mask);          // canonical: fill person i next
    for (int j = 0; j < n; j++)
        if (!(mask & (1 << j)))
            dp[mask | (1 << j)] = min(dp[mask | (1 << j)],
                                      dp[mask] + cost[i][j]);
}
return dp[full - 1];
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 526. Beautiful Arrangement | Medium | Count permutations with a per-position divisibility rule; popcount(mask) = position being filled. The friendliest first bitmask problem. |
| 698. Partition to K Equal Sum Subsets | Medium | dp[mask] = remainder of the current bucket; a full bucket resets to 0. Cross-listed from DFS — the memoized and table forms are the same DAG. |
| 847. Shortest Path Visiting All Nodes | Hard | (node, mask) BFS — cross-listed from the BFS guide; here note *why* mask is in the state: revisits allowed, so "where + what's covered" is the sufficient summary. |
| 1986. Minimum Work Sessions | Medium | dp[mask] = (sessions, remaining time in the open session) — lexicographic pair minimization over masks. Pair-valued DP cells. |
| 1125. Smallest Sufficient Team | Hard | dp over *skill* masks (not people): for each person, `dp[mask | personSkills]` improves from `dp[mask]`. The mask describes coverage, not membership — read which set the bits mean. |
| 473. Matchsticks to Square | Medium | 698 with k = 4; bitmask DP or pruned backtracking both pass — compare them for the "when does memo beat pruning" discussion. |
| 943. Find the Shortest Superstring | Hard | TSP in costume: dp[mask][last] with pairwise-overlap edge costs. The full classical bitmask-TSP — know its shape even if you never finish it live. |
| 1434. Number of Ways to Wear Hats | Hard | Flip the small side: 40 people but 10⁹ hat-assignments? No — iterate **hats** (≤ 40) over **people-masks** (≤ 2¹⁰). Choosing which dimension to mask is the problem. |

**Pitfalls:**
- Masking the wrong (large) dimension — 1434's whole lesson. The mask goes on whichever set is ≤ ~20.
- Bit hygiene: `1 << j` overflows int at j ≥ 31 (use `1LL <<`), and `mask & (1 << j)` needs parentheses under `==` (precedence).
- 2ⁿ × n² transitions at n = 20 is ~4×10⁸ — borderline. Trim with the popcount-canonical trick or precomputed submask lists; quote the arithmetic before claiming it passes.

---

## Pattern 9: Tree DP — States on Subtrees

**Logic:** DP where subproblems are **subtrees**: `dp[node][state]` computed post-order (children before parent), combining child answers. Covered operationally in the Binary Tree guide (Pattern 2 — the return/record split); here, the DP-side view: states per node × O(1) combine = O(n).

**Core insight — why it works:** Trees hand you the subproblem decomposition for free — every node's subtree is a self-contained instance, children's subtrees are disjoint, and post-order is the topological order. What makes it *DP* rather than plain recursion is the **multi-state cell**: when a node's best depends on a decision *about itself* (robbed or not; covered/has-camera/needs-camera), return one value per state and let the parent combine compatible child states. The disjointness of child subtrees is what makes the combine a clean sum/max with no double-counting — the property array DP never gets for free. Rerooting (computing answers for *every* node as root in O(n) via a second top-down pass) is the advanced move worth knowing by name.

**Template (House Robber III — two states up the tree):**
```cpp
// returns {best if node skipped, best if node robbed}
pair<long long,long long> dfs(TreeNode* node) {
    if (!node) return {0, 0};
    auto [lSkip, lRob] = dfs(node->left);
    auto [rSkip, rRob] = dfs(node->right);
    long long rob  = node->val + lSkip + rSkip;          // rob ⇒ children skipped
    long long skip = max(lSkip, lRob) + max(rSkip, rRob); // skip ⇒ children free
    return {skip, rob};
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 337. House Robber III | Medium | The two-state reference above — House Robber's adjacency constraint mapped onto parent-child edges. |
| 124. Binary Tree Maximum Path Sum | Hard | Cross-listed from DFS: the return/record split *is* tree DP with one returned state plus a global. |
| 968. Binary Tree Cameras | Hard | Three states with a greedy combine — the hardest common tree-state design; write the three states' definitions before any code. |
| 543 / 1245. Diameter (binary / general) | Easy/Med | One-state tree DP; in the n-ary version, the top-two-children trick replaces left/right. |
| 834. Sum of Distances in Tree | Hard | The rerooting flagship: post-order pass for subtree sizes/sums, then pre-order pass converting parent's answer to each child's in O(1). Two sweeps, all-roots answers, O(n). |
| 2246. Longest Path With Different Adjacent Characters | Hard | Diameter with a compatibility predicate on edges — the pluggable-test idea (LIS family) appearing on trees. |
| 1372. Longest ZigZag Path | Medium | Two states per node (arrived via left / via right) — direction as state, a clean small multi-state drill. |
| 333. Largest BST Subtree | Medium | Return (min, max, size, isBST) tuples and validate at each node — tree DP returning *structural summaries* rather than costs. |

**Pitfalls:**
- State-compatibility errors in the combine (robbing a node but reading a child's *robbed* value) — write the compatibility rule as a comment per line.
- Rerooting sign errors (moving the root across one edge adds size and subtracts (n − size)) — derive on a 3-node example before generalizing.
- All the DFS-guide pitfalls (return vs record, leaf vs null) apply verbatim — this pattern and that one are the same animal in different lighting.

---

## Cheat Sheet: Problem Phrase → Pattern

| Phrase in the problem | Pattern |
|---|---|
| "ways to climb / decode / reach n", adjacent-interaction rules | 1-D linear (P1) |
| "longest increasing/divisible/chained **subsequence**" | LIS family (P2) |
| "paths in a grid", "min path sum", "maximal square" | Grid DP (P3) |
| "subset summing to target", "fewest coins", "partition equally" | Knapsack (P4) |
| counting where order matters vs not | 377-vs-518 loop order (P4) |
| two strings: edit / align / match / interleave / count embeddings | Two-sequence table (P5) |
| "burst / merge / remove from a contiguous range", costs depend on neighbors | Interval DP (P6) |
| buy/sell with cooldowns, fees, limited transactions | State machine (P7) |
| n ≤ 20, assignment / ordering / partition of a set | Bitmask (P8) |
| best value over subtrees, parent-child constraints | Tree DP (P9) |
| "subarray" with monotone structure | maybe NOT DP — window/Kadane/stack first |

---

## Complexity Summary

- Universal formula: **O(states × transition cost)** — and state-counting is the feasibility check.
- 1-D linear: O(n), usually O(1) space. LIS: O(n²) or **O(n log n)** patience.
- Grid: O(mn), rolls to O(min(m, n)) space. Two-sequence: O(nm), rolls to two rows.
- Knapsack: O(n × capacity) — *pseudo*-polynomial (capacity is a number, not a size); say the word.
- Interval: O(n²) states × O(n) splits = **O(n³)** — budget for n ≤ ~500.
- State machine: O(n × |states|). Bitmask: O(2ⁿ × n) to O(2ⁿ × n²) — n ≤ ~20.
- Tree DP: O(n × states per node). Rerooting: two O(n) passes.

---

## Interview Tips

1. **Say the state sentence first, always.** "dp[i][j] = minimum edits converting the first i chars of A into the first j of B." A precise sentence makes the recurrence falsifiable; a vague one makes the whole solution unreviewable — and interviewers grade the sentence.
2. **Derive transitions by "what was the last move?"** and name the partition: "every way to reach stair n ends with a 1-step or a 2-step — disjoint and exhaustive, so they add." That's the correctness proof in one breath.
3. **Count states before coding.** "n·target ≈ 2×10⁷ states, O(1) each — fits." This is the feasibility check, the complexity analysis, and a senior signal, all in one line of arithmetic.
4. **Start top-down when transitions are gnarly** (intervals, bitmasks, games), then convert if asked — and say that's your plan. Correct-but-recursive beats elegant-but-wrong in every rubric.
5. **Verify base cases against the state sentence** — most wrong DPs are wrong at dp[0]. The empty prefix/interval/set must satisfy the sentence literally.
6. **Know the famous traps cold:** knapsack's loop direction (0/1 = right-to-left), 518-vs-377 loop nesting, state-machine update aliasing, interval fill order, LIS's tails-array-isn't-the-sequence. Each has appeared in thousands of interviews; flagging one before it bites is worth more than silent correctness.
7. **Offer the space optimization, mention the cost:** "this rolls to O(m) space — though we lose reconstruction; keep the table if the actual path is wanted."

---

## Suggested Practice Order

**Week 1 — linear & grids:** 70 → 746 → 198 → 213 → 91 → 53 → 152 → 62 → 64 → 120
**Week 2 — knapsack & LIS:** 416 → 494 → 322 → 518 → 377 → 279 → 1049 → 300 → 673 → 354
**Week 3 — sequences & machines:** 1143 → 583 → 72 → 97 → 718 → 121 → 122 → 309 → 714 → 123
**Week 4 — boss fights:** 115 → 44 → 10 → 516 → 312 → 1039 → 1547 → 188 → 526 → 698 → 337 → 968 → 834

Good luck with the interviews!

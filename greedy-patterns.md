# Greedy — The Complete Interview Pattern Guide (C++)

Greedy is the most dangerous interview topic — not because the code is hard (it's usually the *shortest* code of any pattern), but because greedy is the only family where a plausible solution can be **confidently, silently wrong**. A greedy algorithm makes irrevocable local choices; it's correct only when the problem has special structure, and the interview is really testing whether you can *prove* (or at least argue) that structure exists. This guide covers the proof techniques, every major greedy pattern, the logic, the **core insight** per pattern, C++ templates, strong problem sets with descriptive notes, and the pitfalls.

---

## What Greedy Actually Is

> **A greedy algorithm builds the solution one irrevocable choice at a time, each chosen by a local rule, never reconsidering.** It's correct exactly when the problem has the greedy-choice property: some locally-best option is provably contained in *an* optimal solution.

Compare the family tree: **DP** considers all choices and keeps the best per state — always correct when states are right, but pays in time/space. **Backtracking** considers all choices and can undo. **Greedy** considers one choice and commits — O(n log n) typical, O(1) state, but correctness is a theorem you owe per problem. The trade is power for proof obligation.

**The two conditions** (mirror of DP's two):
1. **Greedy-choice property:** a locally optimal first choice can be extended to a globally optimal solution. This is what exchange arguments establish.
2. **Optimal substructure:** after committing the choice, the remaining problem is a smaller instance of the same problem. (Shared with DP — greedy is DP where the transition's max/min collapses to one provably-best option.)

That last sentence is the deepest framing: **greedy is a degenerate DP** — when you can prove one branch of the recurrence always dominates, the table collapses to a single running state. This is why the fallback ladder, when a greedy idea fails, is: add an undo mechanism (regret greedy, Pattern 8) → fall back to full DP. The ladder turns "my greedy broke" from a dead end into a pivot.

---

## The Proof Toolkit — How to Argue Greedy Is Correct

Interviews don't demand formal proofs, but they absolutely grade the two-sentence sketch. Three techniques cover everything:

**1. The Exchange Argument** (workhorse — covers ~70% of problems).
Template: *"Take any optimal solution OPT that differs from greedy's choice. Swap OPT's choice for greedy's. Show the swap keeps the solution feasible and no worse. Repeat until OPT equals greedy — therefore greedy is optimal."*
Example (earliest-end-first interval scheduling): "Suppose OPT's first interval isn't the earliest-ending one e. Replace OPT's first interval with e: e ends no later, so everything after still fits — same count, still feasible. So *some* optimal solution starts with e; recurse on the rest."
The skill is identifying **what to swap** and **why feasibility survives** — usually because greedy's choice is extreme in the dimension that constrains (ends earliest, weighs least, expires soonest).

**2. Staying Ahead / Invariant Induction** (~25%).
Template: *"After processing the first i elements, greedy's partial state is at least as good as any other strategy's — by induction. At the end, 'at least as good' means optimal."*
Example (Jump Game): "Invariant: `reach` = the farthest index reachable using the first i elements — no strategy can exceed it because it's the max over all of them. If i ever exceeds `reach`, no strategy reaches i."
The skill is choosing the **one number** (or small tuple) that dominates all strategies simultaneously.

**3. Counterexample Hunting** (the disproof — equally important).
Before trusting any greedy rule, spend 60 seconds trying to kill it: tiny inputs (n = 3–5), extreme values, ties, and adversarial orderings. Killing your own rule costs a minute; having the interviewer kill it costs the round. The classic graveyard: coin change greedy fails on coins {1, 3, 4} for amount 6 (greedy: 4+1+1; optimal: 3+3) — greedy-on-coins works for *canonical* systems like US coins but is not a general algorithm, and knowing that distinction cold is a standard checkpoint.

**Recognizing greedy problems:**
- "Minimum number of X to cover/achieve Y", "maximum items selected under a simple constraint" — selection/coverage shapes.
- A natural **sort key** suggests itself (deadline, end time, ratio, size) and the constraint is one-dimensional.
- n up to 10⁵–10⁶ with an optimization ask — DP tables don't fit; the setter intends a sort + sweep.
- The decision *now* doesn't change the *rules* later (no interactions that make early choices poison future options) — when choices do interact, suspect DP.

---

## Pattern 1: Interval Scheduling — Earliest End First

**Logic:** To select the maximum number of non-overlapping intervals: sort by **end time**, sweep left to right, take every interval that starts at/after the last accepted end. Variants: minimum removals (n − kept), minimum arrows/points to hit all intervals (each accepted end is a "shot").

**Core insight — why it works:** The earliest-ending interval is the choice that **leaves the most room** — formally, the exchange argument above: any optimal solution's first interval can be swapped for the earliest-ending one without losing feasibility (it ends no later) or count. Sorting by *start* instead has clean counterexamples (one long early-starting interval blocks two short ones) — the sort key must be the **resource the future cares about**, and the future cares only about when you become free. The same theorem powers "minimum points to stab all intervals": shoot at the earliest deadline (end), because any interval you'd group with it must be alive at that moment.

**Template (max non-overlapping count):**
```cpp
sort(iv.begin(), iv.end(),
     [](auto& a, auto& b) { return a[1] < b[1]; });   // by END — the theorem's key
int count = 0;
long long freeAt = LLONG_MIN;
for (auto& it : iv)
    if (it[0] >= freeAt) {        // >= or > : read the touching rule per problem!
        count++;
        freeAt = it[1];
    }
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 435. Non-Overlapping Intervals | Medium | Min removals = n − max kept; the template plus a subtraction. The exchange argument is the expected narration — practice saying it in two sentences. |
| 452. Minimum Arrows to Burst Balloons | Medium | Stab-all variant: arrow at each accepted end; *touching balloons share an arrow*, so the comparison flips to `>` vs 435's `>=`. The boundary semantics differ per problem — read, don't assume. |
| 646. Maximum Length of Pair Chain | Medium | LIS-shaped but interval-natured: earliest-end greedy beats the O(n²) DP. The flagship example of "looks like DP, is actually greedy" — say why (chains are interval selections). |
| 252 / 253. Meeting Rooms I / II | Easy/Med | I: sort + adjacent overlap check. II is *not* this pattern (it's min resources, not max selection) — heap of end times or sweep line. Distinguishing selection from packing is the test. |
| 1235. Maximum Profit in Job Scheduling | Hard | The boundary post: **weighted** interval scheduling breaks the greedy (a heavy interval may beat three light ones) — DP + binary search over end-sorted jobs. Knowing exactly where the theorem stops is worth as much as the theorem. |
| 757. Set Intersection Size At Least Two | Hard | Stab-all needing 2 points per interval: sort by end (start *descending* on ties), greedily reuse the two latest points. Earliest-deadline reasoning under a thicker requirement. |
| 1024. Video Stitching | Medium | Cross-listed to Pattern 2 — *covering* a segment is reach-greedy, not selection-greedy; adjacent patterns that beginners conflate. |

**Pitfalls:**
- Sorting by start: the classic, counterexampled failure. If you catch yourself typing `a[0] < b[0]` for a selection problem, stop and re-derive.
- Touching-endpoint semantics (`>=` vs `>`) flip between 435 and 452 — they encode whether [1,3] and [3,5] conflict, and the problems disagree.
- Weighted variants void the theorem (1235): the exchange swap can now lose value. Audit for weights before reciting the greedy.

---

## Pattern 2: Coverage & Reach — The Furthest-Extent Sweep

**Logic:** Cover a line/range (reach the last index, stitch [0, T], water a garden) using elements that each cover an interval. Sweep left to right maintaining the **furthest point reachable** with the current budget; when the sweep is about to pass the current coverage edge, you *must* commit another unit — and the provably best unit is the one extending coverage furthest.

**Core insight — why it works:** A staying-ahead argument on one number: `reach` = the farthest coverage achievable with k committed units — by induction it dominates every other strategy's frontier, because it's defined as the max over all available extensions. Jump Game II's layered form makes the hidden structure visible: positions reachable in k jumps form an interval, so the minimum-jumps problem is **implicit BFS where each "layer" is a range** — `[curEnd+1, furthest]` is layer k+1 — and the greedy "jump only when forced, to the furthest option" is exactly BFS's layer expansion without a queue. That equivalence (greedy = BFS on interval-shaped layers) is the satisfying *why* behind a family that otherwise feels like magic.

**Template (Jump Game II — minimum units to cover):**
```cpp
int jumps = 0, curEnd = 0, furthest = 0;
for (int i = 0; i < n - 1; i++) {            // note: n-1 — don't jump FROM the goal
    furthest = max(furthest, i + nums[i]);   // best extension seen in this layer
    if (i == curEnd) {                       // layer exhausted — must commit
        jumps++;
        curEnd = furthest;                   // next layer's edge
        // if (curEnd >= n - 1) break;       // optional early exit
    }
}
return jumps;
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 55. Jump Game | Medium | Feasibility only: single `reach` invariant, fail when `i > reach`. The two-line greedy whose proof (staying ahead) you should be able to give on demand. |
| 45. Jump Game II | Medium | The template; narrate the BFS-layers equivalence — it's the difference between "memorized" and "understood" in the interviewer's notes. |
| 1024. Video Stitching | Medium | Same machinery over clips: per layer, take the clip extending furthest among those starting within current coverage. Sort by start first. |
| 1326. Min Taps to Water a Garden | Hard | Taps are intervals [i−r, i+r]: precompute, per start position, the furthest end any interval starting ≤ it reaches — then it *is* Jump Game II. Reduction recognition. |
| 134. Gas Station | Medium | A cousin via a different invariant: if total gas ≥ total cost an answer exists; when the running tank dips negative at i, **no station in the failed stretch can start** (each had ≥ 0 tank entering and still failed) — restart at i+1. The prefix-invalidation lemma is the proof to give. |
| 763. Partition Labels | Medium | Cut greedy: extend the current chunk to the max last-occurrence of any char seen; cut when i reaches it. `reach` reused as "earliest legal cut point." |
| 330. Patching Array | Hard | Coverage over *sums*: maintain `miss` = smallest unrepresentable value; if next number ≤ miss it extends coverage to miss+num; else patch with miss itself (doubling coverage). The most elegant staying-ahead proof on LeetCode — learn it as a showpiece. |
| 871. Min Refueling Stops | Hard | Coverage where the *choice* of unit needs hindsight → the regret heap (Pattern 8). Listed as the bridge: when "furthest extension" isn't knowable at commit time, add the undo heap. |

**Pitfalls:**
- 45's loop bound `i < n − 1`: including the last index counts a phantom jump *from* the goal. The single most common Jump-II bug.
- Treating coverage problems as selection problems (sorting by end à la Pattern 1) — covering wants maximal *extension*, selection wants minimal *footprint*; the sort keys are opposite.
- 134: restarting at every failure index one-by-one is O(n²); the prefix-invalidation lemma is what licenses the O(n) skip — state it or the optimization looks like a guess.

---

## Pattern 3: Sort by the Right Key — Comparator Design

**Logic:** Many greedy problems are *entirely* the sort: find the ordering under which a simple sweep is optimal. The key may be a single field (deadline, size), a derived quantity (difference of two costs, ratio), or a **pairwise comparator** (`a before b iff a+b > b+a`). The sweep afterward is usually trivial; the comparator is the algorithm.

**Core insight — why it works:** A pairwise comparator *is* an exchange argument compiled into `sort`: "in any optimal arrangement, no adjacent pair improves by swapping" — and since adjacent swaps generate all permutations (bubble-sort fact), local non-improvability under a **consistent** (total, transitive) comparator implies global optimality. The two-sentence proof per problem is then just the adjacent-swap calculation: for Largest Number, comparing concatenations `a+b` vs `b+a` directly measures the swap's effect; for Two City Scheduling, sorting by `costA − costB` orders people by *how much the choice matters*, sending each where their relative saving is largest. Transitivity is the hidden obligation — `a+b > b+a` happens to be transitive (provable via fractional value normalization), but an arbitrary plausible comparator may not be, and a non-transitive comparator makes `std::sort` UB. Mentioning that unprompted is a flex that lands.

**Template (Largest Number — the concatenation comparator):**
```cpp
vector<string> s(nums.size());
transform(nums.begin(), nums.end(), s.begin(),
          [](int x) { return to_string(x); });
sort(s.begin(), s.end(),
     [](const string& a, const string& b) { return a + b > b + a; }); // the swap test
if (s[0] == "0") return "0";                  // all zeros edge case
string ans;
for (auto& x : s) ans += x;
return ans;
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 179. Largest Number | Medium | The comparator flagship (template above). Bring the transitivity caveat and the all-zeros edge case yourself. |
| 1029. Two City Scheduling | Medium | Sort by `costA − costB`; first half → A. Proof: swapping any A-person with any B-person changes cost by exactly the difference of their differences — sorted order makes every such swap non-improving. |
| 406. Queue Reconstruction by Height | Medium | Sort height **descending** (k ascending on ties), insert each at index k: people already placed are all taller, so they're the only ones the new person "sees" — and later (shorter) insertions are invisible to everyone placed. Invariance by construction. |
| 561. Array Partition | Easy | Sort, sum every other element. Exchange proof: pairing adjacent sorted elements wastes the least "min capacity" — any crossing pair can be uncrossed without loss. The friendliest exchange-argument drill. |
| 870. Advantage Shuffle | Medium | The "Tian Ji's horses" classic: sort both; beat each opponent value with your *smallest sufficient* card; if your smallest card can't beat their largest, sacrifice it there. Greedy matching with a sacrifice rule — cross-listed with Pattern 4's two-pointer mechanics. |
| 1665. Minimum Initial Energy | Medium | Sort by `actual − minimum` (the "wasted requirement"): do high-slack tasks last. Adjacent-swap calculation as the proof, again. |
| 502 / 630-style deadline sorts | Hard | Sort by the **constraining** dimension (capital threshold, deadline) as phase one of the regret pattern — the comparator choice survives even when the simple sweep doesn't (Pattern 8). |
| 2126 / 1090-style "take best k under labels" | Medium | Sort by value desc, sweep with per-label counters. Sort + capacity bookkeeping — the bread-and-butter shape of phone screens. |

**Pitfalls:**
- Comparator inconsistency: anything non-transitive or violating strict-weak-ordering (e.g., using `>=`) is UB in `std::sort` — crashes or silent garbage. `a+b > b+a` is safe; your improvised ratio comparator may not be.
- Sorting by an intuitive key instead of the constraining one — re-derive via the adjacent-swap test ("if I swap these two neighbors, does the objective ever improve?") rather than trusting the first instinct.
- 406's tie-break (k *ascending* within equal heights) is load-bearing; descending ties insert in the wrong relative order and fail quietly.

---

## Pattern 4: Two-Pointer Pairing — Match Extremes

**Logic:** Pair items from one or two sorted sequences under a constraint: heaviest with lightest (boats), smallest sufficient resource to each demand (cookies), your weakest winner against each opponent. Sort, then converge or sweep with two pointers, committing the greedy pairing at each step.

**Core insight — why it works:** Sorting exposes extremes, and the exchange arguments all have the same shape: **the most constrained item has the fewest viable partners — serve it with the least over-qualified partner.** Boats: the heaviest person pairs with the lightest or no one; if even the lightest doesn't fit, the heavy one sails alone (any other arrangement wastes more capacity — swap to see it). Cookies: give each child the smallest cookie that satisfies them; using a bigger cookie can only impoverish a future, greedier child (swap the cookies — feasibility survives, count can't drop). The mechanics live in the two-pointers guide; what's new here is the proof layer: each pointer move is justified by a one-line exchange, and being able to attach the sentence to the move is what separates this from pattern-matched code.

**Template (Boats to Save People):**
```cpp
sort(people.begin(), people.end());
int i = 0, j = n - 1, boats = 0;
while (i <= j) {
    if (people[i] + people[j] <= limit) i++;  // lightest fits with heaviest — pair them
    j--;                                       // heaviest departs regardless
    boats++;
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 881. Boats to Save People | Medium | The template; the exchange sentence ("pairing the heaviest with anyone but the lightest wastes capacity") is the expected narration. |
| 455. Assign Cookies | Easy | Smallest-sufficient matching; the gentlest full exchange-proof drill — use it to practice the *language*. |
| 870. Advantage Shuffle | Medium | Cross-listed: beat each opponent with your cheapest winner, sacrifice your weakest card to their strongest when you can't. Two exchange rules cooperating. |
| 2410. Max Matching of Players and Trainers | Medium | 455 restated — recognizing isomorphic problems on sight is the actual skill being sampled. |
| 1433. Check If One String Can Break Another | Medium | Sort both, check domination one way or the other — pairing as a feasibility *test* rather than an optimization. |
| 826. Most Profit Assigning Work | Medium | Sort jobs by difficulty and workers by skill; sweep workers while tracking best-profit-so-far among unlockable jobs. Pairing with a running-max instead of a strict partner. |
| 2592. Maximize Greatness | Medium | Tian Ji again, count-only: smallest element that beats each target. The family resemblance across 870/2592/1433 is the lesson. |

**Pitfalls:**
- Pairing two heavies "because they fit" (881 allows only 2 per boat regardless of weight slack) — read capacity semantics literally.
- In smallest-sufficient matching, iterating demands in the wrong order (largest-first with a smallest-first resource pointer) tangles the pointers — sort both the same direction and sweep once.
- These greedy proofs assume **one** constraint dimension; add a second (weight *and* count limits beyond 2) and you're in DP/flow territory — audit the constraints before reciting.

---

## Pattern 5: Frequency Greedy — Most Constrained First

**Logic:** Arrange/schedule items with multiplicities under separation rules ("no two identical adjacent", "cooldown n between repeats"). Greedy rule: **always place the most frequent remaining item** (that's currently legal). Implemented with a heap (heap guide P4) — or, for the classic problems, replaced entirely by a counting **formula** derived from the bottleneck item.

**Core insight — why it works:** The most frequent item is the most *constrained* — it needs the most separation slots — so deferring it can only destroy options (an exchange: swap a deferred max-frequency placement earlier against any other item; separation constraints only loosen). The formula version is a **bottleneck argument**: the max-frequency item alone forces a skeleton of `(maxFreq − 1)` gaps of size `(n + 1)` plus the final row of items tied at maxFreq — everything else fills gaps for free. Hence Task Scheduler's `max(total, (maxFreq − 1) * (n + 1) + countOfMax)` and Reorganize String's feasibility test `maxFreq ≤ (len + 1) / 2`: when the bound is achievable, filling round-robin achieves it; when total items exceed the skeleton, no idle time is ever needed. Deriving the formula from the skeleton picture — rather than reciting it — is what the interviewer is listening for.

**Template (feasibility + formula, Task Scheduler):**
```cpp
array<int,26> cnt{};
for (char c : tasks) cnt[c - 'A']++;
int maxFreq = *max_element(cnt.begin(), cnt.end());
int tied = count(cnt.begin(), cnt.end(), maxFreq);
int skeleton = (maxFreq - 1) * (n + 1) + tied;     // forced by the bottleneck alone
return max((int)tasks.size(), skeleton);            // dense case needs no idles
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 621. Task Scheduler | Medium | Formula above; the heap simulation (heap guide) is the generalizable sibling for when follow-ups break the formula. Present formula + derivation, mention the simulation. |
| 767. Reorganize String | Medium | Feasibility `maxFreq ≤ (n+1)/2`, then either the heap or the slick **fill-even-indices-first** construction (place the max char at 0,2,4,…, then everything else continues the round). The constructive version is O(n) and proof-carrying. |
| 1054. Distant Barcodes | Medium | 767 with distance 2 on arbitrary values — same even-odd fill, sorted by frequency. |
| 358. Rearrange String k Distance Apart | Hard | General k: the heap + cooldown queue is the honest solution; formulas stop scaling here. Knowing where the formula's domain ends is part of the pattern. |
| 409. Longest Palindrome | Easy | Pure counting greedy: all even counts + (odd counts − 1 each) + one center if any odd exists. A 5-line warm-up for frequency reasoning. |
| 1405. Longest Happy String | Medium | Three letters, runs ≤ 2: take the most frequent unless it would make a triple, else the second. The two-deep fallback (heap guide) — frequency greedy with a legality override. |
| 984. String Without AAA or BBB | Medium | 1405 with two letters — derivable by the same "spend the surplus while legal" rule. Do it before 1405. |
| 2335. Minimum Amount of Time to Fill Cups | Easy | Bottleneck formula again: `max(maxCount, ceil(total / 2))`. The skeleton-vs-density dichotomy in miniature — same shape as 621, smaller numbers. |

**Pitfalls:**
- Reciting 621's formula without the `max(total, …)` clamp — dense task sets need zero idles, and the skeleton alone undercounts them.
- 767's feasibility threshold is `(n+1)/2`, not `n/2` — odd lengths grant the extra slot; off-by-one here flips possible/impossible.
- The formula family assumes the *only* constraint is separation of identical items; add adjacency rules between *different* items and you're back to heap simulation or DP — audit before formula-flexing.

---

## Pattern 6: Lexicographic Greedy — The Stack as Output Buffer

**Logic:** Build the lexicographically smallest/largest result under a budget or a keep-count: removing k digits, deduplicating letters while keeping order, merging two sequences maximally. Maintain the answer-so-far as a **stack**: while the incoming character improves on the stack's top *and the rules permit removing the top* (budget remains / it recurs later), pop; then push.

**Core insight — why it works:** Lexicographic order is decided at the **first position of difference**, so improving an early position is worth any sacrifice later — making greedy-by-prefix provably right: if popping the top yields a smaller character earlier, *every* completion of the popped version beats *every* completion of the kept version. The guard conditions encode feasibility: in Remove K Digits the budget `k` limits pops; in Remove Duplicate Letters a character may be popped **only if it occurs again later** (tracked via last-occurrence counts) — otherwise the swap argument breaks because the popped letter is gone forever. This is the monotonic stack (essentials guide P1) re-employed: same mechanics, but the stack now *is the answer*, and the pop condition is a greedy theorem rather than a domination fact.

**Template (Remove K Digits):**
```cpp
string st;                                    // stack-as-string
for (char c : num) {
    while (!st.empty() && k > 0 && st.back() > c) {
        st.pop_back();                        // earlier smaller digit beats anything later
        k--;
    }
    st.push_back(c);
}
st.resize(st.size() - k);                     // leftover budget: trim from the end
int i = 0;
while (i + 1 < (int)st.size() && st[i] == '0') i++;   // strip leading zeros
return st.substr(i);
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 402. Remove K Digits | Medium | The template; the three edge cases (leftover k, leading zeros, empty → "0") are where submissions die — list them before coding. |
| 316 / 1081. Remove Duplicate Letters | Medium | The recurs-later guard plus an in-stack set (each letter at most once). The guard's *why* — popping a non-recurring letter is irreversible — is the expected sentence. |
| 321. Create Maximum Number | Hard | The composition boss: for each split k = i + (k−i), take the max-subsequence of length i from A (this pattern), length k−i from B, and **merge** them by greatest-suffix comparison. Three greedy layers; the merge's suffix-compare subtlety (`lexicographical_compare` on remaining suffixes) is the hard part. |
| 1673. Most Competitive Subsequence | Medium | 402 rephrased with a keep-count instead of a remove-budget: pop while `(stack size − 1) + remaining ≥ k`. Same theorem, inverted bookkeeping. |
| 738. Monotone Increasing Digits | Medium | Find the first descent, decrement it, set everything after to 9 — a one-shot lexicographic fix; cascade the decrement leftward over ties. Greedy on digits without a stack, same first-difference logic. |
| 670. Maximum Swap | Medium | One swap to maximize: rightmost-max bookkeeping — for each position, swap with the largest digit to its right (latest occurrence on ties). First-difference reasoning, single move. |
| 2030. Smallest K-Length Subsequence With Letter Constraints | Hard | 1673 + a quota ("at least r copies of letter c survive") woven into the pop guard. The pattern under multiple simultaneous feasibility rules — final-boss tier. |

**Pitfalls:**
- Forgetting the leftover-budget trim (402): a non-increasing input never pops, and k removals must come off the tail.
- Popping a letter that never recurs (316): the irreversibility guard is the theorem — drop it and small tests still pass.
- 321's merge by *current characters only* (instead of comparing whole remaining suffixes) fails on ties — the documented trap of that problem.

---

## Pattern 7: Two-Pass Sweeps — Neighbor Constraints

**Logic:** Per-element values constrained by **both** neighbors ("higher-rated child gets more candy than each neighbor"): one pass can't see both sides, so do two — left-to-right enforcing the left-neighbor rule, right-to-left enforcing the right-neighbor rule, combining with `max` so both constraints hold.

**Core insight — why it works:** The constraint graph decomposes by direction: "more than left neighbor if rating higher" depends only on the left prefix (solvable by one forward sweep, greedily assigning the minimum legal value), and the right-rule symmetrically by a backward sweep. Taking the pointwise `max` of two valid-for-their-direction assignments satisfies both rule sets simultaneously — and *minimality* survives because each sweep assigned the least value its direction permitted, so the max is the least value both permit. This decompose-by-direction trick is the same one behind Product of Array Except Self and the two-array Trapping Rain Water — prefix and suffix information meeting in the middle. **Recognition cue:** any per-position requirement phrased against both neighbors, or any quantity that is "min/max of something to my left and something to my right."

**Template (Candy):**
```cpp
vector<int> candy(n, 1);
for (int i = 1; i < n; i++)                      // left rule: minimum legal values
    if (ratings[i] > ratings[i-1]) candy[i] = candy[i-1] + 1;
for (int i = n - 2; i >= 0; i--)                 // right rule, merged with max
    if (ratings[i] > ratings[i+1]) candy[i] = max(candy[i], candy[i+1] + 1);
return accumulate(candy.begin(), candy.end(), 0LL);
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 135. Candy | Hard | The pattern's namesake; the minimality-of-max argument is the narration that elevates the answer. The O(1)-space slope-counting alternative exists — mention, don't attempt live. |
| 42. Trapping Rain Water (two-array form) | Hard | Water[i] = min(maxLeft[i], maxRight[i]) − h[i]: prefix-max and suffix-max sweeps meeting per cell. The two-pointer version (two-pointers guide) is the space-optimized sibling — know both lineages. |
| 238. Product of Array Except Self | Medium | Prefix products × suffix products — the same decomposition with multiplication. Cross-listed from prefix sums; here, see it as a two-pass sweep. |
| 821. Shortest Distance to a Character | Easy | Distance to nearest occurrence = min(forward sweep dist, backward sweep dist). The gentlest two-pass drill. |
| 845. Longest Mountain in Array | Medium | Up-run lengths (forward) and down-run lengths (backward); a peak's mountain = up[i] + down[i] + 1 where both positive. Per-position left/right facts combined. |
| 2055. Plates Between Candles | Medium | Nearest-candle-left and nearest-candle-right sweeps + prefix counts → O(1) queries. Two-pass as query preprocessing. |
| 1671. Min Removals to Make Mountain | Hard | LIS from the left and LDS from the right per index — the two-pass skeleton where each pass is itself a non-trivial algorithm (patience LIS). Composition tier. |

**Pitfalls:**
- Merging with assignment instead of `max` in the backward pass — overwrites valid left-rule values and breaks the forward constraint. The `max` *is* the meeting of the two rule sets.
- Initializing candies to 0 instead of 1 — every element has a floor independent of neighbors; bake base constraints into the init.
- Equal neighbors: 135 imposes **no** constraint on ties (equal ratings may get unequal candy) — adding an imagined ≥ rule inflates the answer. Read which relations are actually constrained.

---

## Pattern 8: Regret Greedy — Commit With an Undo Heap

**Logic & insight (condensed — full treatment in the Heap guide, Pattern 7):** When the greedy choice needs future knowledge ("should I refuel here?"), commit cheaply now but keep all alternatives in a heap; when a constraint finally breaks, retract the **worst** past commitment (largest duration, smallest skipped fuel) — provably optimal by a reversed exchange argument. Sort by the forced dimension (position, deadline, capital), sweep, heap the regrets. This is the official escape hatch when Patterns 1–7 fail for lookahead reasons — the middle rung of the ladder before full DP.

**Problems:** 871 (Min Refueling Stops — "retroactively decide you stopped at the best passed station"), 630 (Course Schedule III — drop the longest course when over deadline), 502 (IPO — unlock-then-take-best), 1642 (Furthest Building — retract a ladder from the smallest climb), 1353 (Max Events — earliest-deadline with expiry).

**Pitfall worth re-flagging:** the regret extreme's direction (max vs min heap) comes from re-deriving the exchange per problem — 630 retracts the *longest*, 1642 the *smallest* — never from pattern-matching the previous solve.

---

## When Greedy FAILS — Know the Boundary

| Situation | Why greedy breaks | Use instead |
|---|---|---|
| Choices interact with future *rules* (weights on intervals, item values vs sizes) | Exchange swap loses value | DP (knapsack, weighted scheduling 1235) |
| Coin change, arbitrary denominations | {1,3,4}/6 counterexample — local best ≠ global | DP (322) |
| The greedy needs future knowledge to pick | Commit-now is uninformed | Regret greedy (P8), then DP |
| Multiple interacting constraint dimensions | One sort key can't serve two masters | DP / flows / matching |
| You can't produce an exchange or invariant sketch | The structure probably isn't there | Counterexample-hunt 60s, then pivot to DP |
| Counting or enumerating (not optimizing) | Greedy yields one solution, not all | DP / backtracking |

One-sentence litmus: **greedy is right when the most constrained element has a provably safe best partner/placement — and "provably" means you can say the exchange in two sentences.** No sentence, no greedy.

---

## Cheat Sheet: Problem Phrase → Pattern

| Phrase in the problem | Pattern |
|---|---|
| "max non-overlapping / min removals / min arrows" | Earliest-end-first (P1) |
| "min jumps / taps / clips to cover [0, T]" | Reach sweep (P2) |
| "arrange to form the largest/smallest …" | Comparator design (P3) |
| "pair people/items under a capacity" | Extreme matching (P4) |
| "rearrange so no two identical are adjacent / cooldown n" | Frequency greedy (P5) |
| "lexicographically smallest after removing k" | Stack-as-output (P6) |
| each element constrained by **both** neighbors | Two-pass sweep (P7) |
| greedy needs hindsight ("min stops/swaps with budget") | Regret heap (P8) |
| weighted intervals / coin change / interacting choices | NOT greedy — DP |

---

## Complexity Summary

- Almost everything: **O(n log n)** for the sort + O(n) sweep — the family's signature profile.
- Reach sweeps, two-pass sweeps, frequency formulas: **O(n)** flat.
- Stack-based lexicographic: **O(n)** (push/pop once).
- Regret greedy: **O(n log n)** (sort + heap ops).
- Space: typically **O(1)**–O(n); greedy's degenerate-DP nature is exactly why no table appears.

---

## Interview Tips

1. **Rule, then proof sketch, then code — in that order, out loud.** "Sort by end time; exchange argument: swapping any optimal solution's first interval for the earliest-ending one stays feasible at equal count." The sketch *is* the deliverable; the code is clerical.
2. **Spend 60 seconds trying to kill your own rule** on 4-element adversarial inputs before committing. Self-found counterexamples are a pivot; interviewer-found ones are a verdict.
3. **Own the famous failure cases:** coin change {1,3,4}, weighted interval scheduling, start-time sorting. Citing the counterexample that breaks the *adjacent wrong* approach shows you know the territory, not just the trail.
4. **Derive comparators by the adjacent-swap test** and mention strict-weak-ordering when the comparator is exotic — `sort` with an inconsistent comparator is UB, and saying so preempts the question.
5. **Name your proof technique:** "staying ahead on `reach`" or "exchange on the first differing choice." Labels signal that the argument is a method, not a vibe.
6. **Know the ladder:** plain greedy → regret greedy (heap) → DP. Announcing the pivot ("the choice needs future info — I'll keep a regret heap; if that fails, DP over (i, budget)") turns a stuck moment into the strongest moment of the interview.

---

## Suggested Practice Order

**Week 1 — proofs on easy ground:** 455 → 561 → 409 → 860 → 605 → 1029 → 881
**Week 2 — the core families:** 435 → 452 → 646 → 55 → 45 → 134 → 763 → 179 → 406
**Week 3 — frequency & lexicographic:** 621 → 767 → 1054 → 984 → 1405 → 402 → 316 → 1673 → 738 → 670
**Week 4 — boss fights:** 135 → 330 → 1024 → 1326 → 871 → 630 → 502 → 1642 → 757 → 321

Good luck with the interviews!

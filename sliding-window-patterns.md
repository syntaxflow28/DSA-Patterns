# Sliding Window — The Complete Interview Pattern Guide (C++)

Sliding window is the contiguous-subarray/substring specialist: it turns "check all O(n²) subarrays" into a single O(n) pass by maintaining a window `[left, right]` whose state is updated incrementally instead of recomputed. This guide covers every major variation: the logic, the **core insight** (why it's correct), C++ templates, a strong problem set per pattern with descriptive notes, and the pitfalls.

---

## How to Recognize a Sliding Window Problem

Ask yourself:
- Does the problem ask about a **contiguous** subarray or substring? (Non-contiguous "subsequence" → usually DP, not a window.)
- Is it asking for **longest / shortest / count / max-sum / existence** of a window with some property?
- Is the property **monotonic** — does growing the window only ever push it in one direction (more invalid, or more valid)?
- Can the window's state (sum, counts, distinct elements) be **updated in O(1)** when one element enters or leaves?

If the first two are yes but monotonicity fails (negative numbers in sum constraints are the classic killer), the window is a trap — reach for prefix sums + hashmap instead.

**The universal mental model:** a window is a *state machine*. `add(x)` and `remove(x)` update the state; a `valid()` predicate reads it. Every pattern below is just a different policy for *when* to expand and *when* to shrink.

---

## Pattern 1: Fixed-Size Window

**Logic:** The window has a constant length k. Build the first window in O(k), then slide: each step, one element enters on the right and one leaves on the left. Update the state with exactly those two changes and evaluate.

**Core insight — why it works:** Consecutive windows of size k overlap in k−1 elements — recomputing from scratch redoes almost all of that work, costing O(nk). The slide exploits the overlap: the *difference* between window i and window i+1 is exactly one entering element and one leaving element, so the new state is derivable from the old in O(1). You're computing a difference, not a window. This "pay only for what changed" idea is the seed of the whole sliding-window family.

**Template (max sum of a size-k window):**
```cpp
long long windowSum = 0, best = LLONG_MIN;
for (int right = 0; right < n; right++) {
    windowSum += nums[right];              // element enters
    if (right >= k) windowSum -= nums[right - k];   // element leaves
    if (right >= k - 1) best = max(best, windowSum); // window is full size
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 643. Maximum Average Subarray I | Easy | The purest drill: maintain the rolling sum, divide by k once at the end. If you find yourself re-summing k elements per step, you've missed the point of the pattern. |
| 1456. Max Vowels in Fixed Window | Medium | State = a single vowel counter: `+1` if the entering char is a vowel, `−1` if the leaving one is. Shows the state can be far smaller than the window itself. |
| 1343. Subarrays of Size K, Avg ≥ Threshold | Medium | Rolling sum + compare against `threshold * k` (precomputed) — comparing integers avoids floating-point entirely. |
| 2090. K Radius Subarray Averages | Medium | The "window" is centered: radius k means size 2k+1. Either slide normally with an index offset, or use prefix sums. Watch for `long long` — sums overflow `int`. |
| 1652. Defuse the Bomb | Easy | Circular fixed window: indices wrap with `% n`. Good drill for handling modular index arithmetic without duplicating the array. |
| 567. Permutation in String | Medium | State = two frequency arrays (or one diff array). The slick version maintains a `matches` counter (how many of the 26 letters have equal counts) so each slide checks validity in O(1) instead of O(26). |
| 438. Find All Anagrams in a String | Medium | Identical machinery to 567, but record *every* index where `matches == 26` instead of returning at the first. A two-for-one problem pair worth doing back-to-back. |
| 1100. Substrings of Size K, No Repeats | Medium | Fixed window + distinct-count state: window is valid iff `distinct == k`. Combines this pattern's slide with Pattern 2's frequency-map state. |
| 30. Substring with Concatenation of All Words | Hard | Fixed window of size `words.size() * wordLen`, sliding in word-sized strides from each of `wordLen` starting offsets. A frequency map over *words* instead of chars. |

**Pitfalls:**
- Off-by-one on when the window becomes full: the first complete window ends at `right == k - 1`. Evaluating earlier compares against partial windows.
- Forgetting to remove the leaving element (`right - k`) — symptoms are answers that only ever grow.
- Sums of k elements can overflow `int`; default to `long long` for rolling sums.

---

## Pattern 2: Variable Window — Longest Valid (Shrink When Broken)

**Logic:** Expand `right` every iteration, folding the new element into the state. The moment the window becomes **invalid**, shrink from `left` until validity is restored. Since the window is valid at the end of every iteration, record `right - left + 1` as a candidate answer there.

**Core insight — why it works:** The whole pattern rests on one property: **invalidity is hereditary upward** — if `[left, right]` is invalid (too many distinct chars, too many zeros, sum too large with positives), then every window *containing* it is also invalid. So when the window breaks, `left` is dead: no future `right` can ever resurrect a window starting there, and abandoning it forever is safe. Both pointers move only forward, each at most n steps — so despite the nested `while`, total work is O(n) *amortized* (each element is added once and removed once; the inner loop's lifetime total is n, not n per iteration).

**Template (longest window satisfying a constraint):**
```cpp
int left = 0, best = 0;
unordered_map<char,int> freq;             // window state — varies by problem
for (int right = 0; right < n; right++) {
    freq[s[right]]++;                     // expand: fold s[right] in
    while (/* window invalid */) {        // shrink until the constraint holds
        if (--freq[s[left]] == 0) freq.erase(s[left]);
        left++;
    }
    best = max(best, right - left + 1);   // window is guaranteed valid here
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 3. Longest Substring Without Repeating Characters | Medium | The canonical instance. Invalid when the entering char's count hits 2. The optimized variant stores last-seen indices and *jumps* `left` past the previous occurrence instead of stepping. |
| 904. Fruit Into Baskets | Medium | "At most 2 distinct values" dressed as fruit. State = count map; shrink while `map.size() > 2`. Recognizing the costume is the entire problem. |
| 159 / 340. Longest Substring with At Most K Distinct | Medium | The generalization of 904 (k = 2). The "at most K distinct" window is a primitive you'll reuse verbatim inside Pattern 4's counting trick. |
| 1004. Max Consecutive Ones III | Medium | Reframe first: "flip ≤ k zeros" = "longest window containing ≤ k zeros". State is just one integer (zero count). The reframing skill matters more than the code. |
| 424. Longest Repeating Character Replacement | Medium | Valid iff `windowSize − maxFreq ≤ k`. The famous subtlety: `maxFreq` is never decreased on shrink, so it can go stale — yet the *answer* stays correct, because a stale `maxFreq` only prevents windows **shorter** than the best already found from being accepted. Be ready to defend this. |
| 2024. Maximize the Confusion of an Exam | Medium | 424's twin with a 2-letter alphabet: answer = max over (window with ≤ k 'T's, window with ≤ k 'F's). Two passes or track both counts in one. |
| 1493. Longest Subarray After Deleting One Element | Medium | Window with at most one zero; answer is `windowSize − 1` because the deletion is mandatory. The mandatory-deletion detail is where most wrong answers come from. |
| 1695. Maximum Erasure Value | Medium | Longest-unique-window (problem 3) but maximize window **sum** instead of length — shows the "best" you record is independent of the validity rule. |
| 2958. Longest Subarray With Each Frequency ≤ K | Medium | Shrink while the entering element's frequency exceeds k. Only the element that just entered can be the violator — a useful simplification to notice. |

**Pitfalls:**
- Recording `best` *inside* the shrink loop or before shrinking — the window must be valid when you measure it. In this pattern, measure after the `while`.
- Shrinking with `if` instead of `while`: one entering element can require multiple departures to fix.
- Forgetting to erase zero-count keys from the map when "number of distinct" is the constraint — `map.size()` silently lies otherwise.

---

## Pattern 3: Variable Window — Shortest Valid (Shrink While Valid)

**Logic:** The mirror image of Pattern 2. Expand `right` until the window becomes **valid**, then shrink from `left` *while it stays valid*, recording the window length on every valid shrink step. Each shrink either finds a shorter valid window or breaks validity and returns control to expansion.

**Core insight — why it works:** Here the hereditary property points the other way: **validity is hereditary upward** (any window containing a valid one is valid — more sum, more coverage). That flips the goal: bigger is trivially valid, so the interesting boundary is the *smallest* valid window, and you find it by shrinking aggressively the moment validity is achieved. For each `right`, the shrink discovers the minimal valid window ending at `right`; the global answer is the min over all of them. Same amortized O(n) argument: each element enters once, leaves once.

**Template (minimum size subarray with sum ≥ target, positives only):**
```cpp
int left = 0, best = INT_MAX;
long long sum = 0;
for (int right = 0; right < n; right++) {
    sum += nums[right];                          // expand
    while (sum >= target) {                      // shrink WHILE valid
        best = min(best, right - left + 1);      // record before breaking it
        sum -= nums[left++];
    }
}
return best == INT_MAX ? 0 : best;
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 209. Minimum Size Subarray Sum | Medium | The canonical instance. Works *only* because elements are positive — adding always grows the sum, removing always shrinks it. With negatives, see the pitfall below. |
| 76. Minimum Window Substring | Hard | The boss fight. State = `need` counts plus a single scalar `have` (count of distinct chars fully satisfied). Maintaining `have` incrementally — increment only when a char's count *reaches* its requirement, decrement only when it *drops below* — is what makes validity checks O(1). |
| 1234. Replace the Substring for Balanced String | Medium | Inverted thinking: the window is the part you *replace*, so it's valid when everything **outside** it has no char exceeding n/4. A great test of whether you can re-aim the validity predicate. |
| 2875. Minimum Size Subarray in Infinite Array | Medium | Target sum in an infinitely repeated array: peel off full copies with division/modulo, then run shortest-valid on a doubled array for the remainder. Window + arithmetic preprocessing. |
| 632. Smallest Range Covering K Lists | Hard | Conceptually "minimum window over merged streams": merge all lists (tagged by origin), then find the shortest span containing at least one element from every list — problem 76 in disguise on a merged sorted sequence. |
| 1658. Min Operations to Reduce X to Zero | Medium | Beautiful inversion: removing prefix+suffix summing to x ⇔ keeping a middle window summing to `total − x`. The *shortest-removal* answer comes from the **longest** valid window — a Pattern 2/3 crossover via complement. |

**Pitfalls:**
- Recording the answer at the wrong moment: in shortest-valid, measure *inside* the shrink loop (while still valid), the opposite of Pattern 2. Mixing up the two templates is the most common sliding-window bug there is.
- **Negative numbers break this pattern** (209 follow-up territory): with negatives, shrinking can *increase* the sum, so "shrink while valid" no longer explores correctly. Minimum-length-with-sum-≥-k on signed arrays is 862 (Shortest Subarray with Sum at Least K) and needs prefix sums + monotonic deque. Knowing this boundary cold is a strong interview signal.
- In 76, comparing full frequency maps per step (O(26) or O(map)) instead of maintaining `have` — accepted, but the O(1) version is what the interviewer wants to see.

---

## Pattern 4: Counting Windows ("At Most K" Arithmetic)

**Logic:** Instead of finding one best window, count *all* valid subarrays. Two tools:
1. **Count-while-sliding:** in a longest-valid loop, after restoring validity, every subarray ending at `right` and starting in `[left, right]` is valid → add `right - left + 1` per iteration.
2. **Exactly-K trick:** `exactly(K) = atMost(K) − atMost(K−1)`, where `atMost` is one count-while-sliding pass.

**Core insight — why it works:** Tool 1 works because of the hereditary structure *within* one iteration: if `[left, right]` is valid and validity is upward-closed under shrinking from the left (fewer distinct values, smaller product), then all `right − left + 1` of its suffixes `[left..right], [left+1..right], …, [right..right]` are valid too — and grouping the count **by right endpoint** guarantees each subarray is counted exactly once. Tool 2 works because "exactly K" is rarely monotonic (you can't slide on it directly), but "at most K" always is — so you express the non-monotonic set as the difference of two monotonic ones. Turning a hard predicate into a difference of easy ones is a trick worth keeping far beyond sliding windows.

**Template (count subarrays with at most K distinct values):**
```cpp
long long atMost(vector<int>& nums, int k) {
    unordered_map<int,int> freq;
    long long count = 0;
    int left = 0;
    for (int right = 0; right < (int)nums.size(); right++) {
        freq[nums[right]]++;
        while ((int)freq.size() > k) {
            if (--freq[nums[left]] == 0) freq.erase(nums[left]);
            left++;
        }
        count += right - left + 1;     // all valid subarrays ending at right
    }
    return count;
}
// exactly K distinct:
long long exactlyK = atMost(nums, k) - atMost(nums, k - 1);
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 713. Subarray Product Less Than K | Medium | Count-while-sliding in its purest form: shrink while `product >= k`, then add `right − left + 1`. Edge case: `k <= 1` means no window is ever valid — handle before the loop. |
| 992. Subarrays with K Different Integers | Hard | *The* exactly-K showcase: two `atMost` passes and a subtraction turn a Hard into two Mediums. If you learn one counting problem, make it this one. |
| 1248. Count Number of Nice Subarrays | Medium | "Exactly k odd numbers" — map odds to 1s, evens to 0s, and it's exactly-K-sum: `atMost(k) − atMost(k−1)`. Also solvable with prefix-sum counting; know both. |
| 930. Binary Subarrays With Sum | Medium | Same shape as 1248 with goal sum instead of odd-count. Careful with `goal = 0`: `atMost(−1)` must return 0, not crash. |
| 2302. Count Subarrays With Score Less Than K | Hard | Score = sum × length, which is still monotonic under expansion (both factors grow with positives) — so plain count-while-sliding applies despite the scary product. |
| 1358. Substrings Containing All Three Characters | Medium | A *shortest-valid* counting twist: once `[left, right]` contains all of a/b/c, every **extension to the right** is valid too → add `n − right` per minimal window (count by left endpoint this time). Compare with the suffix-counting above to see both directions. |
| 2962. Count Subarrays Where Max Appears ≥ K Times | Medium | Valid = window holds the global max ≥ k times; once minimal validity is reached at `left`, add `left + 1` (any start ≤ left works). Another count-by-the-other-endpoint drill. |
| 2799. Count Complete Subarrays | Medium | Window must contain *all* distinct values of the whole array — compute the global distinct count first, then it's the 1358/2962 counting shape. |

**Pitfalls:**
- Double counting. Discipline: pick one endpoint to group by (per-`right` suffixes, or per-`left` extensions) and derive the count formula from that grouping — never mix both in one pass.
- `atMost(k - 1)` with `k = 0` or sentinel values: make `atMost` return 0 gracefully for negative arguments.
- Counts overflow `int` fast (up to n(n+1)/2 ≈ 5×10⁹ for n = 10⁵): accumulate in `long long`.

---

## Pattern 5: Window + Auxiliary Structure (Monotonic Deque & Friends)

**Logic:** Sometimes the window's *boundaries* slide normally, but the question asked of the window (its max, min, median, max−min) can't be maintained with simple counters — removing an element would require knowing the "second best". Pair the window with a structure that supports the query *and* expiry of departed elements: a monotonic deque (max/min in amortized O(1)), a `multiset` (max−min, median in O(log n)), or two heaps with lazy deletion.

**Core insight — why it works:** Take the max-query deque: it stores indices whose values are **strictly decreasing**. When `x` enters, every smaller element at the back is popped — and this is safe because such an element is *both older and smaller* than `x`, so it can never be the maximum of any future window that still contains `x`. It's **dominated**, and dominated elements can be discarded the moment their dominator arrives. The deque is exactly the set of "still potentially useful" elements; the front is the current max, and it's evicted only when its index leaves the window. Every element is pushed once and popped once → amortized O(1) per step. The deeper lesson: when O(1) state isn't enough, find a *dominance relation* and keep only the non-dominated frontier.

**Template (sliding window maximum):**
```cpp
deque<int> dq;                            // indices; values strictly decreasing
vector<int> result;
for (int right = 0; right < n; right++) {
    while (!dq.empty() && nums[dq.back()] <= nums[right])
        dq.pop_back();                    // dominated: older AND <= new element
    dq.push_back(right);
    if (dq.front() <= right - k)
        dq.pop_front();                   // front has left the window
    if (right >= k - 1)
        result.push_back(nums[dq.front()]);
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 239. Sliding Window Maximum | Hard | The monotonic-deque flagship (template above). Interviewers usually walk the ladder: brute O(nk) → heap O(n log n) with lazy deletion → deque O(n). Be able to narrate all three rungs. |
| 1438. Longest Subarray, Abs Diff ≤ Limit | Medium | Variable window where validity = `max − min ≤ limit`. Maintain a max-deque *and* a min-deque (or one `multiset`), shrink while the spread exceeds the limit. Pattern 2's skeleton, Pattern 5's state. |
| 480. Sliding Window Median | Hard | Fixed window + a `multiset` with an iterator pinned to the median, nudged left/right depending on which side of it elements enter and leave. Fiddly; understand the multiset version before attempting two-heaps-with-lazy-deletion. |
| 862. Shortest Subarray with Sum ≥ K (negatives!) | Hard | The "negatives broke my window" problem: switch to prefix sums and run a monotonic deque over *them*, popping prefixes that are dominated (later and smaller). The canonical example of where plain windows end and deque-on-prefix begins. |
| 1696. Jump Game VI | Medium | DP where `dp[i] = nums[i] + max(dp[i−k..i−1])` — the range-max over a sliding window of DP values is exactly problem 239's deque. Window-max as a *component* inside another algorithm. |
| 1499. Max Value of Equation | Hard | Algebra first: maximize `(yi − xi) + (yj + xj)` with `xj − xi ≤ k` → deque over `y − x` values with x-coordinates as expiry. Rearranging the objective until a window-max appears is the actual test. |

**Pitfalls:**
- Storing **values** instead of **indices** in the deque — you then can't tell when the front has expired out of the window.
- `<` vs `<=` when popping the back: for max queries, popping equal values (`<=`) keeps the *newest* of the ties, which survives longest; keeping stale duplicates is correct for the max but wastes work and complicates expiry reasoning.
- Mixing eviction order: pop dominated elements from the back *before* pushing; pop expired elements from the front *before* reading the answer.

---

## Pattern 6: Sort First, Then Slide

**Logic:** Some problems have no usable order until you create one. After sorting, "closeness in value" becomes "closeness in position", and a contiguous window over the sorted array represents a set of nearly-equal values — which is exactly what operations like "raise elements to match" care about.

**Core insight — why it works:** Sorting is legal whenever the answer depends on the **multiset of values**, not their original positions (you're choosing a *subset* to make equal, not a contiguous run of the original array). Once sorted, the key quantity becomes computable: to raise every element in window `[left, right]` up to `nums[right]` costs `nums[right] * windowSize − windowSum` — a formula only meaningful because sorting guarantees `nums[right]` is the window's max. Validity (`cost ≤ k`) is then monotonic in window size, and the standard longest-valid machinery takes over. The pattern is really two moves: *manufacture monotonic structure by sorting, then harvest it with a window.*

**Template (max frequency after at most k increments):**
```cpp
sort(nums.begin(), nums.end());
long long sum = 0;
int left = 0, best = 1;
for (int right = 0; right < n; right++) {
    sum += nums[right];
    // cost to lift the whole window up to nums[right]:
    while ((long long)nums[right] * (right - left + 1) - sum > k)
        sum -= nums[left++];
    best = max(best, right - left + 1);
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 1838. Frequency of the Most Frequent Element | Medium | The template problem. The "aha": elements made equal need not be adjacent originally — sorting is what turns the best subset into a window. |
| 2779. Maximum Beauty of an Array After Operations | Medium | Each element can move ±k, so values match iff they're within 2k: sort, then longest window with `nums[right] − nums[left] ≤ 2k`. Validity check is just the two endpoints — sorted order makes extremes live at the edges. |
| 1498. Number of Subsequences Satisfying Sum Condition | Medium | Sort + converging pointers (a window cousin): for each valid `(left, right)` with `min + max ≤ target`, all `2^(right−left)` subsequences anchored at `left` count. Legal to sort because subsequences care only about the value multiset. |
| 2009. Minimum Operations to Make Array Continuous | Hard | Invert: maximize the elements you *keep*. Sort + dedupe, then for each `left`, the keepable elements are those in `[nums[left], nums[left] + n − 1]` — a window over sorted distinct values. |
| 1818. Minimum Absolute Sum Difference | Medium | Adjacent technique: sort a copy and binary-search the best replacement per index. Worth knowing as the sibling of sort+window when you need *per-element nearest value* rather than a best window. |

**Pitfall:** Sorting is **illegal** when the problem says contiguous-in-the-original-array or asks for original indices. The first question to ask out loud: "does the answer survive reordering?" If yes, sorting may unlock a window; if no, it destroys the problem.

---

## When Sliding Window FAILS — Know the Boundary

This list earns more interview points than any single problem:

| Situation | Why the window breaks | Use instead |
|---|---|---|
| Subarray **sum** constraints with negative numbers | Adding an element can shrink the sum → validity not monotonic → shrink/expand decisions become wrong | Prefix sums + hashmap (560), prefix sums + monotonic deque (862) |
| "Subsequence", not "subarray" | Elements need not be contiguous — no window represents the choice | DP, greedy, sort-based methods |
| Count subarrays with sum == k (signed) | Same negativity issue, in counting form | Prefix-sum frequency hashmap (560) |
| Validity depends on the whole arrangement (e.g., "is a palindrome") | Adding one char can flip validity arbitrarily — no hereditary property in either direction | Expand-from-center, DP, hashing |
| Window query needs max/min/median | State isn't O(1)-maintainable with counters alone | Window + monotonic deque / multiset (Pattern 5) |

The litmus test in one sentence: **a window works iff validity changes monotonically as the window grows, and the state updates in O(1)-ish per element.** Lose either property and you need a different tool.

---

## Cheat Sheet: Problem Phrase → Pattern

| Phrase in the problem | Pattern |
|---|---|
| "subarray/substring of size k" | Fixed-size window |
| "longest substring/subarray such that …" | Variable window, longest valid (shrink when broken) |
| "smallest/minimum window containing …" | Variable window, shortest valid (shrink while valid) |
| "count/number of subarrays with …" | Counting + at-most-K arithmetic |
| "exactly k distinct / exactly k odd …" | `atMost(k) − atMost(k−1)` |
| "maximum/minimum of each window", "max − min ≤ limit" | Window + monotonic deque / multiset |
| "make elements equal with ≤ k operations" | Sort first, then slide |
| sum constraint + array contains negatives | NOT a window → prefix sums (+ hashmap or deque) |

---

## Complexity Summary

- Fixed and variable windows: **O(n)** — every element enters once and leaves once (amortized argument covers the nested `while`).
- Exactly-K counting: two O(n) passes, still **O(n)**.
- Window + deque: **O(n)** amortized (each index pushed and popped at most once).
- Window + multiset/heap: **O(n log n)**.
- Sort-then-slide: **O(n log n)** for the sort, O(n) for the slide.
- Space: O(1) for counter states, O(Σ) for frequency maps (Σ = alphabet size), O(k) or O(n) for deque/multiset variants.

---

## Interview Tips

1. **Name the state and the predicate first.** Before writing the loop, say: "state = frequency map + distinct count; valid = distinct ≤ k." Once those are fixed, the template writes itself — and the interviewer sees structured thinking rather than pattern-matching.
2. **Say which template you're in.** "Longest-valid: I shrink when broken and measure after the shrink" vs "shortest-valid: I shrink while valid and measure inside the shrink." Confusing the two measurement points is the most common sliding-window bug; naming it preempts it.
3. **Defend the O(n) claim with amortization.** "The inner while looks nested, but `left` only ever moves forward, at most n steps over the entire run — each element is added once and removed once." This one sentence is frequently asked for verbatim.
4. **Probe the negative-numbers question yourself.** If sums are involved, ask whether elements can be negative *before* coding. Right answer with negatives = prefix sums; asking unprompted = senior signal.
5. **Edge cases to always check:** k = 0 or k larger than n, empty/size-1 input, `atMost(k−1)` at k = 0, all-identical elements (maximizes expansion everywhere), and overflow on rolling sums and subarray counts (`long long`).
6. **For counting, declare your grouping.** "I count by right endpoint: each iteration adds the valid subarrays ending at `right`." A declared grouping makes double-counting nearly impossible.

---

## Suggested Practice Order

**Week 1 — fixed windows & template fluency:** 643 → 1456 → 1343 → 1652 → 567 → 438
**Week 2 — variable windows, longest:** 3 → 904 → 1004 → 1493 → 1695 → 424 → 2024
**Week 3 — shortest + counting:** 209 → 713 → 930 → 1248 → 1358 → 992 → 76
**Week 4 — auxiliary structures & boss fights:** 239 → 1438 → 1838 → 2779 → 1658 → 862 → 480 → 30

Good luck with the interviews!

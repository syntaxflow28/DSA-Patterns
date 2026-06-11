# Sliding Window — Interview Revision Sheet

## The 30-second decision

1. **Window size given** (`of size k`, `of length len(p)`) → **FIXED**
2. **Property the window must satisfy** (`sum ≤ k`, `at most 2 distinct`, `no repeats`) → **VARIABLE** — hunt for the size
3. Then ask the two discriminators:
   - **"exactly K"?** → not a direct count. Use the subtraction trick.
   - **"max / min"?** → max *across* windows = running best (plain). max *inside* a window = irreversible → **deque**.

## Universal 3-step checklist (patterns 1–5)
1. **ADD** — `right` enters, always update window state
2. **SHRINK** — `if` (fixed, one shift) / `while` (variable, may need many)
3. **RECORD** — wherever the window is in its **valid** state
   - longest → record **after** the shrink loop (loop fixes an invalid window)
   - shortest → record **inside** the shrink loop (loop squeezes a valid window)

---

## 1. Fixed size
- **Tell:** "subarray/substring of size exactly `k`" — size is an input.
- **Key line:** `if (right-left+1 > k) { sum -= a[left]; left++; }` then record when `==k`.
- **Trap:** state must be **reversible** (sum, count, avg). If you need max/min *inside* the window, it's NOT this — go to deque.

## 2. Longest (variable)
- **Tell:** "longest window such that <contents constraint holds>".
- **Key line:** `while (invalid) shrink;` then `best = max(best, right-left+1);` **after** the loop.
- **Trap:** recording *inside* the loop = recording a broken window. Record after.

## 3. Shortest (variable)
- **Tell:** "smallest/minimal window such that <constraint satisfied>".
- **Key line:** `while (valid) { best = min(best, right-left+1); shrink; }` — record **before** the shrink, inside loop.
- **Trap:** inverted from longest. Shrink *while valid*, not while invalid.

## 4. Count subarrays (atMost)
- **Tell:** "number of subarrays with **at most** K …".
- **Key line:** `total += right - left + 1;` (counts all valid subarrays ending at `right`).
- **Trap:** `right-left+1` works only because `left` is the furthest-back legal start. Same as longest skeleton + counting line.

## 5. Exactly K
- **Tell:** the word **"exactly"**.
- **Key line:** `exactly(K) = atMost(K) - atMost(K-1)` (guard `atMost(k<0) = 0`).
- **Trap:** "exactly" looks like a direct count — it isn't. Counting exactly-K directly has no clean expand/shrink rule. Always reduce to two atMost calls.

## 6. Monotonic deque (max/min over fixed windows)
- **Tell:** "max/min of **every** window of size `k`" — and the value is *inside* the window.
- **Why:** sum is reversible (subtract on exit); **max is not** — if the max leaves you can't recover the new max without rescanning. Deque fixes this.
- **Mechanism:** deque of **indices**, values decreasing front→back. Front = current max.
  - **Back exit:** newcomer arrives → pop back while `a[back] <= a[right]` (those are useless forever: younger AND taller survivor exists).
  - **Front exit:** pop front when `front <= right - k` (aged out of window).
  - Read answer at **front** once `right >= k-1`.
- **Complexity:** each index pushed once, popped at most once → **O(n) amortized** (the inner `while` is bounded over the whole run, not per-iteration).
- **Trap:** for **min**, flip the back-exit comparison to `>=`. Don't confuse with "max across windows" (that's plain pattern 1).

---

## My personal traps (the ones I actually missed)
- **"exactly k odd numbers"** → exactly-K subtraction, NOT a plain count. State is just a count of odds (no hashmap).
- **"maximum number of vowels in a substring of length L"** → FIXED with a running count, NOT deque. Vowel count is reversible; I'm maximizing *across* windows, not finding the max *inside* one. The named string was pure distraction (only sets window length).

## Master rule
> Don't pattern-match on keywords. Match on the **operation the window needs.**
> Reversible state (sum/count/avg) → plain sliding. Irreversible (max/min inside window) → deque.
> "exactly" → subtraction. "max/min" → ask: across windows or inside one?

---

# WHEN PLAIN SLIDING WINDOW BREAKS (the hard-tier insight)

## The hidden assumption: MONOTONICITY
Every basic window pattern secretly assumes: as the window grows, the
quantity moves in ONE predictable direction, so "shrink when invalid"
is guaranteed to help. When that's false, the greedy slide is unreliable.

**Diagnosis skill (derive this cold):**
1. Is the constraint monotonic in window size? i.e. if a window is invalid,
   is every LARGER window also invalid? If yes → plain slide works.
2. If NO → plain sliding window is impossible. Find the variable breaking it.

**Two things that kill monotonicity:**
- **Negative numbers** in "subarray sum" problems. Adding an element can
  DECREASE the sum; removing a left element can INCREASE it. So "shrink
  when sum too big" may make things worse. → replace with **prefix sums +
  monotonic deque** (e.g. LC 862 Shortest Subarray with Sum at Least K).
  Note: prefix-sum deque is the SAME "younger + smaller-prefix = useless"
  logic as the height/deque trick, comparison flipped.
- **"at least k" constraints** (LC 395). Shrinking removes chars → counts
  go DOWN → moves you AWAY from "appears at least k times". And validity
  isn't monotonic: a bigger window can FIX an invalid one. No shrink rule.

## The escape hatch: FIX THE TROUBLESOME VARIABLE, ITERATE OVER IT
General reusable move (NOT obvious — bank it from having seen it):
> When one free variable makes the problem non-monotonic / intractable,
> PIN it to a constant, solve the now-easy subproblem, loop over all values.

**LC 395 worked example** ("longest substring where every char appears ≥ k"):
- Culprit variable = number of distinct chars (free → unbounded → no slide).
- Fix it: try each `d` = 1..26. "at most d distinct" is now monotonic →
  slideable, exactly like the at-most-k-distinct pattern.
- Within each d-window: record when `distinct == d && atLeastK == d`.
- Only 26 values of d → O(26·n), still linear.

## The "count the CROSSING, not the state" trick
Maintaining a running tally of "how many things satisfy a threshold"?
Update on the exact TRANSITION, never on the condition being true:
- going up:   `if (cnt[c] == k) atLeastK++;`  // the moment it crosses k
- going down: `if (cnt[c] == k) atLeastK--;`  // checked BEFORE decrement
Using `>= k` instead double-counts every step past the threshold. WRONG.

## Honest note for revision
The creative leap (fix-the-variable) is a banked pattern, not something
you'd invent under pressure. What you CAN derive cold = the diagnosis:
"is this monotonic? if not, which variable breaks it?" Do that first,
then reach for the escape hatch.

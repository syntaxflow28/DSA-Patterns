# Two Pointers — The Complete Interview Pattern Guide (C++)

Two pointers is one of the highest-ROI techniques for coding interviews. Almost every variation turns a brute-force O(n²) scan into an O(n) pass. This guide covers every major pattern: the logic behind it, the **core insight** (why it's correct, not just what to do), how to recognize it, C++ template code, classic problems with descriptive notes, and common pitfalls.

---

## How to Recognize a Two-Pointer Problem

Ask yourself:
- Is the input **sorted** (or can I sort it without breaking the problem)?
- Am I looking for a **pair / triplet / subarray** satisfying a condition?
- Do I need to do something **in-place** with O(1) extra space?
- Is it a **linked list** where I can't index backwards or know the length upfront?
- Am I comparing/merging **two sequences**?

If yes to any → two pointers is probably the intended solution.

---

## Pattern 1: Converging Pointers (Opposite Ends)

**Logic:** Place `left = 0` and `right = n-1`. At each step, evaluate the pair `(nums[left], nums[right])` against your condition. Based on the result, move exactly one pointer inward. The loop ends when they cross, having examined the array in a single pass.

**Core insight — why it works:** The brute force checks all O(n²) pairs. Converging pointers gets away with O(n) because **sortedness makes one of the two possible moves provably useless**. Take pair sum: if `nums[left] + nums[right] < target`, then `nums[left]` paired with *anything* to the left of `right` is even smaller (those values are ≤ `nums[right]`). So `nums[left]` can never be part of the answer — we've eliminated `left` and all (right − left) pairs involving it in one comparison. Every step permanently discards one index along with every pair it could have formed. n indices → n steps → O(n). The technique is really a *search-space pruning* argument: you're not checking fewer pairs by luck, you're proving entire rows of the pair-matrix can't contain the answer.

**Template (pair sum in sorted array):**
```cpp
int left = 0, right = nums.size() - 1;
while (left < right) {
    int sum = nums[left] + nums[right];
    if (sum == target) {
        return {left, right};
    } else if (sum < target) {
        left++;        // every pair (left, j<right) is even smaller — discard left
    } else {
        right--;       // every pair (i>left, right) is even bigger — discard right
    }
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 167. Two Sum II (Sorted Array) | Medium | The canonical example. The sorted input is the giveaway — original Two Sum (unsorted, must return original indices) needs a hashmap instead. |
| 15. 3Sum | Medium | Sort, fix `nums[i]` with an outer loop, then run converging pointers on `(i+1, n-1)` searching for `-nums[i]`. Skipping duplicates at all three positions is half the problem. |
| 16. 3Sum Closest | Medium | Same structure as 3Sum, but instead of matching exactly, track the sum with minimum `abs(sum - target)` seen so far. No dedup needed since you only return one value. |
| 18. 4Sum | Medium | Two nested loops fix the first two numbers, two pointers handle the rest → O(n³). Use `long long` for sums; the test cases overflow `int` deliberately. |
| 11. Container With Most Water | Medium | Area = min(height[l], height[r]) × width. Always move the **shorter** wall: it is the bottleneck, and keeping it caps every narrower container at the same or smaller area. |
| 42. Trapping Rain Water | Hard | Water above index i = min(maxLeft, maxRight) − height[i]. Maintain `leftMax`/`rightMax` and always process the side with the smaller max — that side's water level is already decided. |
| 125. Valid Palindrome | Easy | Compare characters from both ends, skipping non-alphanumerics with inner `while` loops and comparing case-insensitively. |
| 680. Valid Palindrome II | Easy | On the first mismatch you get one free deletion: branch into two checks — skip `left` OR skip `right` — and return true if either inner range is a palindrome. |
| 344. Reverse String | Easy | Swap `s[left]` and `s[right]`, converge. The simplest possible instance of the pattern. |
| 977. Squares of a Sorted Array | Easy | After squaring, the largest values live at the two **ends** (negatives flip). Compare ends and fill the result array **backwards** to get sorted output in O(n). |
| 881. Boats to Save People | Medium | Sort, then greedily pair the heaviest person with the lightest. If even the lightest doesn't fit with the heaviest, the heaviest sails alone. |
| 259. 3Sum Smaller | Medium | Counting variant: when `sum < target`, every pair `(left, left+1..right)` works, so add `right - left` in one shot instead of enumerating. |
| 611. Valid Triangle Number | Medium | Sort, fix the **largest** side `c` from the right, converge on the rest checking `a + b > c`. When it holds, all `right - left` pairs with that `right` also hold. |

**Pitfalls:**
- In 3Sum/4Sum, skip duplicates *after* recording a match: `while (left < right && nums[left] == nums[left+1]) left++;` (and symmetric for right), plus dedup the fixed outer element too.
- In Container With Most Water, candidates often can't justify the move. The proof: keeping the shorter wall, every future pair is narrower *and* its height is still capped by that same short wall → strictly ≤ current area. So discarding it loses nothing.

---

## Pattern 2: Slow & Fast — Read/Write Pointers (In-Place Array Modification)

**Logic:** `fast` is the **reader** — it visits every element exactly once. `slow` is the **writer** — it marks where the next "kept" element should go. Whenever the reader finds an element that belongs in the output, copy it to position `slow` and advance `slow`. At the end, `[0, slow)` is your answer, built in-place.

**Core insight — why it works:** The trick is the invariant the two pointers maintain together: **`[0, slow)` is always the correct, finished output for the prefix `[0, fast)` of the input.** Since `slow ≤ fast` at all times, the writer never overwrites data the reader hasn't consumed yet — that single inequality is what makes "build the output inside the input" safe with zero extra memory. Conceptually you're streaming the array through a filter and writing results back into the same buffer, which is exactly why it's O(n) time, O(1) space, and one pass.

**Template (remove duplicates from sorted array):**
```cpp
int slow = 0;
for (int fast = 0; fast < nums.size(); fast++) {
    if (fast == 0 || nums[fast] != nums[fast - 1]) {
        nums[slow++] = nums[fast];   // keep it: write at the boundary, extend the boundary
    }
}
return slow;   // length of the deduped prefix
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 26. Remove Duplicates from Sorted Array | Easy | The canonical example. An element is "new" iff it differs from its predecessor — only possible because the array is sorted, so duplicates are adjacent. |
| 80. Remove Duplicates II (keep ≤2) | Medium | Elegant generalization: keep `nums[fast]` iff `slow < 2 || nums[fast] != nums[slow-2]`. Comparing against the *written* region (not the read region) enforces "at most 2" automatically. |
| 27. Remove Element | Easy | Same skeleton, condition is simply `nums[fast] != val`. |
| 283. Move Zeroes | Easy | Keep = non-zero. Either overwrite-then-zero-fill the tail, or **swap** `nums[slow]` and `nums[fast]` to preserve relative order in one pass. |
| 443. String Compression | Medium | The reader jumps over an entire run of equal chars (inner while loop), then the writer emits the char plus its count digit-by-digit. Reader and writer move at completely different speeds. |
| 905. Sort Array By Parity | Easy | Stable version uses read/write (evens forward); if order doesn't matter, converging pointers with swaps also works. |
| 2460. Apply Operations to an Array | Easy | Two phases: apply the doubling rule in place, then a standard move-zeroes compaction pass. |

**Pitfalls:**
- `slow` ends up being the **length** of the result, not the index of the last kept element. Returning `slow - 1` or iterating `<= slow` are classic off-by-ones.
- In Move Zeroes, the swap version maintains a cleaner invariant (`[slow, fast)` is all zeros) and avoids a second pass; mention both in an interview.

---

## Pattern 3: Fast & Slow — Floyd's Cycle Detection (Tortoise and Hare)

**Logic:** Both pointers start at the head. `slow` advances 1 node per step, `fast` advances 2. If the list ends (`fast` or `fast->next` becomes null), there is no cycle. If there is a cycle, both pointers eventually get trapped inside it, and `fast` catches `slow` from behind.

**Core insight — why it works:** Once both pointers are inside the cycle, look at the gap between them. Each step, fast gains exactly 1 on slow (relative speed 2 − 1 = 1). A gap that shrinks by exactly 1 each step **must pass through 0** — fast can never "jump over" slow. So a meeting is guaranteed within one cycle length, giving O(n) detection with two pointers' worth of memory instead of a hash set of visited nodes.

**Finding the cycle entry (problem 142):** Let the head→entry distance be `a`, and the meeting point be `b` nodes into the cycle of length `c`. Slow traveled `a + b`; fast traveled `2(a + b)`. Fast's extra distance `a + b` must be whole laps: `a + b = kc`, so `a = kc − b ≡ (c − b) mod c`. But `c − b` is exactly the distance from the meeting point forward to the entry. So: reset one pointer to head, advance both 1 step at a time — they collide precisely at the entry.

**Template:**
```cpp
ListNode *slow = head, *fast = head;
while (fast && fast->next) {
    slow = slow->next;
    fast = fast->next->next;
    if (slow == fast) return true;   // cycle exists
}
return false;
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 141. Linked List Cycle | Easy | Pure detection — the template above verbatim. |
| 142. Linked List Cycle II | Medium | Detection plus the reset trick to locate the entry node. Be ready to reproduce the `a = c − b` argument; interviewers ask "why does that work?" almost every time. |
| 876. Middle of the Linked List | Easy | No cycle involved — just the speed ratio: when `fast` reaches the end, `slow` has covered half the distance, i.e. the middle. One pass, no length computation. |
| 234. Palindrome Linked List | Medium | Three techniques chained: fast/slow to find the middle, reverse the second half in place, then parallel-pointer compare the two halves. O(1) space. |
| 202. Happy Number | Easy | No linked list at all! The sequence `n → sumOfSquaredDigits(n)` either reaches 1 or loops forever — Floyd detects the loop. Shows the pattern applies to *any* deterministic successor function. |
| 287. Find the Duplicate Number | Medium | Brilliant reframing: treat the array as a functional graph `i → nums[i]`. A duplicate value means two nodes point to the same successor → a cycle whose entry **is** the duplicate. Floyd solves it in O(1) space without modifying the array. |
| 143. Reorder List | Medium | Find middle (fast/slow), reverse second half, then interleave-merge the two halves. A great test of combining patterns cleanly. |

**Pitfalls:**
- The loop guard must check **both**: `while (fast && fast->next)`. Checking only `fast->next` segfaults on an empty list; only `fast` segfaults on even-length lists.
- Even-length middle ambiguity: with `slow = fast = head`, `slow` lands on the **second** middle. For problem 234 you typically want the first middle — start `fast = head->next`, or handle the extra node when comparing.

---

## Pattern 4: Fixed-Gap Pointers (Lead/Lag)

**Logic:** Advance a `lead` pointer k steps from the head first. Then move `lead` and `lag` together, one step at a time, until `lead` falls off the end. Because the gap between them is frozen at k, `lag` now stands exactly k nodes from the end.

**Core insight — why it works:** "k-th from the end" normally requires knowing the length — which costs a full first pass. The gap technique **encodes the measurement into the distance between two pointers**: you measure k once (at the start, where it's easy) and then carry that measurement to the end of the list rigidly. It's a one-pass answer to a question about the *end* of a structure you can only walk *forward* — which is exactly why it's a linked-list staple and a favorite "can you do it in one pass?" follow-up.

**Template (remove n-th node from end):**
```cpp
ListNode dummy(0, head);
ListNode *lead = &dummy, *lag = &dummy;
for (int i = 0; i < n + 1; i++)     // n+1 so lag stops *before* the target
    lead = lead->next;
while (lead) {
    lead = lead->next;
    lag = lag->next;
}
lag->next = lag->next->next;        // splice the target out
return dummy.next;
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 19. Remove Nth Node From End | Medium | The dummy node is the real lesson: it makes "delete the head" (when n equals the length) identical to every other case — no special-casing. |
| 1721. Swapping Nodes in a Linked List | Medium | Gap pointers locate the k-th from the end while a simple walk finds the k-th from the front; swap the **values** (swapping nodes is needless pain here). |
| 61. Rotate List | Medium | Rotation by k = cutting the list k nodes from the end and reattaching. Either compute length and connect into a ring, or use a gap of `k % length` to find the new tail directly. |

**Pitfall:** The off-by-one between advancing `n` vs `n + 1` steps. Rule of thumb: to **delete**, `lag` must stop at the node *before* the target → start from a dummy and advance lead `n + 1`. To merely **find** the target, `n` suffices.

---

## Pattern 5: Two Pointers Across Two Sequences (Parallel / Merge Pointers)

**Logic:** One pointer per sequence, both starting at the front (or both at the back). At each step, compare the elements under the pointers and advance one (or both) according to the rule. This is the merge step of merge sort, generalized into a comparison-driven walk.

**Core insight — why it works:** When both sequences are sorted (or processed in a consistent direction), the comparison at the current pair of positions is **globally decisive**: if `a[i] <= b[j]`, then `a[i]` is ≤ everything remaining in *both* sequences, so it can be emitted/consumed immediately and never reconsidered. Each step permanently retires one element, so total work is O(m + n) — you never backtrack because the ordering guarantees no future information could change a past decision.

**Template (merge two sorted arrays into a result):**
```cpp
int i = 0, j = 0;
vector<int> result;
while (i < a.size() && j < b.size()) {
    if (a[i] <= b[j]) result.push_back(a[i++]);
    else              result.push_back(b[j++]);
}
while (i < a.size()) result.push_back(a[i++]);   // drain leftovers
while (j < b.size()) result.push_back(b[j++]);
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 88. Merge Sorted Array | Easy | The twist: merge **backwards**. `nums1` has spare room at its end, so write from index `m+n-1` taking the larger of the two tails — forward merging would overwrite unread `nums1` data. |
| 392. Is Subsequence | Easy | Greedy matching: advance both pointers on a character match, only the haystack pointer otherwise. `s` is a subsequence iff its pointer reaches the end. Greedy is safe because matching a character as early as possible never hurts. |
| 844. Backspace String Compare | Easy | Scan both strings **from the end**: a backspace cancels characters you haven't seen yet when going forward, but going backward you can count `#`s and skip exactly that many characters. O(1) space. |
| 350. Intersection of Two Arrays II | Easy | Sort both, walk in parallel: equal → record and advance both; otherwise advance the pointer at the smaller value. |
| 986. Interval List Intersections | Medium | Intersection of intervals A and B is `[max(starts), min(ends)]` if non-empty. Then advance whichever interval **ends first** — it can't intersect anything else. |
| 165. Compare Version Numbers | Medium | Parse one numeric chunk from each string per step (treating exhausted strings as 0), compare, continue. The parallel walk avoids splitting/allocating. |
| 21. Merge Two Sorted Lists | Easy | The list version of the merge template; a dummy head keeps the splicing branch-free. |
| 415. Add Strings | Easy | Right-to-left parallel scan with a carry variable — the grade-school addition algorithm as two pointers. |

**Pitfall:** Problem 88's entire difficulty is the direction. Forward merging into `nums1` destroys data you still need; backward merging writes only into the spare/consumed region. The general principle: **write into space you've already read or that was empty.**

---

## Pattern 6: Sliding Window (Same-Direction Two Pointers)

Technically a two-pointer subfamily — `left` and `right` both move rightward, together defining a window `[left, right]`. Interviews treat it as its own pattern, so know both names.

**Logic (variable window):** Expand `right` greedily, adding elements to the window's running state (counts, sum, etc.). Whenever the window violates the constraint, shrink from `left` until it's valid again. Record the best window seen.

**Core insight — why it works:** The brute force checks O(n²) subarrays. The window gets away with O(n) because of **monotonicity of validity**: if a window violates the constraint (e.g., sum too big, too many distinct chars), then every window *containing* it violates it too. So once `[left, right]` is invalid, there is no point keeping `left` — no extension to the right will ever fix it, and `left` can be retired forever. Both pointers only ever move forward, each at most n steps → O(n) total, even though a single iteration may shrink many times (amortized analysis: every shrink retires one element permanently).

**Template (longest window satisfying a condition):**
```cpp
int left = 0, best = 0;
for (int right = 0; right < n; right++) {
    add(s[right]);                       // expand: bring s[right] into window state
    while (!valid()) {                   // shrink until constraint holds again
        remove(s[left]);
        left++;
    }
    best = max(best, right - left + 1);  // window [left, right] is valid here
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 3. Longest Substring Without Repeating Characters | Medium | The canonical sliding window. State = char frequency map (or last-seen index for the jump optimization); invalid when the entering char's count hits 2. |
| 209. Minimum Size Subarray Sum | Medium | A **minimize** variant: the loop flips — shrink *while the window is valid* (sum ≥ target), recording the best length on each shrink. Positives-only is what makes it work. |
| 424. Longest Repeating Character Replacement | Medium | Window is valid iff `windowSize - maxFreq <= k` (replace everything except the majority char). Subtle: `maxFreq` may go stale on shrink, but never updating it downward still gives the right *answer length* — a famous interview discussion point. |
| 1004. Max Consecutive Ones III | Medium | Identical skeleton to 424 with a simpler state: window valid iff it contains ≤ k zeros. |
| 904. Fruit Into Baskets | Medium | "Longest subarray with at most 2 distinct values" wearing a costume. State = hashmap of counts; shrink while `map.size() > 2`. |
| 567. Permutation in String | Medium | **Fixed-size** window of length `s1.size()`: slide one char in, one char out, and compare frequency arrays (or maintain a `matches` counter for O(1) checks). |
| 438. Find All Anagrams in a String | Medium | Same machinery as 567, but collect every starting index where the frequencies match instead of returning early. |
| 76. Minimum Window Substring | Hard | The boss fight. Track `need` (required counts) and a single scalar `have` (how many distinct chars are fully satisfied). Expand until `have == need.size()`, then shrink aggressively recording the best. |
| 239. Sliding Window Maximum | Hard | The window alone isn't enough — you need a **monotonic deque** holding indices of decreasing values so the max is always at the front. Window + auxiliary structure. |
| 1456. Max Vowels in Fixed Window | Medium | Cleanest fixed-window drill: add the entering char's contribution, subtract the leaving char's, never recompute. |

**Pitfalls:**
- Sliding window **breaks with negative numbers** in sum problems: adding an element can *decrease* the sum, so validity is no longer monotonic and shrinking can be wrong. Use prefix sums + hashmap instead (e.g., 560. Subarray Sum Equals K). Saying this unprompted is a strong signal to interviewers.
- For fixed-size windows, the whole point is the O(1) slide (add entering, remove leaving). Recomputing the window from scratch turns O(n) into O(nk).

---

## Pattern 7: Expand From Center (Diverging Pointers)

**Logic:** The mirror image of converging — start both pointers *together* and move them *apart* while a symmetric condition holds. For palindromes: pick a center, expand `l--, r++` while `s[l] == s[r]`, and the moment they differ you've found the maximal palindrome at that center.

**Core insight — why it works:** A palindrome is defined by symmetry around its center, and **every palindrome has exactly one center** — either a character (odd length) or the gap between two characters (even length). A string of length n has exactly `2n − 1` such centers. So instead of testing O(n²) substrings for palindromicity (O(n³) total), you enumerate O(n) centers and grow each one maximally in O(n) — and crucially, the maximal palindrome at a center *contains* all shorter palindromes at that center, so one expansion captures them all.

**Template:**
```cpp
int expand(const string& s, int l, int r) {
    while (l >= 0 && r < (int)s.size() && s[l] == s[r]) {
        l--; r++;
    }
    return r - l - 1;   // length of maximal palindrome at this center
}

for (int i = 0; i < n; i++) {
    int odd  = expand(s, i, i);       // center = character i
    int even = expand(s, i, i + 1);   // center = gap between i and i+1
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 5. Longest Palindromic Substring | Medium | Try all 2n−1 centers, track the longest expansion and recover its start index from the returned length. O(n²) time, O(1) space — beats the DP solution's O(n²) space. |
| 647. Palindromic Substrings | Medium | Same expansion, but **count** instead of track: each successful step of an expansion is itself one distinct palindromic substring. |
| 658. Find K Closest Elements | Medium | A diverging-pointers relative: binary-search the closest position, then grow outward — or more elegantly, shrink a window of size k by dropping whichever end is farther from x. |

**Pitfall:** Forgetting the **even-length centers** (the `expand(s, i, i+1)` call). It's the single most common bug in this pattern — "abba" has no character at its center.

---

## Pattern 8: Partitioning Pointers (Dutch National Flag / Quickselect)

**Logic:** Use 2–3 pointers to divide one array into labeled regions and maintain those regions as you scan. Sort Colors uses three pointers and four regions: `[0, low)` = 0s, `[low, mid)` = 1s, `[mid, high]` = unexplored, `(high, n)` = 2s. `mid` walks through the unexplored zone, dispatching each element to its region via swaps.

**Core insight — why it works:** Correctness rests entirely on the **region invariants** holding after every single operation. Each case is designed to preserve them: a 0 swapped to position `low` lands in the 0-region and what comes back (always a 1, since `[low, mid)` is the 1-region) is already classified — so `mid` may advance. A 2 swapped to `high` lands correctly, but what comes back is from the *unexplored* region — so `mid` must **not** advance. When `mid` passes `high`, the unexplored region is empty and the invariants *are* the proof of a fully sorted array. This invariant-first way of thinking is exactly how quicksort/quickselect partitions are proven correct too.

**Template (Sort Colors):**
```cpp
int low = 0, mid = 0, high = nums.size() - 1;
while (mid <= high) {
    if (nums[mid] == 0) {
        swap(nums[low++], nums[mid++]);   // incoming element is a known 1 — safe to advance
    } else if (nums[mid] == 1) {
        mid++;                            // already in the right region
    } else {
        swap(nums[mid], nums[high--]);    // incoming element is UNEXAMINED — do not advance mid
    }
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 75. Sort Colors | Medium | The Dutch National Flag problem (Dijkstra). One pass, O(1) space, three regions. Interviewers will probe the "why not advance mid after swapping with high" question. |
| 215. Kth Largest Element | Medium | Quickselect: partition around a (random) pivot with two pointers, then recurse only into the side containing index k. Average O(n); know the Hoare vs Lomuto partition difference at least by name. |
| 324. Wiggle Sort II | Hard | Find the median (quickselect), then three-way partition around it with a virtual index mapping. Brutally tricky — understand the idea even if you'd never be asked to fully code it. |
| 280. Wiggle Sort | Medium | The gentle cousin: one pass, swap `nums[i]` and `nums[i+1]` whenever the required `<`/`>` relationship at position i is violated. Local fixes suffice because each swap can't break the previous pair. |

**Pitfall:** After swapping with `high`, do **not** increment `mid` — the element that arrived came from unexplored territory and hasn't been classified. Advancing anyway is *the* classic Sort Colors bug, and it's exactly what the invariant analysis catches.

---

## Pattern 9: Sort First, Then Two Pointers

Not a separate mechanic, but a crucial *recognition* pattern: many problems aren't two-pointer-able as given — sorting is the unlock that creates the monotonic structure every pattern above relies on.

**Core insight — when sorting is safe:** Sorting costs O(n log n) and destroys original positions. It's the right move when the answer depends only on **values** (pairs, counts, sums, groupings) and not on indices — or when you can carry original indices along as pairs `(value, index)`. Conversely, if the problem demands original indices (classic Two Sum) or contiguous subarrays of the *original* order, sorting is illegal and you should reach for a hashmap or prefix sums instead. Asking "does order matter?" out loud is itself an interview signal.

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 15. 3Sum | Medium | Sorting enables both halves of the solution: the converging-pointer search *and* clean duplicate skipping (duplicates become adjacent). |
| 881. Boats to Save People | Medium | The greedy "pair heaviest with lightest" is only provably optimal on sorted input — the exchange argument needs the ordering. |
| 1099. Two Sum Less Than K | Easy | Sort + converge: when `sum < k`, record it as a candidate max and advance `left`; otherwise retreat `right`. |
| 923. 3Sum With Multiplicity | Medium | Sort + converge + combinatorics: when matching values repeat, count the runs on each side and multiply (or `C(run, 2)` when both sides hit the same value). |
| 18. 4Sum | Medium | One sort up front serves all the nested fixing and the inner two-pointer scan. |
| 2563. Count Fair Pairs | Medium | Count pairs with sum in `[lo, hi]` = (pairs with sum < hi+1) − (pairs with sum < lo), each counted in O(n) with converging pointers on the sorted array. |

**Counting trick worth memorizing:** in a sorted array with converging pointers, if `nums[left] + nums[right] < target`, then **all** of `(left, left+1), …, (left, right)` are valid → add `right - left` in one step and advance `left`. This is how pair-counting problems run in O(n) after the sort, and it shows up constantly (259, 2563, 611).

---

## Cheat Sheet: Pattern → Signal

| Signal in the problem | Pattern |
|---|---|
| Sorted array + find pair/triplet with sum condition | Converging pointers |
| "In-place", "O(1) space", remove/compact elements | Read/write (slow-fast) |
| Linked list + cycle / middle / "without computing length" | Floyd's tortoise & hare |
| "k-th from the end" in one pass | Fixed-gap pointers |
| Two sorted arrays/lists/strings to merge or compare | Parallel pointers |
| Longest/shortest **contiguous** subarray/substring with property | Sliding window |
| Palindromes in a string | Expand from center |
| Sort array into ≤3 groups in-place | Dutch National Flag |
| Pairs/triplets where order doesn't matter | Sort first, then converge |

---

## Complexity Summary

- Nearly all patterns: **O(n) time** after any required **O(n log n) sort**, **O(1) extra space** (the whole point of the technique).
- 3Sum: O(n²) — outer fixing loop × inner two-pointer scan.
- 4Sum: O(n³) — two fixing loops × inner scan.
- Expand from center: O(n²) worst case (string of identical characters maximizes every expansion).
- Sliding window: O(n) even with the nested shrink loop — amortized, since `left` moves at most n times total across the entire run.

---

## Interview Tips

1. **State the invariant out loud.** "Everything in `[0, slow)` is processed and valid", "the unexplored region is `[mid, high]`" — interviewers grade this heavily because the invariant *is* the correctness proof.
2. **Justify every pointer move.** For converging pointers, explain *why* discarding a side is safe ("the shorter wall caps every narrower container"). A correct answer with no justification scores lower than you'd think.
3. **Edge cases to always check:** empty input, single element, all duplicates, target at the boundary, even vs odd length (palindrome centers, middle of list), and integer overflow on sums (4Sum — use `long long`).
4. **Know when two pointers fails:** unsorted data you can't sort because original indices matter (→ hashmap, as in original Two Sum); negative numbers in window-sum problems (→ prefix sums + hashmap); non-contiguous subsequences (→ DP). Naming the failure mode unprompted is a senior-signal.
5. **`while (left < right)` vs `<=`:** use `<` when a valid answer needs two distinct elements; `<=` when a single element can be the answer (binary-search-style loops). Decide deliberately, don't guess.

---

## Suggested Practice Order

**Week 1 — fundamentals:** 344 → 125 → 167 → 26 → 283 → 27 → 977
**Week 2 — core mediums:** 11 → 15 → 75 → 209 → 3 → 876 → 141 → 19
**Week 3 — harder variations:** 142 → 287 → 234 → 424 → 1004 → 5 → 16 → 986
**Week 4 — boss fights:** 42 → 76 → 239 → 18 → 234 + 143 (combined techniques)

Good luck with the interviews!

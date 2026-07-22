# Stack & Monotonic Stack — The Complete Interview Pattern Guide (C++)

A stack answers one question in O(1), forever: **"what was the most recent thing I haven't resolved yet?"** That single LIFO capability powers a surprising range of patterns — bracket matching, expression parsing, undo/collision simulations, and (once you add an ordering rule) the entire *monotonic stack* family: next-greater-element, histogram spans, subarray-contribution counting, and greedy subsequence building. This guide covers them all: the logic, the **core insight** (why it's correct), C++ templates, strong problem sets with descriptive notes, and the pitfalls.

---

## What a Stack Actually Is (and Isn't)

A stack is a last-in-first-out (LIFO) sequence with three O(1) operations: `push` (add to top), `pop` (remove top), `top` (read top). That's the whole contract. Two consequences define everything a stack can and cannot do:

- **Only the top is reachable.** You can inspect and remove the *most recent* element in O(1), and nothing else. This is exactly the right shape for *nesting* (the innermost unclosed scope is always on top) and for *"most recent unresolved"* questions.
- **Order is reversed on the way out.** Push 1,2,3 and you pop 3,2,1. A stack is a reversal machine; when you need things back in arrival order you need a queue (or two stacks — Pattern 8).

**The mental model:** a stack is a *pile of unfinished business*. Every push is "I'll deal with this later"; every pop is "the most recent unfinished thing is now resolved." The **monotonic** stack sharpens this into "the most recent element that could still matter to the future" — everything provably irrelevant gets popped the moment it's dominated.

**What a stack cannot do** (and what to use instead):
- **Look below the top / search / rank:** O(n). If you need the k-th element or arbitrary lookup, use a heap or an ordered structure.
- **Evict from the *bottom*:** a stack is one-ended. Sliding-window extremes (both ends move) need a **monotonic deque**, not a stack.
- **Preserve arrival order out:** a single stack reverses; FIFO needs a queue or the two-stack trick.

## How to Recognize a Stack Problem

- **Matching / nesting:** parentheses, tags, nested encodings — the innermost open item must close first.
- **"Most recent" / "last unmatched" / undo:** you repeatedly need the latest thing and may cancel it.
- **Adjacent collapsing:** elements annihilate or merge with their immediate neighbor (duplicates, collisions, backspaces).
- **Expression evaluation with precedence or nested scopes:** calculators, RPN, `k[encoded]` strings.
- **"Next/previous greater or smaller element"**, "days until warmer", "stock span", "largest rectangle" → **monotonic stack**.
- **"Smallest/largest subsequence after removing k"**, "most competitive" → **greedy monotonic stack**.
- **Iterative DFS** — the call stack made explicit (see the DFS guide; a stack is its engine).

**The litmus test:** if the answer depends on the *nearest* unresolved item (in position, in nesting, or in value order), a stack is almost certainly the tool.

## C++ Stack Survival Kit

```cpp
#include <stack>
stack<int> st;                 // LIFO adaptor (over deque by default)
st.push(x);                    // add to top
st.pop();                      // remove top — returns void!
int t = st.top();              // read top (UB if empty — always guard)
st.empty();  st.size();

// Very often a vector (or string) is the BETTER stack:
vector<int> v;
v.push_back(x); v.pop_back(); v.back();   // same O(1) ops...
// ...plus you can iterate/inspect BELOW the top and keep the result in place.

string s;                      // a string is a perfectly good char stack
s.push_back(c); s.pop_back(); s.back();
```

Gotchas that cost interview minutes: `pop()` returns `void` — read `top()` first; `top()`/`pop()` on an empty stack is undefined behavior — guard with `empty()`; and reach for **`vector`-as-stack** whenever you need to look beneath the top or return the stack itself as the answer (most monotonic-stack and greedy-build problems).

---

## Pattern 1: Bracket Matching & LIFO Validation

**Logic:** Scan left to right. Push every opener; on a closer, the top *must* be its matching opener — pop it, or fail. A perfectly balanced string leaves the stack empty at the end.

**Core insight — why it works:** Nesting is intrinsically LIFO: the bracket that opened most recently is the one that must close first, and that bracket is *always the top of the stack*. So "does this closer match?" is answered by a single O(1) `top()` — no scanning, no counting per type. The stack's height at any point is the current nesting depth, and its contents are exactly the still-open scopes, innermost on top. For a single bracket type you don't even need the stack — a counter suffices — but the moment types can interleave (`([)]` must be rejected), you need the top element's *identity*, which only a stack preserves.

**Template (multi-type validation):**
```cpp
bool isValid(string s) {
    stack<char> st;
    unordered_map<char,char> match = {{')','('}, {']','['}, {'}','{'}};
    for (char c : s) {
        if (match.count(c) == 0) st.push(c);              // an opener
        else if (st.empty() || st.top() != match[c])      // closer with no/ wrong partner
            return false;
        else st.pop();                                    // matched — resolve it
    }
    return st.empty();                                    // nothing left open
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 20. Valid Parentheses | Easy | The canonical instance (template above). The `([)]` case is the whole point: a per-type counter passes single types but fails on interleaving — narrate *why* you need the stack's top identity. |
| 1021. Remove Outermost Parentheses | Easy | Depth counter, not even a stack: emit a char only when depth (before `(`, after `)`) is ≥ 1. Good for showing "single type → counter beats stack." |
| 921. Minimum Add to Make Parentheses Valid | Medium | Track `open` (unmatched `(`) and `needed` (unmatched `)`); answer is their sum. The stack collapses to two integers because there's one bracket type — recognizing that is the skill. |
| 1249. Minimum Remove to Make Valid Parentheses | Medium | Push *indices* of `(`; on an unmatched `)` mark it for deletion, and delete whatever indices remain on the stack at the end. Indices-not-chars is the reusable trick. |
| 678. Valid Parenthesis String | Medium | Wildcards `*`. Two-stack version: one stack of `(` indices, one of `*` indices; cancel `)` against `(` first, then `*`. The greedy low/high range method is the O(1)-space alternative — know both. |
| 32. Longest Valid Parentheses | Hard | Push indices with a base sentinel `-1` on the stack bottom; on `)`, pop and measure `i - st.top()`. The sentinel-as-base is what makes the length arithmetic clean — DP is the twin solution. |

**Pitfalls:**
- Reading `top()` without an `empty()` guard — a leading closer (`")("`) dereferences an empty stack (UB) instead of returning false.
- Forgetting the final `st.empty()` check: `"((("` never fails mid-scan; it fails only because openers are left over.
- Using a counter when types interleave — it accepts `([)]`. Counter is valid *only* for a single bracket type.

---

## Pattern 2: Stack Simulation & Adjacent Collapsing

**Logic:** Process elements one by one; each new element interacts only with the **top** (its most recent surviving neighbor). It may cancel the top, merge with it, or survive on top itself. The final stack is the answer.

**Core insight — why it works:** These problems share a "reduce nearest neighbors until stable" structure, and the crucial fact is that an interaction *only ever involves adjacent survivors* — so the most recent survivor (the top) is the only element the newcomer can affect. After a collapse, the newcomer may now meet a *new* top and collapse again (hence the inner `while`), but every element is pushed once and popped at most once → O(n) total despite the nested loop. The stack automatically maintains the "current reduced state" so you never re-scan; contrast the naive O(n²) of repeatedly rescanning a string for adjacent pairs.

**Template (asteroid collision):**
```cpp
vector<int> asteroidCollision(vector<int>& a) {
    vector<int> st;                                  // survivors, left to right
    for (int x : a) {
        bool alive = true;
        while (alive && x < 0 && !st.empty() && st.back() > 0) {  // → meets ←
            if (st.back() < -x)      st.pop_back();               // top explodes, keep colliding
            else if (st.back() == -x){ st.pop_back(); alive = false; } // both explode
            else                       alive = false;              // incoming explodes
        }
        if (alive) st.push_back(x);
    }
    return st;
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 1047. Remove All Adjacent Duplicates In String | Easy | The purest collapse: if the incoming char equals the top, pop instead of pushing. The stack *is* the running de-duplicated string. |
| 1209. Remove All Adjacent Duplicates in String II | Medium | Store `(char, count)` on the stack; increment the top's count, pop when it reaches k. Pairing value with a counter is the generalization worth internalizing. |
| 844. Backspace String Compare | Easy | `#` pops the top. Build both strings via stacks and compare — or the O(1)-space two-pointer-from-the-right, a good "can you avoid the stack?" follow-up. |
| 735. Asteroid Collision | Medium | The template. The three-way outcome (top dies / both die / incoming dies) plus "keep colliding" is the state machine to get exactly right — sign encodes direction. |
| 682. Baseball Game | Easy | Operations (`+`, `D`, `C`, number) mutate the top of a score stack. A gentle warm-up in "stack as an undoable ledger." |
| 71. Simplify Path | Medium | Split on `/`; push names, `..` pops, `.`/empty are no-ops. The canonical filesystem-normalization stack — join survivors with `/` at the end. |

**Pitfalls:**
- Popping with `if` instead of `while`: one incoming element may annihilate several survivors in a chain (`735` especially).
- Losing the "keep colliding" loop in `735` — after the top explodes, the incoming asteroid still faces the *new* top.
- In `1209`, forgetting to merge into the existing top's count and pushing a fresh entry instead — the counts silently never reach k.

---

## Pattern 3: Expression Parsing & Evaluation

**Logic:** A stack turns nested / precedence-laden input into a computation. Push operands (and pending context at an opening scope); when a scope closes or a lower-precedence operator arrives, pop and combine. The stack holds *partial results waiting to be finished*.

**Core insight — why it works:** Expressions are trees flattened into text, and a stack is the minimal machine that reconstructs the tree's evaluation order without building the tree. Two sub-cases: **postfix (RPN)** is trivial because operands are already in evaluation order — push values, and each operator consumes the top two. **Infix** needs the stack to *defer*: a `(` (or a `*` after a `+`) means "I can't finish the earlier work yet," so you stack the context and resume it when the scope closes. Each value and operator is pushed and popped once → O(n). The recurring trick is deciding *what* to stack: for `k[str]` you stack the multiplier and the string-so-far; for calculators you stack partial sums so that `+/-` become "push a signed term" and `*//` fold into the top immediately.

**Template (decode `3[a2[c]]`-style nested strings):**
```cpp
string decodeString(string s) {
    stack<int> counts; stack<string> prevs;
    string cur; int num = 0;
    for (char c : s) {
        if (isdigit(c)) num = num * 10 + (c - '0');
        else if (c == '[') { counts.push(num); prevs.push(cur); num = 0; cur.clear(); }
        else if (c == ']') {                      // close scope: repeat & splice back
            string built = prevs.top(); prevs.pop();
            int k = counts.top(); counts.pop();
            while (k--) built += cur;
            cur = built;
        } else cur += c;
    }
    return cur;
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 150. Evaluate Reverse Polish Notation | Medium | The easy case: operands already ordered, so no deferral. Each operator pops two, pushes one. Use `long long` and mind operand order for `-` and `/`. |
| 227. Basic Calculator II | Medium | `+ − * /` no parentheses. Keep a stack of signed terms; fold `*//` into the top on sight, defer `+/-` by pushing. Answer is the stack's sum. The "sign of the *previous* operator" bookkeeping is the crux. |
| 224. Basic Calculator | Hard | `+ −` with parentheses. Stack the *running result and sign* at each `(`, restore at `)`. No `*//`, so the parenthesis handling is the whole difficulty. |
| 772. Basic Calculator III | Hard | The union of 224 and 227 — precedence *and* parentheses. Cleanest as a recursive/stack hybrid; a great "compose two techniques" capstone. |
| 394. Decode String | Medium | The template. Two parallel stacks (counts, strings) mirror the two things you must remember across a scope: how many times, and what came before. |
| 856. Score of Parentheses | Medium | Stack the score of each depth; `(` pushes 0, `)` folds `max(2·inner, 1)` into the parent. Elegant depth-as-stack scoring — the O(1)-space "count deepest pairs" trick is a slick follow-up. |
| 726. Number of Atoms | Hard | Parse elements + counts with a stack of multiplier maps; `)` multiplies the popped scope by the trailing number. Parsing-heavy boss fight tying stacks to hashmaps and sorting. |

**Pitfalls:**
- Operand order for non-commutative ops: in RPN, `a` is popped *second* — `a - b` and `a / b`, never `b - a`.
- Off-by-one on multi-digit numbers: accumulate `num = num*10 + digit` and reset it at the right moments.
- In infix calculators, mishandling the *last* term — flush the pending number/operator after the loop, or the final operand is dropped.

---

## Pattern 4: Monotonic Stack — Next / Previous Greater or Smaller

**Logic:** Maintain a stack whose values are strictly ordered (increasing or decreasing). When a new element would break the order, pop the offenders — and the act of popping *answers a question* for each popped element: the newcomer is its "next greater/smaller," or the surviving element beneath is its "previous greater/smaller."

**Core insight — why it works:** This is a *dominance* argument. Say you want each element's **next greater** to the right. Keep a stack of indices whose values are decreasing (still "waiting" for a bigger element). When `x` arrives, every stacked element smaller than `x` has just found its answer — `x` — and, crucially, can be **discarded forever**: it's smaller than `x` *and* farther left, so it can never be the next-greater of anything `x` also shadows. The remaining stack is exactly the set of elements whose answer is still unknown, kept sorted so the comparison is O(1). Each index is pushed once and popped once → **O(n)** despite the inner `while`. The deep pattern: *the stack is the frontier of elements still relevant to the future; dominated elements leave the instant their dominator appears.* Flip the comparison for smaller-instead-of-greater; read the surviving top instead of the popped element for *previous* instead of *next*; iterate right-to-left to mirror direction.

**Template (next greater element to the right, by value):**
```cpp
vector<int> nextGreater(vector<int>& nums) {
    int n = nums.size();
    vector<int> res(n, -1);                 // -1 = none exists
    stack<int> st;                          // indices; values strictly decreasing
    for (int i = 0; i < n; i++) {
        while (!st.empty() && nums[st.top()] < nums[i]) {
            res[st.top()] = nums[i];        // nums[i] is st.top()'s next-greater
            st.pop();                       // answered → discard
        }
        st.push(i);
    }
    return res;                             // whatever stays on the stack has no next-greater
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 496. Next Greater Element I | Easy | Compute next-greater for the full array into a hashmap, then answer the subset queries. The clean introduction to "answer at pop time." |
| 739. Daily Temperatures | Medium | Next-greater but store the *index distance* (`i - st.top()`) instead of the value. Shows the popped element's answer can be any function of the meeting point. |
| 503. Next Greater Element II | Medium | Circular array: iterate `2n` times using `i % n`, pushing only in the first pass' spirit (don't re-answer). The standard "wrap-around ⇒ double the loop" idiom. |
| 901. Online Stock Span | Medium | Previous-greater-or-equal, *streaming*: each `next()` pops spans of smaller prices and accumulates their widths. Same stack, run online — store `(price, span)` pairs. |
| 1475. Final Prices With a Special Discount | Easy | Next smaller-or-equal to the right = the discount. A gentle drill in flipping the comparison from greater to smaller. |
| 456. 132 Pattern | Medium | Scan right-to-left with a stack of candidate "3"s, tracking the largest value ever popped as "2"; a valid "1" < that "2" wins. The famous "maintain the best popped element" twist — hard to see, worth memorizing. |
| 962. Maximum Width Ramp | Medium | Build a decreasing stack of candidate left ends, then scan from the right popping while a valid ramp exists. Two-phase monotonic stack — not the usual single sweep. |
| 1019. Next Greater Node In Linked List | Medium | Next-greater on a list: convert to a vector (or push during traversal). Confirms the pattern is about *sequence order*, not the container. |

**Pitfalls:**
- Storing **values** instead of **indices** when you need distances or positions (`739`) — you then can't recover where the element was.
- `<` vs `<=` in the pop test controls tie-handling: strict keeps equal earlier elements alive (their "next greater" skips ties), non-strict treats equals as answered. Decide deliberately per problem.
- Direction confusion: *next* reads the popped element, *previous* reads the survivor beneath; left-to-right finds next-on-right, right-to-left finds next-on-left. Say which out loud before coding.

---

## Pattern 5: Histogram Spans — Largest Rectangle & Trapping Rain Water

**Logic:** A monotonic stack whose popped element's answer is a **width**. Keep indices with increasing heights; when a shorter bar arrives, each taller bar popped is "capped" — its rectangle can't extend past the newcomer (right limit) or past the survivor beneath (left limit). Compute area at pop time.

**Core insight — why it works:** A bar's maximal rectangle is bounded by the **first strictly shorter bar on each side** — beyond those, the bar's height no longer fits. A monotonic-increasing stack discovers *both* boundaries in a single motion: when bar `mid` is popped by the incoming bar at `i`, `i` is its right boundary (first shorter on the right) and the new top is its left boundary (first shorter on the left), so `width = i - st.top() - 1`. Every bar is pushed and popped once → O(n), replacing the O(n²) "expand from each bar." The dual view — trapping rain water — pops a valley and fills it between the two taller walls (`min(left, right) - height`), the same boundary discovery aimed at the space *between* bars instead of *under* them. A **sentinel** height of 0 appended at the end flushes the stack so no bar is left uncomputed.

**Template (largest rectangle in a histogram):**
```cpp
int largestRectangleArea(vector<int>& h) {
    int n = h.size(), best = 0;
    stack<int> st;                                   // indices; heights increasing
    for (int i = 0; i <= n; i++) {
        int cur = (i == n) ? 0 : h[i];               // sentinel 0 flushes everything
        while (!st.empty() && h[st.top()] >= cur) {
            int height = h[st.top()]; st.pop();
            int left = st.empty() ? -1 : st.top();   // first shorter bar on the left
            best = max(best, height * (i - left - 1));
        }
        st.push(i);
    }
    return best;
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 84. Largest Rectangle in Histogram | Hard | The template and the parent of the whole pattern. The width formula `i - left - 1` at pop time is the one line to be able to derive under pressure. |
| 85. Maximal Rectangle | Hard | Build a histogram per row (heights of consecutive 1s) and run 84 on each row. The reduction "2D → n histograms" is the entire insight; the stack is reused verbatim. |
| 42. Trapping Rain Water | Hard | Pop a valley, add `(min(h[left], h[i]) - h[popped]) * width`. The two-pointer O(1)-space solution is the celebrated alternative — be ready to present both and compare. |
| 1793. Maximum Score of a Good Subarray | Hard | Largest-rectangle-flavored: expand a window that must contain index `k`, minimized height × width. Monotonic-stack *or* two-pointer from `k` — a good "which lens?" problem. |
| 1130. Minimum Cost Tree From Leaf Values | Medium | Greedy with a decreasing stack: remove the smallest leaf by pairing it with its smaller neighbor, accumulating `smallest × min(neighbors)`. Histogram cousin where popping computes a merge cost. |

**Pitfalls:**
- Skipping the end sentinel — bars still on the stack after the loop never get their area computed; either append a 0 or drain with a post-loop while.
- `>=` vs `>` in the pop test: for area, popping on equal heights (`>=`) is fine because the equal bar to the right will recompute the full width — but be consistent or you double/undercount.
- Width arithmetic with an empty stack after popping: the left boundary is `-1` (nothing shorter to the left), giving width `i`. Forgetting this drops the tallest full-width rectangles.

---

## Pattern 6: Contribution Technique — Summing over All Subarrays

**Logic:** To aggregate a quantity over *all* O(n²) subarrays (sum of minimums, sum of ranges, count of distinct), don't iterate subarrays. Iterate **elements** and ask: "for how many subarrays is *this* element the min / the max / the unique owner?" A monotonic stack gives that count via nearest-smaller/greater boundaries; multiply by the element's value and sum.

**Core insight — why it works:** Reframe "sum over subarrays" as "sum over elements weighted by responsibility." Element `arr[i]` is the minimum of exactly the subarrays whose left end lies in `(PLE, i]` and right end in `[i, NLE)`, where `PLE` = previous-less-element index and `NLE` = next-less-element index. That's `(i - PLE) × (NLE - i)` subarrays, each contributing `arr[i]` to the total of minimums — so the answer is `Σ arr[i] · (i - PLE) · (NLE - i)`, each factor from one monotonic-stack sweep. The subtle, decisive detail is **tie-breaking**: to count each subarray's minimum exactly once when values repeat, make one side strict and the other non-strict (e.g., previous *strictly* less, next less-*or-equal*). Get that asymmetry wrong and equal minimums are double-counted or dropped.

**Template (sum of subarray minimums, single pass):**
```cpp
int sumSubarrayMins(vector<int>& arr) {
    const int MOD = 1e9 + 7;
    int n = arr.size();
    long long ans = 0;
    stack<int> st;                                   // indices; values increasing
    for (int i = 0; i <= n; i++) {
        int cur = (i == n) ? INT_MIN : arr[i];       // sentinel flushes the stack
        while (!st.empty() && arr[st.top()] >= cur) { // pop >= : "next less-or-equal" is i
            int mid = st.top(); st.pop();
            int left = st.empty() ? -1 : st.top();    // "previous strictly less"
            ans = (ans + (long long)arr[mid] * (mid - left) * (i - mid)) % MOD;
        }
        st.push(i);
    }
    return ans;
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 907. Sum of Subarray Minimums | Medium | The template. The whole exam is the strict/non-strict split for duplicates — derive it on paper with `[2,2]` (answer 6) so you can defend the `>=`. |
| 2104. Sum of Subarray Ranges | Medium | `Σ(max − min) = Σmax − Σmin`: run the min pass and a mirrored max pass, subtract. The clean "decompose into two contributions" move; O(n) stack beats the easy O(n²). |
| 1856. Maximum Subarray Min-Product | Medium | For each element as the minimum, its span (from PLE/NLE) times a prefix-sum gives min×sum; take the max. Contribution boundaries feeding a prefix-sum — a lovely composite. |
| 828. Count Unique Characters of All Substrings | Hard | Per character, count substrings where it appears *exactly once* using its previous/next same-character positions — the same "responsibility interval" idea without a literal stack. Cements the reframing. |

**Pitfalls:**
- Wrong tie convention → double counting. The reliable recipe: **strict on one side, non-strict on the other** (here: pop on `>=`, so left boundary is previous-strictly-less, right boundary is next-less-or-equal).
- Overflow: products of two O(n) distances times a value blow past `int` — accumulate in `long long` and apply `MOD` (when asked) after each addition.
- Reusing the exact same comparator for the max pass in `2104` — you must flip *both* the ordering and keep the strict/non-strict asymmetry, or maxes get miscounted.

---

## Pattern 7: Greedy Monotonic Stack — Building the Optimal Subsequence

**Logic:** Build the result on a stack while scanning left to right. Before pushing the current element, pop previously-kept elements that are "worse" than it — as long as you're still allowed to remove them (a budget k, or a "can reappear later" guarantee). The stack ends holding the optimal subsequence.

**Core insight — why it works:** For lexicographic objectives ("smallest/largest number after removing k digits", "most competitive subsequence"), a **more significant position dominates all less significant ones combined** — so a larger digit sitting to the *left* of a smaller digit is always worth removing when a removal is available: swapping in the smaller digit at the higher place strictly improves the result, regardless of what follows. That's an exchange argument, and the monotonic stack executes it greedily: pop while the top is worse (bigger, for "smallest") and budget remains. The guardrails are what make it correct — a removal *count* (`k`), a *"each element still appears later"* check (for "keep every distinct letter"), or a *"enough elements remain to reach target length"* check (for fixed-length subsequences). Each element is pushed and popped at most once → O(n).

**Template (remove k digits to make the smallest number):**
```cpp
string removeKdigits(string num, int k) {
    string st;                                    // the result, used as a stack
    for (char c : num) {
        while (!st.empty() && k > 0 && st.back() > c) {  // a bigger digit sits left of a smaller one
            st.pop_back(); k--;                          // remove it while budget allows
        }
        st.push_back(c);
    }
    while (k > 0) { st.pop_back(); k--; }         // budget left → drop from the tail (largest place)
    int i = 0; while (i < (int)st.size() && st[i] == '0') i++;   // strip leading zeros
    string res = st.substr(i);
    return res.empty() ? "0" : res;
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 402. Remove K Digits | Medium | The template. Two subtleties: leftover budget removes from the tail, and leading zeros are stripped at the end. Both are where wrong answers hide. |
| 1673. Find the Most Competitive Subsequence | Medium | Fixed output length k: pop while the top is bigger *and* enough elements remain (`st.size()-1 + remaining ≥ k`). The "enough left to finish" guard is the reusable variant. |
| 316. Remove Duplicate Letters | Medium | Smallest subsequence containing every distinct letter once: pop a bigger top only if it *appears again later* (last-index check), and skip letters already on the stack. Two guards, both essential. |
| 1081. Smallest Subsequence of Distinct Characters | Medium | Identical to 316 — same code. Worth doing back-to-back to confirm the pattern generalizes and to bank an easy "I've solved its twin." |
| 321. Create Maximum Number | Hard | Compose two ideas: pick the best length-`i` subsequence from one array and `k−i` from the other (this pattern, maximizing), then *merge* them greedily; try all splits. The boss fight — monotonic build + merge. |

**Pitfalls:**
- Missing a guard: `316` needs *both* "top recurs later" and "current not already used"; dropping either produces invalid or non-minimal output.
- Forgetting to spend leftover budget in `402` (`"112"`, k=1 → must drop the trailing digit). The tail removal after the loop is not optional.
- The "enough remain" arithmetic in `1673`/`321` is off-by-one bait — write it as `stackSize + elementsLeft > targetLen` and test on a tiny case.

---

## Pattern 8: Stack Design & Amortization — Min Stack, Queue from Stacks

**Logic:** Augment or combine stacks so an extra query (current minimum, FIFO order) stays O(1). Either store the auxiliary answer *alongside* each element, or use a second stack that lazily reverses order.

**Core insight — why it works:** Two distinct tricks. **(1) Carry the answer with each entry.** Since a stack only ever removes the top, the min *of everything below any element never changes while that element lives* — so store `(value, min-so-far)` at push time and the minimum is always the top's cached field, O(1), no recomputation on pop. This "each frame remembers a prefix aggregate" idea works for any aggregate that's cheap to extend (min, max, gcd, running sum). **(2) Two stacks reverse into a queue.** Pushes pile onto an `in` stack; when you need the front, if `out` is empty you pour `in` into `out`, reversing LIFO into FIFO. Each element moves across at most once, so despite an occasional O(n) transfer, it's **amortized O(1)** — the canonical amortization talking point.

**Template (min stack — O(1) getMin):**
```cpp
class MinStack {
    stack<pair<int,int>> st;                      // (value, min of everything at/below)
public:
    void push(int x) {
        int mn = st.empty() ? x : min(x, st.top().second);
        st.push({x, mn});
    }
    void pop()        { st.pop(); }
    int  top()        { return st.top().first; }
    int  getMin()     { return st.top().second; } // O(1): the cached prefix-min
};
```

**Template (queue from two stacks — amortized O(1)):**
```cpp
class MyQueue {
    stack<int> in, out;
    void shift() { if (out.empty()) while (!in.empty()) { out.push(in.top()); in.pop(); } }
public:
    void push(int x) { in.push(x); }
    int  pop()  { shift(); int v = out.top(); out.pop(); return v; }
    int  peek() { shift(); return out.top(); }
    bool empty(){ return in.empty() && out.empty(); }
};
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 155. Min Stack | Medium | The template. The `(value, min)` pairing beats a parallel "min stack" for clarity; the space-saving "store deltas from min" version is a neat follow-up if pressed on memory. |
| 232. Implement Queue using Stacks | Easy | Two-stack amortization (template). The interview payload is the **amortized O(1)** argument: each element is moved from `in` to `out` at most once in its lifetime. |
| 225. Implement Stack using Queues | Easy | The mirror: one queue, rotate after each push so the newest sits at the front → O(n) push, O(1) pop. Discuss which op you choose to make expensive. |
| 716. Max Stack | Hard | Harder than Min Stack: `popMax` must remove a possibly-buried element, so a plain cached max fails — pair a stack with an ordered structure (or two stacks + tombstones). A genuine step up. |
| 1381. Design a Stack With Increment Operation | Medium | Lazy increments: store pending add-to-everything-below deltas so `increment` is O(1), applied at pop. Same "defer work to where it's cheap" spirit as amortization. |
| 946. Validate Stack Sequences | Medium | Simulate: push pushes, and greedily pop whenever the top equals the next expected pop. Valid iff the stack empties. Stack-as-verifier rather than stack-as-store. |

**Pitfalls:**
- Min Stack: recomputing the min on `pop` by scanning — defeats the point. The min must be cached per frame so pop is O(1).
- Two-stack queue: refilling `out` while it's *non-empty* scrambles order — only pour when `out` is empty.
- Claiming O(1) worst-case for the two-stack queue — it's **amortized** O(1); a single `pop` can be O(n). Precision here is the whole point of the problem.

---

## When a Stack FAILS — Know the Boundary

| Situation | Why the stack breaks | Use instead |
|---|---|---|
| Sliding-window max/min (both ends move) | A stack can only remove from the top; window elements expire from the *bottom* | Monotonic **deque**, O(n) |
| Need k-th element / ranking / order statistics | Only the top is visible; no lateral order | Heap / quickselect / order-statistic tree |
| Search, predecessor/successor, range queries | LIFO exposes one element; no lookup | `multiset` / BST / sorted array + binary search |
| Must emit in arrival (FIFO) order | A single stack reverses order | Queue, or the two-stack trick (Pattern 8) |
| Aggregate needs *removal from both ends* incrementally | Stack augmentation caches only prefix-below | Deque / balanced BST / segment tree |
| Subarray-min counting with careless ties | Symmetric comparisons double-count equal minima | Monotonic stack with **strict/non-strict** asymmetry (Pattern 6) |

The litmus test in one sentence: **a stack is the right tool when the only element you ever need next is the most recent unresolved one** — in nesting, in time, or in value order. Need visibility below the top (search, ranges, both-ended eviction)? Reach for a deque, heap, or ordered structure.

---

## Cheat Sheet: Problem Phrase → Pattern

| Phrase in the problem | Pattern |
|---|---|
| "valid / balanced parentheses", "matching brackets or tags" | Bracket-matching stack |
| "remove adjacent duplicates", "collision", "backspace", "simplify path" | Simulation & collapsing stack |
| "evaluate expression / calculator", "decode `k[...]`", "RPN" | Parsing & evaluation stack |
| "next/previous greater or smaller", "daily temperatures", "stock span" | Monotonic stack |
| "largest rectangle", "trapping rain water", "maximal rectangle" | Histogram spans (widths at pop time) |
| "sum/min/max/range over **all** subarrays" | Contribution technique + monotonic stack |
| "smallest/largest number after removing k", "most competitive subsequence" | Greedy monotonic stack |
| "min stack", "queue from stacks", "max stack" | Stack design + amortization |
| "sliding window maximum" | **Not** a stack → monotonic deque |

---

## Complexity Summary

- push / pop / top: **O(1)**.
- Bracket matching, simulation, parsing: **O(n)** time, **O(n)** space.
- Monotonic stack (all variants — next/prev, histogram, contribution, greedy build): **O(n)** — every index is pushed once and popped once (amortization covers the inner `while`).
- Two-stack queue: **amortized O(1)** per operation (each element crosses `in`→`out` at most once).
- Min Stack: **O(1)** for every operation including `getMin`.
- Space: **O(n)** worst case — a strictly monotonic input (e.g. sorted) keeps the whole array on the stack.

---

## Interview Tips

1. **Name what the stack holds and its invariant** before the loop: "indices with strictly decreasing values — the elements still waiting for a next-greater." Once stated, the pop condition writes itself and the interviewer sees structure, not guessing.
2. **Defend O(n) with amortization.** "The inner `while` looks nested, but each element is pushed once and popped once over the entire run — so total pops ≤ n." This exact sentence is frequently asked for; the two-stack queue needs the same argument for its amortized O(1).
3. **State the direction and strictness for monotonic stacks.** *Next* reads the popped element, *previous* reads the survivor beneath; left-to-right finds next-on-the-right. And `<` vs `<=` decides tie-handling — in subarray-min counting the strict/non-strict split is the correctness crux, so say it aloud.
4. **Use sentinels to kill edge cases.** Appending a `0` height (histogram) or a `-∞` (contribution) flushes the stack so no element is left uncomputed; a `-1` base index (longest valid parentheses) makes the length arithmetic uniform.
5. **Prefer `vector`/`string` as a stack** whenever you must inspect below the top or return the stack itself (every greedy-build and histogram problem). `std::stack` is fine when you only ever touch the top.
6. **Know the escape hatches:** sliding-window extremes → monotonic *deque*; k-th/ranking → heap; search/ranges → ordered structure. Choosing *against* the stack correctly scores as highly as using it well — and "this is where a plain stack stops and a deque begins" (e.g. 84 → 239) is a senior signal.

---

## Suggested Practice Order

**Week 1 — stack fundamentals:** 20 → 1021 → 682 → 1047 → 844 → 735 → 71 → 1249
**Week 2 — parsing & evaluation:** 150 → 227 → 224 → 394 → 856 → 726
**Week 3 — monotonic stack core:** 496 → 739 → 503 → 901 → 456 → 84 → 42 → 85
**Week 4 — advanced & design:** 907 → 2104 → 1856 → 402 → 316 → 1673 → 321 → 155 → 232 → 716

Good luck with the interviews!

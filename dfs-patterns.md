# Depth-First Search — The Complete Interview Pattern Guide (C++)

DFS commits: it follows one path as deep as it can go, and only when stuck does it back up to the last choice point and try the next option. That commit-and-backtrack rhythm makes it the engine behind an enormous share of interview problems — every tree recursion, connectivity check, cycle detection, all of backtracking, and top-down DP are DFS wearing different costumes. This guide covers them all: the logic, the **core insight** (why it's correct), C++ templates, strong problem sets with descriptive notes, and the pitfalls.

---

## What DFS Actually Is

> **DFS is a traversal that fully explores one branch before considering its siblings — equivalently, a stack-driven search where the most recently discovered node is expanded next.**

The deeper way to see it: **DFS is recursion itself.** Any recursive function exploring a structure — trees, graphs, decision sequences, subproblem spaces — *is* a DFS, whether or not you call it that. Three consequences define everything below:

- **Memory = one path.** At any moment, the call stack holds exactly the current root-to-here path: O(depth) memory, versus BFS's O(width) frontier. On deep-narrow structures DFS is light; on a 10⁵-node degenerate tree it overflows the call stack — know both edges of that blade.
- **Two moments per node, and they are not interchangeable.** Each node is touched on the way **down** (pre-order: before its subtree) and on the way **up** (post-order: after its subtree). Pre-order is where *inherited* state flows down (the path so far, choices made). Post-order is where *synthesized* answers flow up (subtree size, height, best path below). **Choosing which moment your logic belongs to is 80% of designing a DFS** — most tree bugs are pre/post confusion.
- **Backtracking is built in.** When a call returns, its local state vanishes — the algorithm has automatically "undone" the descent. Explicit backtracking (Patterns 5–6) just extends this to *shared* state: whatever you mutate on the way in, you restore on the way out, keeping the shared state in sync with the stack.

**DFS vs BFS — the decision rule** (mirror of the BFS guide's table): shortest path / fewest moves / per-level questions → BFS; existence, exhaustive enumeration, connectivity, anything needing the **path itself** or **subtree summaries** → DFS. The structural reason: BFS's frontier knows distances but no paths; DFS's stack *is* a path but knows nothing about minimality.

**The three colors — DFS's state vocabulary for graphs.** White = untouched; **Gray = currently on the stack** (discovery started, not finished); Black = fully explored. The gray set is DFS's superpower: it's the live path, and meeting a gray node means you've found a **back edge** — a cycle through your own ancestry. BFS has no analogue; this is why directed-cycle detection is DFS territory.

---

## How to Recognize a DFS Problem

**1. Structural signals.** Any binary-tree problem (90%+ are DFS); "clone / serialize / validate" a structure; nested structures (parentheses, folders, JSON-like).

**2. Phrasing signals.**
- "**All** paths / **all** combinations / **all** permutations / generate **every** valid …" — exhaustive enumeration = backtracking (Patterns 5–6).
- "Does a path **exist**", "is it connected", "can A reach B" — plain graph DFS (Pattern 4).
- "Detect a **cycle**" in a directed graph — gray-set DFS (Pattern 4).
- "Diameter / max path sum / count subtrees with property" — post-order tree DP (Pattern 2).
- "Root-to-leaf" anything — pre-order path DFS (Pattern 3).
- "Number of ways / min cost" over choices with **overlapping subproblems** — DFS + memo (Pattern 7).

**3. Constraint arithmetic.** n ≤ ~20 with "all subsets/orderings" → 2ⁿ or n! enumeration is *intended*: backtracking. n ≤ 10⁵ with a tree → linear DFS. Exponential-looking recursion but the **distinct argument count is small** (n·k states) → memoize it.

**4. The negative signal.** "Shortest / minimum number of moves" with unit steps — that's BFS; DFS finds *a* path, not the best one, and depth-first order gives no minimality guarantee.

---

## Pattern 1: Tree Recursion Fundamentals

**Logic:** Define the function's meaning *on a single node*, trust it on the children, combine. `maxDepth(node) = 1 + max(maxDepth(l), maxDepth(r))`, base case `nullptr → 0`. Every tree problem is: pick the right return value, the right base case, and the right combiner.

**Core insight — why it works:** Trees are recursively self-similar — every subtree is itself a tree — so structural induction applies directly: if the function is correct on all smaller trees (children), and the combine step is correct, it's correct everywhere. The practical discipline this buys you is the **recursive leap of faith**: never trace into the child calls; *assume* they return what the function's contract promises, and verify only the single-node logic. Tracing recursion three levels deep is how interviews are lost; stating the contract ("this returns the height of the subtree rooted here") is how they're won.

**Template (the shape of all tree recursion):**
```cpp
int dfs(TreeNode* node) {
    if (!node) return BASE;                 // contract for the empty tree
    int left  = dfs(node->left);            // trust: left subtree's answer
    int right = dfs(node->right);           // trust: right subtree's answer
    return combine(left, right, node->val); // single-node logic — the only part to verify
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 104. Maximum Depth of Binary Tree | Easy | The hello-world: `1 + max(l, r)`. Write it as the template consciously — the shape recurs in fifty harder problems. |
| 100. Same Tree | Easy | Recursion on *two* trees in lockstep: both null → true; one null or values differ → false; else recurse pairwise. The two-pointer of tree recursion. |
| 101. Symmetric Tree | Easy | 100 with mirrored pairing: compare (a->left, b->right) and (a->right, b->left). The insight is that symmetry is sameness under reflection. |
| 226. Invert Binary Tree | Easy | Swap children, recurse. Famous for being trivially recursive and famously failed under pressure — the leap of faith drill. |
| 111. Minimum Depth | Easy | The trap: `min(l, r)` is wrong when one child is null (a one-child node is *not* a leaf). The base-case-precision lesson. |
| 110. Balanced Binary Tree | Easy | Return height, but signal imbalance with −1 sentinel so the check is O(n) single-pass instead of O(n²) height-per-node. First taste of "return value carries two meanings." |
| 617. Merge Two Binary Trees | Easy | Lockstep recursion that *builds*: structural recursion producing a structure. |
| 572. Subtree of Another Tree | Easy | Recursion nested in recursion: isSame (100) tried at every node. O(n·m); mention the serialize+string-match or hash improvement as the follow-up. |
| 98. Validate Binary Search Tree | Medium | Pass down (lo, hi) bounds — checking only parent-child locally is *the* classic wrong answer. Alternatively: in-order traversal must be strictly increasing. Both arguments matter. |

**Pitfalls:**
- Base case sloppiness: nullptr vs leaf are different bases (111 punishes conflating them). Decide which your contract uses and write it first.
- Comparing only local parent-child relations when the property is global (98) — the bound must travel down.
- `int` vs `long long` for accumulated sums/bounds (98 with INT_MIN/INT_MAX node values needs `long long` bounds or pointer sentinels).

---

## Pattern 2: Post-Order Tree DP — Synthesize Up

**Logic:** The parent's answer depends on its children's answers, so compute children first (post-order) and return a compact **summary** upward. The signature move of harder problems: the value you *return* (constrained to be combinable by the parent) differs from the value you *record* (the unconstrained global best, updated in a side variable).

**Core insight — why it works:** Post-order *is* dynamic programming on the tree: children are strict subproblems, the post-order guarantees they're solved before the parent needs them, and each node is solved exactly once → O(n). The return/record split exists because of a real asymmetry: a parent can only extend a path that goes *straight down* through one child (a bent path can't be extended upward), so the return value must be the best *extendable* answer, while the best *overall* answer — which may bend at this very node — is recorded globally at the bend point. Diameter, max path sum, longest univalue path are all this one idea: **return the best straight line; record the best bend.**

**Template (diameter — the return/record split):**
```cpp
int best = 0;
int depth(TreeNode* node) {                  // RETURNS: longest downward path (extendable)
    if (!node) return 0;
    int l = depth(node->left);
    int r = depth(node->right);
    best = max(best, l + r);                 // RECORDS: best path bending at this node
    return 1 + max(l, r);                    // parent can only extend one arm
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 543. Diameter of Binary Tree | Easy | The cleanest return/record specimen (template above). If this split isn't second nature, every problem below will fight you. |
| 124. Binary Tree Maximum Path Sum | Hard | Diameter with values and negativity: return `node->val + max(0, max(l, r))` — clamping negative arms to 0 is "decline the child's contribution." Record `node->val + max(0,l) + max(0,r)`. The clamp placement is the entire problem. |
| 687. Longest Univalue Path | Medium | Diameter where arms only count if child value matches. Conditional arm extension — practice for predicates inside the combine. |
| 337. House Robber III | Medium | Return a **pair**: (best if this node robbed, best if not). Parent combines: robbed = val + skip_l + skip_r; skipped = max of each child's pair. Multi-state post-order DP in miniature. |
| 968. Binary Tree Cameras | Hard | Return one of three states (covered-no-camera / has-camera / needs-coverage); greedy: place cameras at parents of needy nodes. State-machine post-order — the hardest common variant. |
| 1245. Tree Diameter (general tree) | Medium | Diameter on an n-ary adjacency list: track the top-two child depths instead of left/right. Same idea unshackled from binary structure. |
| 508. Most Frequent Subtree Sum | Medium | Return subtree sum, record frequencies in a map. The gentle on-ramp to return/record. |
| 1110. Delete Nodes and Return Forest | Medium | Post-order deletion: decide children first so the parent can null its pointers safely, collecting orphaned subtrees as roots. Structural surgery needs post-order for the same reason `rm -r` does. |
| 236. Lowest Common Ancestor | Medium | Return "found p, q, or their LCA below me": null/null → null, x/null → x, x/y → this node *is* the LCA. The most elegant post-order argument on LeetCode — be able to state *why* the both-sides-non-null case is conclusive. |

**Pitfalls:**
- Returning the recorded (bent) value instead of the extendable (straight) one — the parent then double-counts a bend. If your diameter is too big, this is why.
- Forgetting to clamp negative contributions in 124-style sums — `max(0, child)` is a *decision* (take the arm or not), not a detail.
- Multi-state returns (337, 968): define the states' meanings in comments before coding; off-by-one-state is unrecoverable mid-interview otherwise.

---

## Pattern 3: Root-to-Leaf Path DFS — Inherit Down

**Logic:** The mirror of Pattern 2: state accumulates on the way **down** (pre-order) — running sum, the path so far, max seen on the path — and conclusions are drawn at **leaves** (or at every node). Pass inherited state as parameters; parameters auto-rollback on return.

**Core insight — why it works:** A root-to-leaf path corresponds exactly to one chain of recursive calls, so the call stack *is* the path — pass-by-value parameters give each branch its own copy of the inherited state for free, which is why simple path problems need no explicit undo. The pre/post choice is forced by information direction: "sum from root to here" is inherited (only ancestors determine it) → pre-order; "height below here" is synthesized (only descendants determine it) → post-order. Ask "who determines this value, ancestors or descendants?" and the traversal order picks itself. When the inherited state is heavy (a shared `path` vector), you switch to mutate-and-undo — which is Pattern 5's backtracking, revealing these as one family.

**Template (path sum to leaves):**
```cpp
bool dfs(TreeNode* node, int remaining) {          // inherited state as parameter
    if (!node) return false;
    remaining -= node->val;
    if (!node->left && !node->right)               // decide at the LEAF
        return remaining == 0;
    return dfs(node->left, remaining) || dfs(node->right, remaining);
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 112. Path Sum | Easy | The template. Leaf = both children null — testing at nullptr instead miscounts one-child nodes (111's trap again). |
| 113. Path Sum II | Medium | Collect the actual paths: shared `vector<int> path` with push/recurse/pop — the explicit-undo bridge to backtracking. Copy the vector only on success. |
| 129. Sum Root to Leaf Numbers | Easy | Inherited value transforms: `num = num*10 + val`. At leaves, add to the total. Inheritance with arithmetic. |
| 257. Binary Tree Paths | Easy | String-building variant; pass-by-value strings make undo automatic (at an allocation cost worth mentioning). |
| 1448. Count Good Nodes | Medium | Inherit the max on the path so far; a node is good iff `val >= pathMax`. Decision at *every* node, not just leaves — the per-node flavor. |
| 437. Path Sum III | Medium | Paths start anywhere going down: inherit a **prefix-sum frequency map** — current count += freq[runningSum − target] — and undo the map entry post-recursion. The prefix-sum hashmap technique transplanted onto a tree; brilliant crossover problem. |
| 988. Smallest String Starting From Leaf | Medium | Inherit the string, compare at leaves. Note the reversal (leaf→root reading) and why a greedy per-level choice fails — full enumeration is required. |
| 1457. Pseudo-Palindromic Paths | Medium | Inherit a parity **bitmask** (`mask ^= 1 << digit`); at a leaf, palindromic iff `mask & (mask−1) == 0` (≤1 odd count). Path state compressed to one int — elegant and fast. |

**Pitfalls:**
- The shared-mutable trap: a `path` vector passed by reference **must** be popped after recursion — every push needs a structurally guaranteed pop (same scope, no early return skipping it).
- 437's map undo: decrement `freq[runningSum]` after recursing, and seed `freq[0] = 1` — both omissions produce close-but-wrong counts.
- Leaf definition, again: `!left && !right`. Null-based "leaf" tests are the most recycled bug in this whole guide.

---

## Pattern 4: Graph DFS — Connectivity, Cloning, and the Gray Set

**Logic:** DFS on graphs = tree DFS + a `visited` set (graphs have cycles; without the set you recurse forever). Connectivity: one DFS visits exactly one component; an outer scan counts them. Directed-cycle detection: upgrade visited to three **colors** — meeting a **gray** (in-progress) node means a back edge to your own ancestry: a cycle.

**Core insight — why it works:** For connectivity, the argument is the equivalence-class one (a DFS from any node visits exactly its component — no more, no less). For cycles, the gray set is the magic: gray nodes are precisely the current recursion stack — your live ancestry. An edge into a *black* node points at a finished, cycle-free exploration (fine); an edge into a *gray* node points back into your own open path, closing a directed loop — and **every directed cycle must reveal itself this way** (consider the first cycle node the DFS enters; the cycle's edge back into it arrives while it's still gray). This is why two booleans (`visited`, `inStack`) suffice and why plain BFS can't detect directed cycles: BFS has no notion of "the current path." Note the asymmetry: in *undirected* graphs a cycle is just reaching a visited node that isn't your immediate parent — colors unnecessary.

**Template (directed cycle detection, three colors):**
```cpp
vector<int> color;                      // 0 white, 1 gray, 2 black
bool hasCycle(int u, vector<vector<int>>& adj) {
    color[u] = 1;                       // gray: on the current path
    for (int v : adj[u]) {
        if (color[v] == 1) return true;             // back edge → cycle
        if (color[v] == 0 && hasCycle(v, adj)) return true;
    }
    color[u] = 2;                       // black: finished, provably cycle-free below
    return false;
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 547. Number of Provinces | Medium | Component counting on an adjacency *matrix* — the canonical scan-plus-DFS. (Union-Find is the equal-credit alternative; offer both.) |
| 200. Number of Islands (DFS flavor) | Medium | Grid components. DFS is shorter than BFS here; the caveat — recursion depth on snake-shaped islands — is the discussion point that distinguishes candidates. |
| 841. Keys and Rooms | Medium | Pure reachability from node 0: can DFS touch everything? The minimal graph-DFS problem. |
| 133. Clone Graph | Medium | DFS with a `original → copy` map doing double duty as the visited set — clone-on-first-visit, return the mapped copy on revisits. Cycles are exactly why the map must be checked *before* recursing. |
| 207. Course Schedule (DFS flavor) | Medium | Cycle detection verbatim (template above). Pairs with the Kahn's/BFS version — be able to write both and say when you'd pick which (DFS gives the cycle path; Kahn gives the order incrementally). |
| 802. Find Eventual Safe States | Medium | "Safe" = not on/leading-to a cycle = comes out **black** with no gray encounters. The three colors *are* the answer encoding — a beautiful re-read of the template. |
| 399. Evaluate Division | Medium | Weighted reachability: DFS from numerator to denominator multiplying edge ratios. Graph modeling (variables = nodes, equations = weighted edges) is the actual problem. |
| 417. Pacific Atlantic (DFS flavor) | Medium | Two reverse-flow DFS sweeps from the borders, intersect. Same inversion as the BFS guide — the traversal is interchangeable, the inversion isn't. |
| 332. Reconstruct Itinerary | Hard | Not plain DFS: **Hierholzer's** Eulerian-path algorithm — post-order push of airports after exhausting each one's (sorted) edges, then reverse. Knowing this is "use every *edge* once, not every node" is the recognition test. |

**Pitfalls:**
- Two-boolean shortcut confusion: `visited` alone detects cycles in *undirected* graphs but gives false positives in directed ones (a diamond A→B→D, A→C→D is not a cycle). Directed needs the gray/in-stack distinction — this exact example is a common interview probe.
- Forgetting to un-gray (mark black) on return — every node looks like a back edge forever after.
- Clone Graph: inserting into the map *after* recursing into neighbors → infinite loop on any cycle. Map-insert first, recurse second.

---

## Pattern 5: Backtracking — Enumerate by Choose / Explore / Unchoose

**Logic:** Generating all subsets / permutations / combinations / partitions = DFS over the **decision tree**: each level decides one element or position; each branch is one choice. Maintain one shared partial solution: **choose** (mutate it), **explore** (recurse), **unchoose** (restore it exactly). Leaves — or every node, for subsets — emit answers.

**Core insight — why it works:** The decision tree's leaves correspond **bijectively** to the solution space — every solution is reached by exactly one choice sequence, so enumeration is complete and duplicate-free *by construction* (no generate-then-dedupe). The shared-buffer + undo discipline is what makes it cheap: one O(n) path buffer instead of copying partial solutions at every node, with the undo keeping the buffer perfectly synced to the recursion stack. The two structural levers to internalize: a **start index** makes order irrelevant (combinations/subsets — never look backward, killing permutation duplicates of the same set), and **sort + skip-equal-siblings** (`i > start && a[i] == a[i-1]`) kills duplicates from repeated *values* by canonicalizing which copy goes first. Branch count is exponential because the *answer* is exponential — backtracking is optimal for its job; pruning just trims branches provably empty of answers.

**Template (subsets — the master shape):**
```cpp
vector<vector<int>> res;
vector<int> path;
void dfs(int start, vector<int>& nums) {
    res.push_back(path);                       // every node IS a subset
    for (int i = start; i < (int)nums.size(); i++) {
        // duplicates version: if (i > start && nums[i] == nums[i-1]) continue;
        path.push_back(nums[i]);               // choose
        dfs(i + 1, nums);                      // explore (i+1: never look back)
        path.pop_back();                       // unchoose — restore exactly
    }
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 78. Subsets | Medium | The master template above. Also know the include/exclude binary-branch formulation — interviewers ask for the "other way." |
| 90. Subsets II | Medium | Sort + the skip-equal-siblings line. Understand *why* it works: among equal values, only "take the earliest available copy" branches survive — a canonical-form argument, not a hack. |
| 46. Permutations | Medium | Order matters → no start index; a `used[]` array (or in-place swapping) marks taken elements. Compare with 78 to see exactly which lever changed. |
| 47. Permutations II | Hard | Sort + `used` + skip rule `a[i]==a[i-1] && !used[i-1]` — forcing equal values to be consumed left-to-right. The subtlest dedupe rule; derive it, don't memorize it. |
| 39. Combination Sum | Medium | Reuse allowed → recurse with `i`, not `i+1` — one character encodes "may pick again." Prune: break when the candidate exceeds the remaining target (requires sorting). |
| 40. Combination Sum II | Medium | No reuse + duplicate values: `i+1` recursion plus the skip-siblings line. 39 and 40 differ by exactly two tokens — diff them consciously. |
| 77. Combinations | Medium | Subsets constrained to size k, with the strong prune `n − i + 1 ≥ k − path.size()` (not enough elements left → abandon). Pruning arithmetic practice. |
| 17. Letter Combinations of a Phone Number | Medium | Cartesian-product backtracking: level = digit, branches = its letters. The gentlest full-pattern problem — good first write. |
| 22. Generate Parentheses | Medium | Constrained generation: branch on '(' if open < n, on ')' if close < open. The constraint *is* the pruning — invalid prefixes are never extended, so everything generated is valid. |
| 131. Palindrome Partitioning | Medium | Level = "where does the next cut go"; branch only on palindromic prefixes. Partitioning problems all share this cut-position decision tree. |

**Pitfalls:**
- Pushing `path` into results without copying… is actually fine in C++ (`push_back(path)` copies) — the *real* C++ bug is keeping references/iterators into `path` across mutations. In any language: ensure results capture a snapshot, not the live buffer.
- Asymmetric choose/unchoose: an early `return`/`continue` between push and pop corrupts every subsequent branch. Structure so undo is unskippable.
- Dedupe at the wrong level: skip equal **siblings** (same tree depth, `i > start`), never equal parent-child — overskipping silently drops valid answers like {1,1,2}.

---

## Pattern 6: Constraint Search — Backtracking on Boards & Grids

**Logic:** Backtracking where choices live on a 2-D board and constraints couple distant cells: word paths, queen placements, Sudoku digits. Same choose/explore/unchoose engine, plus two upgrades: **in-place marking** (mutate the board as the visited/choice record, restore on backtrack) and **constraint-aware pruning** (test legality *before* descending; track constraint sets incrementally so the test is O(1)).

**Core insight — why it works:** Correctness is Pattern 5's bijection again — completeness comes from trying every legal choice at every step. What's new is the *economics*: raw search spaces here are astronomical (9^81 Sudoku grids), and feasibility comes entirely from **early pruning** — rejecting a partial solution discards its whole exponential subtree. The leverage compounds with depth: a violation caught at level 2 of N-Queens kills ~n⁶ descendants. So the engineering centerpiece is making legality checks O(1): three boolean sets (columns, diagonals r−c+n, anti-diagonals r+c) for queens; rows/cols/boxes sets for Sudoku — updated in choose, rolled back in unchoose, the same discipline as the path buffer. **Prune early, check cheap, undo exactly** is the whole pattern.

**Template (word search — in-place marking):**
```cpp
bool dfs(vector<vector<char>>& b, int r, int c, const string& w, int k) {
    if (k == (int)w.size()) return true;                  // matched everything
    if (r < 0 || r >= m || c < 0 || c >= n || b[r][c] != w[k]) return false;
    char saved = b[r][c];
    b[r][c] = '#';                                        // choose: mark in place
    bool found = dfs(b, r+1, c, w, k+1) || dfs(b, r-1, c, w, k+1)
              || dfs(b, r, c+1, w, k+1) || dfs(b, r, c-1, w, k+1);
    b[r][c] = saved;                                      // unchoose: restore exactly
    return found;
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 79. Word Search | Medium | The template. Short-circuit `||` *is* the early exit; the restore must still run after it — note the template restores before returning. |
| 212. Word Search II | Hard | Many words → walk the board and a **Trie simultaneously**, so one traversal matches all words and dead prefixes prune instantly. Big upgrade habits: collect at trie-end nodes, and *remove found words from the trie* to keep pruning sharp. |
| 51. N-Queens | Hard | One queen per row → recursion level = row; legality = three O(1) lookups (cols, two diagonal families). The diagonal indexing (r−c offset, r+c) is the only real trick. |
| 52. N-Queens II | Hard | Count-only version — same search, integer instead of board reconstruction. Use it to time yourself on the pure engine. |
| 37. Sudoku Solver | Hard | Choose = next empty cell, branches = digits legal by row/col/box sets. The strong variant picks the **most-constrained cell first** (fewest legal digits) — order heuristics as pruning, worth mentioning. |
| 489. Robot Room Cleaner | Hard | Backtracking with a *physical* undo: to backtrack the robot must turn 180°, move, and restore orientation. The undo discipline made literal — a memorable design question. |
| 980. Unique Paths III | Hard | Hamiltonian-path counting on a grid: must cover every walkable cell. Track remaining count; prune when the end is reached early. Exhaustive coverage = no shortcuts exist, so backtracking is *the* tool. |
| 473. Matchsticks to Square | Medium | Partition into 4 equal sides: sort descending (big items first → early failure), skip equal-failed branches. Pruning-order strategy as the difference between TLE and AC. |

**Pitfalls:**
- Restore-after-short-circuit: `return dfs(...) || dfs(...);` before restoring leaks the mark. Compute into a variable, restore, then return (as the template does).
- N-Queens diagonal indexing: r−c is negative — offset by n (`diag[r−c+n]`). The +n is forgotten constantly.
- Pruning that's *almost* sound: a heuristic that can reject a state containing solutions breaks completeness silently. Distinguish "provably empty subtree" (safe) from "looks unpromising" (unsafe) explicitly.

---

## Pattern 7: DFS + Memoization — Top-Down DP

**Logic:** A recursive solution explores a decision space, but the same **subproblem** (same arguments) recurs across branches. Cache each result the first time (`memo[args]`); return the cached value on every revisit. The recursion is unchanged — only the redundancy is gone.

**Core insight — why it works:** The exponential blowup of naive recursion comes from solving identical subproblems in different branch contexts — but if the function is **pure given its arguments** (the answer depends only on the args, not the path that produced them), recomputation is pure waste. Memoization collapses the recursion *tree* into the recursion *DAG* of distinct states: complexity drops from O(branches^depth) to O(states × work-per-state). The recognition skill is **counting states before coding**: word-break has n suffixes; target-sum has n × (sum range) pairs; grid longest-path has m·n cells — if the count is polynomial, memoize and you're done. And the purity test doubles as the *path-state boundary*: if the answer depends on choices carried in shared mutated state (Patterns 5–6's `path`), the function isn't pure in its args and plain memoization is illegal — that's exactly the line between backtracking and DP.

**Template (word break):**
```cpp
unordered_map<int, bool> memo;                 // start index → breakable?
bool dfs(const string& s, int start, unordered_set<string>& dict) {
    if (start == (int)s.size()) return true;
    auto it = memo.find(start);
    if (it != memo.end()) return it->second;   // subproblem seen — reuse
    for (int end = start + 1; end <= (int)s.size(); end++)
        if (dict.count(s.substr(start, end - start)) && dfs(s, end, dict))
            return memo[start] = true;
    return memo[start] = false;
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 139. Word Break | Medium | The gateway: naive is exponential, but only n distinct suffixes exist → n states. Diff the memoized and unmemoized versions to *feel* the tree→DAG collapse. |
| 329. Longest Increasing Path in a Matrix | Hard | DFS-per-cell memoized; the strict increase means no cycles, so no visited set is needed — the memo alone suffices. The cleanest "DFS+memo on a grid" specimen, and a frequent hard. |
| 494. Target Sum | Medium | State = (index, running sum); memo on the pair (offset the sum to index an array). Counting version of top-down DP. |
| 140. Word Break II | Hard | Memoize *lists of sentences* per suffix. Honest caveat: worst cases are exponential in output size — saying "memoization can't beat the size of the answer" is exactly the right boundary statement. |
| 312. Burst Balloons | Hard | The famous reframe: recurse on which balloon bursts **last** in (i, j) — making subintervals independent. Memo on the interval. The reframe, not the memo, is the problem. |
| 1143. Longest Common Subsequence (top-down) | Medium | States = (i, j) pairs. Write it top-down first, then convert to the table — the conversion drill that demystifies bottom-up DP generally. |
| 698. Partition to K Equal Sum Subsets | Medium | Memo on a **bitmask** of used elements — state-space DP where DFS+memo is far more natural than a table. Bridges Patterns 5 and 7. |
| 87. Scramble String | Hard | Memo on (i, j, length) triples over a wild branching recursion — the "memo rescues an absurd-looking recursion" showcase. |

**Pitfalls:**
- Memoizing impure functions: if shared path state influences the answer, the cache poisons later branches with answers from different contexts. Run the purity test first, always.
- Caching only `true` (139-style searches): negative results recur just as much — cache both or keep the exponential blowup on unbreakable inputs.
- Map-key cost: encode multi-field states into one integer (`i * W + j`) or use arrays over `unordered_map` when ranges are dense — the difference between AC and TLE on tight limits.

---

## When DFS FAILS — Know the Boundary

| Situation | Why DFS breaks | Use instead |
|---|---|---|
| Shortest path / fewest moves, unit edges | DFS finds *a* path; depth order ≠ distance order | BFS |
| Weighted shortest path | Same, worse | Dijkstra |
| Recursion depth ~10⁵+ (deep lists/grids/skewed trees) | Call-stack overflow | Iterative DFS with explicit stack, or BFS |
| Exponential space, no overlapping subproblems, need optimum only | Enumeration is hopeless | Greedy with proof, or problem-specific structure |
| Overlapping subproblems but you didn't notice | Exponential when polynomial existed | Add the memo (Pattern 7) — count states! |
| Per-level / ring-structured questions | DFS has no level notion | BFS level loop |
| Dynamic connectivity (edges added over time) | Re-running DFS per query is O(V) each | Union-Find |

One-sentence litmus test: **DFS is the answer when you need the path itself, the subtree's summary, or every solution — and the wrong tool whenever "shortest" or "level" appears with unit steps.**

---

## Cheat Sheet: Problem Phrase → Pattern

| Phrase in the problem | Pattern |
|---|---|
| any binary-tree computation (depth, validate, mirror) | Tree recursion (P1) |
| "diameter / max path sum / best value over subtrees" | Post-order DP, return/record split (P2) |
| "root-to-leaf" sums, paths, counts | Pre-order path DFS (P3) |
| "connected components / can reach / clone the graph" | Graph DFS (P4) |
| "detect cycle" in a **directed** graph | Three-color DFS, gray = stack (P4) |
| "all subsets / permutations / combinations / partitions" | Backtracking (P5) |
| "place queens / solve the board / find the word in grid" | Constraint search + in-place marking (P6) |
| "number of ways / longest / can it be done" + overlapping subproblems | DFS + memo (P7) |
| "use every edge exactly once / itinerary" | Hierholzer (P4, advanced) |
| "shortest / minimum moves" | NOT DFS — go to the BFS guide |

---

## Complexity Summary

- Tree DFS / graph DFS: **O(V + E)** time; **O(depth)** stack (worst O(V) on degenerate shapes).
- Subsets: **O(n · 2ⁿ)**. Permutations: **O(n · n!)**. Output-bound — optimal by necessity.
- Backtracking with pruning: still exponential worst-case; pruning changes the constant universe, not the class.
- DFS + memo: **O(states × work-per-state)** — the whole point; count states first.
- Word Search: O(m·n · 3^L) (3, not 4 — no immediate backstep). N-Queens: ~O(n!).
- Iterative conversion: same complexity; trades call stack for an explicit one.

---

## Interview Tips

1. **State the function's contract before writing it.** "Returns the height of the subtree rooted here" — then take the recursive leap of faith and verify only single-node logic. Tracing into children out loud reads as not trusting recursion.
2. **Pre or post? Ask who determines the value.** Ancestors → pre-order parameter flowing down. Descendants → post-order return flowing up. Both (124, diameter) → the return/record split, and *say* you're splitting them.
3. **Narrate choose/explore/unchoose** as you write backtracking, and make the undo structurally unskippable. The symmetry is what interviewers are watching for.
4. **The gray set is your cycle vocabulary:** "visited alone fails on directed graphs — a diamond isn't a cycle; I need in-stack marking." That one sentence is a known checkpoint.
5. **Before accepting exponential, count the states.** "Branches look exponential, but only n·k distinct (i, sum) pairs exist — memoize." The habit converts half of 'hard backtracking' into routine DP.
6. **Flag recursion-depth risk yourself** on big inputs (10⁵-node trees, long lists, snake grids) and offer the iterative or BFS fallback — it's a senior-signal exactly because it bites in production, not just on judges.

---

## Suggested Practice Order

**Week 1 — tree recursion:** 104 → 226 → 100 → 101 → 111 → 110 → 98 → 112 → 129
**Week 2 — tree DP & paths:** 543 → 687 → 337 → 124 → 236 → 1448 → 113 → 437
**Week 3 — graphs & backtracking:** 547 → 841 → 133 → 207 → 802 → 78 → 90 → 46 → 39 → 22
**Week 4 — boss fights:** 51 → 79 → 212 → 37 → 131 → 139 → 494 → 329 → 968 → 312

Good luck with the interviews!

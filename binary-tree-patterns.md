# Binary Trees — The Complete Interview Pattern Guide (C++)

Binary-tree problems are the purest form of recursion in interviews: every subtree is itself a tree, so structural induction applies directly, and 90%+ of tree problems are a DFS wearing one of a handful of costumes. This guide covers general binary trees — recursion fundamentals, post-order tree DP, root-to-leaf path DFS, construction/serialization, and iterative traversals — with the **core insight** for each pattern, C++ templates, strong problem sets, and the pitfalls. BST-specific patterns (the in-order sorted-order superpower, ordered routing, structural surgery) live in the **BST guide**; non-tree DFS (graphs, grids, memoization) lives in the **DFS guide**; enumeration over decision trees lives in the **Backtracking guide**; level-order traversal lives in the **BFS guide**.

---

## The Two Moments — the Idea Behind Every Tree Pattern

> **Each node is touched twice: on the way down (pre-order, before its subtree) and on the way up (post-order, after its subtree) — and choosing which moment your logic belongs to is 80% of designing a tree algorithm.**

- **Pre-order is where inherited state flows down**: the path so far, the running sum, bounds imposed by ancestors. Only *ancestors* determine these values.
- **Post-order is where synthesized answers flow up**: subtree height, subtree sum, best path below. Only *descendants* determine these values.
- **The decision rule:** ask *"who determines this value — ancestors or descendants?"* Ancestors → pre-order parameter flowing down (Pattern 3). Descendants → post-order return flowing up (Pattern 2). Both at once (diameter, max path sum) → the return/record split, the signature move of hard tree problems.
- **In-order** (left, node, right) is the third moment — on general binary trees it's just another visit order, but on BSTs it yields sorted order, which is why it anchors the BST guide, not this one.

The other master habit: the **recursive leap of faith**. State the function's contract ("returns the height of the subtree rooted here"), *assume* it holds on the children, and verify only the single-node logic. Tracing into child calls three levels deep is how interviews are lost.

---

## How to Recognize Which Pattern

- "Depth / same / symmetric / invert / merge" — plain structural recursion (Pattern 1).
- "Diameter / max path sum / count subtrees with property / best value over subtrees" — post-order DP with the return/record split (Pattern 2).
- "Root-to-leaf" anything — sums, paths, counts — pre-order path DFS (Pattern 3).
- "Construct from traversals / serialize / deserialize / flatten" — build patterns (Pattern 4).
- "Without recursion / design an iterator" — explicit-stack traversals (Pattern 5).
- "Validate BST / kth smallest / sorted anything / insert / delete" — **not this guide**: the BST guide.
- "Level by level / zigzag / right side view / minimum depth fastest" — **not this guide**: BFS level loop (see the BFS guide).

---

## Pattern 1: Tree Recursion Fundamentals

**Logic:** Define the function's meaning *on a single node*, trust it on the children, combine. `maxDepth(node) = 1 + max(maxDepth(l), maxDepth(r))`, base case `nullptr → 0`. Every tree problem is: pick the right return value, the right base case, and the right combiner.

**Core insight — why it works:** Trees are recursively self-similar — every subtree is itself a tree — so structural induction applies directly: if the function is correct on all smaller trees (children), and the combine step is correct, it's correct everywhere. The practical discipline this buys you is the **recursive leap of faith**: never trace into the child calls; *assume* they return what the function's contract promises, and verify only the single-node logic.

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
| 222. Count Complete Tree Nodes | Medium | Naive count is O(n); completeness lets you compare left/right spine heights — equal means a perfect left subtree of known size (2^h − 1). O(log² n): structure exploited, not just traversed. |

**Pitfalls:**
- Base case sloppiness: nullptr vs leaf are different bases (111 punishes conflating them). Decide which your contract uses and write it first.
- Comparing only local parent-child relations when the property is global — the bound must travel down (the classic Validate-BST failure; see the BST guide).
- `int` vs `long long` for accumulated sums — path sums over 10⁴ nodes of ±10⁴ overflow `int` in several problems.

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
| 979. Distribute Coins in Binary Tree | Medium | Return the subtree's coin *surplus* (may be negative); moves += \|left surplus\| + \|right surplus\|. Flow-across-an-edge accounting — a beautiful return/record cousin. |
| 508. Most Frequent Subtree Sum | Medium | Return subtree sum, record frequencies in a map. The gentle on-ramp to return/record. |
| 1110. Delete Nodes and Return Forest | Medium | Post-order deletion: decide children first so the parent can null its pointers safely, collecting orphaned subtrees as roots. Structural surgery needs post-order for the same reason `rm -r` does. |
| 236. Lowest Common Ancestor | Medium | Return "found p, q, or their LCA below me": null/null → null, x/null → x, x/y → this node *is* the LCA. The most elegant post-order argument on LeetCode — be able to state *why* the both-sides-non-null case is conclusive. |
| 863. All Nodes Distance K | Medium | The escape hatch when a path must go *up*: post-order finds the target's ancestors and the distance to each, then DFS down the "other side" of each ancestor. Alternatively: build a parent map and BFS — know both. |

**Pitfalls:**
- Returning the recorded (bent) value instead of the extendable (straight) one — the parent then double-counts a bend. If your diameter is too big, this is why.
- Forgetting to clamp negative contributions in 124-style sums — `max(0, child)` is a *decision* (take the arm or not), not a detail.
- Multi-state returns (337, 968): define the states' meanings in comments before coding; off-by-one-state is unrecoverable mid-interview otherwise.

---

## Pattern 3: Root-to-Leaf Path DFS — Inherit Down

**Logic:** The mirror of Pattern 2: state accumulates on the way **down** (pre-order) — running sum, the path so far, max seen on the path — and conclusions are drawn at **leaves** (or at every node). Pass inherited state as parameters; parameters auto-rollback on return.

**Core insight — why it works:** A root-to-leaf path corresponds exactly to one chain of recursive calls, so the call stack *is* the path — pass-by-value parameters give each branch its own copy of the inherited state for free, which is why simple path problems need no explicit undo. The pre/post choice is forced by information direction: "sum from root to here" is inherited (only ancestors determine it) → pre-order; "height below here" is synthesized (only descendants determine it) → post-order. When the inherited state is heavy (a shared `path` vector), you switch to mutate-and-undo — which is exactly the choose/explore/unchoose discipline of the Backtracking guide, revealing these as one family.

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

## Pattern 4: Construct & Serialize — Trees from Sequences

**Logic:** Rebuild a tree from traversal orders, or flatten one to a string and back. The engine: **pre-order (or post-order) tells you who the root is; in-order (or null markers) tells you where the split is.** Consume the root sequence with a moving index; determine each root's left/right ranges; recurse.

**Core insight — why it works:** A single traversal order doesn't determine a tree (many shapes share a pre-order) — you need a second source of structure. For general binary trees there are two in circulation: (1) an **in-order sequence** — the root's position in it splits left values from right values (hash value→index for O(1) splits); (2) **null markers** in the serialization — the shape is encoded explicitly, so pre-order alone suffices (297). (BSTs offer a third — value bounds replace the in-order — covered in the BST guide.) In every case the pre-order index advances monotonically — it's consumed left-subtree-first, which is *why* recursing left before right is mandatory, and each element is consumed once → O(n).

**Template (105 — pre-order + in-order):**
```cpp
unordered_map<int,int> pos;                    // value → in-order index (built once)
int pre = 0;                                   // pre-order consumption pointer
TreeNode* build(vector<int>& preorder, int lo, int hi) {   // in-order range [lo, hi]
    if (lo > hi) return nullptr;
    TreeNode* root = new TreeNode(preorder[pre++]);        // pre-order names the root
    int mid = pos[root->val];                              // in-order locates the split
    root->left  = build(preorder, lo, mid - 1);            // MUST build left first
    root->right = build(preorder, mid + 1, hi);
    return root;
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 105. Construct from Preorder and Inorder | Medium | The template. The two roles — pre-order names roots, in-order splits ranges — should be speakable as a sentence before you code. |
| 106. Construct from Inorder and Postorder | Medium | Mirror: consume post-order **backwards** (last = root) and build **right subtree first**. Deriving the mirror yourself, rather than memorizing it, is the test. |
| 654. Maximum Binary Tree | Medium | Root = range max, recurse on the sides. O(n²) naive; the monotonic-stack O(n) build is the follow-up (cross-link to the stack guide). |
| 297. Serialize and Deserialize Binary Tree | Hard | Pre-order with explicit `#` null markers → shape fully encoded, one pass each way with an index/stream. The design question: delimiters, negative values, and why level-order also works (BFS flavor). |
| 889. Construct from Preorder and Postorder | Medium | Ambiguous when a node has one child (either side fits) — any valid answer accepted. Understand *where* the ambiguity comes from; it completes the traversal-pairs picture. |
| 114. Flatten Binary Tree to Linked List | Medium | The reverse direction: tree → pre-order list, in place. Reverse-pre-order (right, left, node) with a trailing pointer is the elegant version. |
| 331. Verify Preorder Serialization | Medium | Validation without building: track available "slots" (each node consumes one, non-null opens two). The serialization's shape-encoding argument, run as arithmetic. |

**Pitfalls:**
- Recursion order is load-bearing: with a shared pre-order index, building right before left silently consumes the wrong elements — everything shifts and nothing errors.
- 106's index runs backwards; copying 105's forward habits produces mirrored garbage.
- Serialization edges: negative numbers and multi-digit values kill character-based parsing — use a delimiter and a stream/istringstream, and say so proactively.
- Duplicated values break the value→index map (105/106 guarantee uniqueness; confirm before relying on it — interviewers plant this question).

---

## Pattern 5: Iterative Traversals — The Stack Made Explicit

**Logic:** Replace the call stack with your own `stack<TreeNode*>`. In-order: dive down the left spine pushing everything, pop (that's the in-order visit), then enter the popped node's right child and repeat. This isn't just recursion-phobia: it's the engine of pausable traversals (iterators), O(h)-memory streaming, and the answer to "now do it without recursion."

**Core insight — why it works:** The recursion's call stack at any moment holds exactly the left-spine ancestors whose "visit" moment hasn't come yet — the explicit stack stores precisely that, nothing more. The pop *is* the in-order moment: everything in the left subtree is already emitted, so the node is next; then its right subtree gets the same treatment. Pre-order iterative is easier (visit on push; push right child first so left pops first). Post-order is the awkward one — visit must wait for *both* subtrees — solved by the reverse trick: generate node-right-left pre-order, reverse the output. For O(1) space, **Morris traversal** threads temporary right-pointers from each left subtree's rightmost node back to its root — mention it, code it only if asked.

**Template (iterative in-order):**
```cpp
stack<TreeNode*> st;
TreeNode* cur = root;
while (cur || !st.empty()) {
    while (cur) { st.push(cur); cur = cur->left; }  // dive the left spine
    cur = st.top(); st.pop();
    visit(cur);                                     // the in-order moment
    cur = cur->right;                               // then the right subtree
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 94. Binary Tree Inorder Traversal | Easy | The template. The loop condition (`cur || !st.empty()`) and the two-phase rhythm are what to drill. |
| 144. Binary Tree Preorder Traversal | Easy | Visit on push; push right before left. The one-stack one-pass easy case. |
| 145. Binary Tree Postorder Traversal | Easy | The reverse trick: modified pre-order (node, right, left), reverse at the end. Honest one-stack post-order (with a `last visited` pointer) is the harder follow-up. |
| 341. Flatten Nested Iterator | Medium | Not a binary tree, but the same "pausable DFS with an explicit stack" design — the generalization interviewers reach for after traversals. |
| 173. BST Iterator | Medium | The in-order template *paused between pops* — cross-listed from the BST guide, where lazy sorted streaming is the point. |

**Pitfalls:**
- In-order loop condition: `cur || !st.empty()` — either alone terminates early (empty stack mid-traversal with `cur` set is a legal state).
- Post-order via reverse produces the right *sequence* but visits in the wrong *order* — unusable when you need side effects at true post-order time (deletion, DP). Know which you need.
- Iterator space claims: "O(1)" with a full pre-pushed in-order vector is O(n) — the left-spine-only stack is what earns O(h).

---

## When Binary Trees Are NOT Plain DFS — Know the Boundary

| Situation | Why recursion is wrong (or not enough) | Use instead |
|---|---|---|
| Level-by-level output, zigzag, right-side view | DFS has no level notion (or fakes it with depth params) | BFS level loop (BFS guide) |
| Minimum depth on a wide shallow tree | DFS explores whole subtrees before finding the shallow leaf | BFS — first leaf found is the answer |
| Ordered queries (kth smallest, range, closest) | Plain traversal is O(n) per query | The BST invariant (BST guide) |
| 10⁵-node degenerate (skewed) tree | Call-stack overflow — depth = n | Iterative traversal (Pattern 5) or Morris |
| Path between two arbitrary nodes going *up* | Plain DFS only goes down | Parent map + BFS, or ancestor decomposition (863) |
| Repeated LCA queries on a static tree | O(h) per query adds up | Binary lifting / Euler tour + RMQ (mention only) |

---

## Cheat Sheet: Problem Phrase → Pattern

| Phrase in the problem | Pattern |
|---|---|
| "depth / same / symmetric / invert / merge trees" | Structural recursion (P1) |
| "diameter / max path sum / best over subtrees / count subtrees" | Post-order DP, return/record split (P2) |
| "root-to-leaf" sums, paths, counts | Pre-order path DFS (P3) |
| "construct from traversals / serialize / flatten" | Build patterns — root source + split source (P4) |
| "without recursion / design an iterator" | Explicit stack traversal (P5) |
| "validate BST / kth smallest / sorted / insert / delete" | NOT here — BST guide |
| "level order / zigzag / right side view" | NOT here — BFS guide |
| "all root-to-leaf paths enumerated with shared state" | Choose/explore/unchoose — Backtracking guide |

---

## Complexity Summary

- Any single full traversal (P1–P5): **O(n)** time; **O(h)** stack — h = log n balanced, n degenerate.
- 222 (complete tree count): O(log² n) — two spine walks per level of recursion.
- Construction from traversals: O(n) with the value→index hashmap; O(n²) without (linear scans per split).
- Morris traversal: O(n) time, **O(1)** space — each edge walked at most twice.
- 572 naive: O(n·m); serialize+hash follow-up: O(n+m).

---

## Interview Tips

1. **State the contract, then leap.** "This returns the height of the subtree rooted here." Verify single-node logic only — tracing into children reads as not trusting recursion.
2. **Pre or post? Ask who determines the value.** Ancestors → parameter down. Descendants → return up. Both → return/record split, and *say* you're splitting them.
3. **Leaf = `!left && !right`.** Say the definition out loud before any root-to-leaf problem — the null-based "leaf" test is the most recycled bug in tree interviews.
4. **For construction problems, name the two roles first**: "pre-order names roots; in-order splits ranges." The sentence proves you understand the engine before the indices fly.
5. **Flag recursion-depth risk yourself** on 10⁵-node inputs (skewed trees) and offer the iterative fallback — it's a senior signal because it bites in production, not just on judges.
6. **Know when the problem is secretly a BST problem** — the moment ordering/rank/range language appears, switch guides and say why: the invariant buys O(h).

---

## Suggested Practice Order

**Week 1 — fundamentals:** 104 → 226 → 100 → 101 → 111 → 110 → 617 → 572 → 222
**Week 2 — tree DP:** 543 → 687 → 508 → 337 → 979 → 124 → 236 → 863
**Week 3 — paths:** 112 → 113 → 129 → 257 → 1448 → 437 → 988 → 1457
**Week 4 — build, serialize, iterate, boss fights:** 105 → 106 → 654 → 889 → 114 → 297 → 331 → 94 → 145 → 968

Good luck with the interviews!

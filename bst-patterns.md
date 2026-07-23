# Binary Search Trees — The Complete Interview Pattern Guide (C++)

A BST is a sorted array wearing pointers. Everything left of a node is smaller, everything right is larger — recursively — and that single invariant powers every pattern in this guide: sorted-order traversal for free, O(h) routing instead of O(n) search, structural surgery that rebuilds only one spine, and construction that needs less information than a general tree. This guide covers the full BST territory — the in-order superpower, ordered routing, insert/delete/trim surgery, building from sequences, and ordered iteration — with the **core insight** for each pattern, C++ templates, strong problem sets, and the pitfalls. General binary-tree recursion (depth, diameter, paths, serialization of arbitrary trees) lives in the **Binary Tree guide**.

---

## The Invariant — the Idea Behind Every BST Pattern

> **left subtree < node < right subtree, recursively — which means (1) in-order traversal visits values in sorted ascending order, and (2) any value comparison at a node tells you which subtree to enter.**

Those are the guide's two engines:

- **Consequence 1 — sorted order for free (Patterns 1, 5).** Any problem phrased over the sorted sequence of values — kth smallest, min gap, validation, mode, repair — is an in-order walk with a `prev` pointer. No extraction to an array needed.
- **Consequence 2 — routing (Patterns 2, 3).** `key < node → go left; key > node → go right` collapses the search space by a subtree per step: O(h) search, insert, delete, LCA, successor. A balanced BST *is* binary search as a data structure.
- **The global trap.** The invariant constrains a node against **all** ancestors, not just its parent — a node in the left subtree must be smaller than the *root*, not merely its parent. Every "check parent vs child" solution to a BST problem is wrong for this reason, and interviewers know it.
- **Strictness.** LeetCode BSTs are strict (`<`, `>`, no duplicates unless stated). Confirm before assuming — duplicate policy changes routing and validation.

---

## How to Recognize Which Pattern

- "Validate / kth smallest / minimum difference / mode / two swapped nodes / convert to greater tree" — the in-order invariant (Pattern 1).
- "Search / insert / closest value / LCA **of a BST** / successor" — routing, one-sided recursion (Pattern 2).
- "Delete / trim / split / balance" — structural surgery with return-and-reattach (Pattern 3).
- "Construct from sorted array / sorted list / preorder", "serialize a BST", "how many BSTs" — build patterns (Pattern 4).
- "Design an iterator / two sum in a BST / merge two BSTs' values" — ordered streaming (Pattern 5).
- "Diameter / max path sum / root-to-leaf paths" on a BST — the ordering is a red herring; use the **Binary Tree guide**'s patterns.

---

## Pattern 1: The In-Order Invariant — Sorted Order for Free

**Logic:** In-order traversal of a BST visits values in **strictly increasing order**. Any problem phrased over the *sorted sequence* of a BST's values is solved by an in-order walk carrying a `prev` pointer (or a countdown), never by extracting values into an array first.

**Core insight — why it works:** The BST property is *global*, and in-order is precisely the traversal that linearizes it: left-subtree values, then node, then right-subtree values, each block internally sorted by induction. So the walk gives you adjacent-pair access to the sorted sequence — min gap, inversion detection, duplicate runs — in O(n) time and O(h) space with zero extra structure. The dual formulation for validation: pass down `(lo, hi)` bounds — each node constrains its subtrees' *ranges* — which is the same invariant read top-down instead of linearized. Know both; interviewers ask for "the other one."

**Template (in-order with prev — the workhorse):**
```cpp
TreeNode* prev = nullptr;
bool inorder(TreeNode* node) {                 // validation flavor
    if (!node) return true;
    if (!inorder(node->left)) return false;
    if (prev && prev->val >= node->val) return false;  // sorted ⇒ strictly increasing
    prev = node;                               // the in-order moment: compare, then advance
    return inorder(node->right);
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 98. Validate Binary Search Tree | Medium | Both arguments matter: (lo, hi) bounds flowing down, or in-order strictly increasing. Checking only parent vs child is the canonical wrong answer — a node deep in the left subtree can still violate the *root's* bound. |
| 530. Minimum Absolute Difference in BST | Easy | The min gap in sorted order is between *adjacent* values — exactly what `prev` exposes. The cleanest prev-pointer specimen. |
| 230. Kth Smallest Element in a BST | Medium | In-order with a countdown; stop at zero. Follow-up ("frequent inserts?"): augment nodes with subtree sizes for O(h) rank queries — say it, don't code it. |
| 501. Find Mode in BST | Easy | Duplicates cluster adjacently in sorted order; count runs with `prev`. O(1) extra space (two passes or vector-reset trick) is the follow-up. |
| 99. Recover Binary Search Tree | Medium | Two swapped nodes = one or two *inversions* (`prev->val > cur->val`) in the in-order sequence: first inversion's `prev`, last inversion's `cur` — swap their values. Inversion-detection on the implicit sorted array. |
| 538. Convert BST to Greater Tree | Medium | **Reverse** in-order (right, node, left) visits values descending; carry a running suffix sum. The mirror trick — know that in-order has a mirror. |
| 897. Increasing Order Search Tree | Easy | Rebuild the right-spine-only tree during an in-order walk. Relink-while-traversing practice — pointer discipline under traversal. |
| 783. Minimum Distance Between BST Nodes | Easy | 530 verbatim under a different number — included so you recognize the duplicate and bank the five minutes. |

**Pitfalls:**
- `prev` initialization: use a pointer (nullptr = "nothing yet"), not a sentinel value like INT_MIN — node values can *be* INT_MIN (98's official tests include it), which also forces `long long` if you insist on value bounds.
- Strict vs non-strict: `prev->val >= cur->val` fails validation; `>` alone misses duplicates.
- Early termination: kth-smallest must actually stop (return/flag), or you've done O(n) work for an O(h + k) answer — interviewers notice.
- 99's second swap node is the *last* inversion's `cur`, not the first's — adjacent-node swaps produce only one inversion, and that case breaks "collect two inversions" code written carelessly.

---

## Pattern 2: Routing — Search, Insert, and the Split Point

**Logic:** The invariant is a *routing table*: key < node → everything relevant is left; key > node → right; equal → arrived. Search and insert recurse into **one** child — O(h), not O(n). LCA of two values is the **split point**: the first node where they route different directions. Successor/closest-value are routing walks that remember the best candidate seen.

**Core insight — why it works:** Each comparison discards an entire subtree with certainty — the invariant guarantees the key *cannot* be there — which is the same halving argument as binary search on an array (and degrades the same way: O(n) when the tree is a path, which is why balanced variants exist). The split-point argument for LCA: while both targets route the same way, the current node is a common ancestor but not the lowest (both live in one subtree); the first divergence node has one target on each side, so no deeper node can be an ancestor of both — it's the LCA, found top-down with no post-order needed. Contrast that explicitly with the general-tree LCA (236, Binary Tree guide), which must search both sides *because* it has no routing information.

**Template (LCA of a BST — the split point):**
```cpp
TreeNode* lca(TreeNode* root, TreeNode* p, TreeNode* q) {
    while (root) {
        if (p->val < root->val && q->val < root->val)      root = root->left;   // both left
        else if (p->val > root->val && q->val > root->val) root = root->right;  // both right
        else return root;      // split point (or equal to p/q) — the LCA
    }
    return nullptr;
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 700. Search in a BST | Easy | The routing table in its purest form — three lines. Write it to feel the one-sided recursion before anything harder. |
| 701. Insert into a BST | Medium | Route until null, attach there. Any valid position is accepted — leaf insertion is always valid, no rebalancing asked. First outing of the return-and-reattach idiom (`root->left = insert(root->left, v)`). |
| 235. LCA of a BST | Medium | The template. Pure routing, O(h), iterative in five lines — contrast explicitly with 236's post-order general-tree version; answering one with the other is a self-inflicted wound. |
| 270. Closest BST Value (premium) | Easy | Route toward the target, tracking the best-so-far at each step. The "remember a candidate while routing" move that successor problems reuse. |
| 285. Inorder Successor in BST (premium) | Medium | No traversal needed: successor = leftmost of right subtree, else the last ancestor where you turned left. Routing with a remembered candidate — O(h), no parent pointers. |
| 653. Two Sum IV — Input Is a BST | Easy | The honest answer is in-order + two pointers or a hash set — the BST helps less than it looks. Saying *when the invariant doesn't pay* is part of mastering it. |
| 704-adjacent: 96. Unique Binary Search Trees | Medium | Count shapes: each value as root splits values into left/right counts → Catalan recurrence. A DP problem in BST clothing — recognize it as counting, not construction. |

**Pitfalls:**
- LCA with `<`/`>` only: when the node equals p or q, that node *is* the answer — the equality case must terminate, not route.
- Comparing by pointer when values are the routing key (or vice versa) — pick one, and remember LeetCode guarantees p and q exist in the tree (say you're relying on it).
- Successor logic: "leftmost of right subtree" covers only the has-right-child case; the ancestor fallback is the half that candidates forget.
- Routing on a tree that was *stated* to be a BST but might be degenerate: complexity answers are O(h), and h may be n — say both.

---

## Pattern 3: Structural Surgery — Delete, Trim, Return-and-Reattach

**Logic:** Mutations follow one universal shape: `root->left/right = op(child, ...)` — each call returns the (possibly new) subtree root and the parent reattaches it, so no parent pointers are ever needed. Delete routes to the node, then splices (0–1 children) or swaps with the in-order successor (2 children). Trim uses bounds to discard whole subtrees at once.

**Core insight — why it works:** The return-and-reattach idiom works because every mutation is local once routing is done: the recursion rebuilds only the O(h) spine of ancestors above the change, sharing every untouched subtree — the same structural-sharing insight that powers persistent trees. Delete's two-children case is the one real puzzle: the node's replacement must keep the invariant, and exactly two values can — the in-order predecessor or successor. Take the successor (leftmost of right subtree): it has **no left child by construction** (it's a leftmost node), so deleting *it* is guaranteed to hit the easy 0–1-children case — the hard case provably reduces to the easy one, once. Trim's leverage is the bound argument: if `root->val < lo`, the invariant says the entire left subtree is also `< lo` — discard it without looking.

**Template (delete — the hardest of the family):**
```cpp
TreeNode* deleteNode(TreeNode* root, int key) {
    if (!root) return nullptr;
    if      (key < root->val) root->left  = deleteNode(root->left, key);   // route left
    else if (key > root->val) root->right = deleteNode(root->right, key);  // route right
    else {                                          // found it
        if (!root->left)  return root->right;       // 0–1 children: splice
        if (!root->right) return root->left;
        TreeNode* s = root->right;                  // 2 children: in-order successor
        while (s->left) s = s->left;
        root->val = s->val;                         // copy successor up
        root->right = deleteNode(root->right, s->val);  // delete it below (easy case now)
    }
    return root;                                    // parent reattaches
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 450. Delete Node in a BST | Medium | The template above. The two-children → successor-swap reduction is the interview checkpoint; be able to say *why* the successor has no left child. |
| 669. Trim a BST | Medium | If root < lo, the *entire* left subtree is out too — return trimmed right. Bound reasoning prunes whole subtrees; return-and-reattach does the surgery. |
| 1382. Balance a Binary Search Tree | Medium | In-order flatten to a sorted vector (Pattern 1), then rebuild middle-out (Pattern 4's 108). Two patterns composed — a great "combine your tools" question. |
| 776. Split BST (premium) | Medium | Return a *pair* of roots (≤ target, > target), reattaching recursively on the routing side. Return-and-reattach generalized to two simultaneous trees — the pattern's graduate exam. |
| 701. Insert into a BST | Medium | Cross-listed from Pattern 2: the same idiom in its gentlest form — route, attach, return. Write insert and delete back-to-back to feel the shared skeleton. |
| 938. Range Sum of BST | Easy | Not a mutation, but the same bound-pruning: skip left entirely when node ≤ lo. Trim's read-only twin — and the gentlest first problem of this pattern. |

**Pitfalls:**
- Forgetting to *use* the return value: `deleteNode(root->left, key);` without `root->left =` silently drops the mutation — the single most common BST-surgery bug.
- Delete's successor swap: copy the value then recurse-delete *in the right subtree* for that value; splicing the successor out by pointer without recursion orphans its right child.
- Trim vs delete confusion: trim can discard whole subtrees by bounds; delete must reattach *both* children of the removed node. Applying trim's shortcut to delete loses data.
- Memory hygiene: LeetCode doesn't check `delete`d nodes, but say the words "in production I'd free the spliced node" — it costs nothing and reads as care.

---

## Pattern 4: Build — Sorted Input to Balanced Tree, Preorder to Unique Tree

**Logic:** Two build directions. From **sorted input** (array/list): middle element = root, recurse on halves → height-balanced by construction. From **pre-order alone**: the BST invariant replaces the in-order — pass (lo, hi) bounds and attach each next value where it fits; no second traversal and no null markers needed (which is why serializing a BST is cheaper than serializing a general tree).

**Core insight — why it works:** Sorted-array → BST is binary search run as a *builder*: choosing the middle as root puts half the values on each side at every level, so depth is ⌈log n⌉ — balance isn't checked, it's forced by construction. Pre-order → BST works because the invariant is a second information source: the general-tree ambiguity ("many shapes share a pre-order") disappears when value bounds dictate where each value *must* attach — pre-order of a BST determines the tree uniquely. That's also the compare-and-contrast with 297 (Binary Tree guide): general trees need null markers to pin the shape; BSTs need only the values — half the bytes, same O(n) rebuild.

**Template (1008 — BST from preorder with bounds):**
```cpp
int i = 0;                                       // pre-order consumption pointer
TreeNode* build(vector<int>& pre, int lo, int hi) {
    if (i == (int)pre.size() || pre[i] < lo || pre[i] > hi)
        return nullptr;                          // this value belongs to an ancestor's other side
    TreeNode* root = new TreeNode(pre[i++]);
    root->left  = build(pre, lo, root->val - 1); // left must be < root
    root->right = build(pre, root->val + 1, hi); // right must be > root
    return root;
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 108. Convert Sorted Array to BST | Easy | Middle = root, recurse halves. The one-liner insight: balance by construction, not by checking. |
| 109. Convert Sorted List to BST | Medium | Same idea, but a list has no O(1) middle. Two answers: fast/slow pointer per level (O(n log n)), or the elegant inversion — build **in in-order** while advancing the list pointer (O(n)). The second is the differentiator. |
| 1008. Construct BST from Preorder | Medium | The template. The bounds *are* the in-order information — say that sentence and the interviewer knows you see the structure. |
| 449. Serialize and Deserialize BST | Medium | Pre-order values only, rebuild with 1008 — no null markers. The compare-and-contrast with 297 is the interview conversation: what the invariant buys in bytes. |
| 95. Unique Binary Search Trees II | Medium | Generate all shapes: each value as root, cross-product of left/right sub-results. Enumeration flavor of 96's counting — recursion returning *lists of trees*. |
| 1382. Balance a Binary Search Tree | Medium | Cross-listed from Pattern 3: flatten + 108. The rebuild half lives here. |

**Pitfalls:**
- 108 midpoint bias: `(lo+hi)/2` consistently — either bias is fine, mixing them isn't "more balanced," and `lo + (hi-lo)/2` avoids overflow habits carrying from binary search.
- 1008's bounds test must *not consume*: if `pre[i]` is out of range, return null **without** `i++` — the value belongs to an ancestor's other side. Consuming it there corrupts everything after.
- 109's in-order build: the list pointer advances exactly once per node *between* the two recursive calls — moving it before the left recursion rebuilds the list in the wrong positions.
- Serializing a BST with null markers works but wastes the invariant — fine as a first answer, name the optimization before being asked.

---

## Pattern 5: Ordered Iteration — The BST as a Sorted Stream

**Logic:** Treat the BST as a lazily-unrolled sorted array: an explicit stack holding the left spine yields the next-smallest element on demand — O(h) memory, amortized O(1) per element. This powers the iterator design question, k-way problems over multiple BSTs, and two-pointer techniques (smallest-from-one-end, largest-from-the-other with a reverse spine).

**Core insight — why it works:** The stack holds exactly the ancestors whose values are still pending — the left spine of the unvisited region. Popping yields the minimum of everything remaining (nothing smaller exists: its left subtree is exhausted by the spine construction); pushing the popped node's right-child spine restores the property. Each node is pushed and popped exactly once across the whole iteration → amortized O(1) `next()`, worst-case O(h) — and the amortization argument ("total work / total calls," not per-call) is precisely what the interviewer wants said. With two such streams — or a forward and a reverse spine on one tree — you get sorted-array two-pointer techniques on trees without materializing arrays.

**Template (173 — the pausable in-order):**
```cpp
class BSTIterator {
    stack<TreeNode*> st;
    void pushSpine(TreeNode* n) { while (n) { st.push(n); n = n->left; } }
public:
    BSTIterator(TreeNode* root) { pushSpine(root); }
    int next() {
        TreeNode* n = st.top(); st.pop();
        pushSpine(n->right);                     // restore the invariant
        return n->val;
    }
    bool hasNext() { return !st.empty(); }
};
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 173. BST Iterator | Medium | The template — the pattern's flagship. O(h) space and *amortized* O(1) next() are the claims to defend; a pre-computed vector is the O(n)-space non-answer. |
| 230. Kth Smallest (iterative flavor) | Medium | 173's machinery, stop at the kth pop — cleaner early termination than recursive flags. Same problem as Pattern 1's version; compare the two on purpose. |
| 653. Two Sum IV — Input Is a BST | Easy | The strong solution: forward iterator + backward iterator (reverse spine) = two pointers on the implicit sorted array, O(h) space. Beats the hash set on space and on style points. |
| 1305. All Elements in Two BSTs | Medium | Two iterators merged like merge-sort's merge step — k-way merging of lazily unrolled sorted streams. The pattern composing with itself. |
| 285. Inorder Successor (iterator view) | Medium | Successor = "the next() after finding the node" — a third way to see it, connecting Patterns 2 and 5. |

**Pitfalls:**
- Pushing the whole tree (both children) instead of the left spine — the stack becomes a generic DFS stack and emits the wrong order.
- Claiming O(1) `next()` without the amortized qualifier — worst case is O(h) (a long right-then-left spine); precision here is the point of the question.
- Reverse iterator symmetry: push the *right* spine, descend `n->left` after popping — mirror every occurrence, half-mirrored code emits garbage that looks almost sorted.
- 1305 with arrays-then-sort is O(n log n) and concedes the question — the merge of two O(h) streams is what's being tested.

---

## When the BST Invariant Does NOT Help — Know the Boundary

| Situation | Why the invariant is useless (or a trap) | Use instead |
|---|---|---|
| Diameter, max path sum, root-to-leaf paths on a BST | The question ignores value order — it's shape, not rank | Binary Tree guide patterns |
| Degenerate (path-shaped) BST | h = n; all O(h) claims silently become O(n) | Say it; mention self-balancing trees (AVL/Red-Black = `std::set`) |
| Heavy inserts + rank/order-statistics queries | Plain BST can't count ranks in O(h) without help | Augment with subtree sizes, or policy trees / ordered set |
| "Closest k values" streamed both directions | One iterator only walks one way | Two spines (forward + reverse) — Pattern 5 |
| Level-order / structural printing of a BST | Sorted order ≠ level order | BFS guide |
| Unsorted-array → BST by repeated insert | O(n²) on sorted-ish input — the degenerate trap | Sort once + 108, or note the randomization argument |

---

## Cheat Sheet: Problem Phrase → Pattern

| Phrase in the problem | Pattern |
|---|---|
| "validate / kth smallest / min difference / mode / recover / greater tree" | In-order invariant + prev pointer (P1) |
| "search / insert / closest / LCA of a BST / successor" | Routing, split point (P2) |
| "delete / trim / split / balance" | Surgery + return-and-reattach (P3) |
| "from sorted array / sorted list / from preorder / serialize BST" | Build — invariant as information source (P4) |
| "iterator / two sum in BST / merge two BSTs" | Ordered streaming, lazy spine stack (P5) |
| "how many structurally unique BSTs" | Catalan counting (96 — P2 note, DP guide) |
| "diameter / paths / any shape question" on a BST | NOT here — Binary Tree guide |
| "range queries with updates" | NOT here — ordered set / segment tree territory |

---

## Complexity Summary

- Routing ops (search / insert / delete / LCA / successor / closest): **O(h)** — log n balanced, n degenerate. This *is* the invariant's value; always state both cases.
- In-order walks (validate, kth, gap, mode, recover): **O(n)** time, **O(h)** stack — with early stop, kth smallest is O(h + k).
- Build: 108/109(in-order trick)/1008: **O(n)**. Repeated inserts instead: O(n log n) average, **O(n²)** adversarial.
- Iterator: O(h) space; `next()` amortized **O(1)**, worst-case O(h); full iteration O(n) total.
- Range sum with pruning (938): O(nodes in range + h) — the pruning is the answer, not the traversal.

---

## Interview Tips

1. **Say "in-order gives me sorted order" out loud** the moment you see a BST — it's the recognition checkpoint, and half the BST catalog collapses right there.
2. **Validate-BST is a known trap** — mention the failing local-check counterexample (a left-subtree node exceeding the root) before writing the bounds version, and offer both formulations (bounds down / in-order prev).
3. **Mutating? Return-and-reattach.** `root->left = op(root->left)` — narrate that no parent pointers are needed because each call returns the new subtree root.
4. **Delete's successor argument is the checkpoint**: "the in-order successor is a leftmost node, so it has no left child, so deleting it is the easy case." Three clauses, memorized as logic, not code.
5. **Know the two LCAs**: 235 (BST — routing, O(h), iterative) vs 236 (general — post-order argument). Answering one with the other's solution is a common self-inflicted wound.
6. **Always give O(h) with both readings** — "O(log n) if balanced, O(n) if degenerate; production trees self-balance, interview trees don't." One sentence, senior signal.
7. **When the BST-ness is a red herring** (diameter on a BST, 653's hash-set solution), say so — knowing when the invariant doesn't pay is stronger evidence of mastery than using it.

---

## Suggested Practice Order

**Week 1 — invariant & routing:** 700 → 701 → 938 → 98 → 530 → 230 → 501 → 235 → 270
**Week 2 — surgery & repair:** 450 → 669 → 99 → 538 → 897 → 1382
**Week 3 — build:** 108 → 109 → 1008 → 449 → 96 → 95
**Week 4 — streaming & boss fights:** 94 (warm-up, Binary Tree guide) → 173 → 230 (iterative) → 653 → 1305 → 285 → 776

Good luck with the interviews!

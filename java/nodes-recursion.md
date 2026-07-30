# Nodes & Recursion — Linked Lists and Binary Trees

Java has no pointers, only **references**. In practice that changes very little for these problems — the algorithms are identical to the C++ ones — but three things differ and all three cause bugs: there is no reference-to-a-reference, `==` compares identity (which is exactly what you want here), and there is no manual memory management to worry about.

LeetCode defines these for you; know them cold:

```java
class ListNode {
    int val;
    ListNode next;
    ListNode() {}
    ListNode(int val) { this.val = val; }
    ListNode(int val, ListNode next) { this.val = val; this.next = next; }
}

class TreeNode {
    int val;
    TreeNode left, right;
    TreeNode() {}
    TreeNode(int val) { this.val = val; }
    TreeNode(int val, TreeNode left, TreeNode right) {
        this.val = val; this.left = left; this.right = right;
    }
}
```

---

## References, Not Pointers

```java
node.val               // always a dot — no -> in Java
node == null           // the null test
node == other          // IDENTITY comparison — correct for nodes
node.equals(other)     // NOT overridden on ListNode/TreeNode; same as ==
```

**Use `==` for node comparison.** This is the one place where reference equality is what you actually mean — "is this the same node object", not "does it hold the same value". LC 160 Intersection of Two Linked Lists and LC 236 LCA both depend on it.

### Java Cannot Reassign the Caller's Reference

C++ has `TreeNode*&`. Java passes references **by value**, so:

```java
void insert(TreeNode node, int val) {
    if (node == null) { node = new TreeNode(val); return; }   // does NOTHING to the caller
}
```

The local `node` is rebound; the caller's variable is untouched. Java therefore has exactly one convention for structural modification — **return the new subtree root and reassign**:

```java
TreeNode insert(TreeNode node, int val) {
    if (node == null) return new TreeNode(val);
    if (val < node.val) node.left  = insert(node.left, val);
    else                node.right = insert(node.right, val);
    return node;
}
```

`node.left = insert(node.left, ...)` — the reassignment *is* the algorithm. Forgetting it is why "my insert/delete does nothing".

Mutating a node's *fields* through a reference does affect the caller (`node.val = 5` works). Rebinding the reference itself does not.

---

## `NullPointerException` Discipline

Java throws instead of segfaulting, which is friendlier but still a failed submission.

```java
if (node != null && node.left != null && node.left.val == x)     // short-circuits, safe
```

Standard base cases:

```java
if (head == null) return null;
if (head == null || head.next == null) return head;     // 0 or 1 nodes
if (root == null) return 0;
```

Java 8+ has a compact null-pair handler:

```java
if (a == null || b == null) return a == b;      // both null -> true, one null -> false
```

That one line covers the base case of `isSameTree`, `isSymmetric`, and every structural comparison.

---

## Linked Lists

### The Dummy Head

The highest-value trick in the category — it eliminates every "what if the head itself changes" branch.

```java
ListNode dummy = new ListNode(0);
dummy.next = head;
ListNode prev = dummy;

while (prev.next != null) {
    if (prev.next.val == target) prev.next = prev.next.next;
    else prev = prev.next;
}
return dummy.next;                    // NOT head — head may have been removed
```

Returning `head` instead of `dummy.next` is the standard follow-up bug.

### Core Manipulations

```java
// Insert y after x
y.next = x.next;
x.next = y;

// Delete the node after prev
prev.next = prev.next.next;           // GC reclaims it; no delete needed

// Delete a node given only that node (LC 237)
node.val = node.next.val;
node.next = node.next.next;
```

Order matters: `x.next = y; y.next = x.next;` makes `y` point at itself. Save before you overwrite.

### Reversal

```java
ListNode reverse(ListNode head) {
    ListNode prev = null;
    while (head != null) {
        ListNode next = head.next;    // 1. save
        head.next = prev;             // 2. flip
        prev = head;                  // 3. advance prev
        head = next;                  // 4. advance head
    }
    return prev;                      // prev is the new head
}
```

Recursive form:

```java
ListNode reverse(ListNode head) {
    if (head == null || head.next == null) return head;
    ListNode newHead = reverse(head.next);
    head.next.next = head;
    head.next = null;
    return newHead;
}
```

O(n) stack depth — a 10⁵-node list throws `StackOverflowError`. Java's default stack is smaller than you'd like; prefer the iterative version for anything long.

Reverse a sublist `[left, right]`, 1-indexed (LC 92):

```java
ListNode dummy = new ListNode(0, head);
ListNode prev = dummy;
for (int i = 1; i < left; i++) prev = prev.next;
ListNode cur = prev.next;
for (int i = 0; i < right - left; i++) {          // head-insertion
    ListNode next = cur.next;
    cur.next = next.next;
    next.next = prev.next;
    prev.next = next;
}
return dummy.next;
```

### Slow and Fast Pointers

```java
// Middle — for even length this returns the SECOND middle
ListNode slow = head, fast = head;
while (fast != null && fast.next != null) { slow = slow.next; fast = fast.next.next; }
// For the FIRST middle on even length, start: fast = head.next;

// Nth from the end
ListNode fast = head, slow = head;
for (int i = 0; i < n; i++) fast = fast.next;
while (fast != null) { fast = fast.next; slow = slow.next; }

// Cycle detection + entry (Floyd)
ListNode slow = head, fast = head;
while (fast != null && fast.next != null) {
    slow = slow.next; fast = fast.next.next;
    if (slow == fast) {
        slow = head;
        while (slow != fast) { slow = slow.next; fast = fast.next; }
        return slow;                   // cycle entry
    }
}
return null;
```

`while (fast != null && fast.next != null)` — the order matters; `&&` short-circuits before `fast.next.next` is evaluated.

### Merging

```java
ListNode merge(ListNode a, ListNode b) {
    ListNode dummy = new ListNode(0), tail = dummy;
    while (a != null && b != null) {
        if (a.val <= b.val) { tail.next = a; a = a.next; }
        else                { tail.next = b; b = b.next; }
        tail = tail.next;
    }
    tail.next = (a != null) ? a : b;        // attach the remainder wholesale
    return dummy.next;
}
```

Merge k lists with a heap:

```java
PriorityQueue<ListNode> pq = new PriorityQueue<>(Comparator.comparingInt(n -> n.val));
for (ListNode l : lists) if (l != null) pq.offer(l);
ListNode dummy = new ListNode(0), tail = dummy;
while (!pq.isEmpty()) {
    ListNode n = pq.poll();
    tail.next = n; tail = n;
    if (n.next != null) pq.offer(n.next);
}
tail.next = null;
return dummy.next;
```

`Comparator.comparingInt(n -> n.val)` — without a comparator the heap can't order `ListNode`, since it doesn't implement `Comparable`.

---

## Binary Trees

### Recursive Traversals

```java
void preorder(TreeNode n)  { if (n == null) return; visit(n); preorder(n.left); preorder(n.right); }
void inorder(TreeNode n)   { if (n == null) return; inorder(n.left); visit(n); inorder(n.right); }
void postorder(TreeNode n) { if (n == null) return; postorder(n.left); postorder(n.right); visit(n); }
```

Only the position of `visit` changes. **Inorder on a BST yields sorted output** — that single fact solves LC 98, 230, 501, 538.

Java has no closures over mutable locals, so recursive helpers either take a mutable container or use a field:

```java
// Option 1: an instance field (simplest inside a Solution class)
private int best = Integer.MIN_VALUE;

// Option 2: a one-element array as a mutable box
int[] best = {Integer.MIN_VALUE};
dfs(root, best);

// Option 3: pass and return the accumulator
```

**Lambdas can't capture a mutable local** — `int best = 0; ... best = x;` inside a lambda is a compile error ("must be effectively final"). The `int[1]` box is the standard workaround, and instance fields are cleaner when you're already inside a `Solution` class. Just remember to reset fields between calls (see [gotchas.md](gotchas.md)).

### Iterative Traversals

```java
// Preorder — push right first so left pops first
Deque<TreeNode> st = new ArrayDeque<>();
if (root != null) st.push(root);
while (!st.isEmpty()) {
    TreeNode n = st.pop();
    res.add(n.val);
    if (n.right != null) st.push(n.right);
    if (n.left  != null) st.push(n.left);
}

// Inorder — go left, pop, go right
Deque<TreeNode> st = new ArrayDeque<>();
TreeNode cur = root;
while (cur != null || !st.isEmpty()) {
    while (cur != null) { st.push(cur); cur = cur.left; }
    cur = st.pop();
    res.add(cur.val);
    cur = cur.right;
}

// Postorder — preorder with right-before-left, then reverse
Deque<TreeNode> st = new ArrayDeque<>();
if (root != null) st.push(root);
LinkedList<Integer> res = new LinkedList<>();
while (!st.isEmpty()) {
    TreeNode n = st.pop();
    res.addFirst(n.val);                          // prepend instead of reversing at the end
    if (n.left  != null) st.push(n.left);
    if (n.right != null) st.push(n.right);
}
```

`res.addFirst` on a `LinkedList` avoids the final `Collections.reverse` — one of the few genuinely good uses of `LinkedList`.

**`ArrayDeque` rejects `null`**, so `st.push(node)` throws if you push an unchecked child. Always guard with `if (n.left != null)`.

### Level Order (BFS)

```java
List<List<Integer>> levelOrder(TreeNode root) {
    List<List<Integer>> res = new ArrayList<>();
    if (root == null) return res;
    Deque<TreeNode> q = new ArrayDeque<>();
    q.offer(root);
    while (!q.isEmpty()) {
        int sz = q.size();                        // SNAPSHOT before the inner loop
        List<Integer> level = new ArrayList<>();
        for (int i = 0; i < sz; i++) {
            TreeNode n = q.poll();
            level.add(n.val);
            if (n.left  != null) q.offer(n.left);
            if (n.right != null) q.offer(n.right);
        }
        res.add(level);
    }
    return res;
}
```

`int sz = q.size();` before the loop is mandatory — the queue grows as you enqueue children. Variations: zigzag (`Collections.reverse(level)` on odd rows, or build with `addFirst`), right-side view (`level.get(sz-1)`), bottom-up (`res.add(0, level)` or reverse at the end).

---

## The Four Recursion Signatures

Most tree difficulty is deciding *what the method returns*.

### 1. Aggregate over the subtree

```java
int maxDepth(TreeNode n) {
    if (n == null) return 0;
    return 1 + Math.max(maxDepth(n.left), maxDepth(n.right));
}
```

### 2. Return one thing, record another

The answer may bend *through* a node, but the parent can only extend a straight downward path. Return the extendable value; record the bent one in a field. This is LC 124 Max Path Sum and LC 543 Diameter.

```java
private int best = Integer.MIN_VALUE;

private int gain(TreeNode n) {
    if (n == null) return 0;
    int l = Math.max(0, gain(n.left));       // clamp: a negative branch is never taken
    int r = Math.max(0, gain(n.right));
    best = Math.max(best, n.val + l + r);    // RECORD the bent path through n
    return n.val + Math.max(l, r);           // RETURN what the parent can extend
}
```

More in [tricks/recursion-reframing.md](../tricks/recursion-reframing.md).

### 3. Pass bounds down (BST validation)

```java
boolean valid(TreeNode n, long lo, long hi) {
    if (n == null) return true;
    if (n.val <= lo || n.val >= hi) return false;
    return valid(n.left, lo, n.val) && valid(n.right, n.val, hi);
}
valid(root, Long.MIN_VALUE, Long.MAX_VALUE);
```

Use `long` bounds. With `int` bounds, a node holding `Integer.MIN_VALUE` fails — and LeetCode includes exactly that test.

Comparing only against the immediate parent is the classic wrong answer: it accepts a tree where a deep node violates a distant ancestor's bound.

### 4. Return the (possibly new) subtree root

```java
TreeNode deleteNode(TreeNode root, int key) {
    if (root == null) return null;
    if (key < root.val)      root.left  = deleteNode(root.left, key);
    else if (key > root.val) root.right = deleteNode(root.right, key);
    else {
        if (root.left == null)  return root.right;
        if (root.right == null) return root.left;
        TreeNode succ = root.right;
        while (succ.left != null) succ = succ.left;      // in-order successor
        root.val = succ.val;
        root.right = deleteNode(root.right, succ.val);
    }
    return root;
}
```

Every structural edit uses this shape: recurse, **reassign the child**, return the root.

---

## Tree Recipes

```java
int depth(TreeNode n) { return n == null ? 0 : 1 + Math.max(depth(n.left), depth(n.right)); }

int count(TreeNode n) { return n == null ? 0 : 1 + count(n.left) + count(n.right); }

boolean same(TreeNode a, TreeNode b) {
    if (a == null || b == null) return a == b;
    return a.val == b.val && same(a.left, b.left) && same(a.right, b.right);
}

boolean mirror(TreeNode a, TreeNode b) {
    if (a == null || b == null) return a == b;
    return a.val == b.val && mirror(a.left, b.right) && mirror(a.right, b.left);
}

TreeNode invert(TreeNode n) {
    if (n == null) return null;
    TreeNode t = n.left; n.left = n.right; n.right = t;
    invert(n.left); invert(n.right);
    return n;
}

// LCA, general binary tree — relies on REFERENCE equality
TreeNode lca(TreeNode n, TreeNode p, TreeNode q) {
    if (n == null || n == p || n == q) return n;
    TreeNode l = lca(n.left, p, q);
    TreeNode r = lca(n.right, p, q);
    return (l != null && r != null) ? n : (l != null ? l : r);
}

// LCA in a BST — the split point IS the answer
TreeNode lcaBST(TreeNode n, TreeNode p, TreeNode q) {
    while (n != null) {
        if (p.val < n.val && q.val < n.val) n = n.left;
        else if (p.val > n.val && q.val > n.val) n = n.right;
        else return n;
    }
    return null;
}
```

### Root-to-Leaf Paths — The Copy Rule

```java
List<List<Integer>> res = new ArrayList<>();
List<Integer> path = new ArrayList<>();

void dfs(TreeNode n) {
    if (n == null) return;
    path.add(n.val);
    if (n.left == null && n.right == null) res.add(new ArrayList<>(path));   // COPY
    dfs(n.left);
    dfs(n.right);
    path.remove(path.size() - 1);                                            // undo
}
```

Two rules here, and both are common failure points:

- **`res.add(new ArrayList<>(path))`, never `res.add(path)`.** Without the copy you store a reference to a list that keeps mutating; after backtracking finishes, every row in `res` is empty.
- **A leaf is `n.left == null && n.right == null`**, never just `n == null`. Using `n == null` counts every path twice — once per null child.

---

## Serialization (LC 297)

```java
public String serialize(TreeNode root) {
    StringBuilder sb = new StringBuilder();
    ser(root, sb);
    return sb.toString();
}
private void ser(TreeNode n, StringBuilder sb) {
    if (n == null) { sb.append("#,"); return; }        // explicit null marker
    sb.append(n.val).append(',');
    ser(n.left, sb); ser(n.right, sb);
}

public TreeNode deserialize(String data) {
    Deque<String> q = new ArrayDeque<>(Arrays.asList(data.split(",")));
    return des(q);
}
private TreeNode des(Deque<String> q) {
    String t = q.poll();
    if (t == null || t.equals("#")) return null;
    TreeNode n = new TreeNode(Integer.parseInt(t));
    n.left = des(q); n.right = des(q);
    return n;
}
```

Preorder **with explicit null markers** is uniquely decodable on its own — which is why "construct from preorder + inorder" needs two arrays while this needs one string.

---

## Building Test Trees Locally

```java
// From a LeetCode-style level-order array; null means an absent node
static TreeNode build(Integer[] a) {
    if (a.length == 0 || a[0] == null) return null;
    TreeNode root = new TreeNode(a[0]);
    Deque<TreeNode> q = new ArrayDeque<>();
    q.offer(root);
    int i = 1;
    while (!q.isEmpty() && i < a.length) {
        TreeNode n = q.poll();
        if (i < a.length && a[i] != null) { n.left  = new TreeNode(a[i]); q.offer(n.left); }
        i++;
        if (i < a.length && a[i] != null) { n.right = new TreeNode(a[i]); q.offer(n.right); }
        i++;
    }
    return root;
}

static ListNode buildList(int[] a) {
    ListNode dummy = new ListNode(0), t = dummy;
    for (int x : a) { t.next = new ListNode(x); t = t.next; }
    return dummy.next;
}
```

`Integer[]` rather than `int[]` so that `null` can represent a missing node.

---

## `StackOverflowError`

Java's default thread stack is around 512 KB–1 MB — roughly 10⁴ frames for a typical tree method. A skewed tree or a 10⁵-node linked list will blow it, and LeetCode reports the failure as a runtime error rather than a TLE.

Mitigations, in order of preference:

1. Convert to an iterative traversal with an explicit `Deque`.
2. Shrink each frame — fewer locals, no large objects passed down.
3. Accept it: most LeetCode tree constraints cap at ~10⁴ nodes, where recursion is fine.

Java has no tail-call optimization, so a "tail-recursive" rewrite does not help.

---

## Checklist

- [ ] Null-checked before every field access, with `&&` ordering that short-circuits.
- [ ] Loop condition is `while (fast != null && fast.next != null)`.
- [ ] Structural changes reassign the child: `n.left = helper(n.left)`.
- [ ] Returned `dummy.next`, not `head`.
- [ ] Saved `next` before overwriting it during reversal.
- [ ] Snapshotted `q.size()` before the BFS inner loop.
- [ ] `res.add(new ArrayList<>(path))` — copied, not aliased.
- [ ] Leaf test is `left == null && right == null`, not `n == null`.
- [ ] BST bounds are `long`.
- [ ] Instance fields used as accumulators are reset per call.
- [ ] Recursion depth is safe for the stated constraints.

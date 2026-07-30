# Pointers & Node Structures — Linked Lists and Binary Trees

The pattern guides ([binary-tree-patterns.md](../binary-tree-patterns.md), [bst-patterns.md](../bst-patterns.md)) cover *which traversal to use*. This page covers the C++ mechanics: pointer syntax, null handling, the recursion signatures that make problems easy, and the traversal templates in full.

Neither `ListNode` nor `TreeNode` is part of the STL — LeetCode defines them for you and the definition sits commented out at the top of the editor. Know it by heart:

```cpp
struct ListNode {
    int val;
    ListNode *next;
    ListNode() : val(0), next(nullptr) {}
    ListNode(int x) : val(x), next(nullptr) {}
    ListNode(int x, ListNode *next) : val(x), next(next) {}
};

struct TreeNode {
    int val;
    TreeNode *left, *right;
    TreeNode() : val(0), left(nullptr), right(nullptr) {}
    TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
    TreeNode(int x, TreeNode *l, TreeNode *r) : val(x), left(l), right(r) {}
};
```

---

## Pointer Syntax You Must Not Fumble

| Expression | Meaning |
|---|---|
| `node->val` | member through a pointer — shorthand for `(*node).val` |
| `node.val` | member of an **object** — compile error on a pointer |
| `*node` | dereference to the object |
| `&node` | address of the pointer variable itself |
| `TreeNode*& ref` | **reference to a pointer** — lets a function reassign the caller's pointer |
| `nullptr` | the null pointer. Prefer over `NULL` / `0` |

```cpp
if (node)            // idiomatic "node is not null"
if (node != nullptr) // same thing, more verbose
if (!node) return;   // the standard recursion base case
```

**Null-check before every dereference.** `root->left->val` crashes when `root->left` is null. Chain the checks:

```cpp
if (root && root->left && root->left->val == x) { ... }
```

Short-circuit evaluation makes this safe — `&&` stops at the first false.

### `TreeNode*&` — Reference to a Pointer

This is how you modify the tree structure from inside a helper without returning anything:

```cpp
void insert(TreeNode*& node, int val) {
    if (!node) { node = new TreeNode(val); return; }     // writes into the CALLER's pointer
    if (val < node->val) insert(node->left, val);
    else insert(node->right, val);
}
```

Without the `&`, `node = new TreeNode(val)` only changes the local copy and the tree is never modified. The alternative, and what LeetCode's signatures usually push you toward, is the **return-the-new-root** convention:

```cpp
TreeNode* insert(TreeNode* node, int val) {
    if (!node) return new TreeNode(val);
    if (val < node->val) node->left  = insert(node->left, val);
    else                 node->right = insert(node->right, val);
    return node;                                          // caller reattaches
}
```

`node->left = insert(node->left, ...)` — the reassignment is the whole point. Forgetting it is why "my insert/delete does nothing" happens.

---

## Linked Lists

### The Dummy Head

The single highest-value trick in linked-list problems. It removes every "what if the head itself changes" special case.

```cpp
ListNode dummy(0);              // stack-allocated, no leak, no delete needed
dummy.next = head;
ListNode* prev = &dummy;

while (prev->next) {
    if (prev->next->val == target) prev->next = prev->next->next;   // delete
    else prev = prev->next;
}
return dummy.next;              // NOT head — head may have been removed
```

`ListNode dummy(0);` on the stack is better than `new ListNode(0)` — no allocation, no cleanup. Note you then need `&dummy` and `dummy.next` (dot, not arrow).

### The Core Manipulations

```cpp
// Insert y after x
y->next = x->next;
x->next = y;

// Delete the node after prev
prev->next = prev->next->next;      // the removed node leaks unless you delete it

// Delete a node given ONLY that node (LC 237) — copy the successor into it
node->val = node->next->val;
node->next = node->next->next;
```

**Order matters.** Writing `x->next = y; y->next = x->next;` makes `y` point to itself. Always save what you're about to overwrite.

### Reversal — Memorize This

```cpp
ListNode* reverse(ListNode* head) {
    ListNode* prev = nullptr;
    while (head) {
        ListNode* nxt = head->next;   // 1. save
        head->next = prev;            // 2. flip
        prev = head;                  // 3. advance prev
        head = nxt;                   // 4. advance head
    }
    return prev;                      // prev is the new head; head is now null
}
```

Recursive form, for when the interviewer asks:

```cpp
ListNode* reverse(ListNode* head) {
    if (!head || !head->next) return head;
    ListNode* newHead = reverse(head->next);
    head->next->next = head;          // the node ahead now points back
    head->next = nullptr;             // sever the old forward link
    return newHead;                   // the same new head propagates all the way up
}
```

O(n) stack depth — it will overflow on a 10⁵-node list.

Reverse a sublist `[l, r]` (1-indexed, LC 92):

```cpp
ListNode dummy(0); dummy.next = head;
ListNode* prev = &dummy;
for (int i = 1; i < l; i++) prev = prev->next;
ListNode* cur = prev->next;
for (int i = 0; i < r - l; i++) {          // head-insertion: pull each successor to the front
    ListNode* nxt = cur->next;
    cur->next = nxt->next;
    nxt->next = prev->next;
    prev->next = nxt;
}
return dummy.next;
```

### Two Pointers — Slow and Fast

```cpp
// Middle. For even length this returns the SECOND middle.
ListNode *slow = head, *fast = head;
while (fast && fast->next) { slow = slow->next; fast = fast->next->next; }
// For the FIRST middle on even length, start: fast = head->next;

// Nth from the end
ListNode *fast = head, *slow = head;
for (int i = 0; i < n; i++) fast = fast->next;      // gap of n
while (fast) { fast = fast->next; slow = slow->next; }
// slow is now the nth node from the end

// Cycle detection + entry point (Floyd)
ListNode *slow = head, *fast = head;
while (fast && fast->next) {
    slow = slow->next; fast = fast->next->next;
    if (slow == fast) {                       // meeting point
        slow = head;
        while (slow != fast) { slow = slow->next; fast = fast->next; }
        return slow;                          // cycle entry
    }
}
return nullptr;
```

`while (fast && fast->next)` is the correct loop condition — checking only `fast` crashes on `fast->next->next`, and the order matters because `&&` short-circuits.

### Merging

```cpp
ListNode* merge(ListNode* a, ListNode* b) {
    ListNode dummy(0);
    ListNode* tail = &dummy;
    while (a && b) {
        if (a->val <= b->val) { tail->next = a; a = a->next; }
        else                  { tail->next = b; b = b->next; }
        tail = tail->next;
    }
    tail->next = a ? a : b;              // attach the remainder wholesale
    return dummy.next;
}
```

`tail->next = a ? a : b` handles the leftover tail in one line — no second loop.

### Merge k Lists with a Heap

```cpp
struct Cmp { bool operator()(ListNode* a, ListNode* b) const { return a->val > b->val; } };
priority_queue<ListNode*, vector<ListNode*>, Cmp> pq;    // min-heap by val
for (ListNode* l : lists) if (l) pq.push(l);
ListNode dummy(0); ListNode* tail = &dummy;
while (!pq.empty()) {
    ListNode* n = pq.top(); pq.pop();
    tail->next = n; tail = n;
    if (n->next) pq.push(n->next);
}
tail->next = nullptr;
return dummy.next;
```

The comparator takes `ListNode*` but compares `->val` — a raw pointer comparison would order by memory address.

### Converting To and From a Vector

Legitimate in an interview if you say so out loud, and it makes "sort a linked list" or "is it a palindrome" trivial:

```cpp
vector<int> v;
for (ListNode* p = head; p; p = p->next) v.push_back(p->val);
// ...manipulate v...
ListNode* p = head;
for (int x : v) { p->val = x; p = p->next; }     // write values back, structure untouched
```

O(n) extra space — the interviewer may then ask for the O(1)-space version, which is what the reversal and slow/fast tricks above are for.

---

## Binary Trees

### Recursive Traversals

```cpp
void preorder(TreeNode* n)  { if (!n) return; visit(n); preorder(n->left); preorder(n->right); }
void inorder(TreeNode* n)   { if (!n) return; inorder(n->left); visit(n); inorder(n->right); }
void postorder(TreeNode* n) { if (!n) return; postorder(n->left); postorder(n->right); visit(n); }
```

The only difference is *where* `visit` sits. **Inorder on a BST produces sorted output** — that fact solves a whole class of problems (LC 98, 230, 501, 538).

As a lambda, so it lives inside the LeetCode method:

```cpp
vector<int> res;
function<void(TreeNode*)> dfs = [&](TreeNode* n) {
    if (!n) return;
    dfs(n->left);
    res.push_back(n->val);
    dfs(n->right);
};
dfs(root);
```

### Iterative Traversals

```cpp
// Preorder — push right first so left pops first
stack<TreeNode*> st;
if (root) st.push(root);
while (!st.empty()) {
    TreeNode* n = st.top(); st.pop();
    res.push_back(n->val);
    if (n->right) st.push(n->right);
    if (n->left)  st.push(n->left);
}

// Inorder — go left as far as possible, then pop and go right
stack<TreeNode*> st;
TreeNode* cur = root;
while (cur || !st.empty()) {
    while (cur) { st.push(cur); cur = cur->left; }
    cur = st.top(); st.pop();
    res.push_back(cur->val);
    cur = cur->right;
}

// Postorder — do preorder with right-before-left, then reverse
stack<TreeNode*> st;
if (root) st.push(root);
while (!st.empty()) {
    TreeNode* n = st.top(); st.pop();
    res.push_back(n->val);
    if (n->left)  st.push(n->left);
    if (n->right) st.push(n->right);
}
reverse(res.begin(), res.end());
```

The reversed-preorder postorder is the one to remember — the "real" iterative postorder with a `lastVisited` pointer is longer and easy to botch under pressure.

### Morris Traversal — O(1) Space

Asked for when the interviewer says "now without recursion *and* without a stack":

```cpp
TreeNode* cur = root;
while (cur) {
    if (!cur->left) { res.push_back(cur->val); cur = cur->right; }
    else {
        TreeNode* pred = cur->left;
        while (pred->right && pred->right != cur) pred = pred->right;
        if (!pred->right) { pred->right = cur; cur = cur->left; }   // thread
        else { pred->right = nullptr; res.push_back(cur->val); cur = cur->right; }  // unthread
    }
}
```

It temporarily mutates the tree and restores it. Know that it exists; you'll rarely need to write it.

### Level Order (BFS)

```cpp
vector<vector<int>> levelOrder(TreeNode* root) {
    vector<vector<int>> res;
    if (!root) return res;
    queue<TreeNode*> q; q.push(root);
    while (!q.empty()) {
        int sz = q.size();                        // SNAPSHOT before the inner loop
        vector<int> level;
        for (int i = 0; i < sz; i++) {
            TreeNode* n = q.front(); q.pop();
            level.push_back(n->val);
            if (n->left)  q.push(n->left);
            if (n->right) q.push(n->right);
        }
        res.push_back(level);
    }
    return res;
}
```

`int sz = q.size();` before the loop is mandatory — the queue grows as you push children. Variations: zigzag (`if (level % 2) reverse(level...)`), right side view (`level.back()`), max per level, bottom-up (`reverse(res...)`).

---

## Recursion Signatures — The Real Skill

Most tree difficulty is choosing *what the recursive function returns*. Four shapes cover almost everything.

### 1. Return an aggregate of the subtree

```cpp
int maxDepth(TreeNode* n) {
    if (!n) return 0;
    return 1 + max(maxDepth(n->left), maxDepth(n->right));
}
```

### 2. Return one thing, record another (the "bend" pattern)

The answer may pass *through* a node, but the parent can only use a path that *extends downward*. Return the extendable value; record the bent one in a captured variable. This is LC 124 Max Path Sum and LC 543 Diameter.

```cpp
int best = INT_MIN;
function<int(TreeNode*)> gain = [&](TreeNode* n) -> int {
    if (!n) return 0;
    int l = max(0, gain(n->left));            // clamp at 0: a negative branch is never taken
    int r = max(0, gain(n->right));
    best = max(best, n->val + l + r);         // RECORD the bent path through n
    return n->val + max(l, r);                // RETURN the straight path the parent can extend
};
```

More on this in [tricks/recursion-reframing.md](../tricks/recursion-reframing.md).

### 3. Pass bounds down (BST validation)

```cpp
bool valid(TreeNode* n, long long lo, long long hi) {
    if (!n) return true;
    if (n->val <= lo || n->val >= hi) return false;
    return valid(n->left, lo, n->val) && valid(n->right, n->val, hi);
}
valid(root, LLONG_MIN, LLONG_MAX);
```

Use `long long` bounds — a node with value `INT_MIN` or `INT_MAX` breaks the `int` version, and that's exactly the test case LeetCode includes.

Comparing only against the immediate parent is the classic wrong answer: it accepts a tree where a deep left-subtree node exceeds a distant ancestor.

### 4. Return the (possibly new) subtree root

```cpp
TreeNode* deleteNode(TreeNode* root, int key) {
    if (!root) return nullptr;
    if (key < root->val)      root->left  = deleteNode(root->left, key);
    else if (key > root->val) root->right = deleteNode(root->right, key);
    else {
        if (!root->left)  return root->right;
        if (!root->right) return root->left;
        TreeNode* succ = root->right;
        while (succ->left) succ = succ->left;      // in-order successor
        root->val = succ->val;
        root->right = deleteNode(root->right, succ->val);
    }
    return root;
}
```

Every structural modification uses this shape: recurse, **reassign the child**, return the root.

---

## Tree Recipes

```cpp
// Height / depth
int depth(TreeNode* n) { return n ? 1 + max(depth(n->left), depth(n->right)) : 0; }

// Count nodes
int count(TreeNode* n) { return n ? 1 + count(n->left) + count(n->right) : 0; }

// Same tree
bool same(TreeNode* a, TreeNode* b) {
    if (!a || !b) return a == b;                     // both null -> true, one null -> false
    return a->val == b->val && same(a->left, b->left) && same(a->right, b->right);
}

// Symmetric — compare the mirrored children
bool mirror(TreeNode* a, TreeNode* b) {
    if (!a || !b) return a == b;
    return a->val == b->val && mirror(a->left, b->right) && mirror(a->right, b->left);
}

// Invert
TreeNode* invert(TreeNode* n) {
    if (!n) return nullptr;
    swap(n->left, n->right);
    invert(n->left); invert(n->right);
    return n;
}

// Lowest common ancestor (general binary tree)
TreeNode* lca(TreeNode* n, TreeNode* p, TreeNode* q) {
    if (!n || n == p || n == q) return n;
    TreeNode* l = lca(n->left, p, q);
    TreeNode* r = lca(n->right, p, q);
    return (l && r) ? n : (l ? l : r);               // both sides found -> n is the LCA
}

// LCA in a BST — no full search needed
TreeNode* lcaBST(TreeNode* n, TreeNode* p, TreeNode* q) {
    while (n) {
        if (p->val < n->val && q->val < n->val) n = n->left;
        else if (p->val > n->val && q->val > n->val) n = n->right;
        else return n;                               // the split point IS the LCA
    }
    return nullptr;
}

// Kth smallest in a BST — inorder with a counter
int k, ans;
function<void(TreeNode*)> in = [&](TreeNode* n) {
    if (!n || k == 0) return;
    in(n->left);
    if (--k == 0) { ans = n->val; return; }
    in(n->right);
};

// Root-to-leaf paths — backtracking on a shared vector
vector<int> path;
function<void(TreeNode*)> dfs = [&](TreeNode* n) {
    if (!n) return;
    path.push_back(n->val);
    if (!n->left && !n->right) record(path);         // leaf = BOTH children null
    dfs(n->left); dfs(n->right);
    path.pop_back();                                 // undo before returning
};
```

`if (!a || !b) return a == b;` is the compact null-pair handler — worth internalizing. And a leaf is `!n->left && !n->right`, never just `!n`; confusing the two double-counts paths in every path-sum problem.

---

## Serialization (LC 297)

```cpp
string serialize(TreeNode* root) {
    string s;
    function<void(TreeNode*)> go = [&](TreeNode* n) {
        if (!n) { s += "#,"; return; }               // explicit null marker
        s += to_string(n->val) + ",";
        go(n->left); go(n->right);
    };
    go(root);
    return s;
}

TreeNode* deserialize(const string& data) {
    stringstream ss(data);
    string tok;
    function<TreeNode*()> go = [&]() -> TreeNode* {
        if (!getline(ss, tok, ',') || tok == "#") return nullptr;
        TreeNode* n = new TreeNode(stoi(tok));
        n->left = go(); n->right = go();
        return n;
    };
    return go();
}
```

Preorder **with explicit null markers** is uniquely decodable on its own. Preorder without markers is not — that's why "construct from preorder + inorder" needs two arrays.

---

## Building Trees for Local Testing

LeetCode hands you a built tree, but locally you need one:

```cpp
// From a LeetCode-style level-order array; "" or INT_MIN means null
TreeNode* build(const vector<int>& a) {           // INT_MIN as the null sentinel
    if (a.empty() || a[0] == INT_MIN) return nullptr;
    TreeNode* root = new TreeNode(a[0]);
    queue<TreeNode*> q; q.push(root);
    size_t i = 1;
    while (!q.empty() && i < a.size()) {
        TreeNode* n = q.front(); q.pop();
        if (i < a.size() && a[i] != INT_MIN) { n->left  = new TreeNode(a[i]); q.push(n->left); }
        i++;
        if (i < a.size() && a[i] != INT_MIN) { n->right = new TreeNode(a[i]); q.push(n->right); }
        i++;
    }
    return root;
}

ListNode* buildList(const vector<int>& a) {
    ListNode dummy(0); ListNode* t = &dummy;
    for (int x : a) { t->next = new ListNode(x); t = t->next; }
    return dummy.next;
}
```

---

## Memory: `new`, `delete`, and Why You Can Ignore It

```cpp
TreeNode* n = new TreeNode(5);     // heap allocated
delete n;                          // you own it
```

On LeetCode, **do not `delete` nodes the judge gave you** — it owns them and will free them itself; a double free crashes. Nodes you allocate leak, and that is fine within a submission.

In a real interview, mentioning "in production I'd use `unique_ptr` for ownership" earns a point:

```cpp
struct Node { int val; unique_ptr<Node> left, right; };
```

Smart pointers are awkward for tree algorithms (you can't hold two owning references to a node), so raw pointers remain the interview norm — just say why.

---

## Gotcha Checklist

- [ ] Null-checked before every `->`, including chains like `n->left->val`.
- [ ] Loop condition is `while (fast && fast->next)`, in that order.
- [ ] Structural changes reassign the child: `n->left = helper(n->left)`.
- [ ] Returned `dummy.next`, not `head`.
- [ ] Saved `next` before overwriting it during reversal.
- [ ] Snapshotted `q.size()` before the BFS inner loop.
- [ ] BST bounds are `long long`, not `int`.
- [ ] Leaf test is `!n->left && !n->right`, not `!n`.
- [ ] Backtracking `path.pop_back()` happens on every return route.
- [ ] Recursion depth is safe — a 10⁵-node skewed tree or list overflows the stack; go iterative.

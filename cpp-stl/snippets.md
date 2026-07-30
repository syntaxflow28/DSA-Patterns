# Snippets — Copy-Paste Ready

Boilerplate you'd otherwise retype every session. Everything here is self-contained.

---

## Template Header

```cpp
#include <bits/stdc++.h>          // LeetCode & Codeforces only — GCC/Clang specific
using namespace std;
using ll = long long;
using pii = pair<int,int>;
using pll = pair<ll,ll>;
#define all(x) (x).begin(), (x).end()
#define rall(x) (x).rbegin(), (x).rend()
```

LeetCode already includes the headers, so you can just write the class body.

Fast I/O for Codeforces (irrelevant on LeetCode):

```cpp
ios_base::sync_with_stdio(false);
cin.tie(nullptr);
```

---

## Grid Traversal

```cpp
// 4-directional
const int dr[] = {-1, 1, 0, 0};
const int dc[] = { 0, 0,-1, 1};

// 8-directional
const int dr8[] = {-1,-1,-1, 0, 0, 1, 1, 1};
const int dc8[] = {-1, 0, 1,-1, 1,-1, 0, 1};

// Compact pair form
const vector<pair<int,int>> DIRS = {{-1,0},{1,0},{0,-1},{0,1}};

// Bounds check as a lambda
auto inb = [&](int r, int c){ return r >= 0 && r < R && c >= 0 && c < C; };

for (auto [dr, dc] : DIRS) {
    int nr = r + dr, nc = c + dc;
    if (!inb(nr, nc) || vis[nr][nc]) continue;
    ...
}
```

The `{-1,0},{1,0},{0,-1},{0,1}` order is up/down/left/right. Knight moves:

```cpp
const int kr[] = {-2,-2,-1,-1,1,1,2,2};
const int kc[] = {-1,1,-2,2,-2,2,-1,1};
```

---

## Grid DFS / BFS

```cpp
// Recursive flood fill
function<void(int,int)> dfs = [&](int r, int c) {
    if (r < 0 || r >= R || c < 0 || c >= C) return;
    if (grid[r][c] != '1') return;
    grid[r][c] = '0';                          // mark visited in place
    for (int d = 0; d < 4; d++) dfs(r + dr[d], c + dc[d]);
};

// Multi-source BFS (rotting oranges, walls & gates, 01-matrix)
queue<pair<int,int>> q;
vector<vector<int>> dist(R, vector<int>(C, -1));
for (int r = 0; r < R; r++)
    for (int c = 0; c < C; c++)
        if (isSource(r, c)) { dist[r][c] = 0; q.push({r, c}); }
while (!q.empty()) {
    auto [r, c] = q.front(); q.pop();
    for (int d = 0; d < 4; d++) {
        int nr = r + dr[d], nc = c + dc[d];
        if (nr<0||nr>=R||nc<0||nc>=C || dist[nr][nc] != -1) continue;
        dist[nr][nc] = dist[r][c] + 1;
        q.push({nr, nc});
    }
}
```

Seeding *all* sources before the loop starts is what makes multi-source BFS correct — every cell gets its distance to the nearest source in one pass.

---

## Disjoint Set Union (Union-Find)

```cpp
struct DSU {
    vector<int> p, sz;
    int comps;
    DSU(int n) : p(n), sz(n, 1), comps(n) { iota(p.begin(), p.end(), 0); }
    int find(int x) { return p[x] == x ? x : p[x] = find(p[x]); }   // path compression
    bool unite(int a, int b) {
        a = find(a); b = find(b);
        if (a == b) return false;
        if (sz[a] < sz[b]) swap(a, b);                              // union by size
        p[b] = a; sz[a] += sz[b]; comps--;
        return true;
    }
    bool same(int a, int b) { return find(a) == find(b); }
    int size(int x) { return sz[find(x)]; }
};
```

`unite` returns `false` when the two are already connected — that's your cycle detector (LC 684) and Kruskal's edge filter.

---

## Fenwick Tree (BIT) — Prefix Sums with Updates

```cpp
struct BIT {
    int n; vector<long long> t;
    BIT(int n) : n(n), t(n + 1, 0) {}
    void add(int i, long long v) { for (++i; i <= n; i += i & -i) t[i] += v; }
    long long sum(int i) { long long s = 0; for (++i; i > 0; i -= i & -i) s += t[i]; return s; }
    long long range(int l, int r) { return sum(r) - (l ? sum(l - 1) : 0); }
};
```

0-indexed externally, 1-indexed internally. O(log n) per op. Pair it with coordinate compression for "count smaller after self" (LC 315).

---

## Segment Tree (Range Query + Point Update)

```cpp
struct SegTree {
    int n; vector<long long> t;
    SegTree(const vector<int>& a) : n(a.size()), t(2 * a.size()) {
        for (int i = 0; i < n; i++) t[n + i] = a[i];
        for (int i = n - 1; i > 0; i--) t[i] = t[2*i] + t[2*i+1];
    }
    void update(int i, long long v) {
        for (t[i += n] = v; i > 1; i >>= 1) t[i>>1] = t[i] + t[i^1];
    }
    long long query(int l, int r) {                 // [l, r)
        long long res = 0;
        for (l += n, r += n; l < r; l >>= 1, r >>= 1) {
            if (l & 1) res += t[l++];
            if (r & 1) res += t[--r];
        }
        return res;
    }
};
```

The iterative bottom-up form — shorter and faster than the recursive one. Swap `+` for `min`/`max` (and the identity `0` for `INF`/`-INF`) for other queries.

---

## Trie

```cpp
struct Trie {
    struct Node { Node* ch[26] = {}; bool end = false; };
    Node* root = new Node();
    void insert(const string& w) {
        Node* cur = root;
        for (char c : w) {
            int i = c - 'a';
            if (!cur->ch[i]) cur->ch[i] = new Node();
            cur = cur->ch[i];
        }
        cur->end = true;
    }
    Node* walk(const string& w) {
        Node* cur = root;
        for (char c : w) { cur = cur->ch[c - 'a']; if (!cur) return nullptr; }
        return cur;
    }
    bool search(const string& w) { Node* n = walk(w); return n && n->end; }
    bool startsWith(const string& p) { return walk(p) != nullptr; }
};
```

Array-based variant when you want no pointers:

```cpp
int nxt[MAXN][26] = {}; bool term[MAXN] = {}; int cnt = 1;   // node 0 is the root
```

---

## Custom Hash for `pair` / `unordered_map`

Collision-resistant and anti-hack (`splitmix64` + a time-based seed):

```cpp
struct Hash {
    static uint64_t splitmix64(uint64_t x) {
        x += 0x9e3779b97f4a7c15ULL;
        x = (x ^ (x >> 30)) * 0xbf58476d1ce4e5b9ULL;
        x = (x ^ (x >> 27)) * 0x94d049bb133111ebULL;
        return x ^ (x >> 31);
    }
    size_t operator()(uint64_t x) const {
        static const uint64_t SEED =
            chrono::steady_clock::now().time_since_epoch().count();
        return splitmix64(x + SEED);
    }
    size_t operator()(const pair<int,int>& p) const {
        return (*this)((uint64_t)p.first << 32 | (uint32_t)p.second);
    }
};
unordered_map<pair<int,int>, int, Hash> m;
unordered_map<long long, int, Hash> fast;
```

Simpler alternative that avoids the whole problem — encode the key:

```cpp
auto key = [&](int r, int c){ return (long long)r * C + c; };
unordered_map<long long, int> m;
```

---

## Linked List Helpers

Full treatment — pointer syntax, sublist reversal, merging, null-handling rules — in [pointers-nodes.md](pointers-nodes.md).

```cpp
struct ListNode { int val; ListNode* next; ListNode(int x) : val(x), next(nullptr) {} };

// Dummy head — removes every "what if it's the first node" special case
ListNode dummy(0); dummy.next = head;
ListNode* prev = &dummy;
// ...
return dummy.next;

// Reverse
ListNode* reverse(ListNode* head) {
    ListNode* prev = nullptr;
    while (head) { ListNode* nxt = head->next; head->next = prev; prev = head; head = nxt; }
    return prev;
}

// Middle (slow/fast) — for even length this returns the SECOND middle
ListNode* mid(ListNode* head) {
    ListNode *slow = head, *fast = head;
    while (fast && fast->next) { slow = slow->next; fast = fast->next->next; }
    return slow;
}

// Cycle detection + entry point (Floyd)
ListNode *slow = head, *fast = head;
while (fast && fast->next) {
    slow = slow->next; fast = fast->next->next;
    if (slow == fast) {
        slow = head;
        while (slow != fast) { slow = slow->next; fast = fast->next; }
        return slow;                       // cycle entry
    }
}
return nullptr;
```

For the *first* middle on even lengths, start `fast = head->next`.

---

## Binary Tree Helpers

All four traversals, Morris, LCA, serialization, and the recursion-signature guide are in [pointers-nodes.md](pointers-nodes.md).

```cpp
struct TreeNode { int val; TreeNode *left, *right; TreeNode(int x) : val(x), left(nullptr), right(nullptr) {} };

// Iterative inorder
vector<int> inorder(TreeNode* root) {
    vector<int> res; stack<TreeNode*> st; TreeNode* cur = root;
    while (cur || !st.empty()) {
        while (cur) { st.push(cur); cur = cur->left; }
        cur = st.top(); st.pop();
        res.push_back(cur->val);
        cur = cur->right;
    }
    return res;
}

// Level order
vector<vector<int>> levelOrder(TreeNode* root) {
    vector<vector<int>> res;
    if (!root) return res;
    queue<TreeNode*> q; q.push(root);
    while (!q.empty()) {
        int sz = q.size(); vector<int> level;
        for (int i = 0; i < sz; i++) {
            TreeNode* n = q.front(); q.pop();
            level.push_back(n->val);
            if (n->left) q.push(n->left);
            if (n->right) q.push(n->right);
        }
        res.push_back(level);
    }
    return res;
}
```

---

## Backtracking Skeleton

```cpp
vector<vector<int>> res;
vector<int> path;
function<void(int)> bt = [&](int start) {
    res.push_back(path);                          // or: if (path.size() == k) { ... return; }
    for (int i = start; i < n; i++) {
        if (i > start && nums[i] == nums[i-1]) continue;   // skip duplicates (sorted input)
        path.push_back(nums[i]);
        bt(i + 1);                                // i for reuse-allowed, i+1 for use-once
        path.pop_back();                          // undo
    }
};
sort(nums.begin(), nums.end());
bt(0);
```

`path` is captured by reference and mutated in place — passing it by value is the classic TLE.

---

## Prefix Sums

```cpp
// 1D
vector<long long> pre(n + 1, 0);
for (int i = 0; i < n; i++) pre[i + 1] = pre[i] + a[i];
long long sum = pre[r + 1] - pre[l];                       // a[l..r]

// 2D
vector<vector<long long>> pre(R + 1, vector<long long>(C + 1, 0));
for (int i = 0; i < R; i++)
    for (int j = 0; j < C; j++)
        pre[i+1][j+1] = pre[i][j+1] + pre[i+1][j] - pre[i][j] + g[i][j];
long long sum = pre[r2+1][c2+1] - pre[r1][c2+1] - pre[r2+1][c1] + pre[r1][c1];

// Difference array — O(1) range update, O(n) final build
vector<long long> diff(n + 1, 0);
diff[l] += v; diff[r + 1] -= v;
for (int i = 1; i <= n; i++) diff[i] += diff[i-1];
```

The `n + 1` sizing and 1-indexing removes the `i == 0` special case from every one of these.

---

## Debug Printing

```cpp
template<class T> void pr(const vector<T>& v) {
    for (auto& x : v) cerr << x << ' ';
    cerr << '\n';
}
template<class T> void pr(const vector<vector<T>>& g) {
    for (auto& row : g) pr(row);
    cerr << '\n';
}
#define dbg(x) cerr << #x << " = " << (x) << '\n'
```

Use `cerr`, not `cout` — LeetCode judges `cout` output in some interactive problems, and `cerr` never affects the verdict.

---

## Timing (Local Benchmarking)

```cpp
auto t0 = chrono::high_resolution_clock::now();
solve();
auto t1 = chrono::high_resolution_clock::now();
cerr << chrono::duration_cast<chrono::milliseconds>(t1 - t0).count() << " ms\n";
```

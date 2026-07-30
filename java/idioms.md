# Idioms — Copy-Paste Ready

Reusable Java implementations of the structures LeetCode expects you to write from scratch.

---

## Solution Skeleton

```java
class Solution {
    private static final int MOD = 1_000_000_007;
    private static final int INF = Integer.MAX_VALUE / 2;

    public int solve(int[] nums) {
        int n = nums.length;
        // ...
        return 0;
    }
}
```

Keep helper state **local or reset per call** — LeetCode may reuse the `Solution` instance across test cases, and a stale field produces a wrong answer that only reproduces in the full submission.

---

## Grid Traversal

```java
private static final int[] DR = {-1, 1, 0, 0};
private static final int[] DC = { 0, 0,-1, 1};

private static final int[][] DIRS = {{-1,0},{1,0},{0,-1},{0,1}};
private static final int[][] DIRS8 = {{-1,-1},{-1,0},{-1,1},{0,-1},{0,1},{1,-1},{1,0},{1,1}};

for (int[] d : DIRS) {
    int nr = r + d[0], nc = c + d[1];
    if (nr < 0 || nr >= R || nc < 0 || nc >= C) continue;
    if (vis[nr][nc] || grid[nr][nc] == '0') continue;
    // ...
}
```

The parallel-array form (`DR`/`DC`) avoids allocating the inner `int[]` and is marginally faster in tight loops:

```java
for (int d = 0; d < 4; d++) {
    int nr = r + DR[d], nc = c + DC[d];
    ...
}
```

Knight moves:

```java
private static final int[][] KNIGHT = {{-2,-1},{-2,1},{-1,-2},{-1,2},{1,-2},{1,2},{2,-1},{2,1}};
```

---

## Grid DFS / BFS

```java
// Recursive flood fill — mutate the grid as the visited marker
private void dfs(char[][] g, int r, int c) {
    if (r < 0 || r >= g.length || c < 0 || c >= g[0].length) return;
    if (g[r][c] != '1') return;
    g[r][c] = '0';
    for (int d = 0; d < 4; d++) dfs(g, r + DR[d], c + DC[d]);
}

// Multi-source BFS — rotting oranges, walls & gates, 01-matrix
int[][] dist = new int[R][C];
for (int[] row : dist) Arrays.fill(row, -1);
Deque<int[]> q = new ArrayDeque<>();
for (int r = 0; r < R; r++)
    for (int c = 0; c < C; c++)
        if (isSource(r, c)) { dist[r][c] = 0; q.offer(new int[]{r, c}); }
while (!q.isEmpty()) {
    int[] cur = q.poll();
    int r = cur[0], c = cur[1];
    for (int d = 0; d < 4; d++) {
        int nr = r + DR[d], nc = c + DC[d];
        if (nr < 0 || nr >= R || nc < 0 || nc >= C || dist[nr][nc] != -1) continue;
        dist[nr][nc] = dist[r][c] + 1;
        q.offer(new int[]{nr, nc});
    }
}
```

Seeding **all** sources before the loop is what makes multi-source BFS correct in a single pass.

Encoding a cell into one int avoids the `int[]` allocation per node:

```java
q.offer(r * C + c);
int cell = q.poll(), r = cell / C, c = cell % C;
```

---

## Disjoint Set Union (Union-Find)

```java
class DSU {
    private final int[] parent, size;
    int components;

    DSU(int n) {
        parent = new int[n];
        size = new int[n];
        components = n;
        for (int i = 0; i < n; i++) { parent[i] = i; size[i] = 1; }
    }

    int find(int x) {
        while (parent[x] != x) { parent[x] = parent[parent[x]]; x = parent[x]; }  // path halving
        return x;
    }

    boolean union(int a, int b) {
        a = find(a); b = find(b);
        if (a == b) return false;
        if (size[a] < size[b]) { int t = a; a = b; b = t; }
        parent[b] = a;
        size[a] += size[b];
        components--;
        return true;
    }

    boolean connected(int a, int b) { return find(a) == find(b); }
    int size(int x) { return size[find(x)]; }
}
```

`find` is iterative with path halving — no recursion, so no `StackOverflowError` on a long chain. `union` returns `false` when the pair is already connected: that's your cycle detector (LC 684) and Kruskal's edge filter.

For grid problems, map `(r, c)` to `r * C + c`.

---

## Trie

```java
class Trie {
    private final Trie[] children = new Trie[26];
    private boolean isWord;

    void insert(String word) {
        Trie cur = this;
        for (char c : word.toCharArray()) {
            int i = c - 'a';
            if (cur.children[i] == null) cur.children[i] = new Trie();
            cur = cur.children[i];
        }
        cur.isWord = true;
    }

    private Trie walk(String s) {
        Trie cur = this;
        for (char c : s.toCharArray()) {
            cur = cur.children[c - 'a'];
            if (cur == null) return null;
        }
        return cur;
    }

    boolean search(String word) { Trie n = walk(word); return n != null && n.isWord; }
    boolean startsWith(String prefix) { return walk(prefix) != null; }
}
```

For word-search-on-a-grid problems (LC 212), store the whole word at the terminal node instead of a boolean — it saves rebuilding the string during DFS:

```java
String word;                 // non-null only at a terminal node
```

Array-backed variant when allocation matters:

```java
int[][] next = new int[maxNodes][26];
boolean[] term = new boolean[maxNodes];
int cnt = 1;                 // node 0 is the root
```

---

## Fenwick Tree (BIT)

```java
class BIT {
    private final long[] tree;
    private final int n;

    BIT(int n) { this.n = n; tree = new long[n + 1]; }

    void add(int i, long v) { for (i++; i <= n; i += i & -i) tree[i] += v; }
    long sum(int i) { long s = 0; for (i++; i > 0; i -= i & -i) s += tree[i]; return s; }
    long range(int l, int r) { return sum(r) - (l > 0 ? sum(l - 1) : 0); }
}
```

0-indexed externally, 1-indexed internally. O(log n) per operation. Pair with coordinate compression for "count smaller after self" (LC 315).

---

## Segment Tree (Range Query, Point Update)

```java
class SegTree {
    private final long[] t;
    private final int n;

    SegTree(int[] a) {
        n = a.length;
        t = new long[2 * n];
        for (int i = 0; i < n; i++) t[n + i] = a[i];
        for (int i = n - 1; i > 0; i--) t[i] = t[2 * i] + t[2 * i + 1];
    }

    void update(int i, long v) {
        for (t[i += n] = v; i > 1; i >>= 1) t[i >> 1] = t[i] + t[i ^ 1];
    }

    long query(int l, int r) {                        // [l, r)
        long res = 0;
        for (l += n, r += n; l < r; l >>= 1, r >>= 1) {
            if ((l & 1) == 1) res += t[l++];
            if ((r & 1) == 1) res += t[--r];
        }
        return res;
    }
}
```

The iterative bottom-up form — shorter and faster than the recursive one. Swap `+` for `Math.min`/`Math.max` (and the identity `0` for `INF`/`-INF`) for other query types.

---

## Prefix Sums

```java
// 1D
long[] pre = new long[n + 1];
for (int i = 0; i < n; i++) pre[i + 1] = pre[i] + a[i];
long sum = pre[r + 1] - pre[l];                       // a[l..r]

// 2D
long[][] pre = new long[R + 1][C + 1];
for (int i = 0; i < R; i++)
    for (int j = 0; j < C; j++)
        pre[i+1][j+1] = pre[i][j+1] + pre[i+1][j] - pre[i][j] + g[i][j];
long sum = pre[r2+1][c2+1] - pre[r1][c2+1] - pre[r2+1][c1] + pre[r1][c1];

// Difference array — O(1) range update, O(n) final build
int[] diff = new int[n + 1];
diff[l] += v; diff[r + 1] -= v;
for (int i = 1; i <= n; i++) diff[i] += diff[i - 1];
```

Sizing at `n + 1` and shifting by one removes the `i == 0` special case from all three.

---

## Memoization

```java
// 2D int memo with a -1 sentinel
int[][] memo = new int[n][m];
for (int[] row : memo) Arrays.fill(row, -1);

private int dp(int i, int j) {
    if (i >= n) return 0;
    if (memo[i][j] != -1) return memo[i][j];
    return memo[i][j] = Math.max(dp(i + 1, j), 1 + dp(i + 1, j + 1));
}
```

When `-1` is a legal answer, use `Integer[][]` and check for `null`, or a separate `boolean[][] computed`.

Sparse or non-integer state — encode the key:

```java
Map<Long,Integer> memo = new HashMap<>();
long key = (long) i * 100000 + j;
if (memo.containsKey(key)) return memo.get(key);
```

`Map<String,Integer>` with `i + "," + j` also works and is instant to write, but the string building dominates the runtime. Prefer the `long` key.

---

## Backtracking Skeleton

```java
List<List<Integer>> res = new ArrayList<>();
List<Integer> path = new ArrayList<>();

private void backtrack(int[] nums, int start) {
    res.add(new ArrayList<>(path));                   // COPY, not the live reference
    for (int i = start; i < nums.length; i++) {
        if (i > start && nums[i] == nums[i - 1]) continue;   // skip duplicates (sorted input)
        path.add(nums[i]);
        backtrack(nums, i + 1);                       // i for reuse-allowed, i+1 for use-once
        path.remove(path.size() - 1);                 // undo
    }
}
```

Two rules: **copy on record**, and **undo on the way out**. `path.remove(path.size() - 1)` removes by index, which is what you want here.

With a `StringBuilder` path, `sb.setLength(len)` is the O(1) undo.

---

## Graph Representations

```java
// Adjacency list from an edge array
List<List<Integer>> adj = new ArrayList<>();
for (int i = 0; i < n; i++) adj.add(new ArrayList<>());
for (int[] e : edges) { adj.get(e[0]).add(e[1]); adj.get(e[1]).add(e[0]); }

// Array-of-lists — less boxing overhead
List<Integer>[] adj = new List[n];
for (int i = 0; i < n; i++) adj[i] = new ArrayList<>();

// Weighted
List<int[]>[] adj = new List[n];
adj[u].add(new int[]{v, w});

// Indegree for topological sort
int[] indeg = new int[n];
for (int[] e : edges) indeg[e[1]]++;
Deque<Integer> q = new ArrayDeque<>();
for (int i = 0; i < n; i++) if (indeg[i] == 0) q.offer(i);
List<Integer> order = new ArrayList<>();
while (!q.isEmpty()) {
    int u = q.poll();
    order.add(u);
    for (int v : adj[u]) if (--indeg[v] == 0) q.offer(v);
}
boolean hasCycle = order.size() < n;
```

`List<Integer>[] adj = new List[n]` produces an unchecked-warning; it's the standard trade for avoiding `ArrayList<List<Integer>>` overhead. Add `@SuppressWarnings("unchecked")` if it bothers you.

---

## Dijkstra

```java
long[] dist = new long[n];
Arrays.fill(dist, Long.MAX_VALUE / 2);
dist[src] = 0;
PriorityQueue<long[]> pq = new PriorityQueue<>((a, b) -> Long.compare(a[0], b[0]));
pq.offer(new long[]{0, src});
while (!pq.isEmpty()) {
    long[] cur = pq.poll();
    long d = cur[0];
    int u = (int) cur[1];
    if (d > dist[u]) continue;                        // stale entry — lazy deletion
    for (int[] e : adj[u]) {
        int v = e[0], w = e[1];
        if (d + w < dist[v]) { dist[v] = d + w; pq.offer(new long[]{dist[v], v}); }
    }
}
```

`if (d > dist[u]) continue;` is the workaround for the missing decrease-key — `pq.remove(Object)` is O(n), so you push an improved entry and skip outdated ones as they surface.

---

## Binary Search Templates

```java
// Lower bound: first index with a[i] >= target
int lo = 0, hi = a.length;
while (lo < hi) {
    int mid = (lo + hi) >>> 1;
    if (a[mid] < target) lo = mid + 1;
    else hi = mid;
}
return lo;                              // == a.length if nothing qualifies

// Binary search on the answer: smallest feasible value
int lo = 1, hi = maxPossible, ans = hi;
while (lo <= hi) {
    int mid = lo + (hi - lo) / 2;
    if (feasible(mid)) { ans = mid; hi = mid - 1; }
    else lo = mid + 1;
}
return ans;
```

`(lo + hi) >>> 1` is the overflow-safe midpoint for non-negative bounds — the JDK's own idiom.

---

## Fast I/O (Codeforces, Not LeetCode)

```java
BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
StringTokenizer st = new StringTokenizer(br.readLine());
int n = Integer.parseInt(st.nextToken());

StringBuilder sb = new StringBuilder();
for (...) sb.append(x).append('\n');
System.out.print(sb);
```

`Scanner` is roughly 5–10× slower than `BufferedReader` and will TLE on large inputs. And **build output in a `StringBuilder`** — `System.out.println` in a loop flushes every call.

---

## Debug Printing

```java
System.err.println(Arrays.toString(a));
System.err.println(Arrays.deepToString(grid));
System.err.println(map);                     // HashMap has a readable toString
System.err.println(list);
```

Use `System.err`, not `System.out` — the judge reads stdout, so stray prints there can affect interactive problems.

---

## Timing (Local)

```java
long t0 = System.nanoTime();
solve();
System.err.println((System.nanoTime() - t0) / 1_000_000 + " ms");
```

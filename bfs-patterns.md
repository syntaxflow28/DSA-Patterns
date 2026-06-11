# Breadth-First Search — The Complete Interview Pattern Guide (C++)

BFS explores a graph **in expanding rings**: everything at distance 0, then everything at distance 1, then 2, and so on. That one property — visiting nodes in non-decreasing distance order — makes it *the* algorithm for shortest paths in unweighted graphs, level-by-level tree processing, "minimum number of moves" puzzles, and distance fields radiating from multiple sources. This guide covers every interview variation: the logic, the **core insight** (why it's correct), C++ templates, strong problem sets with descriptive notes, and the pitfalls.

---

## What BFS Actually Is

Strip away the queue and the loop; BFS is one guarantee:

> **BFS visits nodes in non-decreasing order of distance from the source. Therefore, the first time you reach a node is via a shortest path to it.**

Everything else follows from how that guarantee is engineered:

- **The queue is the frontier.** At any moment it holds nodes of at most **two consecutive distances** — some d's at the front, then d+1's behind them. Distances in the queue never decrease front-to-back. This "two-layer invariant" is the heart of every correctness argument below.
- **Why first arrival = shortest path** (the proof interviewers want): induction on distance. All distance-d nodes are dequeued before any distance-(d+1) node — true at d = 0; and a distance-(d+1) node can only be discovered while processing a distance-d node, so it enters the queue after all d's and before any (d+2) can exist. Since a node is discovered the moment *any* neighbor at one-less distance is processed, it cannot be first-reached by a longer route.
- **The load-bearing assumption: every edge costs exactly 1.** The rings are "number of edges" rings. Give edges different weights and ring k no longer means cost k — the guarantee dies, and you need Dijkstra (or the 0-1 BFS trick when weights are only 0 and 1 — Pattern 7).

**The mindset shift that unlocks hard problems:** BFS doesn't need a graph data structure — it needs a **state** and a way to generate **neighboring states**. The "graph" can be implicit and astronomically large: lock combinations ("0000" → 8 neighbors by turning one wheel), words (one-letter mutations), puzzle configurations, or composite states like *(position, keys collected)*. If you can define a state and its transitions, BFS finds the minimum number of moves between any two states. Half of all hard BFS problems are really *state-design* problems (Pattern 4).

**BFS vs DFS vs Dijkstra — the one-line decision rule:**

| Question | Tool |
|---|---|
| Shortest path / fewest moves, edges all equal | **BFS** |
| Shortest path, weighted edges (non-negative) | Dijkstra (heap guide, Pattern 6) |
| Weights are only 0 and 1 | 0-1 BFS with a deque (Pattern 7) |
| Does a path exist / explore everything / enumerate paths | DFS (simpler, less memory on deep graphs) |
| Anything per-level: level order, rightmost per level, "rings" | **BFS** |
| Longest path, count paths, backtracking choices | DFS/DP — BFS's early-stop advantage is useless |

Memory profile worth knowing: BFS holds an entire frontier — up to O(n) nodes (a complete binary tree's last level is n/2). DFS holds one root-to-leaf path. On wide-and-shallow graphs DFS is lighter; on deep-and-narrow graphs BFS is. Mentioning this trade-off unprompted reads well.

---

## The Visited Discipline — BFS's One Sacred Rule

**Mark a node visited when you ENQUEUE it, not when you dequeue it.**

Why it matters: between enqueue and dequeue, a node sits in the queue for a long time. If you only mark at dequeue, every other neighbor that sees it meanwhile pushes a duplicate — on grids this snowballs into exponential queue growth and TLE with a *correct-looking* program. Marking at enqueue means each node enters the queue at most once, capping work at O(V + E).

Contrast with Dijkstra, where it's the opposite (settle on **pop**, because a node's tentative cost can improve while queued). The two rules differ for one reason: in BFS, the first discovery is already optimal; in Dijkstra it isn't. If you can articulate that contrast, you understand both algorithms.

Corollaries:
- The start node is marked **before** the loop, as it's enqueued.
- Storing distance: either a `dist` array doubling as the visited set (`dist[v] == -1` ⇔ unvisited), or push `(node, dist)` pairs, or count layers with the level-size loop. The `dist`-array approach kills two birds and is hardest to get wrong.
- On grids, "visited" can often be written into the grid itself (flip `'1'` → `'0'`, or mark with a sentinel) — O(1) extra space and no separate structure.

---

## How to Recognize a BFS Problem

**1. Phrasing signals.**
- "**Shortest** path", "**minimum number of** steps / moves / operations / mutations / rotations" — with unit-cost moves, this is BFS, full stop.
- "**Nearest** / closest X for every cell" — multi-source BFS (Pattern 3).
- "**Level by level**", "each level's average / max / rightmost", "depth", "rings", "spread/infection per minute" — level-structured BFS (Patterns 1, 3).
- "Can you finish / valid ordering given prerequisites" — topological sort, the BFS flavor (Pattern 6).

**2. Structural signals.** Grids with 4/8-directional movement; trees with per-level questions; transformation puzzles ("turn word A into word B", "reach this lock combination") — implicit graphs begging for state-space BFS.

**3. Constraint arithmetic.** State spaces like 10⁴ lock combos, 12! puzzle states, n·2ⁿ (node, bitmask) pairs: if the state count × branching factor fits in ~10⁷–10⁸, plain BFS works. If it's borderline, bidirectional BFS (Pattern 8) square-roots the explored volume.

**4. The negative signal.** "Longest path", "count all paths", "all solutions" — BFS's superpower is stopping early at first arrival; these problems forbid early stopping, so reach for DFS/DP instead.

---

## Pattern 1: Tree Level-Order — The Level-Size Loop

**Logic:** BFS on a tree, but you need to know **where one level ends and the next begins**. The trick: at the top of each round, snapshot `sz = q.size()` — that's exactly the current level — and process precisely `sz` pops before touching the next level.

**Core insight — why it works:** This is the two-layer invariant made operational. At the moment a round begins, the queue contains *all* of level d and *nothing else* (level d+1 hasn't been generated yet; level d−1 is fully drained). So `q.size()` is a perfect census of the level, and the inner `for` loop is a clean level boundary — no sentinels, no (node, depth) pairs needed. Every "per-level" question (sum, max, rightmost, zigzag direction) becomes trivial bookkeeping inside the inner loop.

**Template:**
```cpp
queue<TreeNode*> q;
if (root) q.push(root);
while (!q.empty()) {
    int sz = q.size();                  // census of the CURRENT level — snapshot it!
    for (int i = 0; i < sz; i++) {      // process exactly one level
        TreeNode* node = q.front(); q.pop();
        // per-level logic: i == 0 is leftmost, i == sz-1 is rightmost
        if (node->left)  q.push(node->left);
        if (node->right) q.push(node->right);
    }
    // level boundary: finalize this level's answer here
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 102. Binary Tree Level Order Traversal | Medium | The reference implementation — collect each inner loop's values into one vector. Get this to muscle memory; six other problems are one-line edits of it. |
| 103. Zigzag Level Order | Medium | Same loop + a boolean flipped per level; reverse the level vector (or index into it from either end) when the flag says so. Don't reverse the *queue* — only the output. |
| 199. Right Side View | Medium | Record only the `i == sz-1` node per level. (The DFS right-first solution is a nice alternative to mention.) |
| 637. Average of Levels | Easy | Sum inside the inner loop, divide at the level boundary. Sum in `long long` — levels can hold many INT-sized values. |
| 515. Largest Value in Each Row | Medium | Max instead of sum. Initialize with the level's first value or INT_MIN, not 0 — values can be negative. |
| 116/117. Populating Next Right Pointers | Medium | Wire `prev->next = node` within the inner loop. The O(1)-space follow-up (walk level d while wiring level d+1 using already-built next pointers) is a frequent senior follow-up. |
| 1161. Maximum Level Sum | Medium | Track the best (sum, level) pair across boundaries; return the smallest level on ties — read the tie rule before coding. |
| 513. Find Bottom Left Tree Value | Medium | The `i == 0` node of the last level processed. Alternatively BFS right-to-left and return the final pop — cute, but the standard loop is clearer. |
| 662. Maximum Width of Binary Tree | Medium | Push (node, index) with heap-numbering (2i, 2i+1); width = last − first index per level. Indices overflow fast — normalize by subtracting the level's first index, and use `unsigned long long`. |

**Pitfalls:**
- Reading `q.size()` inside the loop condition (`for (int i = 0; i < q.size(); i++)`) — the size *changes* as you push children. Snapshot it first; this is the classic level-order bug.
- Null children: either guard before pushing (cleaner) or guard after popping — pick one, never mix, or you'll double-handle nulls.
- In 662, the index arithmetic (not the BFS) is the problem: unnormalized 2i+1 numbering overflows 64 bits on skewed trees of depth ~60.

---

## Pattern 2: Unweighted Shortest Path — Single Source (Grids & Graphs)

**Logic:** Standard BFS from the source, tracking distance (via a `dist` array or the level loop). The first time the target pops — or is enqueued — its distance is final. On grids, "neighbors" are the 4 (or 8) adjacent cells that are in-bounds, passable, and unvisited.

**Core insight — why it works:** This is the first-arrival theorem applied directly: with unit edges, BFS's expanding rings *are* the distance field, so no comparisons, relaxations, or priority queues are needed — arrival order does all the work that Dijkstra's heap does in weighted graphs. The practical consequence: you may **stop at first arrival** at the target. BFS explores ~(area of the disk of radius d) and not one node more; this early-exit property is exactly what's lost in longest-path or all-paths problems, which is why BFS doesn't apply there.

**Template (grid shortest path, dist-array-as-visited):**
```cpp
int dirs[4][2] = {{1,0},{-1,0},{0,1},{0,-1}};
vector<vector<int>> dist(m, vector<int>(n, -1));     // -1 = unvisited
queue<pair<int,int>> q;
dist[sr][sc] = 0; q.push({sr, sc});                  // mark AT enqueue
while (!q.empty()) {
    auto [r, c] = q.front(); q.pop();
    if (r == tr && c == tc) return dist[r][c];       // first arrival = answer
    for (auto& d : dirs) {
        int nr = r + d[0], nc = c + d[1];
        if (nr < 0 || nr >= m || nc < 0 || nc >= n) continue;
        if (grid[nr][nc] != PASSABLE || dist[nr][nc] != -1) continue;
        dist[nr][nc] = dist[r][c] + 1;               // discover = finalize
        q.push({nr, nc});
    }
}
return -1;                                            // unreachable
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 1091. Shortest Path in Binary Matrix | Medium | 8-directional movement and the path *length counts cells, not edges* — start dist at 1. Also check the start/end cells themselves are open; both are the actual test. |
| 1926. Nearest Exit from Entrance in Maze | Medium | Clean template drill with one twist: the entrance doesn't count as an exit — exclude it explicitly. |
| 433. Minimum Genetic Mutation | Medium | The graph is implicit: neighbors = bank strings differing by one character. Tiny state space; a gentle on-ramp to Pattern 4. |
| 752. Open the Lock | Medium | 10⁴ states, 8 neighbors each (±1 per wheel, with wraparound). Deadends = pre-visited nodes. Check the start itself isn't a deadend — the missed edge case in most first attempts. |
| 1306. Jump Game III | Medium | BFS (or DFS) on indices with jumps i ± arr[i]. Connectivity question, so either traversal works — say why you chose one. |
| 1654. Minimum Number of Jumps (Bug Zapper) | Medium | The state needs more than position: a "just moved backward" flag (no two backward jumps in a row), and a bounded domain argument (positions beyond ~6000 never help). Bridges into Pattern 4's state design. |
| 1730. Shortest Path to Get Food | Medium | Pure template on a grid with multiple possible targets — stop at the first `'#'` popped. |
| 909. Snakes and Ladders | Medium | The board *is* a graph: squares 1..n², neighbors = die rolls 1–6 then teleport. The hard part is the boustrophedon (zigzag) square→cell mapping, not the BFS. |

**Pitfalls:**
- Counting cells vs edges (1091): off-by-one in the final answer. Decide the convention from the problem's examples before writing the init.
- Bounds-check **before** array access — `grid[nr][nc]` with nr = −1 is UB that may "work" until it doesn't.
- Re-checking `visited` at pop "just in case" signals the enqueue-marking wasn't trusted; with correct enqueue-marking it's dead code. Know which discipline you're using.

---

## Pattern 3: Multi-Source BFS — Distance Fields

**Logic:** Seed the queue with **all** sources at distance 0 (instead of one), then run perfectly ordinary BFS. The resulting `dist[v]` is the distance from v to its **nearest** source. Equivalent framings: "rot/fire/water spreading simultaneously from many points," or "for every cell, how far to the nearest X?"

**Core insight — why it works:** Imagine a virtual **super-source** S with zero-cost edges to every real source. BFS from S gives `dist(S, v) = min over sources s of dist(s, v)` — and seeding all sources at 0 *is* BFS from S with the trivial first layer skipped. All the layer-ordering proofs go through untouched because the two-layer invariant never cared how many nodes started in layer 0. The payoff is dramatic: "for each of n cells, distance to nearest of k sources" looks like k separate BFS runs (O(k·V)) but is **one** BFS (O(V + E)). The inverted-perspective cue: when the naive loop is "for every empty cell, BFS to find the nearest gate," flip it — BFS *from all gates at once*.

**Template (distance to nearest 0 — LC 542):**
```cpp
queue<pair<int,int>> q;
vector<vector<int>> dist(m, vector<int>(n, -1));
for (int r = 0; r < m; r++)
    for (int c = 0; c < n; c++)
        if (mat[r][c] == 0) { dist[r][c] = 0; q.push({r, c}); }  // ALL sources seeded

while (!q.empty()) {                       // ordinary BFS from here on
    auto [r, c] = q.front(); q.pop();
    for (auto& d : dirs) {
        int nr = r + d[0], nc = c + d[1];
        if (nr < 0 || nr >= m || nc < 0 || nc >= n || dist[nr][nc] != -1) continue;
        dist[nr][nc] = dist[r][c] + 1;
        q.push({nr, nc});
    }
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 542. 01 Matrix | Medium | The template problem. The trap is direction: BFS from the **0s** outward, not from each 1 — recognizing the inversion is the entire problem. |
| 994. Rotting Oranges | Medium | Seed all rotten oranges; answer = number of layers (level-size loop) until no fresh remain. Count fresh oranges up front — if any survive the BFS, return −1. Watch the zero-fresh edge case (answer 0). |
| 286. Walls and Gates | Medium | Seed all gates at 0, fill rooms with nearest-gate distance. The premium-locked classic that 542 clones. |
| 1162. As Far from Land as Possible | Medium | Seed all land; the answer is the **last** layer's distance — the water cell whose nearest land is farthest. Maximin via multi-source BFS. |
| 417. Pacific Atlantic Water Flow | Medium | Two multi-source BFS runs (one per ocean's border cells, flowing *uphill*), intersect the reachable sets. Inverting the flow direction is the insight. |
| 1765. Map of Highest Peak | Medium | Seed water cells at 0; BFS layers directly assign valid heights (adjacent差 ≤ 1 is automatic since BFS layers differ by exactly 1). |
| 2812. Find the Safest Path | Medium | Phase 1: multi-source BFS from all thieves builds the safety field. Phase 2: maximize the path's minimum safety (heap or binary search + BFS). Multi-source as a *subroutine* of a bigger algorithm. |
| 815. Bus Routes | Hard | Not multi-source but multi-*layer modeling*: BFS over routes (not stops) — board all routes through the start stop, neighbors = routes sharing any stop. Choosing the right node granularity is the problem. |

**Pitfalls:**
- Seeding sources without marking them visited/dist=0 — they get "re-discovered" at distance 2 from each other and corrupt the field.
- In layer-counting problems (994), the careless loop counts one layer too many (the final layer that discovers nothing) — count layers only when the layer actually processed something, or track the max dist assigned.
- Inversion blindness: if your plan involves "BFS from every cell," stop and check whether one BFS from all targets answers it. It almost always does.

---

## Pattern 4: State-Space BFS — Designing the Node

**Logic:** When position alone doesn't determine what you can do next, the BFS node must be the **full state**: (position, keys held), (pattern of the puzzle board), (node, bitmask of visited nodes), (position, direction), (word). Transitions define an implicit graph; BFS finds the minimum number of moves. The art is choosing a state that is *sufficient* (captures everything affecting future moves) and *minimal* (no irrelevant fields exploding the space).

**Core insight — why it works:** BFS's guarantee is about *nodes*, and a node is whatever you say it is. If two situations have the same state, BFS treats them as identical — so the state must encode **exactly** the information that affects future options: holding a key changes which doors open → keys belong in the state; the path you took to get here doesn't change anything ahead → it stays out. Sufficiency makes the answer correct; minimality makes it computable (each extra bit doubles the space). The discipline: before coding, *compute the state count* — 4ⁿ lock states, n·2ᵏ position-keys pairs, 12!/2 puzzle boards — and multiply by branching factor. Fits in ~10⁷? BFS away. Doesn't? You need bidirectional BFS, A*, or a smaller state.

**Template (BFS over encoded states):**
```cpp
unordered_set<string> visited;            // or unordered_map<State,int,Hash>
queue<string> q;
q.push(start); visited.insert(start);
int steps = 0;
while (!q.empty()) {
    int sz = q.size();
    for (int i = 0; i < sz; i++) {                 // level loop = move counter
        string cur = q.front(); q.pop();
        if (cur == target) return steps;
        for (string& nxt : neighbors(cur))         // the problem-specific part
            if (!visited.count(nxt)) {
                visited.insert(nxt);               // mark at enqueue, as always
                q.push(nxt);
            }
    }
    steps++;
}
return -1;
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 127. Word Ladder | Hard | State = word. Neighbor generation is the complexity battle: trying all 26·L single-letter mutations and set-checking (O(26·L²) per word) crushes the O(n²) pairwise comparison. The flagship implicit-graph problem. |
| 773. Sliding Puzzle | Hard | State = the 2×3 board serialized to a 6-char string; neighbors = swaps with the blank. 360 reachable states — tiny once you *see* the board as a graph node. |
| 864. Shortest Path to Get All Keys | Hard | State = (row, col, keys-bitmask). The same cell with different keys is a **different node** — the purest demonstration of why state design matters. Space: m·n·2ᵏ ≤ 64·30·30. |
| 847. Shortest Path Visiting All Nodes | Hard | State = (node, visited-bitmask), seeded multi-source from every node with its own bit. n·2ⁿ ≤ 12·4096 states. TSP-shaped, but BFS-solvable because moves cost 1. |
| 1293. Shortest Path with Obstacle Elimination | Hard | State = (row, col, eliminations remaining). Prune: only revisit a cell if you arrive with *more* eliminations left — dominance between states, a recurring state-BFS optimization. |
| 1129. Shortest Path with Alternating Colors | Medium | State = (node, color of last edge). Two distances per node; small and clean — the best first problem for "position isn't enough." |
| 1654. Minimum Number of Jumps | Medium | State = (position, just-went-back flag), plus a bound argument capping positions. Forgetting the flag silently forbids legal paths. |
| 365. Water and Jug Problem | Medium | State = (a, b) jug contents, 6 transitions. (The Bézout/GCD math answer exists, but the BFS models the actual puzzle and generalizes.) |
| 752. Open the Lock | Medium | Cross-listed from Pattern 2 — it *is* state BFS; revisit it here and compute its state count (10⁴ × 8 transitions) as a sizing drill. |

**Pitfalls:**
- Under-specified state — the silent killer. Symptom: BFS returns a path that's illegal on inspection (walked through a door without the key). Fix: ask "do two situations with this same state truly have identical futures?" for every field you omit.
- State encoding for the hash set: pack small fields into one int (`r * n * 64 + c * 64 + keys`) — string concatenation works but costs allocations in the hot loop.
- Forgetting the target test can be *predicate*, not equality: "all keys collected" = `keys == (1<<k) − 1`, "all nodes visited" = full mask. Test at pop or push consistently.

---

## Pattern 5: Connectivity & Flood Fill — BFS as Region Labeler

**Logic:** No distances at all — BFS here is a *paint bucket*. Scan the grid; every time you find an unvisited cell of interest, that's a brand-new region: BFS from it, marking everything reachable, while accumulating region stats (count, area, border-touching). The outer scan counts regions; the inner BFS consumes them.

**Core insight — why it works:** Connectivity is an equivalence relation, and one BFS from any cell visits **exactly** its equivalence class — no more (visited-marking stops at the boundary), no less (BFS reaches everything reachable). So "number of islands" = number of times the outer scan finds virgin territory, and each cell is processed exactly once across all BFS calls → O(mn) total, despite the nested-looking structure (the same amortized accounting as sliding window's inner while). DFS does this job identically; BFS's edge is no recursion-depth risk on huge grids (a 300×300 snake-shaped island overflows the stack in DFS), which is a concrete reason to choose it — have one.

**Template (count islands):**
```cpp
int islands = 0;
for (int r = 0; r < m; r++)
    for (int c = 0; c < n; c++)
        if (grid[r][c] == '1') {
            islands++;                          // virgin territory = new region
            queue<pair<int,int>> q;
            grid[r][c] = '0';                   // grid itself is the visited set
            q.push({r, c});
            while (!q.empty()) {
                auto [cr, cc] = q.front(); q.pop();
                for (auto& d : dirs) {
                    int nr = cr + d[0], nc = cc + d[1];
                    if (nr < 0 || nr >= m || nc < 0 || nc >= n || grid[nr][nc] != '1') continue;
                    grid[nr][nc] = '0';         // mark at enqueue — even here
                    q.push({nr, nc});
                }
            }
        }
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 200. Number of Islands | Medium | The canonical region counter (template above). Asked everywhere; the follow-up ladder goes union-find (for dynamic island addition — LC 305) and the BFS/DFS/UF comparison. |
| 695. Max Area of Island | Medium | Same scan; the inner BFS returns its region's size, outer takes the max. |
| 733. Flood Fill | Easy | Single region from a given seed. Edge case worth saying aloud: new color == old color → return immediately or loop forever. |
| 130. Surrounded Regions | Medium | Inverted: BFS from **border** O's marking the safe ones, then flip everything unmarked. Solving by "capture the surrounded" directly is the trap; "protect the border-connected" is the solution. |
| 1020. Number of Enclaves | Medium | Same border-inversion as 130, counting instead of flipping. Do them back-to-back. |
| 827. Making a Large Island | Hard | Two phases: label every island with an id and size (flood fill), then for each 0-cell, sum the *distinct* neighboring island sizes + 1. The labeling pass is this pattern; the dedup-by-id is the detail that fails people. |
| 542 / 1162 | — | Cross-reference: when regions need *distances*, not labels, you're in Pattern 3 — the scan-and-fill skeleton is the same, the payload differs. |
| 1254. Number of Closed Islands | Medium | 130's logic on 0-islands: discard any region touching the border. One boolean carried through the BFS. |

**Pitfalls:**
- Mutating the input grid as the visited set: free and fast, but *say* you're doing it and offer a copy if the interviewer objects — silent input mutation is a style flag.
- Even with no distances, mark at **enqueue**: marking at pop on flood fill is the most common cause of mysterious TLE on big grids (cells enqueue 4 times each).
- 4-directional vs 8-directional: islands are almost always 4-dir, but read the problem — 1091 was 8-dir and the assumption transfer is a real bug source.

---

## Pattern 6: Topological Sort — Kahn's Algorithm (BFS on Indegrees)

**Logic:** In a DAG of prerequisites, repeatedly peel off nodes with **indegree 0** (nothing left blocking them): seed the queue with all of them; popping a node "completes" it, decrementing each dependent's indegree; any dependent hitting 0 joins the queue. The pop order is a valid topological order. If pops < n, a cycle remains.

**Core insight — why it works:** A node is safe to schedule exactly when all its prerequisites are done — i.e., when its *remaining* indegree is 0. Peeling such a node can only *reduce* others' indegrees, never create new blockage, so the process is monotone and never needs backtracking. The cycle test falls out for free: nodes on a cycle wait on each other forever, so none ever reaches indegree 0 — they're precisely the never-popped nodes, hence `popped < n ⇔ cycle`. Bonus structure: if you run it with the level-size loop, layer k = "tasks startable in round k given unlimited parallelism" — the longest chain / minimum semesters question answered by the same code.

**Template (course schedule):**
```cpp
vector<vector<int>> adj(n);
vector<int> indeg(n, 0);
for (auto& p : prerequisites) {
    adj[p[1]].push_back(p[0]);          // edge: prereq → course
    indeg[p[0]]++;
}
queue<int> q;
for (int i = 0; i < n; i++) if (indeg[i] == 0) q.push(i);   // all unblocked

vector<int> order;
while (!q.empty()) {
    int u = q.front(); q.pop();
    order.push_back(u);
    for (int v : adj[u])
        if (--indeg[v] == 0) q.push(v);  // last blocker removed → ready
}
return (int)order.size() == n ? order : vector<int>{};       // short = cycle
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 207. Course Schedule | Medium | "Can you finish?" = "is it a DAG?" = does Kahn pop all n. The single most-asked topological sort question. |
| 210. Course Schedule II | Medium | Same code, return the pop order. Edge direction (prereq → course, not the reverse) is where the wrong answers come from — draw one example edge before coding. |
| 269. Alien Dictionary | Hard | Build the graph first: compare *adjacent* words, first differing character gives one edge. The "apple" before "app" case (prefix anomaly) is invalid input — detect it during graph building, not after. |
| 310. Minimum Height Trees | Medium | Inverted peeling: repeatedly remove **leaves** (degree 1) layer by layer; the last 1–2 survivors are the centers. Kahn's structure on an undirected tree — the layered peel is the shared DNA. |
| 1136. Parallel Courses | Medium | Level-size loop on Kahn: number of layers = minimum semesters. The "BFS layers = parallel rounds" bonus made into a whole problem. |
| 2115. Recipes from Given Supplies | Medium | Topological sort where ingredients unlock recipes that are themselves ingredients — Kahn with a map from names to nodes. Modeling practice. |
| 802. Find Eventual Safe States | Medium | Reverse all edges, then Kahn from terminal nodes: safe = reachable in the peel. "Reverse the graph and peel" is a reusable move. |
| 444. Sequence Reconstruction | Medium | The order is unique iff the queue holds **exactly one** node at every step — uniqueness via Kahn, a subtle and elegant check. |

**Pitfalls:**
- Edge direction confusion (prereq→course vs course→prereq) flips the output order; one drawn example prevents it.
- Forgetting the cycle check (`order.size() == n`) — on cyclic input the code returns a *partial* order that looks plausible.
- Duplicate edges (269, some inputs of 207) inflate indegrees unless deduped — a node then never reaches 0 and you report a phantom cycle.
- 310 peels by **degree** (undirected), not indegree, and stops at ≤ 2 survivors, not 0 — same skeleton, three changed lines; don't autopilot.

---

## Pattern 7: 0-1 BFS — The Deque Trick

**Logic:** Edge weights are only **0 and 1** (free moves vs costly moves: "follow the arrow free, change it for 1"; "step on empty free, remove obstacle for 1"). Replace the queue with a **deque**: relaxing a 0-edge pushes the neighbor to the **front** (same distance — it belongs with the current layer), a 1-edge pushes to the **back** (next layer). Pop from the front as usual.

**Core insight — why it works:** BFS's whole correctness rests on the queue holding non-decreasing distances. A 0-edge neighbor has the *same* distance as the node being processed — appending it to the back (behind d+1 nodes) would break the ordering, but prepending keeps the deque sorted: it still reads as "some d's, then d+1's." The invariant survives, so first-pop is still optimal — Dijkstra's guarantee at queue prices: **O(V + E)** instead of O(E log V), no heap. The recognition cue is sharp: the moment a shortest-path problem has exactly two move costs and one of them is zero, the deque trick applies. (Three or more distinct weights? The invariant can't be patched — that's Dijkstra's job.)

**Template:**
```cpp
deque<pair<int,int>> dq;
vector<vector<int>> dist(m, vector<int>(n, INT_MAX));
dist[0][0] = 0; dq.push_front({0, 0});
while (!dq.empty()) {
    auto [r, c] = dq.front(); dq.pop_front();
    for (each move with cost w ∈ {0, 1}) {
        if (dist[r][c] + w < dist[nr][nc]) {
            dist[nr][nc] = dist[r][c] + w;
            if (w == 0) dq.push_front({nr, nc});   // same layer — jump the line
            else        dq.push_back({nr, nc});    // next layer — wait your turn
        }
    }
}
```
(Note: 0-1 BFS relaxes like Dijkstra — `dist` comparison, not a visited set — because a node can be improved while in the deque.)

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 1368. Min Cost to Make at Least One Valid Path | Hard | The flagship: following the cell's arrow costs 0, overriding it costs 1. Pure 0-1 BFS; also solvable with Dijkstra at a log-factor penalty — name both. |
| 2290. Minimum Obstacle Removal | Hard | Stepping on empty = 0, removing an obstacle = 1 — the grid *is* the weight function. Identical machinery to 1368. |
| 1293. Shortest Path with Obstacle Elimination | Hard | Cross-listed from Pattern 4: with the (cell, eliminations) state it's plain BFS; reframed as 0/1 costs it's 0-1 BFS. Comparing your own two solutions is excellent interview practice. |
| 934. Shortest Bridge | Medium | Hybrid: flood-fill one island (Pattern 5), then multi-source BFS from it (Pattern 3) to the other. Listed here because the "expand island for free, cross water for 1" reading is a 0-1 BFS in disguise — three patterns, one problem. |
| 542-with-diagonals variants / chess knight oddities | — | Whenever someone bolts a free move onto a unit-cost grid, reach for the deque before reaching for Dijkstra. |

**Pitfalls:**
- Using a visited-at-enqueue set: **wrong here** — like Dijkstra, a deque entry can be stale; relax on distance improvement and skip stale pops (`if (d > dist[r][c]) continue;` if you store distances in the deque).
- Pushing 0-edges to the back "because it still terminates" — it terminates with *wrong answers*; the front-push is correctness, not optimization.
- Reaching for full Dijkstra on 0/1 weights: correct but a log slower, and interviewers fishing for 0-1 BFS will ask if you can do better.

---

## Pattern 8: Bidirectional BFS — Meet in the Middle

**Logic:** When start *and* target are both known, run BFS from both ends simultaneously, always expanding the **smaller frontier** one full layer. The moment any newly generated state appears in the other side's visited set, the shortest path is (steps from start) + (steps from target) + 1.

**Core insight — why it works:** Search volume in a branching-factor-b space grows as b^d — exponential in depth. Two half-depth searches cost 2·b^(d/2), and √(b^d) vs b^d is the difference between 10³ and 10⁶ explored states. Correctness holds because each side independently maintains the BFS layer guarantee, so when the frontiers first touch, both half-distances are minimal and their sum is the true shortest path — *provided* you check for the meeting at **generation time** and expand layer-by-layer (mixing layers breaks the "first touch = optimal" argument). Always expanding the smaller frontier keeps both balls balanced, which is where the √ savings actually comes from.

**Template (sketch):**
```cpp
unordered_set<string> beginSet{start}, endSet{target}, visited;
int steps = 1;
while (!beginSet.empty() && !endSet.empty()) {
    if (beginSet.size() > endSet.size()) swap(beginSet, endSet);  // expand smaller side
    unordered_set<string> next;
    for (const string& cur : beginSet)
        for (string& nb : neighbors(cur)) {
            if (endSet.count(nb)) return steps;     // frontiers touched — done
            if (!visited.count(nb)) {
                visited.insert(nb);
                next.insert(nb);
            }
        }
    beginSet = move(next);
    steps++;
}
return 0;
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 127. Word Ladder | Hard | The classic beneficiary: branching ~26·L makes unidirectional BFS sweat; bidirectional cuts explored words by orders of magnitude. The standard "optimize your BFS" follow-up. |
| 433. Minimum Genetic Mutation | Medium | Same shape, 4-letter alphabet, tiny space — a safe place to *practice* the bidirectional mechanics before needing them. |
| 752. Open the Lock | Medium | Both endpoints known, uniform branching of 8 — textbook bidirectional fit. Deadends go into the shared `visited` up front. |
| 773. Sliding Puzzle | Hard | Works bidirectionally since the target board is known; the state space is small enough that it's optional — discuss the trade-off rather than auto-applying. |
| 126. Word Ladder II | Hard | All shortest paths, not just the length: BFS (optionally bidirectional) builds a layered parent DAG, then DFS backtracks the paths. The BFS/DFS division of labor is the lesson. |

**Pitfalls:**
- Only applicable when the **target state is explicitly known** — "reach any exit" or predicate targets can't anchor the second search.
- Checking the meeting at pop instead of at generation, or expanding sides unevenly mid-layer — both quietly break the optimality argument. Expand whole layers.
- The swap-to-smaller trick is load-bearing for the speedup; without it one side balloons and you've paid bidirectional complexity for unidirectional performance.

---

## When BFS FAILS — Know the Boundary

| Situation | Why BFS breaks | Use instead |
|---|---|---|
| Weighted edges (≥ 0, beyond 0/1) | Rings count edges, not cost — first arrival no longer minimal | Dijkstra (heap) |
| Weights exactly 0 and 1 | Plain queue breaks ordering; deque repairs it | 0-1 BFS (Pattern 7) |
| Negative weights | No greedy settlement is safe at all | Bellman-Ford |
| Longest path / count all paths / enumerate solutions | BFS's value is early stopping; these forbid it | DFS + memo / DP / backtracking |
| State space too big even for BFS (b^d ≫ 10⁸) | Memory dies before time does | Bidirectional, A*, IDA*, better state design |
| Deep recursion fine, frontier huge (wide trees) | O(width) frontier memory | DFS, O(depth) memory |
| Connectivity with *dynamic* additions | Re-running BFS per update is O(V) each | Union-Find |

One-sentence litmus test: **BFS is the answer when every move costs the same and the question is "how few moves" or "what's at each ring."** Different costs → Dijkstra family; no cost question at all → DFS is usually simpler.

---

## Cheat Sheet: Problem Phrase → Pattern

| Phrase in the problem | Pattern |
|---|---|
| "level order / per-level value / zigzag / right side view" | Level-size loop (P1) |
| "shortest path / minimum moves", unit steps | Single-source BFS (P2) |
| "nearest X for every cell", "spreads each minute" | Multi-source BFS (P3) |
| "minimum steps to transform / unlock / solve puzzle" | State-space BFS (P4) |
| state depends on keys / mask / direction / budget | State-space BFS (P4) |
| "number of islands / regions / connected areas" | Flood fill (P5) |
| "prerequisites / valid order / can finish" | Kahn's topological sort (P6) |
| "minimum semesters / parallel rounds" | Kahn + level loop (P6) |
| two move types: one free, one costing 1 | 0-1 BFS deque (P7) |
| known start AND target, huge branching | Bidirectional BFS (P8) |
| "longest path", "all paths" | NOT BFS — DFS/DP |

---

## Complexity Summary

- BFS on explicit graph: **O(V + E)** time, **O(V)** space (visited + frontier).
- Grid BFS: **O(m·n)** — each cell enqueued at most once (the enqueue-marking discipline is what guarantees this).
- Multi-source: still **O(V + E)** — k sources cost nothing extra.
- State-space: **O(states × branching)** — compute it *before* coding; it's the feasibility check.
- Kahn's: **O(V + E)**.
- 0-1 BFS: **O(V + E)** — Dijkstra's answer without Dijkstra's log.
- Bidirectional: ~**O(b^(d/2))** explored vs b^d — a square-root cut.

---

## Interview Tips

1. **Say the guarantee, not the mechanics.** "BFS visits in non-decreasing distance, so first arrival is optimal — that's why no comparisons are needed." One sentence, and the interviewer knows you understand *why*, not just *how*.
2. **Mark visited at enqueue — and say why.** "Otherwise nodes enter the queue multiple times and grid problems TLE." Contrast with Dijkstra's settle-at-pop if you want the senior nod.
3. **For any non-obvious problem, open with the state.** "The node here is (cell, keys-bitmask); the space is m·n·2ᵏ ≈ 60K states — comfortably BFS-able." State design + size arithmetic up front converts a hard problem into a routine one in the interviewer's eyes.
4. **Name the inversion when you use it.** Multi-source ("BFS from all gates, not from each room") and border-inward (130: "protect border-connected, don't hunt surrounded") are *perspective* tricks — flagging them shows pattern fluency.
5. **Know the BFS/DFS choice rationale cold:** shortest/levels → BFS; existence/enumeration/deep-narrow → DFS; recursion-depth risk on big grids → BFS even for plain flood fill.
6. **Level loop vs dist array:** level-size loop when the *layer number* is the answer (steps, minutes, semesters); dist array when *per-node* distances are the answer (fields, comparisons). Choosing the natural one keeps the code short.

---

## Suggested Practice Order

**Week 1 — trees & template fluency:** 102 → 103 → 199 → 637 → 515 → 1161 → 116
**Week 2 — grids & shortest paths:** 733 → 200 → 695 → 1926 → 1091 → 752 → 909
**Week 3 — multi-source & topo:** 542 → 994 → 1162 → 130 → 417 → 207 → 210 → 1136
**Week 4 — boss fights:** 127 → 433 (bidirectional) → 773 → 864 → 847 → 1293 → 1368 → 269 → 815

Good luck with the interviews!

# Shortest Path — The Complete Interview Pattern Guide (C++)

Shortest path looks like one problem but is actually a **family of algorithms selected by the edge weights and the question asked** — and interviews test exactly that selection: BFS when every edge costs 1, a deque when costs are 0/1, Dijkstra for non-negative weights, Bellman-Ford when negatives appear or moves are budgeted, topological relaxation on DAGs, Floyd-Warshall for all pairs. This guide unifies them: the logic, the **core insight** (why each is correct and where its proof breaks), C++ templates, strong problem sets with descriptive notes, and the pitfalls. It cross-references the BFS and Heap guides where they overlap, then goes well beyond them.

---

## What Shortest Path Actually Is

Every algorithm in this guide is built from one atomic operation — **relaxation**:

```cpp
if (dist[u] + w < dist[v]) dist[v] = dist[u] + w;   // edge (u, v, w)
```

"I found a cheaper way to reach v — through u." That's the entire vocabulary. The algorithms differ *only* in the **order** they apply relaxations and in **when they're allowed to stop**:

- **BFS** relaxes in distance-layer order — legal only when all weights are 1, so layer order *is* cost order.
- **Dijkstra** relaxes in tentative-cost order (a heap supplies it) — legal when weights are non-negative, so a settled node can never be improved.
- **Bellman-Ford** gives up on clever ordering and relaxes *every edge*, n−1 times — legal always, because any shortest path has ≤ n−1 edges and each full pass finalizes at least one more edge of it.
- **Topological order** on a DAG relaxes each node after all its predecessors — the DAG hands you a perfect order for free, negative weights and all.
- **Floyd-Warshall** relaxes through intermediate vertices, one at a time, for all pairs simultaneously.

**The correctness backbone** shared by all of them: distances only ever *decrease* toward the truth (relaxation never overshoots — it's always witnessed by an actual path), and each algorithm's ordering guarantees that by the time it reads `dist[u]` to relax outgoing edges, `dist[u]` is either final or will be reprocessed later. When you understand which ordering guarantee each algorithm relies on, you also understand exactly *when each one breaks* — and "when does it break" is the favorite interview probe.

**The decision table — memorize this:**

| Situation | Algorithm | Cost |
|---|---|---|
| All edges weight 1 (unweighted) | BFS | O(V + E) |
| Weights ∈ {0, 1} | 0-1 BFS (deque) | O(V + E) |
| Non-negative weights, single source | Dijkstra | O(E log V) |
| Negative edges allowed / "at most k moves" | Bellman-Ford / layered DP | O(V·E) / O(k·E) |
| Graph is a DAG (any weights) | Topological relaxation | O(V + E) |
| All pairs, small V (≤ ~400) | Floyd-Warshall | O(V³) |
| Path cost = max/min/product of edges | Modified Dijkstra / binary search | varies |

**The second skill — modeling.** Half of hard shortest-path problems hide the graph: nodes might be (cell, keys held), (city, stops used), (node, time mod pattern), or "all positions sharing a value." Before picking an algorithm, answer three questions: *what is a node? what is an edge? what does an edge cost?* — the state-design discipline from the BFS guide, now with weights. Pattern 8 is devoted to it.

---

## How to Recognize a Shortest Path Problem

**1. Phrasing.** "Shortest / minimum cost / minimum time / minimum effort / cheapest / fastest path", "minimum number of moves" (unweighted), "maximum probability" (a disguised shortest path under a log/product transform), "minimize the maximum along the way" (twisted metric).

**2. The weight audit — your first act.** Read the cost model before anything else: all moves equal? → BFS. Two costs, one of them zero? → deque. Arbitrary non-negative? → Dijkstra. Anything negative, or a hop budget? → Bellman-Ford territory. The audit takes ten seconds and selects the algorithm; skipping it is how people run Dijkstra on K-constrained problems and lose 20 minutes.

**3. Structure.** Grids with movement costs; flight/road networks; "convert A into B with operation costs" (implicit state graphs); dependency chains with durations (DAG).

**4. The negative signals.** "Longest simple path" in a general graph — NP-hard, not a relaxation problem (but trivial on DAGs — Pattern 6). "Count paths" alone — DP/BFS-counting, not distance. "Path that visits all nodes" — TSP/bitmask, not shortest path.

---

## Pattern 1: Unweighted — BFS (the degenerate case)

**Logic & insight (condensed — full treatment in the BFS guide):** With unit edges, BFS's layer order *is* cost order, so first arrival = shortest distance and no comparisons are ever needed. Mark visited at **enqueue**. Multi-source variants seed all sources at distance 0. The mistake this pattern exists to prevent: reaching for Dijkstra on an unweighted graph — correct, but a log factor slower and a signal you didn't audit the weights.

**Problems:** 1091 (Shortest Path in Binary Matrix), 127 (Word Ladder), 752 (Open the Lock), 542/994 (multi-source fields), 909 (Snakes and Ladders), 815 (Bus Routes — BFS over *routes*, a modeling gem).

**Pitfall worth re-flagging here:** "minimum number of operations" problems are unweighted shortest paths on state graphs — candidates who only associate BFS with grids miss them. If every operation costs the same, it's BFS, whatever the nodes look like.

---

## Pattern 2: Weights {0, 1} — 0-1 BFS (the deque trick)

**Logic & insight (condensed — full treatment in the BFS guide, Pattern 7):** A 0-edge neighbor has the *same* distance as the current node, so it belongs at the deque's **front** (current layer); a 1-edge neighbor goes to the **back**. The deque stays sorted by distance — Dijkstra's ordering guarantee at queue prices: O(V + E), no heap. Relax like Dijkstra (distance comparison, stale-skip), not like BFS (no visited-at-enqueue).

**Problems:** 1368 (Min Cost Valid Path — follow arrow free, change it for 1), 2290 (Minimum Obstacle Removal), 1293 (Obstacle Elimination — reframable both ways), 934 (Shortest Bridge — flood fill + the "expand free, cross for 1" reading).

**Pitfall:** pushing 0-edges to the back "still terminates" — with wrong answers. The front-push *is* the correctness.

---

## Pattern 3: Dijkstra — The Workhorse

**Logic:** A min-heap holds the frontier keyed by tentative distance. Pop the cheapest node; if the popped entry is stale (worse than the recorded distance), skip it; otherwise the node is **settled** — its distance is final — and you relax its outgoing edges, pushing improved tentative costs. C++ has no decrease-key, so improvements are pushed as duplicates and stale entries die at pop.

**Core insight — why it works:** The settlement is a **cut argument**: when u pops with tentative cost d — the minimum over the entire frontier — any alternative route to u must leave the settled region through some frontier node with cost ≥ d, and with **non-negative edges** can only grow from there. So no future discovery can beat d: settle u permanently, never revisit. That argument is also a precise map of the failure modes: a negative edge beyond the frontier could shrink a path after the exit (→ Bellman-Ford), and a constraint like "≤ k stops" can force a costlier-but-shorter route that Dijkstra's permanent settlement already discarded (→ Pattern 5). One more subtlety worth saying aloud: Dijkstra is BFS with the queue upgraded to a heap — and the visited rule flips accordingly (settle on **pop**, not enqueue) because in BFS first discovery is optimal and here it isn't.

**Template (the canonical form — internalize every line):**
```cpp
vector<long long> dist(n, LLONG_MAX);
priority_queue<pair<long long,int>,
               vector<pair<long long,int>>, greater<>> pq;   // (dist, node) — dist FIRST
dist[src] = 0;
pq.push({0, src});
while (!pq.empty()) {
    auto [d, u] = pq.top(); pq.pop();
    if (d > dist[u]) continue;               // stale duplicate — lazy deletion
    // (optional) if (u == target) return d; // settled = final: safe early exit
    for (auto [v, w] : adj[u])
        if (dist[u] + w < dist[v]) {         // relaxation
            dist[v] = dist[u] + w;
            pq.push({dist[v], v});           // duplicate-push instead of decrease-key
        }
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 743. Network Delay Time | Medium | Vanilla Dijkstra; answer = max settled distance (or −1 if any node unreached). The cleanest place to drill the template and the stale-skip line. |
| 1514. Path with Maximum Probability | Medium | Maximize a **product** of probabilities: flip to a max-heap, relax with `prob[u] * w > prob[v]`. Works because multiplying by values in (0,1] never increases — the same monotonicity Dijkstra's cut needs. (Equivalent: minimize −log p.) |
| 1976. Number of Ways to Arrive | Medium | Dijkstra + path counting: strict improvement copies the count, a tie adds it. The "augment with counts" move (LIS guide) on a weighted graph — mod 1e9+7. |
| 505. The Maze II | Medium | Edges are entire rolls until a wall: nodes = stopping cells, edge weight = roll length. Non-unit weights emerging from movement rules — the weight audit catching what looks like a grid-BFS. |
| 1786. Number of Restricted Paths | Medium | Dijkstra from the destination, then count strictly-decreasing-distance paths with memoized DFS over the distance field. Distance fields as *input to a second algorithm*. |
| 2045. Second Minimum Time | Hard | Track the best **two** distinct arrival times per node; traffic-light waiting modifies the edge cost as a function of departure time. Dijkstra generalized to k-best + time-dependent edges — a genuine boss fight. |
| 2577. Minimum Time to Visit Cell | Hard | You can oscillate to burn time, so arrival parity matters: adjust candidate times to match parity before relaxing. Time-parity inside the metric. |
| 1928. Minimum Cost to Reach Destination in Time | Hard | Two costs (fees to minimize, time as a budget): dist[node][time] — Dijkstra over an augmented state (bridge to Pattern 8). |

**Pitfalls:**
- Pair order: `{node, dist}` with `greater<>` orders by node id — silent, devastating, and the #1 Dijkstra bug. Distance first, always.
- Visited-at-enqueue (BFS habit) is **wrong** here — nodes improve while queued; only the pop is final. The reverse confusion (settle-at-pop in plain BFS) merely wastes time; this direction breaks correctness.
- Omitting the stale-skip keeps answers correct but lets dead entries expand — degraded complexity, and under questioning it reads as not understanding the lazy-deletion contract.
- Overflow: distances sum to V·maxW — `long long` the dist array before the judge teaches you to.

---

## Pattern 4: Twisted Metrics — Minimax, Maximin, and Friends

**Logic:** The "path cost" isn't a sum: minimize the **maximum** edge along the path (effort), maximize the **minimum** (bottleneck/safety), maximize a product (probability). Two interchangeable attacks: (a) **modified Dijkstra** — replace `dist[u] + w` with the metric's combiner (`max(dist[u], w)`, `min(dist[u], w)`, `dist[u] * w`) and flip the heap if maximizing; (b) **binary search on the answer + BFS/DFS feasibility** — "is there a path using only edges ≤ x?" is monotone in x (binary search guide, Pattern 5).

**Core insight — why it works:** Dijkstra's proof never needed *addition* — it needed two properties of the cost combiner: extending a path **never improves** its cost (monotonicity along paths: max can't decrease by adding an edge; a product of (0,1] factors can't grow), and the combiner is computed from the prefix cost and the edge alone. Any metric with those properties inherits the entire cut argument verbatim — that's why `max(dist[u], w)` slots into the template like it was born there. The binary-search attack works for an independent reason: the *answer* is monotone-feasible ("if a path exists under threshold x, it exists under any x' > x"), so threshold-feasibility is a first-true search with a BFS as the predicate. Knowing **both** and choosing by constraints (Dijkstra: one run, E log V; binary search: log(range) BFS runs, simpler code) is the senior move.

**Template (minimize the maximum edge — "effort"):**
```cpp
// identical skeleton to Pattern 3; only the relaxation changes:
long long cand = max(dist[u], (long long)w);   // path cost = worst edge so far
if (cand < dist[v]) {
    dist[v] = cand;
    pq.push({dist[v], v});
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 1631. Path With Minimum Effort | Medium | The flagship minimax: effort = max height-difference along the path. Solve it twice — modified Dijkstra and binary-search+BFS — and compare; interviewers often ask for the second after the first. |
| 778. Swim in Rising Water | Hard | Same metric on cell values: minimize the max cell entered. Third solution: union-find adding cells in height order until start and end connect — three guides meeting at one problem. |
| 1102. Path With Maximum Minimum Value | Medium | The maximin mirror: max-heap, relax with `min(dist[u], cellValue)`. Sign-flip discipline: derive it fresh; copying the minimax code and "flipping things" is the bug factory. |
| 1514. Path with Maximum Probability | Medium | Cross-listed: product metric, max-heap. The (0,1] bound is what preserves monotonicity — say it. |
| 2812. Find the Safest Path | Medium | Two-phase: multi-source BFS builds distance-to-nearest-thief, then maximin path over that field. Pipelines of patterns. |
| 1697. Checking Existence of Edge Length Limited Paths | Hard | Offline minimax for many queries: sort queries and edges, add edges with union-find as the threshold rises. The all-queries version where per-query Dijkstra dies — DSU as the batch minimax engine. |
| 1489. Critical & Pseudo-Critical Edges | Hard | MST-adjacent (minimax's global cousin): an edge is critical iff removing it raises MST cost. Listed as the bridge to the MST family — minimax single-pair paths and MSTs share the bottleneck structure (the minimax path cost between u, v equals the max edge on the MST path). |

**Pitfalls:**
- Initialization flips with the objective: minimax inits to +∞, maximin to 0/−∞, probability to 0 with `prob[src] = 1`. Mechanical sign-flipping without re-deriving inits is the recurring failure.
- For products, floating-point comparison tolerance is fine here (probabilities only multiply), but mention the −log transform as the principled alternative.
- The metric must be path-monotone — a metric like "sum of squares minus detour bonus" that can *improve* by extension voids the cut argument; check before trusting the template.

---

## Pattern 5: Negative Edges & Budgeted Moves — Bellman-Ford and Layered DP

**Logic:** Bellman-Ford abandons smart ordering: relax **every edge**, and repeat the full pass n−1 times. A pass with no successful relaxation means convergence — stop early. An n-th pass that *still* relaxes something proves a **negative cycle** reachable from the source. The interview-critical variant: cap the passes at **k** to answer "cheapest path using at most k edges/stops" — each pass extends optimal paths by at most one edge, so after k passes you have exactly the ≤-k-edge optimum.

**Core insight — why it works:** Induction on path length: after pass i, `dist[v]` is correct for every v whose shortest path uses ≤ i edges — because that path's last edge gets relaxed in pass i with an already-correct predecessor (≤ i−1 edges, previous pass). Shortest simple paths have ≤ n−1 edges, hence n−1 passes. This proof needs *no* assumption about weights — negatives are fine — which is exactly what Dijkstra's cut couldn't offer. The k-stops insight is sharper: the pass count **is** an edge budget, so the constraint that *breaks* Dijkstra (787: a pricier-but-shorter route may be the only legal one, but Dijkstra already settled the node via the cheap long route) is the thing Bellman-Ford *natively expresses*. One implementation subtlety makes or breaks the k-version: relax from a **frozen snapshot** of the previous pass (copy the dist array), or one pass can chain multiple edges and blow the budget.

**Template (k-constrained Bellman-Ford — LC 787):**
```cpp
vector<long long> dist(n, LLONG_MAX);
dist[src] = 0;
for (int pass = 0; pass <= k; pass++) {          // k stops = k+1 edges
    vector<long long> next = dist;                // FROZEN snapshot — essential
    bool changed = false;
    for (auto& [u, v, w] : edges)
        if (dist[u] != LLONG_MAX && dist[u] + w < next[v]) {
            next[v] = dist[u] + w;
            changed = true;
        }
    dist = move(next);
    if (!changed) break;                          // early convergence
}
return dist[dst] == LLONG_MAX ? -1 : dist[dst];
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 787. Cheapest Flights Within K Stops | Medium | *The* problem of this pattern, and a deliberate Dijkstra trap. Alternatives: BFS by layers, or Dijkstra over (node, stops) states (Pattern 8) — present Bellman-Ford as the cleanest, then name the others. |
| 743. Network Delay Time (BF version) | Medium | Re-solve with Bellman-Ford to feel the trade: simpler code, O(V·E) vs O(E log V). Knowing when "dumber but correct" is acceptable is judgment, and judgment is the rubric. |
| Negative-cycle detection (classic ask) | — | Run the n-th pass: any relaxation ⇒ reachable negative cycle. To find *affected* nodes, mark and propagate from n-th-pass relaxed vertices. Standard theory question; have it loaded. |
| Currency arbitrage (system-design-flavored classic) | — | Exchange rates as edges, take −log(rate): an arbitrage loop ⇔ a negative cycle. The famous real-world costume for negative-cycle detection — interviewers love it as a modeling probe. |
| 1928. Minimum Cost with Time Budget | Hard | Budgeted dimension done as DP over (node, time) — Bellman-Ford's pass-as-budget idea generalized from edge-count to arbitrary resources. |
| SPFA (mention) | — | Queue-optimized Bellman-Ford: only re-relax nodes whose dist changed. Fast in practice, O(V·E) worst case — know the name and the caveat; some interviewers ask. |

**Pitfalls:**
- Forgetting the frozen snapshot in the k-version: in-place relaxation lets a single pass use 2–3 fresh edges, answering the wrong question while passing small tests. The single most common 787 failure.
- "k stops" = k intermediate nodes = **k+1 edges** — the off-by-one is in the loop bound; pin it with the problem's example before coding.
- Relaxing from `dist[u] == INF` (INF + w overflows and "improves" things) — guard the read.
- Claiming Bellman-Ford "handles negative cycles": it *detects* them; shortest paths through one are −∞/undefined. Precision here is graded.

---

## Pattern 6: DAGs — Shortest (and Longest!) Paths via Topological Order

**Logic:** If the graph has no cycles, compute a topological order (Kahn's, BFS guide P6, or DFS finish order), then relax each node's outgoing edges **in that order**, once. Done — O(V + E), no heap, no passes, negative weights welcome. And because there are no cycles, flipping `min` to `max` computes **longest** paths just as easily.

**Core insight — why it works:** Every ordering trick in this guide exists to ensure `dist[u]` is final before u's edges are relaxed. On a DAG, topological order *guarantees that structurally*: all of u's predecessors precede it, so by the time u is processed, every path into it has been fully accounted — one relaxation sweep suffices, with zero assumptions about weights. The longest-path bonus is the deeper point: longest simple path is NP-hard *because of cycles* (you'd have to avoid revisits); a DAG has no cycles, so the hardness evaporates and max-relaxation is exactly as easy as min. This is why "critical path / minimum total time with prerequisites" problems — which are longest-path questions — are tractable: dependencies form a DAG by nature. Recognition cue: **prerequisites, build orders, version chains, strictly-increasing moves** — anything where edges can't loop.

**Template (DAG relaxation after Kahn's order):**
```cpp
// topo = topological order (from Kahn's algorithm)
vector<long long> dist(n, LLONG_MAX);
dist[src] = 0;
for (int u : topo)
    if (dist[u] != LLONG_MAX)
        for (auto [v, w] : adj[u])
            dist[v] = min(dist[v], dist[u] + w);   // flip to max for longest path
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 329. Longest Increasing Path in a Matrix | Hard | "Strictly increasing" makes the implicit move-graph a DAG → longest path is legal; the DFS+memo solution (DFS guide P5) *is* topological relaxation in recursive clothing. Connecting those two views is the insight. |
| 1136. Parallel Courses | Medium | Minimum semesters = longest chain in the prerequisite DAG = Kahn's with the level loop. Cross-listed from BFS; re-read it as a longest-path problem. |
| 2050. Parallel Courses III | Hard | Weighted version: finish[v] = max over prerequisites of finish[u], + duration[v] — the **critical path method** (CPM) verbatim, as used in real project scheduling. The flagship DAG-longest-path problem. |
| 1857. Largest Color Value in a Graph | Hard | DP over topological order carrying per-color counts; cycle (Kahn pops < n) ⇒ −1. Topo order as the backbone for arbitrary per-node DP. |
| 851. Loud and Rich | Medium | Propagate "quietest richer person" along the DAG in topo order — relaxation where the "distance" is an argmin over people. The combiner is pluggable; the order does the work. |
| 1786 (revisited) | Medium | The strictly-decreasing-distance condition turns the weighted graph into a DAG over distance values — DAG DP appearing *after* a Dijkstra. Pipelines again. |

**Pitfalls:**
- Verify acyclicity (Kahn count == n) before trusting the sweep — on a cyclic graph the "order" is partial and the relaxation quietly undercounts.
- Longest path with `dist` initialized to 0 instead of −∞ counts unreachable nodes as reachable — init discipline mirrors Pattern 4's.
- Don't reach for Dijkstra on a DAG out of habit: topological relaxation is simpler, faster, and handles negatives — the weight audit's cousin, the *structure audit*.

---

## Pattern 7: All Pairs — Floyd-Warshall

**Logic:** A V×V distance matrix, initialized with direct edges (and 0 on the diagonal), improved by allowing intermediate vertices one at a time: after processing k, `d[i][j]` is the shortest path using only intermediates from {0..k}. Three nested loops — **k outermost** — and 8 lines of code.

**Core insight — why it works:** It's interval-style DP on the *set of allowed intermediates*: the shortest i→j path using intermediates {0..k} either ignores k (previous value stands) or passes through k exactly once — splitting into i→k and k→j, both using only {0..k−1}, both already computed. That case split is exhaustive (a shortest path visits k at most once — no negative cycles), which is the entire proof. The k-outermost order is the DP's dependency order; swapping loops breaks the invariant (a classic gotcha question). When to choose it: V ≤ ~400 (V³ ≈ 6×10⁷), **many queries** between arbitrary pairs, or when the problem is *about* the full distance matrix (counting reachable sets, comparing all cities). Bonus powers: transitive closure (booleans + OR/AND), path *products* (399), and negative-edge tolerance (diagonal going negative ⇔ negative cycle).

**Template:**
```cpp
// d[i][j] = direct edge weight, INF if absent, 0 if i == j
for (int k = 0; k < n; k++)                  // intermediate — OUTERMOST, non-negotiable
    for (int i = 0; i < n; i++)
        for (int j = 0; j < n; j++)
            if (d[i][k] != INF && d[k][j] != INF)
                d[i][j] = min(d[i][j], d[i][k] + d[k][j]);
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 1334. Find the City With the Smallest Number of Neighbors | Medium | The canonical fit: n ≤ 100, and the question *is* the full matrix (count cities within a threshold from each). Anything else is over-engineering. |
| 1462. Course Schedule IV | Medium | Transitive closure: booleans, `reach[i][j] |= reach[i][k] && reach[k][j]`. Reachability queries in O(1) after O(V³) prep. |
| 399. Evaluate Division | Medium | Path **products** over variable-ratio edges: `d[i][j] = d[i][k] * d[k][j]`. The combiner is pluggable here too — Floyd-Warshall as a closure operator, not just a min-sum machine. |
| 1617. Number of Subtrees With Max Distance | Hard | Precompute all-pairs tree distances, then enumerate subsets checking diameters — FW as the preprocessing stage of a bitmask problem. |
| 2959. Number of Reachable Nodes After Closures | Hard | Repeated FW over small n with node subsets removed — brute force made viable purely by V³ being tiny. Constraint arithmetic as the green light. |
| 743 (revisited, third way) | Medium | n ≤ 100 means even single-source problems *can* go through FW. Solving one problem three ways (Dijkstra, BF, FW) is the best calibration exercise in this guide. |

**Pitfalls:**
- k in the middle or inside = wrong answers that pass tiny tests; the loop-order question is asked *because* the broken version looks plausible.
- INF + INF overflow: guard the reads (as in the template) or use INF = a safe large value like 1e18/4.
- Using FW for one source on a big sparse graph: V³ vs E log V can be a 1000× self-inflicted slowdown — let the query pattern pick the tool.

---

## Pattern 8: State-Augmented Graphs — Shortest Path Over Designed States

**Logic:** When legality or cost depends on more than the position — fuel left, stops used, keys held, parity of arrival time, which discounts remain — the graph you must search is over **augmented states** (position, extra). Pick the algorithm by the *edge weights of the augmented graph* (the same decision table as always), and let the state carry the constraint.

**Core insight — why it works:** This is the unifying pattern of the whole guide — and of the BFS guide's state-design section, now weighted. A constraint that breaks an algorithm's proof on the *original* graph often vanishes on the right augmented graph: 787 breaks Dijkstra because settling a city discards the pricier-short route — but over (city, stops-used) states, every node again has a single well-defined optimal cost and **Dijkstra's cut argument is restored**. The price is state-space size (V × budget), so the discipline is the same two-step as ever: prove the state *sufficient* (two situations with equal states have identical futures), then *count* it (states × branching must fit). Plus one new lever unique to weighted search — **dominance pruning**: skip state (v, r) if some recorded (v, r') with r' ≥ r already has cost ≤ yours; strictly-better-in-both states need never be expanded.

**Template (Dijkstra over (node, resource) states):**
```cpp
// dist[v][r] = min cost reaching v with r units of resource used/left
vector<vector<long long>> dist(n, vector<long long>(R + 1, LLONG_MAX));
priority_queue<tuple<long long,int,int>,
               vector<tuple<long long,int,int>>, greater<>> pq;  // (cost, node, res)
dist[src][0] = 0;
pq.push({0, src, 0});
while (!pq.empty()) {
    auto [d, u, r] = pq.top(); pq.pop();
    if (d > dist[u][r]) continue;
    for (auto [v, w, dr] : moves(u, r))           // dr = resource consumed
        if (r + dr <= R && d + w < dist[v][r + dr]) {
            dist[v][r + dr] = d + w;
            pq.push({dist[v][r + dr], v, r + dr});
        }
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 787 (third solution) | Medium | Dijkstra over (city, stops): the trap problem un-trapped by state design. Having BF, layered BFS, *and* this in your pocket — and saying which you'd write live — is the complete answer. |
| 1928. Min Cost to Reach Destination in Time | Hard | (city, time) states, minimize fees: the template verbatim with time as the resource. Prune times beyond the budget. |
| 864. Shortest Path to Get All Keys | Hard | (cell, keys-bitmask), unweighted → **BFS** over augmented states — the decision table applied *after* augmentation: weights, not difficulty, pick the algorithm. |
| 1293. Obstacle Elimination | Hard | (cell, eliminations-left) BFS with the dominance prune ("only revisit with strictly more budget"). The cleanest dominance specimen. |
| 2045. Second Minimum Time (revisited) | Hard | (node, arrival-rank) — the "extra" can be *which arrival this is*, not a resource. State design has more shapes than budgets. |
| 1129. Alternating Colors | Medium | (node, last-edge-color): two distances per node, plain BFS. The smallest possible augmentation — the right first problem here. |
| 2577. Min Time to Visit Cell (revisited) | Hard | (cell, time-parity) folded into the metric — augmentation absorbed into relaxation arithmetic rather than the state tuple. Two valid framings; compare them. |
| 1345. Jump Game IV | Hard | Same-value teleports: BFS where value-groups are virtual hub nodes — **clear each group's list after first use** or the graph is effectively dense and TLEs. Augmentation by *graph surgery* rather than by state. |
| LeetCode-classic "minimum cost with discounts" shapes (e.g., 2093) | Medium | (city, discounts-used) Dijkstra — the most common phone-screen version of this whole pattern. |

**Pitfalls:**
- Sufficiency failures surface as illegal answers (a path that used a key it never held) — re-run the "identical futures?" test per omitted field.
- Sizing failures surface as TLE/MLE: states × log(states) with the heap; do the multiplication aloud before committing.
- Dominance pruning must be *provably* safe (better in **every** dimension) — pruning on one dimension while worse in another silently discards optima.
- The 1345 hub trick: forgetting to clear the value-group turns O(V + E) into O(V²) — the single detail the problem exists to test.

---

## When Each Tool FAILS — The Boundary Map

| Situation | Broken tool & why | Repair |
|---|---|---|
| Weighted edges | BFS — layers ≠ cost | Dijkstra / 0-1 BFS per the audit |
| Negative edge | Dijkstra — the cut argument dies | Bellman-Ford; or DAG order if acyclic |
| Negative **cycle** on a path | Everything — distance is −∞ | Detect (BF's n-th pass) and report |
| "At most k moves/stops" | Dijkstra — settlement discards legal routes | BF with k passes / states (node, used) |
| Longest path, general graph | All of them — NP-hard | DAG? topological max-relax. Else: different problem |
| All pairs, large sparse V | Floyd-Warshall — V³ explodes | Dijkstra per source (E log V each) |
| Edge costs depend on arrival time/parity | Plain relaxation — cost isn't edge-local | Fold time into the state or the metric (2045, 2577) |
| Path cost isn't path-monotone | Modified Dijkstra — cut argument needs monotonicity | Binary search on answer + feasibility BFS, or rethink |

The one-sentence audit that routes everything: **what do edges cost, can anything be negative, is there a budget, and how many (source, target) pairs are asked?** Four answers, one algorithm.

---

## Cheat Sheet: Problem Phrase → Pattern

| Phrase in the problem | Pattern |
|---|---|
| "minimum number of moves/steps", unit costs | BFS (P1) |
| one action free, the other costs 1 | 0-1 BFS (P2) |
| "minimum cost/time", non-negative weights | Dijkstra (P3) |
| "minimize the maximum / maximize the minimum along a path" | Twisted metric (P4) or binary search |
| "maximum probability path" | Product-metric Dijkstra (P4) |
| "within k stops / at most k moves" | Bellman-Ford k passes / state augmentation (P5/P8) |
| arbitrage / negative costs / "can cost decrease forever" | Bellman-Ford + negative cycle (P5) |
| prerequisites with durations, "minimum total time" | DAG longest path / CPM (P6) |
| strictly increasing/decreasing moves | It's a DAG — topological DP (P6) |
| "for every pair of cities…", small n | Floyd-Warshall (P7) |
| cost/legality depends on fuel, keys, stops, parity | State augmentation (P8) |

---

## Complexity Summary

- BFS / 0-1 BFS: **O(V + E)**.
- Dijkstra (binary heap, duplicate-push): **O(E log V)**.
- Bellman-Ford: **O(V·E)**; k-constrained: **O(k·E)**; SPFA: fast in practice, same worst case.
- DAG relaxation: **O(V + E)** including the topological sort — and longest path at the same price.
- Floyd-Warshall: **O(V³)** time, O(V²) space — budget V ≤ ~400.
- State-augmented: **O(S log S + moves)** with S = states — count S first, always.

---

## Interview Tips

1. **Open with the weight audit, out loud.** "Edges are all cost 1 → BFS"; "non-negative reals → Dijkstra"; "there's a stop budget → Bellman-Ford or state Dijkstra." Ten seconds, and it demonstrates the exact judgment the question was chosen to test.
2. **Know each proof's load-bearing beam:** BFS — layers are cost; Dijkstra — non-negativity feeds the cut; Bellman-Ford — passes bound path length; DAG — topo order finalizes predecessors; FW — a shortest path uses intermediate k at most once. "When does it break?" is answered by naming the beam.
3. **787 is the calibration question** of this whole topic: explain *why* Dijkstra fails it (settlement discards a legal pricier-shorter route), then fix it two ways (BF passes; (node, stops) states). Candidates who can do that rarely get another shortest-path question.
4. **The relaxation line is sacred:** dist-first pair ordering, stale-skip, INF-read guards, `long long`. Four mechanical habits that eliminate ~90% of live-coding shortest-path bugs.
5. **Twisted metrics: re-derive, don't flip.** State the combiner, the init, and the heap direction from scratch for maximin/products — mechanical sign-flipping from the minimax code is the documented failure mode.
6. **Longest path: say the magic words.** "NP-hard in general, linear on DAGs — and this graph is a DAG because moves strictly increase." That sentence converts a scary ask into a Pattern 6 sweep.

---

## Suggested Practice Order

**Week 1 — the audit & BFS family:** 1091 → 752 → 542 → 994 → 1368 → 2290 → 934
**Week 2 — Dijkstra fluency:** 743 → 1514 → 505 → 1976 → 1631 → 778 → 1102
**Week 3 — beyond Dijkstra:** 787 (all three ways) → 743 (BF + FW re-solves) → 1334 → 1462 → 399 → 1136 → 2050
**Week 4 — boss fights:** 1129 → 1293 → 864 → 1928 → 1345 → 2045 → 2577 → 1697 → 2812

Good luck with the interviews!

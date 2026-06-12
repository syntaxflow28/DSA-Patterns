# Heap / Priority Queue — The Complete Interview Pattern Guide (C++)

A heap answers one question, extremely fast, forever: **"what is the current best element?"** — minimum or maximum — in O(log n) per insert/remove, with the best always readable in O(1). That single capability powers a surprising range of patterns: top-k selection, merging k streams, running medians, greedy scheduling, and the priority frontier inside Dijkstra and Prim. This guide covers them all: the logic, the **core insight** (why it's correct), C++ templates, strong problem sets with descriptive notes, and the pitfalls.

---

## What a Heap Actually Is (and Isn't)

A binary heap is a complete binary tree (stored as a flat array — children of index i live at 2i+1 and 2i+2) maintaining one invariant: **every parent beats its children** (≥ for max-heap, ≤ for min-heap). That's it. Two consequences define everything a heap can and cannot do:

- **The root is always the global best** — readable in O(1). Insertion and removal repair the invariant along a single root-to-leaf path → O(log n).
- **There is no order *between* siblings or across subtrees.** A heap is *not* sorted; it's the bare minimum structure needed to keep the best on top. This is why a heap of n elements builds in O(n) (`make_heap` / constructor-from-vector) while sorting costs O(n log n) — the heap maintains strictly less information.

**The mental model:** a heap is a *tournament in progress*. You always know the current champion; you know nothing else for sure. Ask it "what's the 3rd best?" and it shrugs — you'd have to pop twice.

**What a heap cannot do** (and what to use instead):
- **Search for / delete an arbitrary element**: O(n). If you need that, use `std::multiset` (ordered, O(log n) erase-by-value) or pair the heap with **lazy deletion** (Pattern 5).
- **Decrease-key** (update a stored priority): `std::priority_queue` doesn't support it. The universal workaround: push a fresh copy with the new priority and skip stale entries at pop time — this is how every real Dijkstra is written in C++.
- **Iterate in sorted order** without destroying it: popping everything is just heapsort, O(n log n).

## How to Recognize a Heap Problem

- The words "**k largest / k smallest / k closest / k most frequent / k-th**" — and k is much smaller than n.
- "**Merge k sorted** lists/arrays/streams."
- A **stream**: elements arrive one at a time and you must answer queries (median, k-th largest) *after every arrival* — you can't sort what hasn't arrived.
- **Repeatedly take the best available option**, possibly pushing back a modified version: simulations (stones smashing), schedulers (most-frequent task next), CPU/room assignment.
- "Minimize/maximize" with a **greedy that needs the current extreme** at every step: meeting rooms, refueling stops, connecting ropes.
- Graph shortest-path / MST where you repeatedly expand the **cheapest frontier** node.

**When a heap is the *wrong* tool — the comparison every interviewer probes:**

| Need | Right tool | Why not a heap |
|---|---|---|
| k-th smallest, all data in hand, one query | Quickselect, O(n) average | Heap's O(n log k) maintains stream-readiness you don't need |
| All n elements in sorted order | Sort, O(n log n) | Popping a heap n times *is* a sort, with worse constants |
| Sliding-window maximum | Monotonic deque, O(n) | Heap gives O(n log n) and needs lazy deletion for expiry |
| Membership / predecessor queries too | `std::multiset` / BST | Heap can't search |
| Only ever need the single min AND data is static | One O(n) scan | No structure needed at all |

The litmus test: **a heap earns its keep when the "best" element changes dynamically — through arrivals, removals, or modifications — and you need it again and again.**

## C++ `std::priority_queue` Survival Kit

```cpp
priority_queue<int> maxHeap;                            // MAX-heap by default!
priority_queue<int, vector<int>, greater<int>> minHeap; // min-heap

// Custom comparator (lambda). Counterintuitive but crucial:
// the comparator answers "does a have LOWER priority than b?"
// so 'greater' logic on top means SMALLEST on top.
auto cmp = [](const pair<int,int>& a, const pair<int,int>& b) {
    return a.second > b.second;          // smaller .second on top
};
priority_queue<pair<int,int>, vector<pair<int,int>>, decltype(cmp)> pq(cmp);

// pairs/tuples compare lexicographically — often you need no custom comparator:
priority_queue<pair<int,int>, vector<pair<int,int>>, greater<>> pq2; // min by .first

// O(n) heapify from existing data (vs n pushes = O(n log n)):
priority_queue<int> h(v.begin(), v.end());
```

Gotchas that cost interview minutes: the default is a **max**-heap (Python's `heapq` is min — don't cross-wire them); `pop()` returns `void`, so read `top()` first; the comparator direction is inverted relative to intuition (when on the fence, test with two elements).

---

## Pattern 1: Top-K — The Size-K Gatekeeper Heap

**Logic:** To track the k **largest** elements of a stream/array, keep a **min**-heap (yes, inverted!) capped at size k. For each element: push it; if size exceeds k, pop the minimum. Whatever survives is the top k, and the heap's root is the k-th largest — the *gatekeeper* every newcomer must beat.

**Core insight — why it works:** The inversion is the whole trick. You don't need the k largest *organized* — you need to defend their boundary, and the boundary of "k largest" is the **smallest member of the club**. A min-heap puts exactly that member on top, making the eviction decision O(1) to identify and O(log k) to execute. The payoff is in the complexity: O(n log k) time and O(k) memory instead of sort's O(n log n)/O(n) — decisive when n is huge or streaming (you never hold more than k elements), and when k ≪ n (log k vs log n). The same inversion mirrored: k smallest elements → max-heap of size k.

**Template (k largest / k-th largest):**
```cpp
priority_queue<int, vector<int>, greater<int>> heap;   // MIN-heap for k LARGEST
for (int x : nums) {
    heap.push(x);
    if ((int)heap.size() > k) heap.pop();   // evict the weakest club member
}
// heap.top() is the k-th largest; heap holds the top k
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 215. Kth Largest Element in an Array | Medium | The reference problem. Heap gives O(n log k); quickselect gives O(n) average — know both and say *when* each wins (stream / stability of worst case → heap; one-shot in-memory → quickselect). |
| 703. Kth Largest Element in a Stream | Easy | The streaming version, where quickselect is impossible and the size-k heap is *the* answer. The class holds the heap; `add()` is the template's loop body. |
| 347. Top K Frequent Elements | Medium | Count with a hashmap, then a size-k min-heap **ordered by frequency** over (count, value) pairs. O(n log k). The O(n) alternative — bucket sort by count — is a strong follow-up to mention. |
| 692. Top K Frequent Words | Medium | Same, but ties break alphabetically — which inverts *one* field of the comparator (smaller string = higher priority). Writing a two-field comparator with mixed directions under pressure is the real test. |
| 973. K Closest Points to Origin | Medium | Size-k **max**-heap on squared distance (k *smallest* → max-heap, the mirror). Use squared distance: exact integer math, no sqrt, no doubles. |
| 1046. Last Stone Weight | Easy | Not top-k but the simplest "repeatedly take the best" simulation: pop two largest, push the difference. A gentle first heap problem. |
| 451. Sort Characters by Frequency | Medium | Full ordering by frequency — here k = distinct count, so honestly a sort/bucket works as well; good for discussing when the heap *isn't* buying anything. |
| 658. Find K Closest Elements | Medium | Solvable with a size-k max-heap on |a−x|, but binary search + window is O(log n + k). A deliberate example where the heap is the *second-best* tool — practice saying so. |

**Pitfalls:**
- The inversion trips everyone at least once: k largest → **min**-heap, k smallest → **max**-heap. If your heap direction matches the superlative in the problem statement, you've probably got it backwards.
- Pushing then conditionally popping (size k+1 transiently) is clean and uniform; the "compare against top before pushing" micro-optimization invites bugs for zero asymptotic gain.
- For frequencies, heap entries must be (count, element) pairs ordered by count — pushing raw elements and "remembering" counts elsewhere doesn't survive comparator scrutiny.

---

## Pattern 2: K-Way Merge — The Frontier Heap

**Logic:** Given k sorted lists, the next element of the merged output is always the smallest among the k current *heads*. Keep those heads in a min-heap: pop the winner, emit it, and push its successor from the same list. Repeat until empty.

**Core insight — why it works:** Sortedness within each list means each list's unseen elements are all ≥ its head — so the k heads form a complete **candidate frontier**: the global next-smallest *must* be one of them, and nothing behind a head can leapfrog it. The heap maintains this k-element frontier at O(log k) per emission instead of O(k) linear scanning. Total: O(N log k) for N total elements — and the same frontier idea generalizes beautifully to *implicit* lists: in a sorted matrix, each row is a list; in "k smallest pairs (a,b)", row i is the virtual sorted list (a_i, b_0), (a_i, b_1), … that you never materialize. Recognizing hidden sorted lists is the pattern's real skill.

**Template (merge k sorted linked lists):**
```cpp
auto cmp = [](ListNode* a, ListNode* b) { return a->val > b->val; }; // min-heap
priority_queue<ListNode*, vector<ListNode*>, decltype(cmp)> pq(cmp);
for (auto* head : lists) if (head) pq.push(head);     // the k frontiers

ListNode dummy, *tail = &dummy;
while (!pq.empty()) {
    ListNode* node = pq.top(); pq.pop();              // global next-smallest
    tail = tail->next = node;
    if (node->next) pq.push(node->next);              // advance that frontier
}
return dummy.next;
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 23. Merge k Sorted Lists | Hard | The canonical instance (template above). Alternative: divide-and-conquer pairwise merging, same O(N log k) — mention it, then use the heap, which is shorter to write correctly. |
| 373. Find K Pairs with Smallest Sums | Medium | Virtual lists: seed the heap with (a_i, b_0) for the first k rows only; popping (i, j) pushes (i, j+1). Seeding *all* pairs is the trap — the frontier discipline is what keeps it O(k log k). |
| 378. Kth Smallest in a Sorted Matrix | Medium | Heap version: rows are the lists; pop k−1 times. O(k log n). The binary-search-on-value version is O(n log range) and wins for large k — being able to *compare* the two solutions is the interview gold here. |
| 786. K-th Smallest Prime Fraction | Medium | Virtual lists again: for each denominator, the fractions with increasing numerators form a sorted list. Same frontier machinery. |
| 632. Smallest Range Covering K Lists | Hard | Frontier with a twist: track the running **max** among heap members alongside the min on top; each pop-and-advance defines a candidate range [min, max]. The range can only improve by advancing the minimum — a lovely exchange argument. |
| 1439. K-th Smallest Sum of a Matrix With Sorted Rows | Hard | Fold rows in one at a time, keeping only the k smallest partial sums (Pattern 1's gatekeeper inside Pattern 2's merge) — patterns compose. |
| 264. Ugly Number II | Medium | Generate-and-merge: three virtual lists (×2, ×3, ×5 of previous uglies) merged via a heap with a seen-set for duplicates. The classic "merge streams you're generating yourself." |

**Pitfalls:**
- In 373/378-style virtual merges, push successors along **one axis only** (j+1, not also i+1) or use a visited set — double-pushing (i+1, j) and (i, j+1) from every pop duplicates states and silently corrupts counts.
- Empty lists must be filtered before seeding; `pq.push(nullptr)` compiles and then segfaults in the comparator.
- The comparator for pointers compares *values through* the pointers — forgetting and comparing addresses produces nondeterministic "sorted" output, a memorably confusing bug.

---

## Pattern 3: Two Heaps — Median Maintenance

**Logic:** Split the stream into halves by value: a **max**-heap `lo` holds the smaller half (its top = the largest small element), a **min**-heap `hi` holds the larger half (its top = the smallest large element). Keep them size-balanced (|lo| − |hi| ∈ {0, 1}). The median is then `lo.top()` (odd count) or the average of the two tops (even).

**Core insight — why it works:** The median is an *order statistic in the middle* — a full sort maintains far more order than needed, and a single heap maintains order around the wrong place (the extremes). Two heaps facing each other maintain order **exactly at the cut point**: every element in `lo` ≤ every element in `hi` (enforced by always routing through a top: push onto `lo`, move `lo`'s top to `hi`, rebalance), and the size constraint pins the cut at the middle. You've arranged O(1) access to precisely the two elements adjacent to the median, at O(log n) per insertion — the general principle being that **paired heaps give you a maintained boundary anywhere in the distribution**, not just at the ends (the same trick maintains any percentile by changing the size ratio).

**Template (median of a stream):**
```cpp
priority_queue<int> lo;                                  // max-heap: smaller half
priority_queue<int, vector<int>, greater<int>> hi;       // min-heap: larger half

void add(int x) {
    lo.push(x);                    // 1. always enter through lo
    hi.push(lo.top()); lo.pop();   // 2. push lo's best across the boundary
    if (hi.size() > lo.size()) {   // 3. rebalance sizes (lo may hold one extra)
        lo.push(hi.top()); hi.pop();
    }
}
double median() {
    if (lo.size() > hi.size()) return lo.top();
    return (lo.top() + (double)hi.top()) / 2.0;
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 295. Find Median from Data Stream | Hard | The canonical instance (template above). The three-step add — enter low, transfer top, rebalance — handles every case uniformly; ad-hoc if/else routing by comparing x to the tops also works but breeds edge-case bugs. Follow-ups ("values in [0,100]?" → counting array; "99% in range?" → hybrid) are standard, prep them. |
| 480. Sliding Window Median | Hard | Two heaps + **lazy deletion** (Pattern 5) for elements leaving the window, with balance tracked by *valid* counts, not heap sizes. Honest assessment: the `multiset`-with-mid-iterator solution is less error-prone in C++ — say so, then implement whichever you've drilled. |
| 1825. Finding MK Average | Hard | Three multisets/heaps partition the stream into bottom-k, middle, top-k with a maintained middle **sum** — the two-heap boundary idea generalized to two boundaries. |
| 2102. Sequentially Ordinal Rank Tracker | Hard | Track the i-th best ever as queries interleave with insertions: a two-heap boundary that only moves one direction. Clean drill for the "maintained cut point" concept. |
| 502. IPO | Hard | Also two heaps but *not* a boundary: a max-heap of affordable profits and a min-heap of locked projects by capital, transferring as capital grows. Listed here as a bridge to Pattern 4's greedy heaps. |

**Pitfalls:**
- Balance convention drift: decide `lo` may exceed `hi` by exactly one, then make *every* branch respect it. Mixed conventions yield medians off by half an element — annoyingly plausible-looking wrong answers.
- The naive add ("compare x with tops, route directly") must handle empty-heap cases explicitly; the always-route-through-`lo` version above never touches an empty top. Prefer it under pressure.
- Integer overflow in `(lo.top() + hi.top()) / 2` when values near INT_MAX — cast to double/long long *before* adding.

---

## Pattern 4: Greedy Scheduling & Simulation — "Best Available Right Now"

**Logic:** A loop of the form: *look at everything currently available, commit to the best one, possibly generate new availability, repeat.* The heap is what makes "best available" O(log n) instead of O(n) per round. Often paired with sorting events by time first: sort once to control *when* things become available; heap to choose *which* available thing to take.

**Core insight — why it works:** Two separate guarantees compose. The heap guarantees you always act on the true current optimum. The **greedy exchange argument** — which you must make per problem — guarantees that acting on the current optimum is safe: e.g., in Meeting Rooms II, if any room can host the next meeting, the one freeing earliest can (swapping assignments never increases room count); in Task Scheduler / Reorganize String, scheduling the most-frequent item first is safest because it has the tightest cooldown pressure — delaying it can only create conflicts later. The heap is the *engine*; the exchange argument is the *license*. In interviews, stating the license is worth as much as driving the engine.

**Template (minimum meeting rooms — sort by start, heap of end times):**
```cpp
sort(intervals.begin(), intervals.end());                // by start time
priority_queue<int, vector<int>, greater<int>> ends;     // min-heap of room end-times
for (auto& iv : intervals) {
    if (!ends.empty() && ends.top() <= iv[0])
        ends.pop();                                      // earliest-freeing room is free → reuse
    ends.push(iv[1]);                                    // occupy (new or reused) room
}
return ends.size();                                      // peak concurrency
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 253. Meeting Rooms II | Medium | The template problem. The heap's top is the only room worth checking — if even the earliest-freeing room is busy, all are. (The sort-both-endpoints "sweep line" solution is a sibling worth knowing.) |
| 621. Task Scheduler | Medium | Round-based: each cycle of n+1 slots, pop the most-frequent tasks, decrement, push back nonzero counts. The closed-form formula `(maxFreq−1)(n+1)+ties` answers faster, but the heap simulation generalizes when follow-ups break the formula. |
| 767. Reorganize String | Medium | Always place the most frequent character that isn't the one just placed: pop best, hold it out one round, push back. Impossibility check up front: `maxFreq > (n+1)/2`. |
| 1405. Longest Happy String | Medium | 767 with three letters and a "≤2 in a row" rule: pop the biggest; if blocked by the run rule, take the second biggest instead. The two-deep fallback is the pattern stressor. |
| 1834. Single-Threaded CPU | Medium | The full sort+heap composition: sort tasks by arrival; loop — feed all arrived tasks into a heap keyed (duration, index); if the heap's empty, jump time to the next arrival. A near-perfect simulation drill. |
| 2402. Meeting Rooms III | Hard | *Two* heaps: free rooms (min by room number) and busy rooms (min by end time), releasing from busy→free as time passes. Multi-heap state machines are where this pattern peaks. |
| 1942. Number of the Smallest Unoccupied Chair | Medium | Same two-heap shape as 2402 with chairs: free-chairs heap + occupied-until heap. Doing 2402 and 1942 together cements the structure. |
| 355. Design Twitter | Medium | News feed = k-way merge (Pattern 2) of followees' tweet lists inside a design problem — patterns hiding inside system-design-flavored questions. |

**Pitfalls:**
- Heap-only solutions to time-driven problems forget that availability changes with time: the sort-by-time outer loop is not optional. Structure as: advance time → release/admit into heap → pop best.
- Push-back amnesia in round-based simulations (621/767): you must *hold* popped items outside the heap until the round's constraint expires, then push back the survivors. Pushing back immediately reintroduces the conflict you were avoiding.
- Tie-breaking is usually load-bearing (smallest index, lexicographic, room number) — encode it as the comparator's second field, don't bolt it on afterward.

---

## Pattern 5: Lazy Deletion — Heaps That Tolerate Removal

**Logic:** `std::priority_queue` can't erase arbitrary elements — but most problems don't need *immediate* removal, only that removed elements never get *used*. So: record deletions in a hashmap (`toDelete[x]++` or "valid if version matches"), and purge at the only place it matters — the top. Before trusting `top()`, pop while it's stale.

**Core insight — why it works:** A stale element buried mid-heap is **harmless** — it influences no decision until it surfaces at the root. So deletion can be deferred until the exact moment the element would have mattered, where it's an O(log n) pop like any other. Amortized accounting stays clean: every element is pushed once and popped once, ever, so total work is O((pushes) log n) regardless of how long stale entries linger. The pattern is also the standard substitute for **decrease-key**: instead of updating a stored priority, push a fresh (better) copy and let the old one die at the top — this is exactly why C++ Dijkstra implementations push duplicates and check `if (d > dist[u]) continue;` at pop.

**Template (heap with deferred removals):**
```cpp
priority_queue<int> pq;
unordered_map<int,int> stale;            // value → pending delete count

void remove(int x) { stale[x]++; }       // O(1) "deletion"
int  top() {
    while (!pq.empty() && stale[pq.top()] > 0)
        { stale[pq.top()]--; pq.pop(); } // purge corpses only at the surface
    return pq.top();
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 480. Sliding Window Median | Hard | The flagship: window departures become lazy deletes; the size *balance* between the two heaps must count only valid elements — track `validLo/validHi` integers alongside. |
| 239. Sliding Window Maximum (heap version) | Hard | Push (value, index); at top, pop while `index <= right − k`. Index-based staleness needs no hashmap at all. O(n log n) — then explain why the monotonic deque beats it at O(n). |
| 1172. Dinner Plate Stacks | Hard | A heap of "leftmost possibly-pushable stack indices" that goes stale as stacks fill/empty — validate on pop. Lazy deletion inside a design problem. |
| 2349. Design a Number Container System | Medium | Per-number min-heap of indices; an index is valid iff the live map still says `index → number`. The version-check flavor of staleness (no counting needed). |
| 716. Max Stack | Hard | Two structures (stack + heap, or two with shared tombstones) where popping from either lazily invalidates entries in the other. A pure lazy-deletion design exercise. |
| Dijkstra (see Pattern 6) | — | `if (d > dist[u]) continue;` *is* lazy deletion — naming that connection in an interview lands well. |

**Pitfalls:**
- Purging anywhere except immediately before reading `top()` — centralize it in one helper or the one call site you forgot becomes the bug.
- In two-heap + lazy setups (480), heap `.size()` lies (it counts corpses): all balance logic must use your own valid-element counters.
- Same-value collisions: deleting "one 5" when several are buried needs *count* semantics (the map above), not a boolean set.

---

## Pattern 6: Heap as the Graph Frontier — Dijkstra, Prim, and Friends

**Logic:** Grow a "settled" region one node at a time: a min-heap holds the **frontier** — reachable-but-unsettled nodes keyed by tentative cost. Pop the cheapest, settle it permanently, relax its edges (pushing improved tentative costs), repeat. Dijkstra keys by path distance; Prim by single-edge cost; the "swim/effort" variants by path-maximum.

**Core insight — why it works:** The settlement is justified by a cut argument: when the frontier's cheapest node u pops with tentative cost d, **no future path can beat d** — any other route to u must exit the settled region through some frontier node with cost ≥ d and (with non-negative edges) only grows from there. So greedy settlement is *permanently* correct, never revisited. The heap makes "cheapest frontier node" O(log n), and lazy deletion (Pattern 5) handles the missing decrease-key: re-push improved entries, discard stale ones at pop with one guard line. Note the load-bearing assumption — **non-negative weights**; with negatives the cut argument dies and you need Bellman-Ford.

**Template (Dijkstra with lazy stale-skip):**
```cpp
vector<long long> dist(n, LLONG_MAX);
priority_queue<pair<long long,int>, vector<pair<long long,int>>, greater<>> pq;
dist[src] = 0; pq.push({0, src});
while (!pq.empty()) {
    auto [d, u] = pq.top(); pq.pop();
    if (d > dist[u]) continue;            // stale entry — lazy deletion in action
    for (auto [v, w] : adj[u])
        if (dist[u] + w < dist[v]) {
            dist[v] = dist[u] + w;
            pq.push({dist[v], v});        // re-push instead of decrease-key
        }
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 743. Network Delay Time | Medium | Vanilla Dijkstra; answer = max settled distance. The cleanest place to drill the template and the stale-skip line. |
| 1631. Path With Minimum Effort | Medium | Dijkstra with a twisted metric: path cost = **max** edge along it, so relax with `max(effort[u], w)` instead of `+`. Works because max, like +, never decreases along a path — generalizing the metric is the lesson. |
| 778. Swim in Rising Water | Hard | Same max-metric on a grid; equivalently binary-search the water level + BFS (Pattern 5 of the binary search guide!). Two guides intersecting at one problem. |
| 1584. Min Cost to Connect All Points | Medium | Prim's MST: identical loop, but the key is the single cheapest edge into the settled set, not cumulative distance. Articulating *that one-line difference* from Dijkstra is a classic interview checkpoint. |
| 787. Cheapest Flights Within K Stops | Medium | A trap: the stop limit breaks Dijkstra's "settled forever" guarantee (a costlier-but-shorter path may be the only legal one). Use Bellman-Ford / BFS-by-layers — knowing where the cut argument fails is the point. |
| 407. Trapping Rain Water II | Hard | Min-heap over the boundary wall, always processing the lowest wall inward: water at a cell is bounded by the *minimum of the maximum* wall on any escape path — Dijkstra-style frontier reasoning on water. Spectacular hard. |
| 2812. Find the Safest Path in a Grid | Medium | Maximize the minimum distance-to-thief along the path: multi-source BFS for distances, then a **max**-heap frontier (or binary search + BFS). The maximin mirror of 1631. |

**Pitfalls:**
- Omitting `if (d > dist[u]) continue;` — still *correct*, but complexity quietly degrades as stale entries get fully expanded; under interview questioning this reads as not understanding the algorithm.
- Pair ordering: `{distance, node}` with distance first, or your `greater<>` heap orders by node id. Silent and devastating.
- Settling on push instead of pop ("mark visited when enqueued") is fine for BFS but **wrong for Dijkstra** — a node can be enqueued multiple times with improving costs; only the pop is final.
- Negative edges: Dijkstra's correctness proof needs non-negativity. If the problem allows negatives, that *is* the question being asked.

---

## Pattern 7: Regret Greedy — Commit Now, Keep the Undo Button

**Logic:** Process items in a forced order (sorted by deadline, position, capital threshold). Greedily **commit to everything**; when a constraint finally breaks (out of fuel, over time budget), use a heap of past commitments to retract the *worst* one. The heap stores your regrets.

**Core insight — why it works:** Pure greedy fails these problems because the best choice depends on the future ("should I refuel here?" depends on stations not yet seen). The repair: make the cheap choice now, but keep all alternatives alive in a heap so the decision is **reversible at exactly its marginal cost**. The exchange argument then runs in reverse — when forced to backtrack, swapping out the single worst commitment (largest course duration, smallest skipped fuel) is provably optimal, because any solution keeping the worst item can be improved by the swap. Sort + commit + heap-of-regrets turns lookahead into hindsight, which is computable. This pattern composes a sorted event order (binary-search-guide thinking) with a heap boundary — and it's the standard killer of "greedy or DP?" hesitation on problems like 871 and 630.

**Template (minimum refueling stops):**
```cpp
sort(stations.begin(), stations.end());          // by position
priority_queue<long long> fuelOptions;           // max-heap: skipped stations' fuel
long long reach = startFuel; int stops = 0, i = 0;
while (reach < target) {
    while (i < (int)stations.size() && stations[i][0] <= reach)
        fuelOptions.push(stations[i++][1]);      // commit: note every passable station
    if (fuelOptions.empty()) return -1;          // stranded, nothing to regret
    reach += fuelOptions.top(); fuelOptions.pop(); // retract best skipped option
    stops++;
}
return stops;
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 871. Minimum Number of Refueling Stops | Hard | The template. Reframe makes it click: "never refuel; when stranded, retroactively decide you *did* stop at the best station already passed." Each pop = one retroactive stop. |
| 630. Course Schedule III | Hard | Sort by deadline; greedily take every course, accumulating time; when over a deadline, drop the **longest** course taken so far (max-heap of durations). The drop can't hurt: removing the longest frees the most time at the cost of one course — the canonical exchange. |
| 502. IPO | Hard | Two-phase: min-heap of projects locked by capital, max-heap of affordable profits; each round, unlock everything affordable, take the single best profit. Regret-free variant — commitments here are irreversible but the unlock/choose split is the same machinery. |
| 1642. Furthest Building You Can Reach | Medium | Spend ladders greedily on every climb (heap of climbs); when a new climb arrives and ladders run out, retract the ladder from the **smallest** climb and pay bricks instead. Min-heap of size ≤ ladders — Pattern 1's gatekeeper as the regret store. |
| 1353. Max Events That Can Be Attended | Medium | Day-by-day: admit events that have started (min-heap by end day), attend the one ending soonest, lazily discard expired ones. Earliest-deadline-first with a heap. |
| 2813. Maximum Elegance of a K-Length Subsequence | Hard | Take top-k by profit, then walk alternatives retracting the worst duplicate-category item (min-stack of regrets). Final-boss tier of the same shape. |

**Pitfalls:**
- Choosing the wrong regret extreme: retract the **max** duration (630) but the **min** climb (1642) — the heap direction comes from re-deriving the exchange argument per problem, never from pattern-matching the previous one.
- Forgetting the stranded check (empty regret heap with the constraint still violated → return −1/impossible) — the failure path is part of the problem.
- These problems *smell* like DP (and 871 has an O(n²) DP). The heap greedy is asymptotically better — offering it as "the optimization over the DP" is a strong interview arc.

---

## When a Heap FAILS — Know the Boundary

| Situation | Why the heap breaks | Use instead |
|---|---|---|
| Need to find/erase arbitrary elements fast, frequently | O(n) search; lazy deletion only helps when staleness surfaces naturally | `std::multiset` / indexed tree |
| Sliding-window max/min | Heap is O(n log n) + staleness bookkeeping | Monotonic deque, O(n) |
| One-shot k-th element, all data present | Heap maintains stream-readiness you don't need | Quickselect, O(n) average |
| Predecessor/successor or range queries | Heap has no lateral order | BST / `multiset` / sorted vector + binary search |
| Priorities mutate constantly in place | No decrease-key; duplicate-push floods the heap if updates ≫ pops | Indexed heap, `multiset`, or rethink |
| Negative edges in shortest path | Dijkstra's cut argument dies | Bellman-Ford / SPFA |

Litmus test in one sentence: **a heap is the right tool when you repeatedly need the current extreme of a dynamically changing set — and nothing more than the extreme.** Need lateral visibility (search, ranges, neighbors)? Reach for an ordered structure instead.

---

## Cheat Sheet: Problem Phrase → Pattern

| Phrase in the problem | Pattern |
|---|---|
| "k largest / smallest / closest / most frequent" | Size-k gatekeeper heap (inverted direction!) |
| "k-th largest **in a stream**" | Size-k heap, kept alive across queries |
| "merge k sorted …" | K-way merge frontier |
| "k smallest pairs / matrix entries" | K-way merge over virtual lists |
| "median from a data stream" | Two heaps facing each other |
| "minimum rooms / machines / chairs needed" | Sort by time + min-heap of end times |
| "schedule tasks with cooldown / no two adjacent" | Max-heap by frequency + hold-out |
| "shortest path / minimize the maximum along a path" | Dijkstra-style frontier heap |
| "maximum events / courses before deadlines", "minimum refuels" | Regret greedy (sort + heap of undo options) |
| "sliding window maximum" | **Not** a heap — monotonic deque |

---

## Complexity Summary

- Build heap from n elements: **O(n)** (constructor), vs O(n log n) for n pushes.
- Push / pop: **O(log n)**; top: **O(1)**.
- Top-k over n elements: **O(n log k)** time, **O(k)** space.
- K-way merge of N total elements, k lists: **O(N log k)**.
- Two-heap median: **O(log n)** per insert, **O(1)** per query.
- Dijkstra / Prim with binary heap: **O(E log V)**.
- Lazy deletion: amortized free — every element still pushed once, popped once.

---

## Interview Tips

1. **Say the direction out loud before coding:** "k largest, so min-heap of size k — the root is the gatekeeper." The inversion is the single most common heap mistake; narrating it inoculates you.
2. **State the exchange argument for greedy-heap solutions.** "The room freeing earliest is the only one worth checking — if it's busy, all are." The heap is the engine; the argument is the license; interviewers grade the license.
3. **Know your escape hatches:** quickselect beats top-k when data is static and in memory; deque beats heap for window extremes; multiset beats heap when you need search or true deletion. Choosing *against* the heap correctly scores as highly as using it well.
4. **Lazy deletion is one idea, everywhere:** tombstone map + purge-at-top, and Dijkstra's `if (d > dist[u]) continue;` is the same move. Connect them explicitly.
5. **C++ specifics that bite:** default `priority_queue` is a MAX-heap; comparator semantics are inverted ("lower priority"); `pop()` returns void; pair ordering is lexicographic (put the key first); use O(n) heapify when data is already in hand.
6. **Complexity arithmetic:** if k ≪ n, log k ≪ log n — say "n log k, not n log n" precisely. If k ≈ n, the heap advantage evaporates and sorting is honest — saying *that* is also worth points.

---

## Suggested Practice Order

**Week 1 — top-k fluency:** 1046 → 703 → 215 → 973 → 347 → 692 → 451
**Week 2 — merge & two heaps:** 23 → 373 → 378 → 264 → 295 → 1825
**Week 3 — greedy scheduling:** 253 → 621 → 767 → 1405 → 1834 → 1942 → 2402
**Week 4 — boss fights:** 743 → 1631 → 1584 → 407 → 871 → 630 → 502 → 1642 → 480

Good luck with the interviews!

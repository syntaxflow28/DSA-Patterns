# Tricks — The Moves That Unlock Problems You Already Understand

The pattern guides in the parent folder answer *which technique*. This folder answers a different and more painful question: **you already know it's interval DP, and you still can't solve it.**

That gap is real and it is not a knowledge gap. LC 312 Burst Balloons is the canonical case — you can correctly identify it as interval DP in thirty seconds and still get nowhere for an hour, because the recurrence only exists if you ask *which balloon bursts **last***. Ask which bursts *first* and the subproblems have ragged, mutually-dependent boundaries and no recurrence can be written at all. Nothing about that move is "dynamic programming knowledge." It's a reframing move, and it transfers: the same move solves 546, 1000, and 1039.

There are perhaps fifty such moves across the entire top interview set. They are what separates people who recognize a pattern from people who finish the problem. This folder names them, explains *why* each works, and lists every well-known problem where the move is the actual unlock.

---

## When You're Stuck — Run This Checklist

Ten minutes in with no traction, work down this list. Each question maps to a trick family. Most stuck problems fall to one of the first six.

| # | Ask yourself | If yes, go to |
|---|---|---|
| 1 | Would fixing the **last** step make the subproblems independent? | [Trick 1, reversal](reversal-and-time.md) |
| 2 | Does this cell depend on things **after** it? Try a right-to-left pass. | [Trick 2, reversal](reversal-and-time.md) |
| 3 | Is the process **destructive** (removals, collapses)? Play it backwards. | [Trick 4, reversal](reversal-and-time.md) |
| 4 | Can I **separate i and j** algebraically so a running best works? | [Trick 1, algebra](algebraic-reformulation.md) |
| 5 | Is "exactly K" blocking me? Subtract two "at most" passes. | [Trick 3, algebra](algebraic-reformulation.md) |
| 6 | Is the **complement** easier? Minimum removed = n − maximum kept. | [Trick 5, algebra](algebraic-reformulation.md) |
| 7 | Is the value too big to index but the **answer** small? Swap them. | [Trick 1, state design](state-design.md) |
| 8 | What do the **constraints** say? n ≤ 20 → bitmask. n ≤ 40 → meet in the middle. | [State design](state-design.md) |
| 9 | Am I enumerating subarrays? Ask what each **element contributes** instead. | [Trick 1, counting](counting-by-contribution.md) |
| 10 | Is the same cell reachable in genuinely **different situations**? Node = (cell, situation). | [Trick 2, graph modeling](graph-modeling.md) |
| 11 | Does the answer **bend** through a node while the parent can only extend? Return one, record another. | [Trick 1, recursion](recursion-reframing.md) |
| 12 | Is the answer suspiciously **small or formula-shaped**? Find the invariant, or bound then construct. | [Invariants](invariants-and-proofs.md) |
| 13 | Does it demand **O(1) space**? The input is your scratch space. | [Trick 1, representation](representation-tricks.md) |
| 14 | Do I need a **lookup key** that doesn't exist? Invent a canonical form. | [Trick 2, representation](representation-tricks.md) |

Two meta-rules worth as much as the list:

- **The constraints are the setter telling you the intended solution.** `n <= 20` is not a courtesy, it is an instruction to bitmask. `n <= 40` means meet in the middle. An explicit "O(1) extra space" means the answer is hidden inside the input array.
- **If the answer is a formula, the interview is the proof, not the code.** Writing code before you have the argument is the failure mode for the entire invariants family.

---

## The Eight Families

| File | What it covers | Flagship problem |
|---|---|---|
| [reversal-and-time.md](reversal-and-time.md) | Running the problem backwards in time, order, or direction | 312 Burst Balloons |
| [algebraic-reformulation.md](algebraic-reformulation.md) | Rewriting the constraint until the algorithm is obvious | 1014 Best Sightseeing Pair |
| [state-design.md](state-design.md) | You know it's DP; the state is the puzzle | 300 LIS in O(n log n) |
| [counting-by-contribution.md](counting-by-contribution.md) | Changing *what you iterate over* | 907 Sum of Subarray Minimums |
| [graph-modeling.md](graph-modeling.md) | It's a graph problem and nobody said so | 815 Bus Routes |
| [recursion-reframing.md](recursion-reframing.md) | The recursion's *contract* is the hard part | 124 Binary Tree Max Path Sum |
| [invariants-and-proofs.md](invariants-and-proofs.md) | Five lines of code, all the difficulty in the argument | 621 Task Scheduler |
| [representation-tricks.md](representation-tricks.md) | Changing how data is stored, not how it's processed | 289 Game of Life |

---

## Master Index — Problem to Trick

Every problem below is one where knowing the pattern is *not sufficient*. Sorted by number. Use it in reverse too: after failing a problem, look up which move you missed.

| # | Problem | The trick you needed | Family |
|---|---|---|---|
| 2 | Add Two Numbers | Dummy head sentinel kills the "first node" special case | [state](state-design.md) |
| 11 | Container With Most Water | Move the limiting side — the extreme element forces the argument | [invariants](invariants-and-proofs.md) |
| 19 | Remove Nth Node From End | Dummy head, so deleting the head isn't special | [state](state-design.md) |
| 28 | First Occurrence in a String | Rolling hash (or KMP) | [representation](representation-tricks.md) |
| 41 | First Missing Positive | Place each value at its own index — the array is the hash table | [representation](representation-tricks.md) |
| 42 | Trapping Rain Water | Two passes: prefix max and suffix max | [reversal](reversal-and-time.md) |
| 46 / 47 | Permutations I / II | Sort, then `i > start && a[i] == a[i-1]` skip | [recursion](recursion-reframing.md) |
| 49 | Group Anagrams | Canonical form as the hash key | [representation](representation-tricks.md) |
| 73 | Set Matrix Zeroes | Store the flags in row 0 and column 0 | [representation](representation-tricks.md) |
| 78 / 90 | Subsets I / II | Same deduplication rule | [recursion](recursion-reframing.md) |
| 84 | Largest Rectangle in Histogram | Trailing sentinel flushes the stack | [state](state-design.md) |
| 98 | Validate BST | Pass the allowed range *down*, don't compare locally | [recursion](recursion-reframing.md) |
| 110 | Balanced Binary Tree | Return height and validity together — O(n), not O(n log n) | [recursion](recursion-reframing.md) |
| 124 | Binary Tree Maximum Path Sum | Return the single branch, record the bend | [recursion](recursion-reframing.md) |
| 129 | Sum Root to Leaf Numbers | Path facts flow down as arguments | [recursion](recursion-reframing.md) |
| 131 | Palindrome Partitioning | Backtracking with the prefix check | [recursion](recursion-reframing.md) |
| 164 | Maximum Gap | Pigeonhole: with n−1 buckets the answer can't be inside one | [invariants](invariants-and-proofs.md) |
| 179 | Largest Number | The exchange argument *is* the comparator: `a+b > b+a` | [invariants](invariants-and-proofs.md) |
| 187 | Repeated DNA Sequences | Rolling hash / bit-packed window | [representation](representation-tricks.md) |
| 188 / 123 | Best Time to Buy and Sell Stock IV / III | Transactions used and holding must both be in the state | [state](state-design.md) |
| 200 | Number of Islands | Pack `(r,c)` into one integer key | [representation](representation-tricks.md) |
| 207 / 210 | Course Schedule I / II | "Prerequisite" is the topological-order fingerprint | [graph](graph-modeling.md) |
| 218 | The Skyline Problem | Coordinate compression plus event sweep | [representation](representation-tricks.md) |
| 233 | Number of Digit One | Digit DP with `tight` | [state](state-design.md) |
| 236 | Lowest Common Ancestor | Bubble up — the node where two non-null results meet | [recursion](recursion-reframing.md) |
| 238 | Product of Array Except Self | Left pass then right pass | [reversal](reversal-and-time.md) |
| 249 | Group Shifted Strings | Canonical form = the difference pattern | [representation](representation-tricks.md) |
| 253 | Meeting Rooms II | Difference array / sweep line | [representation](representation-tricks.md) |
| 269 | Alien Dictionary | The hidden graph is the letter ordering | [graph](graph-modeling.md) |
| 287 | Find the Duplicate Number | Pigeonhole forces a cycle → Floyd; array is read-only so no marking | [invariants](invariants-and-proofs.md) |
| 289 | Game of Life | Two bits per cell: old in bit 0, new in bit 1 | [representation](representation-tricks.md) |
| 292 | Nim Game | The invariant is `n % 4` | [invariants](invariants-and-proofs.md) |
| 300 | Longest Increasing Subsequence | Swap state and value: `tails[len] = min tail` | [state](state-design.md) |
| 305 | Number of Islands II | Union-find only merges — so add, never remove | [reversal](reversal-and-time.md) |
| 309 / 714 | Stock with Cooldown / with Fee | The unfinished commitment goes in the state | [state](state-design.md) |
| **312** | **Burst Balloons** | **Which balloon bursts LAST — then both boundaries are fixed** | [reversal](reversal-and-time.md) |
| 315 | Count of Smaller Numbers After Self | Compress coordinates, sweep with a BIT | [counting](counting-by-contribution.md) |
| 327 | Count of Range Sum | Same, over prefix sums | [counting](counting-by-contribution.md) |
| 332 | Reconstruct Itinerary | "Use every edge once" = Eulerian path | [graph](graph-modeling.md) |
| 337 | House Robber III | Return the (take, skip) pair instead of recursing twice | [recursion](recursion-reframing.md) |
| 354 | Russian Doll Envelopes | Sort to erase one dimension, then LIS on the other | [state](state-design.md) |
| 370 | Range Addition | Difference array | [representation](representation-tricks.md) |
| 375 | Guess Number Higher or Lower II | Interval DP over the split point | [state](state-design.md) |
| 399 | Evaluate Division | Ratios are weighted edges | [graph](graph-modeling.md) |
| 406 | Queue Reconstruction by Height | Exchange argument gives the sort order | [invariants](invariants-and-proofs.md) |
| 435 / 452 | Non-overlapping Intervals / Min Arrows | Minimum removed = n − maximum kept | [algebra](algebraic-reformulation.md) |
| 437 | Path Sum III | Prefix-sum map along the path, undone on the way up | [recursion](recursion-reframing.md) |
| 442 / 448 | Find All Duplicates / Disappeared Numbers | Sign-marking: sign carries presence, magnitude carries value | [representation](representation-tricks.md) |
| 493 | Reverse Pairs | Inversion counting | [counting](counting-by-contribution.md) |
| 496 / 503 / 739 | Next Greater Element family | Scan from the right | [reversal](reversal-and-time.md) |
| 516 | Longest Palindromic Subsequence | State is a range | [state](state-design.md) |
| 523 / 974 | Continuous Subarray Sum / Sums Divisible by K | Remap: prefix modulo becomes the key | [algebra](algebraic-reformulation.md) |
| 525 | Contiguous Array | Remap 0 to −1, then it's "prefix sums equal" | [algebra](algebraic-reformulation.md) |
| 526 | Beautiful Arrangement | Bitmask the used set | [state](state-design.md) |
| 542 / 1162 / 286 | 01 Matrix / As Far from Land / Walls and Gates | All sources in the queue at once = one virtual super-source | [graph](graph-modeling.md) |
| 543 / 687 / 1372 | Diameter / Univalue Path / ZigZag | Return one thing, record another | [recursion](recursion-reframing.md) |
| 546 | Remove Boxes | Last operation, plus a third state dimension | [reversal](reversal-and-time.md) |
| 547 / 684 / 721 / 839 | Provinces / Redundant Connection / Accounts Merge / Similar Strings | "Same as" is a union-find edge | [graph](graph-modeling.md) |
| 560 | Subarray Sum Equals K | Fix the right endpoint, query the map before inserting | [algebra](algebraic-reformulation.md) |
| 600 / 902 / 1012 | Digit-counting problems | Digit DP needs both `tight` and `started` | [state](state-design.md) |
| 621 | Task Scheduler | Bound from the most frequent task, then construct | [invariants](invariants-and-proofs.md) |
| 630 | Course Schedule III | Exchange argument → regret heap | [invariants](invariants-and-proofs.md) |
| 645 | Set Mismatch | Index-as-hash | [representation](representation-tricks.md) |
| 652 | Find Duplicate Subtrees | Serialize with nulls so structure is unambiguous | [representation](representation-tricks.md) |
| 691 / 698 / 1434 | Stickers / Partition to K Subsets / Wear Different Hats | n ≤ 20 means bitmask | [state](state-design.md) |
| 694 | Number of Distinct Islands | Canonical form = the movement path from the entry cell | [representation](representation-tricks.md) |
| 713 / 1358 | Product Less Than K / Substrings with ABC | Fix the right endpoint | [counting](counting-by-contribution.md) |
| 753 | Cracking the Safe | de Bruijn sequence = Eulerian path | [graph](graph-modeling.md) |
| 767 | Reorganize String | Bound then construct | [invariants](invariants-and-proofs.md) |
| 780 | Reaching Points | Backwards the moves are forced; forwards they branch | [reversal](reversal-and-time.md) |
| 785 / 886 | Is Graph Bipartite / Possible Bipartition | "Two hostile groups" = 2-colouring | [graph](graph-modeling.md) |
| 787 | Cheapest Flights Within K Stops | Node = (city, stops used) | [graph](graph-modeling.md) |
| 802 | Find Eventual Safe States | Reverse the edges, then Kahn | [reversal](reversal-and-time.md) |
| 803 | Bricks Falling When Hit | Replay the hits backwards as unions | [reversal](reversal-and-time.md) |
| 805 / 1755 / 2035 | Same Average / Closest Subsequence Sum / Minimize Sum Difference | n ≈ 40 means meet in the middle | [state](state-design.md) |
| 815 | Bus Routes | Make each *route* a node, not each stop | [graph](graph-modeling.md) |
| 828 / 891 | Unique Characters of All Substrings / Subsequence Widths | Count each element's contribution | [counting](counting-by-contribution.md) |
| 847 | Shortest Path Visiting All Nodes | Node = (city, visited bitmask) | [graph](graph-modeling.md) |
| 850 | Rectangle Area II | Coordinate compression | [representation](representation-tricks.md) |
| 864 / 1293 | Get All Keys / Obstacles Elimination | Node = (cell, keys) or (cell, budget) — visited must key on both | [graph](graph-modeling.md) |
| 870 | Advantage Shuffle | Exchange argument | [invariants](invariants-and-proofs.md) |
| 877 / 1025 | Stone Game / Divisor Game | Parity invariant, not DP | [invariants](invariants-and-proofs.md) |
| 907 / 2104 | Sum of Subarray Minimums / Ranges | Contribution × span, with the strict/non-strict tie rule | [counting](counting-by-contribution.md) |
| 930 / 992 / 1248 | Binary Subarrays / K Distinct / Nice Subarrays | exactly(K) = atMost(K) − atMost(K−1) | [algebra](algebraic-reformulation.md) |
| 943 | Find the Shortest Superstring | Bitmask DP over orderings | [state](state-design.md) |
| 968 | Binary Tree Cameras | Three-state return, not a boolean | [recursion](recursion-reframing.md) |
| 979 | Distribute Coins in Binary Tree | Postorder: return the net flow through the edge | [recursion](recursion-reframing.md) |
| 990 | Satisfiability of Equality Equations | Union all equalities *before* testing any inequality | [graph](graph-modeling.md) |
| 991 | Broken Calculator | Backwards, halving is forced | [reversal](reversal-and-time.md) |
| 1000 / 1039 | Merge Stones / Triangulation | Last merge, last triangle | [reversal](reversal-and-time.md) |
| 1014 | Best Sightseeing Pair | Split into `(v[i]+i)` and `(v[j]−j)` | [algebra](algebraic-reformulation.md) |
| 1044 | Longest Duplicate Substring | Binary search the length + rolling hash | [representation](representation-tricks.md) |
| 1094 / 1109 | Car Pooling / Flight Bookings | Difference array | [representation](representation-tricks.md) |
| 1131 | Max of Absolute Value Expression | `\|x\| = max(x, −x)` → four sign cases, each separable | [algebra](algebraic-reformulation.md) |
| 1168 | Optimize Water Distribution | Virtual node 0 turns well costs into edges | [graph](graph-modeling.md) |
| 1371 | Longest Substring with Vowels in Even Counts | Parity bitmask as the prefix key | [algebra](algebraic-reformulation.md) |
| 1373 / 333 | Maximum Sum BST / Largest BST Subtree | Return a summary struct | [recursion](recursion-reframing.md) |
| 1448 | Count Good Nodes | Max-so-far flows down | [recursion](recursion-reframing.md) |
| 1466 | Reorder Routes | Direction matters — traverse both ways, count the wrong ones | [reversal](reversal-and-time.md) |
| 1653 | Minimum Deletions to Make String Balanced | Complement counting | [algebra](algebraic-reformulation.md) |
| 1671 | Minimum Removals to Make Mountain Array | LIS from both directions | [state](state-design.md) |
| 1697 | Edge Length Limited Paths | Sort queries offline, grow the union-find | [graph](graph-modeling.md) |
| 1970 | Last Day Where You Can Still Cross | Reverse time, or binary search the day | [reversal](reversal-and-time.md) |
| 2454 | Next Greater Element IV | Two stacks, scanning right to left | [reversal](reversal-and-time.md) |

---

## How to Use This

Don't read it front to back. Two workflows:

**While stuck:** run the checklist above. Give each question fifteen seconds. The point is to cycle through reframings faster than you can invent them.

**After failing:** find the problem in the master index, read the *whole* trick entry — not just the line for your problem. The value is in the transfer: 312 is only worth an hour if it also hands you 546, 1000, and 1039 for free. Then find another problem in the same entry and solve it cold within a day.

The honest measure of whether this folder worked: when you next see an unfamiliar hard problem, do you *try* running it backwards before you give up?

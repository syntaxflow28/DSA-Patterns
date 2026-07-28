# Representation — Changing How the Data Is Stored

Knowing the pattern is not the same as solving the problem. You can look at LC 289 Game of Life, recognise it instantly as "iterate the grid, count neighbours, apply four rules", write it in ninety seconds — and then read the follow-up asking for it in place, with a *simultaneous* update, and discover that your correct algorithm is unusable. Nothing about the algorithm changed. What changed is that you are no longer allowed a second grid, and the whole problem collapses to a single question: where do you put the answer while the input is still being read?

That question is the subject of this file. The other tricks in this folder change how the data is *processed* — run it backwards, rewrite the condition algebraically, redefine the state. This one changes how the data is **stored**. The algorithm on top usually stays exactly the same; you swap the container, the encoding, or the key, and a constraint that was blocking you evaporates. Two constraints trigger it almost every time: an explicit **O(1) extra space** demand, and a **missing lookup key** — you want a hash map but the thing you would key on is a whole subtree, a whole island shape, or a pair of coordinates that no `unordered_set` accepts.

Treat those constraints as instructions, not decoration. When the statement says "constant extra space", "do it in one pass", "the array is read-only", "`1 <= nums[i] <= n`", or "values up to 1e9 but at most 1e5 of them", the setter is not being tidy — they are naming the trick. The productive question is always **what unused capacity already exists in the data**: spare bits in an `int` that only needs one, a sign bit nobody is using, the fact that indices `0..n-1` are a perfect hash for values `1..n`, the fact that only the *ranks* of the numbers matter and not the numbers, the fact that a range update touches only two positions if you store differences instead of values. Find the slack and the space constraint answers itself.

---

## Trick 1: Hide extra data inside the array you were given

**The tell:** The statement demands O(1) extra space (or "in place", or a simultaneous/atomic update) while you clearly need a second array — a next-generation grid, a "have I seen this value" set, a per-row and per-column flag. Frequently paired with a suspiciously tight value range like `1 <= nums[i] <= n` or "cells are 0 or 1".

**The trick:** Store the auxiliary data *inside the input*, in capacity the input is not using. An `int` cell holding only 0/1 has 30 spare bits — write the next state into bit 1 and keep the old state in bit 0, then shift everything down in a second pass. An array whose values are all positive has an unused sign bit — negate `nums[abs(v)-1]` to record "value `v` was seen". A matrix that needs one flag per row and one per column already *has* a spare row and a spare column: row 0 and column 0. The only rule is that the encoding must be **reversible** for as long as the original value is still needed.

**Why it works:** A simultaneous update fails in place because a write destroys a value that a later read still needs. Two-bit packing fixes that by making the old and new values *coexist* in the same cell: reads use `v & 1` and see an untouched board, writes use `|= 2` and are invisible to those reads, and a final `v >>= 1` commits the entire generation atomically. Sign marking works for the same reason at lower bandwidth — the sign carries one bit of new information while the magnitude preserves the original value in full, so `abs(v)` is a perfect inverse. The failure modes both follow from the same principle: the encoding must not collide with legal data (0 has no sign, which is why 0-based value ranges need an offset or a sentinel), and it must be legal to write at all (LC 287 explicitly forbids mutation, which is precisely why it needs cycle detection rather than this trick).

![A Game of Life cell holding the old state in bit 0 and the new state in bit 1 so that reads with and 1 see the pre-update board while writes have already landed, alongside the LC 442 variant where the sign records presence and the magnitude preserves the value](images/trick-inplace-encode.svg)

**Template:**
```cpp
// LC 289 - two bits per cell: bit0 = current, bit1 = next
void gameOfLife(vector<vector<int>>& b) {
    int m = b.size(), n = b[0].size();
    for (int i = 0; i < m; i++)
        for (int j = 0; j < n; j++) {
            int live = 0;
            for (int di = -1; di <= 1; di++)
                for (int dj = -1; dj <= 1; dj++) {
                    if (!di && !dj) continue;
                    int x = i + di, y = j + dj;
                    if (x >= 0 && x < m && y >= 0 && y < n)
                        live += b[x][y] & 1;          // & 1 == the OLD board
                }
            if ((b[i][j] & 1) && (live == 2 || live == 3)) b[i][j] |= 2;
            if (!(b[i][j] & 1) && live == 3)              b[i][j] |= 2;
        }
    for (auto& row : b) for (int& v : row) v >>= 1;       // commit, all at once
}

// LC 442 - sign marking: index is the hash, sign is the "seen" bit
vector<int> findDuplicates(vector<int>& nums) {
    vector<int> out;
    for (int v : nums) {
        int i = abs(v) - 1;                                // magnitude survives
        if (nums[i] < 0) out.push_back(abs(v));            // marked twice
        else nums[i] = -nums[i];
    }
    return out;
}
```

**Problems:**
| Problem | Difficulty | How the trick applies |
|---|---|---|
| 289. Game of Life | Medium | The flagship. Bit 1 holds the next state, bit 0 the current; `& 1` during the sweep, `>> 1` after it. The two bits are what makes the update *simultaneous* rather than sequential. |
| 73. Set Matrix Zeroes | Medium | Row 0 and column 0 become the flag arrays. Their intersection `mat[0][0]` is claimed by both, so lift one of them into a single `bool firstColZero` and handle column 0 in a separate backward pass. |
| 41. First Missing Positive | Hard | Cyclic placement: swap each value `v` in `1..n` to index `v-1`, then the first index with `nums[i] != i+1` is the answer. The array *is* the hash table, and `1..n` is exactly the index range. |
| 442. Find All Duplicates in an Array | Medium | Negate `nums[abs(v)-1]`; hitting an already-negative slot means the second sighting. O(1) space because presence rides in the sign. |
| 448. Find All Numbers Disappeared in an Array | Easy | Identical marking pass; the answer is every index still positive at the end. Solve this before 442 if the sign trick is new. |
| 645. Set Mismatch | Easy | Both halves at once: the doubly-marked index gives the duplicate, the unmarked index gives the missing value. Good check that you can *read* the encoding, not just write it. |
| 287. Find the Duplicate Number | Medium | The anti-example. Same shape as 442, but the array is read-only, so marking is illegal and you fall back to Floyd cycle detection on `i -> nums[i]`. Knowing why the trick is banned is the point. |

**Pitfalls:**
- Reading `b[x][y]` instead of `b[x][y] & 1` inside the neighbour count. It compiles, produces plausible output on the first row, and silently makes the update sequential.
- Forgetting the commit pass. Every cell ends up holding a 2-bit number and every assertion downstream fails.
- Sign marking a value of 0, or an array containing 0. Negation is a no-op on 0, so either offset the values or use a separate sentinel.
- Mutating `v` and then reusing it: iterate over the *value you copied out* (`int v` by value, `abs(v)`), never over `nums[i]` after `nums[i]` may already have been flipped.
- In 73, writing the flags into row 0 and column 0 and then clearing row 0 first, which erases the column flags before you have read them. Order matters; do the interior first and the two spare lines last.

---

## Trick 2: Invent a canonical form and use it as a hash key

**The tell:** The task is "group these", "count distinct these", or "find duplicates among these", where "these" are objects with no natural key — anagrams, shifted strings, subtrees, island shapes. You want an `unordered_map`, and the only thing missing is something legal to put in the `[]`.

**The trick:** Define a **canonical form**: a function `f` such that two objects are equivalent exactly when `f(a) == f(b)`. Then the problem is one pass with a hash map, and all the difficulty relocates into designing `f`. Sorted string for anagrams, a 26-slot count signature when sorting is too slow, the vector of adjacent character differences for shifted strings, a serialization that includes null markers for subtrees, the DFS movement path relative to the entry cell for island shapes.

**Why it works:** "Equivalent" is an equivalence relation, and every equivalence relation has representatives — you are simply choosing one representative per class and naming it. Correctness needs two properties and it is worth checking them separately: `f` must be **invariant** under the equivalence (two anagrams must sort to the same string) and **injective across classes** (two non-anagrams must not). The second is where careless serializations break. Serializing a tree as `"1,2,3"` without null markers is invariant but not injective — a left-only child and a right-only child collapse to the same string — so `#` placeholders are not cosmetic, they are what makes the structure recoverable. For islands, the raw cell coordinates are *too* injective (translation should not matter), so you subtract the entry cell, and the resulting relative path is invariant under translation but still distinguishes shapes.

**Template:**
```cpp
// LC 49 - canonical form = sorted string (or a 26-count signature)
vector<vector<string>> groupAnagrams(vector<string>& strs) {
    unordered_map<string, vector<string>> g;
    for (auto& s : strs) {
        string key = s;
        sort(key.begin(), key.end());          // O(L log L); counting is O(L)
        g[key].push_back(s);
    }
    vector<vector<string>> out;
    for (auto& [k, v] : g) out.push_back(move(v));
    return out;
}

// LC 652 - canonical form = serialization WITH null markers
string dfs(TreeNode* n, unordered_map<string,int>& cnt, vector<TreeNode*>& out) {
    if (!n) return "#";                        // the marker is the whole trick
    string key = to_string(n->val) + "," + dfs(n->left, cnt, out)
                                   + "," + dfs(n->right, cnt, out);
    if (++cnt[key] == 2) out.push_back(n);
    return key;
}

// LC 694 - canonical form = movement path relative to the entry cell
void walk(vector<vector<int>>& g, int i, int j, char dir, string& path) {
    if (i < 0 || i >= (int)g.size() || j < 0 || j >= (int)g[0].size() || !g[i][j]) return;
    g[i][j] = 0;
    path += dir;
    walk(g, i + 1, j, 'D', path);
    walk(g, i - 1, j, 'U', path);
    walk(g, i, j + 1, 'R', path);
    walk(g, i, j - 1, 'L', path);
    path += 'b';                               // backtrack marker: shape, not just steps
}
```

**Problems:**
| Problem | Difficulty | How the trick applies |
|---|---|---|
| 49. Group Anagrams | Medium | The canonical form is the sorted string. When `L` is large relative to the alphabet, swap to a 26-count signature stringified with separators, dropping `O(L log L)` to `O(L)`. |
| 242. Valid Anagram | Easy | The one-pair version: compare canonical forms directly. Solve it as "equal keys", not as "count and compare", and 49 writes itself. |
| 249. Group Shifted Strings | Medium | Shifting preserves adjacent differences, so the key is the difference pattern mod 26. The subtraction is exactly the "quotient out the symmetry" move that 694 makes with translation. |
| 205. Isomorphic Strings | Easy | Canonicalise by first-occurrence index: `"paper" -> "01023"`. Two strings are isomorphic iff their normalised forms match. |
| 890. Find and Replace Pattern | Medium | The same first-occurrence normalisation applied to a whole list, then a key comparison against the pattern's form. |
| 652. Find Duplicate Subtrees | Medium | Serialize with `#` for nulls so structure is unambiguous, count keys in a map, emit each node whose count hits exactly 2 (hitting `== 2` avoids duplicate output). |
| 694. Number of Distinct Islands | Medium | Record the DFS path relative to the entry cell, with a backtrack marker so different shapes cannot share a path. Then it is just `unordered_set<string>::size()`. |
| 149. Max Points on a Line | Hard | The key is a slope, and slopes must be canonicalised: store `(dy/g, dx/g)` reduced by `gcd` with a fixed sign convention, never a `double`. Floating point is not a canonical form. |

**Pitfalls:**
- Serializing without null markers, or without separators. `"1,2"` and `"12"` collide the moment values exceed one digit; `1 -> (2)` and `1 -> (null, 2)` collide without `#`.
- Choosing a form that is invariant but not injective (too coarse) or injective but not invariant (too fine — raw coordinates in 694, unreduced slopes in 149).
- Sorting strings for 49 when the strings are long and numerous. Count signatures are strictly better asymptotically; know both and say why you picked one.
- Using `double` slopes, `float` keys, or anything with rounding as a hash key. Canonical forms must be exact.
- Building the key by concatenating with `+` inside a hot recursion. For large trees, prefer mapping each distinct key string to an integer id and keying the parent on `(val, leftId, rightId)`.

---

## Trick 3: Compress the coordinates

**The tell:** Values are enormous (`-1e9 <= nums[i] <= 1e9`, or coordinates up to `2^31`) but there are few of them (`n <= 1e5`), and the algorithm you want — a BIT, a segment tree, a bucket array — is indexed by *value*. You cannot allocate `1e9` slots and you do not need to.

**The trick:** Sort the distinct values, then replace each value by its **rank** in that sorted list. Now every value lives in `[0, n)`, and a Fenwick tree or segment tree of size `n` does exactly what a value-indexed structure would have. Do the replacement once, up front, with `sort` + `unique` + `lower_bound`.

**Why it works:** The algorithm only ever *compares* values — "how many earlier elements are smaller than this", "which interval does this endpoint fall in", "is this endpoint left of that one". Rank is an order-isomorphism: it preserves `<`, `=`, and `>` exactly, so any computation built solely from comparisons produces the identical answer on the compressed data. What it does not preserve is arithmetic, which is the boundary of the trick — in LC 218 and LC 850 you compress the x-coordinates for *indexing* but must un-compress them when computing widths and areas, because `rank[b] - rank[a]` is not `b - a`. When the predicate itself involves arithmetic (LC 493's `nums[i] > 2 * nums[j]`, LC 327's `sum[j] - sum[i]` in a range), compress the *set of query values* alongside the data — insert `2 * nums[j]`, or `sum + lower` and `sum + upper`, into the same sorted list before ranking, so the query targets have ranks too.

**Template:**
```cpp
// Coordinate compression, the whole idiom
vector<int> vals(nums.begin(), nums.end());
sort(vals.begin(), vals.end());
vals.erase(unique(vals.begin(), vals.end()), vals.end());
auto rank = [&](int x) {                       // 1-based, BIT-friendly
    return int(lower_bound(vals.begin(), vals.end(), x) - vals.begin()) + 1;
};

// LC 315 - count smaller after self: compress, then sweep right to left over a BIT
vector<int> countSmaller(vector<int>& nums) {
    int n = nums.size(), m = vals.size();
    vector<int> bit(m + 1, 0), res(n);
    auto add = [&](int i) { for (; i <= m; i += i & -i) bit[i]++; };
    auto qry = [&](int i) { int s = 0; for (; i > 0; i -= i & -i) s += bit[i]; return s; };
    for (int i = n - 1; i >= 0; i--) {
        res[i] = qry(rank(nums[i]) - 1);       // strictly smaller, already inserted
        add(rank(nums[i]));
    }
    return res;
}
```

**Problems:**
| Problem | Difficulty | How the trick applies |
|---|---|---|
| 315. Count of Smaller Numbers After Self | Hard | Compress to `[1, n]`, sweep right to left, query the BIT prefix below the current rank. Without compression the BIT would need `2e9` slots. |
| 327. Count of Range Sum | Hard | Compress the prefix sums *together with* `sum - upper` and `sum - lower`, so the two query endpoints have ranks in the same table. The clearest example of compressing queries, not just data. |
| 493. Reverse Pairs | Hard | The predicate is `nums[i] > 2 * nums[j]`, so `2 * nums[j]` must be in the compressed universe as well. Use `long long` before doubling. |
| 218. The Skyline Problem | Hard | Compress the x-coordinates to index the sweep, but emit the *original* x values in the answer. Compression indexes; it never reports. |
| 850. Rectangle Area II | Hard | Compress x into elementary strips, run a segment tree over strip indices, and multiply covered-strip *widths* from the original coordinates. Arithmetic on ranks would be silently wrong. |
| 699. Falling Squares | Hard | Compress the interval endpoints, then range-assign max over a segment tree. Same skeleton as 850 with a different aggregate. |

**Pitfalls:**
- Compressing the data but forgetting the query values (327, 493) — every query then falls between ranks and `lower_bound` quietly returns the wrong slot.
- Doing arithmetic on ranks. Widths, areas, and differences must come from `vals[rank]`, never from the rank itself.
- Forgetting `unique` after `sort`, which leaves duplicate ranks and makes `lower_bound` inconsistent with the tree size.
- Off-by-one between 0-based `lower_bound` output and the 1-based indexing a Fenwick tree needs. Add the `+ 1` inside the `rank` lambda once and never think about it again.
- Overflow in the query value: `2 * nums[j]` and prefix sums both need `long long` before they enter the table.

---

## Trick 4: Use a difference array for range updates

**The tell:** `k` operations each of the form "add `v` to every index in `[l, r]`", and only *one* read of the final array at the end. The naive loop is `O(nk)` and the constraints (`n, k <= 1e5`) make that exactly too slow.

**The trick:** Store **differences**, not values. A range add touches only two positions: `d[l] += v` and `d[r+1] -= v`. After all `k` updates, one prefix-sum pass over `d` reconstructs the final array. Total cost `O(n + k)`. The same object under a different name is the sweep line: `+v` at the start of an event and `-v` past its end, then scan.

**Why it works:** Prefix sum and differencing are inverse operations, so `a` and `d` carry identical information — you are choosing whichever representation makes your dominant operation cheap. In value space, a range add is `O(r - l)` and a point read is `O(1)`; in difference space the costs invert to `O(1)` and `O(n)`. When updates vastly outnumber reads, difference space wins outright. This is also why the technique degrades if reads are *interleaved* with updates: each read costs a fresh prefix sum, and at that point you want a Fenwick tree over the difference array (range update, point query) instead. And it is why LC 253 has a sweep-line solution at all — "how many meetings are live at once" is a range-add of `+1` over each meeting's span, and the answer is the maximum of the prefix sum.

**Template:**
```cpp
// LC 1109 - difference array, one prefix sum at the end
vector<int> corpFlightBookings(vector<vector<int>>& bookings, int n) {
    vector<int> d(n + 1, 0);                   // n+1 so r+1 is always in range
    for (auto& b : bookings) {
        d[b[0] - 1] += b[2];                   // 1-based flights -> 0-based index
        d[b[1]]     -= b[2];                   // "past the end", not "at the end"
    }
    for (int i = 1; i < n; i++) d[i] += d[i - 1];
    d.pop_back();
    return d;
}

// LC 253 - the same object as a sweep line over a sorted event map
int minMeetingRooms(vector<vector<int>>& iv) {
    map<int, int> d;                           // ordered: coordinates are sparse
    for (auto& m : iv) { d[m[0]]++; d[m[1]]--; }
    int cur = 0, best = 0;
    for (auto& [t, delta] : d) best = max(best, cur += delta);
    return best;
}
```

**Problems:**
| Problem | Difficulty | How the trick applies |
|---|---|---|
| 370. Range Addition | Medium | The pattern with nothing else attached. Write this once and the rest of the table is recognition. |
| 1109. Corporate Flight Bookings | Medium | 370 wearing a story, plus 1-based flight labels — the only real work is the index shift. |
| 1094. Car Pooling | Medium | Range add over the trip span, then check the running sum never exceeds `capacity`. Locations are bounded by 1000, so a plain array beats a map. |
| 253. Meeting Rooms II | Medium | Sweep-line form: `+1` at each start, `-1` at each end, answer is the max prefix. Equivalent to the min-heap-of-end-times solution; know both and know they are the same counting argument. |
| 1854. Maximum Population Year | Easy | Difference array over years, prefix sum, take the earliest argmax. The smallest possible instance of the idea. |
| 731. My Calendar II | Medium | The interleaved case: bookings arrive online, so a plain difference array is not enough on its own — an ordered map of deltas rescanned per query is the honest sweep-line answer, and the cost profile explains why. |

**Pitfalls:**
- Subtracting at `r` instead of `r + 1`. The range is inclusive; `d[r] -= v` silently drops the last element.
- Sizing `d` as `n` rather than `n + 1`, so `r = n - 1` writes out of bounds.
- Forgetting that the prefix sum pass is *required* — the difference array is not the answer, it is the compressed form of the answer.
- Using an ordered `map` when coordinates are dense and small (1094): a `vector` is an order of magnitude faster and simpler.
- Overflow when `k` updates all pile onto one index. Size the accumulator to the total, not the per-update value.

---

## Trick 5: Pack a pair into a single key

**The tell:** You need a hash set, a visited array, or a union-find over *pairs* — grid cells `(r, c)`, `(row, column)` groups, `(index, remainder)` states — and the standard library gives you no `hash<pair<int,int>>`, or a nested `unordered_map` is turning into a performance and readability problem.

**The trick:** Encode the pair as one integer. Grid cells become `r * cols + c`. Small bounded pairs become `(a << 16) | b` (or `(long long)a << 32 | b` when both are 32-bit). Now `unordered_set<int>`, a flat `vector<int>` of size `rows * cols`, or a DSU over `rows * cols` elements all just work, with better cache behaviour than any nested container.

**Why it works:** The encoding is a bijection as long as the second component is strictly bounded by the multiplier — `r * cols + c` is invertible precisely because `0 <= c < cols`, which is the same argument that makes positional notation work. The payoff is not only convenience: a flat array indexed by the packed id turns pointer-chasing hash lookups into contiguous reads, which is why every serious grid DSU is written over `id = r * cols + c` rather than over pairs. The trick also unlocks structural moves that pairs obscure — LC 947 unions a stone's *row* with its *column*, two different kinds of object, by mapping columns into a disjoint numeric range (`col + 10001`); once both live in one integer space, one DSU handles them, and the answer falls out as `n - components`.

**Template:**
```cpp
// Grid cell -> single id, and back
inline int id(int r, int c, int cols) { return r * cols + c; }
// r = id / cols;  c = id % cols;

// Two bounded ints -> one key (both < 65536)
inline int key(int a, int b) { return (a << 16) | b; }
// Two full 32-bit ints -> one 64-bit key
inline long long key64(int a, int b) {
    return ((long long)(unsigned)a << 32) | (unsigned)b;
}

// LC 305 - DSU over packed grid ids, activated incrementally
struct DSU {
    vector<int> p;
    DSU(int n) : p(n, -1) {}
    void make(int x) { if (p[x] == -1) p[x] = x; }
    int find(int x) { return p[x] == x ? x : p[x] = find(p[x]); }
    bool unite(int a, int b) {
        a = find(a); b = find(b);
        if (a == b) return false;
        p[a] = b; return true;
    }
};
```

**Problems:**
| Problem | Difficulty | How the trick applies |
|---|---|---|
| 200. Number of Islands | Medium | The DSU solution keys on `r * cols + c`; even the BFS/DFS solution benefits, since `visited` becomes a flat `vector<char>` instead of a set of pairs. |
| 305. Number of Islands II | Hard | Land arrives one cell at a time, so you need a DSU keyed by cell — packing is what makes "union with the four neighbours" a four-line loop over integer ids. |
| 130. Surrounded Regions | Medium | Union every border-connected `'O'` with a single virtual node `rows * cols`. The virtual node only exists because ids are integers with room above the grid. |
| 947. Most Stones Removed with Same Row or Column | Medium | Union `row` with `col + 10001` so rows and columns share one integer space. Answer is `n - components`. The offset *is* the trick. |
| 990. Satisfiability of Equality Equations | Medium | Variables are single lowercase letters, so `c - 'a'` packs them into `[0, 26)` and DSU runs over a 26-slot array. |
| 803. Bricks Falling When Hit | Hard | Reverse time (see `reversal-and-time.md`) *and* pack cells into DSU ids, with a virtual "ceiling" node. Two tricks composing cleanly. |
| 1091. Shortest Path in Binary Matrix | Medium | BFS where marking visited in place (`grid[r][c] = 1`) or on a flat packed array both beat a `set<pair<int,int>>`; a good place to notice that Trick 1 and Trick 5 solve the same annoyance. |

**Pitfalls:**
- Packing with the wrong multiplier — `r * rows + c` instead of `r * cols + c` is the single most common grid bug and only shows up on non-square inputs.
- Overflow: `(a << 16) | b` requires both components under `65536`, and negatives break it entirely. Offset negatives into a non-negative range first, or go to 64 bits.
- Sign extension in `key64`: cast through `unsigned` before widening, or negative `b` will smear ones across the high half.
- Assuming `unordered_set<long long>` is free. For dense bounded domains a flat `vector<char>` indexed by the packed id is dramatically faster; reach for the hash set only when the domain is sparse.
- Forgetting to decode. Store the packed id, but remember `r = id / cols`, `c = id % cols` when you need the coordinates back.

---

## Trick 6: Rolling hash to compare substrings in O(1)

**The tell:** The problem asks about repeated, matching, or longest-common **substrings**, and the natural solution compares substrings pairwise — which is `O(L)` per comparison and blows the budget. Often paired with a monotone property ("if a duplicate of length `k` exists, one of length `k-1` does too") that invites a binary search over the length.

**The trick:** Precompute prefix hashes `h[i] = h[i-1] * B + s[i]` and powers of the base. The hash of `s[l..r]` is then `h[r] - h[l-1] * B^(r-l+1)`, all mod a large prime — one subtraction, one multiplication, O(1). Substring equality becomes integer equality, so a whole sliding window of substrings can be dropped into an `unordered_set`. When the property is monotone in the length, wrap the whole thing in a binary search: `O(n log n)` instead of `O(n^2)`.

**Why it works:** The prefix hash is a polynomial evaluation, and polynomial evaluation telescopes — the contribution of a prefix can be cancelled exactly by multiplying it up to the right power, which is why the window hash is a difference rather than a rescan. Unlike the canonical forms of Trick 2, this is **probabilistic**: distinct substrings can collide. Keep the collision probability negligible by using a modulus around `1e18` with `__int128` arithmetic (or two independent mods near `1e9` combined into a pair), and a base above the alphabet size — ideally randomised, since fixed bases are the target of adversarial anti-hash tests. When correctness must be certain, verify candidate matches with a real comparison, or use KMP (LC 28), whose `O(n + m)` is deterministic and whose failure function is not much longer to write.

**Template:**
```cpp
struct RollingHash {
    static const unsigned long long M = (1ULL << 61) - 1;   // Mersenne prime
    vector<unsigned long long> h, p;
    static unsigned long long mul(unsigned long long a, unsigned long long b) {
        __uint128_t r = (__uint128_t)a * b;
        unsigned long long lo = (unsigned long long)(r & M), hi = (unsigned long long)(r >> 61);
        lo += hi;
        return lo >= M ? lo - M : lo;
    }
    RollingHash(const string& s, unsigned long long B) : h(s.size() + 1, 0), p(s.size() + 1, 1) {
        for (size_t i = 0; i < s.size(); i++) {
            h[i + 1] = mul(h[i], B) + (unsigned char)s[i];
            if (h[i + 1] >= M) h[i + 1] -= M;
            p[i + 1] = mul(p[i], B);
        }
    }
    unsigned long long get(int l, int len) const {          // hash of s[l .. l+len-1]
        unsigned long long x = h[l + len] + M - mul(h[l], p[len]);
        return x >= M ? x - M : x;
    }
};

// LC 1044 - binary search on the length, rolling hash for the O(1) check
// exists(len): drop every length-len hash into a set, report the first repeat
```

**Problems:**
| Problem | Difficulty | How the trick applies |
|---|---|---|
| 1044. Longest Duplicate Substring | Hard | The canonical pairing: duplicate-of-length-`k` is monotone in `k`, so binary search the length and use rolling hashes to test each length in `O(n)`. Verify hits to be safe. |
| 187. Repeated DNA Sequences | Medium | Fixed length 10 over a 4-letter alphabet, so each window packs into 20 bits and the "hash" is exact — a rolling hash with no collision risk at all. The best first exposure. |
| 28. Find the Index of the First Occurrence in a String | Easy | Rabin-Karp is the rolling-hash answer; KMP is the deterministic one. Know both, and be able to say that hashing is `O(n + m)` expected while KMP is `O(n + m)` worst case. |
| 214. Shortest Palindrome | Hard | Compare the prefix hash against the reversed-string hash to find the longest palindromic prefix in one pass. The KMP alternative builds the failure function of `s + '#' + reverse(s)`. |
| 718. Maximum Length of Repeated Subarray | Medium | The `O(nm)` DP is the expected answer, but binary search on the length plus hashing gives `O((n+m) log)` and is the same 1044 skeleton over two arrays. |
| 1147. Longest Chunked Palindrome Decomposition | Hard | Greedy two-pointer matching where each candidate chunk comparison is one hash equality instead of an `O(L)` `substr`. |

**Pitfalls:**
- A modulus near `2^31` or, worse, relying on `unsigned` overflow with base 31. Both are collision-prone on large inputs and both are explicitly targeted by anti-hash tests.
- A fixed, well-known base. Randomise it at startup when the input could be adversarial.
- Overflow before the modulo: `h[i] * B` needs `__int128` (or a mod under `2^31` with 64-bit intermediates). Getting this wrong produces answers that are right on small inputs.
- Reporting a hash match as a real match with no verification, in a problem where a wrong answer is unacceptable. One `compare` on the candidate costs `O(L)` once, not per window.
- Off-by-one in `get(l, len)`: fix the convention (`h[i]` = hash of the first `i` characters, `get` takes a start and a length) and write it in a comment.

---

## Family Cheat Sheet

| Trick | Tell | Move |
|---|---|---|
| 1. Hide data inside the input | "O(1) extra space", "in place", "simultaneous update", `1 <= nums[i] <= n` | Use spare bits, the sign, or spare rows/columns as scratch; commit in a second pass. Requires a reversible encoding and a writable input. |
| 2. Canonical form as a key | "Group these", "count distinct", "find duplicates" — objects with no natural key | Design `f` so equivalent objects share a form and non-equivalent ones do not; then it is one hash map pass. |
| 3. Coordinate compression | Huge value range, small `n`, and a structure indexed by value | Sort + `unique` + `lower_bound` to replace each value by its rank; compress query values too, and never do arithmetic on ranks. |
| 4. Difference array | Many range updates, one final read | `d[l] += v`, `d[r+1] -= v`, prefix sum once. Same object as a sweep line. |
| 5. Pack a pair into one key | Need a set/DSU/visited over pairs or grid cells | `r * cols + c`, or `(a << 16) \| b`; offset disjoint domains into one integer space so a single flat array or DSU serves both. |
| 6. Rolling hash | Substring comparison inside a loop; often monotone in the length | Prefix hashes make any substring hash O(1); binary search the length on top. Probabilistic — big modulus, random base, verify when it matters. |

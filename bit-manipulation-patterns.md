# Bit Manipulation — The Complete Interview Pattern Guide (C++)

Bit manipulation is the technique of treating an integer as a fixed array of bits and operating on it with the bitwise operators (`& | ^ ~ << >>`). It turns problems that *look* like they need extra space or an inner loop into O(1)-space, branch-free, often single-instruction tricks. This guide covers every interview-relevant pattern: the logic, the **core insight** (why the trick is correct, not just the incantation), how to recognize it, C++ templates, classic problems with notes, and the pitfalls that bite everyone.

---

## The Operator Survival Kit

| Op | Meaning | Identity worth memorizing |
|---|---|---|
| `a & b` | AND | `x & 0 = 0`, `x & ~0 = x`, `x & (x-1)` clears lowest set bit |
| `a \| b` | OR | `x \| 0 = x`, used to *set* bits |
| `a ^ b` | XOR | `x ^ x = 0`, `x ^ 0 = x`, self-inverse and commutative |
| `~a` | NOT | `~x = -x - 1` in two's complement |
| `x << k` | shift left | multiply by 2^k (watch overflow) |
| `x >> k` | shift right | floor-divide by 2^k for **non-negative** x |

**Two's complement is the whole game.** In a signed 32-bit int, `-x == ~x + 1`. That single fact powers `x & -x` (lowest set bit), explains why right-shifting a negative number is implementation-defined, and is why you reach for `unsigned`/`long long` the moment a high bit might be set.

---

## How to Recognize a Bit-Manipulation Problem

Ask yourself:
- Does the answer involve **counting, toggling, or combining bits** directly?
- Is there a "every element appears twice except one" / "find the unique" flavor? → **XOR**.
- Is `n` small (≤ ~20) and you need to enumerate **subsets / states**? → **bitmask**.
- Are you asked to do arithmetic **without** `+`, `*`, `/`? → simulate with shifts and XOR.
- Is the constraint "**O(1) extra space**" on a problem that screams hashing? → a bit often replaces the hash set.
- "Power of two/four", "is only one bit set", "bits in a range" → **mask identities**.

If yes to any → bits are probably the intended tool.

---

## Pattern 1: The Bit Toolkit — Test, Set, Clear, Toggle

**Logic:** Address an individual bit by building a mask `1 << i` and combining it with the right operator. These four operations are the alphabet every other pattern is spelled in.

**Core insight — why each works:** A mask `1 << i` is "all zeros except position i". AND with it *probes* (everything else is forced to 0, so the result is non-zero iff bit i was set). OR *forces on* (0-bits of the mask leave the number untouched — OR is the identity with 0). XOR *flips* (XOR with 1 inverts, XOR with 0 preserves). AND with the *complement* `~(1<<i)` *forces off* (the single 0 in the mask zeroes that position, the 1s elsewhere preserve). The asymmetry — OR to set, AND-with-complement to clear — is exactly the 0/1 identities of each operator.

**Template:**
```cpp
bool test (int x, int i) { return (x >> i) & 1; }      // is bit i set?
int  set  (int x, int i) { return x |  (1 << i); }      // turn bit i on
int  clear(int x, int i) { return x & ~(1 << i); }      // turn bit i off
int  toggle(int x, int i){ return x ^  (1 << i); }      // flip bit i
int  lowestBit(int x)    { return x & -x; }             // isolate lowest set bit
int  clearLowest(int x)  { return x & (x - 1); }        // drop lowest set bit
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 190. Reverse Bits | Easy | Pull the low bit off the source with `x & 1`, push it onto the result with `res = (res << 1) \| bit`, repeat 32 times. Direction matters: source low→high, result high→low. |
| 1318. Minimum Flips to Make a OR b Equal to c | Medium | Walk all 32 positions. If `c`'s bit is 0, every set bit among a,b must flip; if it's 1, you need at least one flip when both a,b are 0. Pure per-bit case analysis. |
| 868. Binary Gap | Easy | Track the index of the previous set bit; the gap is the difference. Shift and test one bit at a time. |
| 1009. Complement of Base 10 Integer | Easy | Build a mask of all 1s up to the highest set bit, then XOR — you must not flip the leading zeros, which is the whole subtlety. |

**Pitfalls:**
- `1 << 31` overflows a signed `int` (UB). Use `1u << 31` or `1LL << i` when the bit can reach the sign position.
- Right-shifting a **negative** signed int is implementation-defined (usually arithmetic, sign-extending). Cast to `unsigned` when you want logical shifts, e.g. in Reverse Bits.

---

## Pattern 2: Population Count & Lowest Set Bit

**Logic:** Count how many bits are set, or process each set bit once. The naive way checks all 32 positions; the fast way visits only the set bits using `x & (x-1)`.

**Core insight — why `x & (x - 1)` clears the lowest set bit:** Subtracting 1 from `x` flips the lowest set bit to 0 and turns every 0 below it into a 1 (borrow propagation). ANDing with the original keeps only the bits **above** the lowest set bit — the lowest set bit and everything below it vanish. So each AND removes exactly one set bit, and the loop runs *popcount(x)* times, not 32. Brian Kernighan's trick.

**The Counting Bits DP** (problem 338) is the other half: `bits[i] = bits[i & (i-1)] + 1` — `i & (i-1)` is `i` with one fewer set bit, a strictly smaller index already computed. (Equivalently `bits[i] = bits[i >> 1] + (i & 1)`.)

**Template:**
```cpp
int popcount(int x) {                 // Brian Kernighan
    int count = 0;
    while (x) { x &= (x - 1); count++; }   // each step kills one set bit
    return count;
}
// __builtin_popcount(x) is the library shortcut; know the manual version too.
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 191. Number of 1 Bits | Easy | The canonical popcount. Kernighan's loop runs once per set bit; mention `__builtin_popcount` but be able to derive it. |
| 338. Counting Bits | Easy | DP over `0..n`: `bits[i] = bits[i>>1] + (i&1)`. O(n) total instead of O(n·32). The recurrence reuse is the point. |
| 461. Hamming Distance | Easy | Differing bits = `popcount(a ^ b)`. XOR marks every position where they disagree, popcount counts them. |
| 477. Total Hamming Distance | Medium | Don't compare pairs (O(n²)). For each of the 32 bit positions, if `k` numbers have it set, that bit contributes `k * (n - k)` to the total. Per-column counting. |
| 201. Bitwise AND of Numbers Range | Medium | The AND of `[m, n]` is the **common binary prefix** of m and n: shift both right until equal, then shift back. Any differing low bit gets zeroed somewhere in the range. |

**Pitfalls:**
- For 477, the brute-force pairwise XOR is O(n²) and TLEs. The "count set bits per column" reframing is the expected answer — say it before coding.
- `__builtin_popcount` takes `unsigned int`; use `__builtin_popcountll` for 64-bit values or you silently drop the high half.

---

## Pattern 3: XOR — The Self-Inverse Trick

**Logic:** XOR is commutative, associative, and self-canceling (`x ^ x = 0`, `x ^ 0 = x`). So XOR-ing a whole collection makes every value that appears an **even** number of times disappear, leaving exactly the odd-one-out — with zero extra space and in any order.

**Core insight — why it beats hashing:** A hash set finds the unique element in O(n) time and O(n) space. XOR achieves O(n) time and **O(1) space** because it doesn't *store* what it has seen — it stores the running *parity* of every bit position. Pairs cancel bit-by-bit regardless of arrival order (associativity + commutativity), so the accumulator is a perfect "appeared an odd number of times so far" summary in a single integer. This is the canonical example of replacing memory with an algebraic invariant.

**Template:**
```cpp
int findUnique(vector<int>& nums) {
    int acc = 0;
    for (int x : nums) acc ^= x;   // pairs cancel; the loner survives
    return acc;
}
```

**Two loners (problem 260):** XOR everything to get `a ^ b`. They differ somewhere, so `a ^ b` has at least one set bit; isolate the lowest with `diff = (a^b) & -(a^b)`. That bit splits all numbers into two groups (set vs not). `a` and `b` fall in different groups, every pair stays together — XOR each group separately.

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 136. Single Number | Easy | The pure pattern: XOR the array, pairs vanish, the single survives. O(1) space is the whole reason to use it over a hash set. |
| 268. Missing Number | Easy | XOR `0..n` with all array values; every present number cancels its index-twin and the missing one is left. (Gauss sum also works but can overflow.) |
| 389. Find the Difference | Easy | XOR all chars of both strings together — the one extra letter is the survivor. Treat `char` as its int code. |
| 260. Single Number III | Medium | Two uniques: XOR all, isolate a differing bit `x & -x`, partition into two buckets, XOR each. The "lowest set bit splits the world" idea is the key step. |
| 137. Single Number II | Medium | Every element appears 3× except one. XOR fails (3 is odd). Count bits mod 3 per column, **or** the two-mask state machine `ones = (ones ^ x) & ~twos; twos = (twos ^ x) & ~ones;`. Know at least the mod-3 counting version. |
| 1310. XOR Queries of a Subarray | Medium | Prefix-XOR array: `pre[i] = a[0]^...^a[i-1]`, then `xor(l,r) = pre[r+1] ^ pre[l]`. The prefix-sum trick with XOR as the (self-inverse) "subtraction". |
| 1442. Count Triplets That Can Form Two Arrays of Equal XOR | Medium | `a == b` iff the XOR of the whole `[i,k]` range is 0; count via prefix-XOR equalities. XOR as an equality detector. |

**Pitfalls:**
- XOR only works when the "duplicates" appear an **even** number of times. Single Number II (3×) needs counting mod 3 or the dual-mask trick — applying plain XOR is the classic wrong answer.
- For Single Number III, students forget XOR is *commutative/associative* and overthink ordering. The grouping bit can be **any** differing bit; the lowest is just convenient to extract.

---

## Pattern 4: Power-of-Two & Mask Identities

**Logic:** Many "is this number special?" questions reduce to a one-line bit identity. The reusable star is: **a power of two has exactly one set bit**, so `n & (n - 1) == 0`.

**Core insight:** A power of two is `100…0` in binary. Subtracting 1 turns it into `011…1`, which shares **no** set bit with the original — their AND is 0. Conversely any number with two or more set bits keeps at least one bit after the subtract-and-AND. So `n > 0 && (n & (n-1)) == 0` is an exact, branchless power-of-two test. Building masks of contiguous 1s (`(1 << k) - 1`) extends the idea to "low k bits" extraction.

**Template:**
```cpp
bool isPowerOfTwo(int n)  { return n > 0 && (n & (n - 1)) == 0; }
bool isPowerOfFour(int n) { return n > 0 && (n & (n - 1)) == 0   // one bit...
                                   && (n & 0x55555555);          // ...in an even position
}
int lowKBits(int x, int k){ return x & ((1 << k) - 1); }         // keep the low k bits
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 231. Power of Two | Easy | `n > 0 && (n & (n-1)) == 0`. The `n > 0` guard matters — 0 and negatives sneak through otherwise. |
| 342. Power of Four | Easy | Power of two **and** the single bit sits at an even index — AND against the mask `0x55555555` (bits 0,2,4,…). |
| 762. Prime Number of Set Bits | Easy | `popcount` the range, check primality of small counts (≤ ~32, so a tiny prime set suffices). Combines Pattern 2 with a lookup. |
| 693. Binary Number with Alternating Bits | Easy | `x ^ (x >> 1)` makes alternating bits become all-1s; then test that result is `0111…1` via `y & (y+1) == 0`. |
| 1763. Longest Nice Substring | Easy | Per substring, track lower/upper presence as two 26-bit masks; "nice" iff the masks are equal. Bitmask as a set membership summary. |

**Pitfalls:**
- Forgetting the `n > 0` guard: `0 & -1 == 0` would wrongly report 0 as a power of two; negatives in two's complement also slip through.
- Powers of four vs two: people remember the single-bit test but forget the *position* constraint — `8 = 1000` is a power of two but not four.

---

## Pattern 5: Bitmask as a Set — Subset & Submask Enumeration

**Logic:** When `n ≤ ~20`, represent a subset of `{0,…,n-1}` as the bits of an integer `0 .. 2^n - 1`. Element `i` is in the subset iff bit `i` is set. Iterating `mask` from `0` to `(1<<n)-1` enumerates **every** subset exactly once.

**Core insight — why this is the natural encoding:** There are exactly `2^n` subsets and exactly `2^n` integers in `[0, 2^n)`, and the map "subset ↔ its characteristic bit-vector" is a bijection. So a single `for` loop *is* a complete, duplicate-free powerset enumeration — no recursion, no visited set. The bitwise ops then become set algebra: `|` is union, `&` is intersection, `^` is symmetric difference, `mask & (mask-1)` removes an element, and `__builtin_popcount(mask)` is the set size. This encoding is also the backbone of bitmask DP (see the DP guide's TSP/assignment patterns).

**Template:**
```cpp
// Enumerate all subsets of n elements
for (int mask = 0; mask < (1 << n); ++mask) {
    for (int i = 0; i < n; ++i)
        if (mask & (1 << i)) { /* element i is in this subset */ }
}

// Enumerate all SUBMASKS of a fixed mask (each subset of the set 'mask')
for (int sub = mask; sub; sub = (sub - 1) & mask) { /* use sub */ }
// ...don't forget the empty submask 0 if you need it.
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 78. Subsets | Medium | The bitmask loop generates the powerset without recursion — a clean alternative to backtracking. Each `mask` is one subset; read off the set bits. |
| 1239. Maximum Length of a Concatenated String with Unique Characters | Medium | Represent each word as a 26-bit mask; it's usable only if it has no internal repeats. Combine masks with `&==0` (disjoint) checks — set union via OR. |
| 318. Maximum Product of Word Lengths | Medium | Encode each word's letters as a 26-bit mask; two words "share no letter" iff `maskA & maskB == 0`. Reduces an O(L) char comparison to one AND. |
| 698. Partition to K Equal Sum Subsets | Medium | `dp[mask]` = can we form complete buckets using exactly the chosen elements. Submask/state DP over which elements are used. |
| 847. Shortest Path Visiting All Nodes | Hard | State = `(node, visited-mask)`; BFS over `2^n · n` states. The mask tracks the set of visited nodes compactly. |

**Pitfalls:**
- This only scales to `n ≲ 20–22` (`2^20 ≈ 10^6`). Beyond that, `2^n` explodes — recognize the small-`n` constraint as the *signal*, and don't reach for bitmask when n is large.
- The submask loop `(sub - 1) & mask` is easy to get wrong: it must AND with `mask` each step to stay inside the set, and it **excludes** `sub = 0` (handle the empty submask separately if needed).

---

## Pattern 6: Arithmetic Without `+`, `*`, `/`

**Logic:** Rebuild grade-school arithmetic from bitwise primitives. Addition splits into a sum-without-carry (`a ^ b`) and a carry (`(a & b) << 1`); repeat until the carry is zero.

**Core insight — why XOR-and-carry converges:** `a ^ b` adds each column **ignoring** carries; `(a & b) << 1` is exactly the carry generated into the next column. Re-adding them is still an addition, but the carry term has strictly more trailing zeros each round (the lowest set bit moves left), so after at most ~32 iterations the carry becomes 0 and `a ^ b` is the final sum. You're hand-rolling a ripple-carry adder. Subtraction, multiplication (shift-and-add), and division (shift-and-subtract) all decompose the same way.

**Template:**
```cpp
int add(int a, int b) {
    while (b != 0) {
        unsigned carry = (unsigned)(a & b) << 1;  // bits that carry into the next column
        a = a ^ b;                                // sum without carry
        b = carry;                                // re-add the carry
    }
    return a;
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 371. Sum of Two Integers | Medium | The XOR-sum / AND-carry loop above. Use `unsigned` for the carry shift to dodge signed-overflow UB; the result reinterprets correctly in two's complement. |
| 29. Divide Two Integers | Medium | Repeated doubling: subtract the largest `divisor << k` that still fits, add `1 << k` to the quotient. Work in `long long`/negatives to handle the `INT_MIN / -1` overflow case. |
| 67. Add Binary | Easy | Either simulate column addition with a carry, or loop the XOR/AND-carry trick on the parsed integers. Watch leading-zero and length-mismatch handling. |
| 43. Multiply Strings | Medium | Not strictly bitwise, but the shift-and-add mindset transfers: each digit contributes a shifted partial product. |

**Pitfalls:**
- Signed overflow in C++ is **undefined behavior**; do the carry shift on `unsigned` and let the bit pattern reinterpret as the two's-complement result.
- Divide Two Integers has a single overflow case (`INT_MIN / -1 = INT_MAX + 1`). Special-case it explicitly or do the math in `long long`.

---

## When Bit Manipulation FAILS — Know the Boundary

- **Large `n` for bitmasks.** `2^n` is only tractable to ~20 elements. A "set of items" with n in the thousands needs a real set / DP, not a mask.
- **XOR with non-even multiplicities.** XOR cancels *pairs*. Triples, or "appears k times", break it — switch to per-bit counting mod k or a state machine.
- **Floating point.** Bit tricks assume integers (or careful IEEE-754 reinterpretation). Don't bit-twiddle `double`s in interviews unless the problem is explicitly about their representation.
- **Readability vs. cleverness.** `x & (x-1)` is idiomatic; an obscure 6-line bit hack that saves nothing over a clear loop is a *negative* signal. Use bits where they buy real asymptotic or space wins.

---

## Cheat Sheet: Problem Phrase → Pattern

| Signal in the problem | Pattern |
|---|---|
| "appears twice except one", "find the unique", O(1) space | XOR self-inverse |
| "count set bits", "hamming distance" | Popcount / Kernighan |
| "power of two/four", "only one bit set" | `n & (n-1) == 0` identity |
| "all subsets", n ≤ ~20, "choose any combination" | Bitmask enumeration |
| "without + - * /", binary addition | XOR-sum + carry loop |
| set membership / "uses these letters" | bitmask as a set + OR/AND |
| "AND/OR/XOR over a range or subarray" | prefix-XOR or common-prefix |

---

## Complexity Summary

- Test/set/clear/toggle, power-of-two test, lowest-set-bit: **O(1)**.
- Popcount via Kernighan: **O(set bits)**, at most O(word size).
- XOR over an array, prefix-XOR queries: **O(n)** time, **O(1)** extra (queries O(1) after O(n) prep).
- Subset enumeration: **O(2^n · n)** — the hard ceiling that limits n.
- Bitwise add/divide: **O(word size)** ≈ O(32) per operation.

---

## Interview Tips

1. **Name the identity, then use it.** "A power of two has exactly one set bit, so `n & (n-1)` clears it to zero" scores far higher than silently typing the expression.
2. **Reach for `unsigned`/`long long` deliberately.** Sign bit, `1 << 31`, and carry shifts are the overflow traps. State that you're avoiding signed UB.
3. **XOR is your O(1)-space superpower.** Whenever the brute force is "hash set to find the odd one out", check if the multiplicities are even — if so, XOR wins on space.
4. **Let the constraint tip you off.** `n ≤ 20` almost always means bitmask; "O(1) extra space" on a dedup/unique problem almost always means XOR or a single accumulator.
5. **Don't out-clever yourself.** If a plain loop is just as fast and clearer, prefer it. Bit tricks are for genuine space/asymptotic wins, not showing off.

---

## Suggested Practice Order

**Warm-up:** 191 → 190 → 338 → 461 → 231
**XOR core:** 136 → 268 → 389 → 260 → 137
**Masks & identities:** 342 → 201 → 1310 → 318
**Subset/state:** 78 → 1239 → 698
**Arithmetic:** 371 → 29 → 67

Master the identities first — almost every "hard" bit problem is two or three of these basics composed together.

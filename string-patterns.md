# Strings — The Complete Interview Pattern Guide (C++)

Most "string problems" are other patterns in costume — substring windows are the sliding-window guide, palindrome expansion is the two-pointers guide, bracket matching is the stack guide, LCS/edit-distance is the DP guide, prefix dictionaries are the trie (remaining essentials), and string generation is the backtracking guide. **This sheet covers only what none of those do**: the genuinely string-native machinery — canonical forms and signatures, parsing and state machines, the prefix function (KMP), rolling hashes (Rabin-Karp), and the linear-time toolkit (Z-function, Manacher) — with the **core insight** for each pattern, C++ templates, strong problem sets, and the pitfalls. A routing table at the end sends every "string-looking" problem to its real home.

---

## What Makes a Problem String-Native

> **A problem belongs here when the *content* of the characters — their identity, order, repetition structure — is the algorithm, not just the data being iterated.**

- **Structure-of-content questions**: "are these rearrangements of each other," "does this pattern occur in this text," "does the string repeat itself," "what's the longest border" — no window, no stack, no DP table answers these directly.
- **The two great string tools** both exploit the same fact — a string's prefixes and suffixes overlap in analyzable ways: **KMP's prefix function** (exact, combinatorial, zero false positives) and **rolling hashes** (numeric, flexible, probabilistic). Most hard string problems yield to one of the two; knowing *which* is Pattern 3 vs 4's whole discussion.
- **Parsing problems** are the opposite kind: no deep algorithm at all, just disciplined case handling — and they're asked constantly *because* they measure discipline, not knowledge.
- **The negative signals**: "longest/shortest **substring** with property" → sliding window guide. "Expand around centers" → two pointers guide. "Nested/matching" → stack guide. "Subsequence alignment of two strings" → DP guide. "Many words, shared prefixes" → trie. "Generate all valid strings" → backtracking guide.

---

## Pattern 1: Canonical Forms & Signatures — Equality Under Rearrangement

**Logic:** Two strings are "the same" under some transformation (rearrangement, character renaming) iff they share a **canonical form** — a representative you can compute, hash, and compare. Anagrams: the sorted string, or better, the 26-count frequency vector. Isomorphism/pattern matching: the sequence of first-occurrence indices. Group-by problems become "hashmap keyed on the canonical form."

**Core insight — why it works:** An equivalence relation partitions strings into classes, and a canonical form is a *choice of one name per class* — so "equivalent?" collapses to "same name?", and grouping n strings costs one map pass instead of O(n²) pairwise checks. The engineering question is picking the cheapest sound name: sorting gives O(L log L) keys; a count vector gives O(L) and works because character order is exactly what anagram-equivalence ignores. For isomorphism (205, 290), the name is the *shape* of repetition — map each character to the index of its first occurrence ("abb" → 0,1,1) — because renaming preserves precisely that shape. Choosing a canonical form that ignores exactly what the relation ignores, no more, no less, *is* the correctness argument.

**Template (group anagrams — count-vector key):**
```cpp
unordered_map<string, vector<string>> groups;
for (const string& s : strs) {
    array<int, 26> cnt{};
    for (char c : s) cnt[c - 'a']++;
    string key;                                // canonical name: "2#0#1#...#"
    for (int x : cnt) { key += to_string(x); key += '#'; }   // '#' guards against "1,12" vs "11,2"
    groups[key].push_back(s);
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 242. Valid Anagram | Easy | One count array, increment for s, decrement for t, check all zeros. The unicode follow-up ("what if not a–z?") is standard — answer: hashmap, and mention normalization complexity honestly. |
| 383. Ransom Note | Easy | One-directional counting: the note must be *coverable* by the magazine — counts go negative exactly when it isn't. 242's asymmetric sibling. |
| 49. Group Anagrams | Medium | The template. Sorted-string key is the easy answer; the count-vector key is the O(L) upgrade — present both and say when the difference matters (long strings). |
| 205. Isomorphic Strings | Easy | Two-directional mapping (s→t AND t→s) — one direction alone accepts "badc"/"baba". Or: compare first-occurrence-index encodings of both strings. |
| 290. Word Pattern | Easy | 205 with words instead of characters. Same bijection requirement, same one-direction trap — recognize the reskin and bank the time. |
| 890. Find and Replace Pattern | Medium | Filter by isomorphism: canonical-encode the pattern once, encode each word, compare. Canonical forms as a *filter predicate* — the pattern generalizing. |
| 1657. Determine if Two Strings Are Close | Medium | The subtle one: operations preserve (set of characters, multiset of frequencies) — and those two invariants are also *sufficient*. Proving sufficiency, not computing the signature, is the interview content. |
| 49-adjacent: 438. Find All Anagrams | Medium | NOT here — the moment the anagram check *slides over a text*, it's the sliding-window guide. Knowing the handoff point is part of this pattern. |

**Pitfalls:**
- Delimiters in built keys: `"1,12"` and `"11,2"` collide without a separator — the `#` in the template is load-bearing.
- Isomorphism with one map: s→t injectivity alone is not bijectivity — "badc"→"baba" passes one-way checks. Two maps, or the first-occurrence encoding of *both* strings.
- 1657-style invariant problems: verifying the invariants are *necessary* is easy; the answer is incomplete until you argue they're *sufficient* (transformations exist realizing any signature match).
- `array<int,26>` assumes lowercase input — state the assumption; it's a planted follow-up in half these problems.

---

## Pattern 2: Parsing & State Machines — Discipline, Not Cleverness

**Logic:** Convert, validate, or format text by scanning with explicit state: skip whitespace → read sign → consume digits → clamp, or column-by-column layout arithmetic. The winning move is naming the states and legal transitions *before* writing code — turning an if-else swamp into a small, auditable machine.

**Core insight — why it works:** These problems have no algorithmic depth — atoi is O(n) by any approach — so what's being measured is *specification discipline*: edge-case enumeration, overflow handling, and code whose structure mirrors the spec. A state machine works because valid inputs form a **regular language** (states = "where in the format am I," transitions = "what may come next"), so validity checking is one pass with a state variable, and every malformed input dies at a specific missing transition you can point to. Overflow is the recurring sub-boss: check *before* the multiply-add (`val > (INT_MAX - d) / 10`), or accumulate in `long long` and clamp — pick one idiom, use it every time, and say it aloud.

**Template (atoi — the canonical gauntlet):**
```cpp
int myAtoi(const string& s) {
    int i = 0, n = s.size(), sign = 1;
    long long val = 0;
    while (i < n && s[i] == ' ') i++;                    // state 1: whitespace
    if (i < n && (s[i] == '+' || s[i] == '-'))           // state 2: optional sign
        sign = (s[i++] == '-') ? -1 : 1;
    while (i < n && isdigit(s[i])) {                     // state 3: digits
        val = val * 10 + (s[i++] - '0');
        if (sign * val <= INT_MIN) return INT_MIN;       // clamp early — val can't overflow LL here
        if (sign * val >= INT_MAX) return INT_MAX;
    }
    return (int)(sign * val);                            // anything after digits: ignored
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 8. String to Integer (atoi) | Medium | The template. The four states and the clamp idiom are the deliverable; enumerate the edge inputs (" -042", "4193 with words", "-91283472332") before coding. |
| 65. Valid Number | Hard | The state-machine flagship: states for sign/integer/dot/fraction/exponent, transitions written as a table. Ad-hoc booleans collapse under this one — the machine formulation is *the* solution. |
| 13. Roman to Integer | Easy | One rule: a symbol smaller than its right neighbor subtracts. Single reverse (or forward-peek) pass — resist the urge to hardcode all six subtractive pairs. |
| 12. Integer to Roman | Medium | Greedy over the 13-value table (including the subtractive pairs as first-class entries). The table *is* the algorithm — say why greedy is safe (each symbol's value exceeds the max representable by everything smaller). |
| 165. Compare Version Numbers | Medium | Split on dots, compare numerically, treat missing chunks as 0. All the difficulty is in "1.0" == "1.0.0" — the missing-chunk rule. |
| 6. Zigzag Conversion | Medium | Don't simulate the zigzag — either append to `rows[r]` with a bouncing row index, or jump arithmetic per row (cycle 2·(numRows−1)). The cycle formula is the senior answer. |
| 38. Count and Say | Medium | Run-length encoding iterated n times. Trivial logic, but a clean run-scanning loop (peek-ahead while equal) is a reusable primitive — write it well once. |
| 68. Text Justification | Hard | Pure formatting arithmetic: greedily pack words, distribute `spaces / gaps` with the first `spaces % gaps` gaps getting one extra; last line left-justified. Zero algorithms, maximum care — a famous filter question. |
| 468. Validate IP Address | Medium | Split then validate chunks (v4: 1–3 digits, ≤255, no leading zero; v6: 1–4 hex). A state-machine-lite drill in spec fidelity. |

**Pitfalls:**
- Overflow checked *after* the multiply-add is undefined behavior on `int` — check before, or accumulate in `long long` (and know `long long` alone doesn't save you if input length is unbounded — clamp per step, as the template does).
- 68's space distribution: extra spaces go to the *leftmost* gaps, and single-word lines are left-justified — the two clauses everyone drops.
- Empty-chunk edges: "1..2" in version/IP splitting produces empty tokens — decide their meaning per spec, don't let `stoi` throw.
- Writing 65 as accumulated booleans (`seenDigit`, `seenDot`, `seenExp`…) works but reads as luck; the transition-table formulation is the same length and provably complete.

---

## Pattern 3: The Prefix Function (KMP) — Borders and Failure Links

**Logic:** `pi[i]` = length of the longest proper prefix of `s[0..i]` that is also a suffix ending at i (the longest **border**). Built in O(n) by reusing previous values: on mismatch, fall back to `pi[j-1]` — the next-longest border — instead of restarting. Substring search: compute pi on `pattern + '#' + text`; every position where pi equals the pattern length is a match. Periodicity: `n − pi[n−1]` is the smallest period of the string.

**Core insight — why it works:** The border structure is the key object: all borders of a prefix form a chain (`pi[i]`, `pi[pi[i]−1]`, …), so "longest border extendable by the next character" is found by walking that chain — and the walk amortizes to O(n) because j only grows by 1 per step and each fallback shrinks it. For search, the `#` separator (a character in neither string) guarantees no border crosses it, so a border of length m *is* a full pattern occurrence — search inherits the linear bound with zero false positives, which is KMP's edge over hashing. For repetition (459): the string is a repeated block iff `pi[n−1] > 0` and `n % (n − pi[n−1]) == 0` — the overlap of the string with itself shifted by one period forces the block structure. Be able to draw that overlap picture; it's the *why* interviewers probe.

**Template (prefix function + search):**
```cpp
vector<int> prefixFunction(const string& s) {
    int n = s.size();
    vector<int> pi(n, 0);
    for (int i = 1; i < n; i++) {
        int j = pi[i - 1];
        while (j > 0 && s[i] != s[j]) j = pi[j - 1];   // fall back through the border chain
        if (s[i] == s[j]) j++;                          // extend the border by one
        pi[i] = j;
    }
    return pi;
}
// search: auto pi = prefixFunction(pattern + "#" + text);
// match ends at text position i wherever pi[m + 1 + i] == m  (m = pattern length)
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 28. Find the Index of the First Occurrence | Easy | The judge accepts brute force — write KMP anyway; this is *the* rehearsal problem for the template, and "how would you do it in O(n+m)?" is the guaranteed follow-up. |
| 1392. Longest Happy Prefix | Hard | Literally `pi[n−1]` — one function call. Hard-labeled purely because it requires knowing the tool; the cleanest possible KMP payoff. |
| 459. Repeated Substring Pattern | Easy | The periodicity criterion: `pi[n−1] > 0 && n % (n − pi[n−1]) == 0`. Also know the folk trick — s is periodic iff s occurs in (s+s) with both ends chopped — and why it's the same fact. |
| 686. Repeated String Match | Medium | Search b in a repeated ⌈|b|/|a|⌉ or +1 times — KMP (or hashing) does the search; the bound argument ("one extra copy always suffices") is the actual content. |
| 214. Shortest Palindrome | Hard | Longest palindromic *prefix* via borders: run pi on `s + '#' + reverse(s)` — the final value is the length of the longest prefix of s that's a suffix of reverse(s), i.e. palindromic. The construction is the aha; memorize the recipe, derive the why. |
| 796. Rotate String | Easy | Rotation ⇔ goal is a substring of s+s (same lengths). One line with `find`, but *say* the s+s insight — it's the transferable part, and KMP makes it O(n) if pressed. |

**Pitfalls:**
- `pi[i]` is the longest **proper** border — `pi[0] = 0` always; initializing it to 1 (or letting a border equal the whole prefix) silently breaks the chain logic.
- The fallback is `j = pi[j-1]`, not `pi[i-1]` and not `j-1` — the single most common transcription bug; if your KMP loops forever or misses matches, look here first.
- Forgetting the `#` separator in the concatenation tricks (search, 214): without it, borders can span the boundary and pi can exceed the pattern length — garbage results that *usually* pass small tests.
- Complexity claims: building pi is amortized O(n) — say "amortized, j decreases by at least the fallback amount" if challenged; a per-iteration bound doesn't exist.

---

## Pattern 4: Rolling Hash (Rabin-Karp) — Strings as Numbers

**Logic:** Encode `s[l..r]` as a polynomial-in-base-B number mod M; precompute prefix hashes and powers so any substring hash is O(1): `h(l,r) = h[r+1] − h[l]·B^(r−l+1)`. Now substring equality is (probable) integer equality — which unlocks sliding comparisons (Rabin-Karp search), hashset dedup of substrings (187), and the killer combo: **binary search over length + hash-set check** ("longest duplicate substring"), since "a duplicate of length L exists" is monotone in L.

**Core insight — why it works:** A polynomial hash is injective-in-spirit: equal strings always hash equal (no false negatives), unequal strings collide with probability ~1/M per comparison — so the algorithm is Monte Carlo, and the honesty about that *is* the interview point (KMP when exactness is mandatory; double-hash or verify-on-match to drive error below concern). What hashing buys over KMP is *composability*: KMP answers "does this one pattern occur," while hashes turn every substring into a set-insertable value — enabling dedup, counting distinct, comparing across strings, and the monotone binary search on answer length, none of which the prefix function does naturally. That trade — exactness for algebraic flexibility — is the pattern in one sentence.

**Template (prefix hashes + O(1) substring hash):**
```cpp
const long long B = 131, MOD = 1'000'000'007;
int n; vector<long long> h, p;                 // h[i] = hash of s[0..i-1], p[i] = B^i
void build(const string& s) {
    n = s.size(); h.assign(n + 1, 0); p.assign(n + 1, 1);
    for (int i = 0; i < n; i++) {
        h[i + 1] = (h[i] * B + s[i]) % MOD;
        p[i + 1] = p[i] * B % MOD;
    }
}
long long get(int l, int r) {                  // hash of s[l..r], inclusive
    return ((h[r + 1] - h[l] * p[r - l + 1]) % MOD + MOD) % MOD;  // the +MOD saves you
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 28. First Occurrence (Rabin-Karp flavor) | Easy | Slide a window hash over the text, compare against the pattern hash, verify on match. Write it once next to your KMP to internalize the trade: simpler code, probabilistic guarantee. |
| 187. Repeated DNA Sequences | Medium | All length-10 substrings into a hashset (or the 2-bit encoding into an int — 20 bits total, exact and collision-free). The "encode a fixed window as a number" special case. |
| 1044. Longest Duplicate Substring | Hard | The flagship: binary search the length, hash-set all substrings of that length, verify existence. O(n log n) expected — and the problem that *forces* hashing (suffix automatons aside). |
| 718. Maximum Length of Repeated Subarray | Medium | Same binary-search-plus-hash-set, across *two* sequences. Also solvable by DP (O(nm)) — offering both with the size-based cutover is the strong answer. |
| 1316. Distinct Echo Substrings | Hard | Count substrings of the form a+a: for each even length, compare adjacent half-hashes; dedup with a set of hashes. Hash algebra (compare, dedup) composed — exactly what KMP can't do. |
| 686. Repeated String Match (hash flavor) | Medium | Cross-listed from Pattern 3 — search via hashes instead; identical bound argument. Doing it both ways is the drill. |

**Pitfalls:**
- Negative mod: `(a - b) % MOD` is negative in C++ when b > a — the `+ MOD` in the template is not optional; its absence is the classic silently-wrong hash.
- Overflow: `h[l] * p[len]` needs 64-bit — with MOD ~1e9 the product fits `long long` (barely, ~1e18); with larger mods use `__int128` or a 61-bit Mersenne mod.
- Collision honesty: on adversarial judges (1044 included) single-hash solutions get hacked — say "double hash or verify-on-match" *before* the interviewer raises it.
- Base/mod hygiene: base must exceed the alphabet and be coprime-ish with the mod; base 26 with 'a' mapped to 0 makes "a", "aa", "aaa" all hash 0 — map characters to 1..26, not 0..25.

---

## Pattern 5: The Linear-Time Toolkit — Z-Function & Manacher

**Logic:** Two specialized O(n) engines. **Z-function**: `z[i]` = length of the longest substring starting at i that matches a *prefix* of s — computed in O(n) by maintaining the rightmost known match window `[l, r]` and initializing each `z[i]` from its mirror inside it. **Manacher**: the same maintain-a-window idea applied to palindromes — for each center, initialize the palindrome radius from the mirror center inside the rightmost known palindrome, then extend. Longest palindromic substring in O(n), all palindromic centers counted exactly.

**Core insight — why it works:** Both algorithms share one principle: **previously computed matches constrain future ones, so never re-compare inside a region you've already matched.** If `[l, r]` is a known prefix-match (or palindrome), position i inside it behaves — up to the window edge — exactly like its mirror, so you start from the mirror's answer and only extend past r, and since r only moves forward, total extension work is O(n). Z is the "non-amortized KMP": it answers the same matching questions with simpler, direct reasoning (search = Z on `pattern#text`, `z[i] = m` at matches), and interviewers accept either. Manacher's odd/even unification via `#`-interleaving ("aba" → "^#a#b#a#$") makes every palindrome odd-length in the transformed string — one loop instead of two center types. Honest positioning: expand-around-center (two-pointers guide) is the O(n²) interview default for 5/647; Manacher is the named upgrade you *mention* and code only on explicit request — knowing when not to deploy the heavy tool is part of mastering it.

**Template (Z-function):**
```cpp
vector<int> zFunction(const string& s) {
    int n = s.size();
    vector<int> z(n, 0);
    for (int i = 1, l = 0, r = 0; i < n; i++) {
        if (i < r) z[i] = min(r - i, z[i - l]);        // start from the mirror's answer
        while (i + z[i] < n && s[z[i]] == s[i + z[i]]) z[i]++;   // extend past r only
        if (i + z[i] > r) { l = i; r = i + z[i]; }     // window only moves right → O(n)
    }
    return z;
}
```

**Problems:**
| Problem | Difficulty | Note |
|---|---|---|
| 28. First Occurrence (Z flavor) | Easy | Z on `pattern + '#' + text`; matches where z equals the pattern length. Third solution to the same problem — 28 is this guide's Rosetta stone (brute, KMP, hash, Z). |
| 2223. Sum of Scores of Built Strings | Hard | The definition of the Z-array as a problem statement: answer = n + Σz[i]. Exists purely to reward knowing the tool — the 1392 of Z-functions. |
| 5. Longest Palindromic Substring (Manacher flavor) | Medium | Interview default is expand-around-center (two-pointers guide); Manacher is the O(n) named upgrade. Say the name, sketch the mirror argument, code only on request. |
| 647. Palindromic Substrings (Manacher flavor) | Medium | Each transformed-position radius counts `(radius+1)/2` original palindromes — counting falls out of the radii for free. Same positioning as 5. |
| 214. Shortest Palindrome (Z flavor) | Hard | Z on `s + '#' + reverse(s)`: the longest palindromic prefix is the largest suffix-position where z reaches the string's end. Same construction as the KMP version — two tools, one trick; know it as *one* idea. |

**Pitfalls:**
- Z's `min(r - i, z[i - l])` cap: dropping the `r - i` bound copies mirror information from *beyond* the verified window — subtly wrong on strings like "aaaa…" where every test still passes until it doesn't.
- The window updates only when `i + z[i] > r` — updating unconditionally breaks the O(n) argument (and occasionally correctness).
- Manacher's sentinels (`^`, `$`) exist so the extension loop needs no bounds checks — drop them and add the checks, or keep them; half-doing both is the crash.
- Deploying Manacher unprompted on 5 in a 40-minute interview: high risk, zero extra credit versus a clean O(n²) expansion plus the sentence "Manacher does this in O(n) via mirrored radii." Tool selection is being graded too.

---

## Where String-LOOKING Problems Actually Live — The Routing Table

| Phrase in the problem | It's actually | Guide |
|---|---|---|
| "longest/shortest **substring** with property" | Variable-size window | Sliding window |
| "find all anagrams **in** a string / permutation in string" | Fixed window + counts | Sliding window |
| "expand around / count palindromic substrings (interview default)" | Center expansion | Two pointers |
| "valid parentheses / decode / remove adjacent" | Matching & nesting | Stack |
| "smallest subsequence / remove k digits, keep order" | Monotonic stack, lexicographic greedy | Stack |
| "edit distance / LCS / interleaving / distinct subsequences" | 2-D alignment DP | DP |
| "word break" | 1-D DP / memoized DFS | DP · DFS |
| "many words, prefix queries, autocomplete" | Trie | Remaining essentials |
| "generate all valid strings / partitions" | Choose/explore/unchoose | Backtracking |
| "reverse words / compress in place / merge strings" | Read/write pointers | Two pointers |
| "custom sort order / largest number from pieces" | Comparator greedy | Greedy |
| "serialize a tree" | Tree construction | Binary Tree |
| "is it an anagram / group by rearrangement / isomorphic" | **Pattern 1 here** | — |
| "atoi / validate format / roman / justify" | **Pattern 2 here** | — |
| "pattern occurs in text / repeats itself / longest border" | **Pattern 3 here** | — |
| "longest duplicate substring / distinct substrings / echo" | **Pattern 4 here** | — |
| "O(n) palindromes / prefix-match array" | **Pattern 5 here** | — |

---

## Complexity Summary

- Canonical forms: O(n·L) with count-vector keys (O(n·L log L) with sorted keys) for n strings of length L.
- Parsing/state machines: O(n) single pass — the *complexity* is never the question; the edge cases are.
- Prefix function / Z-function: **O(n)** build (amortized for pi, windowed for Z); search O(n + m) with zero false positives.
- Rolling hash: O(n) build, O(1) per substring hash; search O(n + m) *expected*; collision probability ~comparisons/MOD — double-hash to square it away.
- Binary search + hash set (1044-style): O(n log n) expected time, O(n) space per probe.
- Manacher: O(n) — each of the 2n+1 transformed centers extends the right boundary monotonically.
- C++ reality check: `substr` copies (O(len)) — inside loops prefer indices/`string_view`; `s += c` is amortized O(1) but `s = s + c` is O(n). These bite in exactly the problems on this sheet.

---

## Interview Tips

1. **Route first, out loud.** "This is an anagram-grouping problem — canonical form plus hashmap," or "this is a window problem, not a string problem." Thirty seconds of routing beats ten minutes in the wrong pattern.
2. **Name the canonical form and what it ignores** — "count vector, because anagram-equivalence ignores order and nothing else." That sentence is the correctness proof.
3. **For parsing, enumerate edge inputs before coding** (" -042", "1.0 vs 1.0.0", the all-spaces string). Interviewers grade the enumeration as much as the code.
4. **KMP vs hashing is a stated trade-off**: exact and single-pattern (KMP) versus probabilistic and composable (hashing — sets, dedup, binary search on length). Choosing with reasons is the senior signal; coding either from memory is table stakes.
5. **Say "amortized" about the pi-function build** and "expected / Monte Carlo" about hashing — precision about guarantee *types* distinguishes candidates at the top end.
6. **Know 28 four ways** (brute, KMP, Rabin-Karp, Z) — it's the cheapest possible rehearsal of this entire sheet, and "now do it another way" is a real follow-up.
7. **Don't deploy Manacher unprompted** — offer the O(n²) expansion, name the O(n) upgrade, and let the interviewer choose. Tool restraint reads as experience.

---

## Suggested Practice Order

**Week 1 — signatures & mappings:** 242 → 383 → 49 → 205 → 290 → 890 → 1657
**Week 2 — parsing gauntlet:** 13 → 165 → 8 → 6 → 38 → 468 → 12 → 68 → 65
**Week 3 — KMP:** 28 (brute, then KMP) → 1392 → 459 → 796 → 686 → 214
**Week 4 — hashing & the toolkit:** 28 (hash, then Z) → 187 → 718 → 1044 → 1316 → 2223 → 5/647 (Manacher pass)

Good luck with the interviews!

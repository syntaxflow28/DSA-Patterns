# Numeric, Math, Bit Manipulation & Overflow

Everything numeric that shows up in solutions: `<numeric>` helpers, `<cmath>` traps, integer limits, and the bit tricks that power bitmask DP and subset enumeration.

---

## `<numeric>`

```cpp
#include <numeric>

accumulate(v.begin(), v.end(), 0LL);                       // sum
accumulate(v.begin(), v.end(), 1LL, multiplies<long long>());
accumulate(v.begin(), v.end(), 0, [](int a, int b){ return a ^ b; });
accumulate(strs.begin(), strs.end(), string(""));          // string concatenation

iota(v.begin(), v.end(), 0);                               // 0,1,2,3,... — DSU parent init

partial_sum(v.begin(), v.end(), pre.begin());              // in-place prefix sums
adjacent_difference(v.begin(), v.end(), d.begin());        // d[i] = v[i] - v[i-1]

inner_product(a.begin(), a.end(), b.begin(), 0LL);         // dot product

gcd(a, b);                                                  // C++17, <numeric>
lcm(a, b);                                                  // C++17 — can overflow, use long long
```

**`accumulate`'s init value fixes the return type.** `accumulate(v.begin(), v.end(), 0)` returns `int` and overflows silently even if you assign it to a `long long`. Always write `0LL`.

Manual prefix sums are usually clearer than `partial_sum` because the 1-indexed form removes the `i == 0` special case:

```cpp
vector<long long> pre(n + 1, 0);
for (int i = 0; i < n; i++) pre[i + 1] = pre[i] + nums[i];
long long sum = pre[r + 1] - pre[l];        // sum of nums[l..r]
```

---

## Integer Limits — `<climits>` / `<limits>`

```cpp
INT_MAX     2147483647          (~2.1e9)
INT_MIN    -2147483648
LLONG_MAX   9223372036854775807 (~9.2e18)
LLONG_MIN
UINT_MAX    4294967295

numeric_limits<int>::max();
numeric_limits<double>::infinity();
```

**Never use `INT_MAX` as an "infinity" you then add to** — `INT_MAX + 1` overflows. Use a padded sentinel:

```cpp
const int INF = 1e9;              // 1e9 + 1e9 = 2e9 still fits in int... barely. Prefer:
const long long INF = 1e18;       // safe for long long arithmetic
const int INF = 0x3f3f3f3f;       // ~1.06e9, and memset(dp, 0x3f, sizeof dp) sets it
```

| Type | Range | Use when |
|---|---|---|
| `int` | ±2.1e9 | default |
| `long long` | ±9.2e18 | sums, products, anything near 1e9 |
| `unsigned` | 0..4.3e9 | bit tricks; avoid for arithmetic |
| `double` | ~15 significant digits | geometry, averages |
| `long double` | ~18 digits | rarely needed |

### Overflow — The Rules

```cpp
int a = 1e9, b = 1e9;
long long c = a + b;               // BUG: a + b is computed as int, overflows, THEN widens
long long c = (long long)a + b;    // correct — cast one operand
long long c = 1LL * a * b;         // idiomatic multiplication guard

int mid = (lo + hi) / 2;           // BUG when lo + hi > INT_MAX
int mid = lo + (hi - lo) / 2;      // correct
```

**The type of an expression is decided by its operands, not by where you store it.** Cast *before* the operation, never after.

Signed overflow is undefined behaviour in C++ — the compiler may optimize on the assumption it can't happen, so it can produce results that look impossible.

---

## `<cmath>`

```cpp
abs(x);        // int
llabs(x);      // long long
fabs(x);       // double
pow(a, b);     // returns double — see the warning below
sqrt(x);       // double
cbrt(x);
ceil(x);  floor(x);  round(x);     // all return double
log(x);  log2(x);  log10(x);  exp(x);
INFINITY;  NAN;  isnan(x);  isinf(x);
```

**`pow` returns a `double` and loses precision.** `(int)pow(10, 2)` can evaluate to `99`. For integer powers, write your own:

```cpp
long long ipow(long long b, long long e) {
    long long r = 1;
    while (e) { if (e & 1) r *= b; b *= b; e >>= 1; }
    return r;
}
```

**`sqrt` has the same problem.** For integer square roots:

```cpp
long long r = (long long)sqrtl(n);
while (r * r > n) r--;
while ((r + 1) * (r + 1) <= n) r++;
```

**Comparing doubles with `==` is always wrong.** Use `fabs(a - b) < 1e-9`.

---

## Integer Division & Modulo

```cpp
 7 /  2 ==  3
-7 /  2 == -3        // truncates TOWARD ZERO, not floor
 7 %  3 ==  1
-7 %  3 == -1        // the result takes the sign of the DIVIDEND
```

```cpp
// Ceiling division for non-negative a, b
int c = (a + b - 1) / b;
// Floor division that works for negatives
int fdiv = (a >= 0) ? a / b : -((-a + b - 1) / b);
// Always-positive modulo
int m = ((a % b) + b) % b;
```

The always-positive modulo shows up constantly in circular-array and modular-arithmetic problems.

### Modular Arithmetic

```cpp
const long long MOD = 1e9 + 7;
long long add(long long a, long long b) { return (a + b) % MOD; }
long long sub(long long a, long long b) { return ((a - b) % MOD + MOD) % MOD; }
long long mul(long long a, long long b) { return a % MOD * (b % MOD) % MOD; }

long long power(long long b, long long e, long long m) {   // fast exponentiation
    long long r = 1; b %= m;
    while (e) { if (e & 1) r = r * b % m; b = b * b % m; e >>= 1; }
    return r;
}
long long inv(long long a) { return power(a, MOD - 2, MOD); }   // modular inverse, MOD prime
```

`sub` needs the `+ MOD` because C++'s `%` can return a negative. This is the single most common modular-arithmetic bug.

---

## Bit Manipulation

### The Core Operations

| Task | Expression |
|---|---|
| Get bit i | `(x >> i) & 1` |
| Set bit i | `x \|= (1 << i)` |
| Clear bit i | `x &= ~(1 << i)` |
| Toggle bit i | `x ^= (1 << i)` |
| Is bit i set | `x & (1 << i)` |
| Lowest set bit (value) | `x & -x` |
| Clear the lowest set bit | `x & (x - 1)` |
| Is a power of two | `x > 0 && (x & (x - 1)) == 0` |
| All ones below bit n | `(1 << n) - 1` |
| Check even | `!(x & 1)` |
| Multiply / divide by 2 | `x << 1` / `x >> 1` |

**Use `1LL << i` whenever `i` can be ≥ 31.** `1 << 31` is UB for signed `int`; `1 << 64` is UB always.

### Builtins (GCC/Clang — available on LeetCode)

```cpp
__builtin_popcount(x);        // number of set bits (unsigned int)
__builtin_popcountll(x);      // long long version
__builtin_clz(x);             // leading zeros — UB if x == 0
__builtin_ctz(x);             // trailing zeros — UB if x == 0
__builtin_parity(x);          // popcount % 2

int highestBitPos = 31 - __builtin_clz(x);          // floor(log2(x)), x > 0
int lowestBitPos  = __builtin_ctz(x);
```

C++20 portable equivalents in `<bit>`:

```cpp
#include <bit>
popcount(x);  countl_zero(x);  countr_zero(x);
bit_width(x);          // 1 + floor(log2(x))
has_single_bit(x);     // is a power of two
bit_ceil(x);           // next power of two >= x
```

These require **unsigned** arguments.

### `bitset<N>` — `<bitset>`

```cpp
#include <bitset>
bitset<32> b(x);                 // from an int
bitset<32> b("1011");            // from a string
b[i];  b.set(i);  b.reset(i);  b.flip(i);
b.count();                       // popcount
b.any();  b.none();  b.all();
b.to_string();  b.to_ulong();  b.to_ullong();
b1 & b2;  b1 | b2;  b1 ^ b2;  b << 2;
```

Size must be a compile-time constant. Its real superpower is O(n/64) set operations — a `bitset<100000>` subset-sum DP is 64× faster than `vector<bool>`.

### Subset Enumeration

```cpp
// All 2^n subsets of {0..n-1}
for (int mask = 0; mask < (1 << n); mask++) {
    for (int i = 0; i < n; i++)
        if (mask & (1 << i)) use(i);
}

// All submasks of `mask` — O(3^n) over all masks, not O(4^n)
for (int sub = mask; sub > 0; sub = (sub - 1) & mask) {
    // sub runs over every non-empty subset of mask
}

// Iterate set bits only
for (int m = mask; m; m &= m - 1) {
    int i = __builtin_ctz(m);
    use(i);
}
```

The submask loop is the engine behind partition-style bitmask DP (LC 698, 1723, 1986).

### Classic Bit Problems

```cpp
// Single Number (LC 136) — XOR cancels pairs
int res = 0; for (int x : nums) res ^= x;

// Swap without a temp
a ^= b; b ^= a; a ^= b;

// XOR of 1..n in O(1)
int xorN(int n) { int r[] = {n, 1, n + 1, 0}; return r[n % 4]; }

// Sum of two integers without + (LC 371)
while (b) { unsigned carry = (unsigned)(a & b) << 1; a = a ^ b; b = carry; }

// Reverse 32 bits (LC 190)
uint32_t res = 0;
for (int i = 0; i < 32; i++) { res = (res << 1) | (n & 1); n >>= 1; }
```

XOR facts worth memorizing: `x ^ x == 0`, `x ^ 0 == x`, XOR is commutative and associative.

---

## Randomness

```cpp
#include <random>
mt19937 rng(chrono::steady_clock::now().time_since_epoch().count());
int r = rng() % n;                          // biased but fine for shuffling
uniform_int_distribution<int> dist(1, 6);
int roll = dist(rng);

shuffle(v.begin(), v.end(), rng);           // NOT random_shuffle (removed in C++17)
```

Needed for LC 384 Shuffle an Array, LC 528 Weighted Random Pick, and randomized-pivot Quickselect.

---

## Numeric Gotchas Checklist

- `accumulate(..., 0)` → `int` overflow. Write `0LL`.
- `(lo + hi) / 2` overflows. Write `lo + (hi - lo) / 2`.
- `a * b` where both are `int` overflows *before* being stored in a `long long`. Write `1LL * a * b`.
- `pow(10, 2)` may be `99.999...`. Never cast `pow` to `int`.
- `-7 % 3 == -1`, not `2`. Use `((a % b) + b) % b`.
- `1 << 31` is UB. Use `1LL << 31`.
- `INT_MAX + 1` is UB. Use a padded `INF`.
- `v.size() - 1` on an empty vector is a huge unsigned number.
- `double == double` is unreliable. Compare against an epsilon.
- `__builtin_clz(0)` and `__builtin_ctz(0)` are UB. Guard for zero.

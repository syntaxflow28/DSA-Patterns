# Numbers, Math & Bit Manipulation

Java's integer arithmetic is fully specified — signed overflow *wraps* deterministically rather than being undefined like C++. That's a mixed blessing: no crash, no warning, just a silently wrong answer.

---

## Ranges

| Type | Bits | Range |
|---|---|---|
| `byte` | 8 | −128 … 127 |
| `short` | 16 | ±32 767 |
| `char` | 16 | 0 … 65 535 (**unsigned**) |
| `int` | 32 | ±2 147 483 647 (~2.1e9) |
| `long` | 64 | ±9.22e18 |
| `float` | 32 | ~7 digits |
| `double` | 64 | ~15 digits |

```java
Integer.MAX_VALUE      2147483647
Integer.MIN_VALUE     -2147483648
Long.MAX_VALUE         9223372036854775807L
Double.MAX_VALUE
Integer.MAX_VALUE / 2  // the practical "infinity" — survives one addition
```

`char` is the only unsigned type in Java.

---

## Overflow — The Rules

```java
int a = 1_000_000_000, b = 2_000_000_000;
int  s1 = a + b;                  // wraps to a negative number, no warning
long s2 = a + b;                  // STILL WRONG — the addition happens in int, then widens
long s3 = (long) a + b;           // correct
long s4 = 1L * a * b;             // correct — the leading 1L promotes the whole expression
```

**The expression's type comes from its operands, never from the variable you assign to.** Widen *before* the operation.

```java
int mid = (lo + hi) / 2;          // overflows when lo + hi > 2.1e9
int mid = lo + (hi - lo) / 2;     // correct
int mid = (lo + hi) >>> 1;        // correct — unsigned shift, and this is the JDK's own idiom
```

`>>>` treats the wrapped bit pattern as unsigned, so the halved value comes out right even after the sum overflows. It only works for non-negative `lo` and `hi`, which is the binary-search case.

Detecting overflow explicitly (Java 8+, throws `ArithmeticException`):

```java
Math.addExact(a, b);
Math.multiplyExact(a, b);
Math.subtractExact(a, b);
Math.toIntExact(longValue);       // throws instead of silently truncating
```

Useful in problems that ask you to *detect* overflow rather than avoid it.

**Rule of thumb:** if any intermediate value can exceed ~2e9, make the whole computation `long`. Prefix sums, products, and squared distances almost always need it.

---

## Division and Modulo

```java
 7 /  2 ==  3
-7 /  2 == -3          // truncates TOWARD ZERO
 7 %  3 ==  1
-7 %  3 == -1          // the sign follows the DIVIDEND

Math.floorDiv(-7, 2)   == -4      // true floor division
Math.floorMod(-7, 3)   ==  2      // always non-negative for a positive modulus
```

**`Math.floorMod` is the fix for circular indexing** — no `((i % n) + n) % n` dance needed:

```java
int prev = Math.floorMod(i - 1, n);        // wraps correctly at i == 0
```

Ceiling division:

```java
int c = (a + b - 1) / b;             // non-negative a, b
int c = Math.ceilDiv(a, b);          // Java 18+
```

Integer division by zero throws `ArithmeticException`. Floating-point division by zero gives `Infinity` or `NaN` instead.

---

## `Math`

```java
Math.abs(x);                 // careful: Math.abs(Integer.MIN_VALUE) is still MIN_VALUE
Math.max(a, b);  Math.min(a, b);
Math.pow(a, b);              // returns DOUBLE — precision loss
Math.sqrt(x);                // double
Math.cbrt(x);
Math.ceil(x);  Math.floor(x);  Math.round(x);   // round returns long for a double
Math.log(x);  Math.log10(x);  Math.exp(x);
Math.hypot(a, b);
Math.signum(x);
Math.random();               // [0, 1)
```

**`Math.pow` returns a `double`.** `(int) Math.pow(10, 2)` can evaluate to `99`. Write your own integer power:

```java
static long power(long b, long e) {
    long r = 1;
    while (e > 0) { if ((e & 1) == 1) r *= b; b *= b; e >>= 1; }
    return r;
}
```

Same for `Math.sqrt` — adjust the result:

```java
long r = (long) Math.sqrt(n);
while (r * r > n) r--;
while ((r + 1) * (r + 1) <= n) r++;
```

**`Math.abs(Integer.MIN_VALUE)` returns `Integer.MIN_VALUE`** — there is no positive counterpart in two's complement. It bites in "absolute difference" problems with extreme values.

**Never compare doubles with `==`.** Use `Math.abs(a - b) < 1e-9`, or restructure to integer arithmetic (compare `a*d` vs `c*b` instead of `a/b` vs `c/d`).

---

## `Integer` / `Long` Statics

These are where Java quietly beats C++ for bit work.

```java
Integer.parseInt(s);            Integer.parseInt(s, radix);
Integer.toString(i);            Integer.toString(i, radix);
Integer.toBinaryString(i);      Integer.toHexString(i);      Integer.toOctalString(i);
Integer.valueOf(i);             // boxed — cached for -128..127
Integer.compare(a, b);
Integer.MAX_VALUE / MIN_VALUE;

Integer.bitCount(x);            // popcount
Integer.reverse(x);             // reverse all 32 bits — LC 190 in one call
Integer.reverseBytes(x);
Integer.highestOneBit(x);       // the VALUE of the highest set bit (0 if x == 0)
Integer.lowestOneBit(x);        // == x & -x
Integer.numberOfLeadingZeros(x);
Integer.numberOfTrailingZeros(x);
Integer.signum(x);

Long.bitCount(x);  Long.compare(a, b);  Long.parseLong(s);   // same family for long
```

Unlike C++'s `__builtin_clz`, these are **defined for 0** — `Integer.numberOfTrailingZeros(0)` returns 32, not undefined behaviour.

```java
int log2 = 31 - Integer.numberOfLeadingZeros(x);         // floor(log2(x)), x > 0
boolean isPow2 = x > 0 && Integer.bitCount(x) == 1;
int nextPow2 = Integer.highestOneBit(x - 1) << 1;
```

---

## Bit Manipulation

| Task | Java |
|---|---|
| Get bit i | `(x >> i) & 1` |
| Set bit i | `x \|= (1 << i)` |
| Clear bit i | `x &= ~(1 << i)` |
| Toggle bit i | `x ^= (1 << i)` |
| Test bit i | `(x & (1 << i)) != 0` |
| Lowest set bit (value) | `x & -x` |
| Clear the lowest set bit | `x & (x - 1)` |
| Power of two | `x > 0 && (x & (x - 1)) == 0` |
| n low bits set | `(1 << n) - 1` |
| Count set bits | `Integer.bitCount(x)` |

### Three Shift Operators

```java
x << n     // left shift
x >> n     // ARITHMETIC right shift — preserves the sign bit
x >>> n    // LOGICAL right shift — fills with zeros
```

```java
-8 >> 1   == -4
-8 >>> 1  == 2147483644
```

C++ has no `>>>`. Use `>>>` whenever you're treating an `int` as a bit pattern rather than a number — bitmask iteration, hashing, midpoint computation.

**Shift counts are taken modulo the width.** `1 << 32` is `1`, not `0`. And `1 << 33` is `2`. For bit 32 and beyond, use `long`:

```java
long mask = 1L << 40;          // NOT 1 << 40
```

`1 << 31` is `Integer.MIN_VALUE` — legal in Java (unlike C++'s UB), but rarely what you meant.

### Subset Enumeration

```java
// All 2^n subsets
for (int mask = 0; mask < (1 << n); mask++) {
    for (int i = 0; i < n; i++)
        if ((mask & (1 << i)) != 0) use(i);
}

// Iterate only the set bits
for (int m = mask; m != 0; m &= m - 1) {
    int i = Integer.numberOfTrailingZeros(m);
    use(i);
}

// All submasks of mask — O(3^n) over all masks
for (int sub = mask; sub > 0; sub = (sub - 1) & mask) { }
```

**`(mask & (1 << i)) != 0`, not `== 1`.** The masked result is the bit's *value* (`1 << i`), not `1`. Java requires the explicit boolean comparison anyway — `if (mask & (1 << i))` doesn't compile, which is a nice guardrail C++ lacks.

### Classic Problems

```java
// Single Number (LC 136)
int r = 0; for (int x : nums) r ^= x;

// Sum without + (LC 371)
while (b != 0) { int carry = (a & b) << 1; a = a ^ b; b = carry; }

// Hamming distance
Integer.bitCount(x ^ y);

// Reverse 32 bits (LC 190)
Integer.reverse(n);                 // or the manual loop, if they ask
```

---

## `BitSet` — For Large Bit Arrays

```java
BitSet bs = new BitSet(n);
bs.set(i);  bs.clear(i);  bs.flip(i);
bs.get(i);
bs.cardinality();                 // popcount
bs.nextSetBit(from);              // -1 when there are none
bs.or(other);  bs.and(other);  bs.xor(other);  bs.andNot(other);
```

Grows automatically and packs 64 bits per word, so bulk set operations run in O(n/64). Worth it for subset-sum DP or large sieve problems; a `boolean[]` is simpler for plain visited flags.

---

## Modular Arithmetic

```java
static final int MOD = 1_000_000_007;

long add = (a + b) % MOD;
long sub = ((a - b) % MOD + MOD) % MOD;        // the + MOD is mandatory
long mul = a % MOD * (b % MOD) % MOD;          // both operands must be long

static long power(long b, long e, long m) {
    long r = 1; b %= m;
    while (e > 0) { if ((e & 1) == 1) r = r * b % m; b = b * b % m; e >>= 1; }
    return r;
}
static long inverse(long a) { return power(a, MOD - 2, MOD); }   // MOD must be prime
```

Declare accumulators as `long` even when the final answer is an `int` — `(int)(result % MOD)` at the end. Two values under 1e9 multiplied in `int` overflow instantly.

`Math.floorMod` is the cleaner alternative to `((a - b) % MOD + MOD) % MOD`.

---

## `BigInteger` / `BigDecimal`

```java
import java.math.BigInteger;
BigInteger a = BigInteger.valueOf(x);
BigInteger b = new BigInteger("123456789012345678901234567890");
a.add(b);  a.multiply(b);  a.mod(m);  a.pow(n);  a.gcd(b);
a.compareTo(b);  a.equals(b);
a.intValue();  a.longValue();  a.toString();
BigInteger.ZERO;  BigInteger.ONE;
```

Immutable — every operation returns a new object. Slow, but it makes factorial, big-number-multiply, and arbitrary-precision problems trivial when the constraints allow. Prefer `long` plus modular arithmetic when you can.

---

## Randomness

```java
Random rnd = new Random();
rnd.nextInt(n);            // [0, n)
rnd.nextInt(a, b);         // [a, b), Java 17+
rnd.nextDouble();
ThreadLocalRandom.current().nextInt(a, b);        // faster, no shared state

// Fisher-Yates shuffle
for (int i = a.length - 1; i > 0; i--) {
    int j = rnd.nextInt(i + 1);
    int t = a[i]; a[i] = a[j]; a[j] = t;
}
Collections.shuffle(list);
```

Needed for LC 384 Shuffle an Array, LC 528 Weighted Random Pick, randomized Quickselect, and as the defence against `Arrays.sort(int[])`'s quadratic worst case.

---

## Gotchas Checklist

- `long x = a + b;` with `int` operands overflows *before* widening. Cast first.
- `(lo + hi) / 2` overflows. Use `(lo + hi) >>> 1`.
- `Math.pow(10, 2)` may be `99.999…`. Never cast it to `int`.
- `-7 % 3 == -1`. Use `Math.floorMod`.
- `Math.abs(Integer.MIN_VALUE)` is negative.
- `1 << 32 == 1` — shifts wrap. Use `1L` past bit 30.
- `>>` keeps the sign; `>>>` doesn't. Bit-pattern work wants `>>>`.
- `(mask & bit) != 0`, not `== 1`.
- `Integer.MAX_VALUE` as infinity overflows on the first addition. Use `Integer.MAX_VALUE / 2`.
- `double == double` is unreliable. Compare against an epsilon.
- Integer `/` by zero throws; `double` `/` by zero gives `Infinity`.

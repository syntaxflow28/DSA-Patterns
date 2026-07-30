# Arrays & Primitives

Java has two parallel worlds: primitives (`int`, `char`, `long`) and objects (`Integer`, `Character`, `Long`). Arrays of primitives are fast and memory-tight; collections can only hold objects. Knowing when you're crossing that line is most of Java performance work.

---

## Declaring and Initializing

```java
int[] a = new int[n];               // n zeros
int[] a = {1, 2, 3};                // literal — only at declaration
int[] a = new int[]{1, 2, 3};       // literal anywhere (arguments, returns)
boolean[] vis = new boolean[n];     // all false
char[] c = new char[n];             // all '\u0000'
String[] s = new String[n];         // all null
long[] dp = new long[n];            // 0L

int[][] g = new int[r][c];          // r rows × c columns, all zeros
int[][] g = {{1,2},{3,4}};
int[][] jag = new int[r][];         // rows unallocated — jagged
jag[0] = new int[5];

int[][][] dp3 = new int[n][m][k];
```

Default values are guaranteed: `0`, `0L`, `0.0`, `false`, `'\u0000'`, `null`. No uninitialized-garbage bugs, unlike C++.

```java
a.length          // FIELD, no parentheses — arrays
s.length()        // METHOD — String
list.size()       // METHOD — collections
```

Three different spellings for the same idea. Getting them wrong is a compile error, so it costs seconds, not correctness.

For 2D: `g.length` is the row count, `g[0].length` is the column count.

---

## `java.util.Arrays`

```java
Arrays.fill(a, -1);                       // whole array
Arrays.fill(a, from, to, -1);             // [from, to)
Arrays.sort(a);                           // ascending
Arrays.sort(a, from, to);                 // sort a subrange
Arrays.toString(a);                       // "[1, 2, 3]" — debugging
Arrays.deepToString(g);                   // 2D debugging
Arrays.equals(a, b);                      // element-wise; a == b compares references
Arrays.deepEquals(g, h);                  // 2D
Arrays.hashCode(a);  Arrays.deepHashCode(g);
Arrays.stream(a).sum();                   // int[] -> IntStream
Arrays.asList(1, 2, 3);                   // FIXED-SIZE List view — see the trap below
```

### Copying

```java
int[] b = a.clone();                                  // 1D deep copy
int[] b = Arrays.copyOf(a, n);                        // resize: truncate or pad with 0
int[] b = Arrays.copyOfRange(a, l, r);                // [l, r)
System.arraycopy(src, srcPos, dst, dstPos, len);      // fastest bulk copy
```

**`clone()` on a 2D array is shallow** — it copies the row *references*, so mutating `b[0][0]` also changes `a[0][0]`:

```java
int[][] copy = new int[g.length][];
for (int i = 0; i < g.length; i++) copy[i] = g[i].clone();     // correct deep copy
```

**`Arrays.fill` on a 2D array doesn't work either** — the elements are row references, not ints:

```java
for (int[] row : g) Arrays.fill(row, -1);              // this is the way
```

Both bite in memoization setup (`Arrays.fill(memo, -1)`) and in backtracking that snapshots a grid.

### Binary Search

```java
int i = Arrays.binarySearch(a, key);       // array MUST be sorted
```

The return value is unusual: if found, the index; **if not found, `-(insertionPoint) - 1`** (a negative number). To recover the insertion point:

```java
int idx = Arrays.binarySearch(a, key);
int insertAt = idx >= 0 ? idx : -idx - 1;
```

There is **no `lowerBound`/`upperBound` for arrays** and `binarySearch` gives no guarantee about *which* duplicate it finds. Write your own — it's four lines and you need the pattern anyway:

```java
// first index with a[i] >= key
static int lowerBound(int[] a, int key) {
    int lo = 0, hi = a.length;
    while (lo < hi) { int mid = (lo + hi) >>> 1; if (a[mid] < key) lo = mid + 1; else hi = mid; }
    return lo;
}
// first index with a[i] > key
static int upperBound(int[] a, int key) {
    int lo = 0, hi = a.length;
    while (lo < hi) { int mid = (lo + hi) >>> 1; if (a[mid] <= key) lo = mid + 1; else hi = mid; }
    return lo;
}
```

`>>> 1` is the unsigned right shift — it makes `(lo + hi) >>> 1` overflow-safe even when `lo + hi` exceeds `Integer.MAX_VALUE`. This is Java's idiomatic midpoint, and it's what the JDK itself uses.

For objects, `TreeSet`/`TreeMap` give you `floor`/`ceiling` directly — see [collections.md](collections.md).

---

## The `Arrays.sort` Trap

`Arrays.sort(int[])` uses **dual-pivot quicksort**: fast on average, **O(n²)** on inputs crafted to hit the pivot pattern. LeetCode has anti-quicksort test cases and they will TLE an otherwise correct solution.

`Arrays.sort(Object[])` and `Collections.sort` use **TimSort** — O(n log n) guaranteed, and stable.

Three fixes:

```java
// 1. Box it (simplest; sorting is now guaranteed n log n)
Integer[] boxed = Arrays.stream(a).boxed().toArray(Integer[]::new);
Arrays.sort(boxed);

// 2. Shuffle first — destroys any adversarial pattern
Random rnd = new Random();
for (int i = a.length - 1; i > 0; i--) {
    int j = rnd.nextInt(i + 1);
    int t = a[i]; a[i] = a[j]; a[j] = t;
}
Arrays.sort(a);

// 3. Counting sort when the value range is small
```

Shuffling is the competitive-programming answer: O(n) extra work, no boxing.

**`Arrays.sort(int[], Comparator)` does not exist.** Primitives can't take a comparator — that's the main reason to use `Integer[]`.

---

## 2D Arrays as Data

`int[][]` is Java's natural interval/edge type, and LeetCode passes intervals exactly this way.

```java
// Sort intervals by start
Arrays.sort(intervals, (x, y) -> Integer.compare(x[0], y[0]));

// By end
Arrays.sort(intervals, (x, y) -> Integer.compare(x[1], y[1]));

// By start, tie-break by end descending
Arrays.sort(intervals, (x, y) -> x[0] != y[0] ? Integer.compare(x[0], y[0])
                                              : Integer.compare(y[1], x[1]));
```

**Never `(x, y) -> x[0] - y[0]`** — the subtraction overflows when values span the `int` range. `Integer.compare` is the same length and always correct.

Note that `Arrays.sort(int[][], cmp)` *does* work: `int[][]` is an array of objects (each row is an `int[]`), so TimSort applies.

Building results:

```java
List<int[]> out = new ArrayList<>();
out.add(new int[]{start, end});
return out.toArray(new int[0][]);                    // List<int[]> -> int[][]
```

`new int[0][]` is the idiomatic size hint — the JVM allocates the right size internally.

---

## Primitives vs Boxed Types

| | Primitive | Boxed |
|---|---|---|
| Type | `int`, `char`, `long`, `double`, `boolean` | `Integer`, `Character`, `Long`, `Double`, `Boolean` |
| Null? | no | **yes** |
| `==` | value comparison | **reference comparison** |
| In collections | not allowed | required |
| Memory | 4 bytes | ~16 bytes + reference |

### The `Integer` Cache

Java caches `Integer` objects for `-128..127`. So:

```java
Integer a = 100, b = 100;
a == b;              // true  — same cached object

Integer c = 1000, d = 1000;
c == d;              // FALSE — different objects
c.equals(d);         // true

int e = 1000;
c == e;              // true — comparing to an int unboxes c
```

**Always use `.equals()` for boxed types.** The bug is invisible on small test cases and appears on large ones — the worst possible failure mode.

### Autoboxing Costs

```java
List<Integer> list = new ArrayList<>();
for (int i = 0; i < 1_000_000; i++) list.add(i);      // a million object allocations

int[] a = new int[1_000_000];                          // one 4 MB allocation
```

Rule: **use `int[]` for internal computation, convert to `List<Integer>` only for the return value.**

```java
// int[] -> List<Integer>
List<Integer> list = Arrays.stream(a).boxed().collect(Collectors.toList());
List<Integer> list = new ArrayList<>();
for (int x : a) list.add(x);                            // no stream overhead

// List<Integer> -> int[]
int[] a = list.stream().mapToInt(Integer::intValue).toArray();
int[] a = new int[list.size()];
for (int i = 0; i < a.length; i++) a[i] = list.get(i);
```

`list.toArray(new int[0])` **does not compile** — `toArray` only produces object arrays. `list.toArray(new Integer[0])` works and gives `Integer[]`.

### `Arrays.asList` Is a Fixed-Size View

```java
List<Integer> l = Arrays.asList(1, 2, 3);
l.set(0, 9);      // OK — writes through to the backing array
l.add(4);         // throws UnsupportedOperationException
List<Integer> ok = new ArrayList<>(Arrays.asList(1, 2, 3));   // real, mutable list
```

`List.of(1, 2, 3)` (Java 9+) is fully immutable — even `set` throws. Both are fine as *inputs*; neither can be your accumulator.

And `Arrays.asList(intArray)` produces a `List<int[]>` **with one element**, not a list of ints. Boxing is not automatic for arrays.

---

## Iteration

```java
for (int x : a) { }                            // read-only; assigning to x does nothing
for (int i = 0; i < a.length; i++) a[i] *= 2;  // to modify, you need the index

for (int[] row : g) for (int x : row) { }      // 2D read
for (int i = 0; i < g.length; i++)
    for (int j = 0; j < g[0].length; j++) { }  // 2D with indices
```

The enhanced for loop gives you a **copy** of each primitive. `for (int x : a) x = 0;` compiles and does nothing.

---

## Multi-Dimensional DP Setup

```java
int[][] dp = new int[n + 1][m + 1];               // zeros, no fill needed
long[][] dp = new long[n][m];

// Memo with a -1 sentinel
int[][] memo = new int[n][m];
for (int[] row : memo) Arrays.fill(row, -1);

// "Infinity" that survives one addition
int INF = Integer.MAX_VALUE / 2;                  // NOT MAX_VALUE — INF + x overflows
int[][] dist = new int[n][n];
for (int[] row : dist) Arrays.fill(row, INF);
```

`Integer.MAX_VALUE / 2` (~1.07e9) is the Java idiom for a safe infinity: large enough to lose every `min`, small enough that `INF + INF` doesn't wrap. For `long`, use `Long.MAX_VALUE / 2`.

---

## Quick Reference

| Task | Java |
|---|---|
| Length | `a.length` (array), `s.length()` (String), `c.size()` (collection) |
| Fill | `Arrays.fill(a, v)`; 2D: `for (int[] r : g) Arrays.fill(r, v)` |
| Sort ascending | `Arrays.sort(a)` |
| Sort descending | box it: `Arrays.sort(boxed, Collections.reverseOrder())` |
| Copy | `a.clone()`, `Arrays.copyOfRange(a, l, r)` |
| Deep copy 2D | loop + `row.clone()` |
| Reverse | manual two-pointer swap — there is no `Arrays.reverse` |
| Sum | `Arrays.stream(a).sum()` (returns `int`) or `.asLongStream().sum()` |
| Max / min | `Arrays.stream(a).max().getAsInt()` |
| Print | `Arrays.toString(a)` / `Arrays.deepToString(g)` |
| Contains | `Arrays.stream(a).anyMatch(x -> x == v)` or a `HashSet` |
| Binary search | `Arrays.binarySearch(a, k)` — negative means not found |

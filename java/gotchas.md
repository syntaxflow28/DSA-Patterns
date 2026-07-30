# Gotchas — The Bugs That Cost You the Submission

Java fails quietly. No segfaults, no undefined behaviour — just a wrong number, a `null`, or a `ConcurrentModificationException` far from the cause. Ordered by how often these actually happen.

---

## 1. `==` on Boxed Types

```java
Integer a = 1000, b = 1000;
a == b;               // FALSE — different objects

Integer c = 100, d = 100;
c == d;               // true — the -128..127 cache makes small values collide with correctness
```

Java caches `Integer` objects for **−128 … 127**. Below that boundary `==` appears to work; above it, it doesn't. Small test cases pass, the real one fails.

```java
a.equals(b);              // correct
a.intValue() == b.intValue();
int x = a, y = b; x == y; // unboxing makes == a value comparison
```

Same for `Character`, `Long`, `Boolean`, and `String`. **Always `.equals()` for objects.**

---

## 2. Unboxing `null`

```java
Map<String,Integer> map = new HashMap<>();
int c = map.get("missing");            // NullPointerException — unboxing null
int c = map.getOrDefault("missing", 0);// fix

Integer x = list.get(i);
if (x > 5) { }                          // NPE if x is null

int v = tm.floorKey(k);                 // NPE if no key qualifies — TreeMap returns null
```

Every `Integer`-typed value that can be `null` throws the moment it touches arithmetic or a comparison operator. `getOrDefault`, `merge`, and explicit null checks are the fixes.

---

## 3. `list.remove(int)` vs `list.remove(Object)`

```java
List<Integer> l = new ArrayList<>(List.of(10, 20, 30));
l.remove(1);                       // removes INDEX 1  -> [10, 30]
l.remove(Integer.valueOf(10));     // removes VALUE 10 -> [20, 30]
```

Java resolves `remove(1)` to the `int` overload because exact-match beats boxing. Wrap the value when you mean "remove this element".

---

## 4. `Arrays.sort(int[])` Is O(n²) on Adversarial Input

Primitive sort uses dual-pivot quicksort with no worst-case guarantee. LeetCode has anti-quicksort tests, and they TLE otherwise-correct solutions.

```java
// Fix 1 — shuffle first (O(n), no boxing)
Random rnd = new Random();
for (int i = a.length - 1; i > 0; i--) {
    int j = rnd.nextInt(i + 1);
    int t = a[i]; a[i] = a[j]; a[j] = t;
}
Arrays.sort(a);

// Fix 2 — box it; Object[] sorting is TimSort, O(n log n) guaranteed
Integer[] boxed = Arrays.stream(a).boxed().toArray(Integer[]::new);
Arrays.sort(boxed);
```

`Arrays.sort(Object[])`, `Arrays.sort(int[][], cmp)`, and `Collections.sort` are all TimSort and safe.

---

## 5. Integer Overflow

```java
long s = a + b;                    // int + int computed as int, THEN widened
long s = (long) a + b;             // fix
long p = 1L * a * b;               // fix for products

int mid = (lo + hi) / 2;           // overflows when lo + hi > 2.1e9
int mid = (lo + hi) >>> 1;         // fix
int mid = lo + (hi - lo) / 2;      // fix

int sum = Arrays.stream(a).sum();  // IntStream.sum returns int
long sum = Arrays.stream(a).asLongStream().sum();   // fix
```

Java wraps silently — no exception, no warning. **The expression's type comes from its operands, not from the variable you assign to.**

---

## 6. Comparator Subtraction

```java
(a, b) -> a - b                    // overflows when values straddle the int range
Integer.compare(a, b)              // fix
```

An inconsistent comparator makes TimSort throw:

```
java.lang.IllegalArgumentException: Comparison method violates its general contract!
```

That message is almost always a subtraction overflow or a non-transitive comparison. Also check `.reversed()` placement — it flips the *entire* chain built so far, not just the last key.

---

## 7. Storing a Live Reference Instead of a Copy

```java
res.add(path);                      // stores a REFERENCE; path keeps mutating
res.add(new ArrayList<>(path));     // fix — snapshot it
```

After backtracking pops everything, every row in `res` is the same empty list. This is *the* Java backtracking bug.

Same idea for arrays:

```java
res.add(arr);                       // aliased
res.add(arr.clone());               // fix
```

---

## 8. Shallow 2D Copies and Fills

```java
int[][] copy = grid.clone();                       // copies ROW REFERENCES only
for (int i = 0; i < grid.length; i++)              // fix
    copy[i] = grid[i].clone();

Arrays.fill(grid, -1);                             // fills with the value -1 as a ROW ref — won't compile for int[][]
for (int[] row : grid) Arrays.fill(row, -1);       // fix
```

The memo-initialization case (`Arrays.fill(memo, -1)`) is where this bites most often.

---

## 9. `ConcurrentModificationException`

```java
for (Integer x : list) if (x == 0) list.remove(x);      // throws
list.removeIf(x -> x == 0);                             // fix

for (String k : map.keySet()) map.remove(k);            // throws
map.keySet().removeIf(k -> cond(k));                    // fix
map.entrySet().removeIf(e -> e.getValue() == 0);        // fix

Iterator<Integer> it = list.iterator();                 // fix — explicit iterator
while (it.hasNext()) if (it.next() == 0) it.remove();
```

`entry.setValue()` during iteration is allowed; structural modification is not.

---

## 10. `Arrays.asList` Is Fixed-Size

```java
List<Integer> l = Arrays.asList(1, 2, 3);
l.add(4);                                       // UnsupportedOperationException
l.set(0, 9);                                    // OK — writes through to the backing array

List<Integer> ok = new ArrayList<>(Arrays.asList(1, 2, 3));   // fix
```

`List.of(...)` is fully immutable — even `set` throws. And `.toList()` at the end of a stream (Java 16+) returns an immutable list, unlike `.collect(Collectors.toList())`.

```java
Arrays.asList(intArray);        // gives List<int[]> with ONE element, not a list of ints
```

---

## 11. `char` Arithmetic Promotes to `int`

```java
char c = someChar + 1;            // COMPILE ERROR — int can't narrow implicitly
char c = (char) (someChar + 1);   // fix

sb.append(someChar + 1);          // appends "98", not "b"
sb.append((char) (someChar + 1)); // fix

System.out.println('a' + 'b');    // 195, not "ab"
System.out.println("" + 'a' + 'b'); // "ab"
```

The `append` case compiles, runs, and silently corrupts your output.

---

## 12. `length` vs `length()` vs `size()`

```java
array.length          // FIELD
string.length()       // METHOD
collection.size()     // METHOD
```

Compile errors, so cheap — but they break flow. `grid.length` is the row count, `grid[0].length` is the column count.

---

## 13. `Math.abs(Integer.MIN_VALUE)`

```java
Math.abs(Integer.MIN_VALUE) == Integer.MIN_VALUE      // still negative
```

Two's complement has no positive counterpart for `MIN_VALUE`. Use `Math.abs((long) x)` when the input range allows the extreme.

---

## 14. `-7 % 3 == -1`

```java
int prev = (i - 1) % n;                // negative when i == 0
int prev = Math.floorMod(i - 1, n);    // fix
int prev = ((i - 1) % n + n) % n;      // fix, the manual way
```

The `%` result takes the sign of the dividend. Circular arrays and modular arithmetic hit this constantly.

---

## 15. `Math.pow` Precision

```java
int x = (int) Math.pow(10, 2);         // can be 99
int r = (int) Math.sqrt(n);            // can be off by one
```

`Math.pow` returns a `double`. Write an integer power loop, and adjust `sqrt` results with a `while`. See [numbers-bits.md](numbers-bits.md).

---

## 16. `Integer.MAX_VALUE` as Infinity

```java
int[] dist = new int[n];
Arrays.fill(dist, Integer.MAX_VALUE);
dist[u] + w;                                    // overflows to negative, and the relax passes

Arrays.fill(dist, Integer.MAX_VALUE / 2);       // fix — survives one addition
```

Classic in Dijkstra, Bellman-Ford, and Floyd-Warshall. Use `Long.MAX_VALUE / 2` for `long` distances.

---

## 17. `PriorityQueue` Iteration Isn't Sorted

```java
for (int x : pq) { }                    // internal array order — NOT sorted
while (!pq.isEmpty()) pq.poll();        // the only ordered traversal

pq.toString();                          // heap order; misleading when debugging
pq.remove(Object);                      // O(n)
pq.contains(x);                         // O(n)
```

Only `peek`/`poll` respect the ordering. If you need arbitrary removal with ordering, use a `TreeMap<Integer,Integer>` as a multiset.

---

## 18. `ArrayDeque` Rejects `null`

```java
Deque<TreeNode> st = new ArrayDeque<>();
st.push(node.left);                     // NullPointerException if left is null
if (node.left != null) st.push(node.left);   // fix
```

This is a feature — it's what makes `poll() == null` a reliable empty signal — but it surprises people converting from `LinkedList`.

---

## 19. Mixing `Deque` Vocabularies

```java
dq.push(x);      // adds to the FRONT
dq.poll();       // removes from the FRONT  -> this is a STACK, not a queue
```

For a **stack**: `push` / `pop` / `peek`. For a **queue**: `offer` / `poll` / `peek`. Mixing them silently reverses your traversal order — a BFS that behaves like a DFS.

---

## 20. `String` Immutability and `+=` in a Loop

```java
String s = "";
for (int i = 0; i < n; i++) s += i;       // O(n²) — a new String every iteration

StringBuilder sb = new StringBuilder();
for (int i = 0; i < n; i++) sb.append(i); // fix
```

And `substring` copies in O(n) since Java 7 — a `substring` inside a nested loop is O(n³).

---

## 21. `split` Is a Regex, and Drops Trailing Empties

```java
s.split(".");             // regex "any char" — returns an EMPTY array
s.split("\\.");           // fix

"a,b,,".split(",");       // ["a", "b"] — trailing empties dropped
"a,b,,".split(",", -1);   // ["a", "b", "", ""] — keeps them
```

---

## 22. `int[]` Can't Be a `HashMap` Key

```java
Set<int[]> seen = new HashSet<>();
seen.add(new int[]{1, 2});
seen.contains(new int[]{1, 2});     // FALSE — arrays use identity equals/hashCode
```

Fixes: a `record Cell(int r, int c) {}`, `List.of(r, c)`, a `String` key, or encode into a `long`:

```java
long key = (long) r * COLS + c;      // fastest; cast to long BEFORE multiplying
```

---

## 23. Effectively Final in Lambdas

```java
int count = 0;
list.forEach(x -> count++);          // COMPILE ERROR

int[] count = {0};
list.forEach(x -> count[0]++);       // fix — mutate the object, don't rebind the variable
```

Instance fields have no such restriction, which is why Java tree solutions use `private int best;`.

---

## 24. Stale `Solution` Instance State

LeetCode may reuse your `Solution` object across test cases. A field left over from a previous run causes a wrong answer that **only reproduces in the full submission**, never in a single custom test.

```java
class Solution {
    private int best;                    // dangerous across calls
    public int solve(int[] nums) {
        best = Integer.MIN_VALUE;        // fix — reset at the top of every entry point
        ...
    }
}
```

`static` fields are even worse; avoid them for mutable state.

---

## 25. `StackOverflowError`

Java's default stack holds roughly 10⁴ frames. Deep recursion — a skewed tree, a long linked list, a path-graph DFS — throws `StackOverflowError`, which LeetCode reports as a runtime error rather than a clear message.

Fix: convert to an iterative traversal with an explicit `Deque`. Java has no tail-call optimization, so restructuring the recursion won't help.

---

## 26. Boxing in Hot Loops

```java
List<Integer> list = new ArrayList<>();
for (int i = 0; i < 1_000_000; i++) list.add(i);      // a million allocations

Map<Integer,Integer> map = new HashMap<>();
for (int i = 0; i < n; i++) map.put(i, i);            // boxing on every key and value
```

Use `int[]` for internal computation; convert to `List<Integer>` only for the return value. When the key space is a dense `0..n-1`, an `int[]` beats a `HashMap` outright.

---

## Pre-Submit Checklist

- [ ] `.equals()` for every object comparison; `==` only for nodes and primitives.
- [ ] `getOrDefault` / null checks wherever a boxed value could be `null`.
- [ ] Any sum or product past ~2e9 is computed in `long`, cast before the operation.
- [ ] `(lo + hi) >>> 1` for midpoints.
- [ ] `Integer.compare` in every comparator; no subtraction.
- [ ] Snapshots (`new ArrayList<>(path)`, `arr.clone()`) wherever a result is recorded.
- [ ] 2D copies and `Arrays.fill` done row by row.
- [ ] `removeIf` or an explicit iterator for removal during traversal.
- [ ] Large `int[]` inputs shuffled before `Arrays.sort`.
- [ ] Instance fields reset at the top of the entry method.
- [ ] `StringBuilder` for any string built in a loop.
- [ ] Empty input, single element, and all-equal elements handled.

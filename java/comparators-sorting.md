# Sorting & Comparators

Java's comparator story is better than most people use. `Comparator.comparingInt(...).thenComparing(...)` expresses in one line what a hand-written comparator takes six lines and one subtle bug to express.

---

## Two Sorting Entry Points

```java
Arrays.sort(a);                        // int[], char[], ... — dual-pivot quicksort, NOT stable
Arrays.sort(objArray);                 // Object[] — TimSort, stable, O(n log n) guaranteed
Arrays.sort(objArray, cmp);
Arrays.sort(a, from, to);              // subrange

Collections.sort(list);                // TimSort
Collections.sort(list, cmp);
list.sort(cmp);                        // the modern form
list.sort(null);                       // natural ordering
```

**There is no `Arrays.sort(int[], Comparator)`.** Primitives can't take one. To sort primitives by a custom rule you must box:

```java
Integer[] boxed = Arrays.stream(a).boxed().toArray(Integer[]::new);
Arrays.sort(boxed, (x, y) -> y - x);              // descending
```

And remember `Arrays.sort(int[])` is O(n²) on adversarial input — details and fixes in [arrays.md](arrays.md).

---

## The Comparator Contract

`compare(a, b)` returns:
- **negative** if `a` comes first
- **zero** if they tie
- **positive** if `b` comes first

```java
(a, b) -> a - b        // ascending  — OVERFLOWS
(a, b) -> b - a        // descending — OVERFLOWS
Integer.compare(a, b)  // ascending, always correct
Integer.compare(b, a)  // descending, always correct
```

### Never Subtract

`a - b` overflows when the values straddle the `int` range. `Integer.MIN_VALUE - 1` wraps to a positive number and the comparator reports the wrong order — or, worse, becomes inconsistent and throws:

```
java.lang.IllegalArgumentException: Comparison method violates its general contract!
```

TimSort detects contract violations at runtime and throws. That exception is almost always a subtraction overflow or a comparator that isn't transitive.

Use `Integer.compare`, `Long.compare`, `Double.compare`. Same length, no risk.

---

## Building Comparators Declaratively

```java
import java.util.Comparator;

Comparator.comparingInt(Person::getAge)
Comparator.comparingLong(x -> x.total)
Comparator.comparingDouble(x -> x.score)
Comparator.comparing(Person::getName)                 // any Comparable
Comparator.comparing(Person::getName, cmp)            // with a custom key comparator

.thenComparingInt(Person::getId)                      // tie-break
.thenComparing(Person::getName)
.reversed()                                            // flip the WHOLE chain
Comparator.reverseOrder()
Comparator.naturalOrder()
Comparator.nullsFirst(cmp)  /  nullsLast(cmp)
```

```java
// Age ascending, then name ascending
list.sort(Comparator.comparingInt(Person::getAge).thenComparing(Person::getName));

// Age DESCENDING, then name ascending
list.sort(Comparator.comparingInt(Person::getAge).reversed().thenComparing(Person::getName));
```

**`.reversed()` reverses everything before it in the chain.** `comparingInt(A).thenComparing(B).reversed()` reverses *both* A and B. To reverse only the first key, reverse it inside:

```java
Comparator.comparingInt((Person p) -> -p.getAge()).thenComparing(Person::getName);
// or
Comparator.comparing(Person::getAge, Comparator.reverseOrder()).thenComparing(Person::getName);
```

**Type inference needs help** when the first argument is a lambda rather than a method reference — annotate the parameter:

```java
list.sort(Comparator.comparingInt(p -> p.getAge()));                 // may not compile in a chain
list.sort(Comparator.comparingInt((Person p) -> p.getAge()));        // explicit type, always works
```

---

## Sorting `int[][]` — LeetCode's Interval Type

```java
Arrays.sort(intervals, (a, b) -> Integer.compare(a[0], b[0]));                 // by start
Arrays.sort(intervals, Comparator.comparingInt(a -> a[0]));                    // same, shorter
Arrays.sort(intervals, (a, b) -> Integer.compare(a[1], b[1]));                 // by end

// By start, then by end descending
Arrays.sort(intervals, (a, b) -> a[0] != b[0] ? Integer.compare(a[0], b[0])
                                              : Integer.compare(b[1], a[1]));

// Declarative version
Arrays.sort(intervals, Comparator.<int[]>comparingInt(a -> a[0])
                                 .thenComparing(a -> a[1], Comparator.reverseOrder()));
```

The `Comparator.<int[]>comparingInt` explicit type witness is needed because the compiler can't infer `int[]` from a bare lambda in a chain.

Sort by *end* is the greedy activity-selection order (LC 435, 452); sort by *start* is the merge order (LC 56, 57).

---

## `Comparable` — Natural Ordering on Your Own Type

```java
class Task implements Comparable<Task> {
    int deadline, profit;
    @Override public int compareTo(Task o) {
        if (deadline != o.deadline) return Integer.compare(deadline, o.deadline);
        return Integer.compare(o.profit, profit);        // profit descending
    }
}
Arrays.sort(tasks);
Collections.sort(list);
PriorityQueue<Task> pq = new PriorityQueue<>();          // uses compareTo
```

For records, implementing `Comparable` is often unnecessary — pass a comparator instead and keep the type ordering-agnostic:

```java
record Task(int deadline, int profit) {}
list.sort(Comparator.comparingInt(Task::deadline).thenComparing(Task::profit, Comparator.reverseOrder()));
```

**`compareTo` should be consistent with `equals`** if the type goes into a `TreeMap`/`TreeSet` — those containers use *comparison*, not `equals`, to decide identity. If `compare(a, b) == 0`, the set treats them as the same element and silently drops one.

---

## `PriorityQueue` Comparators

Unlike C++, Java's heap comparator is **not** inverted. The comparator you'd pass to `sort` for ascending order gives you a min-heap.

```java
new PriorityQueue<>();                                        // min-heap (natural order)
new PriorityQueue<>(Comparator.reverseOrder());               // max-heap
new PriorityQueue<>(Comparator.comparingInt(a -> a[0]));      // min by column 0
new PriorityQueue<>((a, b) -> Integer.compare(b[1], a[1]));   // max by column 1

// Max-heap by frequency, tie-break by smaller value
new PriorityQueue<int[]>((a, b) -> a[1] != b[1] ? b[1] - a[1] : a[0] - b[0]);
```

The `b[1] - a[1]` form is fine when values are known-small (frequencies, counts). For anything that could span the `int` range, use `Integer.compare`.

---

## Sorting Recipes

```java
// Descending list of Integer
list.sort(Comparator.reverseOrder());
Collections.sort(list, Collections.reverseOrder());

// Strings by length, then lexicographically
list.sort(Comparator.comparingInt(String::length).thenComparing(Comparator.naturalOrder()));

// Largest Number (LC 179) — custom concatenation order
String[] a = ...;
Arrays.sort(a, (x, y) -> (y + x).compareTo(x + y));

// Sort a map's entries by value descending
List<Map.Entry<String,Integer>> es = new ArrayList<>(map.entrySet());
es.sort(Map.Entry.<String,Integer>comparingByValue().reversed());
es.sort(Map.Entry.comparingByKey());

// Sort indices, leaving the data untouched
Integer[] idx = new Integer[n];
for (int i = 0; i < n; i++) idx[i] = i;
Arrays.sort(idx, Comparator.comparingInt(i -> nums[i]));

// Points by distance from origin
Arrays.sort(pts, Comparator.comparingLong(p -> (long) p[0]*p[0] + (long) p[1]*p[1]));

// Sort chars in a string
char[] c = s.toCharArray(); Arrays.sort(c); String sorted = new String(c);

// Stable multi-key sort by sorting twice (least significant key first)
list.sort(Comparator.comparing(X::secondary));
list.sort(Comparator.comparing(X::primary));       // TimSort is stable, so ties keep the prior order
```

That last one only works because `Collections.sort`/`Arrays.sort(Object[])` are stable. `Arrays.sort(int[])` is not.

---

## Sorting Around a Custom Order

When elements must follow an externally defined ranking (LC 791 Custom Sort String, LC 937 Reorder Log Files):

```java
int[] rank = new int[26];
for (int i = 0; i < order.length(); i++) rank[order.charAt(i) - 'a'] = i;

Character[] c = ...;
Arrays.sort(c, Comparator.comparingInt(ch -> rank[ch - 'a']));
```

Precompute the rank into an array, then sort by the rank — never call `order.indexOf(ch)` inside the comparator, which turns O(n log n) into O(n log n · m).

---

## Binary Search on Sorted Data

```java
Arrays.binarySearch(a, key);                       // negative -> -(insertionPoint) - 1
Collections.binarySearch(list, key);
Collections.binarySearch(list, key, cmp);

int idx = Arrays.binarySearch(a, key);
int insertAt = idx >= 0 ? idx : -idx - 1;
```

For "first index ≥ x" write the loop yourself (see [arrays.md](arrays.md)) — the JDK gives no lower/upper bound for arrays. For objects, `TreeSet.ceiling` / `TreeMap.floorKey` are the ergonomic answer.

Binary search on the answer:

```java
int lo = 1, hi = maxPossible, ans = hi;
while (lo <= hi) {
    int mid = lo + (hi - lo) / 2;            // or (lo + hi) >>> 1
    if (feasible(mid)) { ans = mid; hi = mid - 1; }
    else lo = mid + 1;
}
```

---

## Comparator Checklist

- [ ] `Integer.compare`, never `a - b`.
- [ ] `.reversed()` placement — it flips the whole chain built so far.
- [ ] Explicit lambda parameter types when the compiler complains.
- [ ] Comparator is transitive and consistent, or TimSort throws `IllegalArgumentException`.
- [ ] `Arrays.sort(int[])` risk acknowledged — shuffle or box for large inputs.
- [ ] `TreeMap`/`TreeSet` comparator returning 0 means "same key" — one element gets dropped.
- [ ] No expensive work (`indexOf`, `substring`, map lookups) inside the comparator body.

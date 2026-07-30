# Collections — `List`, `Set`, `Map`, `Deque`, `PriorityQueue`

Java's collections are interfaces with multiple implementations. **Declare the interface, instantiate the implementation** — and the implementation choice is where the performance lives.

```java
List<Integer> list = new ArrayList<>();
Map<String,Integer> map = new HashMap<>();
Deque<Integer> stack = new ArrayDeque<>();
```

---

## `List`

### `ArrayList` — the default

```java
List<Integer> list = new ArrayList<>();
List<Integer> list = new ArrayList<>(capacity);        // pre-size — avoids regrowth
List<Integer> list = new ArrayList<>(otherCollection); // copy

list.add(x);                    // append, O(1) amortized
list.add(i, x);                 // insert at index, O(n)
list.get(i);                    // O(1)
list.set(i, x);                 // O(1)
list.remove(i);                 // BY INDEX — O(n)
list.remove(Integer.valueOf(x));// BY VALUE
list.size();  list.isEmpty();  list.clear();
list.contains(x);               // O(n)
list.indexOf(x);                // O(n), -1 if absent
list.subList(from, to);         // a VIEW, not a copy — mutations write through
list.addAll(other);
list.sort(comparator);
Collections.sort(list);
Collections.reverse(list);
```

### `list.remove(5)` — The Overload Trap

For a `List<Integer>`, `remove(int)` and `remove(Object)` both apply:

```java
List<Integer> l = new ArrayList<>(List.of(10, 20, 30));
l.remove(1);                     // removes INDEX 1 -> [10, 30]
l.remove(Integer.valueOf(10));   // removes the VALUE 10 -> [20, 30]
```

Java picks `remove(int)` for an `int` literal because exact-match beats boxing. `Integer.valueOf(x)` is the disambiguator. This bug produces plausible-looking wrong answers.

### `LinkedList` — Almost Never What You Want

It implements both `List` and `Deque`, which makes it look versatile. But `get(i)` walks the list — O(n). A loop of `list.get(i)` over a `LinkedList` is O(n²).

Use `ArrayList` for indexing, `ArrayDeque` for ends. `LinkedList` is only right when you need a `List` *and* O(1) end operations simultaneously, which basically never happens on LeetCode.

### Building the Return Value

LeetCode signatures usually want `List<List<Integer>>`:

```java
List<List<Integer>> res = new ArrayList<>();
res.add(new ArrayList<>(path));            // COPY — `path` keeps mutating in backtracking
res.add(List.of(1, 2));                    // immutable, fine for a finished row
res.add(Arrays.asList(1, 2));
```

**`res.add(path)` without the copy** stores a reference. After backtracking pops everything, every row in `res` is the same empty list. This is the most common backtracking bug in Java.

### `Collections` Utilities

```java
Collections.sort(list);
Collections.sort(list, cmp);
Collections.reverse(list);
Collections.shuffle(list);
Collections.max(list);  Collections.min(list);
Collections.frequency(list, x);
Collections.swap(list, i, j);
Collections.nCopies(n, x);
Collections.emptyList();
Collections.reverseOrder();               // a descending Comparator
Collections.unmodifiableList(list);
```

---

## `Deque` — Stack, Queue, and Deque in One

**`ArrayDeque` is the answer for all three.** `Stack` is a legacy synchronized class that iterates bottom-to-top (surprising), and `LinkedList` allocates a node per element.

```java
Deque<Integer> dq = new ArrayDeque<>();

// As a stack (LIFO) — always use the *First methods
dq.push(x);        // == addFirst
dq.pop();          // == removeFirst — throws if empty
dq.peek();         // == peekFirst — null if empty

// As a queue (FIFO)
dq.offer(x);       // == addLast
dq.poll();         // == removeFirst — null if empty
dq.peek();         // front

// As a deque
dq.addFirst(x);  dq.addLast(x);
dq.pollFirst();  dq.pollLast();
dq.peekFirst();  dq.peekLast();
dq.size();  dq.isEmpty();
```

**Pick one vocabulary and stay in it.** Mixing `push` (adds to the front) with `poll` (removes from the front) silently gives you a stack when you wanted a queue. For a stack use `push`/`pop`/`peek`; for a queue use `offer`/`poll`/`peek`.

Two families for the same operation, differing only in failure mode:

| Operation | Throws on failure | Returns special value |
|---|---|---|
| Insert | `add`, `addFirst`, `addLast` | `offer`, `offerFirst`, `offerLast` |
| Remove | `remove`, `removeFirst` | `poll`, `pollFirst` (null) |
| Examine | `element`, `getFirst` | `peek`, `peekFirst` (null) |

Prefer the `offer`/`poll`/`peek` family — a `null` you can test beats an exception.

**`ArrayDeque` does not allow `null` elements.** That's what makes `poll() == null` a reliable "empty" signal.

### Canonical Uses

```java
// Iterative DFS
Deque<Integer> st = new ArrayDeque<>();
st.push(start);
while (!st.isEmpty()) {
    int u = st.pop();
    if (vis[u]) continue;
    vis[u] = true;
    for (int v : adj[u]) if (!vis[v]) st.push(v);
}

// BFS with levels
Deque<Integer> q = new ArrayDeque<>();
q.offer(start);
int level = 0;
while (!q.isEmpty()) {
    int sz = q.size();                   // SNAPSHOT before the inner loop
    for (int i = 0; i < sz; i++) {
        int u = q.poll();
        for (int v : adj[u]) if (!vis[v]) { vis[v] = true; q.offer(v); }
    }
    level++;
}

// Monotonic deque — sliding window maximum (LC 239)
Deque<Integer> dq = new ArrayDeque<>();          // holds INDICES, values decreasing
for (int i = 0; i < n; i++) {
    while (!dq.isEmpty() && dq.peekFirst() <= i - k) dq.pollFirst();
    while (!dq.isEmpty() && nums[dq.peekLast()] <= nums[i]) dq.pollLast();
    dq.offerLast(i);
    if (i >= k - 1) res[i - k + 1] = nums[dq.peekFirst()];
}
```

Monotonic structures store **indices**, not values — you need the position to know when an element leaves the window.

---

## `Map`

### `HashMap` — the default

```java
Map<String,Integer> map = new HashMap<>();

map.put(k, v);                              // returns the OLD value, or null
map.get(k);                                 // null if absent
map.getOrDefault(k, 0);                     // the one you want 90% of the time
map.putIfAbsent(k, v);
map.containsKey(k);  map.containsValue(v);  // containsValue is O(n)
map.remove(k);                              // returns the removed value
map.size();  map.isEmpty();  map.clear();
map.keySet();  map.values();  map.entrySet();
```

### The Four Methods That Shorten Everything

```java
map.merge(k, 1, Integer::sum);              // frequency count in one line
map.merge(k, v, (a, b) -> Math.max(a, b));  // keep the max

map.computeIfAbsent(k, x -> new ArrayList<>()).add(v);      // multimap / adjacency list
map.computeIfPresent(k, (key, val) -> val - 1);
map.compute(k, (key, val) -> val == null ? 1 : val + 1);
```

`merge` and `computeIfAbsent` replace the null-check-then-put pattern entirely:

```java
// Before
if (!map.containsKey(k)) map.put(k, new ArrayList<>());
map.get(k).add(v);
// After
map.computeIfAbsent(k, x -> new ArrayList<>()).add(v);
```

### Null Is Real

```java
int c = map.get(k);                    // NullPointerException if k is absent — unboxing null
int c = map.getOrDefault(k, 0);        // safe
Integer c = map.get(k);                // safe, but you must null-check before using it
```

This is Java's counterpart to C++'s `map[key]`-inserts problem, and it's just as common. Unboxing a `null` throws at the point of use, often far from the cause.

### Iteration

```java
for (Map.Entry<String,Integer> e : map.entrySet()) {
    String k = e.getKey();
    int v = e.getValue();
    e.setValue(v + 1);                       // the only safe way to modify during iteration
}
for (String k : map.keySet()) { }
for (int v : map.values()) { }
map.forEach((k, v) -> { });
```

`entrySet()` is one lookup per entry; `for (K k : keySet()) map.get(k)` is two. Prefer `entrySet`.

**Modifying the map itself (put/remove) during iteration throws `ConcurrentModificationException`** — except `entry.setValue()` and `iterator.remove()`.

```java
Iterator<Map.Entry<K,V>> it = map.entrySet().iterator();
while (it.hasNext()) {
    Map.Entry<K,V> e = it.next();
    if (e.getValue() == 0) it.remove();       // safe removal
}
map.entrySet().removeIf(e -> e.getValue() == 0);      // shorter (Java 8+)
```

### `TreeMap` — Sorted, With Navigation

This is Java's answer to "I need the closest key". C++ has only `lower_bound`; Java gives you four directions and they read far better.

```java
TreeMap<Integer,Integer> tm = new TreeMap<>();
TreeMap<Integer,Integer> desc = new TreeMap<>(Collections.reverseOrder());

tm.firstKey();  tm.lastKey();
tm.firstEntry();  tm.lastEntry();
tm.pollFirstEntry();  tm.pollLastEntry();     // retrieve and remove

tm.floorKey(x);      // greatest key <= x
tm.ceilingKey(x);    // smallest key >= x
tm.lowerKey(x);      // greatest key <  x
tm.higherKey(x);     // smallest key >  x
// ...and floorEntry / ceilingEntry / lowerEntry / higherEntry

tm.headMap(x);       // keys < x   (a live VIEW)
tm.tailMap(x);       // keys >= x
tm.subMap(a, b);     // [a, b)
tm.descendingMap();
```

All navigation methods return **`null`** when nothing qualifies. Null-check before unboxing.

```java
// Sweep line / difference over sparse coordinates (LC 253, 731)
TreeMap<Integer,Integer> diff = new TreeMap<>();
diff.merge(start, 1, Integer::sum);
diff.merge(end, -1, Integer::sum);
int cur = 0, best = 0;
for (int d : diff.values()) best = Math.max(best, cur += d);

// TreeMap as a multiset — sorted, with duplicates and O(log n) removal
TreeMap<Integer,Integer> ms = new TreeMap<>();
ms.merge(x, 1, Integer::sum);                            // insert
if (ms.merge(x, -1, Integer::sum) == 0) ms.remove(x);    // remove ONE copy
int min = ms.firstKey(), max = ms.lastKey();
```

That multiset idiom is the workaround for `PriorityQueue`'s O(n) removal — it's how you do sliding-window median or "max with arbitrary deletion" in Java.

### `LinkedHashMap` — Insertion or Access Order

```java
Map<K,V> m = new LinkedHashMap<>();                          // insertion-ordered iteration
Map<K,V> lru = new LinkedHashMap<>(cap, 0.75f, true);        // ACCESS-ordered
```

With `accessOrder = true`, every `get` moves the entry to the end. Override one method and you have an LRU cache (LC 146):

```java
class LRUCache extends LinkedHashMap<Integer,Integer> {
    private final int cap;
    LRUCache(int cap) { super(cap, 0.75f, true); this.cap = cap; }
    @Override protected boolean removeEldestEntry(Map.Entry<Integer,Integer> e) {
        return size() > cap;
    }
    public int get(int k) { return super.getOrDefault(k, -1); }
    public void put(int k, int v) { super.put(k, v); }
}
```

Say out loud that you know the manual `HashMap` + doubly-linked-list version — interviewers usually want that one.

---

## `Set`

```java
Set<Integer> s = new HashSet<>();          // O(1), unordered
Set<Integer> s = new LinkedHashSet<>();    // O(1), insertion-ordered
TreeSet<Integer> s = new TreeSet<>();      // O(log n), sorted + navigation

s.add(x);        // returns false if already present — insert-and-test in one call
s.contains(x);  s.remove(x);  s.size();  s.isEmpty();
s.addAll(o);  s.retainAll(o);  s.removeAll(o);   // union / intersection / difference
```

```java
if (!seen.add(x)) return true;             // "have I seen x before" in one line
```

`TreeSet` carries the same navigation methods as `TreeMap`:

```java
TreeSet<Integer> ts = new TreeSet<>();
ts.first();  ts.last();
ts.floor(x);  ts.ceiling(x);  ts.lower(x);  ts.higher(x);
ts.pollFirst();  ts.pollLast();
ts.headSet(x);  ts.tailSet(x);  ts.subSet(a, b);
ts.descendingSet();
```

`TreeSet` solves LC 220 Contains Duplicate III in a few lines — `ceiling` finds the nearest value within a tolerance.

Set from an array:

```java
Set<Integer> s = new HashSet<>(Arrays.asList(1, 2, 3));
Set<Integer> s = Arrays.stream(a).boxed().collect(Collectors.toSet());
```

---

## `PriorityQueue`

**Min-heap by default** — the opposite of C++'s `priority_queue`.

```java
PriorityQueue<Integer> minHeap = new PriorityQueue<>();
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Comparator.reverseOrder());
PriorityQueue<Integer> maxHeap = new PriorityQueue<>((a, b) -> b - a);   // overflow risk
PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[0] - b[0]);    // overflow risk
PriorityQueue<int[]> pq = new PriorityQueue<>(Comparator.comparingInt(a -> a[0]));  // safe

pq.offer(x);       // O(log n)   — `add` is identical here
pq.poll();         // remove the head, null if empty
pq.peek();         // O(1), null if empty
pq.size();  pq.isEmpty();
pq.remove(Object); // O(n) — a heap can't search
pq.contains(x);    // O(n)
```

**Iterating a `PriorityQueue` does not give sorted order.** `for (int x : pq)` walks the internal array. Only repeated `poll()` is ordered.

Unlike C++, Java's comparator here is **not** inverted — it's the same comparator you'd pass to `sort`. "Ascending comparator → min-heap" is the intuitive mapping.

```java
// Kth largest — a min-heap of size k
PriorityQueue<Integer> pq = new PriorityQueue<>();
for (int x : nums) { pq.offer(x); if (pq.size() > k) pq.poll(); }
return pq.peek();

// Dijkstra — {dist, node}
PriorityQueue<long[]> pq = new PriorityQueue<>((a, b) -> Long.compare(a[0], b[0]));
long[] dist = new long[n];
Arrays.fill(dist, Long.MAX_VALUE / 2);
dist[src] = 0; pq.offer(new long[]{0, src});
while (!pq.isEmpty()) {
    long[] cur = pq.poll();
    long d = cur[0]; int u = (int) cur[1];
    if (d > dist[u]) continue;                       // stale entry — lazy deletion
    for (int[] e : adj[u]) {
        int v = e[0], w = e[1];
        if (d + w < dist[v]) { dist[v] = d + w; pq.offer(new long[]{dist[v], v}); }
    }
}

// Top-k frequent (LC 347)
Map<Integer,Integer> f = new HashMap<>();
for (int x : nums) f.merge(x, 1, Integer::sum);
PriorityQueue<Integer> pq = new PriorityQueue<>(Comparator.comparingInt(f::get));
for (int key : f.keySet()) { pq.offer(key); if (pq.size() > k) pq.poll(); }

// Heapify a collection in O(n)
PriorityQueue<Integer> pq = new PriorityQueue<>(list);
```

**Lazy deletion** (`if (d > dist[u]) continue;`) is the standard workaround for the missing decrease-key, since `pq.remove(Object)` is O(n). Push the improved entry and skip stale ones as they surface.

---

## Records as Map Keys and Heap Elements

Java 16+ gives you `equals`, `hashCode`, and `toString` for free — the clean replacement for `int[]` tuples:

```java
record Cell(int r, int c) {}
record Edge(int to, int weight) {}

Set<Cell> visited = new HashSet<>();
visited.add(new Cell(r, c));                                   // hashing just works

Map<Cell,Integer> dist = new HashMap<>();
PriorityQueue<Edge> pq = new PriorityQueue<>(Comparator.comparingInt(Edge::weight));
```

**`int[]` cannot be a `HashMap` key** — arrays use identity `equals`/`hashCode`, so `new int[]{1,2}` never matches another `new int[]{1,2}`. Options: a `record`, `List.of(r, c)`, a `String` key, or encode into a `long` (`(long) r * COLS + c`). The encoded `long` is fastest; the record is clearest.

---

## Selection Guide

| Need | Use |
|---|---|
| Indexed sequence | `ArrayList` |
| Stack | `ArrayDeque` (`push`/`pop`/`peek`) |
| Queue / BFS | `ArrayDeque` (`offer`/`poll`/`peek`) |
| Sliding-window max | `ArrayDeque` of indices |
| Frequency count | `HashMap` + `merge` |
| Adjacency list | `List<List<Integer>>`, or `Map` + `computeIfAbsent` |
| Membership | `HashSet` |
| Closest key / floor / ceiling | `TreeMap` / `TreeSet` |
| Sweep line | `TreeMap<Integer,Integer>` |
| Sorted multiset with removal | `TreeMap<Integer,Integer>` counting values |
| Top-k, scheduling, Dijkstra | `PriorityQueue` |
| LRU | `LinkedHashMap(cap, 0.75f, true)` |
| Composite key | `record`, or a `long`-encoded key |

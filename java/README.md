# Java for LeetCode — The Complete Reference

Java's problem is not a missing library — it's that the *convenient* thing and the *fast* thing are usually different objects. `Stack` exists but you should use `ArrayDeque`. `LinkedList` implements `List` but indexing it is O(n). `map.get(k)` returns `null`, not 0. Autoboxing makes `==` silently wrong above 127.

This folder is organized around the decisions Java actually forces on you, not around a tour of `java.util`.

---

## The Files

| File | What it covers | Revise when |
|---|---|---|
| [arrays.md](arrays.md) | `int[]`, 2D arrays, the `Arrays` class, primitive vs boxed | Any array problem, DP tables, grids |
| [collections.md](collections.md) | `List`, `Set`, `Map`, `Deque`, `PriorityQueue` — and which impl | Hashing, BFS, heaps, ordered queries |
| [strings.md](strings.md) | `String` immutability, `StringBuilder`, `char[]`, `split` | Parsing, building output, palindromes |
| [comparators-sorting.md](comparators-sorting.md) | `Comparator`, `Comparable`, sorting rules, the `int[]` trap | Every custom sort or heap |
| [numbers-bits.md](numbers-bits.md) | Overflow, `Math`, `Integer`/`Long` statics, bit manipulation | Bitmask DP, math, modular arithmetic |
| [nodes-recursion.md](nodes-recursion.md) | `ListNode`, `TreeNode`, references, recursion patterns | Linked lists, trees, BST |
| [streams.md](streams.md) | Streams, method references, when they're worth it | Converting collections, one-line reductions |
| [idioms.md](idioms.md) | DSU, Trie, BIT, grid dirs, memo, fast I/O | Right before you start typing |
| [gotchas.md](gotchas.md) | The bugs that cost you the submission | After every wrong answer |

---

## The 60-Second Cheat Sheet

```java
int[] a = new int[n];                       // zeros
int[][] g = new int[r][c];                  // r rows, c cols, all zeros
Arrays.fill(a, -1);
Arrays.sort(a);                             // primitives: dual-pivot quicksort
int[] b = a.clone();                        // 1D copy
int[] c = Arrays.copyOfRange(a, l, r);      // [l, r)

List<Integer> list = new ArrayList<>();
Map<Integer,Integer> map = new HashMap<>();
Set<Integer> set = new HashSet<>();
Deque<Integer> stack = new ArrayDeque<>();  // use as stack AND queue AND deque
PriorityQueue<Integer> minHeap = new PriorityQueue<>();          // MIN-heap by default
PriorityQueue<Integer> maxHeap = new PriorityQueue<>((x,y) -> y - x);
TreeMap<Integer,Integer> tm = new TreeMap<>();                   // sorted, floorKey/ceilingKey

map.put(k, map.getOrDefault(k, 0) + 1);     // frequency count
map.merge(k, 1, Integer::sum);              // same thing, shorter
map.computeIfAbsent(k, x -> new ArrayList<>()).add(v);           // multimap

Collections.sort(list);
list.sort((x, y) -> x - y);
Arrays.sort(intervals, (x, y) -> Integer.compare(x[0], y[0]));   // 2D array by column 0

StringBuilder sb = new StringBuilder();
sb.append(x); sb.reverse(); sb.toString();
char[] ch = s.toCharArray();
```

---

## Complexity Master Table

| Type | Add | Remove | Get / Contains | Ordered? | Backing |
|---|---|---|---|---|---|
| `ArrayList` | O(1) amortized at end | O(n) | **O(1) by index** | insertion | array |
| `LinkedList` | O(1) at ends | O(1) at ends | **O(n) by index** | insertion | doubly linked |
| `ArrayDeque` | O(1) both ends | O(1) both ends | no indexing | insertion | circular array |
| `HashMap` / `HashSet` | O(1) avg | O(1) avg | O(1) avg | **none** | hash table |
| `LinkedHashMap` / `Set` | O(1) avg | O(1) avg | O(1) avg | **insertion or access** | hash + linked list |
| `TreeMap` / `TreeSet` | O(log n) | O(log n) | O(log n) | **sorted** | red-black tree |
| `PriorityQueue` | O(log n) | O(log n) `poll` | O(1) `peek`, **O(n) `contains`/`remove(Object)`** | heap | binary heap |

| Operation | Complexity | Note |
|---|---|---|
| `Arrays.sort(int[])` | O(n log n) average, **O(n²) adversarial** | dual-pivot quicksort — see [gotchas.md](gotchas.md) |
| `Arrays.sort(Object[])` / `Collections.sort` | O(n log n) guaranteed | TimSort, **stable** |
| `Arrays.binarySearch` | O(log n) | must be sorted; negative return encodes the insertion point |
| `String.substring` | O(n) — **copies** since Java 7 | not the O(1) view it once was |
| `s1 + s2` in a loop | O(n²) total | use `StringBuilder` |
| `list.contains` | O(n) | use a `HashSet` |
| `pq.remove(Object)` | O(n) | heaps can't search |

---

## Choosing a Type — Decision Table

| You need… | Use | Not |
|---|---|---|
| Stack | `Deque<Integer> st = new ArrayDeque<>()` | `Stack` — legacy, synchronized, iterates bottom-up |
| Queue | `Deque<Integer> q = new ArrayDeque<>()` | `LinkedList` — same API, worse cache behaviour |
| Both ends / monotonic deque | `ArrayDeque` | |
| Frequency count | `HashMap<K,Integer>` + `merge` | |
| Frequency of `a–z` | `int[26]` | a `HashMap` for 26 keys |
| Sorted keys, floor/ceiling queries | `TreeMap` / `TreeSet` | `HashMap` has no order |
| Insertion-ordered map (LRU) | `LinkedHashMap` | |
| Top-k, Dijkstra | `PriorityQueue` | |
| Repeated min/max **plus** arbitrary removal | `TreeMap<Integer,Integer>` as a multiset | `PriorityQueue.remove` is O(n) |
| Fixed-size numeric data | `int[]` | `List<Integer>` — boxing costs 4× memory and time |
| Result the judge expects as `List` | `List<Integer>` / `List<List<Integer>>` | |
| Building a string | `StringBuilder` | `+=` in a loop |

---

## The Java-Only Mistakes

Full list in [gotchas.md](gotchas.md), but these are the ones that decide submissions:

1. **`==` on objects compares references.** `Integer a = 1000, b = 1000; a == b` is `false`. Use `.equals()` — always, for `Integer`, `String`, `Character`.
2. **`map.get(k)` returns `null` for a missing key,** and unboxing `null` throws `NullPointerException`. Use `getOrDefault`.
3. **`list.remove(int)` removes by index; `list.remove(Object)` removes by value.** `list.remove(5)` on a `List<Integer>` removes index 5.
4. **`Arrays.sort(int[])` is O(n²) on adversarial input.** LeetCode has such tests. Box to `Integer[]`, or shuffle first.
5. **`int` overflow is silent.** `(lo + hi) / 2` wraps; so does `a * b` before it's assigned to a `long`.
6. **`char + char` is an `int`.** `char c = 'a' + 1;` needs a cast in most contexts.
7. **`PriorityQueue` with `(a, b) -> a - b`** overflows on large values. Use `Integer.compare(a, b)`.

---

## How to Use This Folder

- **First pass:** [arrays.md](arrays.md) → [collections.md](collections.md) → [comparators-sorting.md](comparators-sorting.md). That's the bulk of what you type.
- **Before a contest:** the cheat sheet above plus [idioms.md](idioms.md).
- **After a wrong answer:** [gotchas.md](gotchas.md). Java's failures are quiet — boxing, null, and overflow account for most of them.
- **Coming from C++:** read [gotchas.md](gotchas.md) first. The algorithms transfer; the object semantics do not.

# Streams & Lambdas — And When Not to Use Them

Streams shine at *converting* between representations and at one-line reductions. They are a liability inside hot loops: boxing, megamorphic call sites, and no early exit make them measurably slower than a plain `for`. Use them at the boundaries of your solution, not in its core.

**Rule of thumb: streams for setup and for the return value; loops for the algorithm.**

---

## Lambdas

```java
x -> x * 2                          // one parameter, no parens needed
(x, y) -> x + y
(int x, int y) -> x + y             // explicit types when inference fails
x -> { int y = x * 2; return y; }   // block body needs an explicit return
() -> doSomething()
```

### Effectively Final — The Big Restriction

A lambda can only capture locals that are never reassigned:

```java
int count = 0;
list.forEach(x -> count++);          // COMPILE ERROR: not effectively final

int[] count = {0};
list.forEach(x -> count[0]++);       // the standard workaround

AtomicInteger count = new AtomicInteger();
list.forEach(x -> count.incrementAndGet());
```

Fields have no such restriction — inside a `Solution` class, an instance field is the cleanest accumulator. This is why Java tree solutions use `private int best;` where C++ would use `[&]` capture.

Mutating a captured *object* is fine; only *rebinding* the variable is banned. `list.add(x)` inside a lambda works.

### Method References

```java
String::valueOf          // static method
Integer::parseInt
Integer::sum
Math::max
String::length           // instance method on the stream element
System.out::println      // instance method on a specific object
ArrayList::new           // constructor
int[]::new               // array constructor
Integer[]::new
```

`map.merge(k, 1, Integer::sum)` reads better than `map.merge(k, 1, (a, b) -> a + b)` and compiles to the same thing.

### Functional Interfaces

| Interface | Signature | Use |
|---|---|---|
| `Function<T,R>` | `R apply(T)` | transform |
| `BiFunction<T,U,R>` | `R apply(T,U)` | `merge`, `compute` |
| `Predicate<T>` | `boolean test(T)` | `filter`, `removeIf` |
| `Supplier<T>` | `T get()` | lazy defaults |
| `Consumer<T>` | `void accept(T)` | `forEach` |
| `UnaryOperator<T>` | `T apply(T)` | `replaceAll` |
| `IntUnaryOperator` | `int applyAsInt(int)` | primitive, no boxing |

The `Int*`/`Long*`/`Double*` variants avoid boxing. In numeric code, prefer `IntPredicate` over `Predicate<Integer>`.

---

## Streams — The Useful 20%

### Creating

```java
Arrays.stream(intArray);                     // IntStream  — primitive, no boxing
Arrays.stream(objArray);                     // Stream<T>
list.stream();
IntStream.range(0, n);                       // [0, n)
IntStream.rangeClosed(1, n);                 // [1, n]
Stream.of(1, 2, 3);
"abc".chars();                               // IntStream of code points
map.entrySet().stream();
```

`IntStream` vs `Stream<Integer>` is the distinction that matters. `IntStream` holds primitives; `.boxed()` converts to `Stream<Integer>`; `.mapToInt()` converts back.

### Transforming

```java
.map(x -> x * 2)
.mapToInt(Integer::intValue)                 // Stream<Integer> -> IntStream
.mapToObj(i -> nums[i])                      // IntStream -> Stream<T>
.boxed()                                     // IntStream -> Stream<Integer>
.filter(x -> x > 0)
.distinct()
.sorted()
.sorted(comparator)
.limit(k)  .skip(k)
.flatMap(list -> list.stream())              // flatten nested collections
```

### Terminating

```java
.sum()                                       // IntStream: returns int (LongStream for big sums)
.count()                                     // returns long
.max().getAsInt()   .min().getAsInt()        // OptionalInt
.average().orElse(0)                         // OptionalDouble
.toArray()                                   // int[] from IntStream, Object[] from Stream
.toArray(String[]::new)                      // typed array from Stream<String>
.collect(Collectors.toList())
.toList()                                    // Java 16+, IMMUTABLE result
.anyMatch(pred)  .allMatch(pred)  .noneMatch(pred)
.reduce(0, Integer::sum)
.findFirst().orElse(null)
.forEach(x -> ...)
```

**`.toList()` is immutable; `.collect(Collectors.toList())` is mutable.** If the judge or your own code then calls `add` or `sort` on the result, you need the collector version — or `.collect(Collectors.toCollection(ArrayList::new))`.

**`IntStream.sum()` returns `int` and overflows.** For large sums:

```java
Arrays.stream(a).asLongStream().sum();
Arrays.stream(a).mapToLong(x -> x).sum();
```

---

## The Conversions Worth Memorizing

```java
// int[] -> List<Integer>
Arrays.stream(a).boxed().collect(Collectors.toList());

// List<Integer> -> int[]
list.stream().mapToInt(Integer::intValue).toArray();

// int[] -> Integer[]
Arrays.stream(a).boxed().toArray(Integer[]::new);

// Integer[] -> int[]
Arrays.stream(boxed).mapToInt(Integer::intValue).toArray();

// List<int[]> -> int[][]
list.toArray(new int[0][]);

// int[] -> Set<Integer>
Arrays.stream(a).boxed().collect(Collectors.toSet());

// String -> List<Character>
s.chars().mapToObj(c -> (char) c).collect(Collectors.toList());

// List<String> -> String
String.join(",", list);
list.stream().collect(Collectors.joining(", ", "[", "]"));

// 2D array -> flat stream
Arrays.stream(grid).flatMapToInt(Arrays::stream);
```

These are the reason to know streams at all. `list.toArray(new int[0])` **does not compile** — `toArray` produces object arrays only, so the `mapToInt` route is mandatory.

---

## Collectors

```java
Collectors.toList()  toSet()  toCollection(ArrayList::new)
Collectors.toMap(keyFn, valFn)
Collectors.toMap(keyFn, valFn, mergeFn)          // mergeFn resolves duplicate keys
Collectors.groupingBy(classifier)                // Map<K, List<T>>
Collectors.groupingBy(classifier, Collectors.counting())   // Map<K, Long>
Collectors.partitioningBy(pred)                  // Map<Boolean, List<T>>
Collectors.joining(", ")
Collectors.counting()  summingInt(fn)  averagingInt(fn)
```

`Collectors.toMap` **throws `IllegalStateException` on a duplicate key** unless you supply a merge function:

```java
Collectors.toMap(k -> k, v -> 1, Integer::sum)
```

Genuinely useful one-liners:

```java
// Group anagrams (LC 49)
Collection<List<String>> groups = Arrays.stream(strs)
    .collect(Collectors.groupingBy(s -> {
        char[] c = s.toCharArray(); Arrays.sort(c); return new String(c);
    })).values();

// Frequency map
Map<Integer,Long> freq = Arrays.stream(nums).boxed()
    .collect(Collectors.groupingBy(x -> x, Collectors.counting()));
```

The frequency version returns `Long` counts, which then need casting everywhere. A plain `HashMap` + `merge` loop is usually the better trade:

```java
Map<Integer,Integer> freq = new HashMap<>();
for (int x : nums) freq.merge(x, 1, Integer::sum);
```

---

## Collection Methods That Take Lambdas

Often better than a stream — they mutate in place with no intermediate objects.

```java
list.forEach(x -> ...);
list.removeIf(x -> x == 0);                     // safe removal, no ConcurrentModificationException
list.replaceAll(x -> x * 2);                    // in-place map
list.sort(comparator);

map.forEach((k, v) -> ...);
map.merge(k, 1, Integer::sum);
map.computeIfAbsent(k, x -> new ArrayList<>());
map.computeIfPresent(k, (key, v) -> v - 1);
map.putIfAbsent(k, v);
map.replaceAll((k, v) -> v * 2);
map.entrySet().removeIf(e -> e.getValue() == 0);
```

`removeIf` is the correct way to delete during traversal — the manual `for` + `remove` throws `ConcurrentModificationException`.

---

## When Streams Cost You

```java
// Inside a hot loop — allocates a stream object per iteration
for (int i = 0; i < n; i++) sum += Arrays.stream(g[i]).sum();

// A plain loop is 3–10× faster
for (int i = 0; i < n; i++) for (int x : g[i]) sum += x;
```

Concrete downsides in a judged submission:

- **Boxing.** `Stream<Integer>` allocates per element. `IntStream` avoids it — check which one you're on.
- **No early exit from a `forEach`.** `return` inside the lambda returns from the *lambda*, not the enclosing method. Use `anyMatch`/`findFirst`, or a plain loop with `break`.
- **No index.** Simulating one with `IntStream.range` is noisier than a `for`.
- **Checked exceptions** can't propagate out of a lambda.
- **Debugging.** A stack trace through stream internals is far harder to read.

Use streams for the conversion lines and the return statement. Write the algorithm as loops.

---

## Optional

Stream terminals return `Optional` / `OptionalInt`:

```java
OptionalInt max = Arrays.stream(a).max();
int m = max.getAsInt();                        // throws if empty
int m = max.orElse(0);                         // safe default
if (max.isPresent()) { }
Arrays.stream(a).max().ifPresent(x -> ...);
```

`.get()` / `.getAsInt()` on an empty `Optional` throws `NoSuchElementException`. When the collection might be empty, use `orElse`.

---

## Quick Reference

| Task | Stream |
|---|---|
| Sum an `int[]` | `Arrays.stream(a).sum()` |
| Sum, overflow-safe | `Arrays.stream(a).asLongStream().sum()` |
| Max of an `int[]` | `Arrays.stream(a).max().getAsInt()` |
| `int[]` → `List<Integer>` | `Arrays.stream(a).boxed().collect(Collectors.toList())` |
| `List<Integer>` → `int[]` | `list.stream().mapToInt(Integer::intValue).toArray()` |
| `List<int[]>` → `int[][]` | `list.toArray(new int[0][])` |
| Sorted distinct | `Arrays.stream(a).distinct().sorted().toArray()` |
| Count matches | `Arrays.stream(a).filter(pred).count()` |
| Join strings | `String.join(",", list)` |
| Group by key | `Collectors.groupingBy(fn)` |
| 0..n−1 sequence | `IntStream.range(0, n)` |

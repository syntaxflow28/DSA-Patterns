# Strings & Characters

`String` is **immutable**. Every operation that appears to modify one actually allocates a new object. That single fact drives every performance decision on this page.

```java
String s = "abc";
s.toUpperCase();          // returns a NEW string; s is unchanged
s = s.toUpperCase();      // this is how you "modify" it
```

---

## Core `String` Methods

```java
s.length();                       // METHOD — arrays use the .length FIELD
s.charAt(i);                      // returns char
s.isEmpty();                      // length == 0
s.isBlank();                      // empty or whitespace only (Java 11)
s.substring(i);                   // i to the end
s.substring(i, j);                // [i, j) — j is EXCLUSIVE
s.indexOf("ab");                  // -1 if absent
s.indexOf('a', from);
s.lastIndexOf("ab");
s.contains("ab");
s.startsWith("ab");  s.endsWith("ab");
s.equals(t);                      // VALUE comparison — use this, never ==
s.equalsIgnoreCase(t);
s.compareTo(t);                   // <0, 0, >0 — lexicographic
s.toUpperCase();  s.toLowerCase();
s.trim();                         // strips ASCII whitespace
s.strip();                        // Unicode-aware (Java 11)
s.replace('a', 'b');              // ALL occurrences, plain chars
s.replace("ab", "cd");            // plain substring
s.replaceAll("[0-9]+", "");       // REGEX
s.repeat(n);                      // Java 11
s.toCharArray();
s.chars();                        // IntStream of code points
String.join(",", list);
String.valueOf(x);
String.format("%d-%s", i, str);
```

**`substring(i, j)` is exclusive at `j`** — `s.substring(0, s.length())` is the whole string. And since Java 7 it **copies** in O(n); the old O(1) shared-array version is gone. `substring` inside a nested loop is an easy O(n³).

---

## `==` vs `.equals()`

```java
String a = "hello";
String b = "hello";
a == b;                     // true — both point to the interned literal

String c = new String("hello");
a == c;                     // FALSE — different objects
a.equals(c);                // true

String d = "hel" + "lo";    // compile-time constant, interned
a == d;                     // true
String e = "hel" + someVar; // built at runtime
a == e;                     // FALSE
```

**Always `.equals()`.** The literal-interning behaviour makes `==` work in small tests and fail on computed strings — the worst kind of bug.

Null-safe form:

```java
"target".equals(s);          // safe even if s is null
Objects.equals(a, b);        // safe both ways
```

---

## `StringBuilder` — Build, Don't Concatenate

```java
String s = "";
for (int i = 0; i < n; i++) s += i;        // O(n²) — a fresh string every iteration

StringBuilder sb = new StringBuilder();
for (int i = 0; i < n; i++) sb.append(i);  // O(n) total
String s = sb.toString();
```

```java
StringBuilder sb = new StringBuilder();
StringBuilder sb = new StringBuilder(capacity);
StringBuilder sb = new StringBuilder("start");

sb.append(x);                     // accepts any type: char, int, String, boolean...
sb.append(c).append(i);           // chainable
sb.insert(i, x);
sb.deleteCharAt(i);
sb.delete(from, to);              // [from, to)
sb.setCharAt(i, c);
sb.charAt(i);
sb.setLength(n);                  // truncate — the fast "pop" in backtracking
sb.reverse();                     // IN PLACE, returns this
sb.length();
sb.indexOf("ab");
sb.toString();
```

`setLength` is the backtracking undo:

```java
int len = sb.length();
sb.append(c);
recurse();
sb.setLength(len);                // O(1) restore — the Java equivalent of path.pop_back()
```

Use `sb.deleteCharAt(sb.length() - 1)` when you appended exactly one char; `setLength` is more general and cheaper.

**`StringBuilder` has no `equals`** — `sb1.equals(sb2)` compares references. Compare `sb1.toString().equals(sb2.toString())`.

`StringBuffer` is the synchronized version. Never use it here.

---

## `char[]` — When You Need Mutation

```java
char[] c = s.toCharArray();
Arrays.sort(c);                          // sorting a string's chars — the anagram key
String sorted = new String(c);
String sorted = String.valueOf(c);       // identical

c[i] = 'x';                              // mutable, unlike String
```

```java
// Anagram grouping key
char[] c = s.toCharArray();
Arrays.sort(c);
map.computeIfAbsent(new String(c), k -> new ArrayList<>()).add(s);

// In-place reversal (LC 344 hands you a char[])
int l = 0, r = c.length - 1;
while (l < r) { char t = c[l]; c[l++] = c[r]; c[r--] = t; }
```

For hot loops, `s.charAt(i)` in a tight loop is slower than converting once to `char[]` and indexing. Worth doing when a string solution is borderline.

---

## Characters

```java
char c = 'a';
int i = c;                        // 97 — implicit widening
char c = (char) (i + 1);          // int -> char needs an EXPLICIT cast

c - 'a';                          // 0..25 — the array-index idiom
(char) ('a' + k);

Character.isLetter(c);
Character.isDigit(c);
Character.isLetterOrDigit(c);
Character.isWhitespace(c);
Character.isUpperCase(c);  Character.isLowerCase(c);
Character.toUpperCase(c);  Character.toLowerCase(c);
Character.getNumericValue(c);     // '7' -> 7
```

**`char + char` is an `int`.** Arithmetic on chars promotes to `int`, so:

```java
char c = 'a' + 1;                 // OK — compile-time constant fits in char
char c = someChar + 1;            // COMPILE ERROR — needs (char)
sb.append(someChar + 1);          // appends the INT 98, not 'b'
sb.append((char) (someChar + 1)); // correct
```

That `append` case is a real bug: it compiles, runs, and produces `"98"` in your output.

---

## Character Frequency — Use an Array

```java
int[] cnt = new int[26];
for (char c : s.toCharArray()) cnt[c - 'a']++;

int[] cnt = new int[128];                    // full ASCII: mixed case, digits, symbols
for (char c : s.toCharArray()) cnt[c]++;     // char auto-widens to the int index
```

A `HashMap<Character,Integer>` for 26 keys costs boxing on every access. Use the array.

```java
// Anagram check (LC 242)
if (s.length() != t.length()) return false;
int[] cnt = new int[26];
for (int i = 0; i < s.length(); i++) { cnt[s.charAt(i) - 'a']++; cnt[t.charAt(i) - 'a']--; }
for (int x : cnt) if (x != 0) return false;
return true;

// int[26] as a map key — Arrays.toString gives a usable String key
map.computeIfAbsent(Arrays.toString(cnt), k -> new ArrayList<>()).add(s);
```

---

## Splitting and Parsing

```java
String[] parts = s.split(",");
String[] parts = s.split("\\s+");        // one or more whitespace — REGEX
String[] parts = s.split("");            // every character
String[] parts = s.split(",", -1);       // KEEP trailing empty strings
```

**`split` takes a regex, not a literal.** Splitting on `.` or `|` or `+` needs escaping:

```java
s.split("\\.");         // split on a literal dot
s.split("\\|");
Pattern.quote(".")      // escape anything programmatically
```

**`split` drops trailing empty strings by default.** `"a,b,,".split(",")` gives `["a","b"]`, not four elements. Pass `-1` as the limit to keep them — this matters in parsing problems.

Whitespace tokenizing:

```java
for (String w : s.trim().split("\\s+")) { }        // trim first, or a leading space yields ""
```

`StringTokenizer` is faster and allocation-free, worth it for heavy input parsing:

```java
StringTokenizer st = new StringTokenizer(s);
while (st.hasMoreTokens()) process(st.nextToken());
```

### Numbers

```java
int i = Integer.parseInt(s);              // throws NumberFormatException on bad input
long l = Long.parseLong(s);
double d = Double.parseDouble(s);
int i = Integer.parseInt(s, 2);           // parse binary
int i = Integer.parseInt(s, 16);          // parse hex

String s = String.valueOf(i);             // preferred
String s = Integer.toString(i);
String s = Integer.toBinaryString(i);
String s = Integer.toHexString(i);
String s = "" + i;                        // works, allocates a StringBuilder
```

`Integer.parseInt` returns `int`; `Integer.valueOf` returns `Integer` (boxed, and cached for −128..127). Use `parseInt` unless you specifically need the object.

Manual digit parsing — what LC 8 "String to Integer (atoi)" actually requires:

```java
int d = c - '0';
char c = (char) (d + '0');
```

---

## Formatting

```java
String.format("%d", 42);
String.format("%.2f", 3.14159);           // "3.14"
String.format("%05d", 42);                // "00042"
String.format("%s-%s", a, b);
String.join("-", "a", "b", "c");          // "a-b-c"
String.join(",", list);                   // List<String> -> "a,b,c"
```

`String.join` needs `CharSequence` elements. For a `List<Integer>`:

```java
list.stream().map(String::valueOf).collect(Collectors.joining(","));
```

---

## Common String Patterns

```java
// Palindrome check
boolean isPal(String s, int l, int r) {
    while (l < r) if (s.charAt(l++) != s.charAt(r--)) return false;
    return true;
}

// Reverse
String r = new StringBuilder(s).reverse().toString();

// Expand around center (LC 5)
for (int i = 0; i < n; i++) { expand(s, i, i); expand(s, i, i + 1); }

// Sliding window over a string
int[] need = new int[128];
for (char c : t.toCharArray()) need[c]++;
int left = 0, missing = t.length();
for (int right = 0; right < s.length(); right++) {
    if (need[s.charAt(right)]-- > 0) missing--;
    while (missing == 0) { ... if (++need[s.charAt(left++)] > 0) missing++; }
}

// Rolling hash (Rabin-Karp)
long B = 131, MOD = 1_000_000_007L, h = 0;
for (char c : s.toCharArray()) h = (h * B + c) % MOD;

// Trie key by char
int idx = c - 'a';
```

---

## Complexity Reference

| Operation | Cost |
|---|---|
| `charAt`, `length` | O(1) |
| `substring(i, j)` | **O(j − i) — copies** |
| `s1 + s2` | O(n + m), new object |
| `s += x` in a loop | **O(n²) total** |
| `sb.append` | O(1) amortized |
| `equals`, `compareTo` | O(n) |
| `indexOf`, `contains` | O(n·m) worst case |
| `split` | O(n) + regex compilation |
| `toCharArray` | O(n), one copy |
| `Arrays.sort(char[])` | O(n log n) |

**Reuse compiled patterns** when a regex runs in a loop:

```java
static final Pattern P = Pattern.compile("\\d+");     // compile once
Matcher m = P.matcher(s);
```

`s.split(regex)` and `s.replaceAll(regex, x)` recompile the pattern on every call.

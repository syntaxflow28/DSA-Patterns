# Strings — `std::string`, Conversions, Parsing, Char Utilities

`std::string` is a `vector<char>` with extra methods, so everything from [containers-sequence.md](containers-sequence.md) applies: `size()`, `push_back`, `back()`, iterators, range-for, and every `<algorithm>` function.

---

## Construction

```cpp
string s = "hello";
string s(5, 'a');                    // "aaaaa"
string s(v.begin(), v.end());        // from a range of chars
string s = string(1, c);             // single char -> string
string s(other, pos, len);           // substring copy
```

---

## Core Operations

```cpp
s.size();  s.length();               // identical
s.empty();
s[i];  s.at(i);  s.front();  s.back();
s.substr(pos);                       // pos to the end
s.substr(pos, len);                  // len chars starting at pos (clamped at the end)
s += "abc";  s += 'c';               // append — O(1) amortized
s.push_back('c');  s.pop_back();
s.append(str);  s.append(n, 'x');
s.insert(pos, "abc");
s.erase(pos, len);
s.replace(pos, len, "new");
s.clear();
s.compare(t);                        // <0, 0, >0 — prefer ==, <, > which work directly
```

`substr` copies — an O(n) allocation. Inside a loop over all substrings that's O(n³) and a common TLE cause. Use indices, or `string_view` (C++17) for read-only slicing:

```cpp
string_view sv(s);
sv.substr(i, len);                   // O(1) — no copy, but the source string must outlive it
```

---

## Searching

```cpp
s.find("abc");                       // index of the first occurrence, or string::npos
s.find('a', startPos);
s.rfind("abc");                      // last occurrence
s.find_first_of("aeiou");            // first index of ANY of those chars
s.find_last_of("aeiou");
s.find_first_not_of(" \t");          // first non-whitespace — used for trimming

if (s.find("abc") != string::npos) { ... }        // the correct existence test

s.starts_with("pre");  s.ends_with("suf");        // C++20
s.contains("mid");                                 // C++23
```

**`string::npos` is `(size_t)-1`, the largest `size_t`.** So `if (s.find(x) >= 0)` is *always true* and `if (s.find(x) != -1)` works only by accident of the conversion. Always compare against `string::npos`.

---

## Conversions

```cpp
// string -> number
int    a  = stoi(s);
long   b  = stol(s);
long long c = stoll(s);
double d  = stod(s);
int base16 = stoi(s, nullptr, 16);         // parse hex
int base2  = stoi(s, nullptr, 2);          // parse binary

size_t pos;
int n = stoi(s, &pos);                     // pos = index after the parsed number

// number -> string
string s = to_string(42);
string s = to_string(3.14);                // gives "3.140000" — 6 decimals, often unwanted

// char <-> int digit
int d = c - '0';
char c = d + '0';

// char -> string
string t(1, c);
string t = string(1, c) + "abc";           // NOT "abc" + c
```

`stoi` throws `invalid_argument` on non-numeric input and `out_of_range` on overflow. LeetCode inputs are usually clean, but "String to Integer (atoi)" (LC 8) explicitly requires manual parsing with clamping — don't use `stoi` there.

**`'a' + 1` is an `int`, not a `char`.** `s += 'a' + 1` appends the integer 98, promoted back to a char — it happens to work, but `s += (char)('a' + 1)` is what you mean.

---

## Character Classification — `<cctype>`

```cpp
isalpha(c);   isdigit(c);   isalnum(c);
isspace(c);   isupper(c);   islower(c);
ispunct(c);
tolower(c);   toupper(c);                 // return INT — cast when assigning to char
```

```cpp
for (char& c : s) c = tolower(c);         // implicit narrowing, fine in practice
transform(s.begin(), s.end(), s.begin(), ::tolower);   // idiomatic whole-string lowercase
```

Pass a non-ASCII negative `char` and these are technically UB; irrelevant for LeetCode's ASCII inputs.

---

## Character Frequency — Don't Use a Map

For a lowercase alphabet, an array is faster and shorter:

```cpp
vector<int> cnt(26, 0);
for (char c : s) cnt[c - 'a']++;

int cnt[128] = {};                       // full ASCII, handles mixed case and symbols
for (char c : s) cnt[(int)c]++;
```

```cpp
// Anagram check (LC 242)
vector<int> cnt(26, 0);
for (char c : s) cnt[c - 'a']++;
for (char c : t) if (--cnt[c - 'a'] < 0) return false;
return true;                              // sizes already checked equal

// A 26-int vector as a hash-map key for grouping anagrams
map<vector<int>, vector<string>> groups;
```

---

## Parsing with `stringstream` — `<sstream>`

The fastest way to split on whitespace:

```cpp
#include <sstream>
stringstream ss(s);
string word;
while (ss >> word) words.push_back(word);        // skips ALL whitespace runs automatically
```

Split on an arbitrary delimiter:

```cpp
stringstream ss(s);
string token;
while (getline(ss, token, ',')) parts.push_back(token);   // keeps empty tokens
```

Mixed extraction:

```cpp
stringstream ss("12 abc 3.5");
int a; string b; double c;
ss >> a >> b >> c;
```

Building strings efficiently:

```cpp
stringstream out;
for (int x : v) out << x << ",";
string res = out.str();
```

For heavy concatenation, `s += piece` on a `string` with `reserve` is actually faster than `stringstream`. Use `stringstream` for parsing, `+=` for building.

Formatting a double to fixed decimals:

```cpp
stringstream ss;
ss << fixed << setprecision(5) << x;       // needs <iomanip>
string s = ss.str();
```

---

## Manual Split (No `stringstream`)

Sometimes clearer and always faster:

```cpp
vector<string> split(const string& s, char delim) {
    vector<string> out;
    string cur;
    for (char c : s) {
        if (c == delim) { out.push_back(cur); cur.clear(); }
        else cur += c;
    }
    out.push_back(cur);
    return out;
}
```

---

## String Algorithms You'll Reuse

```cpp
// Reverse
reverse(s.begin(), s.end());

// Palindrome check
bool isPal(const string& s, int l, int r) {
    while (l < r) if (s[l++] != s[r--]) return false;
    return true;
}

// Sort characters — the canonical anagram key
string key = s;
sort(key.begin(), key.end());

// Trim
size_t a = s.find_first_not_of(" \t\n");
size_t b = s.find_last_not_of(" \t\n");
s = (a == string::npos) ? "" : s.substr(a, b - a + 1);

// Repeat
string r;
for (int i = 0; i < n; i++) r += s;
// or: string r(n, 'x') for a single repeated char

// Join
string join(const vector<string>& v, const string& sep) {
    string res;
    for (int i = 0; i < (int)v.size(); i++) { if (i) res += sep; res += v[i]; }
    return res;
}
```

---

## Rolling Hash (Rabin-Karp)

For substring-matching problems where you need O(1) comparison:

```cpp
const long long B = 131, MOD = 1e9 + 7;
vector<long long> h(n + 1, 0), p(n + 1, 1);
for (int i = 0; i < n; i++) {
    h[i + 1] = (h[i] * B + s[i]) % MOD;
    p[i + 1] = p[i] * B % MOD;
}
auto sub = [&](int l, int r) {                   // hash of s[l..r]
    return ((h[r + 1] - h[l] * p[r - l + 1]) % MOD + MOD) % MOD;
};
```

The `+ MOD) % MOD` is mandatory — the subtraction can go negative, and C++'s `%` keeps the sign of the dividend.

---

## Reading Input (Non-LeetCode)

LeetCode hands you parsed arguments, but for Codeforces or local testing:

```cpp
string line;
getline(cin, line);                 // reads a whole line INCLUDING spaces
cin >> x;                           // leaves the newline in the buffer
cin.ignore();                       // consume it before the next getline
getline(cin, line);
```

Mixing `>>` and `getline` without `cin.ignore()` gives you a mysterious empty line. It catches everyone once.

---

## Complexity Reference

| Operation | Cost |
|---|---|
| `s[i]`, `size()`, `back()` | O(1) |
| `s += c`, `push_back` | O(1) amortized |
| `s += t` | O(len(t)) amortized |
| `substr(p, l)` | **O(l) — copies** |
| `find(t)` | O(n·m) worst case |
| `insert` / `erase` in the middle | O(n) |
| `==`, `<` | O(n) |
| `sort(s.begin(), s.end())` | O(n log n) |

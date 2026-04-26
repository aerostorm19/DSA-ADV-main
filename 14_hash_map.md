# Hash Map / Hash Table

> **Curriculum position:** Hash-Based Structures → #1
> **Interview weight:** ★★★★★ Critical — the single most-used data structure in interview problems after arrays
> **Difficulty:** Beginner concept, deep internals worth mastering
> **Prerequisite:** [01 — Static Array](./01_static_array.md) | [03 — Singly Linked List](./03_singly_linked_list.md)

---

## Table of Contents

1. [Intuition First — The Phone Book Analogy](#1-intuition-first)
2. [The Hash Function — Heart of the Structure](#2-the-hash-function)
3. [Internal Working — Buckets and Slots](#3-internal-working)
4. [Collision Handling — Two Strategies](#4-collision-handling)
5. [Load Factor and Rehashing](#5-load-factor-and-rehashing)
6. [Time & Space Complexity](#6-time--space-complexity)
7. [Complete C++ Implementation](#7-complete-c-implementation)
8. [Core Operations — Visualised](#8-core-operations--visualised)
9. [Common Patterns & Techniques](#9-common-patterns--techniques)
10. [Interview Problems](#10-interview-problems)
11. [Real-World Uses](#11-real-world-uses)
12. [Edge Cases & Pitfalls](#12-edge-cases--pitfalls)
13. [std::unordered\_map — Complete API Reference](#13-stdunordered_map)
14. [Comparison: Hash Map vs BST Map vs Array vs Trie](#14-comparison)
15. [Self-Test Questions](#15-self-test-questions)

---

## 1. Intuition First

A phone book lets you find anyone's number instantly — you do not scan from page one. You open directly to the right letter section, then scan a short alphabetical list. The key insight: the name itself tells you where to look.

A hash map takes this further. Instead of "first letter gives a section," a **hash function** converts any key — a string, an integer, a URL — into a precise array index. That index is the exact slot where the value lives. Finding a value is not a scan; it is a single arithmetic computation followed by one array lookup.

This is the fundamental idea: **keys are their own addresses**. Instead of searching for data, you compute where it should be and go directly there.

```
Key: "alice"
Hash function: h("alice") = 42
Value stored at: array[42] = "+1-555-0101"

Lookup "alice":
  1. Compute h("alice") = 42          ← one operation
  2. Return array[42]                  ← one memory access
  Total: O(1)
```

No binary search. No tree traversal. No pointer following. One computation, one memory access. This is why hash maps are the default answer to "can we do better than O(n) search?"

The price is twofold. First, hash maps use more memory than arrays — they pre-allocate space beyond what is currently needed to keep operations fast. Second, two different keys might hash to the same index — a **collision** — and the map must handle this gracefully. The entire craft of hash map design is managing these two costs.

---

## 2. The Hash Function — Heart of the Structure

The hash function `h(key)` converts an arbitrary key into an integer index in `[0, capacity)`. It must satisfy two properties:

1. **Deterministic**: the same key always produces the same index.
2. **Efficient**: computing the hash must be O(1) or close to it.

And ideally:

3. **Uniform distribution**: keys should spread evenly across all buckets to minimise collisions.
4. **Avalanche effect**: small changes in the key produce large changes in the hash (cryptographic hashes maximise this; hash map hashes approximate it).

### Integer Keys

```cpp
// Simplest: modulo
h(k) = k % capacity

// Problem: if capacity = 100 and keys are multiples of 10, only 10 buckets used.
// Solution: choose capacity as a prime number — primes reduce patterns in modulo.

// Better: multiply-and-shift (Knuth's multiplicative hashing)
// A = 2654435769 (golden ratio × 2^32, odd number)
h(k) = ((unsigned int)(k * 2654435769U)) >> (32 - log2(capacity))

// Even better: FNV-1a, MurmurHash, xxHash for integers
```

### String Keys

```cpp
// Polynomial rolling hash
// h(s) = (s[0]*p^(n-1) + s[1]*p^(n-2) + ... + s[n-1]) % m
// p = 31 or 37, m = large prime or power of 2

size_t hashString(const string& s) {
    size_t hash = 0;
    size_t p = 31;
    size_t p_pow = 1;
    for (char c : s) {
        hash += (c - 'a' + 1) * p_pow;
        p_pow *= p;
    }
    return hash;
}

// std::hash<std::string> in C++ uses a more sophisticated variant (MurmurHash or similar).
```

### What Makes a Bad Hash Function?

```
Identity hash:   h(k) = k
  Problem: keys that are close integers cluster in adjacent buckets.
  For strings: identity makes no sense — cannot use a string as an index.

Constant hash:   h(k) = 0
  Problem: every key maps to bucket 0. Every operation is O(n) — reduces
  the hash map to a linked list.

Division by non-prime: h(k) = k % 100
  Problem: keys that are multiples of 2 or 5 cluster in 50% or 20% of buckets.

Good hash functions are engineered to avoid these patterns.
```

---

## 3. Internal Working — Buckets and Slots

### The Array of Buckets

A hash table is fundamentally an array. Each position in the array is called a **bucket** or **slot**. The hash function maps each key to one bucket.

```
capacity = 8 buckets, indexed 0..7:

Bucket: [  0  ][  1  ][  2  ][  3  ][  4  ][  5  ][  6  ][  7  ]
         empty  empty  empty  empty  empty  empty  empty  empty

insert("cat", 10):  h("cat") = 3
Bucket: [  0  ][  1  ][  2  ][ cat:10 ][  4  ][  5  ][  6  ][  7  ]

insert("dog", 20):  h("dog") = 6
Bucket: [  0  ][  1  ][  2  ][ cat:10 ][  4  ][  5  ][ dog:20 ][  7  ]

insert("fox", 25):  h("fox") = 3  ← COLLISION! Bucket 3 already has "cat".
```

The collision problem is unavoidable — by the **pigeonhole principle**, if you store more keys than buckets, some bucket must have multiple keys. Even with fewer keys than buckets, birthday-paradox mathematics guarantees frequent collisions once load exceeds ~70%.

---

## 4. Collision Handling — Two Strategies

### Strategy 1: Separate Chaining

Each bucket holds a **linked list** (or other structure) of all key-value pairs that hash to that bucket.

```
capacity = 8 buckets, all with chaining:

Bucket 0: → null
Bucket 1: → null
Bucket 2: → null
Bucket 3: → [cat:10] → [fox:25] → null   ← two keys in same bucket
Bucket 4: → null
Bucket 5: → null
Bucket 6: → [dog:20] → null
Bucket 7: → null

Lookup "fox":
  h("fox") = 3
  Traverse chain at bucket 3: cat≠fox → fox=fox → found! Return 25.
```

**Properties of chaining:**
- Buckets can hold unlimited entries (chains grow arbitrarily)
- Load factor > 1 is possible (more entries than buckets)
- Worst case: all keys hash to one bucket → O(n) per operation
- Cache unfriendly: each chain node is a separate heap allocation

**Chaining in `std::unordered_map`:**  
GCC's `std::unordered_map` uses chaining with a singly-linked list per bucket. This is the most common production implementation.

### Strategy 2: Open Addressing

All entries are stored directly in the array. When a collision occurs, the map **probes** (searches) for the next empty slot using a probing sequence.

**Linear Probing** — probe indices: `h(k), h(k)+1, h(k)+2, ...` (mod capacity)

```
capacity = 8, insert "cat"(h=3), "fox"(h=3), "ant"(h=4):

Start: [_][_][_][_][_][_][_][_]

insert("cat"): slot 3 is empty → place at 3.
  [_][_][_][cat:10][_][_][_][_]

insert("fox"): h("fox")=3, slot 3 is occupied.
  Linear probe: try 4, empty → place at 4.
  [_][_][_][cat:10][fox:25][_][_][_]

insert("ant"): h("ant")=4, slot 4 is occupied.
  Linear probe: try 5, empty → place at 5.
  [_][_][_][cat:10][fox:25][ant:7][_][_]

Lookup "fox": h("fox")=3, slot 3 has "cat"≠"fox". Probe 4: "fox"=found! ✓
```

**Clustering problem with linear probing:**

```
A run of occupied consecutive slots = a "cluster."
Inserting into a cluster extends it, making future insertions hit the cluster
and probe even further. Long clusters → O(n) probe sequences.

[_][_][_][cat][fox][ant][elk][_]   ← cluster of length 4 at positions 3-6
insert new key with h=3: must probe 3,4,5,6 before finding empty slot 7.
```

**Quadratic Probing** — probe indices: `h(k), h(k)+1², h(k)+2², h(k)+3², ...`

```
Reduces primary clustering (consecutive runs), but creates secondary clustering
(keys with same hash follow the same probe sequence).
```

**Double Hashing** — probe indices: `h1(k), h1(k)+h2(k), h1(k)+2*h2(k), ...`

```
Uses a second hash function to compute the step size.
Eliminates secondary clustering. Best distribution among open addressing variants.
Both h1 and h2 must be computed, slightly slower.
```

**Comparison of probing strategies:**

| Strategy | Primary Clustering | Secondary Clustering | Cache Behaviour | Complexity |
|---|---|---|---|---|
| Linear probing | Yes (severe) | No | ★★★ Best | Simple |
| Quadratic probing | Reduced | Yes | ★★ Good | Moderate |
| Double hashing | No | No | ★ Poor (random jumps) | Complex |

**Robin Hood Hashing** — a linear probing variant that reduces variance:

When a new key is being inserted and encounters a key with a shorter probe distance (it is "richer"), steal the slot and continue inserting the displaced "poorer" key. This balances probe distances across all stored keys, keeping the maximum probe length bounded.

```
Probe distance (DIB = distance from initial bucket):

Standard:  [cat(DIB=0)][fox(DIB=1)][ant(DIB=2)][elk(DIB=3)]
           fox, ant, elk all suffered because of cat's perfect placement.

Robin Hood: when inserting elk (DIB=3) encounters ant (DIB=2):
  DIB(elk)=3 > DIB(ant)=2 → steal ant's slot, continue placing ant.
  Result: maximum DIB is reduced; average DIB is more uniform.
```

### Open Addressing vs Chaining — The Decision

| Property | Open Addressing | Chaining |
|---|---|---|
| Cache performance | ★★★ All data in one array | ★ Linked list = cache misses |
| Memory per entry | Zero overhead (key+value only) | One pointer per entry minimum |
| Load factor limit | Must stay ≤ 0.7 typically | Can exceed 1.0 |
| Deletion complexity | Complex (needs tombstones) | Simple (remove from chain) |
| Performance at low load | Faster (cache wins) | Similar |
| Performance at high load | Degrades fast | Degrades slowly |
| Used in | Python dicts, Java HashMap (alt), Google dense_hash | std::unordered_map, Java HashMap (default) |

---

## 5. Load Factor and Rehashing

### Load Factor

```
load_factor = number_of_entries / number_of_buckets

Examples:
  8 entries, 16 buckets → load_factor = 0.5   (healthy)
  14 entries, 16 buckets → load_factor = 0.875 (too high — rehash soon)
  0 entries, 16 buckets → load_factor = 0.0   (wasteful memory)
```

The load factor controls the collision rate. At load factor α with good hashing:
- **Chaining**: expected chain length = α. O(1 + α) average lookup.
- **Open addressing**: expected probe length ≈ 1/(1-α). At α=0.9, expect 10 probes per lookup.

**Maximum load factors (typical defaults):**
- `std::unordered_map` (chaining): max_load_factor = **1.0** (rehash when entries > buckets)
- Python dict (open addressing): max_load_factor = **0.67** (rehash when 2/3 full)
- Robin Hood / Swiss Table: max_load_factor = **0.875** (carefully engineered)

### Rehashing

When the load factor exceeds the maximum, the hash map **rehashes**:

```
1. Allocate a new array (typically 2× current capacity)
2. For each existing key-value pair:
   a. Recompute h(key) with the new capacity
   b. Insert into the new array
3. Replace the old array with the new array

Cost: O(n) — must re-insert every existing entry.
Frequency: doubles each time → rehash at sizes 16, 32, 64, 128, ...
Amortised cost per insertion: O(1) — same argument as dynamic array push_back.
```

```
Before rehash (capacity=4, load=1.0):
  Bucket 0: [a:1]
  Bucket 1: [b:2] → [e:5]
  Bucket 2: [c:3]
  Bucket 3: [d:4]

After rehash (capacity=8):
  h(a) = 0 → Bucket 0: [a:1]
  h(b) = 5 → Bucket 5: [b:2]    (new hash values with capacity=8)
  h(c) = 2 → Bucket 2: [c:3]
  h(d) = 3 → Bucket 3: [d:4]
  h(e) = 1 → Bucket 1: [e:5]    (e now has its own bucket)
  Buckets 4,6,7: empty
```

---

## 6. Time & Space Complexity

### Operation Complexities

| Operation | Average Case | Worst Case | Notes |
|---|---|---|---|
| `insert(k, v)` | **O(1)** | O(n) | Worst case: all keys collide in one bucket |
| `lookup(k)` | **O(1)** | O(n) | Worst case: same as insert |
| `delete(k)` | **O(1)** | O(n) | Chaining: O(1); open addressing needs tombstone |
| `contains(k)` | **O(1)** | O(n) | Same as lookup |
| `rehash` | O(n) | O(n) | Triggered when load factor exceeded |
| Iteration | **O(n + capacity)** | O(n + capacity) | Must scan all buckets including empty |
| `size()` | **O(1)** | O(1) | Tracked separately |

**Why worst case O(n)?**  
A deliberately constructed set of keys that all hash to the same bucket degrades the hash map to a linked list. This is a real attack vector — **hash flooding attacks** deliberately craft inputs to force worst-case behaviour. Defences include randomised hash seeds (Python 3.3+) and cryptographic hash functions for user-controlled inputs.

### Space Complexity

| | |
|---|---|
| n entries stored | O(n) |
| Pre-allocated capacity | O(capacity) — typically 2n to 4n |
| Per-entry overhead (chaining) | One pointer (8 bytes) per entry |
| Per-entry overhead (open addressing) | 0 bytes overhead; key+value+metadata |
| Load factor determines utilisation | Typically 50-87% of allocated space is used |

### Amortised Complexity

```
n insertions into an empty hash map (starting capacity = c, doubling on rehash):

Rehashes occur at sizes: c, 2c, 4c, 8c, ..., n
Cost of each rehash: c, 2c, 4c, ..., n

Total rehash cost = c + 2c + 4c + ... + n = 2n - c = O(n)
Total insertion cost (without rehash) = n × O(1) = O(n)

Amortised cost per insertion = O(n) / n = O(1) ✓
```

---

## 7. Complete C++ Implementation

### Implementation 1: Hash Map with Separate Chaining

```cpp
#include <iostream>
#include <vector>
#include <list>
#include <functional>
#include <stdexcept>
#include <string>
using namespace std;

template<typename K, typename V>
class HashMap {
private:
    struct Entry {
        K key;
        V value;
    };

    vector<list<Entry>> buckets_;  // array of chains
    size_t              size_;     // number of stored entries
    size_t              capacity_; // number of buckets
    float               max_load_; // rehash threshold

    size_t bucket_idx(const K& key) const {
        return hash<K>{}(key) % capacity_;
    }

    void rehash() {
        size_t new_cap = capacity_ * 2;
        vector<list<Entry>> new_buckets(new_cap);

        for (auto& chain : buckets_) {
            for (auto& entry : chain) {
                size_t idx = hash<K>{}(entry.key) % new_cap;
                new_buckets[idx].push_back(entry);
            }
        }

        buckets_  = move(new_buckets);
        capacity_ = new_cap;
    }

public:
    explicit HashMap(size_t initial_capacity = 16, float max_load = 0.75f)
        : buckets_(initial_capacity), size_(0),
          capacity_(initial_capacity), max_load_(max_load) {}

    // ── Core Operations ───────────────────────────────────────────────────────

    // O(1) average — insert or update key-value pair
    void insert(const K& key, const V& val) {
        // Check if key already exists — update in place
        size_t idx = bucket_idx(key);
        for (auto& entry : buckets_[idx]) {
            if (entry.key == key) {
                entry.value = val;   // update existing
                return;
            }
        }

        // Key not found — insert new entry
        buckets_[idx].push_back({key, val});
        size_++;

        // Rehash if load factor exceeded
        if ((float)size_ / capacity_ > max_load_) {
            rehash();
        }
    }

    // O(1) average — lookup value by key
    V& get(const K& key) {
        size_t idx = bucket_idx(key);
        for (auto& entry : buckets_[idx]) {
            if (entry.key == key) return entry.value;
        }
        throw out_of_range("key not found");
    }

    const V& get(const K& key) const {
        size_t idx = bucket_idx(key);
        for (const auto& entry : buckets_[idx]) {
            if (entry.key == key) return entry.value;
        }
        throw out_of_range("key not found");
    }

    // O(1) average — operator[] (insert with default value if missing)
    V& operator[](const K& key) {
        size_t idx = bucket_idx(key);
        for (auto& entry : buckets_[idx]) {
            if (entry.key == key) return entry.value;
        }
        // Key not found — insert with default-constructed value
        buckets_[idx].push_back({key, V{}});
        size_++;
        if ((float)size_ / capacity_ > max_load_) {
            rehash();
            // After rehash, bucket_idx changes — must find the entry again
            idx = bucket_idx(key);
            for (auto& entry : buckets_[idx]) {
                if (entry.key == key) return entry.value;
            }
        }
        return buckets_[idx].back().value;
    }

    // O(1) average — check if key exists
    bool contains(const K& key) const {
        size_t idx = bucket_idx(key);
        for (const auto& entry : buckets_[idx]) {
            if (entry.key == key) return true;
        }
        return false;
    }

    // O(1) average — remove key-value pair
    bool erase(const K& key) {
        size_t idx = bucket_idx(key);
        auto& chain = buckets_[idx];
        for (auto it = chain.begin(); it != chain.end(); ++it) {
            if (it->key == key) {
                chain.erase(it);
                size_--;
                return true;
            }
        }
        return false;
    }

    // ── Utility ───────────────────────────────────────────────────────────────

    size_t size()      const { return size_; }
    bool   empty()     const { return size_ == 0; }
    size_t bucket_count() const { return capacity_; }
    float  load_factor()  const { return (float)size_ / capacity_; }

    void print() const {
        for (size_t i = 0; i < capacity_; i++) {
            if (buckets_[i].empty()) continue;
            cout << "Bucket " << i << ": ";
            for (const auto& e : buckets_[i]) {
                cout << "[" << e.key << ":" << e.value << "] ";
            }
            cout << "\n";
        }
        cout << "size=" << size_ << " capacity=" << capacity_
             << " load=" << load_factor() << "\n";
    }
};
```

### Implementation 2: Open Addressing with Linear Probing

```cpp
template<typename K, typename V>
class OpenAddressMap {
private:
    enum class State { EMPTY, OCCUPIED, DELETED };  // tombstone for deletion

    struct Slot {
        K     key;
        V     value;
        State state = State::EMPTY;
    };

    vector<Slot> table_;
    size_t       size_;       // number of OCCUPIED slots
    size_t       capacity_;

    size_t probe(const K& key, size_t start) const {
        return (start + 1) % capacity_;   // linear probing step
    }

    size_t hash_idx(const K& key) const {
        return hash<K>{}(key) % capacity_;
    }

    void rehash() {
        vector<Slot> old = move(table_);
        capacity_ *= 2;
        table_.assign(capacity_, Slot{});
        size_ = 0;

        for (auto& slot : old) {
            if (slot.state == State::OCCUPIED) {
                insert(slot.key, slot.value);   // reinsert live entries
            }
            // DELETED tombstones are dropped — cleanup during rehash
        }
    }

public:
    explicit OpenAddressMap(size_t initial_capacity = 16)
        : table_(initial_capacity), size_(0), capacity_(initial_capacity) {}

    void insert(const K& key, const V& val) {
        if ((float)(size_ + 1) / capacity_ > 0.7f) rehash();

        size_t idx = hash_idx(key);
        size_t first_deleted = SIZE_MAX;   // track first tombstone for insertion

        while (table_[idx].state != State::EMPTY) {
            if (table_[idx].state == State::OCCUPIED && table_[idx].key == key) {
                table_[idx].value = val;   // update existing
                return;
            }
            if (table_[idx].state == State::DELETED && first_deleted == SIZE_MAX) {
                first_deleted = idx;       // remember first tombstone
            }
            idx = probe(key, idx);
        }

        // Insert at first tombstone if found, otherwise at the empty slot
        size_t insert_idx = (first_deleted != SIZE_MAX) ? first_deleted : idx;
        table_[insert_idx] = {key, val, State::OCCUPIED};
        size_++;
    }

    bool get(const K& key, V& out) const {
        size_t idx = hash_idx(key);
        while (table_[idx].state != State::EMPTY) {
            if (table_[idx].state == State::OCCUPIED && table_[idx].key == key) {
                out = table_[idx].value;
                return true;
            }
            idx = probe(key, idx);
        }
        return false;
    }

    bool erase(const K& key) {
        size_t idx = hash_idx(key);
        while (table_[idx].state != State::EMPTY) {
            if (table_[idx].state == State::OCCUPIED && table_[idx].key == key) {
                table_[idx].state = State::DELETED;  // tombstone — do NOT decrement size
                size_--;                               // size = occupied count
                return true;
            }
            idx = probe(key, idx);
        }
        return false;
    }

    bool contains(const K& key) const {
        V dummy;
        return get(key, dummy);
    }

    size_t size() const { return size_; }
};
```

### Implementation 3: Custom Hash for Complex Keys

```cpp
// Custom hash for structs/pairs — required for unordered_map with composite keys

// Option 1: specialize std::hash
struct Point { int x, y; };

namespace std {
    template<>
    struct hash<Point> {
        size_t operator()(const Point& p) const {
            // Combine hashes using the classic hash_combine technique
            size_t h1 = hash<int>{}(p.x);
            size_t h2 = hash<int>{}(p.y);
            // hash_combine: XOR with shifted and golden-ratio mixed hash
            return h1 ^ (h2 * 2654435761ULL + 0x9e3779b9 + (h1 << 6) + (h1 >> 2));
        }
    };
    template<>
    struct equal_to<Point> {
        bool operator()(const Point& a, const Point& b) const {
            return a.x == b.x && a.y == b.y;
        }
    };
}

// Usage:
unordered_map<Point, int> grid_values;
grid_values[{1, 2}] = 42;

// Option 2: lambda comparator (for pairs in competitive programming)
auto pairHash = [](const pair<int,int>& p) {
    size_t h1 = hash<int>{}(p.first);
    size_t h2 = hash<int>{}(p.second);
    return h1 ^ (h2 << 32);   // simple combine for 64-bit systems
};
unordered_map<pair<int,int>, int, decltype(pairHash)> pairMap(0, pairHash);

// Option 3: encode composite key as a single integer (competitive programming trick)
// For coordinates (x, y) where both fit in 32 bits:
auto encode = [](int x, int y) -> long long {
    return ((long long)x << 32) | (unsigned int)y;
};
unordered_map<long long, int> fast_map;
fast_map[encode(3, 7)] = 99;
```

---

## 8. Core Operations — Visualised

### Insert with Chaining — Step by Step

```
HashMap with capacity=8, chaining.
Insert: ("apple", 1), ("mango", 2), ("grape", 3), ("kiwi", 4)

h("apple") = 5
h("mango") = 5   ← collision with "apple"!
h("grape") = 2
h("kiwi")  = 5   ← another collision!

After all insertions:

Bucket 0: [ empty ]
Bucket 1: [ empty ]
Bucket 2: [ grape:3 ] → null
Bucket 3: [ empty ]
Bucket 4: [ empty ]
Bucket 5: [ apple:1 ] → [ mango:2 ] → [ kiwi:4 ] → null
Bucket 6: [ empty ]
Bucket 7: [ empty ]

load_factor = 4/8 = 0.5

Lookup "mango":
  idx = h("mango") = 5
  Chain at 5: apple≠mango → mango=found! Return 2.  ← 2 comparisons
```

### The Tombstone Problem in Open Addressing

```
Open addressing, linear probing. capacity=8.
After inserting A(h=2), B(h=2), C(h=2):
  [_][_][A][B][C][_][_][_]   (A at 2, B at 3, C at 4 via probing)

Delete B (mark as DELETED tombstone):
  [_][_][A][☠][C][_][_][_]   (☠ = tombstone)

Lookup C:
  h(C) = 2. Slot 2 has A ≠ C. Slot 3 has TOMBSTONE → must continue probing!
  Slot 4 has C → found! ✓

Why tombstones and not just EMPTY?
  If we mark B as EMPTY instead of DELETED:
  [_][_][A][_][C][_][_][_]

  Lookup C:
  h(C) = 2. Slot 2 has A ≠ C. Slot 3 is EMPTY → STOP. Return "not found". ✗
  WRONG! C exists at slot 4 but we stopped too early.

Tombstones preserve the probe chain integrity.
They are cleaned up during the next rehash.
```

### Rehash — Keys Redistribute

```
Before rehash: capacity=4, entries: {a:1 at 0, b:2 at 1, e:5 at 1 (chained), c:3 at 2}
load = 4/4 = 1.0 → trigger rehash to capacity=8

New indices (h(key) % 8 may differ from h(key) % 4):
  h(a) % 8 = 0  → Bucket 0
  h(b) % 8 = 5  → Bucket 5  (was bucket 1)
  h(e) % 8 = 1  → Bucket 1  (was bucket 1, now has own bucket!)
  h(c) % 8 = 2  → Bucket 2

After rehash (capacity=8):
  [a:1][e:5][c:3][_][_][b:2][_][_]

The chain at old bucket 1 is broken up — b and e now have separate buckets.
Average chain length drops from 1.0 to 0.5. Performance improves.
```

---

## 9. Common Patterns & Techniques

### Pattern 1: Frequency Counting

```cpp
// Count occurrences of each element — the most common hash map pattern
vector<int> nums = {1, 3, 2, 3, 1, 1, 2};
unordered_map<int, int> freq;
for (int x : nums) freq[x]++;
// freq: {1:3, 2:2, 3:2}

// operator[] with default 0 — missing key initialises to 0 then increments
// Equivalent to: freq[x] = freq.count(x) ? freq[x] + 1 : 1;
```

### Pattern 2: Two-Sum / Complement Search

```cpp
// Store seen elements; check if complement exists
vector<int> twoSum(vector<int>& nums, int target) {
    unordered_map<int, int> seen;  // value → index
    for (int i = 0; i < (int)nums.size(); i++) {
        int complement = target - nums[i];
        if (seen.count(complement)) {
            return {seen[complement], i};
        }
        seen[nums[i]] = i;
    }
    return {};
}
```

### Pattern 3: Grouping by Key

```cpp
// Group strings by their sorted form (anagram detection)
vector<vector<string>> groupAnagrams(vector<string>& strs) {
    unordered_map<string, vector<string>> groups;
    for (const string& s : strs) {
        string key = s;
        sort(key.begin(), key.end());  // canonical form = sorted string
        groups[key].push_back(s);
    }
    vector<vector<string>> result;
    for (auto& [key, group] : groups) result.push_back(group);
    return result;
}
```

### Pattern 4: Memoization (DP with Hash Map)

```cpp
// Top-down DP with hash map cache — handles non-integer/sparse keys
unordered_map<string, long long> memo;

long long solve(const string& state) {
    if (memo.count(state)) return memo[state];
    // ... compute result ...
    return memo[state] = result;
}
```

### Pattern 5: Index / Position Tracking

```cpp
// Track first occurrence of each element
unordered_map<int, int> first_seen;  // value → index
for (int i = 0; i < n; i++) {
    if (!first_seen.count(nums[i])) {
        first_seen[nums[i]] = i;
    }
}

// Track running sum's first occurrence (subarray sum problems)
unordered_map<int, int> prefix_sum_idx;
prefix_sum_idx[0] = -1;  // empty prefix has sum 0 at index -1
int running_sum = 0;
for (int i = 0; i < n; i++) {
    running_sum += nums[i];
    if (prefix_sum_idx.count(running_sum - k)) {
        // found subarray with sum k
    }
    if (!prefix_sum_idx.count(running_sum)) {
        prefix_sum_idx[running_sum] = i;
    }
}
```

---

## 10. Interview Problems

### Problem 1: Longest Substring Without Repeating Characters

**Problem:** Given a string `s`, find the length of the longest substring without duplicate characters.

**Example:** `"abcabcbb"` → `3` (substring `"abc"`)

**Thought process:**

> "Sliding window: maintain a window [left, right] where all characters are unique. As right advances, if the new character is already in the window, shrink from the left until it's removed. A hash map tracks the most recent index of each character, allowing O(1) jump of left past the duplicate."

```cpp
// Time: O(n) | Space: O(min(n, |alphabet|))
int lengthOfLongestSubstring(string s) {
    unordered_map<char, int> last_seen;  // char → most recent index
    int left = 0, result = 0;

    for (int right = 0; right < (int)s.size(); right++) {
        char c = s[right];

        // If c was seen and its last occurrence is within our window [left, right]
        if (last_seen.count(c) && last_seen[c] >= left) {
            // Jump left past the previous occurrence of c
            left = last_seen[c] + 1;
        }

        last_seen[c] = right;   // update most recent index
        result = max(result, right - left + 1);
    }
    return result;
}

/*
Trace: s = "abcabcbb"

right=0, c='a': last_seen={}, insert. last_seen={a:0}. window=[0,0], len=1.
right=1, c='b': not in window. last_seen={a:0,b:1}. window=[0,1], len=2.
right=2, c='c': not in window. last_seen={a:0,b:1,c:2}. window=[0,2], len=3.
right=3, c='a': last_seen[a]=0 >= left=0 → jump left to 1.
  last_seen={a:3,b:1,c:2}. window=[1,3], len=3.
right=4, c='b': last_seen[b]=1 >= left=1 → jump left to 2.
  last_seen={a:3,b:4,c:2}. window=[2,4], len=3.
right=5, c='c': last_seen[c]=2 >= left=2 → jump left to 3.
  last_seen={a:3,b:4,c:5}. window=[3,5], len=3.
right=6, c='b': last_seen[b]=4 >= left=3 → jump left to 5.
  last_seen={a:3,b:6,c:5}. window=[5,6], len=2.
right=7, c='b': last_seen[b]=6 >= left=5 → jump left to 7.
  last_seen={a:3,b:7,c:5}. window=[7,7], len=1.

result = 3. ✓

Key: left JUMPS (not slides one step at a time) because we store indices.
     This makes the algorithm genuinely O(n), not O(n×alphabet_size).
*/
```

**Edge cases:**
- Empty string: return 0
- All same characters (`"aaaa"`): window is always length 1, return 1
- All unique characters: window expands to full string, return n
- Unicode/multi-byte: `char` handles ASCII; for full Unicode use `unordered_map<char32_t, int>`

---

### Problem 2: Subarray Sum Equals K

**Problem:** Given an integer array `nums` and integer `k`, return the total number of subarrays whose elements sum to `k`. Array may contain negatives.

**Example:** `nums = [1, 2, 3], k = 3` → `2` (subarrays `[1,2]` and `[3]`)

**Thought process:**

> "Brute force: check all O(n²) subarrays, O(n) sum each → O(n³). With prefix sums: precompute, check all pairs → O(n²). Can we do O(n)?"
>
> "Key insight: `sum(l,r) = prefix[r+1] - prefix[l]`. We want this to equal k. For each r, how many l's satisfy `prefix[l] = prefix[r+1] - k`? If we store a frequency map of all prefix sums seen so far, each query is O(1)."

```cpp
// Time: O(n) | Space: O(n)
int subarraySum(vector<int>& nums, int k) {
    unordered_map<int, int> prefix_count;
    prefix_count[0] = 1;   // empty prefix has sum 0, count = 1

    int running_sum = 0;
    int count = 0;

    for (int num : nums) {
        running_sum += num;

        // How many previous prefix sums equal (running_sum - k)?
        // If prefix[j] = running_sum - k, then sum(j+1..i) = k.
        count += prefix_count[running_sum - k];

        // Record this prefix sum
        prefix_count[running_sum]++;
    }
    return count;
}

/*
nums = [1, 1, 1], k = 2

prefix_count = {0:1}

i=0: running_sum=1. Look for (1-2)=-1 in map → 0. count=0. prefix_count={0:1,1:1}.
i=1: running_sum=2. Look for (2-2)=0 in map → 1. count=1. prefix_count={0:1,1:2,2:1}.
i=2: running_sum=3. Look for (3-2)=1 in map → 2. count=3. prefix_count={0:1,1:2,2:1,3:1}.

Answer: 3. Subarrays: [1,1](idx 0-1), [1,1](idx 1-2), [1,1,1] no wait —
nums[0]+nums[1]=2 ✓, nums[1]+nums[2]=2 ✓, and [1,1,1]=3≠2. So 2 subarrays? Let me recheck.

Actually for k=2 on [1,1,1]:
  [1,1] at idx 0-1: sum=2 ✓
  [1,1] at idx 1-2: sum=2 ✓
  Answer = 2. Let me retrace:

prefix_count = {0:1}
i=0: sum=1. prefix_count[1-2=-1]=0. count=0. Add sum=1. prefix_count={0:1,1:1}.
i=1: sum=2. prefix_count[2-2=0]=1. count=1. Add sum=2. prefix_count={0:1,1:1,2:1}.
i=2: sum=3. prefix_count[3-2=1]=1. count=2. Add sum=3. prefix_count={0:1,1:1,2:1,3:1}.

Answer: 2 ✓.

Critical: prefix_count[0] = 1 initialisation handles the case where the
entire prefix from index 0 to i sums to k.
*/
```

**Why `prefix_count[0] = 1` is essential:**

```
Without it, for nums=[3], k=3:
  running_sum=3. Look for 3-3=0 in map → 0. count=0. WRONG! Answer should be 1.

With it:
  prefix_count={0:1}
  running_sum=3. Look for 0 → 1. count=1. ✓
  The "0" represents the empty prefix before the array starts.
  finding prefix[0]=0 means the subarray from index 0 to current sums to k.
```

**Edge cases:**
- k=0: count subarrays that sum to 0; prefix_count[0]=1 handles the edge where a prefix itself sums to 0
- Negative numbers: the prefix sum approach handles negatives correctly (unlike sliding window)
- All zeros, k=0: every subarray sums to 0; answer is n(n+1)/2

---

### Problem 3: Copy List with Random Pointer

**Problem:** A linked list where each node has `next` and `random` (may point anywhere or null). Create a deep copy.

**Thought process:**

> "The challenge: when copying node A, its random pointer may point to node C which hasn't been created yet. We need a way to map original nodes to their copies."
>
> "Use a hash map: `original_node → copy_node`. First pass: create all copy nodes and populate the map. Second pass: wire up `next` and `random` pointers using the map."

```cpp
struct Node {
    int val;
    Node* next;
    Node* random;
    Node(int v) : val(v), next(nullptr), random(nullptr) {}
};

// Time: O(n) | Space: O(n)
Node* copyRandomList(Node* head) {
    if (!head) return nullptr;

    unordered_map<Node*, Node*> clone;   // original → copy

    // Pass 1: create all copy nodes
    Node* curr = head;
    while (curr) {
        clone[curr] = new Node(curr->val);
        curr = curr->next;
    }

    // Pass 2: wire up next and random using the map
    curr = head;
    while (curr) {
        clone[curr]->next   = clone[curr->next];    // nullptr if curr->next is null
        clone[curr]->random = clone[curr->random];  // nullptr if curr->random is null
        curr = curr->next;
    }

    return clone[head];
}

/*
List: A → B → C → null
      A.random = C, B.random = A, C.random = null

Pass 1: clone = {A: A', B: B', C: C'}

Pass 2:
  curr=A: A'.next = clone[B] = B'. A'.random = clone[C] = C'.
  curr=B: B'.next = clone[C] = C'. B'.random = clone[A] = A'.
  curr=C: C'.next = clone[null] = null. C'.random = clone[null] = null.

Result: A' → B' → C' with correct random pointers. ✓

Note: clone[nullptr] = 0 (default-initialised to nullptr for pointer maps) ✓
      unordered_map returns the default-constructed value for missing keys,
      which for pointer types is nullptr. This elegantly handles null random pointers.
*/
```

**O(1) space alternative — interleaving:**

```cpp
// Space-optimised: O(1) space (excluding output) using node interleaving
Node* copyRandomListO1(Node* head) {
    if (!head) return nullptr;

    // Step 1: Interleave copies A→A'→B→B'→C→C'
    Node* curr = head;
    while (curr) {
        Node* copy = new Node(curr->val);
        copy->next = curr->next;
        curr->next = copy;
        curr = copy->next;
    }

    // Step 2: Set random pointers for copies
    curr = head;
    while (curr) {
        if (curr->random) {
            curr->next->random = curr->random->next;  // copy of random
        }
        curr = curr->next->next;
    }

    // Step 3: Separate the interleaved list into two lists
    Node* dummy = new Node(0);
    Node* copy_curr = dummy;
    curr = head;
    while (curr) {
        copy_curr->next = curr->next;
        curr->next = curr->next->next;
        copy_curr = copy_curr->next;
        curr = curr->next;
    }

    return dummy->next;
}
// The hash map solution is clearer and preferred in interviews unless
// the interviewer specifically asks for O(1) space.
```

---

## 11. Real-World Uses

| Domain | Use Case | Why Hash Map |
|---|---|---|
| **Databases** | Query result caching, index lookup | O(1) lookup for indexed fields |
| **OS kernel** | File descriptor table, page table | O(1) map from fd/virtual page to physical |
| **Compilers** | Symbol table (variable name → type/address) | O(1) identifier resolution during compilation |
| **DNS** | DNS resolver cache | Domain name → IP address, O(1) lookup |
| **CDN / Web** | URL routing, session token storage | O(1) request routing to backend |
| **Distributed systems** | Consistent hashing for key-value stores | Maps keys to server nodes |
| **Language runtimes** | Python dicts, JavaScript objects, Lua tables | Core data model for dynamic languages |
| **Caching** | LRU cache (HashMap + DLL), Redis | O(1) cache hit/miss check |
| **Networking** | ARP table (IP → MAC), routing table | O(1) packet forwarding decisions |
| **Graph algorithms** | Visited set in BFS/DFS, adjacency lookup | O(1) visited check, O(1) neighbour lookup |
| **Machine learning** | Feature hashing, word embeddings lookup | Map token strings to embedding vectors |
| **Blockchain** | Transaction lookup by hash, Merkle nodes | O(1) transaction verification |

**Python's `dict` — the most refined hash map implementation:**

Python's dictionary is the core of the language — objects, namespaces, keyword arguments, and module imports all use dicts internally. CPython's implementation uses a sophisticated open-addressing scheme:

- Initial capacity: 8 slots.
- Growth factor: doubles when 2/3 full (load factor threshold = 0.67).
- Hash randomisation: since Python 3.3, hash seeds are randomised per-process (PYTHONHASHSEED) to prevent hash flooding attacks on web applications.
- Compact dict (Python 3.6+): stores indices into a separate compact array of entries, preserving insertion order while saving memory.
- Ordered by insertion: Python 3.7+ guarantees dict preserves insertion order — implemented by maintaining a separate insertion-order array alongside the hash index.

This means every Python function call (to look up local variable names in `locals()`), every attribute access (`obj.attr`), and every import statement hits a hash table. The performance of Python programs is inseparable from the performance of its hash maps.

---

## 12. Edge Cases & Pitfalls

### Pitfall 1: operator[] Creates Default-Inserted Values

```cpp
unordered_map<string, int> m;

// WRONG: checking existence via operator[] — INSERTS the key!
if (m["missing_key"] == 0) {   // "missing_key" is now in the map with value 0!
    // ...
}
cout << m.size();   // 1, not 0 — "missing_key" was inserted!

// CORRECT: use count() or find()
if (m.count("missing_key")) {
    // key exists
}
if (m.find("missing_key") != m.end()) {
    // key exists
}
// Or in C++20:
if (m.contains("missing_key")) {
    // key exists
}
```

### Pitfall 2: Iterating While Modifying

```cpp
unordered_map<string, int> m = {{"a",1},{"b",2},{"c",3}};

// WRONG: erasing while iterating with range-for
for (auto& [key, val] : m) {
    if (val == 2) m.erase(key);   // iterator invalidated → undefined behaviour
}

// CORRECT: erase returns the next valid iterator
for (auto it = m.begin(); it != m.end(); ) {
    if (it->second == 2) it = m.erase(it);   // erase returns next iterator
    else                  ++it;
}

// Or: collect keys to erase, then erase after
vector<string> to_erase;
for (auto& [key, val] : m) if (val == 2) to_erase.push_back(key);
for (const string& k : to_erase) m.erase(k);
```

### Pitfall 3: Hash Collisions with std::pair as Key

```cpp
// WRONG: std::pair has no default hash in C++ — compile error
unordered_map<pair<int,int>, int> m;   // compile error!
m[{1,2}] = 3;

// CORRECT option 1: custom hash
struct PairHash {
    size_t operator()(const pair<int,int>& p) const {
        return hash<long long>{}(((long long)p.first << 32) | (unsigned)p.second);
    }
};
unordered_map<pair<int,int>, int, PairHash> m;

// CORRECT option 2: encode as single key
unordered_map<long long, int> m;
auto key = [](int x, int y) { return ((long long)x << 32) | (unsigned)y; };
m[key(1,2)] = 3;
```

### Pitfall 4: Floating-Point Keys Are Unreliable

```cpp
// WRONG: floating-point keys suffer from precision issues
unordered_map<double, string> m;
m[0.1 + 0.2] = "a";    // key = 0.30000000000000004
m.count(0.3);           // 0 — different bit pattern! key not found.

// NEVER use float/double as hash map keys in practice.
// Convert to rational (numerator/denominator) or multiply by scale factor and use int.
```

### Pitfall 5: Modifying Keys While in the Map

```cpp
// Hash maps assume keys do not change after insertion.
// For chaining: changing a key changes its hash → it is now in the WRONG bucket.
// For open addressing: similar corruption.

// WRONG: getting a non-const reference to a key and modifying it
// unordered_map keys are const — this is a compile error with structured bindings:
for (auto& [key, val] : m) {
    key = "new_key";   // COMPILE ERROR: key is const
}
// But if you store pointers or objects with mutable state inside the key... be careful.
```

### Pitfall 6: Hash Flooding / DoS Vulnerability

```cpp
// Without hash randomisation, an attacker can craft input that forces
// all keys to hash to the same bucket → O(n) per operation → DoS.

// In competitive programming: if using std::unordered_map and getting TLE
// on adversarial test cases, switch to:
// 1. Custom hash with randomisation:
struct SafeHash {
    static uint64_t splitmix64(uint64_t x) {
        x += 0x9e3779b97f4a7c15;
        x = (x ^ (x >> 30)) * 0xbf58476d1ce4e5b9;
        x = (x ^ (x >> 27)) * 0x94d049bb133111eb;
        return x ^ (x >> 31);
    }
    size_t operator()(uint64_t x) const {
        static const uint64_t FIXED_RANDOM =
            chrono::steady_clock::now().time_since_epoch().count();
        return splitmix64(x + FIXED_RANDOM);
    }
};
unordered_map<int, int, SafeHash> safe_map;

// 2. Use std::map (BST) — guaranteed O(log n) worst case
// 3. Reserve buckets upfront to reduce rehash frequency
```

### Pitfall 7: Forgetting to Handle the Null/Zero Default

```cpp
// unordered_map[key] for a missing int key returns 0 (default-constructed int).
// This causes subtle bugs when 0 is a valid value:

unordered_map<int,int> m;
m[5] = 0;    // intentionally store 0 as value

if (!m[5]) { // evaluates to true — but 5 IS in the map!
    cout << "5 not found";  // wrong!
}

// CORRECT: distinguish "not in map" from "in map with value 0"
if (!m.count(5)) {
    cout << "5 not found";
}
```

---

## 13. std::unordered_map

```cpp
#include <unordered_map>
using namespace std;

// Declaration:
unordered_map<K, V>                           m;  // default hash
unordered_map<K, V, Hash>                     m;  // custom hash
unordered_map<K, V, Hash, KeyEqual>           m;  // custom hash + equality
unordered_map<K, V, Hash, KeyEqual, Allocator> m; // full control

// ── Construction ──────────────────────────────────────────────────────────────
unordered_map<int,int> m1;                          // empty
unordered_map<int,int> m2 = {{1,10},{2,20},{3,30}}; // initialiser list
unordered_map<int,int> m3(m2);                      // copy — O(n)
unordered_map<int,int> m4(move(m2));                // move — O(1)
unordered_map<int,int> m5(16);                      // pre-allocate 16 buckets

// ── Core Operations ───────────────────────────────────────────────────────────
m1[key]              // O(1) avg — returns ref; INSERTS if missing!
m1.at(key)           // O(1) avg — returns ref; throws out_of_range if missing
m1.insert({key,val}) // O(1) avg — inserts if not present; does NOT overwrite
m1.insert_or_assign(key, val)  // O(1) avg — inserts or overwrites (C++17)
m1.emplace(key, val) // O(1) avg — construct in-place
m1.erase(key)        // O(1) avg — returns number erased (0 or 1)
m1.erase(iterator)   // O(1) avg — erase at iterator, returns next iterator
m1.find(key)         // O(1) avg — returns iterator or end()
m1.count(key)        // O(1) avg — returns 0 or 1 (not multimap!)
m1.contains(key)     // O(1) avg — C++20, returns bool
m1.clear()           // O(n)     — removes all elements

// ── Capacity & Bucket Info ────────────────────────────────────────────────────
m1.size()            // number of elements
m1.empty()           // true if size() == 0
m1.bucket_count()    // current number of buckets
m1.load_factor()     // size / bucket_count
m1.max_load_factor() // threshold before rehash (default 1.0)
m1.max_load_factor(0.5)  // set custom threshold
m1.reserve(n)        // ensure can hold n elements without rehash
m1.rehash(n)         // set bucket count to at least n

// ── Iteration (ORDER NOT GUARANTEED) ─────────────────────────────────────────
for (auto& [key, val] : m1) { /* ... */ }          // C++17 structured binding
for (auto it = m1.begin(); it != m1.end(); ++it) {
    cout << it->first << ":" << it->second << "\n";
}

// ── Performance Tuning ────────────────────────────────────────────────────────
// Pre-reserve when you know the approximate final size:
m1.reserve(1000000);   // avoids multiple rehashes during bulk insertion

// Lower max_load_factor for less collision (more memory):
m1.max_load_factor(0.5);   // rehash when 50% full instead of 100%

// ── Related containers ────────────────────────────────────────────────────────
unordered_set<K>          // hash set — only keys, no values
unordered_multimap<K,V>   // allows duplicate keys
unordered_multiset<K>     // allows duplicate keys, no values
map<K,V>                  // ordered (BST), O(log n) operations, keys sorted
set<K>                    // ordered set (BST)
```

---

## 14. Comparison

| Feature | `unordered_map` | `map` (BST) | Array/Vector | Trie |
|---|---|---|---|---|
| Lookup | **O(1) avg** | O(log n) | O(1) if indexed by int | O(m) — key length |
| Insert | **O(1) avg** | O(log n) | O(1) push_back | O(m) |
| Delete | **O(1) avg** | O(log n) | O(n) | O(m) |
| Ordered iteration | ✗ No | ✓ Yes (sorted) | N/A | ✓ Lexicographic |
| Range queries | ✗ No | ✓ Yes (lower/upper_bound) | ✓ With sorting | ✓ Prefix queries |
| Prefix search | ✗ No | Possible but slow | No | ✓ Optimal |
| Memory | Moderate (load factor overhead) | High (3 pointers/node) | Low | High (node per char) |
| Cache | Moderate (chaining = misses) | Poor (pointer chasing) | ★★★ Best | Poor |
| Worst case | O(n) — hash flooding | O(log n) always | O(n) search | O(m) always |
| Key type constraints | Needs hashable key | Needs comparable key (operator<) | Integer keys only | String/sequence keys |
| Guaranteed order | No | Yes | No | Yes (lexicographic) |

**The decision rule:**

```
Default choice for key-value storage: unordered_map
  → O(1) average lookup is almost always what you need.
  → Use when key order does not matter.

Switch to map (BST) when:
  → You need sorted iteration (print all keys in order)
  → You need range queries (all keys between A and B)
  → You need floor/ceiling/predecessor/successor queries
  → Worst-case guarantees matter more than average-case speed
  → Key type is not easily hashable (no std::hash specialisation)

Switch to array when:
  → Keys are integers in a small known range [0, N)
  → Memory is the bottleneck (arrays have zero per-element overhead)

Switch to trie when:
  → Keys are strings and you need prefix matching
  → Autocomplete, spell-check, IP routing lookups
```

---

## 15. Self-Test Questions

1. **Explain how a hash function converts a string key to an array index. What two properties must any hash function satisfy? What is the "avalanche effect" and why is it desirable?**

2. **Trace inserting the keys `["cat", "dog", "fox", "pig"]` with hashes `[3, 6, 3, 3]` into a hash map of capacity 8 using chaining. Draw the final bucket array.**

3. **What is a tombstone in open addressing? Why is it needed? What happens if you mark deleted slots as EMPTY instead?**

4. **Prove that `n` insertions into an empty hash map (starting capacity c, doubling on rehash) has amortised O(1) cost per insertion using the aggregate method.**

5. **Why does `unordered_map<int,int> m; m[5]` insert the key 5 with value 0 into the map? When is this a bug vs intended behaviour? What should you use instead to check existence?**

6. **In the Subarray Sum Equals K problem, why must `prefix_count[0] = 1` be initialised before the loop? Construct a concrete example where omitting it gives a wrong answer.**

7. **A hash map's default max_load_factor is 1.0. If you lower it to 0.5, what is the effect on: (a) average lookup time, (b) memory usage, (c) frequency of rehashing?**

8. **What is hash flooding? How does Python 3.3+ defend against it? What alternative can competitive programmers use when `std::unordered_map` times out on adversarial inputs?**

9. **When would you choose `std::map` over `std::unordered_map`? Give two concrete problems where `map` is necessary and `unordered_map` cannot be directly substituted.**

10. **Design a time-based key-value store: `set(key, value, timestamp)` and `get(key, timestamp)` which returns the most recent value at or before timestamp. What data structure would you use and why? Write the core logic.**

---

## Quick Reference Card

```
Hash Map — O(1) average insert, lookup, delete. The universal key-value store.

Two implementations:
  Chaining (std::unordered_map): each bucket holds a linked list.
                                  O(1+α) average; handles high load gracefully.
  Open addressing:                all entries in one array; best cache.
                                  Must keep load < 0.7.

Hash function must be:
  Deterministic, fast, and uniformly distributed.
  For strings: polynomial rolling hash.
  For integers: multiply-shift or golden-ratio hash.

Load factor = entries / buckets. Rehash (O(n)) when threshold exceeded.
Amortised O(1) per insert — same argument as dynamic array.

std::unordered_map API:
  m[key]          → insert with default value if missing (careful!)
  m.at(key)       → throws if missing
  m.count(key)    → 0 or 1, safe existence check
  m.find(key)     → iterator or end(), safe existence + access
  m.contains(key) → bool (C++20)
  m.insert({k,v}) → does NOT overwrite existing
  m.insert_or_assign(k,v) → always writes (C++17)
  m.erase(key)    → removes; returns count erased
  m.reserve(n)    → pre-allocate for n elements

Interview patterns:
  1. Frequency count: freq[x]++ for each x
  2. Complement search: for each x, check map for (target - x)
  3. Grouping: group elements by canonical key (sorted string for anagrams)
  4. Prefix sum tracking: prefix_count[sum]++ for subarray sum queries
  5. Memoization: cache results of expensive subproblems

Pitfall checklist:
  ✗ m[key] on missing key inserts it — use count() or find() to check
  ✗ Erasing during range-for — use iterator-based erase loop
  ✗ std::pair has no default hash — define custom hash
  ✗ float/double as keys — precision issues make equality unreliable
  ✗ INT_MAX + w overflow in Dijkstra — use long long or guard
  ✗ Hash flooding on adversarial input — add random salt to hash

Default choice rule:
  unordered_map  → when you need O(1) lookup and order doesn't matter
  map (BST)      → when you need sorted order or range queries
  array          → when keys are small integers in [0, N)
  trie           → when keys are strings with prefix queries
```

---

*Previous: [13 — Priority Queue](./13_priority_queue.md)*
*Next: [15 — Hash Set](./15_hash_set.md)*
*See also: [Collision handling deep dive](./hash/collision_handling.md) | [Consistent Hashing](./distributed/consistent_hashing.md) | [Trie](./trees/trie.md)*

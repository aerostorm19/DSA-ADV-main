# Collision Handling: Open Addressing — Linear Probing

> **Curriculum position:** Hash-Based Structures → #3
> **Interview weight:** ★★★☆☆ Medium — Python's dict, Robin Hood, and Swiss Table all descend from this; the clustering analysis is a common deep-dive question
> **Difficulty:** Intermediate — the concept is clean; the deletion problem and clustering mathematics are subtle
> **Prerequisite:** [14 — Hash Map / Hash Table](./14_hash_map.md) | [15 — Collision: Chaining](./15_collision_chaining.md)

---

## Table of Contents

1. [Intuition First — Parking Lot with No Overflow](#1-intuition-first)
2. [Internal Working — The Probe Sequence](#2-internal-working)
3. [The Clustering Problem — Primary Clustering Explained](#3-the-clustering-problem)
4. [Deletion and the Tombstone Solution](#4-deletion-and-the-tombstone-solution)
5. [Deletion Without Tombstones — The Backward Shift Algorithm](#5-deletion-without-tombstones)
6. [Load Factor and the Dramatic Performance Cliff](#6-load-factor-and-the-performance-cliff)
7. [Time & Space Complexity](#7-time--space-complexity)
8. [Complete C++ Implementation](#8-complete-c-implementation)
9. [Core Operations — Visualised](#9-core-operations--visualised)
10. [Robin Hood Hashing — The Linear Probing Evolution](#10-robin-hood-hashing)
11. [Interview Problems](#11-interview-problems)
12. [Real-World Uses](#12-real-world-uses)
13. [Edge Cases & Pitfalls](#13-edge-cases--pitfalls)
14. [Linear Probing vs Other Open Addressing Strategies](#14-comparison)
15. [Self-Test Questions](#15-self-test-questions)

---

## 1. Intuition First

Imagine a parking lot where every spot is numbered. You prefer to park at your assigned spot (your hash). If it is occupied, you try the next spot. Then the next. You continue driving forward until you find an empty space.

When you leave, you do not remove the spot number — you leave a "temporarily vacant" marker that tells the next driver "someone was here, keep checking forward; do not assume nothing is beyond this point."

This is **open addressing with linear probing**. Every key-value pair is stored directly in the main array — no separate linked list, no heap allocation per entry. When the preferred slot is occupied, the probe sequence is:

```
h(k), h(k)+1, h(k)+2, h(k)+3, ...   (all mod capacity)
```

The word "open" means the key is stored in whichever slot it finds open — it is not locked to its hash bucket. The word "linear" means the probe sequence advances by a fixed step of 1 each time.

Why does this matter? Because **cache performance**. All candidate slots are adjacent in memory. When the CPU loads the first slot, it fetches an entire cache line — 64 bytes — which typically contains the next 8-16 slots as well. Subsequent probes that land within that cache line are essentially free. Compare this to chaining, where each chain node is a separate heap allocation at an arbitrary address — each `next` pointer dereference is a potential cache miss.

This cache advantage makes linear probing the fastest hash table strategy on modern hardware for small-to-medium load factors. It is used in Python's dict (prior to 3.6's compact dict redesign, and still in the internal detail), in Robin Hood hashing, in Google's Swiss Table, in Abseil's flat_hash_map, and in most high-performance key-value systems that care about throughput.

The cost: a phenomenon called **primary clustering** — occupied slots tend to merge into long runs, making probe sequences progressively longer. And deletion is non-trivial, requiring either a tombstone mechanism or a backward-shift algorithm. Both are covered in depth below.

---

## 2. Internal Working — The Probe Sequence

### The Array Layout

Unlike chaining, the entire hash table is a single flat array. Each slot holds either:
- A key-value pair (OCCUPIED)
- An empty sentinel (EMPTY)
- A deleted sentinel, or tombstone (DELETED)

```
capacity = 8 slots, 0-indexed:

Slot:  [  0  ][  1  ][  2  ][  3  ][  4  ][  5  ][  6  ][  7  ]
State: [EMPTY][EMPTY][EMPTY][EMPTY][EMPTY][EMPTY][EMPTY][EMPTY]
```

### Insert

```
insert("cat", 10):   h("cat") % 8 = 3. Slot 3 is EMPTY → place here.

Slot:  [  0  ][  1  ][  2  ][ cat:10 ][  4  ][  5  ][  6  ][  7  ]

insert("dog", 20):   h("dog") % 8 = 6. Slot 6 is EMPTY → place here.

Slot:  [  0  ][  1  ][  2  ][ cat:10 ][  4  ][  5  ][ dog:20 ][  7  ]

insert("fox", 25):   h("fox") % 8 = 3. Slot 3 is OCCUPIED by "cat".
  Linear probe: try slot 4. EMPTY → place here.

Slot:  [  0  ][  1  ][  2  ][ cat:10 ][ fox:25 ][  5  ][ dog:20 ][  7  ]

insert("elk", 30):   h("elk") % 8 = 3. Slot 3 is OCCUPIED.
  Probe slot 4: OCCUPIED. Probe slot 5: EMPTY → place here.

Slot:  [  0  ][  1  ][  2  ][ cat:10 ][ fox:25 ][ elk:30 ][ dog:20 ][  7  ]
```

### Lookup

```
lookup("fox"):  h("fox") % 8 = 3.
  Slot 3: "cat" ≠ "fox". Probe forward.
  Slot 4: "fox" = "fox". FOUND. Return 25. ✓

lookup("hen"):  h("hen") % 8 = 3.
  Slot 3: "cat" ≠ "hen". Probe forward.
  Slot 4: "fox" ≠ "hen". Probe forward.
  Slot 5: "elk" ≠ "hen". Probe forward.
  Slot 6: "dog" ≠ "hen". Probe forward.
  Slot 7: EMPTY. STOP. "hen" is NOT in the table. ✓

The EMPTY slot acts as a sentinel: if the key existed, it would have been
placed before this EMPTY slot (or at its hash, then probed forward).
Reaching EMPTY during a probe means the key does not exist.
```

### The Probe Termination Invariant

This invariant is the foundation of correctness:

> **A key k, if it exists in the table, is always found before the first EMPTY slot in the probe sequence starting at h(k).**

This invariant holds because during insertion: if a slot is occupied, we probe forward. We never skip EMPTY slots (that would break the invariant). Therefore, any key in the table is reachable from its hash without passing through an EMPTY slot.

---

## 3. The Clustering Problem — Primary Clustering Explained

### What Is Primary Clustering?

When collisions are resolved by linear probing, occupied slots tend to merge into long **runs** — contiguous sequences of occupied slots. Any new key that hashes anywhere into a run extends the run, regardless of where in the run the collision occurred.

```
Before (sparse): two separate runs of length 2:
  [_][ A ][ B ][_][_][ C ][ D ][_][_][_]
   0   1    2   3  4   5    6   7   8   9

Insert E with h(E) = 1:
  Probe 1: occupied (A). Probe 2: occupied (B). Probe 3: empty → place E at 3.
  [_][ A ][ B ][ E ][_][ C ][ D ][_][_][_]
  Run 1 (A,B,E) grew from 2 to 3.

Insert F with h(F) = 5:
  Probe 5: occupied (C). Probe 6: occupied (D). Probe 7: empty → place F at 7.
  [_][ A ][ B ][ E ][_][ C ][ D ][ F ][_][_]
  Run 2 (C,D,F) grew from 2 to 3.

Insert G with h(G) = 4:
  Probe 4: EMPTY. Place G at 4.
  [_][ A ][ B ][ E ][ G ][ C ][ D ][ F ][_][_]
  G connects run 1 and run 2 into ONE GIANT run of 7!
  [A,B,E,G,C,D,F] — all 7 slots are one cluster.

Now any key hashing to positions 1-7 must probe through up to 7 slots.
Any new insertion into this region makes the cluster even longer.
```

### The Mathematical Consequence

Let α be the load factor. The expected probe length for a successful lookup in a linear probing table is:

```
Successful lookup:   (1/2) × (1 + 1/(1-α))
Unsuccessful lookup: (1/2) × (1 + 1/(1-α)²)

At α=0.5:  successful ≈ 1.5,  unsuccessful ≈ 2.5
At α=0.75: successful ≈ 2.5,  unsuccessful ≈ 8.5
At α=0.9:  successful ≈ 5.5,  unsuccessful ≈ 50.5   ← dramatic degradation
At α=0.99: successful ≈ 50.5, unsuccessful ≈ 5000.5  ← catastrophic!
```

This 1/(1-α)² dependence means performance degrades **quadratically** as α approaches 1. This is the mathematical manifestation of primary clustering.

Compare with chaining, where probe length = 1 + α (linear in α). At α=0.9, chaining expects 1.9 comparisons; linear probing expects 50.5. **Linear probing is catastrophically worse at high load.**

**Practical implication:** Keep α ≤ 0.70 for linear probing. Most implementations use α ≤ 0.70 or α ≤ 0.875 (Robin Hood with variance reduction).

---

## 4. Deletion and the Tombstone Solution

### The Deletion Problem

Simply marking a deleted slot as EMPTY breaks the probe invariant.

```
Table: [_][ cat:10 ][ fox:25 ][ elk:30 ][ dog:20 ][_][_][_]
          h=1        h=1        h=1        h=6

All of cat, fox, elk hash to slot 1 and were placed by probing forward.

Delete "fox" at slot 2 → mark as EMPTY:
  [_][ cat:10 ][  EMPTY  ][ elk:30 ][ dog:20 ][_][_][_]

Lookup "elk": h("elk") = 1.
  Slot 1: "cat" ≠ "elk". Probe forward.
  Slot 2: EMPTY → STOP. Return "not found". ✗ WRONG! elk is at slot 3.

The EMPTY gap at slot 2 breaks the probe chain for "elk".
```

### The Tombstone Solution

Mark deleted slots with a special DELETED sentinel (tombstone, ☠). Probing treats tombstones as OCCUPIED (continue probing), but insertion treats them as EMPTY (can place a new key here).

```
Delete "fox" → mark slot 2 as DELETED:
  [_][ cat:10 ][  ☠  ][ elk:30 ][ dog:20 ][_][_][_]

Lookup "elk": h("elk") = 1.
  Slot 1: "cat" ≠ "elk". Probe forward.
  Slot 2: DELETED → continue (NOT a termination signal).
  Slot 3: "elk" = "elk". FOUND. Return 30. ✓

Insert "new_key" with h=1:
  Slot 1: OCCUPIED. Probe forward.
  Slot 2: DELETED → can place here!
  new_table: [_][ cat:10 ][ new_key:val ][ elk:30 ][ dog:20 ][_][_][_]
```

### The Tombstone Accumulation Problem

Tombstones solve correctness but introduce a subtle long-term degradation:

```
After many inserts and deletes, the table may look like:
  [☠][☠][ a:1 ][☠][☠][ b:2 ][☠][☠][ c:3 ][☠]...

Lookup "a" (h=0): must probe through 2 tombstones before finding it.
Lookup "notexist" (h=0): must probe through the ENTIRE table
  because tombstones never terminate a probe sequence!

Worst case: all slots are either DELETED or the target — O(capacity) probe.

Mitigation:
  1. Rehash when tombstone ratio exceeds a threshold (e.g., 20%).
  2. During rehash, tombstones are dropped — the table is rebuilt cleanly.
  3. Robin Hood hashing reduces tombstone impact through variance reduction.
```

---

## 5. Deletion Without Tombstones — The Backward Shift Algorithm

An elegant alternative: when deleting a slot, shift subsequent entries backward to fill the gap, restoring the invariant without tombstones.

### The Algorithm

```
When deleting slot i:
  1. Mark slot i as EMPTY (the gap).
  2. Scan forward from i+1. For each occupied slot j:
     a. If h(table[j]) would be placed at or before i in a normal probe:
        (i.e., the entry "naturally belongs" at or before i)
        → Move table[j] to slot i, set slot j to EMPTY, set i = j.
        → Continue scanning from j+1 (now with the new gap at i=j).
     b. Otherwise: keep table[j] in place; continue to j+1.
  3. Stop when we hit an EMPTY slot (probe chain cannot extend past EMPTY).
```

### When Does an Entry "Belong" at or Before i?

Entry at slot `j` should move back to fill gap at `i` if its initial home `h` satisfies:

```
The entry at j is "out of place" if it had to probe past position i to get here.
In linear probing (no wrapping complications):

  if (i <= j):   h <= i   OR   h > j   (wrapped around)
  if (i >  j):   h <= i   AND  h > j   (i is after j due to wrap)

Simplified for most cases (no wrap): the entry at j belongs before or at i
  if h(table[j]) % capacity <= i.
```

### Concrete Trace

```
Table: [_][ A ][ B ][ C ][ D ][ E ][_][_]
        0   1    2    3    4    5   6   7

Hash values: h(A)=1, h(B)=1, h(C)=1, h(D)=1, h(E)=5

Delete B at slot 2 (gap = slot 2):

Scan j=3 (C, h=1): h(C)=1 ≤ gap=2? YES (1 ≤ 2). Move C to slot 2.
  [_][ A ][  C  ][ gap ][ D ][ E ][_][_]
  New gap = slot 3.

Scan j=4 (D, h=1): h(D)=1 ≤ gap=3? YES (1 ≤ 3). Move D to slot 3.
  [_][ A ][ C ][ D ][ gap ][ E ][_][_]
  New gap = slot 4.

Scan j=5 (E, h=5): h(E)=5 ≤ gap=4? NO (5 > 4). E belongs AFTER slot 4. Keep in place.

Scan j=6: EMPTY. Stop.

Final: [_][ A ][ C ][ D ][ _ ][ E ][_][_]
        0   1    2    3    4    5   6   7

Verify:
  Lookup A (h=1): slot 1 → found ✓
  Lookup C (h=1): slot 1 (A≠C), slot 2 → found ✓
  Lookup D (h=1): slot 1 (A≠D), slot 2 (C≠D), slot 3 → found ✓
  Lookup E (h=5): slot 5 → found ✓
  Lookup B (h=1): slot 1 (A≠B), slot 2 (C≠B), slot 3 (D≠B), slot 4 (EMPTY) → not found ✓

All correct without any tombstones.
```

**Backward shift vs tombstones — when to use each:**

| Criterion | Tombstones | Backward Shift |
|---|---|---|
| Implementation complexity | Simple | Moderate |
| Memory: deleted slots | Waste space indefinitely | Immediately reclaimed |
| Probe length for existing keys | Unaffected | Slightly shorter (fewer gaps) |
| Probe for unsuccessful lookups | Degrades over time | No degradation |
| Rehash required | Yes, periodically | Less frequently |
| Safe with external iterators | Yes | No — moves entries |
| Works with Robin Hood | Naturally | Requires adjustment |
| Used in | Most implementations | Python's compact dict, Abseil |

---

## 6. Load Factor and the Performance Cliff

### The Knuth Formula — Revisited

The exact expected probe length formulas (Knuth, 1963) are:

```
Linear probing, load factor α = n/m:

E[probes, successful]   = (1/2)(1 + 1/(1-α))
E[probes, unsuccessful] = (1/2)(1 + 1/(1-α)²)

These assume uniform hashing. Real hash functions perform similarly
due to the avalanche effect.
```

### The Performance Cliff — Plotted as Values

```
α     Successful   Unsuccessful   Notes
─────────────────────────────────────────────
0.10    1.06          1.12        Near-perfect — almost no probing
0.25    1.17          1.39        Very fast — 17% overhead
0.50    1.50          2.50        Acceptable for most uses
0.60    1.75          3.56        Still fast
0.70    2.17          5.67        Start to slow — rehash threshold
0.75    2.50          8.50        Java's HashMap threshold (for chaining)
0.80    3.00         13.00        Noticeably slower
0.90    5.50         50.50        ← Performance cliff begins here
0.95   10.50        200.50        ← Serious degradation
0.99   50.50       5000.50        ← Catastrophic — never reach this

Rule: keep α ≤ 0.70 for linear probing. Above 0.75 is risky territory.
```

### Why the Cliff Is Sharp

At α=0.90, almost every probe is obstructed. The expected run length in a 90%-full table is:

```
E[run length] ≈ 1/(1-α)² = 1/(0.1)² = 100

A run of 100 consecutive occupied slots is expected!
Any key hashing into this run must probe through up to 100 slots.
```

This is the mathematical realisation of primary clustering. The 1/(1-α)² term grows without bound as α → 1, making linear probing essentially unusable at very high load.

---

## 7. Time & Space Complexity

### Operation Complexities

| Operation | α ≤ 0.7 (practical) | Worst case | Formula |
|---|---|---|---|
| `insert(k, v)` | **O(1)** | O(n) | E = ½(1 + 1/(1-α)) |
| `lookup(k)` successful | **O(1)** | O(n) | E = ½(1 + 1/(1-α)) |
| `lookup(k)` unsuccessful | **O(1)** | O(n) | E = ½(1 + 1/(1-α)²) |
| `delete(k)` with tombstone | **O(1)** | O(n) | Same as lookup + mark |
| `delete(k)` with shift | **O(1) avg** | O(n) | Shift chain ≤ run length |
| `rehash(2m)` | O(n) | O(n) | Re-insert all entries |
| Iteration | O(m) | O(m) | Must scan all m slots |
| Build from n entries | O(n) avg | O(n²) | n inserts, each O(1) avg |

### Space Complexity

| Component | Space | Notes |
|---|---|---|
| Slot array | O(m) = O(n/α) | Contiguous, one allocation |
| Per-entry overhead | **0 bytes** | No pointers — pure key+value storage |
| Metadata per slot | 1-2 bits for state | Often encoded in key (sentinel value) |
| vs chaining | Chaining: +8 bytes/entry (next ptr) | Open addressing wins on memory |
| Effective memory | n × (key+value) / α | α=0.7 → 43% overhead |

### Cache Performance

```
For int keys (4 bytes) and int values (4 bytes), slot = 8 bytes:
  Cache line = 64 bytes = 8 slots loaded per cache miss.

If probing starts at slot h and the key is at slot h+3:
  All of h, h+1, h+2, h+3 are in the SAME cache line → zero extra misses!

Chaining equivalent: each node at a random heap address → 3 cache misses.

For small types, open addressing with linear probing can be 5-10× faster
than chaining in practice due to this cache advantage.
```

---

## 8. Complete C++ Implementation

### Full Open Addressing Hash Table with Linear Probing

```cpp
#include <iostream>
#include <vector>
#include <optional>
#include <functional>
#include <stdexcept>
#include <string>
using namespace std;

template<typename K, typename V,
         typename Hash  = hash<K>,
         typename Equal = equal_to<K>>
class LinearProbingMap {
private:
    // ── Slot State ────────────────────────────────────────────────────────────
    enum class State : uint8_t { EMPTY, OCCUPIED, DELETED };

    struct Slot {
        K       key;
        V       value;
        State   state = State::EMPTY;
    };

    // ── Data ──────────────────────────────────────────────────────────────────
    vector<Slot>   table_;        // flat array of slots
    size_t         size_;         // number of OCCUPIED entries
    size_t         capacity_;     // total number of slots
    size_t         tombstones_;   // number of DELETED slots
    float          max_load_;     // rehash when (size+tombstones)/capacity > this
    Hash           hasher_;
    Equal          equal_;

    // ── Private Utilities ─────────────────────────────────────────────────────

    size_t home(const K& key) const {
        return hasher_(key) % capacity_;
    }

    // Probe sequence: linear scan with wrap-around
    size_t next_slot(size_t idx) const {
        return (idx + 1) % capacity_;
    }

    // Find the slot index for a key.
    // Returns {slot_index, found}
    // If found: table_[slot_index].state == OCCUPIED and key matches.
    // If not found: slot_index points to the first DELETED or EMPTY slot
    //   that would be the insertion point.
    pair<size_t, bool> find_slot(const K& key) const {
        size_t idx = home(key);
        size_t first_deleted = capacity_;  // sentinel: no deleted seen yet

        size_t probes = 0;
        while (probes < capacity_) {
            const Slot& s = table_[idx];

            if (s.state == State::EMPTY) {
                // Key doesn't exist. Return first tombstone if seen, else this EMPTY slot.
                return {first_deleted < capacity_ ? first_deleted : idx, false};
            }

            if (s.state == State::DELETED) {
                if (first_deleted == capacity_) first_deleted = idx;
            } else {
                // OCCUPIED
                if (equal_(s.key, key)) return {idx, true};  // found!
            }

            idx = next_slot(idx);
            probes++;
        }
        // Table is full of OCCUPIED/DELETED — return first deleted if seen
        return {first_deleted, false};
    }

    void do_rehash(size_t new_cap) {
        vector<Slot> old_table = move(table_);
        table_.assign(new_cap, Slot{});
        capacity_   = new_cap;
        size_       = 0;
        tombstones_ = 0;

        for (auto& slot : old_table) {
            if (slot.state == State::OCCUPIED) {
                insert(move(slot.key), move(slot.value));
            }
            // DELETED slots are simply dropped — tombstones disappear on rehash
        }
    }

    bool needs_rehash() const {
        // Rehash when effective load (size + tombstones) exceeds threshold.
        // Tombstones count because they slow down unsuccessful lookups.
        return (float)(size_ + tombstones_) / capacity_ > max_load_;
    }

public:
    // ── Constructors ──────────────────────────────────────────────────────────

    explicit LinearProbingMap(size_t initial_cap = 16, float max_load = 0.70f)
        : table_(initial_cap, Slot{}), size_(0), capacity_(initial_cap),
          tombstones_(0), max_load_(max_load) {
        // Capacity should be prime or a power of 2.
        // Power of 2: fast modulo via bitwise AND: idx & (capacity-1)
        // Prime: better distribution for multiplicative hash functions
    }

    ~LinearProbingMap() = default;  // vector cleans up automatically

    LinearProbingMap(const LinearProbingMap&)            = delete;
    LinearProbingMap& operator=(const LinearProbingMap&) = delete;

    LinearProbingMap(LinearProbingMap&&)            = default;
    LinearProbingMap& operator=(LinearProbingMap&&) = default;

    // ── Insert / Update ───────────────────────────────────────────────────────

    void insert(const K& key, const V& val) {
        if (needs_rehash()) do_rehash(capacity_ * 2);

        auto [idx, found] = find_slot(key);
        if (found) {
            table_[idx].value = val;  // update existing
        } else {
            if (table_[idx].state == State::DELETED) tombstones_--;
            table_[idx] = {key, val, State::OCCUPIED};
            size_++;
        }
    }

    void insert(K&& key, V&& val) {
        if (needs_rehash()) do_rehash(capacity_ * 2);

        K key_copy = key;  // need a copy for find_slot since key may be moved
        auto [idx, found] = find_slot(key_copy);
        if (found) {
            table_[idx].value = move(val);
        } else {
            if (table_[idx].state == State::DELETED) tombstones_--;
            table_[idx] = {move(key), move(val), State::OCCUPIED};
            size_++;
        }
    }

    // ── operator[] ────────────────────────────────────────────────────────────

    V& operator[](const K& key) {
        if (needs_rehash()) do_rehash(capacity_ * 2);
        auto [idx, found] = find_slot(key);
        if (!found) {
            if (table_[idx].state == State::DELETED) tombstones_--;
            table_[idx] = {key, V{}, State::OCCUPIED};
            size_++;
        }
        return table_[idx].value;
    }

    // ── Lookup ────────────────────────────────────────────────────────────────

    V& at(const K& key) {
        auto [idx, found] = find_slot(key);
        if (!found) throw out_of_range("key not found");
        return table_[idx].value;
    }

    const V& at(const K& key) const {
        auto [idx, found] = find_slot(key);
        if (!found) throw out_of_range("key not found");
        return table_[idx].value;
    }

    optional<V> get(const K& key) const {
        auto [idx, found] = find_slot(key);
        if (!found) return nullopt;
        return table_[idx].value;
    }

    bool contains(const K& key) const {
        auto [idx, found] = find_slot(key);
        return found;
    }

    // ── Deletion — Tombstone Strategy ─────────────────────────────────────────

    bool erase(const K& key) {
        auto [idx, found] = find_slot(key);
        if (!found) return false;
        table_[idx].state = State::DELETED;
        table_[idx].key   = K{};   // release key (especially for strings)
        table_[idx].value = V{};   // release value
        size_--;
        tombstones_++;
        return true;
    }

    // ── Deletion — Backward Shift Strategy (No Tombstones) ───────────────────

    bool erase_shift(const K& key) {
        auto [idx, found] = find_slot(key);
        if (!found) return false;

        // Mark gap at idx, then shift subsequent entries backward
        table_[idx].state = State::EMPTY;
        size_--;

        size_t gap = idx;
        size_t j   = next_slot(gap);

        while (table_[j].state == State::OCCUPIED) {
            size_t h = home(table_[j].key);

            // Check if table_[j] "belongs" at or before the gap position.
            // An entry at slot j with home h should move back to fill gap i if:
            //   the entry's natural probe path would have gone through position i.
            //   In circular terms: is gap between h and j (inclusive)?
            bool should_move;
            if (gap <= j) {
                // Normal (no wrap): move if h is in [0..gap] — i.e., h ≤ gap OR h > j
                should_move = (h <= gap) || (h > j);
            } else {
                // Wrapped: gap > j. Move if h is in (j..gap]
                should_move = (h > j) && (h <= gap);
            }

            if (should_move) {
                table_[gap] = move(table_[j]);
                table_[j].state = State::EMPTY;
                gap = j;
            }
            j = next_slot(j);
        }
        return true;
    }

    // ── Utility ───────────────────────────────────────────────────────────────

    size_t size()         const { return size_; }
    bool   empty()        const { return size_ == 0; }
    size_t capacity()     const { return capacity_; }
    float  load_factor()  const { return (float)size_ / capacity_; }
    size_t tombstone_count() const { return tombstones_; }

    void clear() {
        table_.assign(capacity_, Slot{});
        size_ = tombstones_ = 0;
    }

    void reserve(size_t n) {
        size_t required = (size_t)ceil((float)n / max_load_);
        if (required > capacity_) do_rehash(required);
    }

    // ── Diagnostics ───────────────────────────────────────────────────────────

    void print() const {
        for (size_t i = 0; i < capacity_; i++) {
            cout << "[" << i << "] ";
            switch (table_[i].state) {
                case State::EMPTY:    cout << "EMPTY\n"; break;
                case State::DELETED:  cout << "DELETED (tombstone)\n"; break;
                case State::OCCUPIED: cout << table_[i].key << ":" << table_[i].value << "\n"; break;
            }
        }
        cout << "size=" << size_ << " tombstones=" << tombstones_
             << " capacity=" << capacity_
             << " load=" << load_factor() << "\n";
    }

    // Probe length distribution — useful for debugging hash quality
    void probe_stats() const {
        size_t total_probes = 0, max_probe = 0, count = 0;
        for (size_t i = 0; i < capacity_; i++) {
            if (table_[i].state != State::OCCUPIED) continue;
            size_t h = home(table_[i].key);
            size_t probe_len = (i >= h) ? i - h : capacity_ - h + i;  // circular distance
            total_probes += probe_len + 1;  // +1 for the slot itself
            max_probe = max(max_probe, probe_len + 1);
            count++;
        }
        if (count == 0) return;
        cout << "avg_probe=" << (float)total_probes/count
             << " max_probe=" << max_probe << "\n";
    }

    // ── Iteration ─────────────────────────────────────────────────────────────

    class Iterator {
        const LinearProbingMap* map_;
        size_t                  idx_;

        void advance() {
            while (idx_ < map_->capacity_ &&
                   map_->table_[idx_].state != State::OCCUPIED) {
                idx_++;
            }
        }
    public:
        Iterator(const LinearProbingMap* m, size_t i) : map_(m), idx_(i) { advance(); }
        pair<const K&, const V&> operator*() const {
            return {map_->table_[idx_].key, map_->table_[idx_].value};
        }
        Iterator& operator++() { idx_++; advance(); return *this; }
        bool operator!=(const Iterator& o) const { return idx_ != o.idx_; }
    };

    Iterator begin() const { return {this, 0}; }
    Iterator end()   const { return {this, capacity_}; }
};
```

### Power-of-Two Capacity Optimisation

```cpp
// When capacity is a power of 2, modulo becomes a fast bitwise AND:
// idx % capacity == idx & (capacity - 1)   (when capacity is a power of 2)

// This eliminates an integer division (typically 3-5 cycles) per probe.
// At millions of hash table operations per second, this matters.

size_t next_slot_fast(size_t idx) const {
    return (idx + 1) & (capacity_ - 1);   // capacity must be power of 2
}

size_t home_fast(const K& key) const {
    return hasher_(key) & (capacity_ - 1);   // capacity must be power of 2
}

// Trade-off: power-of-2 capacity can amplify patterns in poor hash functions.
// A hash that returns only even numbers would use only half the buckets
// with power-of-2 capacity (all even numbers ≡ 0 or 2 or 4... mod 2^k).
// Prime capacity avoids this: any non-zero hash uses all buckets.
// Production compromise: use a well-mixed hash (MurmurHash, xxHash) with
// power-of-2 capacity — both fast division and good distribution.
```

---

## 9. Core Operations — Visualised

### Insert with Probe Sequence

```
capacity=10, load=0 initially.
Insert keys with their hash values:

h("apple")=3, h("mango")=3, h("grape")=7, h("kiwi")=3, h("plum")=7

insert("apple"):
  [_][_][_][ apple ][_][_][_][_][_][_]   slot 3, 0 probes

insert("mango"):  h=3, slot 3 occupied.
  Probe 4: empty.
  [_][_][_][ apple ][ mango ][_][_][_][_][_]   slot 4, 1 probe

insert("grape"):  h=7, slot 7 empty.
  [_][_][_][ apple ][ mango ][_][_][ grape ][_][_]   slot 7, 0 probes

insert("kiwi"):   h=3, slot 3 (apple), slot 4 (mango), slot 5 empty.
  [_][_][_][ apple ][ mango ][ kiwi ][_][ grape ][_][_]   slot 5, 2 probes

insert("plum"):   h=7, slot 7 (grape), slot 8 empty.
  [_][_][_][ apple ][ mango ][ kiwi ][_][ grape ][ plum ][_]   slot 8, 1 probe

Final state:
  Slot 0: EMPTY
  Slot 1: EMPTY
  Slot 2: EMPTY
  Slot 3: apple (home=3, probe_len=0)
  Slot 4: mango (home=3, probe_len=1)   ← had to probe 1 past home
  Slot 5: kiwi  (home=3, probe_len=2)   ← had to probe 2 past home
  Slot 6: EMPTY
  Slot 7: grape (home=7, probe_len=0)
  Slot 8: plum  (home=7, probe_len=1)   ← had to probe 1 past home
  Slot 9: EMPTY

Cluster 1 (slots 3-5): apple, mango, kiwi — all hash to slot 3
Cluster 2 (slots 7-8): grape, plum — all hash to slot 7

Now insert "lime" with h=4:
  Slot 4 is occupied (mango). Probe 5: occupied (kiwi). Probe 6: EMPTY.
  lime goes to slot 6 — extending cluster 1 even though lime's home is 4!

  [_][_][_][ apple ][ mango ][ kiwi ][ lime ][ grape ][ plum ][_]

  Both clusters are now merged into ONE cluster: slots 3-8!
  This is primary clustering: lime's collision at 4 grew the cluster 3-6.
```

### Tombstone Behaviour — Correct vs Incorrect Deletion

```
After the above, delete "mango" at slot 4.

WITH TOMBSTONES (correct):
  [_][_][_][ apple ][ ☠ ][ kiwi ][ lime ][ grape ][ plum ][_]

  Lookup "kiwi" (h=3):
    Slot 3: apple≠kiwi. Slot 4: TOMBSTONE → continue. Slot 5: kiwi → FOUND ✓
  Lookup "lime" (h=4):
    Slot 4: TOMBSTONE → continue. Slot 5: kiwi≠lime. Slot 6: lime → FOUND ✓

WITHOUT TOMBSTONES (incorrect — EMPTY instead of DELETED):
  [_][_][_][ apple ][ _ ][ kiwi ][ lime ][ grape ][ plum ][_]

  Lookup "kiwi" (h=3):
    Slot 3: apple≠kiwi. Slot 4: EMPTY → STOP. Return "not found". ✗ WRONG!
```

### Backward Shift Deletion — No Tombstones

```
Table: [_][_][_][ apple:1 ][ mango:2 ][ kiwi:3 ][ lime:4 ][ grape:5 ][ plum:6 ][_]
Homes: h(apple)=3,h(mango)=3,h(kiwi)=3,h(lime)=4,h(grape)=7,h(plum)=7

Delete "mango" at slot 4. Gap = slot 4.

Scan forward from slot 5:
  j=5: kiwi, h=3. Should move to fill gap at 4?
    gap(4) ≤ j(5), so check: h(3) ≤ gap(4)? YES (3 ≤ 4). Move kiwi to 4.
    [_][_][_][ apple ][ kiwi ][  gap  ][ lime ][ grape ][ plum ][_]
    New gap = slot 5.

  j=6: lime, h=4. Should move to fill gap at 5?
    h(4) ≤ gap(5)? YES (4 ≤ 5). Move lime to 5.
    [_][_][_][ apple ][ kiwi ][ lime ][  gap  ][ grape ][ plum ][_]
    New gap = slot 6.

  j=7: grape, h=7. Should move to fill gap at 6?
    h(7) ≤ gap(6)? NO (7 > 6). grape stays.

  j=8: plum, h=7. Should move to fill gap at 6?
    h(7) ≤ gap(6)? NO (7 > 6). plum stays.

  j=9: EMPTY → stop.

Final: [_][_][_][ apple ][ kiwi ][ lime ][_][ grape ][ plum ][_]

Verify all entries still findable:
  apple (h=3): slot 3 ✓ (no probe needed)
  kiwi  (h=3): slot 3 (apple≠kiwi), slot 4 (kiwi) ✓
  lime  (h=4): slot 4 (kiwi≠lime), slot 5 (lime) ✓
  grape (h=7): slot 7 ✓
  plum  (h=7): slot 7 (grape≠plum), slot 8 (plum) ✓
  mango (h=3): slot 3,4,5,6 — slot 6 is EMPTY → not found ✓

Perfect: no tombstones, all correct.
```

---

## 10. Robin Hood Hashing — The Linear Probing Evolution

Robin Hood hashing is linear probing with one twist: **take from the rich (short probe) and give to the poor (long probe)**. When inserting a new key and encountering an existing key with a *shorter* probe distance (DIB — Distance from Initial Bucket), steal the slot and continue inserting the displaced key.

```
DIB (distance from initial bucket) = current_slot - home_slot (mod capacity)

Example: capacity=10

Table: [_][ A(h=1,DIB=0) ][ B(h=1,DIB=1) ][ C(h=1,DIB=2) ][_]...

Insert D with h=2 (DIB=0 at slot 2):
  Slot 2: B has DIB=1. D has DIB=0. D's DIB < B's DIB → keep B (it's "poorer").
  Probe slot 3: C has DIB=2. D has DIB=1. D's DIB < C's DIB → keep C.
  Probe slot 4: EMPTY. Place D at slot 4 (DIB=2).

Standard linear probing: D probes 2 slots.
Result: [_][A(D=0)][B(D=1)][C(D=2)][D(D=2)][_]... — D has long probe distance.

NOW insert E with h=2 (DIB=0 at slot 2):
  Slot 2: B has DIB=1. E has DIB=0. E's DIB(0) < B's DIB(1) → STEAL slot 2.
    Place E at slot 2. Continue inserting displaced B.
  Slot 3: C has DIB=2. B (now displaced) has DIB=1. B's DIB < C's DIB → keep C.
  Slot 4: D has DIB=2. B has DIB=2. Equal → keep D (first come, first served).
  Slot 5: EMPTY. Place B at slot 5 (DIB=3).

Result: [_][A(D=0)][E(D=0)][C(D=2)][D(D=2)][B(D=3)][_]

Why Robin Hood is better:
  Standard LP: max DIB can be arbitrarily large.
  Robin Hood:  max DIB is much more tightly bounded.
  The variance of probe lengths is reduced — no single key gets
  catastrophically unlucky. Average case is similar, worst case much better.
```

### Robin Hood in Practice

```cpp
void insert_robin_hood(const K& key, const V& val) {
    K   cur_key = key;
    V   cur_val = val;
    size_t idx  = home(cur_key);
    size_t dib  = 0;   // distance from initial bucket for the key being inserted

    for (size_t probe = 0; probe < capacity_; probe++, idx = next_slot(idx), dib++) {
        if (table_[idx].state == State::EMPTY || table_[idx].state == State::DELETED) {
            table_[idx] = {move(cur_key), move(cur_val), State::OCCUPIED};
            table_[idx].dib = dib;
            size_++;
            return;
        }

        // Robin Hood: if current occupant has shorter probe distance, steal its slot
        if (table_[idx].dib < dib) {
            swap(cur_key,           table_[idx].key);
            swap(cur_val,           table_[idx].value);
            swap(dib,               table_[idx].dib);
            // Now continue inserting the displaced element
        }
    }
}
// Robin Hood lookup: can terminate early if probe length exceeds table's max DIB.
// Robin Hood deletion: backward shift (no tombstones needed — DIBs guide the shift).
```

---

## 11. Interview Problems

### Problem 1: Design HashMap — Open Addressing Version

**Problem:** Same as the chaining version — implement `put`, `get`, `remove`. But implement it with open addressing and linear probing.

**This directly tests:** probe sequence, tombstone semantics, the termination invariant, and correct predecessor-free deletion.

```cpp
class MyHashMap {
private:
    static const int CAP = 10007;  // prime capacity
    static const int EMPTY = -1;
    static const int DELETED = -2;

    int keys_[CAP];
    int vals_[CAP];

public:
    MyHashMap() {
        fill(keys_, keys_ + CAP, EMPTY);
        fill(vals_, vals_ + CAP, 0);
    }

    int hash(int key) { return key % CAP; }

    void put(int key, int value) {
        int idx = hash(key);
        int first_del = -1;
        for (int i = 0; i < CAP; i++) {
            int slot = (idx + i) % CAP;
            if (keys_[slot] == key) {
                vals_[slot] = value; return;  // update
            }
            if (keys_[slot] == DELETED && first_del == -1) first_del = slot;
            if (keys_[slot] == EMPTY) {
                // Place at first tombstone if seen, else at this EMPTY slot
                int target = (first_del != -1) ? first_del : slot;
                keys_[target] = key;
                vals_[target] = value;
                return;
            }
        }
        // Table full of tombstones — should not happen with proper load management
        if (first_del != -1) { keys_[first_del] = key; vals_[first_del] = value; }
    }

    int get(int key) {
        int idx = hash(key);
        for (int i = 0; i < CAP; i++) {
            int slot = (idx + i) % CAP;
            if (keys_[slot] == key) return vals_[slot];
            if (keys_[slot] == EMPTY) return -1;
            // DELETED: continue probing
        }
        return -1;
    }

    void remove(int key) {
        int idx = hash(key);
        for (int i = 0; i < CAP; i++) {
            int slot = (idx + i) % CAP;
            if (keys_[slot] == key) {
                keys_[slot] = DELETED;  // tombstone
                return;
            }
            if (keys_[slot] == EMPTY) return;  // not found
        }
    }
};
```

**Key discussion points:**
- Why is the capacity prime? To reduce patterns in the modulo hash
- What happens if `remove` uses EMPTY instead of DELETED? Breaks subsequent lookups
- When would you rehash? When tombstone ratio becomes high (e.g., 20%)
- How would you handle arbitrary key types? Template + std::hash + std::equal_to

---

### Problem 2: Find All Duplicates in an Array Using Array as Hash Table

**Problem:** Given an integer array `nums` of length n where `nums[i] ∈ [1, n]`, find all elements that appear twice. Must run in O(n) time and O(1) extra space.

**This is the "array as open addressing hash table" pattern** — the array itself is the hash table, with `h(x) = x - 1` as the perfect hash function (no collisions because values are in [1,n]).

```cpp
// Time: O(n) | Space: O(1) — uses the input array itself as the hash table
// The "marking" technique: negate arr[abs(val)-1] to record seeing val.
vector<int> findDuplicates(vector<int>& nums) {
    vector<int> result;
    int n = nums.size();

    for (int i = 0; i < n; i++) {
        int idx = abs(nums[i]) - 1;   // h(val) = val - 1 (0-indexed)

        if (nums[idx] < 0) {
            // Already negated → val has been seen before → duplicate!
            result.push_back(abs(nums[i]));
        } else {
            // First time seeing val → mark it by negating the slot
            nums[idx] = -nums[idx];
        }
    }

    // Restore array (optional — restore if modification is not allowed)
    // for (int& x : nums) x = abs(x);

    return result;
}

/*
Trace: nums = [4, 3, 2, 7, 8, 2, 3, 1]

i=0: val=4, idx=3. nums[3]=7>0 → mark: nums[3]=-7. nums=[4,3,2,-7,8,2,3,1]
i=1: val=3, idx=2. nums[2]=2>0 → mark: nums[2]=-2. nums=[4,3,-2,-7,8,2,3,1]
i=2: val=2, idx=1. nums[1]=3>0 → mark: nums[1]=-3. nums=[4,-3,-2,-7,8,2,3,1]
i=3: val=7 (abs(-7)=7). idx=6. nums[6]=3>0 → mark: nums[6]=-3.
i=4: val=8, idx=7. nums[7]=1>0 → mark: nums[7]=-1.
i=5: val=2, idx=1. nums[1]=-3<0 → DUPLICATE! result=[2].
i=6: val=3 (abs(-3)=3). idx=2. nums[2]=-2<0 → DUPLICATE! result=[2,3].
i=7: val=1, idx=0. nums[0]=4>0 → mark: nums[0]=-4.

result = [2, 3] ✓

The array-as-hash-table pattern:
  Hash function: h(val) = val - 1 (maps [1..n] to [0..n-1])
  No collisions possible: each value has a unique slot
  Marking: use sign bit to encode "seen" state (negative = seen)
  This is open addressing with perfect hashing — zero probing needed.
*/
```

---

### Problem 3: Longest Consecutive Sequence

**Problem:** Given an unsorted array of integers, find the length of the longest consecutive elements sequence. Must run in O(n) time.

**Example:** `[100, 4, 200, 1, 3, 2]` → `4` (sequence `1,2,3,4`)

**Why the hash set (open addressing) is essential:**

```cpp
// Time: O(n) — each number is processed at most twice (once as start, once as continuation)
// Space: O(n) — hash set for O(1) membership test
int longestConsecutive(vector<int>& nums) {
    unordered_set<int> num_set(nums.begin(), nums.end());
    int longest = 0;

    for (int num : num_set) {
        // Only start counting from the BEGINNING of a sequence.
        // A number is a sequence start if (num-1) is NOT in the set.
        if (!num_set.count(num - 1)) {
            int cur = num;
            int length = 1;

            while (num_set.count(cur + 1)) {
                cur++;
                length++;
            }
            longest = max(longest, length);
        }
    }
    return longest;
}

/*
Trace: [100, 4, 200, 1, 3, 2]
num_set = {100, 4, 200, 1, 3, 2}

num=100: is 99 in set? No → start sequence from 100.
  101 in set? No. Length=1.
num=4: is 3 in set? YES → not a sequence start. Skip.
num=200: is 199 in set? No → start from 200.
  201 in set? No. Length=1.
num=1: is 0 in set? No → start sequence from 1.
  2 in set? YES (length=2). 3? YES (length=3). 4? YES (length=4). 5? No.
  Length=4.
num=3: is 2 in set? YES → skip.
num=2: is 1 in set? YES → skip.

longest = max(1, 1, 4) = 4 ✓

Why O(n) and not O(n²)?
  Each number is a sequence start at most once (the num-1 check).
  Each number is visited in a while loop at most once (as a continuation).
  Total iterations: n (outer for) + n (inner while, across all sequences) = 2n = O(n).
  The hash set's O(1) lookup makes both the start-check and continuation-check cheap.
*/
```

---

## 12. Real-World Uses

| Domain | Implementation | Linear Probing Detail |
|---|---|---|
| **CPython dict** (3.6+) | Compact dict with open addressing | Indices into a compact entries array; probing via `i = (5*i + 1 + perturb) >> 5` |
| **Google Abseil `flat_hash_map`** | Robin Hood + SSE2/NEON | Groups 16 slots; SIMD checks 16 slots in one instruction |
| **Rust `HashMap`** | Robin Hood hashing | Linear probing with Robin Hood displacement; backward shift deletion |
| **Java `IdentityHashMap`** | Linear probing | Special case using `==` (reference equality) not `.equals()` |
| **Lua tables** | Open addressing | Both array part and hash part; linear probing in hash part |
| **SQLite** | Linear probing for in-memory hash joins | Hash table within B-tree pages for fast joins |
| **LLVM** | `DenseMap<K,V>` | Linear probing with tombstones; cache-aligned slot arrays |
| **Redis Cluster** | Slot allocation table | Hash ring with linear probe for cluster node assignment |

**CPython's dict — a masterclass in open addressing:**

Python's dict uses a complex probe sequence rather than pure linear probing:

```python
# Python's actual probe sequence (simplified from CPython/Objects/dictobject.c):
def probe(initial_hash, perturb, capacity):
    i = initial_hash & (capacity - 1)   # initial slot
    while True:
        yield i
        # Perturbation: mixes in upper bits of the hash to reduce clustering
        i = (5 * i + 1 + perturb) % capacity
        perturb >>= 5   # perturb decays toward 0 over time
        # As perturb → 0, the sequence degenerates to (5i+1) % capacity
        # which visits all slots when capacity is a power of 2 (proved by number theory)
```

The `perturb` variable mixes in upper bits of the hash to reduce clustering in the first few probes. This is a hybrid between linear probing (fast cache behaviour) and pseudo-random probing (better distribution). After a few probes, `perturb` decays to 0 and the sequence becomes the purely arithmetic `(5i+1) % m`.

**Google Abseil's Swiss Table — linear probing at CPU instruction speed:**

```
Swiss Table layout: groups of 16 slots

For each group of 16 consecutive slots:
  - 16 bytes of metadata (one byte per slot): either 0xFF (empty), 0xFE (tombstone),
    or the lower 7 bits of the slot's hash.
  - 16 key-value pairs.

Lookup:
  1. Compute h(key). Split into h1 (upper bits) and h2 (lower 7 bits).
  2. Find the group at h1 % num_groups.
  3. Use SSE2 _mm_cmpeq_epi8 to compare h2 against all 16 metadata bytes SIMULTANEOUSLY.
     This single instruction checks 16 slots in parallel.
  4. Any match: check the full key for equality (eliminate false positives from h2 collision).
  5. If no match in this group: move to the next group (linear probe at the group level).

Result: each 16-slot group check costs ONE SSE2 instruction instead of 16 comparisons.
At 16 slots per SIMD instruction, the throughput is roughly 16× that of scalar comparison.
This is the state of the art in hash table design as of 2024.
```

---

## 13. Edge Cases & Pitfalls

### Pitfall 1: Treating DELETED as EMPTY During Lookup

```cpp
// WRONG: stopping at tombstones during lookup
while (table_[idx].state != State::EMPTY &&
       table_[idx].state != State::DELETED) {   // ← stops at tombstone!
    if (equal_(table_[idx].key, key)) return table_[idx].value;
    idx = next_slot(idx);
}
return null;  // Returns null when key might exist AFTER the tombstone

// CORRECT: only stop at EMPTY during lookup
while (table_[idx].state != State::EMPTY) {
    if (table_[idx].state == State::OCCUPIED && equal_(table_[idx].key, key)) {
        return table_[idx].value;
    }
    idx = next_slot(idx);  // continue past tombstones
}
return null;  // EMPTY slot found before the key → key doesn't exist
```

### Pitfall 2: Infinite Loop When Table Is Full of Tombstones

```cpp
// If the table is entirely DELETED tombstones and no EMPTY slots:
// Lookup will cycle forever (never hits EMPTY, never finds the key).

// Prevention: include tombstones in the rehash trigger:
bool needs_rehash() const {
    // Rehash when EITHER occupancy is high OR tombstone ratio is high
    return (float)(size_ + tombstones_) / capacity_ > max_load_;
}
// This ensures tombstones don't accumulate to dangerous levels.

// Alternatively: bound the probe length to capacity:
for (size_t probe = 0; probe < capacity_; probe++) {
    // ... probe logic ...
}
return false;  // Key not found after probing all slots
```

### Pitfall 3: Off-by-One in Modular Wrap-Around

```cpp
// WRONG: non-modular advance assumes linear layout
size_t next_slot(size_t idx) const {
    return idx + 1;   // overflows at the end of the array!
}
// Accessing table_[capacity_] is out-of-bounds → undefined behaviour.

// CORRECT: always use modulo (or bitwise AND for power-of-2 capacity)
size_t next_slot(size_t idx) const {
    return (idx + 1) % capacity_;
}
// Or for power-of-2 capacity (faster):
size_t next_slot(size_t idx) const {
    return (idx + 1) & (capacity_ - 1);
}
```

### Pitfall 4: Backward Shift — Wrong "Should Move" Condition

```cpp
// The condition for moving entry at slot j to fill gap at i:
// The entry's home h must be "between" i and j in the circular sense.
// An entry should NOT move if its home is AFTER the gap (it probed from BEFORE the gap).

// WRONG (only handles non-wrapped case):
bool should_move = (home(table_[j].key) <= gap);
// Fails when gap > j (the gap has wrapped around).

// CORRECT (handles both wrapped and non-wrapped):
size_t h = home(table_[j].key);
bool should_move;
if (gap <= j) {
    should_move = (h <= gap) || (h > j);
} else {
    should_move = (h <= gap) && (h > j);
}
```

### Pitfall 5: Load Factor Computed with Size Only (Ignoring Tombstones)

```cpp
// WRONG: only counting occupied slots
float load_factor() { return (float)size_ / capacity_; }
// If 5 slots occupied and 45 slots are tombstones in capacity=50:
// load_factor() = 5/50 = 0.10 — looks healthy!
// But actually 50/50 = 1.0 of slots are non-empty (size + tombstones = 50).
// Lookups for absent keys will probe ALL 50 slots before finding EMPTY.

// CORRECT: count tombstones toward the load for rehash decisions
bool needs_rehash() const {
    return (float)(size_ + tombstones_) / capacity_ > max_load_;
}
// Occupied load for performance tracking:
float true_load() { return (float)size_ / capacity_; }
// Effective load for rehash decisions:
float effective_load() { return (float)(size_ + tombstones_) / capacity_; }
```

### Pitfall 6: Not Releasing Memory in Deleted Slots

```cpp
// After erasing a slot, the key and value objects remain constructed
// (holding their allocated memory) until the slot is reused or the table is destroyed.

// For types like std::string that hold heap memory:
bool erase(const K& key) {
    auto [idx, found] = find_slot(key);
    if (!found) return false;
    table_[idx].state = State::DELETED;
    // WRONG: key and value still hold their strings
    // They will not be freed until the slot is overwritten or the table is destroyed.

    // CORRECT: explicitly destroy the key and value
    table_[idx].key   = K{};   // default construct (empty string) — releases memory
    table_[idx].value = V{};   // default construct — releases memory
    table_[idx].state = State::DELETED;
    return true;
}
```

### Pitfall 7: Primary Clustering from a Bad Hash Function

```cpp
// If hash function returns only multiples of capacity/p for some p:
// All keys land in a fraction of the buckets → severe clustering.

// Example: h(k) = k * 2 with capacity = 8
// h(0)=0, h(1)=2, h(2)=4, h(3)=6, h(4)=0, h(5)=2, ...
// Only even slots 0,2,4,6 are ever used. Odd slots always empty.
// Effective utilisation: 50%. Expected probe length at load 0.5: as if load is 1.0.

// Fix: use a hash function that avalanches well (MurmurHash, xxHash, FNV-1a)
// or salt the hash with a random seed to break deterministic patterns.
```

---

## 14. Comparison — Linear Probing vs Other Strategies

| Feature | Linear Probing | Quadratic Probing | Double Hashing | Robin Hood | Chaining |
|---|---|---|---|---|---|
| Probe sequence | `h, h+1, h+2, ...` | `h, h+1, h+4, h+9,...` | `h, h+h2, h+2h2,...` | `h, h+1,...` + swap | linked list |
| Primary clustering | ✗ Severe | ✓ Reduced | ✓ Eliminated | ✓ Reduced | N/A |
| Secondary clustering | N/A | ✗ Yes | ✓ Eliminated | ✓ Reduced | N/A |
| Cache performance | ★★★ Best | ★★ Good | ★ Poor (jumps) | ★★★ Best | ★ Poor |
| Memory overhead | ✓ 0 per entry | ✓ 0 per entry | ✓ 0 per entry | ✓ 0 per entry | ✗ 8B per entry |
| Deletion complexity | Tombstone/shift | Tombstone | Tombstone | Backward shift | Simple unlink |
| Max useful load | 0.70 | 0.70 | 0.80 | 0.875 | > 1.0 |
| Worst case lookup | O(n) | O(n) | O(n) | O(log n) est. | O(n) |
| Implementation complexity | Low | Low | Moderate | Moderate | Low |
| Used in | Python dict, LLVM | Some academic | Classic textbooks | Rust, Abseil | std::unordered_map |

**The evolution of linear probing:**

```
Pure linear probing (1953):
  Simple. Fast. Severe clustering at high load.

Quadratic probing:
  Reduces primary clustering. Secondary clustering remains.
  Cannot guarantee visiting all slots unless capacity is prime.

Double hashing:
  Eliminates both types of clustering.
  Two hash functions: more computation, poor cache (unpredictable jumps).

Robin Hood (1985, Celis):
  Keeps linear probing's cache advantage.
  Reduces probe length variance by balancing DIBs.
  Backward shift deletion (no tombstones).
  Used in: Rust HashMap, Google Abseil flat_hash_map.

Swiss Table (2017, Abseil team):
  Robin Hood at the slot level, SIMD for group-level parallel comparison.
  Checks 16 slots simultaneously with one CPU instruction.
  State of the art as of 2024.
```

---

## 15. Self-Test Questions

1. **Trace the insertion of keys with hashes [3, 3, 3, 3, 4] into a linear probing table of capacity 8. What is the cluster length? How many probes does the last insertion require?**

2. **Explain primary clustering in your own words. Why does it not occur in quadratic probing? Why does it still slow down quadratic probing (secondary clustering)?**

3. **The Knuth formula gives expected probe length for unsuccessful lookup as ½(1 + 1/(1-α)²). Compute the expected probe length for α = 0.5, 0.7, 0.9. What does this tell you about safe operating load factors?**

4. **Draw the state of a capacity-6 table before and after deleting the middle element of a 3-entry cluster, using (a) tombstones and (b) backward shift. Verify all remaining entries are still findable in both cases.**

5. **A table has 40 occupied slots, 30 tombstones, and capacity 100. What is the true load factor? What is the effective load factor? Should a rehash be triggered (assume max_load=0.70)? After rehash to capacity 200, how many occupied slots remain? How many tombstones?**

6. **Implement the backward shift deletion condition. Given a gap at position `i` and an entry at position `j` with home hash `h`, write the exact condition (handling the circular case) that determines whether the entry should move.**

7. **Why does linear probing have better cache behaviour than quadratic probing or double hashing? Quantify the difference in terms of cache lines loaded per probe sequence.**

8. **Robin Hood hashing steals slots from "rich" entries (low DIB) for "poor" entries (high DIB). After stealing, the displaced entry continues inserting. Prove that this process always terminates — i.e., there cannot be an infinite chain of displacements.**

9. **CPython's probe sequence uses `i = (5*i + 1 + perturb) % m` with decaying `perturb`. Explain why this visits all m slots when m is a power of 2 (hint: consider the sequence when perturb=0). Why is this property important?**

10. **Design a linear probing hash table optimised for the case where keys are 64-bit integers and values are 32-bit integers. What slot layout would you use? What capacity choice? What hash function? Justify each decision in terms of cache line utilisation.**

---

## Quick Reference Card

```
Linear Probing — O(1) average lookup using contiguous array probing.

Probe sequence: h(k), h(k)+1, h(k)+2, ... (mod capacity)
Termination:    STOP at EMPTY slot (key not found if we reach EMPTY before key)
DO NOT stop at DELETED (tombstone) — continue probing past it.

Three slot states:
  EMPTY   (never occupied)  → terminates probe sequence
  OCCUPIED (has key-value)  → compare key; continue if mismatch
  DELETED  (tombstone ☠)   → skip during lookup; reuse during insert

Key formulas (load factor α = n/m):
  E[probes, successful]   = ½(1 + 1/(1-α))
  E[probes, unsuccessful] = ½(1 + 1/(1-α)²)
  Max safe load: α ≤ 0.70 (performance cliff above this)

Deletion strategies:
  Tombstone: mark as DELETED. Simple. Accumulate over time → rehash.
  Backward shift: shift subsequent entries back to fill gap. Complex. No tombstones.

Insert at tombstone: when inserting, track first DELETED seen.
  If key not found → insert at first DELETED (not at EMPTY).
  This reclaims tombstone slots and keeps chains shorter.

Rehash trigger: (size + tombstones) / capacity > max_load
  NOT just size/capacity — tombstones slow down unsuccessful lookups too!
  On rehash: tombstones are dropped — table rebuilt clean.

Pitfall checklist:
  ✗ Stopping at tombstones during lookup (treats them like EMPTY) → key not found
  ✗ Marking deleted slots as EMPTY → breaks probe invariant for subsequent keys
  ✗ Load factor ignoring tombstones → table appears healthy while probes are O(n)
  ✗ Non-modular slot advance → out-of-bounds array access
  ✗ Wrong backward shift condition in wrapped case
  ✗ Not releasing string/object memory in deleted slots

vs chaining:
  +Cache-friendly (contiguous array, no pointer chasing)
  +Zero per-entry pointer overhead (8 bytes saved per entry)
  -Primary clustering degrades performance at high load
  -Deletion requires tombstones or backward shift
  -Max load must stay ≤ 0.70 for good performance

Evolution:
  Linear probing → Robin Hood (reduce DIB variance)
  Robin Hood → Swiss Table (SIMD parallel slot checking)
```

---

*Previous: [15 — Collision: Chaining](./15_collision_chaining.md)*
*Next: [17 — Collision: Quadratic Probing](./17_quadratic_probing.md)*
*See also: [14 — Hash Map / Hash Table](./14_hash_map.md) | [Robin Hood Hashing](./hash/robin_hood.md) | [Swiss Table](./hash/swiss_table.md)*

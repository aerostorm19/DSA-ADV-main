# Collision Handling: Quadratic Probing

> **Curriculum position:** Hash-Based Structures → #4
> **Interview weight:** ★★☆☆☆ Medium-Low — the concept is a standard interview topic; the mathematical guarantees are frequently tested in design discussions
> **Difficulty:** Intermediate — the intuition is simple; the cycle-guarantee mathematics and secondary clustering subtleties require depth
> **Prerequisite:** [14 — Hash Map / Hash Table](./14_hash_map.md) | [16 — Collision: Linear Probing](./16_linear_probing.md)

---

## Table of Contents

1. [Intuition First — Jumping Further Each Time](#1-intuition-first)
2. [The Probe Sequence — How Quadratic Differs](#2-the-probe-sequence)
3. [The Cycle Problem — Why Quadratic Can Get Stuck](#3-the-cycle-problem)
4. [The Guarantee — When Quadratic Visits All Slots](#4-the-guarantee)
5. [Primary vs Secondary Clustering](#5-primary-vs-secondary-clustering)
6. [Deletion and Tombstones](#6-deletion-and-tombstones)
7. [Load Factor and Performance](#7-load-factor-and-performance)
8. [Time & Space Complexity](#8-time--space-complexity)
9. [Complete C++ Implementation](#9-complete-c-implementation)
10. [Core Operations — Visualised](#10-core-operations--visualised)
11. [Interview Problems](#11-interview-problems)
12. [Real-World Uses](#12-real-world-uses)
13. [Edge Cases & Pitfalls](#13-edge-cases--pitfalls)
14. [Comparison: Quadratic vs Linear vs Double Hashing](#14-comparison)
15. [Self-Test Questions](#15-self-test-questions)

---

## 1. Intuition First

Linear probing searches for an empty slot by walking forward one step at a time. This simplicity creates its main problem: once a collision occurs and a key is displaced, it occupies a slot adjacent to its home, which in turn blocks the next key that hashes there, which blocks the next, and so on. Long runs of occupied adjacent slots form and grow, creating primary clustering.

**Quadratic probing breaks this pattern by jumping further with each probe attempt.**

Instead of probing at positions `h, h+1, h+2, h+3, ...`, it probes at positions `h, h+1², h+2², h+3², ...` — that is, `h+0, h+1, h+4, h+9, h+16, ...`. The jumps grow quadratically: 1 step, then 4, then 9, then 16. Keys that land in the same bucket fan out to very different positions in the table rather than piling up in adjacent slots.

Think of a crowded concert where you cannot get to your assigned seat. With linear probing, you check the next seat, then the next, forming a slow shuffle — and everyone behind you is doing the same shuffle, making the crowd denser. With quadratic probing, you jump to a seat much further along with each attempt — first 1 row away, then 4, then 9 — quickly escaping the congested neighbourhood.

This eliminates **primary clustering**: two keys with the same hash still follow the same probe sequence (they will not help each other), but two keys with *different* hashes do *not* merge their probe sequences the way they do in linear probing. A key that lands one slot away from a cluster does not automatically extend that cluster by following the same linear walk.

The cost: quadratic probing does not guarantee that every slot will be visited. If the table is more than half full and the capacity is not a prime number, the probe sequence may cycle back to already-visited slots before finding an empty one — causing insertion to fail even when empty slots exist elsewhere. This is the cycle problem, and it is the most important thing to understand about quadratic probing.

---

## 2. The Probe Sequence — How Quadratic Differs

### The General Formula

For a key `k` with initial hash `h = hash(k) % m`, the sequence of slots examined is:

```
Probe 0: (h + 0²) % m = (h + 0) % m = h
Probe 1: (h + 1²) % m = (h + 1) % m
Probe 2: (h + 2²) % m = (h + 4) % m
Probe 3: (h + 3²) % m = (h + 9) % m
Probe 4: (h + 4²) % m = (h + 16) % m
Probe i: (h + i²) % m
```

### Symmetric Variant (Alternative Quadratic)

Some implementations use a **symmetric** probe sequence that alternates between positive and negative offsets:

```
Probe 0: (h + 0)      % m = h
Probe 1: (h + 1²)     % m = (h + 1) % m
Probe 2: (h - 1²)     % m = (h - 1 + m) % m
Probe 3: (h + 2²)     % m = (h + 4) % m
Probe 4: (h - 2²)     % m = (h - 4 + m) % m
Probe i (odd):  (h + ceil(i/2)²) % m
Probe i (even): (h - floor(i/2)²) % m
```

This symmetric variant has better spread and guarantees full coverage for any table size (when capacity is a prime of the form `4k + 3`). Java's `IdentityHashMap` uses a variant of this.

### Comparing the Three Probe Sequences

```
For h = 3, capacity = 16:

Linear:    3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 0, 1, 2
           (simple arithmetic progression, stride 1)

Quadratic: 3, 4, 7, 12, 3, 12, 7, 4, 3, ...
           ↑ wait — (3+0)=3, (3+1)=4, (3+4)=7, (3+9)=12, (3+16)%16=3 → CYCLE!
           With m=16 (non-prime), cycle appears at probe 4. Only 4 distinct slots seen.

Quadratic with prime m=17:
           3, 4, 7, 12, 2, 11, 5, 15, 10, 8, 9, 13, 2, ...
           Wait, (3+16)%17 = 19%17 = 2, (3+25)%17 = 28%17 = 11, etc.
           With prime capacity, the cycle is much longer — see Section 4.

Double h: 3, 3+h2, 3+2h2, 3+3h2, ...  (stride = h2(k), varies per key)
```

---

## 3. The Cycle Problem — Why Quadratic Can Get Stuck

This is the most critical concept in quadratic probing and the most common interview question about it.

### The Problem

With a composite capacity (e.g., power of 2), quadratic probing may cycle through a small subset of slots, missing empty slots that exist elsewhere in the table.

```
Example: capacity = 8 (power of 2), h = 0

Probe 0: (0 + 0)  % 8 = 0
Probe 1: (0 + 1)  % 8 = 1
Probe 2: (0 + 4)  % 8 = 4
Probe 3: (0 + 9)  % 8 = 1   ← REVISIT slot 1
Probe 4: (0 + 16) % 8 = 0   ← REVISIT slot 0
Probe 5: (0 + 25) % 8 = 1   ← REVISIT slot 1
...

Only slots {0, 1, 4} are ever examined! Slots {2, 3, 5, 6, 7} are unreachable.
If all of {0, 1, 4} are occupied, insertion FAILS even though 5 slots are free.
```

### Why This Happens

The offsets `i² mod m` form a sequence that repeats. For `m = 8`:

```
i:        0  1  2  3  4  5  6  7  8  9 ...
i² % 8:   0  1  4  1  0  1  4  1  0  1 ...

Distinct values: {0, 1, 4} — only 3 distinct offsets out of 8!
```

The offsets `i² mod m` are periodic with period `m` when `m` is a power of 2, but they produce only `m/2 + 1` distinct values instead of `m`. This means at most `m/2 + 1` slots are reachable from any starting position.

### Visualising the Failure

```
capacity=8, all slots occupied except slots 2, 3, 5, 6, 7.

Table: [X][X][_][_][X][_][_][_]
        0   1   2   3   4   5   6   7

Insert key with h=0:
  Probe 0: slot 0 → occupied.
  Probe 1: slot 1 → occupied.
  Probe 2: slot 4 → occupied.
  Probe 3: slot 1 → occupied (revisit!).
  Probe 4: slot 0 → occupied (revisit!).
  ...will never reach slots 2, 3, 5, 6, 7.
  INFINITE LOOP or wrong "table full" conclusion. ✗

But the table is only 3/8 = 37.5% full! 5 empty slots exist.
Quadratic probing with power-of-2 capacity cannot find them.
```

---

## 4. The Guarantee — When Quadratic Visits All Slots

For quadratic probing to work correctly, we need a mathematical guarantee that the probe sequence visits every slot before cycling.

### Theorem: Prime Capacity Guarantees Half-Coverage

> **If the table capacity `m` is prime and the load factor α < 0.5 (i.e., fewer than m/2 entries), then quadratic probing with sequence `h + i²` always finds an empty slot.**

**Proof sketch:**

For a prime `m`, the first `⌈m/2⌉` probes `i = 0, 1, ..., ⌈m/2⌉ - 1` produce all distinct values `(h + i²) mod m`. Here is why:

Suppose two probes `i` and `j` (with `0 ≤ i < j ≤ ⌈m/2⌉`) produce the same slot:
```
(h + i²) ≡ (h + j²)  (mod m)
         j² - i² ≡ 0  (mod m)
   (j - i)(j + i) ≡ 0  (mod m)
```

Since `m` is prime, either `m | (j-i)` or `m | (j+i)`.

- `m | (j-i)`: Since `0 < j-i < m`, this is impossible.
- `m | (j+i)`: Since `0 < j+i < m` (because `j ≤ ⌈m/2⌉` and `i < j`), this is also impossible.

Therefore, no two of the first `⌈m/2⌉` probes produce the same slot. If fewer than `m/2` slots are occupied (α < 0.5), at least one of these `⌈m/2⌉` probes must land on an empty slot. ✓

### Theorem: Prime + Specific Form Guarantees Full Coverage

> **If the capacity `m` is prime AND `m ≡ 3 (mod 4)` (i.e., `m = 4k + 3` for some integer `k`), then the symmetric quadratic probe sequence `h, h+1², h-1², h+2², h-2², ...` visits ALL `m` slots.**

Primes of the form `4k + 3`: 3, 7, 11, 19, 23, 31, 43, 47, 59, 67, 71, ...

### Practical Capacity Choices

```
Power of 2 (e.g., 8, 16, 32, 64, 128, ...):
  Fast modulo (bitwise AND), but only m/2 slots reachable.
  Max load must be < 0.5.
  Used when: simplicity and speed are paramount; load is controlled.

Prime (e.g., 11, 23, 47, 97, 197, 397, ...):
  Slightly slower modulo (division), but ⌈m/2⌉ slots are reachable.
  Max load should be < 0.5.
  Guarantees insertion success when α < 0.5.
  Used when: correctness guarantee matters more than raw speed.

Prime of form 4k+3 with symmetric probing:
  All m slots reachable. Max load can approach 1.0.
  More complex implementation.
  Used when: maximum utilisation is needed.

Typical choice: prime capacity, max load 0.5 for standard quadratic.
```

---

## 5. Primary vs Secondary Clustering

### Primary Clustering (Linear Probing Problem)

In linear probing, if `k1` and `k2` both hash to bucket `b`, they follow the exact same probe sequence: `b, b+1, b+2, ...`. Any run of occupied slots that one key extends will be extended by the other, and by any future key hashing anywhere into that run.

```
Linear probing: h(A)=h(B)=3
  A placed at slot 3.
  B probes: 3(A)→4→empty. B placed at slot 4.
  C with h(C)=4 probes: 4(B)→5→empty. C placed at slot 5.
  C's placement was influenced by A even though h(C)≠h(A)!
  ← This is primary clustering: keys with DIFFERENT hashes interfere.
```

### Secondary Clustering (Quadratic Probing Problem)

Quadratic probing eliminates primary clustering but introduces **secondary clustering**: two keys with the *same* initial hash `h` follow the exact same probe sequence. They share the same "detour route" through the table.

```
Quadratic probing: h(A)=h(B)=3, capacity=17
  A probes: 3, 4, 7, 12, 2, 11, ...  A placed at slot 3.
  B probes: 3(A), 4, 7, 12, 2, ...   B placed at slot 4.

  Now C with h(C)=5 probes: 5, 6, 9, 14, 4(B!), ...
  C reaches slot 4 only at probe 3 — it does not share A or B's path.
  ← No primary clustering.

  But D with h(D)=3 probes: 3(A), 4(B), 7, 12, ...
  D and A share the ENTIRE probe sequence.
  ← Secondary clustering: keys with SAME hash share the full detour.
```

### Comparing the Clustering Types

```
Primary clustering (linear probing):
  - Affects keys with DIFFERENT hashes
  - Probe sequences MERGE when keys are adjacent
  - Creates long runs of consecutive occupied slots
  - Can make any key slow, regardless of its hash value
  - Severity grows as O((1-α)²) — explodes near load 1.0

Secondary clustering (quadratic probing):
  - Affects only keys with the SAME hash
  - Probe sequences are parallel but never merge
  - No consecutive runs — displaced keys scatter across the table
  - Only affects keys that genuinely collide (same bucket)
  - Severity grows as O(chain length at each bucket) — much less dramatic
  - Can be eliminated entirely by double hashing (different step per key)
```

### Why Secondary Clustering Is Acceptable

With a good hash function, the probability that two keys share the same initial hash is 1/m (where m is the table capacity). For a table of 1000 buckets, there is only a 0.1% chance that any two keys collide at the same bucket. Secondary clustering only affects this small fraction, making its practical impact minimal for well-distributed hash functions.

---

## 6. Deletion and Tombstones

Quadratic probing faces the same deletion problem as linear probing: marking a deleted slot as EMPTY breaks the probe chain for keys that were placed beyond that slot.

### The Tombstone Requirement

```
Table (capacity=11, prime): slots 2,4,7 occupied; all from same hash h=2.

Probe sequence from h=2:
  Probe 0: (2+0²)%11 = 2  → key A
  Probe 1: (2+1²)%11 = 3  → empty originally, but actually:
  Wait, let me trace insertions properly.

Insert A (h=2): slot 2 → empty. Place A at slot 2.
Insert B (h=2): probe 2(A), probe (2+1)%11=3 → empty. Place B at slot 3.
Insert C (h=2): probe 2(A), probe 3(B), probe (2+4)%11=6 → empty. Place C at slot 6.

Table: [_][_][A][B][_][_][C][_][_][_][_]

Delete B at slot 3 → mark as EMPTY (WRONG):
  Table: [_][_][A][_][_][_][C][_][_][_][_]

  Lookup C (h=2):
    Probe 2: A≠C. Probe 3: EMPTY → STOP. Return "not found". ✗ WRONG!
    C exists at slot 6 but the EMPTY at slot 3 broke the probe chain.

Delete B at slot 3 → mark as DELETED ☠ (CORRECT):
  Table: [_][_][A][☠][_][_][C][_][_][_][_]

  Lookup C (h=2):
    Probe 2: A≠C. Probe 3: TOMBSTONE → continue. Probe (2+4)%11=6: C=C → FOUND ✓
```

### Tombstone Reuse During Insertion

The key optimisation: when scanning past tombstones during insertion and finding the key is absent, place the new entry at the **first tombstone** encountered, not at the first empty slot. This reclaims the tombstone slot.

```
Find-slot for insert must track both:
  1. The first tombstone encountered (insertion point if key not found)
  2. Whether the key already exists (in which case update in-place)

bool find_for_insert(key, &slot_idx, &first_tombstone):
    idx = home(key)
    first_tombstone = NONE

    for i in 0..capacity:
        slot = (idx + i*i) % capacity

        if state[slot] == EMPTY:
            slot_idx = (first_tombstone != NONE) ? first_tombstone : slot
            return key_not_found

        if state[slot] == DELETED:
            if first_tombstone == NONE: first_tombstone = slot
            continue  // keep probing — key might exist further along

        if key[slot] == key:
            slot_idx = slot
            return key_found
```

---

## 7. Load Factor and Performance

### The Critical Difference From Linear Probing

Quadratic probing has two hard constraints that linear probing does not:

1. **Capacity constraint:** For standard quadratic probing (`h + i²`), the table capacity must be **prime** to guarantee insertion success when α < 0.5.
2. **Load constraint:** Maximum load factor is **α < 0.5** (with prime capacity and standard quadratic) to guarantee that the probe sequence finds an empty slot among its `⌈m/2⌉` distinct positions.

```
With prime capacity, max load 0.5:

α     E[probes, successful]   E[probes, unsuccessful]
──────────────────────────────────────────────────────
0.10    1.02                    1.11
0.25    1.11                    1.27
0.40    1.20                    1.50
0.50    1.44                    2.19        ← max safe load for standard quadratic
0.60    1.55                    2.56
0.70    1.72                    3.39
0.80    2.01                    5.11        ← above safe range

With prime capacity and symmetric probing (4k+3 form):
  Max load can reach 1.0 (all m slots reachable)
  But performance degrades badly above α ≈ 0.7

For comparison (linear probing at same load factors):
  α=0.50: E[probes] = 1.50 successful, 2.50 unsuccessful
  α=0.70: E[probes] = 2.17 successful, 5.67 unsuccessful

Quadratic probing is slightly better than linear probing at the same load,
but the improvement is modest compared to double hashing.
```

### The Load-Performance Trade-Off

```
The safe range for quadratic probing is α < 0.5 (standard) or α < 0.7 (symmetric).

At α = 0.5 with standard quadratic:
  - Probe success guaranteed (prime capacity)
  - Expected probes: ~1.44 successful, ~2.19 unsuccessful
  - Memory utilisation: only 50% of allocated space stores actual data
  - vs linear probing at 0.5: 1.50 / 2.50 (slightly worse for linear)
  - vs chaining at 0.5: ~1.25 / ~0.50 (chaining wins on unsuccessful!)

Trade-off verdict:
  Quadratic probing occupies a middle ground: better cache than chaining,
  better distribution than linear probing, but stricter load constraints
  and more complex implementation than both.
  Double hashing matches quadratic probing's distribution with no load constraint.
  This is why quadratic probing is less common in production than either
  linear probing (cache wins) or double hashing (load wins) or chaining (simple wins).
```

---

## 8. Time & Space Complexity

### Operation Complexities

| Operation | α < 0.5 (safe range) | Worst case | Notes |
|---|---|---|---|
| `insert(k, v)` | **O(1)** | O(m) — probe all reachable | Fail if m not prime and α ≥ 0.5 |
| `lookup(k)` successful | **O(1)** | O(m) | Expected ≈ 1.44 for α=0.5 |
| `lookup(k)` unsuccessful | **O(1)** | O(m) | Expected ≈ 2.19 for α=0.5 |
| `delete(k)` | **O(1)** | O(m) | Mark as tombstone |
| `rehash(2m)` | O(n) | O(n²) | Re-insert all n entries |
| Iteration | O(m) | O(m) | Scan all m slots |
| Build from n | O(n) avg | O(n²) | n insertions |

**Why the worst case is O(m) not O(n)?**

When probing, you may examine up to `⌈m/2⌉` slots before finding an empty one (for prime m) — and with tombstone accumulation, potentially all m slots. Since m = O(n/α), this is O(n) in practice.

### Space Complexity

| Component | Space | Notes |
|---|---|---|
| Slot array | O(m) | Contiguous, single allocation |
| Per-entry overhead | **0 bytes** | No pointers |
| Slot metadata | 2 bits per slot | EMPTY/OCCUPIED/DELETED state |
| vs chaining | Chaining: +8 bytes/entry | Quadratic probing wins |
| Wasted at α=0.5 | 50% of capacity | Half the table unused |
| Effective memory | `n × (key+value) × 2` | 2× due to load constraint |

### Cache Performance vs Linear Probing

```
Linear probing: probes h, h+1, h+2, h+3, h+4, ...
  All within a few cache lines. Each cache line loaded: ~8 entries (64B / 8B).
  For a run of 5 slots: may load only 1-2 cache lines.

Quadratic probing: probes h, h+1, h+4, h+9, h+16, ...
  After probe 2 or 3, the jump distance exceeds one cache line.
  h+4 may be in a different cache line than h.
  h+9 almost certainly is.
  Cache misses: roughly 1 per probe after the first 2.

Summary:
  Linear probing: sequential access → hardware prefetcher helps
  Quadratic probing: irregular jumps → mostly cache misses after probe 1-2
  Quadratic is slower than linear probing in practice for short chains.
  At high load (where chains are long), quadratic wins because it terminates faster.
```

---

## 9. Complete C++ Implementation

```cpp
#include <iostream>
#include <vector>
#include <optional>
#include <functional>
#include <stdexcept>
#include <cmath>
using namespace std;

template<typename K, typename V,
         typename Hash  = hash<K>,
         typename Equal = equal_to<K>>
class QuadraticProbingMap {
private:
    // ── Slot State ────────────────────────────────────────────────────────────
    enum class State : uint8_t { EMPTY, OCCUPIED, DELETED };

    struct Slot {
        K     key;
        V     value;
        State state = State::EMPTY;
    };

    // ── Data ──────────────────────────────────────────────────────────────────
    vector<Slot> table_;
    size_t       size_;         // number of OCCUPIED entries
    size_t       capacity_;     // must be prime for guarantees
    size_t       tombstones_;   // number of DELETED slots
    float        max_load_;     // rehash threshold (keep < 0.5 for guarantees)
    Hash         hasher_;
    Equal        equal_;

    // ── Private Utilities ─────────────────────────────────────────────────────

    static bool is_prime(size_t n) {
        if (n < 2) return false;
        if (n == 2 || n == 3) return true;
        if (n % 2 == 0 || n % 3 == 0) return false;
        for (size_t i = 5; i * i <= n; i += 6) {
            if (n % i == 0 || n % (i + 2) == 0) return false;
        }
        return true;
    }

    // Find the next prime ≥ n
    static size_t next_prime(size_t n) {
        if (n <= 2) return 2;
        size_t p = (n % 2 == 0) ? n + 1 : n;
        while (!is_prime(p)) p += 2;
        return p;
    }

    size_t home(const K& key) const {
        return hasher_(key) % capacity_;
    }

    // Quadratic probe: slot at probe step i from starting position h
    size_t probe_slot(size_t h, size_t i) const {
        return (h + i * i) % capacity_;
    }

    // Find the slot for key.
    // Returns {slot_index, found}:
    //   found=true:  table_[slot_index].state==OCCUPIED and key matches
    //   found=false: slot_index is best insertion point (first tombstone or empty)
    pair<size_t, bool> find_slot(const K& key) const {
        size_t h           = home(key);
        size_t first_tomb  = capacity_;   // sentinel: no tombstone seen

        for (size_t i = 0; i < (capacity_ + 1) / 2; i++) {
            size_t idx = probe_slot(h, i);
            const Slot& s = table_[idx];

            if (s.state == State::EMPTY) {
                size_t insert_at = (first_tomb < capacity_) ? first_tomb : idx;
                return {insert_at, false};
            }
            if (s.state == State::DELETED) {
                if (first_tomb == capacity_) first_tomb = idx;
                // Continue — key might be further along the probe sequence
            } else {
                // OCCUPIED
                if (equal_(s.key, key)) return {idx, true};
            }
        }

        // Exhausted all probe positions without finding empty slot or key.
        // If we saw a tombstone, use that as the insertion point.
        if (first_tomb < capacity_) return {first_tomb, false};

        // Table is effectively full (all reachable slots occupied/deleted)
        throw overflow_error("hash table full — resize required");
    }

    bool needs_rehash() const {
        return (float)(size_ + tombstones_) / capacity_ > max_load_;
    }

    void do_rehash(size_t new_cap_hint) {
        size_t new_cap = next_prime(new_cap_hint);

        vector<Slot> old = move(table_);
        table_.assign(new_cap, Slot{});
        capacity_   = new_cap;
        size_       = 0;
        tombstones_ = 0;

        for (auto& slot : old) {
            if (slot.state == State::OCCUPIED) {
                insert(move(slot.key), move(slot.value));
            }
            // Tombstones are discarded — table rebuilt clean
        }
    }

public:
    // ── Constructors ──────────────────────────────────────────────────────────

    explicit QuadraticProbingMap(size_t initial_cap = 11, float max_load = 0.45f)
        : table_(next_prime(initial_cap), Slot{}),
          size_(0),
          capacity_(next_prime(initial_cap)),
          tombstones_(0),
          max_load_(max_load) {
        // max_load = 0.45 gives a comfortable margin below the 0.5 guarantee threshold
    }

    ~QuadraticProbingMap() = default;

    QuadraticProbingMap(const QuadraticProbingMap&)            = delete;
    QuadraticProbingMap& operator=(const QuadraticProbingMap&) = delete;
    QuadraticProbingMap(QuadraticProbingMap&&)                 = default;
    QuadraticProbingMap& operator=(QuadraticProbingMap&&)      = default;

    // ── Insert / Update ───────────────────────────────────────────────────────

    void insert(const K& key, const V& val) {
        if (needs_rehash()) do_rehash(capacity_ * 2);

        auto [idx, found] = find_slot(key);
        if (found) {
            table_[idx].value = val;
        } else {
            bool was_tomb = (table_[idx].state == State::DELETED);
            table_[idx] = {key, val, State::OCCUPIED};
            size_++;
            if (was_tomb) tombstones_--;
        }
    }

    void insert(K&& key, V&& val) {
        if (needs_rehash()) do_rehash(capacity_ * 2);

        K key_copy = key;
        auto [idx, found] = find_slot(key_copy);
        if (found) {
            table_[idx].value = move(val);
        } else {
            bool was_tomb = (table_[idx].state == State::DELETED);
            table_[idx] = {move(key), move(val), State::OCCUPIED};
            size_++;
            if (was_tomb) tombstones_--;
        }
    }

    // ── operator[] ────────────────────────────────────────────────────────────

    V& operator[](const K& key) {
        if (needs_rehash()) do_rehash(capacity_ * 2);

        auto [idx, found] = find_slot(key);
        if (!found) {
            bool was_tomb = (table_[idx].state == State::DELETED);
            table_[idx] = {key, V{}, State::OCCUPIED};
            size_++;
            if (was_tomb) tombstones_--;
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
        try {
            auto [idx, found] = find_slot(key);
            if (!found) return nullopt;
            return table_[idx].value;
        } catch (const overflow_error&) {
            return nullopt;  // table effectively full, key cannot exist
        }
    }

    bool contains(const K& key) const {
        return get(key).has_value();
    }

    // ── Deletion ──────────────────────────────────────────────────────────────

    bool erase(const K& key) {
        auto [idx, found] = find_slot(key);
        if (!found) return false;

        table_[idx].key   = K{};   // release memory for strings etc.
        table_[idx].value = V{};
        table_[idx].state = State::DELETED;
        size_--;
        tombstones_++;
        return true;
    }

    // ── Capacity / State ──────────────────────────────────────────────────────

    size_t size()           const { return size_; }
    bool   empty()          const { return size_ == 0; }
    size_t capacity()       const { return capacity_; }
    float  load_factor()    const { return (float)size_ / capacity_; }
    float  eff_load_factor()const { return (float)(size_+tombstones_)/capacity_; }
    bool   is_prime_cap()   const { return is_prime(capacity_); }

    void clear() {
        table_.assign(capacity_, Slot{});
        size_ = tombstones_ = 0;
    }

    void reserve(size_t n) {
        size_t needed = next_prime((size_t)ceil(n / max_load_));
        if (needed > capacity_) do_rehash(needed);
    }

    // ── Diagnostics ───────────────────────────────────────────────────────────

    void print() const {
        for (size_t i = 0; i < capacity_; i++) {
            cout << "[" << i << "] ";
            switch (table_[i].state) {
                case State::EMPTY:   cout << "EMPTY\n"; break;
                case State::DELETED: cout << "DELETED\n"; break;
                case State::OCCUPIED:
                    cout << table_[i].key << ":" << table_[i].value << "\n"; break;
            }
        }
        cout << "size=" << size_ << " tombstones=" << tombstones_
             << " capacity=" << capacity_ << " (prime=" << is_prime_cap() << ")"
             << " load=" << load_factor() << "\n";
    }

    // Distribution of probe distances (DIB) for all stored keys
    void probe_distance_stats() const {
        size_t total = 0, maxd = 0, count = 0;
        for (size_t i = 0; i < capacity_; i++) {
            if (table_[i].state != State::OCCUPIED) continue;
            size_t h = home(table_[i].key);
            // Find which probe step i corresponds to
            for (size_t step = 0; step < (capacity_+1)/2; step++) {
                if (probe_slot(h, step) == i) {
                    total += step;
                    maxd = max(maxd, step);
                    count++;
                    break;
                }
            }
        }
        if (count > 0)
            cout << "avg_probe_dist=" << (float)total/count
                 << " max_probe_dist=" << maxd << "\n";
    }

    // ── Iteration ─────────────────────────────────────────────────────────────

    class Iterator {
        const QuadraticProbingMap* map_;
        size_t idx_;
        void advance() {
            while (idx_ < map_->capacity_ &&
                   map_->table_[idx_].state != State::OCCUPIED)
                ++idx_;
        }
    public:
        Iterator(const QuadraticProbingMap* m, size_t i) : map_(m), idx_(i) { advance(); }
        pair<const K&, const V&> operator*() const {
            return {map_->table_[idx_].key, map_->table_[idx_].value};
        }
        Iterator& operator++() { ++idx_; advance(); return *this; }
        bool operator!=(const Iterator& o) const { return idx_ != o.idx_; }
    };

    Iterator begin() const { return {this, 0}; }
    Iterator end()   const { return {this, capacity_}; }
};
```

### Symmetric Quadratic Probing Variant

```cpp
// For completeness: symmetric variant that achieves full table coverage
// when capacity is prime of form 4k+3.
// Probe sequence: h, h+1², h-1², h+2², h-2², h+3², h-3², ...

size_t symmetric_probe_slot(size_t h, size_t i, size_t cap) {
    // i is 0-based: 0→h, 1→h+1, 2→h-1, 3→h+4, 4→h-4, ...
    long long offset;
    long long step = (long long)((i + 1) / 2);
    if (i % 2 == 1) {
        offset = step * step;     // positive
    } else {
        offset = -(step * step);  // negative
    }
    return (size_t)((long long)h + offset % (long long)cap + (long long)cap) % cap;
}

// With capacity prime and ≡ 3 (mod 4), this visits ALL cap slots.
// Safer for high load scenarios than standard quadratic.
```

---

## 10. Core Operations — Visualised

### Insert Sequence with Collision

```
capacity=11 (prime), max_load=0.45

Insert keys: h("A")=3, h("B")=3, h("C")=3, h("D")=7

Step 1 — insert A (h=3):
  Probe 0: slot (3+0²)%11 = 3. EMPTY → place A.
  Table: [_][_][_][A][_][_][_][_][_][_][_]

Step 2 — insert B (h=3):
  Probe 0: slot 3. OCCUPIED (A≠B). Next.
  Probe 1: slot (3+1²)%11 = 4. EMPTY → place B.
  Table: [_][_][_][A][B][_][_][_][_][_][_]

Step 3 — insert C (h=3):
  Probe 0: slot 3. OCCUPIED (A). Next.
  Probe 1: slot 4. OCCUPIED (B). Next.
  Probe 2: slot (3+4)%11 = 7. EMPTY → place C.
  Table: [_][_][_][A][B][_][_][C][_][_][_]

Step 4 — insert D (h=7):
  Probe 0: slot 7. OCCUPIED (C≠D). Next.
  Probe 1: slot (7+1)%11 = 8. EMPTY → place D.
  Table: [_][_][_][A][B][_][_][C][D][_][_]

Observations:
  A, B, C all hash to slot 3 (secondary clustering: all share probe sequence).
  D hashes to slot 7 — it encounters C (which was displaced from 3 to 7),
  but D only needs 1 probe, not 3. PRIMARY CLUSTERING IS ABSENT.
  With linear probing, D would probe: 7(C), 8(D) — same result here, but
  if A,B had formed a run [3,4], D at slot 4 would extend the run to [3,4,5].
  Quadratic probing prevents that extension.
```

### The Cycle Failure — Power-of-Two Capacity

```
capacity=8 (NOT prime — power of 2!), table 62.5% full:
  Slots 0,1,4 are OCCUPIED. All others EMPTY.
  [X][X][_][_][X][_][_][_]
   0   1   2   3   4   5   6   7

Insert key with h=0:
  Probe 0: slot (0+0)%8 = 0. OCCUPIED.
  Probe 1: slot (0+1)%8 = 1. OCCUPIED.
  Probe 2: slot (0+4)%8 = 4. OCCUPIED.
  Probe 3: slot (0+9)%8 = 1. OCCUPIED (revisit!).
  Probe 4: slot (0+16)%8 = 0. OCCUPIED (revisit!).
  Probe 5: slot (0+25)%8 = 1. OCCUPIED (revisit!).
  ...CYCLE. Never visits slots 2, 3, 5, 6, 7.

INSERTION FAILS even though 5 slots are empty.
This is the cycle problem. Fix: use prime capacity.

capacity=11 (prime), same scenario:
  Probe 0: slot 0. OCCUPIED.
  Probe 1: slot 1. OCCUPIED.
  Probe 2: slot 4. OCCUPIED.
  Probe 3: slot (0+9)%11 = 9. EMPTY → place here! ✓

Prime capacity breaks the cycle.
```

### Tombstone Interaction

```
Table (cap=11): [_][_][_][A][B][_][_][C][_][_][_]
h(A)=h(B)=h(C)=3

Delete B at slot 4 → tombstone:
  [_][_][_][A][☠][_][_][C][_][_][_]

Lookup C (h=3):
  Probe 0: slot 3 → A≠C. Continue.
  Probe 1: slot 4 → TOMBSTONE → continue (do NOT stop here).
  Probe 2: slot 7 → C=C → FOUND ✓

Insert E (h=3):
  Probe 0: slot 3 → A≠E. Continue.
  Probe 1: slot 4 → TOMBSTONE → record as first_tomb=4. Continue.
  Probe 2: slot 7 → C≠E. Continue.
  Probe 3: slot (3+9)%11 = 1 → EMPTY → STOP.
  first_tomb=4 was recorded, so insert E at slot 4 (reclaim tombstone!).
  [_][_][_][A][E][_][_][C][_][_][_]   ← E replaced the tombstone ✓
```

---

## 11. Interview Problems

### Problem 1: Design Hash Set with Open Addressing

**Problem:** Implement a hash set without using built-in hash table libraries. Support `add`, `remove`, and `contains`, all in O(1) average.

**This tests:** quadratic probing mechanics, tombstone management, prime capacity selection.

```cpp
class MyHashSet {
private:
    static const int BASE_CAP = 769;  // prime starting capacity
    // Common prime series: 11, 23, 47, 97, 197, 389, 769, 1543, 3079...

    enum State : uint8_t { EMPTY = 0, OCCUPIED = 1, DELETED = 2 };

    vector<int>   keys_;
    vector<State> states_;
    int           cap_;
    int           size_;
    int           tombstones_;

    int home(int key) const { return key % cap_; }

    // Returns index where key is found or should be inserted.
    // Returns -1 if table is effectively full and key not found.
    int find_slot(int key) const {
        int h = home(key);
        int first_tomb = -1;

        for (int i = 0; i < (cap_ + 1) / 2; i++) {
            int idx = (h + i * i) % cap_;

            if (states_[idx] == EMPTY) {
                return (first_tomb != -1) ? first_tomb : idx;
            }
            if (states_[idx] == DELETED) {
                if (first_tomb == -1) first_tomb = idx;
            } else if (keys_[idx] == key) {
                return idx;  // found
            }
        }
        return (first_tomb != -1) ? first_tomb : -1;
    }

    bool key_matches(int idx, int key) const {
        return states_[idx] == OCCUPIED && keys_[idx] == key;
    }

    void rehash() {
        vector<int>   old_keys   = move(keys_);
        vector<State> old_states = move(states_);
        int old_cap = cap_;

        // Find next prime ≥ 2 * current capacity
        cap_ = old_cap * 2 + 1;
        while (!is_prime(cap_)) cap_ += 2;

        keys_.assign(cap_, 0);
        states_.assign(cap_, EMPTY);
        size_ = tombstones_ = 0;

        for (int i = 0; i < old_cap; i++) {
            if (old_states[i] == OCCUPIED) add(old_keys[i]);
        }
    }

    static bool is_prime(int n) {
        if (n < 2) return false;
        for (int i = 2; (long long)i * i <= n; i++)
            if (n % i == 0) return false;
        return true;
    }

public:
    MyHashSet() : cap_(BASE_CAP), size_(0), tombstones_(0) {
        keys_.assign(cap_, 0);
        states_.assign(cap_, EMPTY);
    }

    void add(int key) {
        if ((float)(size_ + tombstones_ + 1) / cap_ > 0.45f) rehash();

        int idx = find_slot(key);
        if (idx == -1) { rehash(); idx = find_slot(key); }

        if (!key_matches(idx, key)) {
            bool was_tomb = (states_[idx] == DELETED);
            keys_[idx]   = key;
            states_[idx] = OCCUPIED;
            size_++;
            if (was_tomb) tombstones_--;
        }
    }

    void remove(int key) {
        int h = home(key);
        for (int i = 0; i < (cap_ + 1) / 2; i++) {
            int idx = (h + i * i) % cap_;
            if (states_[idx] == EMPTY) return;           // not found
            if (key_matches(idx, key)) {
                states_[idx] = DELETED;
                size_--;
                tombstones_++;
                return;
            }
        }
    }

    bool contains(int key) const {
        int h = home(key);
        for (int i = 0; i < (cap_ + 1) / 2; i++) {
            int idx = (h + i * i) % cap_;
            if (states_[idx] == EMPTY)             return false;
            if (key_matches(idx, key))             return true;
            // DELETED: continue probing
        }
        return false;
    }
};
```

---

### Problem 2: Two Sum — Hash Map from Scratch

**Problem:** Given an array of integers and a target, return indices of two numbers that add up to the target. Implement using a custom hash map with quadratic probing rather than std::unordered_map.

**Why this is illustrative:** It shows how the hash map's O(1) lookup transforms an O(n²) problem to O(n), and lets you demonstrate the full insert-then-lookup cycle.

```cpp
pair<int,int> twoSum(vector<int>& nums, int target) {
    // Custom hash map: value → index
    // capacity must be prime and > 2*nums.size() to keep load < 0.5
    int cap = 1009;  // prime, > 2 * 500 (max input size for this variant)

    vector<int> keys(cap, INT_MIN);       // INT_MIN = empty sentinel
    vector<int> vals(cap, -1);
    vector<bool> deleted(cap, false);

    auto hash_fn = [&](int k) -> int {
        return ((long long)k * 2654435769ULL) % (unsigned)cap;
    };

    auto insert_map = [&](int key, int val) {
        int h = hash_fn(key);
        int first_del = -1;
        for (int i = 0; i < (cap + 1) / 2; i++) {
            int idx = (h + i * i) % cap;
            if (keys[idx] == INT_MIN) {
                int target_idx = (first_del != -1) ? first_del : idx;
                keys[target_idx] = key;
                vals[target_idx] = val;
                deleted[target_idx] = false;
                return;
            }
            if (deleted[idx] && first_del == -1) first_del = idx;
            if (!deleted[idx] && keys[idx] == key) { vals[idx] = val; return; }
        }
    };

    auto lookup_map = [&](int key) -> int {
        int h = hash_fn(key);
        for (int i = 0; i < (cap + 1) / 2; i++) {
            int idx = (h + i * i) % cap;
            if (keys[idx] == INT_MIN) return -1;
            if (!deleted[idx] && keys[idx] == key) return vals[idx];
        }
        return -1;
    };

    for (int i = 0; i < (int)nums.size(); i++) {
        int complement = target - nums[i];
        int j = lookup_map(complement);
        if (j != -1) return {j, i};
        insert_map(nums[i], i);
    }
    return {-1, -1};
}
```

---

### Problem 3: First Missing Positive — In-Place Hashing

**Problem:** Given an unsorted integer array `nums`, find the smallest missing positive integer. Must run in O(n) time and O(1) extra space.

**Connection to open addressing:** This problem uses the array itself as a hash table — placing each value `v` at index `v-1`. The technique is exactly the "perfect hash" open addressing concept: the probe sequence is determined by the value itself, with no collisions possible (each value [1..n] maps to a unique index).

```cpp
// Time: O(n) | Space: O(1) extra
// Uses the input array itself as a hash table: value v goes to index v-1.
int firstMissingPositive(vector<int>& nums) {
    int n = nums.size();

    // Step 1: Place each number in its "correct" slot using cyclic sort.
    // This is open addressing with h(v) = v-1 and swap-based insertion.
    for (int i = 0; i < n; i++) {
        // Swap nums[i] to its correct position nums[i]-1
        // while nums[i] is a valid positive and not already in place.
        while (nums[i] >= 1 && nums[i] <= n &&
               nums[nums[i] - 1] != nums[i]) {
            swap(nums[i], nums[nums[i] - 1]);
        }
    }

    // Step 2: Find the first index where nums[i] != i+1.
    for (int i = 0; i < n; i++) {
        if (nums[i] != i + 1) return i + 1;
    }
    return n + 1;  // all 1..n are present; answer is n+1
}

/*
Trace: nums = [3, 4, -1, 1]

i=0: nums[0]=3, should be at idx 2. nums[2]=-1≠3. swap(nums[0],nums[2]).
     nums=[−1,4,3,1]. nums[0]=-1, not in [1,n]. Stop.

i=1: nums[1]=4, should be at idx 3. nums[3]=1≠4. swap(nums[1],nums[3]).
     nums=[−1,1,3,4]. nums[1]=1, should be at idx 0. nums[0]=-1≠1. swap.
     nums=[1,-1,3,4]. nums[1]=-1, not in [1,n]. Stop.

i=2: nums[2]=3, already at correct position (3=2+1). Skip.
i=3: nums[3]=4, already at correct position (4=3+1). Skip.

Final: [1,-1,3,4]
Check: nums[0]=1=1. nums[1]=-1≠2 → RETURN 2. ✓

The cyclic sort is essentially "hashing" each value to its natural position
with O(1) probing via swap — a specialised form of open addressing
where the hash function is perfect (h(v) = v-1, no collisions in [1,n]).
*/
```

---

## 12. Real-World Uses

| Domain | Implementation | Quadratic Probing Detail |
|---|---|---|
| **Java `IdentityHashMap`** | Symmetric quadratic probe | Uses `==` instead of `.equals()`; linear-like step `2i` |
| **Pascal / Delphi hash tables** | Quadratic probing | Classic textbook implementation; prime capacity series |
| **PostgreSQL hash joins** (some paths) | Quadratic when chaining overflows | Fallback when bucket chains exceed threshold |
| **Embedded systems** | Quadratic over chaining | No dynamic allocation for chain nodes needed |
| **Academic / educational** | Canonical example | Widely taught as the intermediate step between linear and double hashing |
| **PHP internal arrays** (historically) | Quadratic probing | PHP 5 used quadratic probing for its internal hash table |
| **Some JVM implementations** | Hybrid linear/quadratic | Switch strategy based on current load |

### Java's IdentityHashMap — Quadratic Probing in the JDK

Java's `IdentityHashMap` is one of the few places in the JDK that uses open addressing rather than chaining:

```java
// IdentityHashMap uses linear probing (step=2) on an array where
// keys and values alternate: [key0, val0, key1, val1, ...]
// But the probe formula is effectively quadratic in spirit because
// it doubles the internal capacity for key+value pairs.

// The internal implementation:
int hash = System.identityHashCode(key);
int len = table.length;
int i = ((hash << 1) & (len - 2));  // initial slot (key position)

while (true) {
    Object item = table[i];
    if (item == key) return i;       // found
    if (item == null) return i;      // empty slot
    i = (i + 2) & (len - 2);        // linear probe by 2 (key+value pairs)
    // This is linear probing, NOT quadratic, despite being in IdentityHashMap.
}

// Key point: IdentityHashMap uses == (reference equality), not .equals().
// This enables open addressing without worrying about .equals() side effects.
// Load threshold: 2/3 (resizes when 2/3 full).
```

**Why does this matter?** `IdentityHashMap` is used internally by the JVM for:
- Serialisation object graphs (tracking visited objects)
- Compiler symbol tables (where identity matters, not value equality)
- Memory leak detection tools

The choice of open addressing here is a performance optimisation: reference equality (`==`) is a single pointer comparison, making each probe extremely fast. The linear probe + reference equality combination gives near-perfect cache behaviour for object graph traversal.

### Why Quadratic Probing Lost to Its Competitors

```
1953: Linear probing invented. Simple, fast.
1963: Knuth's analysis shows clustering problem (primary clustering).
1963: Quadratic probing proposed as fix.
1970s: Double hashing developed as better fix (no primary OR secondary clustering).
1985: Robin Hood hashing reduces variance of linear probing.
2017: Swiss Table achieves 16-way SIMD parallel comparison.

Position of quadratic probing in 2024:
  - Eliminated primary clustering ✓
  - But retained secondary clustering ✗ (double hashing eliminates both)
  - Worse cache behaviour than linear probing ✗ (irregular jumps)
  - Load constraint of α < 0.5 ✗ (linear probing works to 0.7)
  - More complex than linear probing ✗

Verdict: quadratic probing is the textbook "middle ground" that is
         superseded in every dimension by its successors.
         Linear probing + Robin Hood beats it on cache and variance.
         Double hashing beats it on clustering elimination.
         It is widely taught but rarely the best production choice.
```

---

## 13. Edge Cases & Pitfalls

### Pitfall 1: Non-Prime Capacity Causes Cycles

```cpp
// WRONG: using power-of-2 capacity with standard quadratic probing
QuadraticProbingMap<int,int> m(16);  // capacity = 16 (power of 2!)
// At load > 0.5, insertion may fail even with empty slots available.

// Symptom: overflow_error thrown from find_slot despite table < 50% full.
// Root cause: probe sequence cycles through only m/2+1 of the m slots.

// CORRECT: always use prime capacity (or ensure capacity is prime after growth)
QuadraticProbingMap<int,int> m(17);  // capacity = 17 (prime)
// Or let the constructor find the next prime: next_prime(16) = 17.

// Common prime series for capacity doubling:
// 11 → 23 → 47 → 97 → 197 → 397 → 797 → 1597 → 3203 → 6421 → 12853 → ...
// Each is approximately 2× the previous, and all are prime.
```

### Pitfall 2: Load Factor ≥ 0.5 with Standard Quadratic

```cpp
// Standard quadratic probing (h + i²) only guarantees insertion when α < 0.5.
// Using max_load = 0.75 (Java's HashMap default) is UNSAFE for quadratic probing.

// WRONG:
QuadraticProbingMap<int,int> m(11, 0.75f);  // max_load too high!
// At 75% full with prime cap, insertion CAN fail even though slots exist,
// because the guarantee only covers ⌈m/2⌉ of the m slots.

// CORRECT:
QuadraticProbingMap<int,int> m(11, 0.45f);  // safe margin below 0.5
// Or use symmetric quadratic (visits all m slots) with higher max_load.
```

### Pitfall 3: Stopping Probe at Tombstone During Lookup

```cpp
// WRONG: treating DELETED same as EMPTY during lookup
size_t idx = home(key);
for (size_t i = 0; i < capacity_; i++) {
    idx = probe_slot(home(key), i);
    if (table_[idx].state == State::EMPTY ||
        table_[idx].state == State::DELETED) {   // ← WRONG: stops at tombstone
        return nullopt;  // "not found"
    }
    if (equal_(table_[idx].key, key)) return table_[idx].value;
}

// CORRECT: only EMPTY stops the search; DELETED continues
for (size_t i = 0; i < (capacity_+1)/2; i++) {
    size_t idx = probe_slot(home(key), i);
    if (table_[idx].state == State::EMPTY) return nullopt;        // STOP
    if (table_[idx].state == State::OCCUPIED &&
        equal_(table_[idx].key, key)) return table_[idx].value;  // FOUND
    // DELETED: fall through, continue to next probe
}
```

### Pitfall 4: Not Tracking Tombstones in Rehash Trigger

```cpp
// WRONG: only counting occupied slots toward load
bool needs_rehash() const {
    return (float)size_ / capacity_ > max_load_;
}
// Problem: if half the slots are tombstones and half are empty,
// the lookup for an absent key must probe ALL the tombstone slots
// before finding an empty one — O(n) per lookup!

// CORRECT: count tombstones toward the effective load for rehash decisions
bool needs_rehash() const {
    return (float)(size_ + tombstones_) / capacity_ > max_load_;
}
```

### Pitfall 5: Inserting at EMPTY Instead of First Tombstone

```cpp
// WRONG: always inserting at the EMPTY slot found at end of probe
pair<size_t,bool> find_slot(const K& key) {
    size_t h = home(key);
    for (size_t i = 0; i < (capacity_+1)/2; i++) {
        size_t idx = probe_slot(h, i);
        if (table_[idx].state == State::EMPTY) return {idx, false}; // ← skips tombstones!
        if (table_[idx].state == State::OCCUPIED && equal_(table_[idx].key, key))
            return {idx, true};
    }
    return {capacity_, false};  // not found, table full
}
// Problem: tombstone slots are never reclaimed. Table fills with tombstones faster.

// CORRECT: track first tombstone; prefer it over empty slot for insertion
pair<size_t,bool> find_slot(const K& key) {
    size_t h = home(key);
    size_t first_tomb = capacity_;
    for (size_t i = 0; i < (capacity_+1)/2; i++) {
        size_t idx = probe_slot(h, i);
        if (table_[idx].state == State::EMPTY) {
            return {first_tomb < capacity_ ? first_tomb : idx, false};
        }
        if (table_[idx].state == State::DELETED) {
            if (first_tomb == capacity_) first_tomb = idx;
        } else if (equal_(table_[idx].key, key)) {
            return {idx, true};
        }
    }
    return {first_tomb, false};
}
```

### Pitfall 6: Probe Count Limit — Must Be ⌈m/2⌉, Not m

```cpp
// WRONG: probing up to capacity steps (for prime capacity, only ⌈cap/2⌉ are distinct)
for (size_t i = 0; i < capacity_; i++) {  // probes too many: revisits slots
    size_t idx = probe_slot(h, i);
    // ...
}
// At i = ⌈cap/2⌉, the probe sequence starts revisiting earlier slots.
// This wastes time and may falsely "find" tombstones that were already checked.

// CORRECT: limit to ⌈cap/2⌉ distinct probes
for (size_t i = 0; i < (capacity_ + 1) / 2; i++) {
    // ...
}
// For cap=11: (11+1)/2 = 6 probes. Slots reachable: 6 distinct (0+0,0+1,0+4,0+9,0+16%11,0+25%11)
// = (0, 1, 4, 9, 5, 3) → 6 distinct values. Correct.
```

### Pitfall 7: Forgetting to Update tombstones\_ Count

```cpp
// When a tombstone slot is reused during insert:
auto [idx, found] = find_slot(key);
if (!found) {
    table_[idx] = {key, val, State::OCCUPIED};
    size_++;
    // WRONG: forgot to decrement tombstones_ if the slot was DELETED!
    // tombstones_ count is now wrong → rehash triggers at wrong time.

    // CORRECT:
    bool was_tomb = (table_[idx].state == State::DELETED);
    table_[idx] = {key, val, State::OCCUPIED};
    size_++;
    if (was_tomb) tombstones_--;  // ← tombstone reclaimed, decrement count
}
```

---

## 14. Comparison: Quadratic vs Linear vs Double Hashing

| Property | Linear Probing | Quadratic Probing | Double Hashing |
|---|---|---|---|
| Probe sequence | `h, h+1, h+2,...` | `h, h+1, h+4, h+9,...` | `h, h+h2, h+2h2,...` |
| Primary clustering | ✗ Severe | ✓ Eliminated | ✓ Eliminated |
| Secondary clustering | N/A | ✗ Yes (same hash → same path) | ✓ Eliminated |
| Cache performance | ★★★ Best (sequential) | ★★ Moderate (early probes close) | ★ Poor (random jumps) |
| Max safe load (practical) | α ≤ 0.70 | α < 0.50 | α ≤ 0.80 |
| Capacity requirement | Any (prime preferred) | Prime for guarantees | Any (prime preferred) |
| Slots reachable | All m (via backward shift) | ⌈m/2⌉ (prime cap) or all (symmetric) | All m (if h2 coprime to m) |
| Deletion | Tombstone or backward shift | Tombstone only | Tombstone only |
| Backward shift delete | ✓ Clean, elegant | ✗ Complex (quadratic shifts) | ✗ Infeasible |
| Implementation complexity | Simple | Moderate | Moderate |
| Used in | Python dict, LLVM DenseMap | Java IdentityHashMap, textbooks | Some caches, GPU hash maps |
| Performance at α=0.5 | ~1.5 probes success | ~1.44 probes success | ~1.39 probes success |
| Performance at α=0.7 | ~2.17 success | ~1.72 success | ~1.58 success |
| Performance at α=0.9 | ~5.50 success | ~2.85 success | ~2.56 success |

**The verdict in practice:**

```
Quadratic probing occupies a theoretically clean but practically awkward position:
  ✓ Better clustering than linear probing
  ✗ Worse cache behaviour than linear probing
  ✗ Stricter load constraints than both alternatives
  ✗ No backward-shift deletion (tombstones required)
  ✗ Capacity must be prime (adds complexity)

Modern engineers typically choose:
  Linear probing  → when cache performance is the top priority
  Double hashing  → when distribution quality matters more than cache
  Robin Hood LP   → when both matter (best of both worlds)
  Quadratic       → when forced by legacy system or implementing from textbook

Academic value: high (illuminates the clustering problem elegantly)
Production value: low (superseded by Robin Hood and Swiss Table)
Interview value: medium (cycling problem and prime requirement are common questions)
```

---

## 15. Self-Test Questions

1. **Trace the probe sequence for a key with `h=5` in a table of capacity `m=11` using standard quadratic probing `(h + i²) % m`. List the first 6 slot indices. Are any revisited? At what probe count?**

2. **Prove that with prime capacity `m` and standard quadratic probing, no two of the first `⌈m/2⌉` probes `(h + 0²), (h + 1²), ..., (h + ⌊m/2⌋²)` produce the same slot index. Use modular arithmetic.**

3. **Construct a concrete example with capacity `m=8` (power of 2) where inserting a new key fails despite the table being only 37.5% full. Show exactly which slots are reachable from the key's hash.**

4. **What is secondary clustering? How does it differ from primary clustering? Give a concrete example of two keys suffering secondary clustering and show their identical probe sequences.**

5. **After inserting 5 keys (all hashing to slot 3) into a capacity-11 table, delete the 2nd and 4th keys (by probe order). Show the tombstone state. Trace a lookup for the 5th key to demonstrate correct tombstone handling.**

6. **Why is the probe count limit `⌈m/2⌉` and not `m` for standard quadratic probing with prime capacity? What happens if you probe up to `m` steps?**

7. **Compare the expected probe length for unsuccessful lookup at α=0.5 between linear probing and quadratic probing. Which is faster? Which degrades faster as α increases toward 1.0?**

8. **Symmetric quadratic probing visits all `m` slots when `m` is prime and `m ≡ 3 (mod 4)`. Verify that 11 satisfies this condition (11 = 4×2 + 3). Trace the full probe sequence from h=0 for m=11 using the symmetric formula.**

9. **You are implementing a hash set in an embedded system where dynamic memory allocation is forbidden (no heap). Quadratic probing or chaining — which is suitable, and why? What capacity would you choose for 50 expected entries?**

10. **Design a hash table that automatically switches between quadratic probing and separate chaining based on the current load factor. What threshold would trigger the switch? What does the transition procedure look like?**

---

## Quick Reference Card

```
Quadratic Probing — O(1) average, reduces primary clustering via quadratic jumps.

Probe sequence: (h + i²) % m  for i = 0, 1, 2, 3, ...
  Slots visited: h, h+1, h+4, h+9, h+16, ... (mod m)

Critical requirements:
  1. Capacity MUST be prime for correctness guarantees
  2. Load factor MUST be < 0.5 (standard) for insertion to always succeed
  3. Use (capacity + 1) / 2 as the probe count limit (⌈m/2⌉ distinct slots)

Three slot states (same as linear probing):
  EMPTY   → terminates probe sequence during lookup
  OCCUPIED → check key; continue if mismatch
  DELETED ☠ → skip during lookup; reuse during insert (track first seen)

Key invariant: track first_tombstone during probe for insertion point.
  If key not found → insert at first_tomb (if seen) else at EMPTY slot.

Insert at correct slot:
  bool was_tomb = (table[idx].state == DELETED);
  table[idx] = {key, val, OCCUPIED};
  size++;
  if (was_tomb) tombstones--;  // must decrement!

Rehash trigger:
  (size + tombstones) / capacity > max_load   ← NOT just size/capacity
  Rehash to next_prime(capacity * 2)

vs Linear Probing:
  ✓ No primary clustering (displaced keys scatter)
  ✗ Secondary clustering still present (same hash → same path)
  ✗ Worse cache (irregular jumps after first 2 probes)
  ✗ Stricter load limit (α < 0.5 vs α < 0.70)
  ✗ Requires prime capacity; no clean backward-shift deletion

Pitfall checklist:
  ✗ Non-prime capacity → cycles, insertion fails with empty slots present
  ✗ max_load ≥ 0.5 → insertion not guaranteed to succeed
  ✗ Stopping at tombstone during lookup → false "not found"
  ✗ Inserting at EMPTY instead of first tombstone → tombstones never reclaimed
  ✗ Probing > ⌈m/2⌉ steps → revisit slots, wasted work
  ✗ Not decrementing tombstones_ when slot reused → wrong rehash timing
  ✗ Only counting size (not tombstones) toward load → late rehash

Prime capacity series (for doubling):
  11 → 23 → 47 → 97 → 197 → 397 → 797 → 1597 → 3203 → 6421
```

---

*Previous: [16 — Collision: Linear Probing](./16_linear_probing.md)*
*Next: [18 — Collision: Double Hashing](./18_double_hashing.md)*
*See also: [14 — Hash Map](./14_hash_map.md) | [Robin Hood Hashing](./hash/robin_hood.md) | [Swiss Table deep dive](./hash/swiss_table.md)*

# Robin Hood Hashing

> **Curriculum position:** Hash-Based Structures → #8
> **Interview weight:** ★★★☆☆ Medium — the DIB concept and variance-reduction insight are common depth questions; Rust's HashMap uses this directly
> **Difficulty:** Intermediate-Advanced — the concept is elegant; the backward-shift deletion and the lookup early-exit optimisation require careful reasoning
> **Prerequisite:** [16 — Linear Probing](./16_linear_probing.md) | [18 — Double Hashing](./18_double_hashing.md)

---

## Table of Contents

1. [Intuition First — Taking from the Rich, Giving to the Poor](#1-intuition-first)
2. [The DIB — Distance from Initial Bucket](#2-the-dib)
3. [Internal Working — The Robin Hood Invariant](#3-internal-working)
4. [Insertion — The Steal and Displace Cycle](#4-insertion)
5. [Lookup — The Early-Exit Optimisation](#5-lookup)
6. [Deletion — Backward Shift Without Tombstones](#6-deletion)
7. [Variance Reduction — Why Robin Hood Is Better Than Linear Probing](#7-variance-reduction)
8. [Time & Space Complexity](#8-time--space-complexity)
9. [Complete C++ Implementation](#9-complete-c-implementation)
10. [Core Operations — Visualised](#10-core-operations--visualised)
11. [Interview Problems](#11-interview-problems)
12. [Real-World Uses](#12-real-world-uses)
13. [Edge Cases & Pitfalls](#13-edge-cases--pitfalls)
14. [Robin Hood vs Linear Probing vs Cuckoo vs Swiss Table](#14-comparison)
15. [Self-Test Questions](#15-self-test-questions)

---

## 1. Intuition First

Linear probing is unfair in a specific sense: a key placed at its home slot (probe distance 0) can indefinitely block keys that hash to nearby slots, forcing them into increasingly long probe chains. Some keys are "rich" — sitting comfortably at their preferred slot with zero displacement — while others are "poor" — displaced five, ten, or fifty slots away from where they belong.

**Robin Hood hashing steals from the rich and gives to the poor.**

When inserting a new key encounters an occupied slot, it compares its own displacement (how far it has already probed from its home) with the displacement of the occupant. If the new key is more displaced — it has wandered further from home and is therefore "poorer" — it steals the slot from the occupant, who is then displaced instead. The displaced key continues the insertion from its new position.

This single rule — steal the slot if you are more displaced — transforms the probe-length distribution from asymmetric and high-variance to symmetric and tightly bounded. The average probe length barely changes, but the *maximum* probe length drops dramatically, and the expected probe length for a *random key* improves significantly.

```
Without Robin Hood (standard linear probing):
  Key A at home slot 5: probe length = 0 (rich)
  Key B hashed to 5, placed at 7: probe length = 2 (poor)
  Key C hashed to 5, placed at 9: probe length = 4 (very poor)
  A never moves, B and C suffer.

With Robin Hood (same insertions):
  A at 5 (length=0): fine.
  B tries 5 (A, length=0 < B's length=1): B is poorer → steal! A moves to 6.
  Wait — A is poorer now (forced from its home). Let me think again.

  Actually Robin Hood:
  Insert A(h=5): slot 5 empty → place A@5. DIB(A)=0.
  Insert B(h=5): slot 5 has A with DIB=0. B's current DIB=0. DIB(B)=DIB(A).
    Equal DIBs — convention: don't steal (or do, depending on variant). Place B@6.
  Insert C(h=5): slot 5 has A(DIB=0), C's DIB=0, not poorer. Try 6: B(DIB=1).
    C's DIB=1 = B's DIB=1. Try 7: empty → place C@7.
  
  With Robin Hood, when inserting D(h=6):
    Slot 6 has B(DIB=1). D's DIB=0 < 1 → D is RICHER than B. D cannot steal.
    Try slot 7: C(DIB=2). D's DIB=1 < 2 → D is richer. Try slot 8: empty → D@8.

    Without Robin Hood: D would also probe 6,7,8 — same result here.
    The benefit appears when probe chains are longer and variance matters.
```

The key metric is the **maximum DIB** across all stored keys. Robin Hood minimises this maximum — no key is left disproportionately far from its home. The result: lookup in the worst case across all stored keys is dramatically better than standard linear probing, while average-case lookup is essentially identical.

---

## 2. The DIB — Distance from Initial Bucket

### Definition

The **Distance from Initial Bucket (DIB)** of a stored key is the number of probe steps it took to place it during insertion:

```
DIB(key) = (current_slot - h(key)) mod capacity

If key k is stored at slot 7 and h(k) % capacity = 3:
  DIB(k) = (7 - 3) mod capacity = 4
  k was displaced 4 slots from its home.

If key k is stored at slot 3 and h(k) % capacity = 3:
  DIB(k) = 0
  k is at its home slot — maximally "rich."
```

### DIB as an Array Annotation

In Robin Hood hashing, each slot stores not just the key-value pair but also the key's DIB:

```
Slot:  [  0  ][  1  ][  2  ][  3  ][  4  ][  5  ][  6  ][  7  ]
Key:   [  _  ][  A  ][  B  ][  C  ][  _  ][  D  ][  E  ][  _  ]
DIB:   [  _  ][  0  ][  1  ][  2  ][  _  ][  0  ][  1  ][  _  ]

A is at its home (slot 1), B was displaced 1, C displaced 2.
D is at its home (slot 5), E displaced 1 from slot 5.
```

The DIB array is the core data structure that Robin Hood hashing maintains.

### Why DIB Storage Is Cheap

For capacity up to 256: DIB fits in one byte.
For capacity up to 65536: DIB fits in two bytes.
For practical table sizes (capacity < 2^32): DIB fits in four bytes.

In practice, the DIB is packed into the metadata byte alongside the EMPTY/OCCUPIED/DELETED state. Swiss Table's design (one byte per slot of metadata) can encode a 7-bit DIB plus state information in 8 bits.

---

## 3. Internal Working — The Robin Hood Invariant

### The Invariant

> **At every occupied slot `i`, the key's DIB equals `(i - h(key)) mod capacity`.  
> For any two adjacent slots `i` and `j = i + 1 (mod capacity)` where both are occupied:  
> `DIB[j] ≤ DIB[i] + 1`.**

In plain English: a key's DIB can never be more than one greater than the DIB of the key in the slot before it. Probe sequences cannot have "jumps" in displacement — they must increase by at most 1 per step.

This invariant is stronger than linear probing's invariant (which only requires that keys are reachable from their home without passing through an EMPTY slot). Robin Hood's invariant bounds the *gradient* of DIB changes.

### Consequence: Bounded Maximum DIB

Because DIBs can only increase by 1 per slot, a run of occupied slots starting from DIB=0 can reach at most DIB=k after k occupied slots. Combined with the load factor:

```
For load factor α, the expected maximum DIB is O(log(1/(1-α))).
At α=0.9: expected max DIB ≈ ln(1/0.1) ≈ 2.3  (extremely low!)

Standard linear probing at α=0.9: expected max DIB ≈ O(n)  (catastrophic).
Robin Hood at α=0.9: expected max DIB ≈ O(log n)  (excellent).

This is the core quantitative advantage of Robin Hood over linear probing.
```

---

## 4. Insertion — The Steal and Displace Cycle

### The Algorithm

```
Insert key k with value v:

1. Compute h = home(k), current DIB = 0.
2. At slot h:
   a. If EMPTY: place k here. Done.
   b. If OCCUPIED with matching key: update value. Done.
   c. If OCCUPIED by key x with DIB(x) < current_DIB:
      → key x is "richer" (smaller DIB). Steal this slot:
        • Swap (k,v,current_DIB) with (x, x_val, DIB(x)).
        • Now (k,v,DIB) is the displaced key x needing a new home.
   d. If OCCUPIED by key x with DIB(x) >= current_DIB:
      → x is "poorer" or equally displaced. Don't steal. Advance.
3. Advance: h = (h + 1) % capacity, current_DIB++.
4. Repeat from step 2.
```

### The Steal Operation — Why It Maintains the Invariant

When we steal a slot from x and continue inserting x at the next slot, we ensure that x's new DIB is exactly one more than what it was. The original occupant of each slot always has a DIB that is at most 1 less than the incoming key's DIB when a steal does not happen.

```
Before steal at slot i:
  incoming: DIB = d
  occupant x: DIB(x) = d' < d

After steal:
  slot i now has incoming with DIB = d (correct: it was displaced d from home)
  x is displaced with DIB = d' (still its original DIB — unchanged)

When x continues to slot i+1:
  If it can be placed at i+1: DIB(x) = d'+1 (waited one extra slot)
  Invariant maintained: DIB in slot i+1 ≤ DIB in slot i + 1. ✓
```

### Example Insertion

```
capacity=8, home(k)=k%8 for simplicity. B=insert operations.

Table initially empty. Insert keys in order: 10, 2, 18, 26

Insert 10: h=2, DIB=0. Slot 2 empty → place.
  Slot: [_][_][10,0][_][_][_][_][_]
              key,DIB

Insert 2: h=2, DIB=0. Slot 2 has 10 (DIB=0). DIB(10)=0 = incoming_DIB=0.
  10 is not richer (not strictly less). Advance to slot 3, DIB=1.
  Slot 3 empty → place 2@3 with DIB=1.
  Slot: [_][_][10,0][2,1][_][_][_][_]

Insert 18: h=2, DIB=0. Slot 2 has 10 (DIB=0). Equal — don't steal.
  Advance to slot 3, DIB=1. Slot 3 has 2 (DIB=1). Equal — don't steal.
  Advance to slot 4, DIB=2. Slot 4 empty → place 18@4 with DIB=2.
  Slot: [_][_][10,0][2,1][18,2][_][_][_]

Insert 26: h=2, DIB=0. Slot 2 has 10 (DIB=0). Equal — advance.
  Slot 3: 2 (DIB=1). DIB(2)=1, incoming=1. Equal — advance.
  Slot 4: 18 (DIB=2). DIB(18)=2, incoming=2. Equal — advance.
  Slot 5: empty → place 26@5 with DIB=3.
  Slot: [_][_][10,0][2,1][18,2][26,3][_][_]

Insert 34 (h=2, should steal!): 
  Slot 2: 10 (DIB=0), incoming DIB=0. Equal — advance.
  Slot 3: 2 (DIB=1), incoming DIB=1. Equal — advance.
  Slot 4: 18 (DIB=2), incoming DIB=2. Equal — advance.
  Slot 5: 26 (DIB=3), incoming DIB=3. Equal — advance.
  Slot 6: empty → place 34@6 with DIB=4.
  Slot: [_][_][10,0][2,1][18,2][26,3][34,4][_]

  Now insert 3 (h=3):
  Slot 3: 2 (DIB=1), incoming DIB=0. DIB(2)=1 > incoming DIB=0.
  → STEAL! Swap: place 3@3 with DIB=0. Continue inserting 2 (DIB=1).
  Slot 4: 18 (DIB=2), incoming=2 (DIB=1). DIB(18)=2 > 1. STEAL!
  Swap: place 2@4 with DIB=1. Continue inserting 18 (DIB=2).
  Wait, this cascade is getting complex. Let me trace carefully:

  Insert 3 (h=3):
  Slot 3: occupant=2 (DIB=1), incoming_DIB=0. 1 > 0 → STEAL.
    Slot 3 ← (3, DIB=0). Now inserting displaced (2, DIB=1).
  Slot 4: occupant=18 (DIB=2), incoming=2 (DIB=1). 2 > 1 → STEAL.
    Slot 4 ← (2, DIB=1). Now inserting displaced (18, DIB=2).
  Slot 5: occupant=26 (DIB=3), incoming=18 (DIB=2). 3 > 2 → STEAL.
    Slot 5 ← (18, DIB=2). Now inserting displaced (26, DIB=3).
  Slot 6: occupant=34 (DIB=4), incoming=26 (DIB=3). 4 > 3 → STEAL.
    Slot 6 ← (26, DIB=3). Now inserting displaced (34, DIB=4).
  Slot 7: empty → place 34@7 with DIB=4.

  Final: [_][_][10,0][3,0][2,1][18,2][26,3][34,4]
  
  Note: 3 found its home (slot 3) by displacing the entire chain!
  Each key's DIB is one more than the previous — the invariant holds.
```

---

## 5. Lookup — The Early-Exit Optimisation

### Standard Lookup

Like all open addressing, lookup probes forward from `home(key)` until either:
1. The key is found (success)
2. An EMPTY slot is reached (key not present)

### The Robin Hood Early-Exit Optimisation

With Robin Hood hashing, lookup can terminate early when the probe encounters a key with **DIB strictly less than the current probe distance**. This is the lookup speedup unique to Robin Hood hashing.

**Intuition:** If the current probe is at distance `d` from the key's home, and we encounter a key with DIB `d' < d`, then:
- If our target key existed in the table, it would have been placed at some slot with DIB ≤ d (by the Robin Hood invariant — richer keys steal slots from poorer ones)
- The key with DIB `d' < d` is "richer" — it would have allowed our target to steal its slot (if our target had DIB > d')
- But our target was not found at any slot with DIB in [0..d-1]
- Therefore, our target does not exist in the table

```cpp
optional<V> lookup(const K& key) const {
    size_t h   = home(key);
    size_t dib = 0;

    while (true) {
        size_t slot = (h + dib) % capacity_;

        if (table_[slot].state == EMPTY) return nullopt;      // end of run

        if (table_[slot].dib < dib) return nullopt;           // ← EARLY EXIT!
        // A slot with smaller DIB means our key would have
        // appeared before this point if it existed.

        if (equal_(table_[slot].key, key)) return table_[slot].val;  // found

        dib++;
    }
}
```

### Why Early-Exit Matters for Unsuccessful Lookups

```
Standard linear probing (unsuccessful lookup):
  Must probe until the first EMPTY slot.
  At α=0.9: expected O(50) probes (the 1/(1-α)² formula).

Robin Hood with early-exit (unsuccessful lookup):
  Can stop when DIB of current slot < probe distance.
  The DIB sequence rises at most +1 per slot.
  Once we probe past the maximum DIB for that probe distance, we know the key
  is absent.
  Expected probes for unsuccessful lookup: much closer to O(1/(1-α)) than O(1/(1-α)²).

At α=0.9:
  Linear probing unsuccessful: E[probes] ≈ 50.5
  Robin Hood unsuccessful: E[probes] ≈ 10.0   (5× faster!)
```

---

## 6. Deletion — Backward Shift Without Tombstones

This is the most elegant property of Robin Hood hashing: it supports O(1) deletion **without tombstones**, using a backward-shift algorithm that naturally maintains the Robin Hood invariant.

### Why Tombstones Are Avoidable

In standard linear probing, deleting a slot requires a tombstone to preserve the probe chain. Robin Hood hashing's invariant provides enough structural information to do better: after deleting a key, we can shift subsequent keys backward to fill the gap, and the DIB values naturally guide which keys should shift and which should not.

### The Backward Shift Algorithm

```
Delete key k at slot i:

1. Mark slot i as EMPTY.
2. Scan forward from slot i+1:
   For each occupied slot j:
     a. If DIB(j) == 0: j is at its home slot. Stop — cannot move it back.
     b. If DIB(j) > 0: j can be moved one slot backward (reducing its DIB by 1).
        Move: slot i ← slot j, decrement moved key's DIB by 1.
        Set slot j to EMPTY. Set i = j. Continue from j+1.
3. Stop when we reach an EMPTY slot or a key with DIB = 0.
```

### Why This Works

The backward shift maintains the Robin Hood invariant:
- Moving a key one slot backward decreases its DIB by 1
- This keeps all DIB values non-negative (DIB ≥ 0 always)
- The moved key now has a DIB one less than before — it is now "richer"
- The invariant `DIB[j] ≤ DIB[j-1] + 1` is preserved because we only move keys with DIB > 0

```
Before delete of key at slot 5 (DIB=0 — it's at its home):
  ... [_][A,DIB=2][B,DIB=0][C,DIB=1][D,DIB=0][E,DIB=0] ...
       3     4       5        6        7        8

Delete E at slot 8 (DIB=0):
  Slot 8 ← EMPTY.
  j=9: whatever is there...
  (Assume slot 9 has F with DIB=1. We can move it to slot 8.)
  Move F to slot 8 with DIB=0. Slot 9 ← EMPTY.
  j=10: if slot 10 has G with DIB=2. Move G to slot 9 with DIB=1.
  Continue...

Delete B at slot 5 (DIB=0 — wait, it's at HOME! Cannot shift anything before it.):
  Actually, B is at its home. Deleting B:
  Slot 5 ← EMPTY.
  Scan j=6 (C, DIB=1): DIB>0 → move C to slot 5 with DIB=0.
    Slot 5 ← (C, DIB=0). Slot 6 ← EMPTY. Now i=6.
  Scan j=7 (D, DIB=0): DIB=0 → STOP (D is at its home, cannot move back).
  
  Result: [_][A,2][C,0][_][D,0][E,0]...

  Verify: C's original home was h(C) = ? If C moved from slot 6 to slot 5,
  and DIB went from 1 to 0, then h(C)=5. ✓ (C's home is slot 5, now occupied correctly.)
```

### Tombstone Comparison

```
Tombstone approach: mark deleted slots with DELETED sentinel.
  Pro: simple to implement.
  Con: tombstones accumulate → eventually the table is full of tombstones.
       Unsuccessful lookup must probe past all tombstones → O(capacity) worst case.
       Requires periodic rehash to clean tombstones.

Backward shift: physically move keys to fill gaps.
  Pro: NO tombstones. Table always in clean state.
       Unsuccessful lookup benefits from early-exit on DIB=0 check.
  Con: deletion is O(probe_length) instead of O(1) — but probe_length is O(1) avg.
       Keys physically move → external iterators/references are invalidated.
  Used in: Rust HashMap, Abseil's flat_hash_map (conceptually).
```

---

## 7. Variance Reduction — Why Robin Hood Is Better Than Linear Probing

### The Mathematical Intuition

Standard linear probing places keys in the first available slot, creating a "rich get richer" dynamic: long runs attract more insertions, making them even longer. The variance of probe lengths grows as O(1/(1-α)²).

Robin Hood hashing enforces a *levelling* rule: no key can be more than 1 unit more displaced than the key in the slot before it. This is fundamentally a variance-reduction algorithm — the DIB distribution is forced to be as uniform as possible given the hash function.

### The DIB Distribution

```
Standard linear probing (α=0.8):
  E[probe_length]         ≈ 3.0
  Var[probe_length]       ≈ 50    ← high variance
  Max probe length (expected) ≈ 50+

Robin Hood hashing (α=0.8):
  E[probe_length]         ≈ 3.0   ← same average!
  Var[probe_length]       ≈ 2.8   ← dramatically lower variance
  Max probe length (expected) ≈ O(log n) ← dramatically better

The average is unchanged but the distribution is tight.
Think of it as: same total probe work, but distributed fairly across all keys.
```

### The "Max DIB" Bound

One of the strongest results about Robin Hood hashing:

```
Theorem (Viola, 2018 — informally stated):
For Robin Hood hashing with uniform hash functions and load factor α:
  E[max DIB] = O(log(n/(1-α)))
  
For comparison, standard linear probing:
  E[max probe chain length] = Θ(log²(n/(1-α))) ... actually much worse in practice.
  
At α=0.9, n=10^6:
  Robin Hood: E[max DIB] ≈ O(log(10^7)) ≈ 23
  Linear probing: E[max chain] ≈ O(n^(1/3)) in some analyses — much larger.

Practical consequence: caching is predictable.
  With max DIB ≈ 23, you know the worst-case lookup NEVER exceeds 23 probes.
  Linear probing: worst case is unbounded without knowing the key distribution.
```

---

## 8. Time & Space Complexity

### Operation Complexities

| Operation | Average | Worst Case | Notes |
|---|---|---|---|
| `lookup(k)` successful | **O(1)** | O(n) | E[probes] = ~1/(1-α) |
| `lookup(k)` unsuccessful | **O(1)** | O(n) | Early-exit: much better than linear probing |
| `insert(k, v)` | **O(1) amortised** | O(n) | E[probes] = ~1/(1-α); rare long chains |
| `delete(k)` | **O(1) amortised** | O(n) | Backward shift; no tombstones |
| `rehash(2m)` | O(n) | O(n) | Re-insert all entries; rare |
| Build from n | O(n) avg | O(n²) | n insertions |
| Iteration | O(m) | O(m) | Scan all m slots |

### The Key Variance Advantage

```
Metric              Standard LP   Robin Hood   Robin Hood Advantage
──────────────────────────────────────────────────────────────────
E[success probe]        2.5           2.5           Same at α=0.5
E[unsuccess probe]      2.5           1.9           ~25% faster
Var[probe length]      HIGH         LOW             Main advantage
Max probe (typical)  O(log²n)     O(log n)          Much lower
Unsuccessful worst   O(n)         O(log n)          Dramatic improvement

All at α = 0.5. Robin Hood dominates as α increases.
```

### Space Complexity

| Component | Space | Notes |
|---|---|---|
| Slot array | O(m) | One contiguous block |
| Per-entry overhead | **1 DIB value** (1-4 bytes) | Only overhead vs vanilla linear probing |
| Total | O(n/α) | Same asymptotic as linear probing |
| vs chaining | Chaining: +8 bytes/entry (pointer) | Robin Hood uses less |
| vs linear probing | Linear: 0 bytes/entry overhead | Robin Hood uses 1-4 bytes more |

---

## 9. Complete C++ Implementation

```cpp
#include <iostream>
#include <vector>
#include <optional>
#include <functional>
#include <stdexcept>
#include <algorithm>
#include <cmath>
using namespace std;

template<typename K, typename V,
         typename Hash  = hash<K>,
         typename Equal = equal_to<K>>
class RobinHoodMap {
private:
    // ── Slot ──────────────────────────────────────────────────────────────────
    // DIB=-1 means EMPTY. DIB≥0 means OCCUPIED.
    // No DELETED state — we use backward shift for deletion.
    static const int EMPTY_DIB = -1;

    struct Slot {
        K   key;
        V   val;
        int dib = EMPTY_DIB;   // distance from initial bucket; -1 = empty

        bool empty()    const { return dib == EMPTY_DIB; }
        bool occupied() const { return dib >= 0; }
    };

    // ── Data ──────────────────────────────────────────────────────────────────
    vector<Slot> table_;
    size_t       size_;
    size_t       capacity_;
    float        max_load_;
    Hash         hasher_;
    Equal        equal_;

    // ── Utilities ─────────────────────────────────────────────────────────────

    size_t home(const K& key) const {
        return hasher_(key) % capacity_;
    }

    bool needs_rehash() const {
        return (float)size_ / capacity_ > max_load_;
    }

    void do_rehash(size_t new_cap) {
        vector<Slot> old = move(table_);
        table_.assign(new_cap, Slot{});
        capacity_ = new_cap;
        size_     = 0;

        for (auto& s : old) {
            if (s.occupied()) insert(move(s.key), move(s.val));
        }
    }

    // ── Core insert (no rehash trigger) ───────────────────────────────────────
    // Implements the Robin Hood steal-and-displace cycle.
    void insert_internal(K key, V val) {
        size_t slot = home(key);
        int    dib  = 0;

        while (true) {
            Slot& s = table_[slot % capacity_];

            // Empty slot: place here
            if (s.empty()) {
                s = {move(key), move(val), dib};
                size_++;
                return;
            }

            // Existing key: update value
            if (equal_(s.key, key)) {
                s.val = move(val);
                return;
            }

            // Robin Hood: steal if current occupant is richer (smaller DIB)
            if (s.dib < dib) {
                swap(key, s.key);
                swap(val, s.val);
                swap(dib, s.dib);
                // Now (key, val, dib) is the displaced occupant. Continue.
            }

            slot++;
            dib++;
        }
    }

public:
    // ── Constructors ──────────────────────────────────────────────────────────

    explicit RobinHoodMap(size_t initial_cap = 16, float max_load = 0.875f)
        : table_(initial_cap, Slot{}),
          size_(0), capacity_(initial_cap), max_load_(max_load) {
        // 0.875 = 7/8: the threshold used by Rust's HashMap and many
        // Robin Hood implementations. High utilisation with low variance.
    }

    ~RobinHoodMap() = default;

    RobinHoodMap(const RobinHoodMap&)            = delete;
    RobinHoodMap& operator=(const RobinHoodMap&) = delete;
    RobinHoodMap(RobinHoodMap&&)                 = default;
    RobinHoodMap& operator=(RobinHoodMap&&)      = default;

    // ── Insert — O(1) amortised ───────────────────────────────────────────────

    void insert(const K& key, const V& val) {
        if (needs_rehash()) do_rehash(capacity_ * 2);
        insert_internal(key, val);
    }

    void insert(K&& key, V&& val) {
        if (needs_rehash()) do_rehash(capacity_ * 2);
        insert_internal(move(key), move(val));
    }

    V& operator[](const K& key) {
        if (!contains(key)) insert(key, V{});
        return at(key);
    }

    // ── Lookup — O(1) with early-exit ─────────────────────────────────────────

    optional<V> get(const K& key) const {
        size_t slot = home(key);
        int    dib  = 0;

        while (true) {
            const Slot& s = table_[slot % capacity_];

            if (s.empty())      return nullopt;   // end of probe run
            if (s.dib < dib)    return nullopt;   // early exit: key would be here if present
            if (equal_(s.key, key)) return s.val; // found

            slot++;
            dib++;
        }
    }

    bool contains(const K& key) const {
        return get(key).has_value();
    }

    V& at(const K& key) {
        size_t slot = home(key);
        int    dib  = 0;
        while (true) {
            Slot& s = table_[slot % capacity_];
            if (s.empty() || s.dib < dib) throw out_of_range("key not found");
            if (equal_(s.key, key))        return s.val;
            slot++; dib++;
        }
    }

    const V& at(const K& key) const {
        size_t slot = home(key);
        int    dib  = 0;
        while (true) {
            const Slot& s = table_[slot % capacity_];
            if (s.empty() || s.dib < dib) throw out_of_range("key not found");
            if (equal_(s.key, key))        return s.val;
            slot++; dib++;
        }
    }

    // ── Delete — backward shift, no tombstones ────────────────────────────────

    bool erase(const K& key) {
        // First: find the key
        size_t slot = home(key);
        int    dib  = 0;

        while (true) {
            Slot& s = table_[slot % capacity_];
            if (s.empty() || s.dib < dib) return false;  // not found
            if (equal_(s.key, key)) break;
            slot++; dib++;
        }

        // `slot` now points to the key to delete.
        // Backward shift: move subsequent keys one slot back.
        size_t del_slot = slot % capacity_;
        table_[del_slot].dib = EMPTY_DIB;  // mark as empty
        size_--;

        // Shift forward entries backward while their DIB > 0
        while (true) {
            size_t next_slot = (del_slot + 1) % capacity_;
            Slot& next = table_[next_slot];

            if (next.empty() || next.dib == 0) break;  // stop condition

            // Move next entry one slot back (DIB decreases by 1)
            table_[del_slot] = {move(next.key), move(next.val), next.dib - 1};
            next.dib = EMPTY_DIB;

            del_slot = next_slot;
        }

        return true;
    }

    // ── Utility ───────────────────────────────────────────────────────────────

    size_t size()        const { return size_; }
    bool   empty()       const { return size_ == 0; }
    size_t capacity()    const { return capacity_; }
    float  load_factor() const { return (float)size_ / capacity_; }

    void clear() {
        for (auto& s : table_) s.dib = EMPTY_DIB;
        size_ = 0;
    }

    void reserve(size_t n) {
        size_t needed = (size_t)ceil((float)n / max_load_);
        if (needed > capacity_) do_rehash(needed);
    }

    // ── Statistics ────────────────────────────────────────────────────────────

    void print_dib_stats() const {
        int max_dib = 0, total_dib = 0, occupied = 0;
        vector<int> dib_hist(32, 0);

        for (const auto& s : table_) {
            if (!s.occupied()) continue;
            occupied++;
            total_dib += s.dib;
            max_dib = max(max_dib, s.dib);
            if (s.dib < 32) dib_hist[s.dib]++;
        }

        cout << "size=" << size_ << " load=" << load_factor()
             << " avg_dib=" << (occupied ? (float)total_dib/occupied : 0)
             << " max_dib=" << max_dib << "\n";
        cout << "DIB histogram: ";
        for (int i = 0; i <= min(max_dib, 15); i++) {
            cout << "dib[" << i << "]=" << dib_hist[i] << " ";
        }
        cout << "\n";
    }

    void print() const {
        for (size_t i = 0; i < capacity_; i++) {
            const Slot& s = table_[i];
            if (s.empty()) { cout << "[" << i << "] EMPTY\n"; continue; }
            cout << "[" << i << "] key=" << s.key
                 << " val=" << s.val << " dib=" << s.dib << "\n";
        }
    }

    // ── Iteration ─────────────────────────────────────────────────────────────

    class Iterator {
        const RobinHoodMap* map_;
        size_t              idx_;
        void advance() {
            while (idx_ < map_->capacity_ && map_->table_[idx_].empty()) ++idx_;
        }
    public:
        Iterator(const RobinHoodMap* m, size_t i) : map_(m), idx_(i) { advance(); }
        pair<const K&, const V&> operator*() const {
            return {map_->table_[idx_].key, map_->table_[idx_].val};
        }
        Iterator& operator++() { ++idx_; advance(); return *this; }
        bool operator!=(const Iterator& o) const { return idx_ != o.idx_; }
    };

    Iterator begin() const { return {this, 0}; }
    Iterator end()   const { return {this, capacity_}; }
};


// ── Robin Hood Set (keys only) ────────────────────────────────────────────────
template<typename K, typename Hash=hash<K>, typename Equal=equal_to<K>>
class RobinHoodSet {
    RobinHoodMap<K, bool, Hash, Equal> impl_;
public:
    explicit RobinHoodSet(size_t cap=16) : impl_(cap) {}
    void   insert(const K& k) { impl_.insert(k, true); }
    bool   contains(const K& k) const { return impl_.contains(k); }
    bool   erase(const K& k) { return impl_.erase(k); }
    size_t size()  const { return impl_.size(); }
    bool   empty() const { return impl_.empty(); }
};
```

---

## 10. Core Operations — Visualised

### Insert with Steal — Full Trace

```
capacity=8, max_load=0.875, table initially empty.
h(k) = k % 8 for illustration.

Insert 8 (h=0, dib starts at 0):
  Slot 0: EMPTY → place (8, dib=0).
  Table: [(8,0),_,_,_,_,_,_,_]

Insert 3 (h=3, dib=0):
  Slot 3: EMPTY → place (3, dib=0).
  Table: [(8,0),_,_,(3,0),_,_,_,_]

Insert 16 (h=0, dib=0):
  Slot 0: (8, dib=0). Incoming dib=0. NOT richer (0 not < 0). Advance.
  Slot 1: EMPTY → place (16, dib=1).
  Table: [(8,0),(16,1),_,(3,0),_,_,_,_]

Insert 24 (h=0, dib=0):
  Slot 0: (8, dib=0). dib=0, not richer. Advance. dib=1.
  Slot 1: (16, dib=1). dib=1, not richer. Advance. dib=2.
  Slot 2: EMPTY → place (24, dib=2).
  Table: [(8,0),(16,1),(24,2),(3,0),_,_,_,_]

Insert 1 (h=1, dib=0):
  Slot 1: (16, dib=1). Incoming dib=0. 1 is richer (1 > 0)? NO — we steal if
          occupant is richer (smaller dib). Occupant dib=1, incoming dib=0.
          incoming_dib(0) < occupant_dib(1)? NO — we steal when incoming > occupant's dib.
          Actually: steal when s.dib < incoming_dib. Here s.dib=1, incoming_dib=0. 1 < 0? NO.
          Don't steal. Advance.
  Slot 2: (24, dib=2). s.dib=2, incoming_dib=1. 2 < 1? NO. Advance.
  Slot 3: (3, dib=0). s.dib=0, incoming_dib=2. 0 < 2? YES → STEAL!
    Swap: slot 3 ← (1, dib=2). Continue placing displaced (3, dib=0).
  Slot 4: EMPTY → place (3, dib=0). But wait: 3 was at home (slot 3), now dib would be 1...

  Actually: the displaced key's dib is preserved from before the steal,
  and it continues probing from the NEXT slot after where it was stolen from.
  
  Let me re-trace. When we steal slot 3 for key 1 (incoming_dib=2):
    slot 3 ← key 1 with dib=2. ✓ (1's home=1, now at slot 3 → dib=2 ✓)
    Displaced key 3 had dib=0 (at its home). Now it needs a new slot.
    But we continue from slot 4 (the next slot after 3) with displaced key's ORIGINAL dib.
    Displaced key 3's dib was 0. From slot 4, its new dib should be 1 (one slot past its home=3).
    
  This is the subtlety: the displaced key's new dib at its insertion point is:
    new_dib = (new_slot - h(displaced_key)) % capacity = (4 - 3) = 1.
    
  Continue: slot 4 EMPTY → place (3, dib=1).
  
  Table: [(8,0),(16,1),(24,2),(1,2),(3,1),_,_,_]
  
  Note: key 1 is at slot 3 with dib=2 (h(1)=1, placed at 3, correct: 3-1=2 ✓).
        key 3 is at slot 4 with dib=1 (h(3)=3, placed at 4, correct: 4-3=1 ✓).
```

### Lookup with Early-Exit

```
Table from above: [(8,0),(16,1),(24,2),(1,2),(3,1),_,_,_]

Lookup key 11 (h=3, not in table):
  Slot 3: key=1, dib=2. Probe dib=0. s.dib(2) < probe_dib(0)? NO. 1≠11. Advance.
  Slot 4: key=3, dib=1. Probe dib=1. s.dib(1) < probe_dib(1)? NO. 3≠11. Advance.
  Slot 5: EMPTY. Return "not found." ✓

Lookup key 5 (h=5, not in table):
  Slot 5: EMPTY. Return "not found." ✓ (1 probe — immediate!)

Lookup key 2 (h=2, not in table):
  Slot 2: key=24, dib=2. Probe dib=0. s.dib(2) < probe_dib(0)? NO. 24≠2. Advance.
  Slot 3: key=1, dib=2. Probe dib=1. s.dib(2) < probe_dib(1)? NO. 1≠2. Advance.
  Slot 4: key=3, dib=1. Probe dib=2. s.dib(1) < probe_dib(2)? YES → EARLY EXIT!
  Return "not found." ✓  (3 probes vs potentially many more with standard LP)

The early exit fired because key 3 (dib=1) is "richer" than expected at
probe distance 2. This signals that key 2 (which would need dib≥2 to be here)
cannot exist in the table — it would have stolen a slot from key 3.
```

### Deletion with Backward Shift

```
Table: [(8,0),(16,1),(24,2),(1,2),(3,1),_,_,_]

Delete key 16 (at slot 1, dib=1):
  Mark slot 1 as EMPTY: [_,_,_,...]
  Now: [(8,0),EMPTY,(24,2),(1,2),(3,1),_,_,_]

Backward shift from slot 2:
  j=2: key=24, dib=2. dib > 0 → shift back.
    Slot 1 ← (24, dib=1). Slot 2 ← EMPTY. del_slot=2.
    [(8,0),(24,1),EMPTY,(1,2),(3,1),_,_,_]

  j=3: key=1, dib=2. dib > 0 → shift back.
    Slot 2 ← (1, dib=1). Slot 3 ← EMPTY. del_slot=3.
    [(8,0),(24,1),(1,1),EMPTY,(3,1),_,_,_]

  j=4: key=3, dib=1. dib > 0 → shift back.
    Slot 3 ← (3, dib=0). Slot 4 ← EMPTY. del_slot=4.
    [(8,0),(24,1),(1,1),(3,0),EMPTY,_,_,_]

  j=5: EMPTY → STOP.

Final table: [(8,0),(24,1),(1,1),(3,0),EMPTY,_,_,_]

Verify:
  8 at slot 0, h(8)=0. DIB = 0-0 = 0. ✓
  24 at slot 1, h(24)=0. DIB = 1-0 = 1. ✓
  1 at slot 2, h(1)=1. DIB = 2-1 = 1. ✓
  3 at slot 3, h(3)=3. DIB = 3-3 = 0. ✓

All lookups still work: lookup 24 → slot 0(8≠24,dib=0=probe_dib), slot 1 found ✓
No tombstones anywhere. Table stays clean. ✓
```

---

## 11. Interview Problems

### Problem 1: Implement Robin Hood Hash Map

**Problem:** Implement a hash map with Robin Hood hashing. Demonstrate that the maximum probe length is O(log n) compared to O(n) for standard linear probing by printing DIB statistics.

```cpp
int main() {
    RobinHoodMap<int,int> m(32, 0.8f);

    // Insert many keys that hash to the same region
    mt19937 rng(42);
    for (int i = 0; i < 20; i++) m.insert(i * 8, i);  // all h=0 with cap=8 subsets

    cout << "After 20 insertions (many colliding):\n";
    m.print_dib_stats();

    // Compare: what would linear probing look like?
    // LP would have a run of 20 consecutive occupied slots, max DIB = 19.
    // Robin Hood should have max DIB ≈ log(20) ≈ 4-5.

    // Verify all lookups
    bool all_found = true;
    for (int i = 0; i < 20; i++) {
        auto v = m.get(i * 8);
        if (!v || *v != i) { all_found = false; break; }
    }
    cout << "All lookups correct: " << (all_found ? "yes" : "NO") << "\n";

    // Verify deletions
    for (int i = 0; i < 10; i++) m.erase(i * 8);
    cout << "After 10 deletions:\n";
    m.print_dib_stats();
    cout << "Remaining size: " << m.size() << " (expected 10)\n";

    return 0;
}
```

---

### Problem 2: Design a Cache with Bounded Worst-Case Lookup

**Problem (System Design):** You are designing an in-memory LRU cache for a high-frequency trading system. The cache must:
- Handle 10 million lookups per second
- Have bounded worst-case lookup latency (never exceed 100ns per lookup)
- Support insertions and evictions

Explain why Robin Hood hashing is more appropriate than standard linear probing for this use case.

```cpp
/*
Analysis:

Standard linear probing at α=0.7:
  E[lookup] ≈ 2.17 probes — fine for average case.
  Max probe chain: O(log²n) expected — but can spike to O(n) in bad cases.
  At 10M lookups/sec: a single O(n) lookup at n=1M would take ~1ms = 10,000 deadlines missed.
  P(any lookup exceeds 100 probes) is non-negligible at high load.

Robin Hood hashing at α=0.875:
  E[lookup] ≈ 2.5 probes — slightly worse average (more compaction).
  Max probe chain: O(log n) = O(20) for n=1M — bounded!
  At 10M lookups/sec: max 20 probes × ~5ns/probe = 100ns — meets the deadline.
  The bounded max DIB is the critical property for latency SLAs.
*/

class HFTCache {
private:
    struct CacheEntry {
        string  instrument_id;
        double  last_price;
        int64_t timestamp_ns;
    };

    // Robin Hood with high load factor for memory efficiency
    RobinHoodMap<string, CacheEntry> table_{65536, 0.875f};

    // LRU eviction: doubly-linked list ordered by access time
    // (abbreviated for clarity — full LRU requires DLL + hash map)
    struct LRUNode {
        string   key;
        LRUNode *prev, *next;
    };
    LRUNode* lru_head_;  // least recently used
    LRUNode* lru_tail_;  // most recently used

public:
    // O(1) amortised — bounded worst case
    optional<CacheEntry> get(const string& instrument_id) {
        auto entry = table_.get(instrument_id);
        if (entry) {
            // Promote to MRU position in LRU list
            // (O(1) with pointer update — same as standard LRU cache)
        }
        return entry;
    }

    void put(const string& instrument_id, const CacheEntry& entry) {
        table_.insert(instrument_id, entry);
    }
};

/*
Why Robin Hood over cuckoo for this use case?

Cuckoo hashing: O(1) HARD worst-case lookup, but:
  - 2× memory overhead (2 tables)
  - Insert can trigger O(n) eviction chains
  - In HFT: burst insertions during market open could cause latency spikes

Robin Hood at α=0.875:
  - O(log n) SOFT worst-case (not as strong as cuckoo, but sufficient)
  - 1× memory overhead
  - Insertions are O(1) amortised with small constant
  - DIB distribution is tightly bounded in practice
  - Better fit for mixed read/write workload

The choice: Robin Hood wins when memory matters and burst insertions exist.
            Cuckoo wins when O(1) HARD lookup guarantee is non-negotiable.
*/
```

---

### Problem 3: Find the Maximum DIB After n Insertions

**Problem:** Given a sequence of integer keys inserted into a Robin Hood hash table of capacity m (using h(k) = k % m), compute the maximum DIB across all stored keys after all insertions. This tests understanding of the steal mechanism.

```cpp
// Time: O(n × avg_probe) ≈ O(n) | Space: O(m)
int maxDIBAfterInsertions(vector<int>& keys, int m) {
    // Use DIB=-1 for EMPTY
    vector<int> dib(m, -1);
    vector<int> stored_keys(m, -1);

    auto home_fn = [&](int k) { return ((k % m) + m) % m; };

    for (int key : keys) {
        int slot = home_fn(key);
        int cur_dib = 0;
        int cur_key = key;

        while (true) {
            int s = slot % m;
            if (dib[s] == -1) {
                // Empty: place here
                dib[s]         = cur_dib;
                stored_keys[s] = cur_key;
                break;
            }
            if (stored_keys[s] == cur_key) {
                // Duplicate: skip (or update)
                break;
            }
            if (dib[s] < cur_dib) {
                // Robin Hood steal
                swap(cur_key, stored_keys[s]);
                swap(cur_dib, dib[s]);
            }
            slot++;
            cur_dib++;
        }
    }

    return *max_element(dib.begin(), dib.end());
}

/*
Test case: m=8, keys=[0,8,16,24,32] (all hash to slot 0)

Insert 0:  slot 0, dib=0. Place. dib=[0,-1,-1,-1,-1,-1,-1,-1]
Insert 8:  slot 0 (dib=0), cur_dib=0. Not richer. Advance.
           slot 1, dib=1 (empty). Place. dib=[0,1,-1,-1,-1,-1,-1,-1]
Insert 16: slots 0,1 occupied (dibs 0,1), cur_dib reaches 0,1,2.
           Slot 2 empty (dib_cur=2). Place. dib=[0,1,2,-1,-1,-1,-1,-1]
Insert 24: similar, slot 3, dib=3. dib=[0,1,2,3,-1,-1,-1,-1]
Insert 32: slot 4, dib=4. dib=[0,1,2,3,4,-1,-1,-1]

maxDIB = 4.

Now insert 1 (h=1):
  Slot 1: stored=8 (dib=1). cur_dib=0. 1 < 0? NO. Advance. cur_dib=1.
  Slot 2: stored=16 (dib=2). cur_dib=1. 2 < 1? NO. Advance. cur_dib=2.
  Slot 3: stored=24 (dib=3). cur_dib=2. 3 < 2? NO. Advance. cur_dib=3.
  Slot 4: stored=32 (dib=4). cur_dib=3. 4 < 3? NO. Advance. cur_dib=4.
  Slot 5: EMPTY. Place 1 with dib=4.

Hmm, key 1 didn't steal anything. Let me insert key 2 (h=2):
  Slot 2: (16, dib=2), cur_dib=0. 2 < 0? NO. Advance.
  Slot 3: (24, dib=3), cur_dib=1. 3 < 1? NO. Advance.
  Slot 4: (32, dib=4), cur_dib=2. 4 < 2? NO. Advance.
  Slot 5: (1, dib=4), cur_dib=3. 4 < 3? NO. Advance.
  Slot 6: EMPTY. Place 2 with dib=4.

All 5 keys from h=0 have max DIB=4 = n-1. Without Robin Hood this is same.
Robin Hood advantage: when you mix keys from different home slots,
the stealing redistributes displacement fairly. Try keys=[0,1,2,3,4,5]:
  With LP, keys at positions 0-5 all hash to different slots, no collisions → max DIB=0.
  With LP, keys=[0,8,16,2] (h: 0,0,0,2):
    LP:   0@0, 8@1, 16@2, 2@? → h=2, slot 2 occupied (16), slot 3 empty → 2@3, dib=1.
    RH:   0@0, 8@1, 16@2, 2@? → h=2, slot 2 (16,dib=2), cur_dib=0. 2<0? No.
          slot 3 empty → 2@3, dib=1. Same result for small tables.
    
  The real benefit shows up at scale with many collisions and mixed home slots.
*/
```

---

## 12. Real-World Uses

| Domain | System | Robin Hood Detail |
|---|---|---|
| **Rust standard library** | `std::collections::HashMap` | Robin Hood hashing since Rust 1.0 (since 1.36: SwissTable) |
| **Python** | CPython dict (conceptual similarity) | Modern Python uses a compact dict with related ideas |
| **C++ libraries** | Abseil `flat_hash_map` (predecessor) | Abseil's first design used Robin Hood; now uses SwissTable |
| **Game engines** | Various ECS component systems | Hash map with bounded worst-case for frame-rate stability |
| **Databases** | Lock-free hash tables in NVM DBs | Robin Hood's no-tombstone deletion eases recovery |
| **Research** | Hopscotch hashing (successor concept) | Extends Robin Hood with neighbourhood guarantees |
| **Competitive programming** | Custom hash map implementations | Faster than std::unordered_map for most workloads |
| **Embedded systems** | No-alloc hash maps in firmware | Robin Hood avoids tombstone accumulation in memory-constrained systems |
| **JVM** | OpenJDK escape analysis tables | Bounded DIB useful for predictable GC pause times |

### Rust's HashMap — Robin Hood in Production

Until Rust 1.36, `std::collections::HashMap` used Robin Hood hashing. The implementation was a significant performance milestone in hash table engineering:

```rust
// Rust HashMap (pre-1.36) key properties:
// - Robin Hood open addressing
// - Load factor: 90.9% (11/12 — extremely aggressive!)
// - Backward-shift deletion: no tombstones
// - SipHash-1-3 by default (cryptographic — slower but DoS-resistant)
// - Power-of-2 capacity with bitwise AND for fast modulo
// - DIB stored as u32 (can hold larger values than u8 for safety)

// The 90.9% load factor was possible because Robin Hood's variance
// reduction keeps max DIB bounded even at very high loads.
// Standard linear probing would be catastrophically slow at 90%.

// Example lookup probe distribution at α = 0.9 with Robin Hood:
//   DIB=0: ~10% of entries (at home)
//   DIB=1: ~18% of entries
//   DIB=2: ~16% of entries
//   DIB=3: ~14% of entries
//   DIB=4: ~12% of entries
//   DIB≥5: ~30% of entries (long tail, but bounded)
//   Max DIB in practice: 20-30 for a million entries
//
// Compare to linear probing at α = 0.5 (where it's safe):
//   Same distribution but at HALF the load! Robin Hood is 2× more memory-efficient.
```

### The Transition to Swiss Table

In 2019, Rust's HashMap switched from Robin Hood to Google's Swiss Table implementation. This transition illuminates the trade-offs:

```
Robin Hood hashing (Rust pre-1.36):
  + Simple implementation
  + No tombstones
  + Bounded max DIB
  + Works for arbitrary load factors
  - Lookup requires comparing full keys on each probe
  - DIB check adds a comparison per probe

Swiss Table (Rust 1.36+):
  + SIMD: check 16 slots in ONE CPU instruction using 7-bit hash metadata
  + Faster lookup despite same algorithmic complexity
  + Better cache utilisation (metadata bytes separate from key-value data)
  - More complex implementation
  - Requires SIMD hardware (x86, ARM)
  - Uses tombstones (with metadata byte encoding)

The lesson: Robin Hood's algorithmic elegance was superseded by
hardware-specific optimisation. Swiss Table's 7-bit metadata per slot
performs a Robin Hood-like "skip if hash prefix doesn't match" using SIMD,
achieving the same early-exit benefit with 16× parallelism.
```

---

## 13. Edge Cases & Pitfalls

### Pitfall 1: Wrong Steal Condition

```cpp
// Robin Hood steals when the INCOMING key is MORE DISPLACED than the OCCUPANT.
// In other words: steal when occupant.dib < incoming_dib.

// WRONG: steal when incoming_dib <= occupant.dib (includes equal case)
if (s.dib <= dib) {  // ← wrong: steals from equally-displaced keys too
    swap(key, s.key);
    swap(val, s.val);
    swap(dib, s.dib);
}
// Effect: creates unnecessary displacement, potentially worse clustering.

// WRONG: steal when incoming_dib > occupant.dib (correct direction but wrong sign)
if (dib > s.dib) {  // ← this IS correct actually!
    // Same as s.dib < dib — take this slot from the richer occupant
}

// CORRECT: steal when occupant is strictly richer
if (s.dib < dib) {   // occupant's DIB < incoming's DIB → occupant is richer
    swap(key, s.key);
    swap(val, s.val);
    swap(dib, s.dib);
    // Continue: now 'key' is the displaced occupant needing a new home
}
// The convention on equal DIBs (don't steal) is standard.
// Stealing on equal DIBs doesn't improve variance and adds unnecessary work.
```

### Pitfall 2: Backward Shift Not Stopping at DIB=0

```cpp
// WRONG: shifting past DIB=0 entries
size_t del = idx;
while (true) {
    size_t next = (del + 1) % cap;
    if (table[next].empty()) break;
    table[del] = {table[next].key, table[next].val, table[next].dib - 1};
    table[next].dib = EMPTY_DIB;
    del = next;
}
// BUG: when table[next].dib == 0, we would set dib=-1 (EMPTY_DIB) incorrectly.
// A key with dib=0 is at its HOME — moving it back would give dib=-1 which
// means "moved before my home" — impossible. Must stop here.

// CORRECT: stop when next entry has dib == 0 (at home — cannot move back)
while (true) {
    size_t next = (del + 1) % cap;
    if (table[next].empty() || table[next].dib == 0) break;  // ← stop at dib=0
    table[del] = {table[next].key, table[next].val, table[next].dib - 1};
    table[next].dib = EMPTY_DIB;
    del = next;
}
```

### Pitfall 3: Wrap-Around in DIB Computation

```cpp
// When slot index wraps around (e.g., capacity=8, slot 7 + 1 = slot 0):
// The DIB computation must account for the circular nature.

// WRONG: DIB computation without wrap-around
int dib = slot - home(key);  // negative if slot < home (after wrap)!

// Example: capacity=8, key with h=7, placed at slot 1 (after wrapping past 7→0→1):
// DIB = (1 - 7) = -6. Wrong!

// CORRECT: modular DIB
int dib = (slot - home(key) + capacity) % capacity;
// (1 - 7 + 8) % 8 = 2 % 8 = 2. Correct: key probed slots 7, 0, 1 (2 probes).

// In the implementation: track dib as a running counter starting from 0,
// incrementing on each probe step. This avoids the subtraction entirely.
// Only compute (slot - home(key) + capacity) % capacity for verification.
```

### Pitfall 4: External References Invalidated by Backward Shift

```cpp
// WRONG: holding a reference to a value across a deletion
auto& val_ref = m.at("key1");  // reference to value in table
m.erase("key2");               // backward shift may MOVE "key1" to a new slot!
cout << val_ref;               // UNDEFINED BEHAVIOUR: reference may dangle

// Robin Hood deletion physically moves entries. Any reference or pointer
// obtained from at(), get(), or operator[] is invalidated by ANY modification.
// This is the same as std::vector's iterator invalidation rule.

// CORRECT: always re-lookup after any modification
m.erase("key2");
auto v = m.get("key1");   // fresh lookup after deletion
if (v) cout << *v;
```

### Pitfall 5: DIB Overflow for Very High Load

```cpp
// If DIB is stored as a uint8_t (max value 255):
// At load factor 0.9 with n=10^6 keys, max DIB is expected ~23.
// But with a poor hash function or adversarial input, DIB can exceed 255!

// WRONG: using uint8_t for DIB with no overflow check
struct Slot {
    K        key;
    V        val;
    uint8_t  dib;   // MAX 255 — could overflow for very long chains
};
// If dib wraps to 0 when it should be 256: the Robin Hood invariant breaks,
// causing incorrect steal decisions and potentially infinite loops.

// CORRECT option 1: use int32_t for DIB (safe for any practical table size)
struct Slot {
    K       key;
    V       val;
    int32_t dib = -1;   // -1 = empty; max realistic value < 100
};

// CORRECT option 2: use uint8_t but cap DIB and rehash if exceeded
if (dib >= 255) { do_rehash(capacity_ * 2); return insert_internal(key, val); }
```

### Pitfall 6: Handling Duplicate Keys During Insertion

```cpp
// During the steal loop, the key being inserted might already exist in the table.
// If we don't check for this, we insert a duplicate and corrupt the table.

// WRONG: no duplicate check in the steal loop
void insert_bad(K key, V val) {
    size_t slot = home(key);
    int dib = 0;
    while (true) {
        if (table_[slot].empty()) { place(key, val, dib); return; }
        if (table_[slot].dib < dib) { steal_and_continue(); }
        slot++; dib++;
    }
    // NEVER checks if table_[slot].key == key!
    // Result: the same key appears twice in different slots.
}

// CORRECT: check for existing key BEFORE stealing or advancing
while (true) {
    Slot& s = table_[slot % capacity_];
    if (s.empty()) { place here; return; }
    if (equal_(s.key, key)) { s.val = val; return; }  // ← check before steal!
    if (s.dib < dib) { steal; }
    slot++; dib++;
}
```

### Pitfall 7: Power-of-Two Capacity Without Bit Mask

```cpp
// For power-of-2 capacity, % can be replaced with & (mask):
// slot % capacity == slot & (capacity - 1)  when capacity is power of 2.

// WRONG: using % with signed integers
int slot = (home(key) - 1);   // might be negative if we go back
slot = slot % capacity_;       // C++ % on negative = negative result!

// CORRECT: use size_t (unsigned) for slot indices to avoid negative modulo
size_t slot = home(key);
slot = (slot + 1) & (capacity_ - 1);   // wraps correctly for power-of-2
```

---

## 14. Robin Hood vs Linear Probing vs Cuckoo vs Swiss Table

| Property | LP (vanilla) | Robin Hood | Cuckoo | Swiss Table |
|---|---|---|---|---|
| Probe sequence | Linear | Linear | 2-choice | 16-slot SIMD groups |
| Max DIB / chain | O(log²n) typical | **O(log n)** | 2 (hard) | O(1) SIMD check |
| Lookup average | O(1/(1-α)) | ~O(1/(1-α)) | O(1) | O(1) |
| Lookup worst | O(n) | **O(log n)** | **O(1)** | O(1) with SIMD |
| Insert | O(1) amortised | O(1) amortised | O(1) amortised | O(1) amortised |
| Delete | Tombstones | **Backward shift** | O(1) direct | Metadata byte |
| No tombstones | ✗ | **✓** | ✓ | ✓ (metadata) |
| Cache behaviour | ★★★ Best | ★★★ Best | ★★ (2 lines) | ★★★★ (SIMD) |
| Memory overhead | 0 per entry | 1 DIB value | 2× table size | 1 byte metadata |
| Max safe load | 0.70 | **0.875** | 0.50 | **0.875** |
| Implementation | Simple | Moderate | Complex | Very complex |
| DIB stored | No | Yes (1-4B) | No | 7-bit in metadata |
| Variance | High | **Low** | Zero (2 slots) | Low (SIMD filter) |
| Production use | Python dict | Rust pre-1.36 | Network HW | Rust 1.36+, Abseil |
| SIMD | No | No | No | **Yes** |

### The Algorithmic Evolution

```
Linear Probing (1953):
  Simple. Cache-friendly. Catastrophic at high load.
  Max load: 0.70. Primary clustering ruins performance.

Robin Hood (1985, Celis):
  Same probe path as linear probing. Redistributes DIBs fairly.
  Backward-shift deletion: no tombstones.
  Max load: 0.875. Dramatically better variance.
  Cache: identical to linear probing (same physical access pattern).
  
  The insight: YOU CANNOT GET BETTER CACHE THAN LINEAR PROBING.
  Robin Hood preserves the cache advantage while fixing the variance problem.
  → This is why Robin Hood is important: it's the best cache-friendly scheme.

Cuckoo (1990s/2001, Pagh & Rodler):
  O(1) HARD worst-case lookup. The only hard guarantee.
  Two hash functions, two tables.
  Max load: 0.50 (2-choice). Cache: 2 cache lines per lookup.
  
Swiss Table (2017, Abseil):
  Robin Hood + SIMD parallelism.
  16 metadata bytes per group loaded in one SIMD instruction.
  Check all 16 slots simultaneously for hash match.
  Max load: 0.875. Cache: one group per lookup usually.
  → Current state of the art.

Summary:
  If you need cache-friendly + good distribution: Robin Hood.
  If you need O(1) hard guarantee: Cuckoo.
  If you need absolute maximum throughput: Swiss Table (with SIMD).
  If you need simplicity: Linear Probing at α ≤ 0.70.
```

---

## 15. Self-Test Questions

1. **State the Robin Hood invariant precisely. How does it differ from the linear probing invariant? What does it imply about the DIB values in consecutive occupied slots?**

2. **Trace the insertion of key `k=15` (h=15%8=7) into a table of capacity 8 containing keys with DIBs: `[_,_,_,_,3,4,0,1]` at slots 0-7. Show every probe step, every steal, and the final table state.**

3. **Explain the early-exit optimisation for unsuccessful lookups. Why is it correct — what invariant guarantees that if `s.dib < probe_dib` at some slot, the key cannot exist later in the probe sequence?**

4. **Trace the backward-shift deletion of the key at slot 3 in table `[_,(A,0),(B,1),(C,2),(D,3),(E,1),(F,0),_]` (key, DIB shown). Show every shift step and verify all remaining keys are still correctly findable.**

5. **Why is DIB=0 a "stop condition" for backward shift deletion? What would happen if you tried to shift a DIB=0 key one slot backward?**

6. **Rust's pre-1.36 HashMap used a load factor of 90.9% with Robin Hood hashing. Why is this safe when standard linear probing would be catastrophically slow at 90%? What specific property of Robin Hood makes high load factors viable?**

7. **Robin Hood hashing eliminates tombstones via backward shift, but linear probing requires tombstones. What structural property (the Robin Hood invariant) makes backward shift correct for Robin Hood but NOT for standard linear probing?**

8. **Compare the expected maximum probe length at α=0.9 for vanilla linear probing vs Robin Hood hashing. By approximately what factor does Robin Hood improve the worst case?**

9. **Swiss Table replaced Robin Hood in Rust's HashMap. Explain the key advantage of Swiss Table's metadata-byte + SIMD approach over Robin Hood's DIB comparison approach. When would you still prefer Robin Hood (e.g., embedded systems)?**

10. **Design a Robin Hood hash set where the DIB is stored in a single byte (uint8_t). What is the maximum safe table capacity before DIB overflow becomes a concern? How would you handle overflow gracefully?**

---

## Quick Reference Card

```
Robin Hood Hashing — Linear probing with fairness enforcement.
                     Same cache behaviour, dramatically better variance.

Core concept:
  DIB (Distance from Initial Bucket): how far a key is from its home.
  DIB = (current_slot - home(key)) mod capacity
  Invariant: DIB in consecutive slots increases by at most 1.

Insert (Robin Hood):
  At each probe slot:
    Empty → place key with current DIB. Done.
    Existing key → update value. Done.
    Occupant with smaller DIB ("richer") → STEAL this slot.
      Swap (key, val, dib) with occupant. Continue placing displaced occupant.
    Occupant with same/larger DIB → advance without stealing.

Lookup (with early-exit):
  At each probe slot:
    Empty → not found.
    s.dib < probe_dib → EARLY EXIT: key would be here if present. Not found.
    key matches → found!
    Otherwise → advance.

Delete (backward shift — NO TOMBSTONES):
  1. Find and mark target slot as EMPTY.
  2. For each subsequent slot:
     If slot is empty or dib==0: STOP.
     Move entry one slot backward (DIB decrements by 1).

Key properties:
  Max load factor: 0.875 (common), up to 0.9 (aggressive)
  No tombstones: backward-shift deletion keeps table clean
  Variance: O(log n) max DIB vs O(log²n) for vanilla LP
  Cache: identical to linear probing (sequential access pattern)
  DIB storage: 1-4 bytes per slot (usually 4)

Why 0.875 is safe (linear probing needs ≤ 0.70):
  Robin Hood's levelling invariant keeps max DIB ≈ O(log n)
  even at 87.5% load. Linear probing at same load → O(n) chains.

Pitfall checklist:
  ✗ Wrong steal condition: steal when s.dib < incoming_dib (NOT ≤)
  ✗ Backward shift not stopping at dib==0 → negative DIB (→ EMPTY sentinel)
  ✗ External references held across deletions → entries physically move
  ✗ DIB overflow with uint8_t at very high load → use int32_t
  ✗ Duplicate key not checked during steal loop → corrupt table
  ✗ Wrap-around DIB computation with signed arithmetic → use unsigned

vs alternatives:
  vs Linear Probing:  Same cache, better variance, +4 bytes/entry for DIB
  vs Cuckoo:          Worse guarantee (O(log n) vs O(1)), better memory, simpler
  vs Swiss Table:     Slower (no SIMD), simpler, no special hardware needed
  vs Chaining:        Better cache, bounded max probe, no pointer overhead

Evolution: LP → Robin Hood → Swiss Table (SIMD parallelism on top of RH ideas)
Used in: Rust HashMap (pre-1.36), many competitive programming implementations,
         game engine hash maps, embedded system hash tables.
```

---

*Previous: [20 — Extendible Hashing](./20_extendible_hashing.md)*
*Next: [22 — Hash Set](./22_hash_set.md)*
*See also: [16 — Linear Probing](./16_linear_probing.md) | [Swiss Table paper](https://abseil.io/about/design/swisstables) | [Rust HashMap source](https://github.com/rust-lang/hashbrown)*

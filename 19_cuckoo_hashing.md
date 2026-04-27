# Cuckoo Hashing

> **Curriculum position:** Hash-Based Structures → #6
> **Interview weight:** ★★★☆☆ Medium — the O(1) worst-case lookup is a classic systems interview topic; the cycle detection and stash mechanism are deeper questions
> **Difficulty:** Advanced — the eviction chain logic and cycle detection require careful thought; the amortised analysis is non-trivial
> **Prerequisite:** [14 — Hash Map](./14_hash_map.md) | [18 — Double Hashing](./18_double_hashing.md)

---

## Table of Contents

1. [Intuition First — Eviction and the Bird](#1-intuition-first)
2. [Internal Working — Two Tables, Two Functions](#2-internal-working)
3. [The Eviction Chain — How Insertion Works](#3-the-eviction-chain)
4. [Cycle Detection and Rehash](#4-cycle-detection-and-rehash)
5. [The Stash — Handling Unavoidable Cycles](#5-the-stash)
6. [Cuckoo Hashing Variants](#6-cuckoo-hashing-variants)
7. [Load Factor and Performance](#7-load-factor-and-performance)
8. [Time & Space Complexity](#8-time--space-complexity)
9. [Complete C++ Implementation](#9-complete-c-implementation)
10. [Core Operations — Visualised](#10-core-operations--visualised)
11. [Interview Problems](#11-interview-problems)
12. [Real-World Uses](#12-real-world-uses)
13. [Edge Cases & Pitfalls](#13-edge-cases--pitfalls)
14. [Comparison: Cuckoo vs Other Hash Tables](#14-comparison)
15. [Self-Test Questions](#15-self-test-questions)

---

## 1. Intuition First

The cuckoo bird is famous for its reproductive strategy: it lays its egg in another bird's nest, and when the cuckoo chick hatches, it pushes the other eggs out to claim the nest for itself.

Cuckoo hashing works exactly the same way.

Every key has two candidate slots — one in each of two separate hash tables. The key is always stored in exactly one of those two slots. When you want to look up a key, you check exactly two locations and return immediately. No probing. No chain traversal. Two memory accesses, and you are done.

The elegance is in the insertion. When both of a new key's candidate slots are occupied, the new key **evicts** the occupant of one slot, taking that slot for itself. The evicted key then goes to its *other* candidate slot (it has two, and we know which one it just left). If that slot is also occupied, another eviction happens. This chain continues until either an empty slot is found or a cycle is detected.

```
Key A lives at table1[h1(A)] or table2[h2(A)].
Currently at table1[h1(A)] = slot 3.
If slot 3 is occupied by A, lookup returns A. Done in O(1).
If slot 3 is empty or has a different key, check table2[h2(A)] = slot 7.
O(1) regardless. Two checks. Always.
```

The trade-off: insertions are amortised O(1) but occasionally trigger long eviction chains or cycle detection. The lookup guarantee — true O(1) worst-case, not just average — is the defining property that makes cuckoo hashing irreplaceable in latency-sensitive systems. When the difference between "usually fast" and "guaranteed fast" matters (network switches, real-time systems, predictable-latency databases), cuckoo hashing is the answer.

---

## 2. Internal Working — Two Tables, Two Functions

### The Physical Layout

```
Cuckoo hash table with capacity n (n slots per table):

Table 1:  [slot 0][slot 1][slot 2]...[slot n-1]     ← indexed by h1(key)
Table 2:  [slot 0][slot 1][slot 2]...[slot n-1]     ← indexed by h2(key)

Each slot stores exactly one key-value pair (or is empty).
Each key is stored in EXACTLY ONE of its two candidate slots.

For key k:
  Candidate 1: table1[ h1(k) ]
  Candidate 2: table2[ h2(k) ]
  k is stored in exactly one of these two locations.
```

### The Invariant

> **Every key `k` exists at exactly one of `table1[h1(k)]` or `table2[h2(k)]` — never in both, never in neither (unless absent).**

This single invariant is what makes O(1) lookup possible. The lookup algorithm is:

```cpp
bool lookup(K key, V& out) const {
    // Check exactly two locations. Period.
    int idx1 = h1(key) % n;
    if (table1[idx1].occupied && table1[idx1].key == key) {
        out = table1[idx1].val; return true;
    }
    int idx2 = h2(key) % n;
    if (table2[idx2].occupied && table2[idx2].key == key) {
        out = table2[idx2].val; return true;
    }
    return false;  // key not present
}
```

This is two array accesses plus two key comparisons. On modern hardware with 64-byte cache lines, both accesses often hit the same cache line if the tables are small enough, or resolve within two cache misses at most. There is no loop, no probe count, no amortisation — O(1) is a hard guarantee.

### Two Independent Hash Functions

The correctness of cuckoo hashing requires that `h1` and `h2` are **independent** — knowing `h1(k)` tells you nothing about `h2(k)`. If they are correlated, the cuckoo graph (defined below) develops structure that makes cycles more likely, degrading insertion performance.

In practice, independence is achieved by:
- Using two completely different hash families
- Using one hash function with two different random seeds
- Using the Kirsch-Mitzenmacher technique from double hashing: `h2(k) = h(k, seed2)` where seed2 ≠ seed1

---

## 3. The Eviction Chain — How Insertion Works

### The Cuckoo Graph

The mathematical structure underlying cuckoo hashing is the **cuckoo graph**: a bipartite graph where:
- Left vertices = slots in table1 (n vertices)
- Right vertices = slots in table2 (n vertices)
- Each stored key = an edge from `table1[h1(k)]` to `table2[h2(k)]`

An insertion succeeds if and only if the connected component containing the new edge's two vertices has at most as many edges as vertices (i.e., it is a tree or unicyclic graph — the graph-theoretic condition for insertability).

If the component has more edges than vertices (a cycle), a rehash is needed.

### The Insertion Algorithm

```
Insert key k:

1. If table1[h1(k)] is empty → place k there. Done.
2. If table2[h2(k)] is empty → place k there. Done.
3. Both slots occupied. Start eviction:
   a. Choose one slot (typically table1[h1(k)]).
   b. Evict the current occupant x = table1[h1(k)].
   c. Place k at table1[h1(k)].
   d. Now try to place x in ITS OTHER slot (table2[h2(x)]).
   e. If table2[h2(x)] is empty → place x there. Done.
   f. Otherwise evict the occupant there and repeat with the new evicted key.
4. If the chain length exceeds a threshold → cycle detected → rehash.
```

### The "Other Slot" Concept

This is the key insight: when a key is evicted, it always goes to its **other** candidate. If it was just displaced from table1, it must go to table2 (and vice versa). This prevents it from immediately going back to where it came from (which would cause an immediate cycle).

```
Key x was at table1[h1(x)].
x is evicted → x tries table2[h2(x)].
If evicted from there → x tries table1[h1(x)] (its original slot — but now freed).
```

The eviction chain alternates between table1 and table2, ensuring progress.

---

## 4. Cycle Detection and Rehash

### When Does Insertion Fail?

Insertion fails (and triggers a rehash) when the eviction chain forms a **cycle** — it returns to a slot that was already visited during this insertion attempt.

```
Example of a cycle:

h1(A) = 3, h2(A) = 1
h1(B) = 3, h2(B) = 5
h1(C) = 5, h2(C) = 1

State: A@table1[3], B@table2[5], C@table2[1]

Insert D: h1(D)=3, h2(D)=5. Both occupied.

Eviction chain:
  Evict A from table1[3]. Place D there.
  A→ tries table2[h2(A)=1]. Occupied by C. Evict C.
  C → tries table1[h1(C)=5]. Empty. Place C. Done? Wait.
  Actually let me redo:

h1(A)=0, h2(A)=0
h1(B)=0, h2(B)=0
Insert A: table1[0]=A.
Insert B: table1[0]=A (occupied), table2[0]=B (empty). Place B@table2[0].
Insert C: h1(C)=0, h2(C)=0. table1[0]=A (h1), table2[0]=B (h2). Both occupied.
Evict A from table1[0]. Place C@table1[0].
A → tries table2[h2(A)=0]. Occupied by B. Evict B.
B → tries table1[h1(B)=0]. Occupied by C! C was just placed here.
B → tables table2[h2(B)=0]. Still occupied by... we just evicted it.
CYCLE: we're going around {table1[0], table2[0]} forever.
```

### The Maximum Kick Threshold

The standard approach: if the eviction chain reaches length `MAX_KICKS = O(log n)` without finding an empty slot, declare a cycle and rehash.

```cpp
bool insert_with_eviction(K key, V val) {
    static const int MAX_KICKS = 2 * log2(capacity_) + 1;

    for (int kick = 0; kick < MAX_KICKS; kick++) {
        // Try table1
        int i1 = h1(key) % n;
        if (!table1[i1].occupied) { table1[i1] = {key, val, true}; return true; }

        // Try table2
        int i2 = h2(key) % n;
        if (!table2[i2].occupied) { table2[i2] = {key, val, true}; return true; }

        // Both occupied — evict from table1 (alternating eviction policy)
        swap(key, table1[i1].key);
        swap(val, table1[i1].val);
        // Now key is the evicted key. Loop continues to place it in table2.
        // Next iteration: try to place evicted key in table2 (it was from table1).
        // But we need to alternate tables...
    }
    return false;  // cycle detected — rehash needed
}
```

### The Cycle Probability — Mathematical Analysis

For n keys in a cuckoo table of total capacity 2n (n slots each table), load factor α = 0.5:

```
The cuckoo graph has n left vertices, n right vertices, and n edges.
A cycle exists in a component when: edges ≥ vertices in that component.

For a random bipartite graph with n edges on n+n vertices:
  P(graph has a cycle) ≈ 1 - e^(-n²/(4n²)) = 1 - e^(-1/4) ≈ 22%  at α=0.5

Practical consequence:
  With 22% probability, a single insertion attempt at α=0.5 encounters a cycle.
  But with good hash functions and rehash, this is transparent to the caller.

At α < 0.5 (say 0.4):
  P(cycle) ≈ e^(-(1-2α)²/2) → much lower
  At α=0.4: P(cycle) ≈ 2%

Recommendation: maintain α ≤ 0.45 per table (total load ≤ 0.9 of combined capacity)
to keep rehash frequency low.
```

### Rehash Strategy

```cpp
void rehash() {
    // Double the capacity and re-insert all keys with NEW hash seeds.
    // New seeds are critical: the old seeds caused the cycle.
    // With new seeds, the same set of keys likely forms no cycles.
    size_t new_n = n * 2;
    generate_new_seeds();   // randomise h1_seed and h2_seed

    auto old1 = move(table1_);
    auto old2 = move(table2_);
    table1_.assign(new_n, Slot{});
    table2_.assign(new_n, Slot{});
    n = new_n;
    size_ = 0;

    for (auto& slot : old1) if (slot.occupied) insert(slot.key, slot.val);
    for (auto& slot : old2) if (slot.occupied) insert(slot.key, slot.val);
}
// Amortised analysis: rehash is O(n) but happens at most O(log n) times
// before succeeding with high probability (geometric series argument).
```

---

## 5. The Stash — Handling Unavoidable Cycles

### The Problem with Pure Cuckoo Hashing

Even with the cycle detection and rehash strategy, there exists a class of inputs — specifically keys that form cycles in the cuckoo graph regardless of hash function — that can cause an exponential number of rehashes. In adversarial settings (or very unlucky random settings), this makes pure cuckoo hashing unreliable.

### The Stash Solution (Kirsch, Mitzenmacher, Wieder, 2009)

Add a small overflow buffer called a **stash** — a linear list of at most `s` key-value pairs that cannot be placed in the main tables due to cycles. Keys in the stash are searched linearly during lookup.

```
Lookup with stash:
  1. Check table1[h1(key)] — O(1)
  2. Check table2[h2(key)] — O(1)
  3. Scan stash (at most s elements) — O(s) = O(1) since s is constant

Insertion with stash:
  1. Try to insert normally (eviction chain).
  2. If cycle detected after MAX_KICKS:
     a. If stash has room (size < s): add to stash. Done.
     b. If stash is full: rehash (must succeed with overwhelming probability).
```

### Stash Size and Failure Probability

```
With stash size s:
  P(table full after n insertions at α < 0.5) < n × (1/(2n))^(s+1)

For n = 10^6 and s = 4:
  P(failure) < 10^6 × (1/2×10^6)^5 = 10^6 × 10^-30 ≈ 10^-24

This is essentially zero. A stash of 4 elements makes cuckoo hashing
reliable for tables of any practical size.

Lookup cost with stash:
  Worst case: 2 table accesses + 4 stash comparisons = 6 operations.
  Still O(1) with a tiny constant.
```

### Implementation

```cpp
// Stash as a small array of key-value pairs
struct StashEntry { K key; V val; bool occupied = false; };
static const int STASH_SIZE = 4;
StashEntry stash_[STASH_SIZE];

bool lookup_with_stash(K key, V& out) {
    // Check tables first (O(1))
    if (table1_[h1(key)].occupied && table1_[h1(key)].key == key) {
        out = table1_[h1(key)].val; return true;
    }
    if (table2_[h2(key)].occupied && table2_[h2(key)].key == key) {
        out = table2_[h2(key)].val; return true;
    }
    // Linear scan of tiny stash (O(1) with s constant)
    for (auto& e : stash_) {
        if (e.occupied && e.key == key) { out = e.val; return true; }
    }
    return false;
}
```

---

## 6. Cuckoo Hashing Variants

### 2-Choice Cuckoo (Classic, described above)

- 2 tables, 2 hash functions
- Each key has exactly 2 candidate slots
- Max load: ~50% per table (total α ≤ 0.5 for stability)

### d-Ary Cuckoo Hashing (d > 2)

Use `d` hash functions and `d` tables. Each key has `d` candidate slots.

```
3-Choice Cuckoo:
  Key k candidates: table1[h1(k)], table2[h2(k)], table3[h3(k)]
  Lookup: exactly 3 memory accesses (O(1) worst case).
  Max load: ~91% (drastically higher than 2-choice's 50%)

2-Choice at 50% load, 3-Choice at 91%:
  3× more memory-efficient while still guaranteeing O(1) lookup.

4-Choice: max load ≈ 97%
d-Choice: max load → 1 - 1/e^(1/d) as d→∞ → approaches 100%

Trade-off: each additional function adds 1 memory access to lookup.
Most practical systems use 2 or 3 tables.
```

### Cuckoo Hashing with Buckets (Blocked Cuckoo)

Instead of one slot per location, store a small bucket of `b` slots:

```
Each position holds b keys (typical: b = 4 or 8, fits one cache line).

Lookup:  check table1 bucket → compare up to b keys (one cache line load)
         check table2 bucket → compare up to b keys (one cache line load)
         Always 2 cache line loads. O(1) worst case.

Max load: can exceed 95% with bucket size 4.

Used in: high-performance network packet classifiers, where DRAM bandwidth
is the bottleneck and each lookup must minimise memory accesses.
```

### Generalised Cuckoo (Arbitrary Number of Tables)

Intel's Paper Cuckoo Hashing (2010) uses 4 hash functions on 2 tables, allowing each key to be placed in 4 different locations (2 buckets × 2 positions per bucket). This achieves >99% load with O(1) lookup.

---

## 7. Load Factor and Performance

### The Phase Transition

Cuckoo hashing exhibits a sharp **phase transition** at load factor 0.5 (for 2-choice):

```
α < 0.49:  Expected insertion O(1), rehash probability negligible
α = 0.49:  Occasionally slow insertions, rare rehash
α = 0.50:  Phase transition — sudden dramatic increase in insertion failures
α > 0.50:  Frequent cycles, exponential expected insertion time (unusable)

This is analogous to the percolation phase transition in physics.
Below the threshold: the cuckoo graph is a forest (no cycles) with probability → 1.
Above the threshold: a giant connected component with cycles emerges.

With stash (s=4): effective threshold moves to ~0.9 total load (both tables combined).
With d-ary (d=3): threshold moves to ~0.91 per table.
```

### Expected Performance Metrics

```
2-Choice Cuckoo, α = 0.45 per table (0.9 total):

Lookup (successful):    2.0 memory accesses always (O(1) worst case) ✓
Lookup (unsuccessful):  2.0 memory accesses always (O(1) worst case) ✓
Insert (no eviction):   2.0 memory accesses (direct placement)
Insert (with eviction): O(1) amortised (expected eviction chain length = O(1))
Insert (with rehash):   O(n) for the rehash; amortised O(1) overall
Delete:                 2.0 memory accesses + mark empty (O(1))

For comparison (all at α=0.7):
  Linear probing:    E[successful lookup] ≈ 2.17
  Double hashing:    E[successful lookup] ≈ 1.58
  Cuckoo:            WORST-CASE lookup = 2.0 (same every time!)
  Chaining:          E[successful lookup] ≈ 1.35 (better average; O(n) worst)

The key insight: cuckoo provides a GUARANTEE, not just an expectation.
```

---

## 8. Time & Space Complexity

### Operation Complexities

| Operation | Average | Worst Case | Notes |
|---|---|---|---|
| `lookup(k)` successful | **O(1)** | **O(1)** ← hard guarantee | Exactly 2 table accesses |
| `lookup(k)` unsuccessful | **O(1)** | **O(1)** ← hard guarantee | Exactly 2 table accesses |
| `insert(k, v)` | **O(1) amortised** | O(n) on rehash | Chain usually short; rehash rare |
| `delete(k)` | **O(1)** | **O(1)** | Find + mark empty; 2 table accesses |
| `rehash()` | O(n) | O(n log n) | Expected O(n) with random seeds |
| Build from n entries | O(n) avg | O(n²) | Rare rehashes amortised away |
| Iteration | O(n) | O(n) | Scan both tables + stash |

### Space Complexity

| Component | Space | Notes |
|---|---|---|
| Table 1 | O(n) | n slots |
| Table 2 | O(n) | n slots |
| Stash | O(1) | constant s entries |
| Total | O(n) | 2n slots for n/2 used ≈ 50% utilisation (2-choice) |
| vs chaining | Chaining: O(n) + pointer overhead | Cuckoo: same O(n), zero pointer overhead |
| Memory efficiency | 2× overhead vs array at 50% load | Can be improved with buckets or d-ary |

### The Lookup Guarantee — Why It Matters

```
Standard hash tables:
  Average lookup: O(1). Fine 99.9% of the time.
  Worst case: O(n). Can occur during hash flooding attacks or adversarial input.
  P(lookup > k probes) decreases exponentially, but never reaches zero.

Cuckoo hashing:
  ALL lookups: exactly 2 memory accesses.
  P(lookup > 2 accesses) = 0.

Why this matters:
  In a network switch processing 1 billion packets/second:
    Each packet must check the forwarding table.
    A single O(n) lookup at the wrong moment drops packets.
    Cuckoo's guarantee eliminates this failure mode.

  In a real-time system with 1ms deadlines:
    O(1) expected does not mean O(1) guaranteed.
    A 0.001% probability of O(n) lookup × 10^9 operations/day = 10^4 violations/day.
    Cuckoo's guarantee means exactly 0 violations.
```

---

## 9. Complete C++ Implementation

```cpp
#include <iostream>
#include <vector>
#include <optional>
#include <functional>
#include <stdexcept>
#include <random>
#include <cmath>
using namespace std;

template<typename K, typename V, typename Equal = equal_to<K>>
class CuckooHashMap {
private:
    // ── Slot ─────────────────────────────────────────────────────────────────
    struct Slot {
        K    key;
        V    val;
        bool occupied = false;
    };

    // ── Stash entry ───────────────────────────────────────────────────────────
    struct StashEntry {
        K    key;
        V    val;
        bool used = false;
    };

    // ── State ─────────────────────────────────────────────────────────────────
    vector<Slot>  t1_, t2_;           // two tables, each of size n_
    size_t        n_;                  // slots per table
    size_t        size_;               // total occupied entries (tables + stash)
    Equal         eq_;

    // Hash seeds (randomised on rehash to escape adversarial inputs)
    uint64_t      seed1_, seed2_;

    static const int STASH_CAP = 8;
    StashEntry    stash_[STASH_CAP];
    int           stash_size_ = 0;

    static const int MAX_KICKS_MULTIPLIER = 6;  // MAX_KICKS = multiplier × log2(n)

    // ── Hash functions ────────────────────────────────────────────────────────
    // Seeded MurmurHash-inspired mixing for integer keys
    static uint64_t mix(uint64_t key, uint64_t seed) {
        key += seed;
        key ^= key >> 30;
        key *= 0xbf58476d1ce4e5b9ULL;
        key ^= key >> 27;
        key *= 0x94d049bb133111ebULL;
        key ^= key >> 31;
        return key;
    }

    size_t h1(const K& key) const {
        return mix(hash<K>{}(key), seed1_) % n_;
    }

    size_t h2(const K& key) const {
        return mix(hash<K>{}(key), seed2_) % n_;
    }

    static uint64_t random_seed() {
        static mt19937_64 rng(random_device{}());
        return rng();
    }

    int max_kicks() const {
        return MAX_KICKS_MULTIPLIER * max(1, (int)log2((double)n_));
    }

    // ── Stash operations ──────────────────────────────────────────────────────
    bool stash_lookup(const K& key, V& out) const {
        for (int i = 0; i < stash_size_; i++) {
            if (stash_[i].used && eq_(stash_[i].key, key)) {
                out = stash_[i].val; return true;
            }
        }
        return false;
    }

    bool stash_insert(const K& key, const V& val) {
        if (stash_size_ >= STASH_CAP) return false;
        stash_[stash_size_++] = {key, val, true};
        size_++;
        return true;
    }

    bool stash_erase(const K& key) {
        for (int i = 0; i < stash_size_; i++) {
            if (stash_[i].used && eq_(stash_[i].key, key)) {
                stash_[i] = stash_[--stash_size_];  // replace with last entry
                stash_[stash_size_].used = false;
                size_--;
                return true;
            }
        }
        return false;
    }

    // ── Rehash ────────────────────────────────────────────────────────────────
    void rehash(size_t new_n) {
        // Try up to 3 times with new random seeds before doubling capacity
        for (int attempt = 0; attempt < 3; attempt++) {
            vector<Slot> old1 = move(t1_);
            vector<Slot> old2 = move(t2_);
            auto old_stash_size = stash_size_;
            StashEntry   old_stash[STASH_CAP];
            copy(stash_, stash_ + stash_size_, old_stash);

            t1_.assign(new_n, Slot{});
            t2_.assign(new_n, Slot{});
            n_          = new_n;
            size_       = 0;
            stash_size_ = 0;
            seed1_      = random_seed();
            seed2_      = random_seed();

            bool ok = true;
            // Re-insert all entries from old tables
            for (auto& s : old1) if (s.occupied && !do_insert(s.key, s.val)) { ok = false; break; }
            if (ok) for (auto& s : old2) if (s.occupied && !do_insert(s.key, s.val)) { ok = false; break; }
            if (ok) for (int i = 0; i < old_stash_size; i++) {
                if (old_stash[i].used && !do_insert(old_stash[i].key, old_stash[i].val)) {
                    ok = false; break;
                }
            }

            if (ok) return;   // rehash succeeded with these seeds
            // Failed again — restore and try next attempt with doubled size
            if (attempt == 1) new_n *= 2;   // grow on second failure
        }
        throw runtime_error("cuckoo: rehash failed after 3 attempts");
    }

    // ── Core insertion (no rehash trigger) ────────────────────────────────────
    // Returns true if successful, false if cycle detected (caller must rehash).
    bool do_insert(K key, V val) {
        // Step 1: quick check for existing key (avoid duplicate insert)
        size_t i1 = h1(key), i2 = h2(key);
        if (t1_[i1].occupied && eq_(t1_[i1].key, key)) {
            t1_[i1].val = val; return true;  // update
        }
        if (t2_[i2].occupied && eq_(t2_[i2].key, key)) {
            t2_[i2].val = val; return true;  // update
        }
        for (int i = 0; i < stash_size_; i++) {
            if (stash_[i].used && eq_(stash_[i].key, key)) {
                stash_[i].val = val; return true;
            }
        }

        // Step 2: eviction loop
        int kicks = max_kicks();
        bool in_table1 = true;  // which table we're trying to place in

        for (int kick = 0; kick < kicks; kick++) {
            if (in_table1) {
                size_t idx = h1(key);
                if (!t1_[idx].occupied) {
                    t1_[idx] = {key, val, true};
                    size_++;
                    return true;
                }
                // Evict current occupant
                swap(key, t1_[idx].key);
                swap(val, t1_[idx].val);
                // Evicted key (now in key/val) needs to go to table2
                in_table1 = false;
            } else {
                size_t idx = h2(key);
                if (!t2_[idx].occupied) {
                    t2_[idx] = {key, val, true};
                    size_++;
                    return true;
                }
                // Evict current occupant
                swap(key, t2_[idx].key);
                swap(val, t2_[idx].val);
                // Evicted key goes back to table1
                in_table1 = true;
            }
        }

        // Cycle detected: try stash
        if (stash_insert(key, val)) return true;

        return false;   // stash also full — caller must rehash
    }

public:
    // ── Construction ──────────────────────────────────────────────────────────

    explicit CuckooHashMap(size_t initial_n = 16)
        : t1_(initial_n, Slot{}), t2_(initial_n, Slot{}),
          n_(initial_n), size_(0),
          seed1_(random_seed()), seed2_(random_seed()) {}

    ~CuckooHashMap() = default;

    CuckooHashMap(const CuckooHashMap&)            = delete;
    CuckooHashMap& operator=(const CuckooHashMap&) = delete;
    CuckooHashMap(CuckooHashMap&&)                 = default;

    // ── O(1) Lookup — worst case guaranteed ──────────────────────────────────

    optional<V> get(const K& key) const {
        // Check exactly two table slots
        size_t i1 = h1(key);
        if (t1_[i1].occupied && eq_(t1_[i1].key, key)) return t1_[i1].val;

        size_t i2 = h2(key);
        if (t2_[i2].occupied && eq_(t2_[i2].key, key)) return t2_[i2].val;

        // Check stash (tiny, at most STASH_CAP entries)
        for (int i = 0; i < stash_size_; i++) {
            if (stash_[i].used && eq_(stash_[i].key, key)) return stash_[i].val;
        }
        return nullopt;
    }

    V& at(const K& key) {
        size_t i1 = h1(key);
        if (t1_[i1].occupied && eq_(t1_[i1].key, key)) return t1_[i1].val;
        size_t i2 = h2(key);
        if (t2_[i2].occupied && eq_(t2_[i2].key, key)) return t2_[i2].val;
        for (int i = 0; i < stash_size_; i++) {
            if (stash_[i].used && eq_(stash_[i].key, key)) return stash_[i].val;
        }
        throw out_of_range("key not found");
    }

    bool contains(const K& key) const {
        return get(key).has_value();
    }

    // ── O(1) amortised Insert ─────────────────────────────────────────────────

    void insert(const K& key, const V& val) {
        // Grow if load exceeds threshold
        float total_load = (float)size_ / (2.0f * n_);
        if (total_load > 0.45f) {
            rehash(n_ * 2);
        }

        if (!do_insert(key, val)) {
            // Cycle with stash full → must rehash
            rehash(n_);   // same size but new seeds (first try)
            if (!do_insert(key, val)) {
                rehash(n_ * 2);   // larger table
                if (!do_insert(key, val)) {
                    throw runtime_error("cuckoo: insert failed after rehash");
                }
            }
        }
    }

    V& operator[](const K& key) {
        auto opt = get(key);
        if (!opt) insert(key, V{});
        return at(key);
    }

    // ── O(1) Delete — worst case guaranteed ──────────────────────────────────

    bool erase(const K& key) {
        size_t i1 = h1(key);
        if (t1_[i1].occupied && eq_(t1_[i1].key, key)) {
            t1_[i1].occupied = false;
            t1_[i1].key = K{};
            t1_[i1].val = V{};
            size_--;
            return true;
        }
        size_t i2 = h2(key);
        if (t2_[i2].occupied && eq_(t2_[i2].key, key)) {
            t2_[i2].occupied = false;
            t2_[i2].key = K{};
            t2_[i2].val = V{};
            size_--;
            return true;
        }
        return stash_erase(key);
    }

    // ── Utility ───────────────────────────────────────────────────────────────

    size_t size()        const { return size_; }
    bool   empty()       const { return size_ == 0; }
    size_t capacity()    const { return 2 * n_; }
    float  load_factor() const { return (float)size_ / (2.0f * n_); }

    void clear() {
        for (auto& s : t1_) s.occupied = false;
        for (auto& s : t2_) s.occupied = false;
        stash_size_ = 0;
        size_       = 0;
    }

    void print() const {
        cout << "Table 1:\n";
        for (size_t i = 0; i < n_; i++) {
            if (t1_[i].occupied)
                cout << "  [" << i << "] " << t1_[i].key << ":" << t1_[i].val << "\n";
        }
        cout << "Table 2:\n";
        for (size_t i = 0; i < n_; i++) {
            if (t2_[i].occupied)
                cout << "  [" << i << "] " << t2_[i].key << ":" << t2_[i].val << "\n";
        }
        if (stash_size_ > 0) {
            cout << "Stash:\n";
            for (int i = 0; i < stash_size_; i++) {
                cout << "  stash[" << i << "] "
                     << stash_[i].key << ":" << stash_[i].val << "\n";
            }
        }
        cout << "size=" << size_ << " n=" << n_
             << " load=" << load_factor() << "\n";
    }
};
```

---

## 10. Core Operations — Visualised

### Successful Insert (No Eviction)

```
n=4 (4 slots per table), currently empty.
h1(A)=1, h2(A)=2, h1(B)=3, h2(B)=1, h1(C)=0, h2(C)=3

Insert A: h1(A)=1. t1[1] empty → place A at t1[1]. ✓
  T1: [_][A][_][_]    T2: [_][_][_][_]

Insert B: h1(B)=3. t1[3] empty → place B at t1[3]. ✓
  T1: [_][A][_][B]    T2: [_][_][_][_]

Insert C: h1(C)=0. t1[0] empty → place C at t1[0]. ✓
  T1: [C][A][_][B]    T2: [_][_][_][_]

All insertions required 1 table access each. O(1) no eviction needed.
```

### Eviction Chain

```
n=4, current state: T1=[C][A][_][B]  T2=[_][_][_][_]
Insert D: h1(D)=1, h2(D)=2.
  t1[h1(D)=1] = A (occupied). t2[h2(D)=2] empty → place D at t2[2]. ✓

Insert E: h1(E)=1, h2(E)=2.
  t1[1]=A occupied. t2[2]=D occupied. Start eviction.

Kick 0 (try t1): t1[h1(E)=1] = A. Evict A.
  E → t1[1]. Evicted = A.
  T1: [C][E][_][B]    T2: [_][_][D][_]
  A needs to go to its other slot: t2[h2(A)=2].

Kick 1 (try t2): t2[h2(A)=2] = D. Evict D.
  A → t2[2]. Evicted = D.
  T1: [C][E][_][B]    T2: [_][_][A][_]
  D needs to go to t1[h1(D)=1]. But t1[1]=E (occupied)!

Kick 2 (try t1): t1[h1(D)=1] = E. Evict E.
  D → t1[1]. Evicted = E.
  T1: [C][D][_][B]    T2: [_][_][A][_]
  E needs to go to t2[h2(E)=2]. But t2[2]=A (occupied)!

Kick 3 (try t2): t2[h2(E)=2] = A. CYCLE! (we were just evicting trying to place E!)

Actually—let me reconsider with a cleaner example showing successful resolution:

n=4, T1=[_][A][_][_]  T2=[_][_][_][_]
h1(A)=1, h2(A)=3
h1(B)=1, h2(B)=2
h1(C)=2, h2(C)=2

Insert B: h1(B)=1. t1[1]=A (occupied). t2[h2(B)=2] empty → place B at t2[2]. ✓
  T1: [_][A][_][_]    T2: [_][_][B][_]

Insert C: h1(C)=2. t1[2] empty → place C at t1[2]. ✓
  T1: [_][A][C][_]    T2: [_][_][B][_]

Insert D: h1(D)=1, h2(D)=2. Both occupied.
  Kick 0: Evict A from t1[1]. Place D@t1[1]. Evicted=A, try t2[h2(A)=3].
  T1: [_][D][C][_]    T2: [_][_][B][_]    evicting A → t2[3]

  Kick 1: t2[3] empty. Place A@t2[3]. Done! ✓
  T1: [_][D][C][_]    T2: [_][_][B][A]

Lookup D: t1[h1(D)=1] = D ✓ (1 access)
Lookup A: t1[h1(A)=1] = D ≠ A. t2[h2(A)=3] = A ✓ (2 accesses)
Lookup B: t1[h1(B)=1] = D ≠ B. t2[h2(B)=2] = B ✓ (2 accesses)

ALL lookups: at most 2 table accesses. Guarantee holds. ✓
```

### Cycle Detection — When Rehash Is Needed

```
n=2, T1=[X][Y]  T2=[P][Q]
h1(A)=0, h2(A)=0
h1(X)=0, h2(X)=0
h1(Y)=1, h2(Y)=0
h1(P)=0, h2(P)=0
h1(Q)=1, h2(Q)=1

Insert A: h1(A)=0 → t1[0]=X (occupied). h2(A)=0 → t2[0]=P (occupied).

Kick 0: Evict X from t1[0]. A→t1[0]. Evicted=X. X→t2[h2(X)=0].
Kick 1: t2[0]=P (occupied). Evict P. X→t2[0]. Evicted=P. P→t1[h1(P)=0].
Kick 2: t1[0]=A (occupied). Evict A. P→t1[0]. Evicted=A. A→t2[h2(A)=0].
Kick 3: t2[0]=X. Evict X. A→t2[0]. Evicted=X. X→t1[h1(X)=0].
Kick 4: t1[0]=P. Evict P. X→t1[0]. Evicted=P. P→t2[h2(P)=0].
...

Pattern: T1[0] and T2[0] are a cycle involving A, X, P.
After MAX_KICKS: declare cycle. Add A to stash (if room) or rehash.
With new random seeds, h1 and h2 change → keys redistribute → cycle broken.
```

---

## 11. Interview Problems

### Problem 1: Implement Cuckoo Hash Set

**Problem:** Implement a hash set supporting `add(key)`, `remove(key)`, and `contains(key)`. The `contains` operation must have O(1) **worst-case** time complexity.

**This is the pure cuckoo hashing question.** The O(1) worst-case requirement is the distinguishing constraint that rules out all other open addressing strategies.

```cpp
class CuckooHashSet {
private:
    static const int N     = 1009;  // prime, slots per table
    static const int KICKS = 20;    // max evictions before rehash
    static const int EMPTY  = INT_MIN;  // sentinel for empty slot

    int t1_[N], t2_[N];

    static int h1(int x) { return abs(x * 2654435761u) % N; }
    static int h2(int x) {
        uint32_t h = (uint32_t)x ^ 0xdeadbeef;
        h = (h ^ (h >> 16)) * 0x45d9f3b;
        h = (h ^ (h >> 16));
        return h % N;
    }

public:
    CuckooHashSet() {
        fill(t1_, t1_ + N, EMPTY);
        fill(t2_, t2_ + N, EMPTY);
    }

    // O(1) WORST CASE — check exactly two slots
    bool contains(int key) const {
        return t1_[h1(key)] == key || t2_[h2(key)] == key;
    }

    // O(1) amortised
    void add(int key) {
        if (contains(key)) return;

        int cur = key;
        for (int kick = 0; kick < KICKS; kick++) {
            int i1 = h1(cur);
            if (t1_[i1] == EMPTY) { t1_[i1] = cur; return; }
            swap(cur, t1_[i1]);       // evict and take slot in t1

            int i2 = h2(cur);
            if (t2_[i2] == EMPTY) { t2_[i2] = cur; return; }
            swap(cur, t2_[i2]);       // evict and take slot in t2
        }
        // Cycle detected — for this example, we just add to a stash
        // (full solution would rehash with new h1/h2)
        throw overflow_error("cuckoo: cycle detected, rehash needed");
    }

    // O(1) WORST CASE
    void remove(int key) {
        int i1 = h1(key);
        if (t1_[i1] == key) { t1_[i1] = EMPTY; return; }
        int i2 = h2(key);
        if (t2_[i2] == key) { t2_[i2] = EMPTY; return; }
    }
};

/*
Key interview points:
1. contains() checks EXACTLY 2 slots — no loop, no amortisation.
   This is the O(1) worst-case guarantee.
2. add() may evict multiple keys but each displaced key goes to its OTHER table.
3. The alternating swap pattern: t1→t2→t1→t2 prevents immediate re-cycles.
4. MAX_KICKS = 2 * log(n) is sufficient by theory; 20 is practical for n=1009.
5. remove() is also O(1) worst case — same 2 table accesses as contains.
*/
```

---

### Problem 2: Two Sum — O(1) Worst-Case per Query

**Problem:** Given a static array and a stream of target queries, answer each "does any pair sum to target?" in O(1) worst case per query (not just O(1) average).

**Why cuckoo:** Standard hash maps give O(1) average lookup. For worst-case O(1) per query, you need cuckoo hashing.

```cpp
class TwoSumChecker {
private:
    CuckooHashSet seen_;   // cuckoo-based for O(1) worst-case

public:
    TwoSumChecker(vector<int>& nums) {
        for (int x : nums) seen_.add(x);
    }

    // O(1) WORST CASE — both seen_.contains calls are O(1) worst case
    bool hasPairSumming(int target, const vector<int>& nums) const {
        for (int x : nums) {
            int comp = target - x;
            if (comp != x && seen_.contains(comp)) return true;
            if (comp == x) {
                // Check if x appears at least twice (need a count map)
                // Simplified: omit for clarity
            }
        }
        return false;
    }
};

// Why O(1) worst case matters here:
// If this runs in a real-time fraud detection system:
//   1 billion transactions/day × average O(1) ≠ zero slow transactions.
//   1 billion transactions/day × O(1) WORST CASE = zero slow transactions.
```

---

### Problem 3: Consistent Hashing with O(1) Worst-Case Node Lookup

**Problem (System Design):** You are designing a distributed key-value store where:
- `lookup(key)` must return the responsible server in O(1) worst case
- Servers can be added/removed
- Keys should be distributed approximately uniformly

**Discuss how cuckoo hashing applies here vs standard hashing.**

```cpp
/*
Standard consistent hashing:
  - Hash ring with virtual nodes
  - Lookup: O(log n) binary search on ring
  - Problem: guaranteed O(1) lookup? NO.

Cuckoo approach for server lookup table:
  - Each key has two candidate servers (from h1, h2)
  - Lookup: check both servers' tables in O(1)
  - Placement: cuckoo insertion assigns key to one of two servers

But the real application is at the server level:

Each server uses cuckoo hashing INTERNALLY for its key-value store:
  - Server receives a key → must locate it in its local table
  - Cuckoo gives O(1) WORST CASE for this local lookup
  - Critical for predictable latency under load

The design interview answer:
  1. Use consistent hashing ring (with binary search O(log n)) to route to server
  2. Each server uses cuckoo hashing for its internal table (O(1) worst case)
  3. The bottleneck is now the ring lookup (O(log n)), not the local lookup
  4. Alternative: use d-left hashing for the ring → O(1) routing too

Implementation sketch for server-side cuckoo store:
*/

class DistributedNode {
private:
    CuckooHashMap<string, string> store_;  // cuckoo for O(1) worst-case lookup

public:
    // O(1) worst case — cuckoo guarantees this
    optional<string> get(const string& key) {
        return store_.get(key);
    }

    // O(1) amortised
    void put(const string& key, const string& val) {
        store_.insert(key, val);
    }

    // O(1) worst case
    bool del(const string& key) {
        return store_.erase(key);
    }
};

/*
Key discussion points for the interview:

1. WHY O(1) worst case matters for a server:
   A server handling 100K req/s with average O(1) lookup:
   0.1% of requests that hit O(log n) pathological cases = 100 slow req/s.
   A server with O(1) WORST CASE: zero slow requests, predictable p99 latency.

2. The load factor choice:
   Cuckoo at 45% load vs chaining at 75% load:
   Cuckoo uses ~2.2× the memory of chaining for the same n entries.
   Trade-off: memory for latency guarantee.

3. When to rehash:
   In a live server, cuckoo rehash is O(n) — cannot stall request processing.
   Solution: background rehash + double-buffering (serve from old table
   while new table is being built, then atomically swap).
*/
```

---

## 12. Real-World Uses

| Domain | System | Cuckoo Detail |
|---|---|---|
| **Network switches** | Intel DPDK, Open vSwitch | Flow table lookup: 2 DRAM accesses = O(1) hard guarantee at line rate |
| **Network security** | Snort, Suricata IDS | Packet signature matching: cuckoo for O(1) per-packet classification |
| **Database systems** | MemSQL (SingleStore) | In-memory hash join with O(1) worst-case probe |
| **Web caches** | CDN edge nodes | URL → backend server mapping; cuckoo for consistent low latency |
| **Cryptocurrency** | Ethereum Ethash PoW | Cuckoo Cycle: PoW puzzle based on finding cycles in a cuckoo graph |
| **Systems research** | RAMCloud (Stanford) | Key-value store: cuckoo for μs-latency O(1) hash table operations |
| **GPU databases** | RAPIDS cuDF | Parallel hash tables: cuckoo avoids warp divergence, O(1) worst case |
| **Compilers** | LLVM TableGen | Symbol table lookups during compilation |
| **Bloom filter alternative** | Cuckoo filters (Fan et al., 2014) | Better than Bloom: supports deletion, lower false positive rate |

### Open vSwitch — Cuckoo Hashing in the Linux Kernel Network Stack

Open vSwitch (OVS) processes millions of network flows per second in the Linux kernel. Each incoming packet must be classified — its header is hashed to look up the matching flow rule.

```
Flow table lookup requirements:
  - Rate: 10+ million lookups per second per CPU core
  - Latency: must complete before the network interface drops the packet
  - Guarantee: O(1) worst case — any slowdown causes packet loss

OVS implementation:
  Two hash functions: h1 = Jenkins hash, h2 = Murmur3
  Two tables with buckets of 4 entries each (4-way cuckoo)
  Each lookup: load 2 cache lines (one per table bucket), compare 4+4=8 entries
  Worst case: exactly 2 cache line loads regardless of table fullness

At 10Gbps line rate: 14.88 million 64-byte packets/second.
Each packet: 2 cache misses max.
At 100ns per cache miss: 200ns/packet — within the 67ns/packet budget at 10Gbps.

Without O(1) guarantee:
  Any O(n) lookup during high load → packets dropped → customers notice.
  This is why OVS uses cuckoo, not linear probing or chaining.
```

### Cuckoo Filters — Beyond Hash Tables

A **cuckoo filter** (Fan, Andersen, Kaminsky, Mitzenmacher, 2014) uses cuckoo hashing to build a set-membership structure that:
- Supports **deletion** (unlike Bloom filters)
- Achieves lower false positive rate per bit than Bloom filters
- Has O(1) worst-case membership test

```cpp
// Cuckoo filter: stores fingerprints (8-16 bit hashes) instead of full keys
class CuckooFilter {
    static const int BUCKETS = 1024;
    static const int SLOTS_PER_BUCKET = 4;
    uint16_t table_[BUCKETS][SLOTS_PER_BUCKET] = {};

    uint16_t fingerprint(const string& key) const {
        uint16_t fp = hash<string>{}(key) & 0xFFFF;
        return (fp == 0) ? 1 : fp;  // 0 = empty sentinel
    }

    int index1(const string& key) const { return hash<string>{}(key) % BUCKETS; }
    int index2(int i1, uint16_t fp) const {
        // Alternative index: i2 = i1 XOR h(fp)
        // Key property: from either index, you can compute the other!
        return (i1 ^ (fp * 0x5bd1e995)) % BUCKETS;
    }

public:
    bool insert(const string& key) {
        uint16_t fp = fingerprint(key);
        int i1 = index1(key), i2 = index2(i1, fp);

        // Try i1 first
        for (int s = 0; s < SLOTS_PER_BUCKET; s++) {
            if (table_[i1][s] == 0) { table_[i1][s] = fp; return true; }
        }
        // Try i2
        for (int s = 0; s < SLOTS_PER_BUCKET; s++) {
            if (table_[i2][s] == 0) { table_[i2][s] = fp; return true; }
        }
        // Both full — evict and relocate (cuckoo style)
        int cur_bucket = i1;
        uint16_t cur_fp = fp;
        for (int kick = 0; kick < 500; kick++) {
            int s = rand() % SLOTS_PER_BUCKET;
            swap(cur_fp, table_[cur_bucket][s]);
            cur_bucket = index2(cur_bucket, cur_fp);
            for (int ss = 0; ss < SLOTS_PER_BUCKET; ss++) {
                if (table_[cur_bucket][ss] == 0) {
                    table_[cur_bucket][ss] = cur_fp;
                    return true;
                }
            }
        }
        return false;  // table too full
    }

    bool contains(const string& key) const {
        uint16_t fp = fingerprint(key);
        int i1 = index1(key), i2 = index2(i1, fp);
        for (int s = 0; s < SLOTS_PER_BUCKET; s++) {
            if (table_[i1][s] == fp || table_[i2][s] == fp) return true;
        }
        return false;
    }

    bool remove(const string& key) {
        uint16_t fp = fingerprint(key);
        int i1 = index1(key), i2 = index2(i1, fp);
        for (int s = 0; s < SLOTS_PER_BUCKET; s++) {
            if (table_[i1][s] == fp) { table_[i1][s] = 0; return true; }
        }
        for (int s = 0; s < SLOTS_PER_BUCKET; s++) {
            if (table_[i2][s] == fp) { table_[i2][s] = 0; return true; }
        }
        return false;
    }
};
```

---

## 13. Edge Cases & Pitfalls

### Pitfall 1: Both Hash Functions Identical or Highly Correlated

```cpp
// WORST CASE: h1 = h2 means every key has only ONE effective candidate slot.
// Insertion degrades to linear probing; O(1) guarantee violated.

// WRONG:
int h1(int key) { return key % N; }
int h2(int key) { return key % N; }  // identical!

// With h1=h2, every key maps to the same two slots (t1[i] and t2[i]).
// These two slots form a tiny "cycle" immediately for any two keys with same hash.
// Insertion always fails after 2 kicks.

// CORRECT: use genuinely independent hash functions with different seeds.
int h1(int key) { return hash_with_seed(key, seed1_) % N; }
int h2(int key) { return hash_with_seed(key, seed2_) % N; }
// seed1_ ≠ seed2_, both random 64-bit values.
```

### Pitfall 2: MAX_KICKS Too Small

```cpp
// If MAX_KICKS is too small, legitimate insertions are incorrectly
// classified as cycles, triggering unnecessary rehashes.

// WRONG:
const int MAX_KICKS = 5;
// At load α=0.4, the expected eviction chain length is ~5.
// With MAX_KICKS=5, roughly half of insertions trigger rehash unnecessarily!

// CORRECT: MAX_KICKS should be O(log n).
// Theory: O(log n) kicks are sufficient to resolve any non-cyclic insertion.
const int MAX_KICKS = 2 * (int)(log2(n_) + 1);
// For n=1024: MAX_KICKS = 22. For n=1M: MAX_KICKS = 42.
// In practice: values of 20-50 work well for n up to millions.
```

### Pitfall 3: Rehash Not Randomising Hash Seeds

```cpp
// WRONG: rehash with same hash functions
void rehash() {
    // ... rebuild table with same h1, h2 ...
    // If the current set of keys caused a cycle with seed1, seed2,
    // they will ALWAYS cause the same cycle.
    // Rehash loop: insert → cycle → rehash → same cycle → infinite loop!
}

// CORRECT: rehash with NEW RANDOM SEEDS
void rehash() {
    seed1_ = random_seed();   // ← critical: new seeds change h1 and h2
    seed2_ = random_seed();   // ← same keys now map to different slots
    // ... rebuild table ...
}
// With new random seeds, the probability of the same cycle occurring
// with the same key set is negligibly small.
```

### Pitfall 4: Forgetting to Check Both Tables During Delete

```cpp
// WRONG: only deleting from table1
void erase(int key) {
    int i1 = h1(key);
    if (t1_[i1] == key) t1_[i1] = EMPTY;
    // Forgot table2! Key may be in t2_[h2(key)].
}

// CORRECT: check both tables (and stash)
void erase(int key) {
    int i1 = h1(key);
    if (t1_[i1] == key) { t1_[i1] = EMPTY; return; }
    int i2 = h2(key);
    if (t2_[i2] == key) { t2_[i2] = EMPTY; return; }
    stash_erase(key);
}
```

### Pitfall 5: Wrong Eviction Alternation

```cpp
// The eviction chain must ALTERNATE between tables.
// If evicted from table1 → go to table2.
// If evicted from table2 → go to table1.

// WRONG: always trying table1 first after eviction
for (int kick = 0; kick < MAX_KICKS; kick++) {
    int i1 = h1(cur_key);
    if (t1[i1] empty) { place here; return; }
    swap(cur_key, t1[i1]);   // evict from t1
    // BUG: always trying t1 next! Evicted key needs to try t2.
    // This creates a 2-cycle: t1[i] → evict → t1[i] → evict → ...
}

// CORRECT: track which table the evicted key came from
bool in_t1 = true;  // start by trying table1
for (int kick = 0; kick < MAX_KICKS; kick++) {
    if (in_t1) {
        if (t1[h1(k)] empty) { place in t1; return; }
        swap(k, t1[h1(k)]); in_t1 = false;  // evicted from t1, next try t2
    } else {
        if (t2[h2(k)] empty) { place in t2; return; }
        swap(k, t2[h2(k)]); in_t1 = true;   // evicted from t2, next try t1
    }
}
```

### Pitfall 6: Load Factor Computation Counting Both Tables

```cpp
// WRONG: computing load as size / n (one table's capacity)
float load_factor() { return (float)size_ / n_; }
// This shows load = 1.0 when size = n (half the total capacity used).
// With this computation, you'd rehash far too early (at 50% of total capacity).

// CORRECT: load = size / (2*n) = fraction of total combined capacity
float load_factor() { return (float)size_ / (2.0f * n_); }
// Rehash when this exceeds 0.45 (for 2-choice cuckoo).
// At 45% combined load: each individual table is ~45% full.
// Well below the 50% phase transition threshold.
```

### Pitfall 7: Eviction Chain Losing the Inserted Key

```cpp
// Subtle bug: if the swap semantics are wrong, you might lose the key being inserted.

// WRONG: after swapping, continuing with the wrong key
K evicted_key;
t1[h1(new_key)] = new_key;  // place new key
evicted_key = old_occupant;  // save evicted
// ... now try to place new_key in t2? But new_key is already in t1!
// We should be placing evicted_key in t2.

// CORRECT: after evicting, the EVICTED key needs a new home.
K cur = new_key;  // cur = the key currently needing placement
while (...) {
    swap(cur, t1[h1(cur)]);  // cur is now the EVICTED key
    // now try to place cur (the evicted one) in t2
    if (t2[h2(cur)] empty) { place cur in t2; return; }
    swap(cur, t2[h2(cur)]);  // cur is now the NEW evicted key
    // loop: place cur in t1 again
}
```

---

## 14. Comparison: Cuckoo vs Other Hash Tables

| Property | Cuckoo | Linear Probing | Double Hashing | Chaining |
|---|---|---|---|---|
| Lookup worst case | **O(1) GUARANTEED** | O(n) | O(n) | O(n) |
| Lookup average | O(1) | O(1) | O(1) | O(1) |
| Insert worst case | O(n) on rehash | O(n) | O(n) | O(n) on rehash |
| Insert amortised | O(1) | O(1) | O(1) | O(1) |
| Delete worst case | **O(1) GUARANTEED** | O(1) + tomb | O(1) + tomb | O(1) |
| Memory per entry | 2 slots (2× overhead) | 1 slot (1/α overhead) | 1 slot | 1 slot + ptr |
| Max load factor | 0.50 per table (0.90 total w/ stash) | 0.70 | 0.80 | > 1.0 |
| Memory efficiency | 50% utilisation per table | 70% | 80% | 100%+ |
| Cache misses/lookup | 2 (different cache lines) | 1-2 (sequential) | 2 (random) | 1+ (chain) |
| Tombstones needed | No | Yes | Yes | No |
| Stash needed | Yes (for reliability) | No | No | No |
| Clustering | None | Primary | None | None |
| Implementation complexity | High | Low | Moderate | Low |
| Hash function requirements | 2 independent | 1 good | 2 independent | 1 good |
| Rehash strategy | New seeds + retry | Straightforward | Straightforward | Straightforward |
| Real-time safety | **Yes** | No (O(n) worst) | No | No |
| GPU-friendly | Yes | No (divergence) | Yes | No |

**The definitive use-case decision:**

```
Use CUCKOO when:
  ✓ O(1) worst-case lookup is a hard requirement (not just average)
  ✓ Real-time systems where tail latency matters (p99, p99.9)
  ✓ Network data plane (packet classification, flow tables)
  ✓ GPU parallel hash tables (no warp divergence)
  ✓ Delete is frequent (no tombstones accumulate)
  ✓ You need a Bloom-filter-like structure that supports deletion (cuckoo filter)

Do NOT use CUCKOO when:
  ✗ Memory is tight (2× overhead is unacceptable)
  ✗ Load factor > 50% is needed with 2-choice (use d-ary cuckoo or chaining)
  ✗ Insert latency spikes are unacceptable (rehash = O(n) blocking)
  ✗ Hash function computation is expensive (2 evaluations per probe vs 1)
  ✗ Implementation complexity is a concern (use linear probing or chaining)
  ✗ The key set is adversarial without random seeds (use randomised cuckoo)
```

---

## 15. Self-Test Questions

1. **State the cuckoo hashing invariant. Explain why O(1) worst-case lookup follows directly from this invariant — without needing any probabilistic argument.**

2. **Trace the complete eviction chain for inserting key `E` into a 2-choice cuckoo table where `h1(E)=2, h2(E)=1`, and the current state is: t1=[_,A,B,_], t2=[C,D,_,_] with `h1(B)=2, h2(B)=3, h1(D)=1, h2(D)=0`. Show each swap and the table state after each kick.**

3. **Explain the cuckoo graph construction. When does an insertion fail? State the graph-theoretic condition precisely (in terms of edges, vertices, and connected components).**

4. **Why must MAX_KICKS be Ω(log n) and not O(1)? What happens if you set MAX_KICKS = 3 for a table of 1 million entries?**

5. **A cuckoo hash table has a cycle and triggers a rehash. Why must the new hash functions use different random seeds rather than just rebuilding with the same seeds? What is the probability that the same keys cause a cycle again with new random seeds?**

6. **Explain the stash mechanism. A stash of size 4 reduces insertion failure probability to essentially zero. Prove this using the formula P(failure) < n × (1/(2n))^(s+1) for n=10^6, s=4.**

7. **Compare 2-choice and 3-choice cuckoo hashing. What is the maximum load factor for each? What is the lookup cost? When would you choose 3-choice over 2-choice?**

8. **A cuckoo filter stores fingerprints (not full keys). During deletion, you delete a fingerprint from one of two buckets. What is the subtle correctness problem that can arise, and how does the fingerprint design prevent false negatives?**

9. **In the Open vSwitch flow table, why is cuckoo hashing chosen over linear probing despite linear probing's superior cache behaviour? Quantify the cache miss count per lookup for each approach and explain which matters more for line-rate packet processing.**

10. **Design a lock-free concurrent cuckoo hash table for a multi-threaded server. What operations need atomic guarantees? How do you prevent a reader from seeing a key in neither table during an eviction? (Hint: consider hazard pointers or epoch-based reclamation.)**

---

## Quick Reference Card

```
Cuckoo Hashing — O(1) WORST-CASE lookup. The only open addressing strategy
                 with this guarantee.

Structure: 2 tables of n slots each + optional stash of s entries.
  Key k is in EXACTLY ONE of: t1[h1(k)] or t2[h2(k)] (or stash).

Lookup (O(1) GUARANTEED, always 2 accesses):
  if t1[h1(k)].key == k: return t1[h1(k)].val
  if t2[h2(k)].key == k: return t2[h2(k)].val
  scan stash (at most s=4 entries)
  return not_found

Insert (O(1) amortised, O(n) on rehash):
  Eviction loop (MAX_KICKS = O(log n)):
    Try t1[h1(cur)]: empty → place. Done.
    Evict occupant x. Place cur. cur = x. Try t2[h2(cur)] next.
    (Alternate t1↔t2 on each kick)
  If loop exhausted: add to stash or rehash.

Delete (O(1) GUARANTEED, 2 accesses):
  If t1[h1(k)].key == k: mark empty.
  elif t2[h2(k)].key == k: mark empty.
  else: remove from stash.

Key requirements:
  h1 and h2 must be INDEPENDENT (use different random seeds)
  MAX_KICKS = O(log n) (theory: 2*log2(n) + 1 is sufficient)
  Rehash with NEW RANDOM SEEDS when cycle detected

Phase transition at α = 0.5 per table:
  α < 0.49: stable, insertions O(1) amortised
  α = 0.50: phase transition — insertion failures become frequent
  Keep α ≤ 0.45 per table (total ≤ 0.90 with stash)

Stash: a safety net of s=4-8 entries for unavoidable cycles.
  With s=4: P(failure after n inserts) < n × (1/2n)^5 ≈ 0 for any n.
  Lookup cost with stash: 2 table accesses + ≤ s linear comparisons.

Variants:
  d-ary cuckoo (d=3): 3 tables, max load ~91%, lookup = 3 accesses
  Blocked cuckoo: buckets of b=4 keys per position, max load ~95%
  Cuckoo filter: stores fingerprints, supports deletion unlike Bloom

Pitfall checklist:
  ✗ h1 = h2 or correlated → every key has one effective slot → O(n) chains
  ✗ MAX_KICKS too small → spurious rehashes
  ✗ Rehash without new seeds → same cycle forever → infinite loop
  ✗ Delete only checks t1 (not t2 and stash) → ghost entries
  ✗ Wrong alternation (always t1 after eviction) → immediate 2-cycles
  ✗ Load factor as size/n (not size/2n) → rehash threshold wrong

When O(1) worst-case matters:
  Network switches, packet classifiers, real-time databases,
  GPU hash tables, any system where p99 latency = p50 latency.
```

---

*Previous: [18 — Double Hashing](./18_double_hashing.md)*
*Next: [19 — Robin Hood Hashing](./19_robin_hood.md)*
*See also: [Bloom Filter](./probabilistic/bloom_filter.md) | [Cuckoo Filter](./probabilistic/cuckoo_filter.md) | [Open vSwitch flow tables](https://docs.openvswitch.org)*

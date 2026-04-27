# Collision Handling: Double Hashing

> **Curriculum position:** Hash-Based Structures → #5
> **Interview weight:** ★★★☆☆ Medium — the theoretical optimum of open addressing; frequently asked in systems design and algorithm depth questions
> **Difficulty:** Intermediate-Advanced — the concept requires two hash functions and the coprimality constraint; the mathematical guarantees are the deepest in the open addressing family
> **Prerequisite:** [16 — Linear Probing](./16_linear_probing.md) | [17 — Quadratic Probing](./17_quadratic_probing.md)

---

## Table of Contents

1. [Intuition First — Two Keys, Two Paths](#1-intuition-first)
2. [The Probe Sequence — Formula and Derivation](#2-the-probe-sequence)
3. [The Coprimality Constraint — Why It Guarantees Full Coverage](#3-the-coprimality-constraint)
4. [Designing h2 — The Five Standard Approaches](#4-designing-h2)
5. [Eliminating Both Clustering Types](#5-eliminating-both-clustering-types)
6. [Deletion and Tombstones](#6-deletion-and-tombstones)
7. [Load Factor and Performance](#7-load-factor-and-performance)
8. [Time & Space Complexity](#8-time--space-complexity)
9. [Complete C++ Implementation](#9-complete-c-implementation)
10. [Core Operations — Visualised](#10-core-operations--visualised)
11. [Interview Problems](#11-interview-problems)
12. [Real-World Uses](#12-real-world-uses)
13. [Edge Cases & Pitfalls](#13-edge-cases--pitfalls)
14. [Comparison: The Full Open Addressing Family](#14-comparison)
15. [Self-Test Questions](#15-self-test-questions)

---

## 1. Intuition First

Linear probing walks one step at a time from the collision point — every key blocked at the same bucket takes the same sequential detour, and their paths merge into ever-growing clusters. Quadratic probing jumps geometrically, breaking primary clusters but leaving secondary ones: two keys that share the same initial bucket still follow the exact same detour route.

**Double hashing eliminates the "same route" problem entirely.** It gives every key its own unique step size, derived from a *second* independent hash function.

Think of a hotel with 1000 rooms. Your assigned room is full. With linear probing, everyone goes to room+1, then room+2 — the lobby becomes a long queue. With quadratic probing, different wings of the hotel are tried — still, two guests assigned the same original room take the same sequence of alternative wings. With double hashing, your step size is derived from your own personal booking code — a second identifier that is different for every guest. Two guests originally assigned the same room now diverge immediately at the first probe and follow entirely independent paths.

```
Linear:    probe sequence identical for ALL keys with same home → primary clustering
Quadratic: probe sequence identical for keys with same HOME HASH → secondary clustering
Double:    probe sequence identical only for keys with SAME HOME AND SAME h2 → minimal clustering
```

The step size for double hashing is:

```
step = h2(key)
probe at: h1(key), h1(key)+step, h1(key)+2×step, h1(key)+3×step, ... (all mod m)
```

The only requirement: `step` must never be 0 (no movement) and must be **coprime** to the table capacity `m` (ensures the probe visits every slot before cycling). When these conditions hold, double hashing achieves the best distribution of any open addressing strategy, approaching the theoretical ideal of **uniform hashing** — where each probe is as if chosen independently at random.

The trade-off: two hash function evaluations per probe instead of one. For expensive hash functions (long strings, complex objects), this doubles the hashing cost. For cheap hash functions (integers), the overhead is negligible.

---

## 2. The Probe Sequence — Formula and Derivation

### The Formula

```
h1(key) = primary hash → initial bucket
h2(key) = secondary hash → step size

Probe sequence for key k:
  slot(k, 0) = h1(k)                    mod m
  slot(k, 1) = h1(k) +   h2(k)          mod m
  slot(k, 2) = h1(k) + 2×h2(k)         mod m
  slot(k, 3) = h1(k) + 3×h2(k)         mod m
  slot(k, i) = (h1(k) + i × h2(k))     mod m
```

### Why This Sequence Differs Per Key

With linear probing: `slot(k, i) = h1(k) + i`. Two keys `a` and `b` with `h1(a) = h1(b)` follow `h1, h1+1, h1+2, ...` — identical.

With double hashing: `slot(k, i) = h1(k) + i × h2(k)`. Even if `h1(a) = h1(b)`, if `h2(a) ≠ h2(b)`, their sequences diverge immediately after the first probe:

```
Key a: h1=5, h2=3 → probes: 5, 8, 11, 14, 2, ...
Key b: h1=5, h2=7 → probes: 5, 12, 3, 10, 1, ...

After slot 5 (both start here), they go completely different directions.
NO clustering. NO shared detour. Truly independent probe sequences.
```

### The Arithmetic Progression

Each probe sequence `h1, h1+h2, h1+2h2, ...` is an **arithmetic progression** in modular arithmetic. The entire sequence is determined by two numbers: the starting position `h1` and the common difference `h2`. 

The key question: does this arithmetic progression visit all `m` slots before repeating? The answer depends on the relationship between `h2` and `m` — specifically, their greatest common divisor.

---

## 3. The Coprimality Constraint — Why It Guarantees Full Coverage

### The Fundamental Theorem

> **An arithmetic progression `{a, a+d, a+2d, ...} mod m` visits exactly `m / gcd(d, m)` distinct values before cycling.**

**Proof sketch:**

The sequence visits value `a + k×d` for `k = 0, 1, 2, ...`. It returns to the starting point when `k×d ≡ 0 (mod m)`, i.e., when `k×d` is divisible by `m`. The smallest such positive `k` is `m / gcd(d, m)`.

Therefore:
- If `gcd(d, m) = 1` (d and m are coprime): visits `m/1 = m` distinct slots — the full table.
- If `gcd(d, m) = 2`: visits only `m/2` distinct slots.
- If `gcd(d, m) = m`: `d ≡ 0 (mod m)`, i.e., step = 0 → revisits only the starting slot!

### Concrete Examples

```
m = 10:

Step h2=3: gcd(3,10)=1 → visits 10/1=10 distinct slots. GOOD.
  Sequence from 0: 0,3,6,9,2,5,8,1,4,7 → all 10 ✓

Step h2=5: gcd(5,10)=5 → visits 10/5=2 distinct slots. BAD.
  Sequence from 0: 0,5,0,5,... → only 2 slots!

Step h2=4: gcd(4,10)=2 → visits 10/2=5 distinct slots. BAD.
  Sequence from 0: 0,4,8,2,6,0,4,... → only 5 slots!

Step h2=7: gcd(7,10)=1 → visits all 10 slots. GOOD.
  Sequence from 0: 0,7,4,1,8,5,2,9,6,3 → all 10 ✓
```

### The Three Standard Approaches to Guarantee Coprimality

**Approach 1: Prime table capacity**

If `m` is prime, then `gcd(h2, m) = 1` for any `h2` in `[1, m-1]`. So any step from 1 to m-1 works. Simplest guarantee.

```
m = 11 (prime):
  h2 = 1: gcd(1,11) = 1 ✓ visits all 11
  h2 = 5: gcd(5,11) = 1 ✓ visits all 11
  h2 = 10: gcd(10,11) = 1 ✓ visits all 11
  All non-zero steps work!
```

**Approach 2: Power-of-2 capacity + odd step**

If `m` is a power of 2, then `h2` must be **odd** (`gcd(odd, 2^k) = 1` since the odd number has no factor of 2).

```
m = 8 = 2^3:
  h2 = 3 (odd): gcd(3,8)=1 ✓
  h2 = 5 (odd): gcd(5,8)=1 ✓
  h2 = 4 (even): gcd(4,8)=4 → only 2 slots visited! ✗
  h2 = 6 (even): gcd(6,8)=2 → only 4 slots visited! ✗

Guarantee: h2(key) = 2×something + 1  (force odd)
```

**Approach 3: Prime capacity + h2 = prime - (key mod prime)**

A classical textbook formulation:

```
m = prime_1  (table capacity)
p = prime_2  (a different prime, typically smaller than m)

h1(k) = k mod m
h2(k) = p - (k mod p)   → always in [1, p], coprime to m since gcd(h2, m) = gcd(p-x, m)
                           and for carefully chosen p, this gives good distribution.

Example: m=11, p=7:
  k=15: h1=15%11=4, h2=7-(15%7)=7-1=6. gcd(6,11)=1 ✓
  k=26: h1=26%11=4, h2=7-(26%7)=7-5=2. gcd(2,11)=1 ✓
  Both h1=4, but h2=6 and h2=2 → different probe sequences from slot 4!
```

---

## 4. Designing h2 — The Five Standard Approaches

The choice of second hash function critically affects both the distribution quality and the practical performance.

### Approach 1: Division-Based (Classic Textbook)

```cpp
// For integer keys, prime table size m:
size_t h2(int key, size_t prime_p) {
    // prime_p < m, prime_p ≠ m
    return prime_p - (key % prime_p);
    // Result: always in [1, prime_p] ⊆ [1, m-1]
    // gcd(h2, m) is typically 1 when m is prime
}

// Example: m=11, prime_p=7
// h2(0) = 7-0=7, h2(7) = 7-0=7, h2(14) = 7-0=7  (all map to 7 for multiples of 7)
// h2(1) = 7-1=6, h2(8) = 7-1=6, h2(15) = 7-1=6
// Better: different residues mod prime_p give different step sizes
```

### Approach 2: Fibonacci/Knuth Multiplicative

```cpp
// Multiplicative method: multiply key by golden ratio constant
size_t h2_multiplicative(size_t key, size_t m) {
    // Use a different constant than h1 to ensure independence
    // A = 2654435761 (a different Knuth constant) = 2^32 / phi²
    size_t hash = key * 2654435761ULL;
    // Force odd (for power-of-2 m) by setting least-significant bit:
    return (hash >> (64 - (int)log2(m))) | 1;  // odd step for 2^k table
}
```

### Approach 3: FNV Variant (For String Keys)

```cpp
// For string keys: use two independent FNV-1a constants
size_t h1_string(const string& s, size_t m) {
    size_t hash = 14695981039346656037ULL;  // FNV offset basis 1
    for (char c : s) hash = (hash ^ (unsigned char)c) * 1099511628211ULL;
    return hash % m;
}

size_t h2_string(const string& s, size_t m) {
    size_t hash = 2166136261ULL;            // FNV offset basis 2 (different)
    for (char c : s) hash = (hash ^ (unsigned char)c) * 16777619ULL;
    size_t step = hash % m;
    return (step == 0) ? 1 : step;  // never return 0
}
```

### Approach 4: SipHash Pair (Cryptographic Quality)

```cpp
// For security-sensitive applications (prevent hash flooding attacks):
// Use SipHash-2-4 with two different key schedules.
// Two independent 128-bit seeds produce completely independent hash functions.

pair<size_t, size_t> siphash_double(const void* data, size_t len,
                                     uint64_t seed1, uint64_t seed2) {
    size_t h1 = siphash24(data, len, seed1) % m;
    size_t h2 = siphash24(data, len, seed2) % m;
    if (h2 == 0) h2 = 1;
    return {h1, h2};
}
```

### Approach 5: Wang Hash + Complement (For Integer Keys, Simple)

```cpp
// Fast integer double hash: one function, two independent outputs
// Uses bit-mixing to get two independent values from one integer key
size_t wang_h1(size_t key, size_t m) {
    key = (~key) + (key << 21);
    key = key ^ (key >> 24);
    key = (key + (key << 3)) + (key << 8);
    key = key ^ (key >> 14);
    return key % m;
}

size_t wang_h2(size_t key, size_t m) {
    // Use a different mixing sequence
    key = key ^ (key >> 30);
    key *= 0xbf58476d1ce4e5b9ULL;
    key = key ^ (key >> 27);
    key *= 0x94d049bb133111ebULL;
    key = key ^ (key >> 31);
    size_t step = key % m;
    return (step == 0) ? 1 : step;  // ensure non-zero
}
```

### Summary: h2 Design Rules

```
h2 MUST satisfy:
  1. h2(key) ≠ 0 for all keys          (zero step means no movement → infinite loop)
  2. gcd(h2(key), m) = 1               (coprimality ensures all slots reachable)
  3. h2 is independent from h1         (different probe sequences for different keys)
  4. h2 distributes uniformly in [1,m) (good spread, avoid clustering)

h2 MUST NOT:
  - Return the same value for all keys  (secondary clustering if h2 is constant)
  - Return values that share a common factor with m (reduces slots reachable)
  - Be a simple multiple of h1          (strong correlation → clustering)
```

---

## 5. Eliminating Both Clustering Types

This section is the theoretical core of double hashing and the main reason it is studied.

### Primary Clustering (Linear Probing)

```
Cause: every key blocked at bucket b probes b, b+1, b+2, ...
       Keys with DIFFERENT initial buckets b and b+3 can extend
       each other's probe runs if their runs happen to be adjacent.

Double hashing fix: different keys take completely different directions.
  Key a (h1=5, h2=3): 5, 8, 11, 14, ...
  Key b (h1=8, h2=5): 8, 13, 18, 23, ...

  Even though b starts at slot 8 (where a's second probe landed),
  b's step is 5 (not 3), so their subsequent paths diverge.
  b does NOT extend a's cluster.

→ PRIMARY CLUSTERING ELIMINATED.
```

### Secondary Clustering (Quadratic Probing)

```
Cause: all keys with the same initial hash h1 follow the same probe
       sequence: h1, h1+1², h1+2², h1+3², ...

Double hashing fix: even keys with the same h1 diverge if they have different h2.
  Key a (h1=5, h2=3): 5, 8, 11, 14, 2, ...
  Key b (h1=5, h2=7): 5, 12, 3, 10, 1, ...

  Both start at slot 5, but their SECOND probes are slots 8 and 12.
  They immediately diverge. Their only shared probe is slot 5 itself.

→ SECONDARY CLUSTERING REDUCED TO: keys sharing BOTH h1 AND h2 (very rare).
```

### What Clustering Remains?

With double hashing, two keys cluster only if they share the same `(h1, h2)` pair — both their initial bucket and their step size are identical. The probability of this is approximately `1/m²` (if both hash functions are independent and uniform). For a table of 1000 buckets, this is a `0.0001%` chance — essentially negligible.

```
Probability of true clustering in double hashing:
  = P(h1(a) = h1(b)) × P(h2(a) = h2(b))
  ≈ (1/m) × (1/m)
  = 1/m²

For m=1000: P ≈ 0.000001 — one collision per million key pairs.
This is why double hashing is said to approximate UNIFORM HASHING.
```

### The Uniform Hashing Ideal

Uniform hashing is the theoretical model where each of the `m!` permutations of the table slots is equally likely as the probe sequence for any key. It is not achievable in practice (would require storing a full random permutation per key), but double hashing approximates it closely. The expected probe lengths under uniform hashing are:

```
E[probes, successful, uniform]   = (1/α) × ln(1/(1-α))
E[probes, unsuccessful, uniform] = 1/(1-α)

For α=0.5:  successful ≈ 1.39, unsuccessful = 2.0
For α=0.7:  successful ≈ 1.58, unsuccessful = 3.33
For α=0.9:  successful ≈ 2.56, unsuccessful = 10.0

These are LOWER BOUNDS — the best any hashing scheme can do.
Double hashing approaches these numbers closely in practice.
```

---

## 6. Deletion and Tombstones

Double hashing has the same deletion challenge as linear and quadratic probing: marking a deleted slot as EMPTY breaks the probe chain for keys placed beyond it.

### Why Backward Shift Is Infeasible for Double Hashing

Recall that linear probing can use a backward-shift deletion algorithm (no tombstones needed) because the probe sequence is a simple arithmetic progression where "belongs before position i" has a clean geometric interpretation.

For double hashing, determining whether a key "belongs before" a given gap requires knowing that key's `h2` value — which means you would need to know every key's step size to perform the shift correctly. This makes backward-shift deletion impractical (you would need to store h2 per slot, adding 8 bytes of overhead per entry — defeating the memory advantage of open addressing).

```
Why backward shift fails for double hashing:

Linear probing: key at slot j with home h belongs before gap i
  if h ≤ i (in the non-wrapped case). Simple! No knowledge of step needed.

Double hashing: key at slot j was placed there by following its unique step h2.
  To know if it "belongs before" gap i, you need to know its probe sequence,
  which requires its h2 value.
  h2 is computed from the key, so you'd need the key anyway — but if you
  have the key, you can just re-insert it. This makes the algorithm
  equivalent to "delete + re-insert everything", which is O(n) per deletion.

→ Tombstones are the only practical deletion strategy for double hashing.
```

### Tombstone Management

Identical to linear and quadratic probing: DELETED slots are skipped during lookup (probe continues) but can be reused during insertion (first tombstone is the insertion point).

```cpp
// The critical sequence during insertion:
// 1. Probe the sequence until EMPTY or key found.
// 2. Track the FIRST tombstone encountered.
// 3. If key not found → insert at first tombstone (if seen) else at EMPTY.
// 4. Decrement tombstones_ counter when a tombstone is reused.

// Periodic rehash removes tombstones — same trigger as other open addressing:
// rehash when (size + tombstones) / capacity > max_load
```

---

## 7. Load Factor and Performance

### Expected Probe Lengths

Double hashing's probe lengths, under the assumption that h1 and h2 are independent and uniform, approach the uniform hashing ideal:

```
Load factor α   E[probes, successful]   E[probes, unsuccessful]
──────────────────────────────────────────────────────────────────────
0.10             1.05                     1.11
0.25             1.14                     1.33
0.50             1.39                     2.00        ← uniform hashing ideal
0.60             1.53                     2.50
0.70             1.58 (actual)            3.33
0.75             1.70                     4.00
0.80             1.80                     5.00
0.90             2.56                     10.00
0.95             3.15                     20.00

For comparison at α=0.7:
  Linear probing:    2.17 successful, 5.67 unsuccessful
  Quadratic probing: 1.72 successful, 3.39 unsuccessful
  Double hashing:    1.58 successful, 3.33 unsuccessful ← best open addressing
  Chaining (α=0.7):  1.35 successful, 0.70 unsuccessful ← chaining wins on unsuccessful!
```

### The Performance Advantage Over Quadratic Probing

At high load factors, double hashing's advantage over quadratic probing becomes significant:

```
α=0.9:
  Quadratic: E[success] ≈ 2.85, E[unsuccess] ≈ ?
  Double:    E[success] ≈ 2.56, E[unsuccess] ≈ 10.0

  10% improvement on successful lookups.
  Double hashing can safely operate at α=0.9 where quadratic probing
  would need to rehash (beyond its 0.5 safety limit).
  This memory efficiency advantage is the main reason double hashing
  is used in high-load scenarios.
```

### The Cache Performance Disadvantage

```
Linear probing: probes at adjacent slots → hardware prefetcher works perfectly.
  After loading slot h, the CPU already has h+1, h+2, ... in the L1 cache.

Double hashing: probes at h, h+h2, h+2h2, ... with large, irregular steps.
  Each probe is likely a cache miss.
  At h2=373, successive probes are 373 slots apart = completely different cache lines.

Practical consequence:
  Linear probing at α=0.5: ~3-5 ns per lookup (mostly cache hits)
  Double hashing at α=0.5: ~15-25 ns per lookup (mostly cache misses)
  5-8× slower in wall-clock time despite better algorithmic probe counts.

When double hashing wins in practice:
  → Large values (structs > 64 bytes): cache misses are unavoidable anyway
  → Very high load (α > 0.7): double hashing's fewer probes compensate
  → Memory-constrained: can store more entries per megabyte than linear probing
  → Adversarial inputs: double hashing's independent probe sequences resist attacks
```

---

## 8. Time & Space Complexity

### Operation Complexities

| Operation | Average (α ≤ 0.8) | Worst case | Notes |
|---|---|---|---|
| `insert(k, v)` | **O(1)** | O(m) | O(1/(1-α)) expected |
| `lookup(k)` successful | **O(1)** | O(m) | Approaches uniform hashing ideal |
| `lookup(k)` unsuccessful | **O(1)** | O(m) | E[probes] = 1/(1-α) |
| `delete(k)` | **O(1)** | O(m) | Tombstone + lookup cost |
| `rehash(m')` | O(n) | O(n²) | Re-insert all entries; rare |
| `build` from n entries | O(n) avg | O(n²) | n insertions |
| Iteration | O(m) | O(m) | Scan all m slots |

**Two hash evaluations per probe:** Each step in the probe sequence requires evaluating only h1 once (at the start) and incrementing by h2 (also computed once at the start). The total cost is **h1 once + h2 once + linear iteration** — not "two evaluations per probe" as sometimes stated.

```
Cost structure:
  Start of search: compute h1(key) and h2(key)  → 2 hash evaluations (once total)
  Each probe step: slot = (current + h2) % m    → 1 addition + 1 modulo per step
  Total for k probes: 2 hash evaluations + k arithmetic operations
```

### Space Complexity

| Component | Space | Notes |
|---|---|---|
| Slot array | O(m) = O(n/α) | Contiguous, single allocation |
| Per-entry overhead | **0 bytes** | No pointers, no stored h2 |
| State per slot | 2 bits | EMPTY / OCCUPIED / DELETED |
| Total effective space | `n × (key+value) / α` | At α=0.8: 25% overhead |
| vs chaining (α=0.8) | Chaining: 8 bytes/entry extra | Double hashing wins memory |

---

## 9. Complete C++ Implementation

```cpp
#include <iostream>
#include <vector>
#include <optional>
#include <functional>
#include <stdexcept>
#include <cmath>
#include <string>
using namespace std;

// ── Second hash function helper ────────────────────────────────────────────────
// Provides a second, independent hash function for double hashing.
// The result is always ≥ 1 and coprime with m (when m is prime, any value works).
template<typename K>
struct SecondHash {
    size_t operator()(const K& key) const {
        // Default: a different multiplicative constant from std::hash
        // (std::hash typically uses one Knuth constant; we use another)
        size_t h = hash<K>{}(key);
        // Mix the bits to get a value independent from h1
        h ^= (h >> 17);
        h *= 0xbf58476d1ce4e5b9ULL;
        h ^= (h >> 31);
        return h;
    }
};

// ── Double Hashing Hash Map ────────────────────────────────────────────────────
template<typename K, typename V,
         typename Hash1 = hash<K>,
         typename Hash2 = SecondHash<K>,
         typename Equal = equal_to<K>>
class DoubleHashMap {
private:
    enum class State : uint8_t { EMPTY, OCCUPIED, DELETED };

    struct Slot {
        K     key;
        V     value;
        State state = State::EMPTY;
    };

    vector<Slot> table_;
    size_t       size_;
    size_t       capacity_;
    size_t       tombstones_;
    float        max_load_;
    Hash1        h1_;
    Hash2        h2_;
    Equal        equal_;

    // ── Helpers ───────────────────────────────────────────────────────────────

    static bool is_prime(size_t n) {
        if (n < 2) return false;
        if (n == 2 || n == 3) return true;
        if (n % 2 == 0 || n % 3 == 0) return false;
        for (size_t i = 5; i * i <= n; i += 6)
            if (n % i == 0 || n % (i + 2) == 0) return false;
        return true;
    }

    static size_t next_prime(size_t n) {
        size_t p = (n < 2) ? 2 : (n % 2 == 0 ? n + 1 : n);
        while (!is_prime(p)) p += 2;
        return p;
    }

    // Primary hash: slot in [0, m)
    size_t slot1(const K& key) const {
        return h1_(key) % capacity_;
    }

    // Secondary hash: step size in [1, m)
    // CRITICAL: must never return 0 (infinite loop) and must be coprime with m.
    // With prime capacity, any value in [1, m-1] is coprime.
    size_t step(const K& key) const {
        size_t s = h2_(key) % capacity_;
        return (s == 0) ? 1 : s;   // guard against zero step
    }

    // Find the slot for key. Returns {index, found}.
    // found=true:  table_[index] is OCCUPIED with matching key
    // found=false: table_[index] is the best insertion point
    pair<size_t, bool> find_slot(const K& key) const {
        size_t h    = slot1(key);
        size_t d    = step(key);
        size_t tomb = capacity_;   // first tombstone index (capacity = "none seen")
        size_t idx  = h;

        for (size_t probe = 0; probe < capacity_; probe++) {
            const Slot& s = table_[idx];

            if (s.state == State::EMPTY) {
                return {tomb < capacity_ ? tomb : idx, false};
            }
            if (s.state == State::DELETED) {
                if (tomb == capacity_) tomb = idx;
            } else if (equal_(s.key, key)) {
                return {idx, true};
            }

            idx = (idx + d) % capacity_;
        }

        // All slots probed — use tombstone if available, else table is full
        if (tomb < capacity_) return {tomb, false};
        throw overflow_error("double hash map: all slots probed, table full");
    }

    bool needs_rehash() const {
        return (float)(size_ + tombstones_) / capacity_ > max_load_;
    }

    void do_rehash(size_t hint) {
        size_t new_cap = next_prime(hint);
        vector<Slot> old = move(table_);
        table_.assign(new_cap, Slot{});
        capacity_   = new_cap;
        size_       = 0;
        tombstones_ = 0;

        for (auto& slot : old) {
            if (slot.state == State::OCCUPIED)
                insert(move(slot.key), move(slot.value));
        }
    }

public:
    // ── Construction ──────────────────────────────────────────────────────────

    explicit DoubleHashMap(size_t initial_cap = 11, float max_load = 0.70f)
        : table_(next_prime(initial_cap), Slot{}),
          size_(0),
          capacity_(next_prime(initial_cap)),
          tombstones_(0),
          max_load_(max_load) {}

    ~DoubleHashMap() = default;

    DoubleHashMap(const DoubleHashMap&)            = delete;
    DoubleHashMap& operator=(const DoubleHashMap&) = delete;
    DoubleHashMap(DoubleHashMap&&)                 = default;
    DoubleHashMap& operator=(DoubleHashMap&&)      = default;

    // ── Core Operations ───────────────────────────────────────────────────────

    void insert(const K& key, const V& val) {
        if (needs_rehash()) do_rehash(capacity_ * 2);
        auto [idx, found] = find_slot(key);
        if (found) {
            table_[idx].value = val;
        } else {
            bool was_tomb = (table_[idx].state == State::DELETED);
            table_[idx]   = {key, val, State::OCCUPIED};
            size_++;
            if (was_tomb) tombstones_--;
        }
    }

    void insert(K&& key, V&& val) {
        if (needs_rehash()) do_rehash(capacity_ * 2);
        K kcopy = key;
        auto [idx, found] = find_slot(kcopy);
        if (found) {
            table_[idx].value = move(val);
        } else {
            bool was_tomb = (table_[idx].state == State::DELETED);
            table_[idx]   = {move(key), move(val), State::OCCUPIED};
            size_++;
            if (was_tomb) tombstones_--;
        }
    }

    V& operator[](const K& key) {
        if (needs_rehash()) do_rehash(capacity_ * 2);
        auto [idx, found] = find_slot(key);
        if (!found) {
            bool was_tomb = (table_[idx].state == State::DELETED);
            table_[idx]   = {key, V{}, State::OCCUPIED};
            size_++;
            if (was_tomb) tombstones_--;
        }
        return table_[idx].value;
    }

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
            return found ? optional<V>(table_[idx].value) : nullopt;
        } catch (...) { return nullopt; }
    }

    bool contains(const K& key) const {
        return get(key).has_value();
    }

    bool erase(const K& key) {
        auto [idx, found] = find_slot(key);
        if (!found) return false;
        table_[idx].key   = K{};
        table_[idx].value = V{};
        table_[idx].state = State::DELETED;
        size_--;
        tombstones_++;
        return true;
    }

    // ── Utility ───────────────────────────────────────────────────────────────

    size_t size()           const { return size_; }
    bool   empty()          const { return size_ == 0; }
    size_t capacity()       const { return capacity_; }
    float  load_factor()    const { return (float)size_ / capacity_; }

    void clear() {
        table_.assign(capacity_, Slot{});
        size_ = tombstones_ = 0;
    }

    void reserve(size_t n) {
        size_t needed = next_prime((size_t)ceil((float)n / max_load_));
        if (needed > capacity_) do_rehash(needed);
    }

    void print() const {
        for (size_t i = 0; i < capacity_; i++) {
            cout << "[" << i << "] ";
            switch (table_[i].state) {
                case State::EMPTY:    cout << "EMPTY\n";    break;
                case State::DELETED:  cout << "DELETED\n";  break;
                case State::OCCUPIED:
                    cout << table_[i].key << ":" << table_[i].value << "\n"; break;
            }
        }
        cout << "size=" << size_ << " tomb=" << tombstones_
             << " cap=" << capacity_ << " load=" << load_factor() << "\n";
    }

    // ── Iteration ─────────────────────────────────────────────────────────────

    class Iterator {
        const DoubleHashMap* map_;
        size_t idx_;
        void advance() {
            while (idx_ < map_->capacity_ &&
                   map_->table_[idx_].state != State::OCCUPIED) ++idx_;
        }
    public:
        Iterator(const DoubleHashMap* m, size_t i) : map_(m), idx_(i) { advance(); }
        pair<const K&, const V&> operator*() const {
            return {map_->table_[idx_].key, map_->table_[idx_].value};
        }
        Iterator& operator++() { ++idx_; advance(); return *this; }
        bool operator!=(const Iterator& o) const { return idx_ != o.idx_; }
    };

    Iterator begin() const { return {this, 0}; }
    Iterator end()   const { return {this, capacity_}; }
};

// ── Specialised integer version: classic textbook double hashing ─────────────
// Uses the canonical formulas h1(k) = k % m and h2(k) = q - (k % q)
// where m is prime and q is a smaller prime.
class IntDoubleHashMap {
private:
    static const size_t DEFAULT_PRIME = 11;
    static const size_t AUX_PRIME     = 7;   // q < m

    enum class State : uint8_t { EMPTY, OCCUPIED, DELETED };
    struct Slot { int key; int val; State state = State::EMPTY; };

    vector<Slot> table_;
    size_t cap_, size_, tombs_;

    size_t h1(int key) const { return (size_t)abs(key) % cap_; }
    size_t h2(int key) const {
        size_t s = AUX_PRIME - ((size_t)abs(key) % AUX_PRIME);
        return s == 0 ? 1 : s;   // p - (k % p) is always in [1, p]; never 0
    }

public:
    explicit IntDoubleHashMap(size_t cap = DEFAULT_PRIME)
        : table_(cap, Slot{}), cap_(cap), size_(0), tombs_(0) {}

    bool insert(int key, int val) {
        if ((float)(size_ + tombs_ + 1) / cap_ > 0.70f) return false;  // needs resize
        size_t idx = h1(key), d = h2(key), first_tom = cap_;
        for (size_t p = 0; p < cap_; p++, idx = (idx + d) % cap_) {
            if (table_[idx].state == State::EMPTY) {
                size_t ins = (first_tom < cap_) ? first_tom : idx;
                table_[ins] = {key, val, State::OCCUPIED};
                size_++;
                if (first_tom < cap_) tombs_--;
                return true;
            }
            if (table_[idx].state == State::DELETED && first_tom == cap_) first_tom = idx;
            if (table_[idx].state == State::OCCUPIED && table_[idx].key == key) {
                table_[idx].val = val; return true;
            }
        }
        return false;
    }

    int get(int key, int not_found = -1) const {
        size_t idx = h1(key), d = h2(key);
        for (size_t p = 0; p < cap_; p++, idx = (idx + d) % cap_) {
            if (table_[idx].state == State::EMPTY) return not_found;
            if (table_[idx].state == State::OCCUPIED && table_[idx].key == key)
                return table_[idx].val;
        }
        return not_found;
    }

    bool remove(int key) {
        size_t idx = h1(key), d = h2(key);
        for (size_t p = 0; p < cap_; p++, idx = (idx + d) % cap_) {
            if (table_[idx].state == State::EMPTY) return false;
            if (table_[idx].state == State::OCCUPIED && table_[idx].key == key) {
                table_[idx].state = State::DELETED;
                size_--; tombs_++;
                return true;
            }
        }
        return false;
    }
};
```

---

## 10. Core Operations — Visualised

### Insert — Diverging Probe Paths

```
capacity = 11 (prime), h1(k) = k % 11, h2(k) = 7 - (k % 7)

Insert keys: 18, 29, 40 (all have h1 = 7)

h1(18)=18%11=7,  h2(18)=7-(18%7)=7-4=3
h1(29)=29%11=7,  h2(29)=7-(29%7)=7-1=6
h1(40)=40%11=7,  h2(40)=7-(40%7)=7-5=2

Insert 18: probe 7. EMPTY → place.
  [_][_][_][_][_][_][_][18][_][_][_]

Insert 29: h1=7, h2=6.
  Probe (7+0×6)%11 = 7: OCCUPIED (18≠29). Probe (7+1×6)%11 = 13%11 = 2: EMPTY → place.
  [_][_][29][_][_][_][_][18][_][_][_]

Insert 40: h1=7, h2=2.
  Probe (7+0×2)%11 = 7: OCCUPIED (18≠40). Probe (7+1×2)%11 = 9: EMPTY → place.
  [_][_][29][_][_][_][_][18][_][40][_]

Final positions: 18@7, 29@2, 40@9.

With QUADRATIC probing (same h1=7):
  18@7 (probe 0). 29@8 (probe 7(occ),7+1=8, EMPTY). 40@11%11=0 (probe 7,8,7+4=0).
  All three cluster in/near slots 7-8-0. ← SECONDARY CLUSTERING

With DOUBLE HASHING:
  18@7, 29@2, 40@9 ← SCATTERED across table, no clustering. ✓

The three keys with the same h1 ended up in slots {7, 2, 9} — spread evenly.
h2=3,6,2 gave completely independent paths from their shared starting slot 7.
```

### Trace: Lookup With Tombstones

```
Table after inserting {18,29,40} and deleting 18:
  [_][_][29][_][_][_][_][☠][_][40][_]
                           ↑ tombstone at slot 7

Lookup 40 (h1=7, h2=2):
  Probe 0: slot (7+0×2)%11 = 7 → TOMBSTONE. Continue (do NOT stop).
  Probe 1: slot (7+1×2)%11 = 9 → OCCUPIED. key=40=40 → FOUND ✓

Lookup 18 (h1=7, h2=3) — should return "not found":
  Probe 0: slot 7 → TOMBSTONE. Continue.
  Probe 1: slot (7+3)%11 = 10 → EMPTY → STOP. Return "not found". ✓

  Note: 18 was at slot 7, now tombstone. The EMPTY at slot 10 correctly
  terminates the search because 18 WAS at slot 7 — if it existed, it would
  have been found at probe 0 or somewhere before an EMPTY in its sequence.

Insert 51 (h1=51%11=7, h2=7-(51%7)=7-2=5):
  Probe 0: slot 7 → TOMBSTONE. Record first_tomb=7. Continue.
  Probe 1: slot (7+5)%11 = 1 → EMPTY → STOP.
  first_tomb=7 → insert at slot 7 (reclaim tombstone). ✓

  [_][_][29][_][_][_][_][51][_][40][_]
  tombstones_ decremented from 1 to 0.
```

### The Coprimality Effect — Visualised

```
capacity = 10 (not prime), step = 5: gcd(5,10) = 5 → only visits 2 slots!

From slot 3 with step 5:
  probe 0: 3
  probe 1: (3+5)%10 = 8
  probe 2: (3+10)%10 = 3 ← CYCLE! Only slots {3, 8} visited.

From slot 3 with step 3: gcd(3,10) = 1 → visits all 10 slots.
  probe 0: 3
  probe 1: 6
  probe 2: 9
  probe 3: 2
  probe 4: 5
  probe 5: 8
  probe 6: 1
  probe 7: 4
  probe 8: 7
  probe 9: 0 → all 10 visited ✓

If capacity is PRIME (say 11), step 5: gcd(5,11) = 1 → visits all 11.
If capacity is PRIME, ANY step from 1 to 10 gives full coverage.
→ Prime capacity makes the coprimality constraint automatic.
```

---

## 11. Interview Problems

### Problem 1: Implement Dictionary with Double Hashing

**Problem:** Implement a dictionary class with `set(key, value)`, `get(key)` (returns -1 if missing), and `delete(key)` operations. The implementation must use double hashing for collision resolution.

**What interviewers test:** understanding of h2 design, coprimality guarantee, tombstone semantics, and load factor management.

```cpp
class Dictionary {
private:
    static const int M = 1009;   // prime capacity
    static const int Q = 997;    // prime auxiliary (< M) for h2

    enum State : int8_t { EMPTY = 0, OCCUPIED = 1, DELETED = -1 };

    int   keys_  [M];
    int   vals_  [M];
    State states_[M];

    int h1(int key) const { return abs(key) % M; }

    // Classic: Q - (key % Q), always in [1, Q] ⊂ [1, M-1]
    // gcd(h2, M): since both Q and M are prime and Q ≠ M, gcd = 1 always.
    int h2(int key) const { return Q - (abs(key) % Q); }

public:
    Dictionary() {
        fill(states_, states_ + M, EMPTY);
    }

    void set(int key, int value) {
        int h = h1(key), d = h2(key), first_del = -1;
        for (int i = 0; i < M; i++) {
            int idx = (h + (long long)i * d) % M;
            if (states_[idx] == EMPTY) {
                int ins = (first_del != -1) ? first_del : idx;
                keys_[ins]   = key;
                vals_[ins]   = value;
                states_[ins] = OCCUPIED;
                return;
            }
            if (states_[idx] == DELETED && first_del == -1) first_del = idx;
            if (states_[idx] == OCCUPIED && keys_[idx] == key) {
                vals_[idx] = value; return;  // update
            }
        }
        // Table full — in production: rehash. Here: overwrite first tombstone.
        if (first_del != -1) {
            keys_[first_del]   = key;
            vals_[first_del]   = value;
            states_[first_del] = OCCUPIED;
        }
    }

    int get(int key) const {
        int h = h1(key), d = h2(key);
        for (int i = 0; i < M; i++) {
            int idx = (h + (long long)i * d) % M;
            if (states_[idx] == EMPTY)   return -1;    // not found
            if (states_[idx] == OCCUPIED && keys_[idx] == key)
                return vals_[idx];                      // found
            // DELETED: continue
        }
        return -1;
    }

    void deleteKey(int key) {
        int h = h1(key), d = h2(key);
        for (int i = 0; i < M; i++) {
            int idx = (h + (long long)i * d) % M;
            if (states_[idx] == EMPTY) return;          // not found
            if (states_[idx] == OCCUPIED && keys_[idx] == key) {
                states_[idx] = DELETED;                  // tombstone
                return;
            }
        }
    }
};

/*
Key design decisions to discuss:
1. Both M=1009 and Q=997 are prime. Since they are different primes,
   gcd(Q - (k%Q), M) = 1 is guaranteed for any key k.
   (Because Q and M share no common factors, and h2 ≤ Q < M.)

2. h2 = Q - (key % Q) always returns a value in [1, Q].
   Q > 0 and key%Q ∈ [0, Q-1], so h2 ∈ [1, Q]. Never 0. ✓

3. Overflow guard: use (long long)(i * d) before % M to prevent
   integer overflow when M and d are both close to INT_MAX.
*/
```

---

### Problem 2: Cuckoo Hashing — Double Hashing Taken to the Extreme

**Problem:** Implement a hash table using cuckoo hashing, which uses two hash functions and guarantees O(1) worst-case lookup.

**Why this connects to double hashing:** Cuckoo hashing uses two independent hash functions (like double hashing) but instead of probing, it places each key at one of its two candidate slots — evicting the current occupant if necessary, who then re-hashes to its *other* slot.

```cpp
// Cuckoo hashing: worst-case O(1) lookup at cost of complex insert
class CuckooHashSet {
private:
    static const int CAP = 1009;   // prime

    int table1_[CAP], table2_[CAP];   // two tables
    bool occ1_[CAP],  occ2_[CAP];

    int h1(int key) const { return abs(key) % CAP; }
    int h2(int key) const { return abs(key ^ (key >> 16)) % CAP; }

public:
    CuckooHashSet() {
        fill(occ1_, occ1_ + CAP, false);
        fill(occ2_, occ2_ + CAP, false);
    }

    // O(1) worst-case — key is in table1_[h1(key)] OR table2_[h2(key)]
    bool contains(int key) const {
        return (occ1_[h1(key)] && table1_[h1(key)] == key) ||
               (occ2_[h2(key)] && table2_[h2(key)] == key);
    }

    // O(1) amortized insert — may evict chain of keys
    bool insert(int key, int max_kicks = 2 * CAP) {
        if (contains(key)) return true;

        int cur = key;
        for (int kick = 0; kick < max_kicks; kick++) {
            // Try to place cur in table1
            int pos1 = h1(cur);
            if (!occ1_[pos1]) { table1_[pos1] = cur; occ1_[pos1] = true; return true; }
            swap(cur, table1_[pos1]);   // evict occupant, keep evicted for next round

            // Try to place cur (the evicted key) in table2
            int pos2 = h2(cur);
            if (!occ2_[pos2]) { table2_[pos2] = cur; occ2_[pos2] = true; return true; }
            swap(cur, table2_[pos2]);   // evict from table2, continue with new evicted
        }
        // Cycle detected after max_kicks — need rehash (rare)
        return false;
    }
};

/*
Cuckoo hashing analysis:
  Lookup: check exactly 2 slots — ALWAYS O(1) worst case.
  Insert: usually O(1), rarely triggers a chain of evictions.
          Expected amortised O(1) for α < 0.5.
  Space: two separate tables (2× memory for same n entries vs single table).
  Load limit: α < 0.5 per table (50% per table = 50% total load).

Connection to double hashing:
  Both use two independent hash functions.
  Double hashing: uses h1 as start, h2 as step → probe sequence.
  Cuckoo hashing: uses h1 and h2 as TWO CANDIDATE SLOTS → kick-out policy.
  Same two-function idea, completely different resolution strategy.
  Cuckoo achieves O(1) worst-case at the cost of insert complexity.
*/
```

---

### Problem 3: Isomorphic Strings — Hash Map with Independent Mappings

**Problem:** Given two strings `s` and `t`, determine if they are isomorphic — there exists a bijection between characters of `s` and `t` such that replacing each character in `s` with its mapped character produces `t`.

**Connection to double hashing:** This problem requires two independent mappings (s→t and t→s). The "double" aspect — maintaining two hash maps that must be mutually consistent — mirrors the principle of using two independent hash functions.

```cpp
// Time: O(n) | Space: O(1) — at most 256 distinct chars
bool isIsomorphic(string s, string t) {
    if (s.size() != t.size()) return false;

    // Two maps: s-char → t-char, and t-char → s-char
    // This is the "double hash" of the problem: both mappings must be consistent.
    int s_to_t[256] = {};   // s character → what it maps to in t
    int t_to_s[256] = {};   // t character → what it maps to in s
    // 0 = unmapped (sentinel; valid chars start at 1 by adding 1 to index)

    for (int i = 0; i < (int)s.size(); i++) {
        int sc = (unsigned char)s[i];
        int tc = (unsigned char)t[i];

        if (s_to_t[sc] == 0 && t_to_s[tc] == 0) {
            // Neither mapped yet — create the bidirectional mapping
            s_to_t[sc] = tc;
            t_to_s[tc] = sc;
        } else if (s_to_t[sc] != tc || t_to_s[tc] != sc) {
            // Conflict: existing mapping is inconsistent
            return false;
        }
    }
    return true;
}

/*
Trace: s="egg", t="add"

i=0: s[0]='e'(101), t[0]='a'(97). Neither mapped. s_to_t[101]=97, t_to_s[97]=101.
i=1: s[1]='g'(103), t[1]='d'(100). Neither mapped. s_to_t[103]=100, t_to_s[100]=103.
i=2: s[2]='g'(103), t[2]='d'(100). s_to_t[103]=100=t[2] ✓, t_to_s[100]=103=s[2] ✓.
Return true ✓.

s="foo", t="bar":
i=0: 'f'↔'b' mapped.
i=1: 'o'↔'a' mapped.
i=2: 'o'→'a' (from s_to_t) but t[2]='r'≠'a'. Return false ✓.

The "double" mapping (s→t AND t→s) prevents:
  - Two s-chars mapping to same t-char: caught by t_to_s check.
  - Same s-char mapping to different t-chars: caught by s_to_t check.
  Bijection guaranteed by maintaining BOTH independent mappings.
*/
```

---

## 12. Real-World Uses

| Domain | Use | Double Hashing Detail |
|---|---|---|
| **UNIX file systems** | Cuckoo hashing in NFS name cache | Two hash functions for O(1) worst-case file lookup |
| **GPU hash tables** | CUDA parallel hash tables | Double hashing avoids thread divergence from sequential probing |
| **Network switches** | TCAM alternative flow tables | Two hash functions for O(1) flow classification |
| **Bloom filters** | `k` hash functions derived from 2 | Kirsch-Mitzenmacher: simulate k functions using `h1 + i*h2` |
| **Consistent hashing** | Distributed systems | Two hash rings for redundancy and load balancing |
| **High-load caches** | In-memory caches (Redis, Memcached alternatives) | When α > 0.7, double hashing provides better expected probe counts |
| **Compiler internals** | LLVM's `DenseMap` (conceptual) | Independent mixing of key bits for primary and secondary slots |
| **Database hash joins** | External memory hash join | Two passes with independent hash functions |
| **Cryptography** | Derive multiple keys from one password | `h1(password)` for encryption key, `h2(password)` for MAC key |

### Bloom Filters — The Most Important Application of Double Hashing

A Bloom filter uses `k` independent hash functions to map each inserted element to `k` bits in a bit array. The query returns "possibly present" if all `k` bits are set. The false positive rate is controlled by choosing `k` and the bit array size `m`.

**The problem:** generating `k` truly independent hash functions is expensive (requires `k` distinct hash computations).

**The Kirsch-Mitzenmacher optimisation (2006):** simulate `k` hash functions using only 2:

```cpp
// Instead of k independent hash functions:
// h_i(key) = h1(key) + i × h2(key)   for i = 0, 1, ..., k-1

// This is EXACTLY double hashing!
// The i-th Bloom filter hash is the i-th probe in a double hash sequence.

class BloomFilter {
    vector<bool> bits_;
    size_t       m_;   // number of bits
    size_t       k_;   // number of hash functions
    hash<string> h1_;

    // Second hash: a different mixing of the same key
    size_t second_hash(const string& key) const {
        size_t h = 0;
        for (char c : key) h = h * 131 + c;
        return h;
    }

public:
    BloomFilter(size_t m, size_t k) : bits_(m, false), m_(m), k_(k) {}

    void insert(const string& key) {
        size_t a = h1_(key), b = second_hash(key);
        for (size_t i = 0; i < k_; i++) {
            // i-th probe of double hash sequence, mod m
            bits_[(a + i * b) % m_] = true;
        }
    }

    bool possibly_contains(const string& key) const {
        size_t a = h1_(key), b = second_hash(key);
        for (size_t i = 0; i < k_; i++) {
            if (!bits_[(a + i * b) % m_]) return false;
        }
        return true;
    }
};

/*
Kirsch-Mitzenmacher showed that using h1(k) + i*h2(k) as the i-th hash
function achieves the SAME asymptotic false positive rate as k truly
independent hash functions.

Cost reduction: k hash evaluations → 2 hash evaluations + k multiplications.
For k=8 (typical Bloom filter): 8 expensive hash calls → 2 calls + 8 multiplies.
This is the single biggest practical application of double hashing.

Redis, Cassandra, HBase, BigTable: all use this Bloom filter construction.
Every time you use a Bloom filter in a real system, you are using double hashing.
*/
```

### GPU Hash Tables — Why Cache-Indifference Matters Here

```
GPU architecture difference from CPU:
  CPU: few fast cores, deep cache hierarchy, cache misses are catastrophic.
  GPU: thousands of slow cores, flat memory, warp-divergence is catastrophic.

Linear probing on GPU:
  Thread 0 probes slot 5, 6, 7, 8, ...
  Thread 1 probes slot 3, 4, 5, 6, ...
  When both threads reach slot 5 simultaneously → CONFLICT (need sync).
  Threads in the same warp take different paths → WARP DIVERGENCE → slow.

Double hashing on GPU:
  Thread 0: h1=5, h2=7 → probes 5, 12, 2, 9, ...
  Thread 1: h1=3, h2=11 → probes 3, 14, 8, 2, ...
  Different keys → different probe sequences → different slots → no conflict!
  All threads in a warp execute the same instructions → NO DIVERGENCE. ✓

This is why GPU-accelerated databases (RAPIDS cuDF, cuBLAS hash operations)
use double hashing rather than linear probing, despite linear probing's
cache advantages on CPU.
```

---

## 13. Edge Cases & Pitfalls

### Pitfall 1: h2 Returns Zero

```cpp
// WORST possible bug: step size of 0 means the probe never moves.
// insert("key", val): probes slot h1, finds it occupied, tries (h1+0)%m=h1 again.
// INFINITE LOOP.

// WRONG: no zero guard
size_t step(const K& key) const {
    return h2_(key) % capacity_;  // can return 0 when h2_(key) is a multiple of m!
}

// CORRECT: ensure non-zero
size_t step(const K& key) const {
    size_t s = h2_(key) % capacity_;
    return (s == 0) ? 1 : s;  // fallback to step=1 (linear probing) if zero
}
// Alternative: use h2_(key) % (capacity_ - 1) + 1 to map to [1, m-1] directly
size_t step(const K& key) const {
    return h2_(key) % (capacity_ - 1) + 1;   // range [1, m-1]
}
```

### Pitfall 2: h2 Returns a Value Not Coprime With Capacity

```cpp
// If capacity = 12 (not prime) and h2 often returns 4 (gcd(4,12)=4):
// Probe visits only 12/4 = 3 distinct slots.
// Insertion fails when those 3 slots are occupied.

// SYMPTOMS:
// - Insertion throws "table full" even at low load
// - About 1/4 of keys fail to insert (specifically those where h2%3==0)

// ROOT CAUSE: capacity not prime, h2 returns even values with high probability.

// FIX 1: Use prime capacity (any step 1..m-1 is automatically coprime).
// FIX 2: If power-of-2 capacity, force h2 to be odd:
size_t step(const K& key) const {
    size_t s = h2_(key);
    return (s % 2 == 0) ? s + 1 : s;  // force odd for power-of-2 capacity
}
```

### Pitfall 3: h1 and h2 Are Not Independent

```cpp
// WRONG: h2 is just a transformation of h1 — highly correlated
size_t h2_bad(const K& key) const {
    return h1_(key) * 2 + 1;  // completely determined by h1!
    // If h1(a) = h1(b) → h2(a) = h2(b) → same probe sequence → secondary clustering!
}

// WRONG: h2 uses only part of h1's information
size_t h2_also_bad(const K& key) const {
    size_t h = h1_(key);
    return h >> 1;  // just shifts h1 right — strongly correlated
}

// CORRECT: use a genuinely different mixing function
size_t step(const K& key) const {
    // Use a completely different hash family
    size_t h = 0;
    // For string keys:
    for (const char& c : key)  h = h * 37 + (unsigned char)c;  // prime 37, not 31
    size_t s = h % capacity_;
    return (s == 0) ? 1 : s;
}
```

### Pitfall 4: Integer Overflow in Probe Index Computation

```cpp
// For large capacity values and large step sizes:
// (h + i * step) may overflow before % capacity is applied.

// WRONG: potential overflow
size_t idx = (h + i * step) % capacity_;
// If h, i, step are all size_t (unsigned 64-bit), i*step can overflow.
// Example: i=1e9, step=1e10 → i*step = 1e19 > 2^64 → wraps around!

// CORRECT: use intermediate modulo
size_t idx = (h % capacity_ + (i % capacity_) * (step % capacity_)) % capacity_;
// Or: accumulate instead of computing i*step directly
size_t idx = h;
for (size_t probe = 0; probe < limit; probe++) {
    // use idx, then advance:
    idx = (idx + step) % capacity_;  // accumulate step, never store i*step
}
```

### Pitfall 5: Rehash Breaks All h1 and h2 Mappings

```cpp
// When capacity changes during rehash, h1(k) % new_cap ≠ h1(k) % old_cap.
// ALL keys move to new positions. This is expected and correct.
// But: if any external code caches a slot index (not the key itself), it breaks.

// WRONG: caching slot index across a potential insert (which may trigger rehash)
size_t my_slot = find_slot("alice").first;
map.insert("bob", 99);   // may trigger rehash → my_slot is now invalid!
auto& val = table_[my_slot];  // UB if table_ was reallocated

// CORRECT: never cache slot indices; always use the key for re-lookup.
map.insert("bob", 99);
auto& val = map.at("alice");  // fresh lookup after any modification
```

### Pitfall 6: Same h2 for All Keys (Constant Step)

```cpp
// If h2 returns the same value for all keys, double hashing degenerates
// to linear probing with a fixed non-1 step. Eliminates no clustering.

// WRONG: h2 is effectively constant for a uniform distribution of keys
size_t step_bad(const K& key) const {
    return 7;  // constant — ALL keys use step 7 → secondary clustering
}

// This creates secondary clustering even worse than standard quadratic:
// All keys with same h1 follow h1, h1+7, h1+14, ... (same sequence).
// Keys with DIFFERENT h1 but h1_a+7 = h1_b also cluster!
```

### Pitfall 7: Long Long Overflow in Classic Formula

```cpp
// The classic h2(k) = Q - (k % Q) with large Q and k can overflow
// when k is large and Q is close to INT_MAX.

// WRONG:
int h2(int key) const { return Q - (key % Q); }
// If key = INT_MIN = -2147483648, then key % Q is implementation-defined in C++!
// (Modulo of negative numbers is implementation-defined before C++11.)

// CORRECT: use abs() and handle zero step
int h2(int key) const {
    int s = (int)(Q - ((long long)abs(key) % Q));
    return (s == 0) ? 1 : s;
}
```

---

## 14. Comparison: The Full Open Addressing Family

| Property | Linear | Quadratic | Double Hashing | Robin Hood | Swiss Table |
|---|---|---|---|---|---|
| Probe sequence | `h, h+1, h+2,...` | `h, h+1, h+4,...` | `h, h+d, h+2d,...` | `h, h+1,...` + Robin Hood | 16-slot SIMD groups |
| Primary clustering | ✗ Severe | ✓ None | ✓ None | ✓ Reduced | ✓ None |
| Secondary clustering | N/A | ✗ Yes | ✓ Nearly none | ✓ Reduced | ✓ None |
| Approaches uniform hashing | ✗ No | ✗ No | ✓ Closely | Partial | ✓ Closely |
| Cache performance | ★★★ Best | ★★ Good | ★ Poor (random jumps) | ★★★ Best | ★★★★ Best (SIMD) |
| Hash computations/probe | 1 | 1 | 2 (once at start) | 1 | 1 |
| Max safe load | 0.70 | 0.50 (prime cap) | 0.80+ | 0.875 | 0.875 |
| Backward-shift delete | ✓ Possible | ✗ Complex | ✗ Infeasible | ✓ Natural | ✓ With metadata |
| Deletion method | Shift or tomb | Tombstone | Tombstone | Backward shift | Metadata byte |
| Capacity constraint | Prime preferred | Prime required | Prime recommended | None | Power of 2 |
| Expected probes (α=0.7) | 2.17 | 1.72 | 1.58 | ~1.45 | ~1.1 |
| Worst-case lookup | O(n) | O(n) | O(n) | O(log n) est. | O(n) theoretically |
| Implementation complexity | Simple | Moderate | Moderate | Complex | Very complex |
| Used in production | Python dict, LLVM | Java IdentityHashMap | Bloom filters, GPU | Rust HashMap | Abseil, SwissDB |

**The open addressing evolution in one line:**

```
Linear(simple,clustering) → Quadratic(less clustering,load limit) →
Double(no clustering,cache miss) → Robin Hood(cache+distribution) →
Swiss Table(SIMD+everything)
```

**When to actually choose double hashing:**

```
Choose double hashing when:
  ✓ Very high load factor required (α > 0.75) and cache misses are acceptable
  ✓ Generating k hash functions from 2 (Bloom filters — this is the main use)
  ✓ GPU/parallel environment where cache coherence ≠ performance
  ✓ Security-sensitive: two independent functions resist some hash flooding attacks
  ✓ Teaching/proving: demonstrates that uniform hashing is achievable cheaply

Do NOT choose double hashing when:
  ✗ CPU cache performance is critical → use linear probing or Robin Hood
  ✗ Keys are small (cache advantage of linear probing dominates)
  ✗ Frequent deletion → tombstones accumulate; no backward-shift alternative
  ✗ Power-of-2 capacity needed → extra complexity to ensure coprimality
  ✗ Simplicity matters → linear probing is just as good for most loads
```

---

## 15. Self-Test Questions

1. **For `h1(k) = k % 11` and `h2(k) = 7 - (k % 7)`, compute the probe sequences for keys `k=3` and `k=14`. Show that despite sharing `h1(3) = h1(14) = 3`, their probe sequences diverge immediately after the first slot.**

2. **Prove that with prime capacity `m`, the probe sequence `h, h+d, h+2d, ...` visits all `m` slots for any step `d ∈ [1, m-1]`. Start from the fundamental theorem about arithmetic progressions mod m.**

3. **Construct a concrete example with `capacity=12` (non-prime) and `h2=4` where insertion fails despite the table being only 25% full. Show exactly which slots are reachable and which are not.**

4. **Why is backward-shift deletion infeasible for double hashing but practical for linear probing? What specific information would you need per slot to make it work, and what would be the memory overhead?**

5. **The Kirsch-Mitzenmacher construction simulates k Bloom filter hash functions as `h1(key) + i * h2(key) mod m` for `i = 0..k-1`. Why does this achieve the same asymptotic false positive rate as k truly independent functions? What property of double hashing enables this?**

6. **Compare the expected unsuccessful probe length at α=0.8 for linear probing, quadratic probing, and double hashing. At what load factor does double hashing's advantage over quadratic probing become significant (say, >50% fewer probes)?**

7. **Design an h2 function for string keys that is (a) always non-zero, (b) coprime with a prime capacity m for any key, and (c) independent from h1. Write the code and justify each property.**

8. **A cuckoo hash table uses two independent hash functions for O(1) worst-case lookup. Explain the connection between cuckoo hashing and double hashing. What is the fundamental difference in how the two functions are used?**

9. **In a GPU parallel hash table, why does linear probing cause warp divergence while double hashing does not? Describe the exact sequence of events that creates divergence in linear probing.**

10. **You are implementing a rate limiter that stores IP addresses (as 32-bit integers) and their request counts. You expect 50,000 concurrent IPs at peak load. Design a double-hashing table for this: choose `m`, `h1`, `h2`, and `max_load`. Justify each choice mathematically.**

---

## Quick Reference Card

```
Double Hashing — O(1) average, eliminates all clustering via key-specific step sizes.

Probe sequence: slot(k, i) = (h1(k) + i × h2(k)) % m   for i = 0, 1, 2, ...

Two hash functions:
  h1(k) = primary hash → initial slot ∈ [0, m)
  h2(k) = step size    ∈ [1, m), must be coprime with m

Coprimality guarantee: gcd(h2(k), m) = 1  →  all m slots reachable
  Prime m: ANY h2 ∈ [1, m-1] is coprime → automatic guarantee
  Power-of-2 m: h2 must be ODD (force with: h2 | 1)
  Classic formula: h2(k) = Q - (k % Q) where Q is prime < m

ALWAYS guard against h2 = 0:
  return (h2_(key) % m == 0) ? 1 : h2_(key) % m;

Clustering eliminated:
  Primary clustering:   ✓ completely (different keys take different paths)
  Secondary clustering: ✓ nearly   (only keys with same (h1,h2) share path)

vs alternatives:
  Linear:    better cache, worse clustering, worse probe counts at high load
  Quadratic: better cache than double, worse distribution, stricter load limit
  Chaining:  no load limit, better unsuccessful lookup, worse cache, +8B/entry

Deletion: TOMBSTONES only (backward shift is infeasible — need key's h2 per slot)
  Mark as DELETED on erase.
  Skip DELETED during lookup (do NOT stop at tombstone).
  Reuse first DELETED during insert (track as first_tomb in probe loop).
  Rehash trigger: (size + tombstones) / capacity > max_load

Max safe load: α ≤ 0.80 (practical), approaches uniform hashing ideal.
Expected probes (α=0.7): ~1.58 successful, ~3.33 unsuccessful.

The Bloom filter connection:
  k hash functions from 2: h_i(key) = (h1(key) + i × h2(key)) % m
  Double hashing IS the standard way to implement Bloom filters efficiently.

Pitfall checklist:
  ✗ h2 returning 0 → infinite loop (guard with: if s==0 return 1)
  ✗ h2 not coprime with m → only m/gcd(h2,m) slots reachable
  ✗ h2 correlated with h1 → secondary clustering returns
  ✗ Non-prime capacity without odd-step enforcement → clustering
  ✗ Integer overflow in (h + i*step) → use accumulation: idx=(idx+step)%m
  ✗ Tombstones in load factor check → count both size AND tombstones

Prime capacity series: 11, 23, 47, 97, 197, 397, 797, 1597, 3203, 6421
```

---

*Previous: [17 — Collision: Quadratic Probing](./17_quadratic_probing.md)*
*Next: [19 — Hash Set](./19_hash_set.md)*
*See also: [Bloom Filter](./probabilistic/bloom_filter.md) | [Cuckoo Hashing](./hash/cuckoo_hashing.md) | [Robin Hood Hashing](./hash/robin_hood.md)*

# Collision Handling: Separate Chaining

> **Curriculum position:** Hash-Based Structures → #2
> **Interview weight:** ★★★☆☆ Medium — understanding chaining is expected in design rounds; implementing it from scratch tests pointer mastery
> **Difficulty:** Intermediate — the concept is simple; the engineering details are rich
> **Prerequisite:** [14 — Hash Map / Hash Table](./14_hash_map.md) | [03 — Singly Linked List](./03_singly_linked_list.md)

---

## Table of Contents

1. [Intuition First — Filing Cabinet with Folders](#1-intuition-first)
2. [Internal Working — Buckets and Chains](#2-internal-working)
3. [The Chain Data Structure — Why Linked Lists?](#3-the-chain-data-structure)
4. [Load Factor and Expected Chain Length](#4-load-factor-and-expected-chain-length)
5. [Time & Space Complexity](#5-time--space-complexity)
6. [Complete C++ Implementation](#6-complete-c-implementation)
7. [Core Operations — Visualised](#7-core-operations--visualised)
8. [Chain Optimisations — Beyond Linked Lists](#8-chain-optimisations)
9. [Interview Problems](#9-interview-problems)
10. [Real-World Uses](#10-real-world-uses)
11. [Edge Cases & Pitfalls](#11-edge-cases--pitfalls)
12. [Chaining vs Open Addressing — The Full Decision](#12-chaining-vs-open-addressing)
13. [Self-Test Questions](#13-self-test-questions)

---

## 1. Intuition First

Imagine a library with 26 filing cabinets, one per letter of the alphabet. When a new book arrives, the librarian computes its first letter and drops it into that cabinet's folder. Multiple books starting with the same letter share the same cabinet — they form a stack inside it. Finding any book means opening the right cabinet and scanning its contents.

This is separate chaining. The "filing cabinets" are **buckets** (an array). The "folders inside" are **chains** (linked lists or other structures). When two keys hash to the same bucket — a **collision** — they coexist peacefully in the same chain, one after the other.

The name "separate" distinguishes this from open addressing: colliding entries are stored **separately** from the main array, in a secondary structure attached to each bucket. No entry ever evicts or displaces another. Growth is handled by extending the chain, not by probing for an empty slot elsewhere.

This design has one deep consequence: **chaining decouples the collision problem from the storage problem**. Collisions become a minor inconvenience — a short scan through a chain — rather than a fundamental structural challenge. The hash table is never "full" in the open addressing sense; you can always add more entries to a chain, even if every bucket is occupied.

The tradeoff: each chain is a separate heap allocation, and following `next` pointers is a cache miss. In a heavily loaded table with long chains, lookup degenerates from O(1) to O(n). Chaining is the right choice when the load factor is unpredictable or when deletions are frequent. Open addressing wins when cache performance is critical and the load factor is controlled.

---

## 2. Internal Working — Buckets and Chains

### The Structure

```
Hash table with capacity = 8 and chaining:

Bucket array (lives in one contiguous block):
┌─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┐
│  0  │  1  │  2  │  3  │  4  │  5  │  6  │  7  │
└──┬──┴──┬──┴──┬──┴──┬──┴──┬──┴──┬──┴──┬──┴──┬──┘
   │     │     │     │     │     │     │     │
  null  null [c:3] [a:1]  null  null [b:2] null
              ↓     ↓           
            [f:6]  [d:4]        
              ↓     ↓           
             null  [e:5]         
                    ↓           
                   null         

Keys and their hash values:
  "a" → h=3, "b" → h=6, "c" → h=2, "d" → h=3, "e" → h=3, "f" → h=2

Bucket 2 chain: c → f → null       (2 entries)
Bucket 3 chain: a → d → e → null   (3 entries)
Bucket 6 chain: b → null           (1 entry)
Others:         null                (0 entries)

load_factor = 6 entries / 8 buckets = 0.75
```

### The Lookup Process

```
Lookup key "d":
  Step 1: idx = h("d") = 3               ← one hash computation
  Step 2: traverse chain at bucket 3:
    node = bucket[3] = [a:1]  →  "a" == "d"? No. next.
    node = [d:4]              →  "d" == "d"? YES. Return 4.
  Total: 1 hash + 2 key comparisons = O(chain_length_at_bucket_3)
```

### The Invariant

For every key `k` stored in the table:
1. `k` is in the chain at `bucket[h(k) % capacity]`
2. `k` appears **exactly once** in the entire table
3. The order within a chain does not affect correctness (but affects constant factors)

---

## 3. The Chain Data Structure — Why Linked Lists?

The classic choice for chains is a singly linked list. Each node stores the key, value, and a pointer to the next node.

### Why Not Arrays per Bucket?

```cpp
// Naive idea: each bucket is a vector<pair<K,V>>
vector<vector<pair<K,V>>> buckets(capacity);

// Problems:
// 1. Each bucket allocation is initially empty (zero capacity).
//    Pushing the first entry triggers a tiny heap allocation — same cost as a node.
// 2. When a bucket vector grows, it reallocates and copies — O(chain_length).
//    Linked list inserts at head: always O(1).
// 3. Bucket vectors consume capacity even when partially filled (wasted space).
// 4. Iterating across all entries means iterating across all bucket vectors —
//    the outer vector is traversed first, which is fine, but inner reallocation
//    invalidates iterators to individual entries.
```

### Why Linked Lists Win for Chaining

```
Linked list node:
┌────────┬────────┬──────────┐
│  key   │ value  │  next*   │
└────────┴────────┴──────────┘

Advantages:
  Insert at head: O(1) — no shifting, no reallocation.
  Delete with predecessor pointer: O(1).
  No wasted capacity — each node is exactly as large as one entry.
  Intrusive lists: node can be part of the entry struct itself (zero extra allocation).

Disadvantage:
  Every node is a separate heap allocation — cache miss on every chain traversal.
  8 bytes of pointer overhead per entry (on 64-bit systems).
```

### Node-per-Entry vs Node-Pool Allocation

```cpp
// Standard: allocate each node individually (the obvious approach)
struct Node {
    K        key;
    V        value;
    Node*    next;
};
Node* n = new Node{key, val, head};  // one malloc per insert

// Optimised: allocate nodes from a fixed-size pool
struct NodePool {
    static const int BLOCK = 1024;
    vector<array<Node, BLOCK>> blocks;
    int pos = BLOCK;

    Node* alloc() {
        if (pos == BLOCK) { blocks.emplace_back(); pos = 0; }
        return &blocks.back()[pos++];
    }
};
// Pool allocation: no heap fragmentation, better cache locality.
// Used in production hash table implementations (e.g., Abseil's dense_hash_map).
```

---

## 4. Load Factor and Expected Chain Length

This is the analytical foundation of chaining. Everything about chaining performance flows from one formula.

### The Simple Model (Uniform Hashing Assumption)

Assume the hash function distributes keys uniformly across all buckets. With `n` entries and `m` buckets:

```
load factor: α = n / m

Expected number of keys in any given bucket: α (by linearity of expectation)

Expected chain length seen by a lookup:
  Successful lookup:   1 + α/2   (average position in chain = middle)
  Unsuccessful lookup: α         (must traverse the full chain)
```

For α = 0.75 (the default for Java's HashMap):
- Successful lookup: 1 + 0.75/2 ≈ 1.375 comparisons on average
- Unsuccessful lookup: 0.75 comparisons on average

For α = 1.0 (the default for C++'s `std::unordered_map`):
- Successful lookup: 1 + 1/2 = 1.5 comparisons on average
- Unsuccessful lookup: 1.0 comparison on average

These numbers are remarkably small — even at load factor 2.0, the average chain length is only 2. This is why chaining is robust to moderate overloading.

### The Probabilistic Worst Case

Even with a perfect hash function, random chance can create long chains:

```
Probability that any specific bucket has ≥ k entries (n items, m buckets):
  ≤ (n/m)^k / k!   ← tail of Poisson distribution

For n=m=1000 (load=1.0):
  P(chain length ≥ 5)  ≈ 0.004   (0.4%)
  P(chain length ≥ 10) ≈ 10^-8   (negligible)

Maximum chain length (with high probability): O(log n / log log n)

This is the expected worst-case — the actual worst case with adversarial
keys is O(n) if the hash function is deterministic and the attacker knows it.
```

### Load Factor Threshold Selection

```
Trade-off:

Low α (e.g., 0.1):  → Very short chains, O(1) lookup
                     → 90% of capacity wasted (memory inefficient)

High α (e.g., 5.0): → Chains of average length 5, O(5) = O(1) lookup (still!)
                     → Compact memory usage
                     → But: rehash overhead if α grows without bound

Typical choices:
  std::unordered_map:  max_load_factor = 1.0 (rehash when n > m)
  Java HashMap:        load_factor     = 0.75
  Python dict:         load_factor     = 0.67 (but uses open addressing, not chaining)
  Custom high-perf:    0.5 to 0.8 depending on access pattern

Rule of thumb: keep α ≤ 1.0 for O(1) expected operations with chaining.
              Higher α is acceptable (chains are still short) but risks
              degenerate worst cases and poor cache behaviour.
```

---

## 5. Time & Space Complexity

### Operation Complexities

| Operation | Average Case | Worst Case | Notes |
|---|---|---|---|
| `insert(k, v)` | **O(1 + α)** ≈ O(1) | O(n) | Worst: all keys in one chain |
| `lookup(k)` | **O(1 + α)** ≈ O(1) | O(n) | Must scan the chain |
| `delete(k)` | **O(1 + α)** ≈ O(1) | O(n) | Find + unlink from chain |
| `contains(k)` | **O(1 + α)** ≈ O(1) | O(n) | Same as lookup |
| `rehash(new_m)` | **O(n + m)** | O(n + m) | n entries + m bucket pointers |
| Iteration | **O(n + m)** | O(n + m) | Must scan all buckets for non-empty |
| Build from n entries | **O(n)** | O(n²) | Average with good hash function |

**Why O(1 + α) and not just O(1)?**  
The `1` accounts for the hash computation (O(1)). The `α` accounts for traversing the chain (expected length = α). At α = 1.0, this is O(2) = O(1). At α = 10, this is O(11) ≈ O(1) technically, but with a large constant — which is why practical implementations keep α bounded.

### Space Complexity

| Component | Space |
|---|---|
| Bucket array | O(m) = O(n/α) — one pointer per bucket |
| Chain nodes | O(n) — one node per stored entry |
| Node overhead | 8 bytes per node (one `next` pointer, 64-bit) |
| Total | **O(n + m)** — typically O(n) for α ≥ 1 |
| vs open addressing | Open addressing: 0 pointer overhead; chaining: 8 bytes/entry |

**Memory comparison for 1,000,000 integer key-value pairs:**

```
Chaining (α=0.75):
  Buckets: 1,333,333 pointers × 8B = 10.7 MB
  Nodes: 1,000,000 × (4+4+8)B = 16 MB  (int key, int val, next ptr)
  Total: ~26.7 MB

Open addressing (α=0.75):
  Array: 1,333,333 × 8B = 10.7 MB  (just key+value, no pointers)
  Total: ~10.7 MB

Conclusion: chaining uses 2.5× more memory for this case.
For large values (e.g., 64-byte structs), the ratio shrinks toward 1.0.
```

---

## 6. Complete C++ Implementation

### Full Hash Table with Separate Chaining

```cpp
#include <iostream>
#include <vector>
#include <forward_list>
#include <functional>
#include <optional>
#include <stdexcept>
#include <string>
using namespace std;

template<typename K, typename V,
         typename Hash  = hash<K>,
         typename Equal = equal_to<K>>
class ChainingHashMap {
private:
    // ── Node ─────────────────────────────────────────────────────────────────
    struct Node {
        K       key;
        V       value;
        Node*   next;
        Node(const K& k, const V& v, Node* n = nullptr)
            : key(k), value(v), next(n) {}
    };

    // ── State ─────────────────────────────────────────────────────────────────
    vector<Node*>   buckets_;      // array of chain heads (nullptr = empty chain)
    size_t          size_;         // total number of stored key-value pairs
    size_t          capacity_;     // number of buckets
    float           max_load_;     // rehash when size/capacity exceeds this
    Hash            hasher_;
    Equal           equal_;

    // ── Private Utilities ─────────────────────────────────────────────────────

    size_t bucket_of(const K& key) const {
        return hasher_(key) % capacity_;
    }

    // Find a node by key in a given bucket's chain.
    // Returns {found_node, predecessor_node} (predecessor is nullptr for head).
    pair<Node*, Node*> find_in_chain(size_t idx, const K& key) const {
        Node* prev = nullptr;
        Node* curr = buckets_[idx];
        while (curr) {
            if (equal_(curr->key, key)) return {curr, prev};
            prev = curr;
            curr = curr->next;
        }
        return {nullptr, prev};   // not found; prev = last node (for append)
    }

    void free_chain(Node* head) {
        while (head) {
            Node* next = head->next;
            delete head;
            head = next;
        }
    }

    void rehash(size_t new_cap) {
        vector<Node*> old_buckets(new_cap, nullptr);
        old_buckets.swap(buckets_);
        capacity_ = new_cap;
        size_ = 0;

        for (Node* head : old_buckets) {
            Node* curr = head;
            while (curr) {
                Node* next = curr->next;
                // Re-insert into new bucket (reuse the node — no new allocation)
                size_t new_idx = bucket_of(curr->key);
                curr->next = buckets_[new_idx];
                buckets_[new_idx] = curr;
                size_++;
                curr = next;
            }
        }
    }

public:
    // ── Constructors & Destructor ─────────────────────────────────────────────

    explicit ChainingHashMap(size_t initial_cap = 16, float max_load = 1.0f)
        : buckets_(initial_cap, nullptr), size_(0),
          capacity_(initial_cap), max_load_(max_load) {}

    ~ChainingHashMap() {
        clear();   // free all nodes before buckets_ is destroyed
    }

    // Disable copy (implement deep copy if needed)
    ChainingHashMap(const ChainingHashMap&)            = delete;
    ChainingHashMap& operator=(const ChainingHashMap&) = delete;

    // Move constructor
    ChainingHashMap(ChainingHashMap&& other) noexcept
        : buckets_(move(other.buckets_)), size_(other.size_),
          capacity_(other.capacity_), max_load_(other.max_load_) {
        other.size_ = other.capacity_ = 0;
    }

    // ── Insert / Update ───────────────────────────────────────────────────────

    // O(1) average — insert key-value pair; update if key already exists
    void insert(const K& key, const V& val) {
        // Check load factor BEFORE inserting (to avoid rehash after the insert)
        if ((float)(size_ + 1) / capacity_ > max_load_) {
            rehash(capacity_ * 2);
        }

        size_t idx = bucket_of(key);
        auto [found, _prev] = find_in_chain(idx, key);

        if (found) {
            found->value = val;   // update existing entry
        } else {
            // Insert at head of chain — O(1), and most recently inserted is found first
            buckets_[idx] = new Node(key, val, buckets_[idx]);
            size_++;
        }
    }

    // ── operator[] — insert with default value if missing ─────────────────────
    V& operator[](const K& key) {
        if ((float)(size_ + 1) / capacity_ > max_load_) {
            rehash(capacity_ * 2);
        }
        size_t idx = bucket_of(key);
        auto [found, _prev] = find_in_chain(idx, key);

        if (found) return found->value;

        // Not found — insert node with default-constructed value
        buckets_[idx] = new Node(key, V{}, buckets_[idx]);
        size_++;
        return buckets_[idx]->value;
    }

    // ── Lookup ────────────────────────────────────────────────────────────────

    // O(1) average — get value by key; throws if not found
    V& at(const K& key) {
        size_t idx = bucket_of(key);
        auto [found, _prev] = find_in_chain(idx, key);
        if (!found) throw out_of_range("key not found");
        return found->value;
    }

    const V& at(const K& key) const {
        size_t idx = bucket_of(key);
        auto [found, _prev] = find_in_chain(idx, key);
        if (!found) throw out_of_range("key not found");
        return found->value;
    }

    // O(1) average — return optional value (no exception)
    optional<V> get(const K& key) const {
        size_t idx = bucket_of(key);
        auto [found, _prev] = find_in_chain(idx, key);
        if (!found) return nullopt;
        return found->value;
    }

    // O(1) average — check existence
    bool contains(const K& key) const {
        size_t idx = bucket_of(key);
        auto [found, _prev] = find_in_chain(idx, key);
        return found != nullptr;
    }

    // ── Deletion ──────────────────────────────────────────────────────────────

    // O(1) average — remove key; returns true if found and removed
    bool erase(const K& key) {
        size_t idx = bucket_of(key);
        auto [found, prev] = find_in_chain(idx, key);
        if (!found) return false;

        if (prev) {
            prev->next = found->next;   // bypass found in chain
        } else {
            buckets_[idx] = found->next;  // found was the head — update head
        }
        delete found;
        size_--;
        return true;
    }

    // ── Utility ───────────────────────────────────────────────────────────────

    size_t size()          const { return size_; }
    bool   empty()         const { return size_ == 0; }
    size_t bucket_count()  const { return capacity_; }
    float  load_factor()   const { return (float)size_ / capacity_; }
    float  max_load_factor() const { return max_load_; }

    void max_load_factor(float f) { max_load_ = f; }

    void clear() {
        for (Node*& head : buckets_) {
            free_chain(head);
            head = nullptr;
        }
        size_ = 0;
    }

    void reserve(size_t n) {
        size_t required_buckets = (size_t)ceil(n / max_load_);
        if (required_buckets > capacity_) rehash(required_buckets);
    }

    // ── Statistics & Diagnostics ──────────────────────────────────────────────

    // Distribution of chain lengths across all buckets
    void stats() const {
        size_t empty_buckets = 0, max_chain = 0, total_chain = 0;
        for (Node* head : buckets_) {
            size_t len = 0;
            for (Node* n = head; n; n = n->next) len++;
            if (len == 0) empty_buckets++;
            max_chain = max(max_chain, len);
            total_chain += len;
        }
        cout << "size=" << size_
             << " capacity=" << capacity_
             << " load=" << load_factor()
             << " empty_buckets=" << empty_buckets
             << " max_chain=" << max_chain
             << " avg_chain=" << (capacity_ > 0 ? (float)total_chain/capacity_ : 0)
             << "\n";
    }

    void print() const {
        for (size_t i = 0; i < capacity_; i++) {
            if (!buckets_[i]) continue;
            cout << "Bucket " << i << ": ";
            for (Node* n = buckets_[i]; n; n = n->next) {
                cout << "[" << n->key << ":" << n->value << "]";
                if (n->next) cout << " → ";
            }
            cout << " → null\n";
        }
    }

    // ── Iteration ─────────────────────────────────────────────────────────────
    // Forward iterator — traverses all non-null chain nodes across all buckets

    class Iterator {
        const ChainingHashMap* table_;
        size_t                 bucket_idx_;
        Node*                  node_;

        void advance_to_next_node() {
            while (!node_ && bucket_idx_ < table_->capacity_) {
                node_ = table_->buckets_[bucket_idx_++];
            }
        }

    public:
        Iterator(const ChainingHashMap* t, size_t idx, Node* n)
            : table_(t), bucket_idx_(idx), node_(n) {
            if (!node_) advance_to_next_node();
        }

        pair<const K&, V&> operator*() const { return {node_->key, node_->value}; }

        Iterator& operator++() {
            node_ = node_->next;
            if (!node_) advance_to_next_node();
            return *this;
        }

        bool operator!=(const Iterator& o) const {
            return node_ != o.node_ || bucket_idx_ != o.bucket_idx_;
        }
    };

    Iterator begin() const { return Iterator(this, 0, nullptr); }
    Iterator end()   const { return Iterator(this, capacity_, nullptr); }
};
```

### Hash Table Using std::forward\_list (Cleaner but Slightly Slower)

```cpp
// Using std::forward_list (singly linked list) as the chain — cleaner interface
#include <forward_list>

template<typename K, typename V>
class ForwardListHashMap {
private:
    using Chain    = forward_list<pair<K,V>>;
    using ChainIt  = typename Chain::iterator;

    vector<Chain> buckets_;
    size_t        size_;
    size_t        capacity_;

    size_t idx(const K& k) const { return hash<K>{}(k) % capacity_; }

public:
    explicit ForwardListHashMap(size_t cap = 16)
        : buckets_(cap), size_(0), capacity_(cap) {}

    void insert(const K& key, const V& val) {
        auto& chain = buckets_[idx(key)];
        for (auto& [k, v] : chain) {
            if (k == key) { v = val; return; }   // update
        }
        chain.emplace_front(key, val);            // insert at head
        size_++;
        if ((float)size_ / capacity_ > 0.75f) {
            // Rehash omitted for brevity
        }
    }

    optional<V> get(const K& key) const {
        const auto& chain = buckets_[idx(key)];
        for (const auto& [k, v] : chain) {
            if (k == key) return v;
        }
        return nullopt;
    }

    bool erase(const K& key) {
        auto& chain = buckets_[idx(key)];
        auto prev = chain.before_begin();
        for (auto it = chain.begin(); it != chain.end(); prev = it, ++it) {
            if (it->first == key) {
                chain.erase_after(prev);
                size_--;
                return true;
            }
        }
        return false;
    }

    bool contains(const K& key) const { return get(key).has_value(); }
    size_t size() const { return size_; }
};
```

---

## 7. Core Operations — Visualised

### Insert — Head Insertion vs Tail Insertion

```
Why insert at the HEAD of the chain (not the tail)?

Tail insertion: must traverse to the end to append. O(chain_length).
Head insertion: update one pointer. O(1).

Both are equivalent for correctness; head insertion is strictly faster.

Example: insert("e", 5) into bucket 3 where chain is [a:1] → [d:4] → null

Head insertion:
  new_node = Node("e", 5, buckets[3])
                                  ↑── new_node.next = old head (a:1)
  buckets[3] = new_node
  Result: [e:5] → [a:1] → [d:4] → null   ← 2 pointer writes, O(1)

Tail insertion:
  traverse: a → d → null (end)
  d.next = new_node
  Result: [a:1] → [d:4] → [e:5] → null   ← O(chain_length) to find tail
```

### Delete — Predecessor Tracking

```
Delete key "d" from bucket 3 chain: [e:5] → [a:1] → [d:4] → null

Step 1: hash("d") = 3. Start traversal.

Step 2: Track prev and curr simultaneously:
  prev=null,    curr=[e:5]  → "e"≠"d". prev=curr, curr=curr.next.
  prev=[e:5],   curr=[a:1]  → "a"≠"d". prev=curr, curr=curr.next.
  prev=[a:1],   curr=[d:4]  → "d"="d"! FOUND.

Step 3: Bypass [d:4]:
  prev->next = curr->next = null

Step 4: delete curr (free [d:4]).

Result: [e:5] → [a:1] → null ✓

Why track prev?
  Singly linked list cannot delete a node given only a pointer to it —
  you must update the PREDECESSOR's next pointer.
  This is why we track prev during traversal.

Alternative (DLL chain): prev pointer already in the node → O(1) delete
given only the node. But DLL nodes cost 16 bytes of pointers vs 8.
```

### Rehash — Redistributing Chains

```
Before rehash: capacity=4, load=1.0
  Bucket 0: [a:1] → null              (h("a")%4=0)
  Bucket 1: [b:2] → [e:5] → null     (h("b")%4=1, h("e")%4=1 — collision!)
  Bucket 2: [c:3] → null              (h("c")%4=2)
  Bucket 3: [d:4] → null              (h("d")%4=3)

Rehash to capacity=8:
  Recompute h(key) % 8 for every entry.
  Nodes are REUSED — we do not allocate new nodes, just update next pointers.

  a: h("a")%8 = 0 → Bucket 0: [a:1]
  b: h("b")%8 = 5 → Bucket 5: [b:2]    (no longer in bucket 1!)
  c: h("c")%8 = 2 → Bucket 2: [c:3]
  d: h("d")%8 = 3 → Bucket 3: [d:4]
  e: h("e")%8 = 1 → Bucket 1: [e:5]    (no longer sharing with b!)

After rehash: capacity=8, load=0.625
  Bucket 0: [a:1]
  Bucket 1: [e:5]   ← e now has its own bucket
  Bucket 2: [c:3]
  Bucket 3: [d:4]
  Bucket 4: null
  Bucket 5: [b:2]   ← b now has its own bucket
  Bucket 6: null
  Bucket 7: null

Chain lengths: all 0 or 1. Lookup time: O(1) with 0 chain traversal on average. ✓
Key insight: rehash does NOT require new node allocation — nodes are relinked.
Cost: O(n) to relink all nodes + O(old_cap + new_cap) to reinitialise bucket arrays.
```

---

## 8. Chain Optimisations — Beyond Linked Lists

Production hash tables often replace the linked list chain with more sophisticated structures to improve worst-case performance and cache behaviour.

### Optimisation 1: Move-to-Front (Heuristic)

After finding a key in the chain, move it to the front. Frequently accessed keys stay at the head, reducing traversal for hot data.

```cpp
V& get_with_mtf(const K& key) {
    size_t idx = bucket_of(key);
    Node* prev = nullptr;
    Node* curr = buckets_[idx];

    while (curr) {
        if (equal_(curr->key, key)) {
            // Move to front
            if (prev) {
                prev->next    = curr->next;
                curr->next    = buckets_[idx];
                buckets_[idx] = curr;
            }
            return curr->value;
        }
        prev = curr;
        curr = curr->next;
    }
    throw out_of_range("key not found");
}
// MTF works well when access distribution is skewed (Zipfian/80-20 rule).
// Cost: extra pointer writes on every hit. Neutral for uniform access.
```

### Optimisation 2: Java 8's TreeMap Fallback

Java's HashMap switches a bucket's chain from a singly linked list to a balanced BST (TreeMap) when the chain length reaches 8. This caps worst-case lookup at O(log n) per bucket instead of O(n).

```
Chain length ≤ 8:  LinkedList  → O(chain_length) per lookup
Chain length > 8:  TreeMap     → O(log chain_length) per lookup

After rehash that reduces chain length ≤ 6: convert back to LinkedList.

Implementation threshold:
  TREEIFY_THRESHOLD   = 8    (list → tree when chain hits this length)
  UNTREEIFY_THRESHOLD = 6    (tree → list when chain drops to this length)
  MIN_TREEIFY_CAPACITY = 64  (only treeify if total capacity ≥ 64)
```

### Optimisation 3: Storing Hash Values in Nodes

Cache the computed hash value in each node to avoid recomputation during rehash.

```cpp
struct Node {
    K      key;
    V      value;
    size_t cached_hash;   // pre-computed hash(key)
    Node*  next;
};

// During rehash: use cached_hash instead of recomputing
size_t new_idx = node->cached_hash % new_capacity;
// Benefit: for expensive-to-hash keys (long strings), avoids O(|key|) work per rehash.
// Cost: 8 extra bytes per node.
```

### Optimisation 4: Inline First Entry in Bucket (Hybrid)

For the common case where a bucket has exactly one entry, store it inline in the bucket array (no heap allocation). Only allocate chain nodes when a second entry arrives.

```cpp
struct Bucket {
    K     key;
    V     value;
    Node* overflow;   // nullptr = only one entry; chain for 2nd+ entries
    bool  occupied;
};

// When occupied=true and overflow=nullptr: one entry, zero heap allocations.
// When overflow!=nullptr: second+ entries are in the linked chain.
// Eliminates heap allocation for the common single-entry bucket case.
// Used in some custom hash table implementations (e.g., in compilers).
```

---

## 9. Interview Problems

### Problem 1: Design a Hash Map from Scratch

**Problem:** Implement `MyHashMap` without using any built-in hash table libraries:
- `MyHashMap()` — initialise with all keys mapped to no value
- `put(key, value)` — insert or update
- `get(key)` — return value or -1 if not found
- `remove(key)` — remove if exists

Keys and values are integers in `[0, 10^6]`. All operations must work correctly.

**This is the direct implementation question.** Interviewers want to see: understanding of buckets, chaining, collision handling, and the full lookup/insert/delete cycle.

```cpp
class MyHashMap {
private:
    static const int CAPACITY = 1009;   // prime number reduces collisions

    struct Node {
        int   key, val;
        Node* next;
        Node(int k, int v, Node* n = nullptr) : key(k), val(v), next(n) {}
    };

    Node* buckets_[CAPACITY];   // array of chain heads

    int bucket_of(int key) const { return key % CAPACITY; }

public:
    MyHashMap() {
        fill(begin(buckets_), end(buckets_), nullptr);
    }

    ~MyHashMap() {
        for (Node* head : buckets_) {
            while (head) {
                Node* next = head->next;
                delete head;
                head = next;
            }
        }
    }

    // O(chain_length) average
    void put(int key, int value) {
        int idx = bucket_of(key);
        // Search for existing key
        for (Node* n = buckets_[idx]; n; n = n->next) {
            if (n->key == key) { n->val = value; return; }
        }
        // Insert at head
        buckets_[idx] = new Node(key, value, buckets_[idx]);
    }

    // O(chain_length) average
    int get(int key) {
        int idx = bucket_of(key);
        for (Node* n = buckets_[idx]; n; n = n->next) {
            if (n->key == key) return n->val;
        }
        return -1;
    }

    // O(chain_length) average
    void remove(int key) {
        int idx = bucket_of(key);
        Node* prev = nullptr;
        Node* curr = buckets_[idx];
        while (curr) {
            if (curr->key == key) {
                if (prev) prev->next = curr->next;
                else       buckets_[idx] = curr->next;
                delete curr;
                return;
            }
            prev = curr;
            curr = curr->next;
        }
    }
};

/*
Design decisions to discuss with interviewer:
1. CAPACITY = 1009 (prime): reduces patterns in h(k) % m; good distribution.
2. Insert at head: O(1) vs O(chain) for tail insert. Always prefer head.
3. Destructor frees all nodes: prevents memory leaks.
4. No rehashing: acceptable for this LeetCode problem since keys ≤ 10^6
   and CAPACITY=1009 keeps max load ≤ 992 (fine for demonstration).
5. Production version would: track size_, rehash when load exceeds threshold,
   use a prime capacity series (11→23→47→...), handle arbitrary key types.
*/
```

**Interviewer follow-ups to prepare for:**
- "Why did you choose this capacity?" → Prime reduces collision patterns in modulo
- "How would you support arbitrary key types?" → Template + std::hash
- "What is the worst-case for get()?" → O(n) if all keys hash to same bucket
- "How do you prevent that worst case?" → Good hash function + rehashing
- "What changes if you need thread safety?" → Lock per bucket (finer than single mutex)

---

### Problem 2: Group Anagrams

**Problem:** Given an array of strings, group strings that are anagrams of each other together. Return the groups in any order.

**Example:** `["eat","tea","tan","ate","nat","bat"]` → `[["bat"],["nat","tan"],["ate","eat","tea"]]`

**Why this is a chaining-insight problem:** The hash map groups multiple values under one key. The "chain" in conceptual terms is the list of strings stored under each canonical key. This directly mirrors how chaining stores multiple entries per bucket.

```cpp
// Time: O(n × m log m) where n = strings, m = max string length
// Space: O(n × m)
vector<vector<string>> groupAnagrams(vector<string>& strs) {
    // Canonical key = sorted string: "eat" → "aet", "tea" → "aet", "ate" → "aet"
    // All anagrams share the same canonical key.
    unordered_map<string, vector<string>> groups;

    for (const string& s : strs) {
        string key = s;
        sort(key.begin(), key.end());
        groups[key].push_back(s);   // groups[key] is the "chain" of anagrams
    }

    vector<vector<string>> result;
    result.reserve(groups.size());
    for (auto& [key, group] : groups) {
        result.push_back(move(group));
    }
    return result;
}

// Alternative: frequency count as key (avoids O(m log m) sort per string)
// Key = array of 26 counts, encoded as a string like "1,0,0,0,1,0,...,1,0,0"
vector<vector<string>> groupAnagramsLinear(vector<string>& strs) {
    unordered_map<string, vector<string>> groups;

    for (const string& s : strs) {
        int freq[26] = {};
        for (char c : s) freq[c - 'a']++;

        // Encode frequency array as a canonical string key
        string key;
        for (int i = 0; i < 26; i++) {
            key += to_string(freq[i]);
            key += ',';   // delimiter prevents ambiguity ("10" ≠ "1","0")
        }
        groups[key].push_back(s);
    }

    vector<vector<string>> result;
    for (auto& [key, group] : groups) result.push_back(move(group));
    return result;
}
// Frequency-count approach: O(n × m) vs O(n × m log m) for sort.
// For short strings, sorting is faster in practice (small constant factor).
// For long strings, frequency count wins.

/*
Trace: ["eat","tea","tan"]

Sort-based:
  "eat" → key="aet" → groups={aet:["eat"]}
  "tea" → key="aet" → groups={aet:["eat","tea"]}
  "tan" → key="ant" → groups={aet:["eat","tea"], ant:["tan"]}

The vector<string> inside groups IS the "chain" — multiple values per key.
The hash map with chaining (conceptually) maps each canonical key to a chain
of all strings that share that canonical form.
*/
```

**Edge cases:**
- Single-character strings: each is its own anagram group of size 1
- Empty string `""`: canonical key is `""` — all empty strings group together
- Duplicate strings (`["ab","ab"]`): both map to same key, appear in same group
- All strings unique anagrams: result has `n` groups of size 1

---

### Problem 3: LRU Cache — HashMap + Doubly Linked List

**Problem:** Design a cache with capacity `cap` supporting O(1) `get(key)` and O(1) `put(key, value)`. When capacity is exceeded, evict the least recently used entry.

**Why this is a chaining-insight problem:** The LRU cache is literally a hash table whose "chain" structure is replaced by a doubly linked list shared across all buckets. Understanding why the DLL and hash map must work together — and what each provides — requires deep understanding of what chaining does and what it does not do.

```cpp
class LRUCache {
private:
    struct Node {
        int   key, val;
        Node* prev;
        Node* next;
        Node(int k, int v) : key(k), val(v), prev(nullptr), next(nullptr) {}
    };

    int cap_;
    unordered_map<int, Node*> map_;   // key → node in DLL
    Node* head_;   // most recently used (MRU) sentinel
    Node* tail_;   // least recently used (LRU) sentinel

    void remove(Node* node) {
        node->prev->next = node->next;
        node->next->prev = node->prev;
    }

    void insert_front(Node* node) {
        node->next = head_->next;
        node->prev = head_;
        head_->next->prev = node;
        head_->next = node;
    }

public:
    explicit LRUCache(int cap) : cap_(cap) {
        head_ = new Node(-1, -1);   // dummy MRU sentinel
        tail_ = new Node(-1, -1);   // dummy LRU sentinel
        head_->next = tail_;
        tail_->prev = head_;
    }

    ~LRUCache() {
        Node* curr = head_;
        while (curr) {
            Node* next = curr->next;
            delete curr;
            curr = next;
        }
    }

    // O(1) — get value; promote to MRU position
    int get(int key) {
        if (!map_.count(key)) return -1;
        Node* node = map_[key];
        remove(node);
        insert_front(node);   // move to front (most recently used)
        return node->val;
    }

    // O(1) — insert or update; evict LRU if over capacity
    void put(int key, int value) {
        if (map_.count(key)) {
            // Update existing: change value and promote to MRU
            map_[key]->val = value;
            remove(map_[key]);
            insert_front(map_[key]);
        } else {
            if ((int)map_.size() == cap_) {
                // Evict LRU: the node just before the tail sentinel
                Node* lru = tail_->prev;
                remove(lru);
                map_.erase(lru->key);
                delete lru;
            }
            Node* node = new Node(key, value);
            map_[key] = node;
            insert_front(node);
        }
    }
};

/*
Why HashMap + DLL? What does each component provide?

HashMap:   O(1) lookup — given a key, find the exact DLL node in O(1).
           Without this: finding a node requires O(n) scan of the DLL.

DLL:       O(1) arbitrary deletion — given a node pointer, remove in O(1).
           O(1) LRU identification — tail.prev is always the LRU.
           O(1) MRU promotion — insert_front is O(1).
           Without this: updating order after each access would be O(n).

Together:
  get(key): map_[key] → node in O(1). Then DLL remove+insert_front in O(1).
  put(key): evict tail.prev in O(1) via DLL. Insert at front in O(1).
            Update map_ in O(1).

Chaining connection: the DLL is, conceptually, a single global chain that
orders all entries by recency. The hash map replaces the per-bucket chain
traversal with a global chain that has a special ordering property (LRU).
Understanding chaining — what a chain is, why it enables O(1) deletion with
a pointer, why it supports multiple values per "key" — is the foundation
that makes the LRU design legible.
*/
```

---

## 10. Real-World Uses

| Domain | Implementation | Chaining Detail |
|---|---|---|
| **GCC `std::unordered_map`** | Singly linked list per bucket | Standard chaining; nodes allocated per entry |
| **Java `HashMap`** | Linked list → Red-Black tree (Java 8+) | Tree fallback when chain ≥ 8; untreeify at ≤ 6 |
| **Node.js V8 engine** | Chained hash map for object properties | Property names hash to bucket; chain holds property descriptors |
| **CPython `dict`** (before 3.6) | Open addressing | Python switched from chaining to open addressing for speed |
| **PostgreSQL hash joins** | Chaining for in-memory hash join | Probe relation keys chain in build-phase hash table |
| **Redis** | Chaining with rehash | Uses two hash tables concurrently during rehash for O(1) amortised |
| **Linux kernel `hlist`** | Doubly-linked chain (DLL chain) | Each bucket holds an `hlist_head`; nodes are `hlist_node` structs |
| **DNS resolver cache** | Chaining hash table | Domain names hash to bucket; multiple TTL entries chain in same bucket |
| **Compiler symbol tables** | Chaining with scope awareness | Each scope level may chain entries for the same identifier |

**Redis's Incremental Rehash — Chaining Done Right:**

Redis maintains two hash tables simultaneously during rehash to avoid blocking for O(n) time:

```
Rehash state: ht[0] = old table (capacity C), ht[1] = new table (capacity 2C)

On every hash table operation:
  1. Perform the requested operation on the correct table.
  2. Also "rehash" one bucket from ht[0] into ht[1]:
     - Move all nodes in ht[0]->buckets[rehash_idx] to ht[1]
     - Set ht[0]->buckets[rehash_idx] = null
     - Increment rehash_idx

After C such steps: ht[0] is completely empty → swap ht[0] and ht[1].

Result: rehash is spread over C operations. No single operation costs O(n).
        During rehash, lookups check ht[0] first, then ht[1].

This is only possible because chaining allows nodes to be relinked between
buckets without moving the data — just update the next pointers.
Open addressing cannot do incremental rehash this cheaply because it must
re-probe to find empty slots in the new table.
```

**Linux Kernel `hlist` — Zero-Overhead Chaining:**

The Linux kernel uses a specialised intrusive doubly-linked list for hash table chaining:

```c
struct hlist_head {
    struct hlist_node* first;   // 8 bytes per bucket — one pointer
};

struct hlist_node {
    struct hlist_node*  next;
    struct hlist_node** pprev;  // pointer to the pointer pointing to this node
};
// pprev enables O(1) deletion without knowing the predecessor:
// *pprev = next   ← unlinks this node without a prev pointer to update separately

// The hash table:
struct hlist_head htable[HTABLE_SIZE];   // array of bucket heads

// Each kernel object that participates in a hash table embeds an hlist_node:
struct task_struct {
    // ... many fields ...
    struct hlist_node pid_link;   // links this process into the PID hash table
};

// Lookup by PID:
int bucket = pid_hashfn(pid);
hlist_for_each_entry(task, &pid_hash[bucket], pid_link) {
    if (task->pid == pid) return task;
}
```

The `pprev` double-indirection is a clever trick: it is a pointer to the `next` field of the predecessor (or the `first` field of the bucket head). This allows O(1) deletion without a traditional `prev` pointer, while keeping bucket heads at 8 bytes (single pointer) instead of 16 bytes (DLL head needs both `next` and `prev`). This saves 8 bytes per bucket — for a hash table with 65,536 buckets (typical for PID tables), that is 512 KB.

---

## 11. Edge Cases & Pitfalls

### Pitfall 1: Forgetting the Predecessor During Deletion

```cpp
// WRONG: only tracking curr, losing predecessor
Node* curr = buckets_[idx];
while (curr) {
    if (curr->key == key) {
        delete curr;    // curr is freed, but nothing updates prev->next or bucket[idx]
        size_--;
        return true;
    }
    curr = curr->next;
}
// Result: dangling pointer in the chain. Accessing bucket[idx] after this = UB.

// CORRECT: track prev pointer throughout traversal
Node* prev = nullptr;
Node* curr = buckets_[idx];
while (curr) {
    if (curr->key == key) {
        if (prev) prev->next       = curr->next;  // bypass curr
        else       buckets_[idx]   = curr->next;  // curr was head — update head
        delete curr;
        size_--;
        return true;
    }
    prev = curr;
    curr = curr->next;
}
```

### Pitfall 2: Memory Leak in Destructor

```cpp
// WRONG: destroying the bucket array without freeing the chains
~ChainingHashMap() {
    // buckets_ vector destructor runs — but it only frees the pointer array,
    // NOT the nodes that the pointers point to!
    // Every Node is a heap allocation that must be explicitly freed.
}
// Result: ALL nodes leak. For a map with 1,000,000 entries = 1M Node objects leaked.

// CORRECT: free each chain before the bucket array is destroyed
~ChainingHashMap() {
    for (Node* head : buckets_) {
        while (head) {
            Node* next = head->next;
            delete head;
            head = next;
        }
    }
}
// Or equivalently: call clear() which does the same thing.
```

### Pitfall 3: Rehash Invalidates Bucket Indices

```cpp
// WRONG: caching a bucket index across a rehash
size_t idx = bucket_of(key);
insert("new_key", value);   // may trigger rehash — capacity changes!
// Now bucket_of(key) % new_capacity ≠ old idx
// Accessing buckets_[idx] after rehash is the OLD bucket at the OLD capacity
// After rehash, that index may not even exist or holds different data.

// CORRECT: always recompute bucket index after any insertion that might rehash
size_t fresh_idx = bucket_of(key);   // recompute after insertion
```

### Pitfall 4: Using Non-Hashable or Non-Comparable Key Types

```cpp
// Attempting to use a struct as a key without defining hash and equality:
struct Coord { int x, y; };
unordered_map<Coord, int> m;   // COMPILE ERROR: no hash<Coord> specialisation

// Must define BOTH:
// 1. hash<Coord> (or custom hash functor)
// 2. operator== for Coord (for key comparison in the chain)

struct CoordHash {
    size_t operator()(const Coord& c) const {
        return hash<long long>{}(((long long)c.x << 32) | (unsigned)c.y);
    }
};
struct CoordEqual {
    bool operator()(const Coord& a, const Coord& b) const {
        return a.x == b.x && a.y == b.y;
    }
};
unordered_map<Coord, int, CoordHash, CoordEqual> m;   // correct
```

### Pitfall 5: Infinite Loop When Key Equality Is Broken

```cpp
// If operator== is not a true equivalence relation (reflexive, symmetric, transitive),
// the chain traversal may loop or fail to find an existing key.

// Example: broken equality (returns false for equal values due to float comparison)
struct BadHash {
    bool operator()(double a, double b) const {
        return a - b < 1e-9;   // NOT symmetric: (1.0, 1.0+1e-10) → true but reversed may not be
    }
};
// Result: the same key may be inserted multiple times if equality fails on lookup.
// Chain grows unboundedly with duplicate "different" keys.
```

### Pitfall 6: Long Chains from Poor Hash Function

```cpp
// Toy hash: h(k) = k % 4 for keys {0, 4, 8, 12, 16, 20}
// All 6 keys map to bucket 0 → chain length 6.

// With capacity=8 and same hash:
// h(0)%8=0, h(4)%8=4, h(8)%8=0, h(12)%8=4, h(16)%8=0, h(20)%8=4
// Buckets 0 and 4 each have chain length 3; others empty.

// Fix 1: use a prime capacity (7 or 11 instead of 8) — reduces pattern-matching.
// Fix 2: use a better hash function (MurmurHash, FNV-1a, xxHash).
// Fix 3: randomise the hash seed.
```

### Pitfall 7: Thread Safety — Chaining Is Not Inherently Thread-Safe

```cpp
// Two threads simultaneously inserting into DIFFERENT buckets:
// Thread 1: insert key with h=3 → writes buckets_[3]
// Thread 2: insert key with h=7 → writes buckets_[7]
// Appears safe (different memory) but size_ is shared!
// Both threads read size_, compute size_+1, and write size_+1 — RACE CONDITION.

// Thread-safe approaches:
// 1. One global mutex: simple but coarse (single writer blocks everyone).
// 2. One mutex per bucket: finer-grained; readers in different buckets proceed concurrently.
// 3. Reader-writer lock per bucket: concurrent reads, exclusive writes.
// 4. Lock-free: atomic operations on bucket heads (complex; used in high-performance systems).

// Per-bucket locking (the standard production approach):
vector<mutex> bucket_locks_(capacity_);
void insert(const K& key, const V& val) {
    size_t idx = bucket_of(key);
    lock_guard<mutex> lock(bucket_locks_[idx]);  // only locks bucket idx
    // ... insert into chain at idx ...
}
```

---

## 12. Chaining vs Open Addressing — The Full Decision

| Criterion | Chaining | Open Addressing |
|---|---|---|
| Load factor flexibility | ✓ Can exceed 1.0 | ✗ Must stay < 1.0 |
| Memory overhead | ✗ 8 bytes per entry (next pointer) | ✓ 0 bytes per entry |
| Cache performance | ✗ Each node = cache miss | ✓ All entries in one block |
| Deletion | ✓ Simple — unlink from chain | ✗ Complex — needs tombstones |
| Worst-case lookup | O(n) — degenerate chain | O(n) — degenerate probe sequence |
| Rehash cost | ✓ Relink nodes — no data copy | ✗ Must re-probe for empty slots |
| Implementation complexity | ✓ Simpler | ✗ More tricky |
| Thread-safe per-bucket locking | ✓ Natural — one mutex per bucket | ✗ Harder — probing crosses buckets |
| Incremental rehash | ✓ Redis uses this | ✗ Very difficult |
| Large values (structs > 64B) | ✓ Overhead ratio drops | ✗ Moving data on rehash is expensive |
| Small integer keys | ✗ Overhead ratio high | ✓ Compact, fast |
| Used in | `std::unordered_map`, Java HashMap, Redis | Python dict, Google Swiss Table, Rust HashMap |

**The decision rule:**

```
Use CHAINING when:
  → Load factor is unpredictable or may exceed 1.0
  → Frequent deletion (no tombstone management needed)
  → Thread safety is required (per-bucket locking is clean)
  → Incremental rehash is needed (Redis, real-time systems)
  → Values are large structs (overhead ratio becomes negligible)
  → You are implementing std::unordered_map equivalent

Use OPEN ADDRESSING when:
  → Cache performance is critical (read-heavy, small values)
  → Memory is constrained (no pointer overhead)
  → Load factor is controlled (≤ 0.7 in practice)
  → Keys are small primitives (int, short)
  → You are implementing a language runtime dict (Python, V8)
```

---

## 13. Self-Test Questions

1. **Draw the bucket array and chains after inserting keys with hashes `[2, 5, 2, 5, 2, 8]` into a capacity-8 chaining hash map. What is the load factor? Which buckets have the longest chains?**

2. **Why is inserting at the HEAD of the chain preferred over the TAIL? Under what access pattern would tail insertion actually be better?**

3. **Trace the deletion of a middle node in a chain of length 4. Write the exact pointer assignments. What happens if you free the node before updating the predecessor's `next` pointer?**

4. **Prove that the expected chain length in a bucket is exactly `α = n/m` under the uniform hashing assumption. Use linearity of expectation.**

5. **In Java 8, a chain converts to a Red-Black tree when it reaches length 8. What specific problem does this solve? Why is length 8 chosen rather than 2 or 50?**

6. **The Linux kernel's `hlist_node` uses `pprev` (pointer to pointer) instead of a regular `prev` pointer. Explain exactly what `pprev` points to and how it enables O(1) deletion without a `prev` pointer while keeping bucket heads at 8 bytes.**

7. **Redis uses two simultaneous hash tables during rehash, migrating one bucket per operation. Why is this strategy only practical with chaining and not open addressing?**

8. **Design a thread-safe hash map using per-bucket locking. What is the minimal set of locks needed for concurrent readers? Does your design avoid deadlock? What operation requires all bucket locks simultaneously?**

9. **You have 10 million entries with average key size 20 bytes and value size 8 bytes. Compare total memory usage for a chaining hash map (load factor 1.0) vs open addressing (load factor 0.75). Which wins?**

10. **Implement a `move_to_front` optimisation for chaining. Under what statistical model of access patterns does this guarantee O(1) amortised lookup? When does it make things worse?**

---

## Quick Reference Card

```
Separate Chaining — each bucket holds a linked list of all keys hashing there.

Structure:
  buckets_[capacity_]: array of Node* (chain heads)
  Each node: {key, value, next*}
  Empty bucket: buckets_[i] == nullptr
  Chain traversal: curr = buckets_[idx]; while(curr) { ...; curr = curr->next; }

Core invariant: key k lives at buckets_[h(k) % capacity_]

All operations O(1 + α) average, O(n) worst case:
  Insert:  check chain for duplicates → insert at HEAD if not found
  Lookup:  traverse chain comparing keys
  Delete:  traverse tracking prev → unlink node → free memory

Load factor α = size / capacity:
  Expected chain length = α
  Keep α ≤ 1.0 for O(1) guarantees
  Rehash to 2× capacity when α > max_load_factor (default 1.0 in C++)

vs Open Addressing:
  Chaining: +flexible load, +simple delete, +per-bucket locking, -cache misses
  Open addr: +cache-friendly, +no ptr overhead, -tombstone complexity, -load limit

Chain insertion — ALWAYS at HEAD:
  new_node->next = buckets_[idx]
  buckets_[idx]  = new_node
  size_++

Chain deletion — ALWAYS track prev:
  if (prev) prev->next = curr->next
  else      buckets_[idx] = curr->next
  delete curr; size_--

Destructor — MUST free every node:
  for (Node* head : buckets_)
      while (head) { Node* next = head->next; delete head; head = next; }

Common pitfalls:
  ✗ Freeing node before updating prev->next (dangling pointer)
  ✗ Skipping chain traversal in destructor (memory leak)
  ✗ Not checking if node was the head before updating bucket pointer
  ✗ Caching bucket index across a potential rehash
  ✗ Missing operator== for key type (silently incorrect lookups)
```

---

*Previous: [14 — Hash Map / Hash Table](./14_hash_map.md)*
*Next: [16 — Collision: Open Addressing](./16_open_addressing.md)*
*See also: [Hash Set](./15_hash_set.md) | [LRU Cache](./specialized/lru_cache.md) | [Linux hlist](https://elixir.bootlin.com/linux/latest/source/include/linux/list.h)*

# Extendible Hashing

> **Curriculum position:** Hash-Based Structures → #7
> **Interview weight:** ★★★☆☆ Medium — a classic database systems question; the directory doubling and bucket splitting mechanics are standard in DB interview loops at companies with storage infrastructure
> **Difficulty:** Advanced — the interplay between global depth, local depth, and the directory is conceptually rich and easy to get wrong
> **Prerequisite:** [14 — Hash Map](./14_hash_map.md) | [15 — Collision: Chaining](./15_collision_chaining.md)

---

## Table of Contents

1. [Intuition First — Growing Without Moving Everything](#1-intuition-first)
2. [Internal Working — Directory, Buckets, and Depths](#2-internal-working)
3. [The Two Depths — Global and Local](#3-the-two-depths)
4. [Directory Doubling — How the Table Grows](#4-directory-doubling)
5. [Bucket Splitting — Redistributing Entries](#5-bucket-splitting)
6. [Shrinking — Directory Halving and Bucket Merging](#6-shrinking)
7. [Time & Space Complexity](#7-time--space-complexity)
8. [Complete C++ Implementation](#8-complete-c-implementation)
9. [Core Operations — Visualised Step by Step](#9-core-operations--visualised)
10. [Interview Problems](#10-interview-problems)
11. [Real-World Uses](#11-real-world-uses)
12. [Edge Cases & Pitfalls](#12-edge-cases--pitfalls)
13. [Extendible vs Linear Hashing vs Static Hashing](#13-comparison)
14. [Self-Test Questions](#14-self-test-questions)

---

## 1. Intuition First

Every static hash table faces the same dilemma: pick a capacity upfront, and either waste memory (too large) or suffer performance degradation (too small requiring a full rebuild). The rebuild problem is the deeper issue — when a static table overflows, you must:
1. Allocate a new, larger array
2. Recompute every key's new hash location
3. Copy all n entries to the new locations
4. Free the old array

This O(n) rehash blocks all operations and makes static hashing unsuitable for systems that cannot afford a pause — databases, file systems, network equipment.

**Extendible hashing grows incrementally instead of all at once.**

The key insight: instead of one flat array of slots, use a **directory** (an array of pointers) pointing to **buckets** (fixed-size overflow pages). When a bucket fills up, split *only that bucket* — not the whole table. The directory may need to double in size (cheap, since it only holds pointers), but the actual data pages only split one at a time.

Think of a library with an index card catalogue. The catalogue has entries like "Ba-Bl → Drawer 1", "Bm-Bz → Drawer 2", etc. When Drawer 1 overflows with books, you:
1. Add a new drawer (split the bucket)
2. Update the catalogue to split "Ba-Bl" into "Ba-Bf → Drawer 1" and "Bg-Bl → Drawer 3"
3. Move only the books from the overflowed drawer

You did not touch Drawers 2, 4, 5, or any other drawer. Ninety percent of the catalogue entries and data remain completely undisturbed. This is extendible hashing.

The result: O(1) amortised insertions with no full-table rehash, O(1) lookups, and graceful growth that is gentle on memory and I/O. This is why virtually every serious database system — PostgreSQL, IBM DB2, Oracle — uses some form of extendible or linear hashing for its hash indexes.

---

## 2. Internal Working — Directory, Buckets, and Depths

### The Three-Level Structure

```
┌─────────────────────────────────────────────────────────────────┐
│                         DIRECTORY                               │
│  Global Depth = 2  (we use the first 2 bits of h(key))         │
│                                                                 │
│  Index  Hash prefix  Pointer                                    │
│  [00]   "00..."   ──────────────────┐                          │
│  [01]   "01..."   ─────────────┐    │                          │
│  [10]   "10..."   ────────┐    │    │                          │
│  [11]   "11..."   ─────┐  │    │    │                          │
└─────────────────────────────────────────────────────────────────┘
                          │  │    │    │
                          ▼  ▼    │    ▼
                     ┌───────┐    │  ┌───────┐
                     │Bucket │    │  │Bucket │
                     │  C    │    │  │  A    │
                     │ LD=2  │    │  │ LD=2  │
                     │[10,15]│    │  │[0, 4] │
                     └───────┘    │  └───────┘
                                  ▼
                             ┌───────┐
                             │Bucket │
                             │  B    │
                             │ LD=1  │ ← local depth 1: both "01" and "11"
                             │[5, 9] │   point to this bucket (shared)
                             └───────┘
```

### Components

**Directory:** An array of `2^GD` pointers (GD = global depth). Entry `i` points to the bucket that holds all keys whose hash starts with the `GD`-bit binary representation of `i`.

**Bucket:** A fixed-size page holding at most `B` key-value pairs. Each bucket has a **local depth (LD)** indicating how many bits of the hash were used to place it.

**Global Depth (GD):** The number of hash bits currently used to index the directory. Determines directory size: `directory_size = 2^GD`.

**Local Depth (LD):** The number of hash bits that uniquely identify a bucket. Always `LD ≤ GD`. If `LD < GD`, multiple directory entries point to the same bucket (that bucket has room for more entries without needing its own directory entries yet).

### How Lookup Works

```
Lookup key k:
  1. Compute h(k).
  2. Extract the top GD bits: prefix = h(k) >> (bit_width - GD).
  3. Follow directory[prefix] to get the bucket pointer.
  4. Linear scan of the bucket (at most B entries).
  
Total: 1 hash + 1 directory access + ≤ B key comparisons.
If B is small (e.g., 4), this is genuinely O(1).
```

---

## 3. The Two Depths — Global and Local

Understanding the relationship between GD and LD is the entire key to understanding extendible hashing.

### Global Depth (GD)

The directory uses the top `GD` bits of each key's hash to route lookups. Doubling the directory increments GD by 1. The directory has exactly `2^GD` entries.

### Local Depth (LD)

Each bucket knows how many bits of the hash were used when it was created. If a bucket has `LD = k`, then all entries in that bucket have the same top `k` bits of their hash value.

### The Key Relationship: How Many Directory Entries Point to a Bucket

If a bucket has local depth `LD` and the directory has global depth `GD`:

```
Number of directory entries pointing to this bucket = 2^(GD - LD)

GD=3, LD=3: 2^0 = 1 entry points to it (bucket has unique identity in directory)
GD=3, LD=2: 2^1 = 2 entries point to it (two directory slots share this bucket)
GD=3, LD=1: 2^2 = 4 entries point to it (four directory slots share this bucket)
GD=3, LD=0: 2^3 = 8 entries point to it (all directory slots share this bucket!)
```

A bucket with `LD < GD` is "shared" by multiple directory entries. It can absorb more insertions before needing to split, because when it does split, the directory already has separate entries for both halves.

### Concrete Example

```
GD = 2 (directory has 4 entries: 00, 01, 10, 11)

Bucket A (LD=2): only directory entry "00" points to it.
  Contains keys whose hash starts with "00".

Bucket B (LD=1): directory entries "01" AND "11" both point to it.
  Contains keys whose hash starts with "0" (the 1 bit that B tracks).
  Wait — that is wrong. Let me clarify:

LD=1 means the bucket was created when only 1 bit was used.
The bit that distinguishes it is the top 1 bit of its keys.
In a directory with GD=2:
  - Entry "01" has top 1 bit = "0"
  - Entry "11" has top 1 bit = "1"
  These are different top 1 bits, so they cannot share a LD=1 bucket.

Correct example of sharing:
  Directory entries "10" and "11" can share a LD=1 bucket.
  Both have top 1 bit = "1". The bucket was created tracking only the "1" prefix.
  LD=1 bucket receives keys from both "10" and "11" directory entries.

Summary:
  LD=2 bucket: exactly 1 directory entry points to it (fully split)
  LD=1 bucket: exactly 2 directory entries point to it (partially shared)
  LD=0 bucket: exactly 4 directory entries point to it (unsplit, initial state)
```

---

## 4. Directory Doubling — How the Table Grows

### When Is Doubling Needed?

Doubling is needed when a **full bucket must split but its local depth already equals the global depth** (LD == GD). In this case, the directory does not have enough entries to distinguish the two halves of the splitting bucket. It must double.

```
Before doubling: GD=1, directory=[entry_0, entry_1]
Bucket A at entry_1 is full AND LD(A)=1=GD.

We need to split Bucket A into two buckets:
  Bucket A_0 for keys with "10" prefix
  Bucket A_1 for keys with "11" prefix
But the directory only has entries "0" and "1" — no "10" or "11"!
We must double the directory first.
```

### The Doubling Procedure

```
Before: GD=1, directory=[ptr_0, ptr_1], size=2

Double:
  new_directory = [ptr_0, ptr_0, ptr_1, ptr_1]  // size 4
  GD = GD + 1 = 2

Key operation: each existing entry is duplicated.
  Old entry 0 ("0" prefix) → new entries 00 and 01 both point to old bucket for "0"
  Old entry 1 ("1" prefix) → new entries 10 and 11 both point to old bucket for "1"

Cost: O(2^GD) — proportional to directory size, NOT to number of stored keys.
  This is the key advantage: the directory is small (just pointers),
  so doubling it is cheap even if many keys are stored.
```

### Directory Doubling Is NOT Full Rehash

```
Static hash table rehash: O(n) — must re-insert every key.
Directory doubling: O(2^GD) — only copies pointer array; keys do NOT move.

Example: n = 1,000,000 keys, GD = 20 (directory size = 2^20 = 1,048,576)
  Static rehash: move 1,000,000 entries.
  Directory double: copy 1,048,576 pointers.

But bucket sizes are small (B = 4-8 pages in a DB), so GD grows slowly:
  After 10 splits, GD ≤ 10 → directory size ≤ 1024 entries.
  A 1024-pointer array is tiny — fits in cache.
  Doubling it costs microseconds, not milliseconds.
```

---

## 5. Bucket Splitting — Redistributing Entries

### When Does a Bucket Split?

A bucket splits when it is **full** and a new key must be inserted into it. This is independent of the directory state — it is purely a function of the bucket's load.

### The Splitting Procedure

```
Split bucket B (which has LD = d, and the triggering key uses directory entry i):

1. Increment B's local depth: LD(B) = d + 1.
2. Create new bucket B' with LD(B') = d + 1.
3. Redistribute keys:
   - Keys where bit (d+1) of hash = 0 stay in B.
   - Keys where bit (d+1) of hash = 1 go to B'.
4. Update directory entries:
   - Half of the entries that previously pointed to B now point to B'.
   - Specifically: entries with index i where bit (d+1) of i = 1.

Case 1: LD(B) < GD before split.
  Directory already has separate entries for both halves. No doubling needed.
  Just update half the entries to point to B'. O(2^(GD-LD)) directory writes.

Case 2: LD(B) = GD before split.
  Directory must double first. Then split. O(2^GD) for doubling.
```

### The Split Operation in Detail

```
GD=2, directory=[A, B, C, B]   (B is pointed to by entries 01 and 11, LD(B)=1)
B is full with keys {k1, k2, k3, k4} where:
  h(k1) = 0101...  top 2 bits = "01"
  h(k2) = 0110...  top 2 bits = "01"  wait...

Let me be precise. B handles keys whose top 1 bit = "1" (since LD=1 means we only
tracked 1 bit). So B holds keys with h(k) starting with "1": both "10..." and "11..."

Split B (LD was 1, now becomes 2):
  B'  keeps keys where bit 2 (the second bit) = 0 → hash starts with "10"
  B'' holds keys where bit 2 (the second bit) = 1 → hash starts with "11"

Update directory:
  Entry "10" (index 2) → now points to B' (keys "10...")
  Entry "11" (index 3) → now points to B'' (keys "11...")
  LD(B') = 2, LD(B'') = 2

Directory after split: [A, B_original, B', B'']
  (A is the bucket for "00...", B_original for "01...", B' for "10...", B'' for "11...")
```

### What If All Keys Go to One Half?

This is a critical edge case — after splitting, all keys might still be in the same bucket (bad hash function or highly clustered keys). In this case:

```
All keys hash to "10..." → B' is full again. B'' is empty.

Re-split B':
  B' becomes two buckets tracking "100..." and "101..."
  But GD=2, and we now need 3 bits → must double directory again.
  GD=3, directory doubles from size 4 to size 8.
  New B'_0 handles "100", B'_1 handles "101".

If keys still cluster (e.g., all have "100..."):
  Split again: LD grows faster than GD.
  In degenerate case (all keys have same hash): infinite splits.
  → This is why a good hash function is critical for extendible hashing.
```

---

## 6. Shrinking — Directory Halving and Bucket Merging

While splits handle growth, merges handle deletion-induced sparseness.

### Bucket Merging

Two buckets can merge when:
1. They are **split-image** buckets (they were created by splitting the same parent)
2. Their combined size fits in a single bucket
3. They have the same local depth

```
Merge condition: Let B1 and B2 be split-image buckets with LD = d.
  If |B1| + |B2| ≤ B (bucket capacity):
    Merge B1 and B2 into B3 with LD = d - 1.
    Update directory: all entries pointing to B1 or B2 now point to B3.
```

### Directory Halving

After enough merges, if every bucket has LD < GD, the directory can be halved:

```
Halving condition: max(LD over all buckets) < GD

If all LD ≤ GD - 1:
  Directory can be halved: new_dir[i] = dir[2*i] for i in 0..2^(GD-1)-1
  GD = GD - 1

Note: halving discards every other entry. This is only valid when
the remaining entries are sufficient (i.e., every pair of entries that
would be merged after halving actually points to the same bucket).
```

### Why Shrinking Is Often Omitted

```
Shrinking is implemented in academic systems but often omitted in production:
  1. Directory is small — even a "too large" directory wastes little memory.
  2. Shrinking complicates the implementation significantly.
  3. Future insertions may require re-growing, wasting the shrink effort.
  4. In database systems, storage is typically not reclaimed immediately.

Many real implementations (including many textbook presentations) only
implement growth (splits and doubling) and leave shrinking as optional.
```

---

## 7. Time & Space Complexity

### Operation Complexities

| Operation | Average | Worst Case | Notes |
|---|---|---|---|
| `lookup(k)` | **O(B)** ≈ O(1) | O(B) | 1 dir access + scan ≤ B entries |
| `insert(k, v)` no split | **O(B)** ≈ O(1) | O(B) | 1 dir access + scan + append |
| `insert(k, v)` with split | O(B + 2^(GD-LD)) | O(n + 2^GD) | Split + dir update |
| `insert(k, v)` amortised | **O(B)** ≈ O(1) | — | Each key causes at most O(1) splits over lifetime |
| `insert` with dir double | O(2^GD) | O(2^GD) | Directory copy; keys don't move |
| `delete(k)` no merge | **O(B)** | O(B) | Find + remove |
| `delete(k)` with merge | O(B) | O(B + 2^GD) | Merge + dir update |
| Build from n entries | **O(n)** avg | O(n²) | n insertions amortised |

**Why O(B) not O(1)?** Each bucket holds up to B entries. Lookup scans all B entries in the bucket. For B = 4 (typical for disk-based DB pages), this is genuinely O(1) with a constant of 4.

### Space Complexity

| Component | Space | Notes |
|---|---|---|
| Directory | O(2^GD) pointers | GD grows as O(log n) in expectation |
| Buckets | O(n) total entries | All entries stored exactly once |
| Per-entry overhead | Near zero in buckets | Just key+value arrays |
| Total | **O(n + 2^GD)** | Typically O(n) since GD = O(log n) |

### The GD Growth Rate

```
With a good hash function, GD grows as log₂ of the number of buckets.
Number of buckets ≈ n/B (where B = bucket capacity).
So: GD ≈ log₂(n/B) = log₂(n) - log₂(B)

For n = 1,000,000 entries with B = 4:
  Expected buckets: 250,000
  Expected GD: log₂(250,000) ≈ 17-18
  Directory size: 2^18 ≈ 262,144 entries × 8 bytes = ~2 MB

For n = 1,000,000,000 entries with B = 4:
  Expected GD ≈ 28
  Directory size: 2^28 ≈ 268 million entries × 8 bytes = ~2 GB
  ← The directory itself becomes large for very large n.

Practical solution: keep directory on disk too (in database systems).
The directory typically fits in memory even for large tables because GD
grows logarithmically.
```

### Comparing to Static Hashing

```
Static hash table rehash: O(n) — moves EVERY entry.
Extendible split:         O(B) — moves at most B entries (one bucket's contents).
Extendible dir double:    O(2^GD) — copies directory pointers (not entries).

For n=1,000,000 and B=4:
  Static rehash: 1,000,000 moves.
  Extendible split: 4 moves.
  Extendible double: 2^17 ≈ 131,072 pointer copies.

Directory doubling is cheaper than full rehash by a factor of ~n/2^GD ≈ n/n = O(1).
(The directory size 2^GD ≈ n/B = n/4, so the double is n/4 pointer copies vs n data moves.)
```

---

## 8. Complete C++ Implementation

```cpp
#include <iostream>
#include <vector>
#include <memory>
#include <optional>
#include <functional>
#include <stdexcept>
#include <cassert>
#include <cmath>
using namespace std;

template<typename K, typename V,
         typename Hash  = hash<K>,
         typename Equal = equal_to<K>>
class ExtendibleHashMap {
private:
    // ── Bucket ────────────────────────────────────────────────────────────────
    struct Bucket {
        int              local_depth;
        int              capacity;      // max entries (B)
        vector<K>        keys;
        vector<V>        vals;

        Bucket(int ld, int cap) : local_depth(ld), capacity(cap) {}

        bool full()  const { return (int)keys.size() >= capacity; }
        bool empty() const { return keys.empty(); }
        int  size()  const { return (int)keys.size(); }

        // Find key in bucket. Returns index or -1.
        int find(const K& key, const Equal& eq) const {
            for (int i = 0; i < (int)keys.size(); i++) {
                if (eq(keys[i], key)) return i;
            }
            return -1;
        }

        void insert(const K& k, const V& v) {
            keys.push_back(k);
            vals.push_back(v);
        }

        void update(int idx, const V& v) {
            vals[idx] = v;
        }

        void remove(int idx) {
            keys.erase(keys.begin() + idx);
            vals.erase(vals.begin() + idx);
        }
    };

    // ── State ─────────────────────────────────────────────────────────────────
    int                                global_depth_;  // GD: bits used to index dir
    int                                bucket_cap_;    // B: max entries per bucket
    vector<shared_ptr<Bucket>>         directory_;     // 2^GD pointers
    size_t                             size_;           // total number of entries
    Hash                               hasher_;
    Equal                              equal_;

    // ── Hash prefix computation ───────────────────────────────────────────────
    // Extract the top `depth` bits of the hash value as an index.
    size_t hash_prefix(const K& key, int depth) const {
        if (depth == 0) return 0;
        size_t h = hasher_(key);
        // Use the LOWEST `depth` bits (more portable than using highest bits)
        return h & ((1ULL << depth) - 1);
    }

    // Index into directory for key given current global_depth
    size_t dir_index(const K& key) const {
        return hash_prefix(key, global_depth_);
    }

    // ── Directory doubling ────────────────────────────────────────────────────
    // Doubles the directory size, duplicating each entry.
    void double_directory() {
        int old_size = (int)directory_.size();
        directory_.resize(old_size * 2);
        // Copy: new entry [i + old_size] = old entry [i]
        for (int i = 0; i < old_size; i++) {
            directory_[i + old_size] = directory_[i];
        }
        global_depth_++;
    }

    // ── Bucket splitting ──────────────────────────────────────────────────────
    // Split the bucket at directory index `dir_idx`.
    // Returns true if split resolved the overflow (may recurse if skewed).
    void split_bucket(int dir_idx) {
        auto bucket = directory_[dir_idx];

        // If bucket's local depth equals global depth, double directory first
        while (bucket->local_depth == global_depth_) {
            double_directory();
        }

        int ld    = bucket->local_depth;
        int new_ld = ld + 1;

        // Create two new buckets
        auto b0 = make_shared<Bucket>(new_ld, bucket_cap_);  // entries with bit = 0
        auto b1 = make_shared<Bucket>(new_ld, bucket_cap_);  // entries with bit = 1

        // Redistribute entries based on bit at position `ld`
        for (int i = 0; i < bucket->size(); i++) {
            size_t prefix = hash_prefix(bucket->keys[i], new_ld);
            int discriminating_bit = (prefix >> ld) & 1;  // the ld-th bit
            if (discriminating_bit == 0) b0->insert(bucket->keys[i], bucket->vals[i]);
            else                          b1->insert(bucket->keys[i], bucket->vals[i]);
        }

        // Update directory entries to point to appropriate new bucket
        // All entries pointing to `bucket` with (ld-th bit = 0) → b0
        // All entries pointing to `bucket` with (ld-th bit = 1) → b1
        int dir_size = (int)directory_.size();
        for (int i = 0; i < dir_size; i++) {
            if (directory_[i] == bucket) {
                int discriminating_bit = (i >> ld) & 1;
                directory_[i] = (discriminating_bit == 0) ? b0 : b1;
            }
        }
    }

public:
    // ── Construction ──────────────────────────────────────────────────────────

    explicit ExtendibleHashMap(int bucket_capacity = 4)
        : global_depth_(0), bucket_cap_(bucket_capacity), size_(0) {
        // Start with depth 0: one bucket, one directory entry.
        auto initial_bucket = make_shared<Bucket>(0, bucket_cap_);
        directory_.push_back(initial_bucket);
    }

    // ── Lookup — O(B) ─────────────────────────────────────────────────────────

    optional<V> get(const K& key) const {
        size_t idx    = dir_index(key);
        auto&  bucket = directory_[idx];
        int    pos    = bucket->find(key, equal_);
        if (pos == -1) return nullopt;
        return bucket->vals[pos];
    }

    bool contains(const K& key) const {
        return get(key).has_value();
    }

    V& at(const K& key) {
        size_t idx = dir_index(key);
        auto&  b   = directory_[idx];
        int    pos = b->find(key, equal_);
        if (pos == -1) throw out_of_range("key not found");
        return b->vals[pos];
    }

    // ── Insert — O(B) amortised ───────────────────────────────────────────────

    void insert(const K& key, const V& val) {
        // Check for existing key → update in-place
        size_t idx = dir_index(key);
        auto   b   = directory_[idx];
        int    pos = b->find(key, equal_);
        if (pos != -1) {
            b->update(pos, val);
            return;
        }

        // Insert new entry
        if (!b->full()) {
            b->insert(key, val);
            size_++;
            return;
        }

        // Bucket full — split
        split_bucket((int)idx);
        size_++;

        // After splitting, re-navigate (directory/bucket pointers may have changed)
        // Try inserting again (may recurse if split was degenerate)
        insert_after_split(key, val);
    }

private:
    // After a split, re-attempt insertion. Handles degenerate cases where
    // all keys hash to one half after the split (rare with good hash functions).
    void insert_after_split(const K& key, const V& val) {
        for (int attempt = 0; attempt < 64; attempt++) {
            size_t idx = dir_index(key);
            auto   b   = directory_[idx];

            // Already present (shouldn't happen here, but be safe)
            int pos = b->find(key, equal_);
            if (pos != -1) { b->update(pos, val); return; }

            if (!b->full()) {
                b->insert(key, val);
                return;
            }

            // Still full after previous split — split again
            split_bucket((int)idx);
        }
        // Degenerate case: hash function is very poor (all keys same hash)
        throw runtime_error("extendible hash: too many splits, check hash function");
    }

public:
    // ── Delete — O(B) ─────────────────────────────────────────────────────────

    bool erase(const K& key) {
        size_t idx = dir_index(key);
        auto&  b   = directory_[idx];
        int    pos = b->find(key, equal_);
        if (pos == -1) return false;
        b->remove(pos);
        size_--;

        // Optionally: try to merge with split-image bucket
        try_merge(idx);
        return true;
    }

private:
    // Find the split-image of the bucket at directory index i.
    // The split-image is the bucket created when bucket i was split.
    // They differ exactly in the (LD-1)-th bit.
    int split_image_index(int i, int ld) const {
        if (ld == 0) return -1;  // no split image for root bucket
        // Toggle the (ld-1)-th bit of i
        return i ^ (1 << (ld - 1));
    }

    void try_merge(size_t idx) {
        auto b = directory_[idx];
        if (b->local_depth == 0) return;  // cannot merge root bucket

        int sibling_idx = split_image_index((int)idx, b->local_depth);
        if (sibling_idx < 0 || sibling_idx >= (int)directory_.size()) return;

        auto sibling = directory_[sibling_idx];

        // Can only merge if same local depth and combined size fits in one bucket
        if (sibling->local_depth != b->local_depth) return;
        if (b->size() + sibling->size() > bucket_cap_) return;

        // Merge sibling into b
        for (int i = 0; i < sibling->size(); i++) {
            b->insert(sibling->keys[i], sibling->vals[i]);
        }
        b->local_depth--;

        // Update all directory entries pointing to sibling → point to b
        for (auto& entry : directory_) {
            if (entry == sibling) entry = b;
        }

        // Try directory halving if all local depths < global depth
        try_halve_directory();
    }

    void try_halve_directory() {
        if (global_depth_ == 0) return;

        // Check if all buckets have local_depth < global_depth
        for (auto& entry : directory_) {
            if (entry->local_depth >= global_depth_) return;
        }

        // Halve: new_dir[i] = dir[i] for i in 0..2^(GD-1)-1
        int half = (int)directory_.size() / 2;
        directory_.resize(half);
        global_depth_--;
    }

public:
    // ── Utility ───────────────────────────────────────────────────────────────

    size_t size()          const { return size_; }
    bool   empty()         const { return size_ == 0; }
    int    global_depth()  const { return global_depth_; }
    size_t dir_size()      const { return directory_.size(); }

    int num_buckets() const {
        // Count unique bucket pointers
        vector<const Bucket*> seen;
        for (auto& p : directory_) {
            if (find(seen.begin(), seen.end(), p.get()) == seen.end())
                seen.push_back(p.get());
        }
        return (int)seen.size();
    }

    void print() const {
        cout << "ExtendibleHashMap: GD=" << global_depth_
             << " dir_size=" << directory_.size()
             << " num_buckets=" << num_buckets()
             << " entries=" << size_ << "\n";

        // Print each unique bucket
        vector<Bucket*> printed;
        for (size_t i = 0; i < directory_.size(); i++) {
            Bucket* b = directory_[i].get();
            if (find(printed.begin(), printed.end(), b) != printed.end()) continue;
            printed.push_back(b);

            // Find which directory entries point to this bucket
            cout << "  Bucket(LD=" << b->local_depth << ", entries=[";
            for (int j = 0; j < b->size(); j++) {
                cout << b->keys[j];
                if (j < b->size()-1) cout << ",";
            }
            cout << "]) ← dir entries {";
            for (size_t di = 0; di < directory_.size(); di++) {
                if (directory_[di].get() == b) cout << di << " ";
            }
            cout << "}\n";
        }
    }

    V& operator[](const K& key) {
        if (!contains(key)) insert(key, V{});
        return at(key);
    }
};
```

---

## 9. Core Operations — Visualised Step by Step

### Building From Scratch: Insert Sequence

```
Bucket capacity B=2. Start: GD=0, one bucket (LD=0) for all keys.
Hash function: h(k) = k (for simplicity, keys are integers).
We use the lowest GD bits as the directory index.

Insert 5: dir[0]=Bucket_A. A not full (0/2). Insert.
  Bucket_A (LD=0): [5]
  Directory: [A]    GD=0

Insert 2: dir[0]=A. A not full (1/2). Insert.
  Bucket_A (LD=0): [5, 2]
  Directory: [A]    GD=0

Insert 9: dir[0]=A. A is FULL (2/2). SPLIT.
  LD(A)=0 = GD=0 → double directory first.
  GD=1. Directory: [A, A]   (both entries still point to A)

  Now split A (LD was 0, new LD=1):
    Discriminating bit = bit 0 of hash.
    h(5)=5=101₂ → bit 0 = 1 → goes to Bucket_B (bit=1 group)
    h(2)=2=010₂ → bit 0 = 0 → goes to Bucket_C (bit=0 group)

    Bucket_B (LD=1): [5]    ← keys with h(k) bit 0 = 1
    Bucket_C (LD=1): [2]    ← keys with h(k) bit 0 = 0
    Directory: [C, B]    (dir[0]→C for prefix "0", dir[1]→B for prefix "1")
    GD=1

  Now insert 9: dir[h(9)&1 = dir[1]] = B. B not full. Insert.
  Bucket_B (LD=1): [5, 9]
  Directory: [C, B]    GD=1

Insert 3: dir[h(3)&1=1]=B. B is FULL (2/2). SPLIT.
  LD(B)=1 = GD=1 → double directory first.
  GD=2. Directory: [C, B, C, B]  (old entries duplicated)

  Split B (LD was 1, new LD=2):
    Discriminating bit = bit 1 of hash.
    h(5)=5=101₂ → bit 1 = 0 → goes to Bucket_D
    h(9)=9=1001₂→ bit 1 = 0 → goes to Bucket_D

  Wait, both 5 and 9 have bit 1 = 0!
  Bucket_D (LD=2): [5, 9]  ← h(k) bits 1-0 = "01"
  Bucket_E (LD=2): []      ← h(k) bits 1-0 = "11"  (empty!)

  Directory entries pointing to old B (indices 1 and 3):
    dir[1] (="01"): bit 1 of 1 = 0 → point to D
    dir[3] (="11"): bit 1 of 3 = 1 → point to E

  Directory: [C, D, C, E]  GD=2

  Now insert 3: dir[h(3)&3=3]=E. E is empty. Insert.
  Bucket_E (LD=2): [3]
  Directory: [C, D, C, E]   GD=2

Final state:
  Bucket_C (LD=1): [2]        ← "x0" prefix (both "00" and "10" point here)
  Bucket_D (LD=2): [5, 9]     ← "01" prefix
  Bucket_E (LD=2): [3]        ← "11" prefix

  dir[0]="00" → C
  dir[1]="01" → D
  dir[2]="10" → C  ← same bucket as dir[0]! (C has LD=1, GD=2)
  dir[3]="11" → E

Lookup 2: dir[h(2)&3=2]→C. Scan C: found 2. ✓
Lookup 5: dir[h(5)&3=1]→D. Scan D: found 5. ✓
Lookup 9: dir[h(9)&3=1]→D. Scan D: found 9. ✓
Lookup 3: dir[h(3)&3=3]→E. Scan E: found 3. ✓
```

### Directory State Visualised

```
After all 4 insertions:

GD = 2  →  directory has 4 entries

       Hash prefix    Dir entry   Bucket    LD    Contents
         "00"          [0]    →   C         1     [2]
         "01"          [1]    →   D         2     [5, 9]
         "10"          [2]    →   C         1     [2]      ← shared! same bucket as [0]
         "11"          [3]    →   E         2     [3]

Note: C has LD=1 < GD=2, so 2 directory entries point to it.
      D and E have LD=2 = GD=2, so exactly 1 directory entry each.
```

---

## 10. Interview Problems

### Problem 1: Implement Extendible Hash Map

**Problem:** Implement an extendible hash map with bucket capacity 3. Show insert and lookup working correctly through a sequence of insertions that requires at least one directory doubling and bucket split.

**Target sequence:** Insert 10 integers and demonstrate the directory evolution.

```cpp
int main() {
    ExtendibleHashMap<int,string> m(3);  // bucket capacity = 3

    auto show = [&](int k, const string& v) {
        m.insert(k, v);
        cout << "After insert(" << k << "):\n";
        m.print(); cout << "\n";
    };

    // These will trigger splits and doublings
    show(1,  "one");    // dir=[A], GD=0
    show(5,  "five");   // dir=[A], GD=0
    show(3,  "three");  // dir=[A], GD=0; A full → split!
    show(7,  "seven");  // after split, dir=[C,B], GD=1
    show(9,  "nine");   // ...
    show(2,  "two");
    show(11, "eleven");
    show(4,  "four");

    // Verify all lookups
    for (int k : {1,2,3,4,5,7,9,11}) {
        auto v = m.get(k);
        cout << "get(" << k << ") = "
             << (v ? *v : "NOT FOUND") << "\n";
    }

    cout << "Final: GD=" << m.global_depth()
         << " dir_size=" << m.dir_size()
         << " unique_buckets=" << m.num_buckets()
         << " entries=" << m.size() << "\n";
    return 0;
}

/*
Expected progression:
  After insert(1): GD=0, 1 bucket, 1 entry
  After insert(5): GD=0, 1 bucket, 2 entries
  After insert(3): GD=0, 1 bucket, 3 entries [FULL]
  After insert(7): GD=1, 2 buckets (split triggered), 4 entries
  ...

Key invariants to verify after each insertion:
  1. Every inserted key is findable via get()
  2. Sum of all bucket sizes = m.size()
  3. For every directory entry i: all keys in dir[i]'s bucket have
     (h(key) & ((1<<GD)-1)) beginning with the same LD bits as i
*/
```

---

### Problem 2: Database Buffer Pool Page Management

**Problem (System Design):** A database buffer pool manager must map 64-bit page IDs to in-memory frames. The mapping must support O(1) average lookup and grow dynamically as more pages are pinned. Describe how extendible hashing would implement this, and what happens during a bucket split.

```cpp
// Database buffer pool hash table using extendible hashing.
// Each "bucket" is a fixed-size array that fits in one OS page.

struct Frame {
    int    frame_id;
    bool   is_dirty;
    int    pin_count;
};

class BufferPoolHashTable {
private:
    // In a real DB: buckets live on disk pages.
    // Here: simulated with vectors.
    ExtendibleHashMap<uint64_t, Frame*> table_;

public:
    explicit BufferPoolHashTable(int bucket_slots = 8)
        : table_(bucket_slots) {}

    Frame* lookup(uint64_t page_id) {
        auto opt = table_.get(page_id);
        return opt ? *opt : nullptr;
    }

    void pin(uint64_t page_id, Frame* frame) {
        table_.insert(page_id, frame);
    }

    bool unpin(uint64_t page_id) {
        return table_.erase(page_id);
    }
};

/*
Key discussion points for the interview:

1. WHY extendible hashing for buffer pool?
   - Pages are pinned/unpinned frequently → dynamic size needed
   - O(1) lookup required for every page access (hot path)
   - Cannot afford O(n) rehash when pinning new pages
   - Bucket size = disk page size → natural fit for I/O alignment

2. What happens during a bucket split in a DB context?
   - Split is I/O: one disk read (old bucket), two disk writes (new buckets)
   - Directory update: in-memory, fast
   - Other buckets: ZERO I/O → only the split bucket is touched

3. How does extendible hashing compare to the alternative (chaining)?
   - Chaining: each bucket is a chain of overflow pages
   - Extendible: each bucket is exactly one page, then splits
   - Extendible: better I/O locality (one page access per lookup)
   - Chaining: simpler but may require multiple I/Os for long chains

4. Contention: when a popular page_id is in a full bucket:
   - Split happens, key moves or stays (redistribution)
   - New directory entry created
   - This is transparent to the caller — next lookup finds it correctly

PostgreSQL's implementation:
   - Uses linear probing in each page (not extendible hashing at bucket level)
   - But the page table for the buffer pool manager uses a flat hash table
     because the number of frames is fixed.
   - Extendible hashing is used in PostgreSQL's hash INDEX (not buffer manager).
*/
```

---

### Problem 3: Detecting When Directory Doubling Is Needed

**Problem:** Given the current state of an extendible hash table (global depth, each bucket's local depth and fullness), determine: (a) whether the next insertion will require a directory doubling, and (b) after the insertion, what the new directory will look like.

```cpp
// This problem tests deep understanding of the GD vs LD relationship.

struct BucketState {
    int local_depth;
    int current_size;
    int capacity;      // B
    int dir_entries;   // 2^(GD - LD) entries point to this bucket
};

struct TableState {
    int global_depth;
    vector<BucketState> buckets;  // unique buckets (not all dir entries)
};

// Determine if inserting into the bucket with hash prefix `prefix`
// will require a directory doubling.
bool requires_doubling(const TableState& state, int prefix) {
    // Find the bucket for this prefix
    // (In practice: follow the directory pointer. Here: use prefix.)
    for (auto& b : state.buckets) {
        // Bucket handles prefix if top LD bits of prefix match
        int mask = (1 << b.local_depth) - 1;
        int b_prefix = prefix & mask;
        // Check if this bucket would be selected
        // (simplified: assume single bucket per LD-bit prefix for this example)
        if (b.current_size == b.capacity) {
            // This bucket is full. Will it need a directory doubling?
            return b.local_depth == state.global_depth;
        }
    }
    return false;  // bucket not full, no split needed
}

/*
Key insight to demonstrate in interview:

Directory doubling is needed IFF:
  The target bucket is FULL AND local_depth == global_depth.

In other words:
  - If the bucket has LD < GD: the directory already has entries for both
    halves. Split → update half the entries. No doubling.
  - If the bucket has LD = GD: the directory cannot distinguish the two
    halves after the split. Must double first to create those entries.

Example:
  GD=2, Bucket A has LD=1. A is full.
    LD(1) < GD(2). Directory entries "00" and "10" both point to A.
    After split: LD(A0)=2, LD(A1)=2.
    dir["00"] → A0, dir["10"] → A1. No doubling. ✓

  GD=2, Bucket B has LD=2. B is full.
    LD(2) = GD(2). Only one directory entry ("11" say) points to B.
    After split, we need entries "011" and "111" — 3-bit prefixes.
    Must double: GD=3, directory grows 4→8 entries. Then split. Required.
*/
```

---

## 11. Real-World Uses

| Domain | System | Extendible Hashing Detail |
|---|---|---|
| **Database hash indexes** | PostgreSQL `hash` index AM | Dynamic hash index using extendible hashing for `WHERE col = val` queries |
| **Database buffer pool** | IBM DB2 | Buffer pool page table uses extendible hashing for page_id→frame mapping |
| **File systems** | ext4 directory index (HTree) | Extendible hashing for large directory lookups (millions of files per dir) |
| **File systems** | XFS directory B+tree | Hybrid: B+tree with extendible hash leaves |
| **Key-value stores** | Oracle BerkeleyDB | Hash access method uses extendible hashing for dynamic bucket management |
| **In-memory databases** | H-Store / VoltDB | Partition-local hash tables use extendible hashing |
| **Object storage** | HDFS INode table | NameNode uses extendible hashing for inode ID to metadata mapping |
| **Network hardware** | TCAM management | Extendible hash tables for prefix-to-action mapping in programmable NICs |
| **Distributed systems** | Chord DHT | Conceptually: consistent hashing is a distributed analog of extendible hashing |

### PostgreSQL Hash Index — Extendible Hashing in Production

PostgreSQL's hash index implementation (in `src/backend/access/hash/`) directly uses extendible hashing. Here is the high-level architecture:

```
PostgreSQL Hash Index:
  Primary page (page 0): stores the metadata (global_depth, etc.)
  Bucket pages: one or more 8KB disk pages per bucket
  Overflow pages: linked from bucket pages when bucket is full
  Directory: stored in "meta-pages" with 2^GD entries pointing to bucket pages

Insert a row with key k:
  1. Read meta-page to get GD.                          [1 I/O]
  2. Compute h(k), extract GD bits → bucket number.     [0 I/O, just arithmetic]
  3. Read directory page for that bucket number.         [1 I/O]
  4. Read bucket page.                                   [1 I/O]
  5. If bucket page has space: insert. Mark page dirty.  [1 write, eventually]
  6. If full: split.                                     [several I/Os]

For lookups: steps 1-4 only = 3 I/Os.
With caching: meta-page and directory in memory = 1 I/O.

Split:
  - Allocate new bucket page.                            [1 I/O write]
  - Read old bucket, redistribute.                       [1 I/O read, 2 writes]
  - Update directory entry.                              [1 I/O write]
  - If GD doubles: read and rewrite directory pages.    [O(2^GD / page_size) I/Os]
  - Other buckets: ZERO I/O.

This is why extendible hashing is used for database indexes:
  The split cost is proportional to the split bucket + directory,
  NOT to the total number of stored tuples.
  For a table with 10 million rows, a split touches ≤ 2 pages of data.
```

### ext4 HTree Directory Index

The Linux ext4 filesystem uses a B-tree variant (called HTree) for large directories. The leaf nodes of the HTree use an extendible hashing scheme to map filename hashes to directory entries.

```
ext4 directory lookup for filename "alice":
  1. Hash "alice" using htree_hash(): h = 0x4f3a2b1c
  2. Walk HTree from root using the hash value.
  3. At the leaf node: use the hash as an extendible hash key.
  4. Follow directory entry pointer to the actual dir entry.

Without HTree (small directories):
  Linear scan of all directory entries → O(n) for large directories.

With HTree + extendible hashing:
  O(log n) for HTree traversal + O(B) for bucket scan = O(1) effectively.

This is why ext4 can handle directories with millions of files efficiently.
In the Linux kernel source: fs/ext4/htree.c implements this.
```

---

## 12. Edge Cases & Pitfalls

### Pitfall 1: Degenerate Split Loop — All Keys Hash to One Half

```cpp
// WORST CASE: poor hash function where all keys have the same low bits.
// Example: h(k) = k for even keys → bit 0 = 0 for all → all go to same half.

ExtendibleHashMap<int,int> m(4);  // bucket capacity 4
for (int i = 0; i < 100; i += 2) m.insert(i, i);  // all even numbers
// h(0)=0, h(2)=2, h(4)=4, ... — all even, bit 0 = 0 for all.
// First split: dir[0] = all keys (bit 0 = 0). dir[1] = empty.
// Second split: dir[00] = all keys (bit 1:0 = 00). dir[10] = empty.
// This continues: GD grows to log2(n) while only one bucket ever fills.
// Result: GD=50 for 100 keys → directory size = 2^50 → MEMORY EXPLOSION.

// FIX: ensure hash function distributes all bits, not just the lowest.
// Use MurmurHash, xxHash, or splitmix64 instead of identity hash.
// Test: verify that for your key distribution, bits 0..GD-1 are all well-distributed.
```

### Pitfall 2: Split Does Not Update ALL Pointing Entries

```cpp
// When splitting bucket B with LD=d, ALL directory entries pointing to B
// must be updated — not just the one that triggered the insert.

// WRONG: only updating the triggering directory entry
void split_wrong(int trigger_idx) {
    auto b = directory_[trigger_idx];
    // ... create b0, b1 ...
    directory_[trigger_idx] = b1;  // update only this one entry!
    // ALL OTHER entries still pointing to b are now stale!
    // Lookup via those entries goes to b (with wrong LD),
    // and new inserts route there too — corruption.
}

// CORRECT: scan all directory entries, update those pointing to b
for (int i = 0; i < (int)directory_.size(); i++) {
    if (directory_[i] == old_bucket) {
        int bit = (i >> old_ld) & 1;
        directory_[i] = (bit == 0) ? b0 : b1;
    }
}
```

### Pitfall 3: Forgetting to Double Before Splitting When LD == GD

```cpp
// WRONG: splitting without checking if LD == GD first
void split_wrong(int idx) {
    auto bucket = directory_[idx];
    int ld = bucket->local_depth;
    // Directly split without checking LD vs GD:
    // If LD == GD, the directory has no entries for the two halves.
    // The update loop finds nothing to update for the new ld+1 bit.
    // Result: both new buckets exist but are unreachable from the directory!
}

// CORRECT: double first if LD == GD
while (bucket->local_depth == global_depth_) {
    double_directory();   // creates entries for the finer-grained distinction
}
// Now LD < GD, and the directory has entries for both halves.
```

### Pitfall 4: Off-by-One in Bit Extraction

```cpp
// Which bits of the hash are used as the directory index?
// Convention: use the LOWEST `depth` bits.

// WRONG: using highest bits (endian confusion)
size_t dir_index_wrong(const K& key) const {
    size_t h = hasher_(key);
    return h >> (64 - global_depth_);  // highest GD bits
}
// This works for lookups but the split logic must be consistent.
// If you use highest bits for lookup but lowest for split,
// you will route to the wrong bucket after a split.

// CORRECT: use LOWEST bits consistently throughout
size_t hash_prefix(const K& key, int depth) const {
    if (depth == 0) return 0;
    return hasher_(key) & ((1ULL << depth) - 1);
}
// dir_index uses hash_prefix(key, global_depth_).
// split uses hash_prefix(key, new_local_depth) to assign to b0 or b1.
// Both use the same convention: lowest depth bits.
```

### Pitfall 5: Shared Bucket Mutation After Split

```cpp
// After splitting, the old bucket pointer is replaced.
// But if you cached the old bucket pointer elsewhere, it now points
// to a bucket that may contain redistributed (incomplete) entries.

// WRONG: caching bucket pointer before insertion that triggers split
auto bucket_ptr = directory_[dir_index(key)];
m.insert(key, val);  // may split! directory_[...] now points to new bucket
// bucket_ptr still points to the OLD bucket — may have missing entries.
// Using bucket_ptr for subsequent lookups gives wrong results.

// CORRECT: always re-navigate via the directory after any mutation
// Never cache directory entries or bucket pointers across mutations.
```

### Pitfall 6: Global Depth Overflow

```cpp
// For 64-bit hashes, GD can grow at most to 64.
// With degenerate keys (all same low bits), GD can grow to 64 → 2^64 directory entries.
// This will exhaust memory long before that.

// Guard against unreasonable GD growth:
static const int MAX_GLOBAL_DEPTH = 20;  // 2^20 = 1M directory entries max

while (bucket->local_depth == global_depth_) {
    if (global_depth_ >= MAX_GLOBAL_DEPTH) {
        throw runtime_error("extendible hash: GD limit reached, check hash function");
    }
    double_directory();
}
```

### Pitfall 7: Merge Attempts Without Checking Split-Image Condition

```cpp
// WRONG: merging any two adjacent buckets
void try_merge_wrong(int idx) {
    int sibling_idx = idx ^ 1;  // flip last bit — may not be split image!
    // sibling may have a DIFFERENT local depth than the bucket at idx.
    // Merging them would corrupt the local depth invariant.
}

// CORRECT: merge only when both buckets have the SAME local depth
// AND they are true split images (differ exactly in the LD-th bit).
void try_merge(int idx) {
    auto b = directory_[idx];
    int ld = b->local_depth;
    if (ld == 0) return;

    // Split image differs in bit (ld-1)
    int sibling_idx = idx ^ (1 << (ld - 1));

    auto sibling = directory_[sibling_idx];
    if (sibling->local_depth != ld) return;  // different LD → cannot merge
    if (b->size() + sibling->size() > bucket_cap_) return;  // too many entries
    // ... proceed with merge
}
```

---

## 13. Comparison: Extendible vs Linear Hashing vs Static Hashing

| Property | Static Hashing | Extendible Hashing | Linear Hashing |
|---|---|---|---|
| Lookup | O(1) avg | **O(B) = O(1)** avg | O(B) avg |
| Insert (no overflow) | O(1) | O(B) | O(B) |
| Insert (with overflow) | O(n) rehash | O(B) split + O(2^GD) double | O(B) split only |
| Split granularity | Full table | One bucket + dir | One bucket (no dir) |
| Directory structure | None | Yes (2^GD pointers) | None (implicit ordering) |
| Growth pattern | Double all at once | Double dir + split bucket | Sequential, one bucket at a time |
| Shrink support | Yes (rehash) | Yes (merge + halve) | Limited |
| Memory overhead | Low | Medium (directory) | Low |
| Concurrency | Simple (one lock) | Moderate (bucket locks) | Complex (growing sequence) |
| Worst-case split | O(n) | O(2^GD) dir double | O(1) — no doubling! |
| Hash function sensitivity | Moderate | High (bad hash → GD explosion) | Low (hash used modularly) |
| Used in | Simple caches | DB hash indexes, file systems | Persistent storage, some DBs |
| Implementation complexity | Simple | Moderate | Complex |

### When to Choose Extendible Hashing

```
USE EXTENDIBLE HASHING WHEN:
  ✓ Dynamic size is required (unknown final n at creation time)
  ✓ Cannot afford O(n) blocking rehash (real-time, high-throughput systems)
  ✓ Bucket size maps naturally to a physical unit (disk page, cache line)
  ✓ Deletion and bucket merging are needed (database indexes)
  ✓ The key distribution is reasonably uniform (good hash function available)

DO NOT USE WHEN:
  ✗ Key distribution is highly skewed (degenerate splits → GD explosion)
  ✗ n is known and fixed at creation (static hashing is simpler and faster)
  ✗ Memory for directory is a concern (for very large n, 2^GD can be large)
  ✗ Concurrency is critical (directory doublings require broader locking)
  ✗ Simple implementation is required (linear probing or chaining is simpler)

CHOOSE LINEAR HASHING OVER EXTENDIBLE WHEN:
  ✓ You need truly incremental growth (one bucket at a time, no directory)
  ✓ The directory overhead is unacceptable
  ✓ You need predictable (non-bursty) expansion cost
  ✓ Key distribution is unknown and may be skewed (linear hashing is more robust)
```

---

## 14. Self-Test Questions

1. **State the extendible hashing invariant relating global depth (GD), local depth (LD), and the number of directory entries pointing to a bucket. If GD=4 and a bucket has LD=2, how many directory entries point to it?**

2. **Trace a complete directory doubling. Start with GD=1, directory=[B1, B2]. Show the exact state of the directory after doubling, including which entries point to which buckets. What is the new GD?**

3. **Explain the difference between a bucket split that requires directory doubling and one that does not. What is the exact condition that determines which case applies?**

4. **In the worst case, inserting a single key into an extendible hash table causes the GD to increase by 1 and requires O(2^GD) work for the directory doubling. Why is this O(1) amortised over all insertions?**

5. **After splitting a bucket with LD=d and GD=d (requiring a doubling to GD=d+1), how many directory entries are updated? How many point to the original bucket before the split, and how are they redistributed?**

6. **Construct a 6-key example where all keys hash to the same bucket, forcing 3 consecutive splits. Show the directory and bucket state after each split. What does this reveal about the importance of hash function quality?**

7. **Describe the bucket merge and directory halving procedures. What two conditions must be satisfied to merge two buckets? What condition must hold for the entire table before directory halving is possible?**

8. **PostgreSQL uses extendible hashing for its hash indexes. For a table with 1 million rows and bucket size 8, what is the expected GD? What is the expected directory size in bytes? How many I/O operations does a typical equality lookup require?**

9. **Compare extendible hashing and linear hashing on the criterion of "worst-case cost of a single insertion." Which has lower worst-case cost and why?**

10. **A concurrent extendible hash table must handle simultaneous lookups and a directory doubling. What is the minimum locking granularity needed? Can readers proceed without blocking during a doubling? What about during a bucket split?**

---

## Quick Reference Card

```
Extendible Hashing — Dynamic hash table that grows one bucket at a time.

Three components:
  Directory: array of 2^GD pointers to buckets
  Buckets:   fixed-size pages of B key-value pairs
  Depths:    GD (global, directory-wide), LD (local, per bucket)

Invariant: bucket with LD has 2^(GD-LD) directory entries pointing to it.

Lookup (O(B) ≈ O(1)):
  prefix = h(key) & ((1<<GD) - 1)
  bucket = directory[prefix]
  scan bucket for key (at most B comparisons)

Insert (O(B) amortised):
  1. Navigate to bucket via directory.
  2. If not full: insert directly.
  3. If full:
     a. If LD == GD: double directory (O(2^GD) pointer copies).
     b. Split bucket: redistribute B entries based on bit (LD).
        - Keys with bit LD = 0 → new bucket b0 (LD+1)
        - Keys with bit LD = 1 → new bucket b1 (LD+1)
     c. Update ALL directory entries pointing to old bucket.
     d. Retry insertion in the correct new bucket.

Directory doubling:
  new_dir[i]           = old_dir[i]    for i < 2^GD
  new_dir[i + 2^GD]    = old_dir[i]    for i < 2^GD
  GD = GD + 1
  → Only pointer copies, no data movement!

Key insight: splitting touches ONE bucket.
             All other buckets (and their directory entries) are untouched.
             This is the O(1) amortised cost vs O(n) for static rehash.

Delete: find and remove, then optionally merge split-image buckets.
Merge condition: same LD, combined size ≤ B, are true split images.

Critical pitfalls:
  ✗ Poor hash function → degenerate splits → GD explodes → memory exhaustion
  ✗ Splitting without doubling when LD == GD → unreachable new buckets
  ✗ Updating only one directory entry after split → stale pointers
  ✗ Merging buckets with different LD → local depth invariant broken
  ✗ Caching bucket pointers across mutations → stale references

GD growth rate: O(log₂(n/B)) expected with uniform hash function
  n=1M, B=4: expected GD ≈ 18, directory ≈ 256K entries ≈ 2MB

Real-world: PostgreSQL hash indexes, DB2 buffer pool,
            ext4 HTree directory index, BerkeleyDB hash access method.
```

---

*Previous: [19 — Cuckoo Hashing](./19_cuckoo_hashing.md)*
*Next: [21 — Robin Hood Hashing](./21_robin_hood.md)*
*See also: [Linear Hashing](./hash/linear_hashing.md) | [PostgreSQL Hash Index](https://www.postgresql.org/docs/current/hash-index.html) | [ext4 HTree](https://www.kernel.org/doc/html/latest/filesystems/ext4/dynamic.html)*

# Dynamic Array / std::vector

> **Curriculum position:** Linear Structures → #2  
> **Interview weight:** ★★★★★ Critical — the default container for every interview problem  
> **Difficulty:** Beginner concept, deep internals worth mastering  
> **Prerequisite:** [01 — Static Array](./01_static_array.md)

---

## Table of Contents

1. [Intuition First — What Problem Does It Solve?](#1-intuition-first)
2. [Internal Working — The Grow-and-Copy Engine](#2-internal-working)
3. [Amortized Analysis — Why push_back is O(1)](#3-amortized-analysis)
4. [Memory Layout & Iterator Invalidation](#4-memory-layout--iterator-invalidation)
5. [Time & Space Complexity](#5-time--space-complexity)
6. [std::vector API — Complete Reference](#6-stdvector-api--complete-reference)
7. [Common Patterns & Techniques](#7-common-patterns--techniques)
8. [Interview Problems](#8-interview-problems)
9. [Real-World Uses](#9-real-world-uses)
10. [Edge Cases & Pitfalls](#10-edge-cases--pitfalls)
11. [Implementing a Dynamic Array from Scratch](#11-implementing-a-dynamic-array-from-scratch)
12. [Comparison: vector vs array vs list vs deque](#12-comparison)
13. [Self-Test Questions](#13-self-test-questions)

---

## 1. Intuition First

Static arrays force you to know the size upfront. That is fine for a fixed-size lookup table, but most real problems involve data whose size you only learn at runtime — reading lines from a file, collecting results from a BFS, building a list of matching records.

The dynamic array solves this with one elegant idea: **start small, and double when full.**

Think of it like moving into a new city. You rent a small studio (capacity = 4). Life accumulates — books, furniture, gadgets. When the studio is full, you don't squeeze items into hallways. You find a new apartment twice as big, move everything over, and continue accumulating. The moves are infrequent, and each new place lasts twice as long before the next move. Over a long period, the average cost per item acquired is constant.

That is exactly what `std::vector` does with memory.

Internally it is still a **contiguous block of memory** — identical to a static array in layout. So it keeps the O(1) random access and the cache-friendliness of a static array, while adding the ability to grow at runtime. You pay for this with occasional O(n) resize operations and ~24 bytes of bookkeeping overhead (three pointers).

**The key insight:** a dynamic array is a static array with a resizing strategy baked in. Everything you know about static arrays applies — the address formula, cache lines, two pointers — all of it still works.

---

## 2. Internal Working — The Grow-and-Copy Engine

### The Three-Pointer Model

Every `std::vector` internally maintains three pointers (or equivalently, a pointer + two sizes):

```
Heap memory block:
┌────┬────┬────┬────┬────┬────┬────┬────┐
│ 10 │ 20 │ 30 │ 40 │    │    │    │    │
└────┴────┴────┴────┴────┴────┴────┴────┘
  ↑                    ↑                  ↑
begin               end (size=4)     end_of_storage (capacity=8)

vector object (lives on stack, 24 bytes):
┌──────────┬──────────┬──────────┐
│  *begin  │   *end   │ *cap_end │
└──────────┴──────────┴──────────┘
```

- **`begin`** — pointer to the first element  
- **`end`** — pointer one past the last valid element; `end - begin = size()`  
- **`end_of_storage`** — pointer to one past the last allocated byte; `end_of_storage - begin = capacity()`

`size()` = number of elements currently stored  
`capacity()` = number of elements that fit without reallocation

### The Growth Sequence

When `push_back` is called and `size == capacity`:

```
Step 1: Allocate a new block — typically 2× the current capacity
Step 2: Move (or copy) all existing elements to the new block
Step 3: Destroy elements in the old block
Step 4: Free the old block
Step 5: Update begin, end, end_of_storage to point into the new block
Step 6: Insert the new element

Visualised (growth factor = 2):

Initial:  capacity=1, size=0
push_back(A):            [A]               cap=1,  size=1
push_back(B):  reallocate→ [A][B][ ][ ]   cap=2→4, size=2  ← wait, this is wrong

Actually:
push_back(A):            [A]               cap=1,  size=1
push_back(B):  full → realloc → [A][B]     cap=2,  size=2
push_back(C):  full → realloc → [A][B][C][ ] cap=4, size=3
push_back(D):            [A][B][C][D]      cap=4,  size=4
push_back(E):  full → realloc → [A][B][C][D][E][ ][ ][ ] cap=8, size=5
```

**Growth factor is implementation-defined:**
- GCC's libstdc++: **growth factor = 2** (doubles each time)
- MSVC's STL: **growth factor = 1.5** (multiplies by 1.5 each time)
- Facebook's folly::fbvector: **growth factor = 1.5** (chosen to allow memory reuse)

**Why 1.5 can be better than 2:** With growth factor 2, the new allocation is always larger than the sum of all previous blocks combined, so the allocator can never reuse old memory. With factor 1.5, old blocks can eventually be reused. In practice for most applications, the difference is negligible — but it is worth knowing if you are interviewed about it.

### What Happens During Reallocation

```cpp
// Simplified view of what push_back does internally:
void push_back(const T& value) {
    if (size_ == capacity_) {
        // Grow
        size_t new_cap = (capacity_ == 0) ? 1 : capacity_ * 2;
        T* new_data = static_cast<T*>(::operator new(new_cap * sizeof(T)));

        // Move existing elements (move is O(1) per element for most types)
        for (size_t i = 0; i < size_; i++) {
            new (new_data + i) T(std::move(data_[i]));  // placement new + move
            data_[i].~T();                                // destroy old
        }

        ::operator delete(data_);  // free old block
        data_     = new_data;
        capacity_ = new_cap;
    }

    // Construct new element in place
    new (data_ + size_) T(value);
    size_++;
}
```

---

## 3. Amortized Analysis — Why push_back is O(1)

This is one of the most important concepts in data structures. Individual `push_back` operations can take O(n) time (during a reallocation). Yet we say `push_back` is **O(1) amortized**. Here is the proof.

### The Aggregate Method

Suppose we perform n `push_back` operations on an empty vector (growth factor = 2).

Reallocation happens at sizes: 1, 2, 4, 8, 16, ..., n  
Cost of each reallocation = number of elements copied = 1, 2, 4, 8, ..., n/2

Total extra work from all reallocations:
```
1 + 2 + 4 + 8 + ... + n/2 + n  =  n + n/2 + n/4 + ... + 1
                                 =  n × (1 + 1/2 + 1/4 + ...)
                                 =  n × 2
                                 =  2n
```

Total cost for n push_back operations = n (the insertions) + 2n (the copies) = **3n = O(n)**

**Amortized cost per operation = O(n) / n = O(1)**

### The Accounting / Banker's Analogy

Think of each `push_back` as depositing $3 into a bank account:
- $1 pays for the insertion itself
- $2 is saved for the future reallocation

When a reallocation doubles the array from size k to 2k, we copy k elements. The k elements being moved each contributed $2 in savings, giving us $2k to pay for exactly k copies ($2 each). The account never goes negative. Every operation is "paid for" — hence O(1) amortized.

### The Consequence: Reserve When You Know the Size

```cpp
// BAD: unknown final size → O(log n) reallocations
vector<int> v;
for (int i = 0; i < 1000000; i++) v.push_back(i);
// Caused ~20 reallocations, each copying progressively larger arrays

// GOOD: know the size → zero reallocations
vector<int> v;
v.reserve(1000000);       // single allocation upfront
for (int i = 0; i < 1000000; i++) v.push_back(i);
// Identical end result, but no wasted copies
```

`reserve()` is the single most impactful performance optimisation for vectors. Use it whenever you know (or can estimate) the final size.

---

## 4. Memory Layout & Iterator Invalidation

### Layout is Identical to a Static Array

Because `vector` is contiguous, everything from the static array chapter applies:
- `v[i]` computes the address as `base + i × sizeof(T)` — O(1), no indirection
- Sequential iteration is cache-friendly (same 64-byte cache line behaviour)
- Row-major 2D access still matters for `vector<vector<int>>`
- SIMD/vectorisation works the same way

### Iterator Invalidation — The #1 Source of Bugs

**Any operation that causes reallocation invalidates ALL iterators, pointers, and references into the vector.**

```cpp
vector<int> v = {1, 2, 3, 4, 5};
int* p = &v[0];           // pointer to first element
auto it = v.begin();      // iterator to first element

v.push_back(6);           // might reallocate!

// p and it are now DANGLING — undefined behaviour to dereference
cout << *p;               // UB: may crash, may print garbage
cout << *it;              // UB: same issue
```

**Operations that MAY invalidate iterators (cause reallocation):**
- `push_back`, `emplace_back` — if `size == capacity`
- `insert`, `emplace` — always invalidates at/after insertion point; full invalidation if realloc
- `resize` — if new size > capacity
- `reserve` — if new capacity > current capacity
- `assign` — always

**Operations that NEVER invalidate iterators:**
- `pop_back` — only end iterator is invalidated
- `erase` — only iterators at/after erase point are invalidated
- `swap` — all iterators remain valid (they now refer to the other vector's elements)
- `clear` — past-the-end iterator invalidated; no realloc

```cpp
// Safe pattern: do not hold iterators across modifying operations
vector<int> v = {1, 2, 3};
for (int i = 0; i < (int)v.size(); i++) {  // index-based: always safe
    if (v[i] % 2 == 0) v.push_back(v[i] * 10);  // might reallocate, but i is just an int
}

// DANGEROUS pattern: range-for while mutating
for (auto it = v.begin(); it != v.end(); ++it) {
    if (*it % 2 == 0) v.push_back(*it * 10);  // push_back can invalidate it → UB
}
```

### The vector<vector<int>> Pitfall

A 2D vector is NOT contiguous across rows:

```
vector<vector<int>> grid(3, vector<int>(4));

Memory layout:
  grid[0] → [heap block A: 4 ints]
  grid[1] → [heap block B: 4 ints]   ← NOT adjacent to A
  grid[2] → [heap block C: 4 ints]   ← NOT adjacent to B

Compare with int grid[3][4]:
  [row0 col0][row0 col1][row0 col2][row0 col3][row1 col0]...  ← all contiguous
```

For large matrices, a flat `vector<int>` of size `rows × cols` accessed as `v[i * cols + j]` is significantly faster due to cache locality.

---

## 5. Time & Space Complexity

### Operation Complexities

| Operation | Time | Notes |
|---|---|---|
| `v[i]`, `v.at(i)` | **O(1)** | Direct address computation, same as static array |
| `v.front()`, `v.back()` | **O(1)** | Direct pointer dereference |
| `push_back(x)` | **O(1) amortized** | Occasionally O(n) during reallocation |
| `emplace_back(args...)` | **O(1) amortized** | Constructs in-place; no copy/move |
| `pop_back()` | **O(1)** | Destructs last element, decrements size |
| `insert(it, x)` | **O(n)** | Shifts elements; O(1) if inserting at end |
| `erase(it)` | **O(n)** | Shifts elements left to fill gap |
| `erase(first, last)` | **O(n)** | Linear in elements remaining after range |
| `reserve(n)` | **O(n)** | Copies/moves all elements to new allocation |
| `resize(n)` | **O(n)** | Initialises new elements; may reallocate |
| `clear()` | **O(n)** | Destructs all elements (size→0, capacity unchanged) |
| `size()`, `capacity()` | **O(1)** | Just pointer arithmetic |
| `empty()` | **O(1)** | `begin == end` |
| `find` (unsorted) | **O(n)** | Linear scan |
| `binary_search` (sorted) | **O(log n)** | Requires sorted vector |
| Iteration (begin→end) | **O(n)** | Cache-friendly sequential access |

### Space Complexity

| | |
|---|---|
| Storage for n elements | **O(n)** |
| Internal overhead | 3 pointers = 24 bytes (on 64-bit) |
| Worst-case wasted capacity | up to **2× - 1** of elements (just after reallocation) |
| `shrink_to_fit()` | Requests capacity == size; implementation may ignore it |

---

## 6. std::vector API — Complete Reference

```cpp
#include <vector>
#include <algorithm>
#include <numeric>
using namespace std;

int main() {

    // ── Construction ──────────────────────────────────────────────────────────

    vector<int> v1;                        // empty, size=0, capacity=0
    vector<int> v2(5);                     // 5 elements, all zero-initialised
    vector<int> v3(5, 42);                 // 5 elements, all = 42
    vector<int> v4 = {1, 2, 3, 4, 5};     // initialiser list
    vector<int> v5(v4);                    // copy constructor — O(n)
    vector<int> v6(v4.begin(), v4.end());  // range constructor — O(n)
    vector<int> v7(move(v5));              // move constructor — O(1)! v5 is now empty

    // ── Size and Capacity ─────────────────────────────────────────────────────

    v4.size();           // 5   — elements currently stored
    v4.capacity();       // ≥5  — elements that fit without reallocation
    v4.empty();          // false
    v4.max_size();       // implementation-defined maximum

    v4.reserve(100);     // ensure capacity ≥ 100; no size change
    v4.resize(8);        // size → 8; new elements zero-initialised
    v4.resize(3);        // size → 3; elements 3..4 destroyed
    v4.shrink_to_fit();  // hint: reduce capacity to size (may be ignored)

    // ── Element Access ────────────────────────────────────────────────────────

    v4[0];               // O(1), NO bounds checking — undefined behaviour if OOB
    v4.at(0);            // O(1), throws std::out_of_range if OOB — use in debug
    v4.front();          // first element — undefined if empty
    v4.back();           // last element  — undefined if empty
    v4.data();           // raw pointer to underlying array — safe for C APIs

    // ── Modifiers ─────────────────────────────────────────────────────────────

    v4.push_back(99);           // append by copy — O(1) amortized
    v4.emplace_back(99);        // append by construction in-place — prefer this
    v4.pop_back();              // remove last — O(1); does NOT check empty!

    auto it = v4.begin() + 2;
    v4.insert(it, 77);          // insert before position — O(n)
    v4.insert(it, 3, 77);       // insert 3 copies of 77
    v4.insert(it, {1,2,3});     // insert initialiser list

    v4.emplace(it, 77);         // construct in-place before position — prefer over insert

    v4.erase(v4.begin());           // erase single element — O(n)
    v4.erase(v4.begin(), v4.begin() + 3);  // erase range — O(n)

    v4.clear();          // destroy all elements; size=0; capacity unchanged
    v4.swap(v6);         // O(1)! Swaps internal pointers; all iterators remain valid

    // ── Iteration ─────────────────────────────────────────────────────────────

    // Range-for (safest, most readable)
    for (int x : v4) cout << x << " ";

    // Index-based (use when you need the index)
    for (int i = 0; i < (int)v4.size(); i++) cout << v4[i] << " ";

    // Iterator-based (necessary for erase-during-iteration)
    for (auto it = v4.begin(); it != v4.end(); ) {
        if (*it % 2 == 0) it = v4.erase(it);  // erase returns next valid iterator
        else               ++it;
    }

    // ── Algorithms with vector ────────────────────────────────────────────────

    vector<int> nums = {5, 3, 1, 4, 2};

    sort(nums.begin(), nums.end());                      // [1,2,3,4,5]
    sort(nums.begin(), nums.end(), greater<int>());      // [5,4,3,2,1]
    reverse(nums.begin(), nums.end());
    int s = accumulate(nums.begin(), nums.end(), 0);     // sum
    auto it2 = find(nums.begin(), nums.end(), 3);        // linear search
    bool found = binary_search(nums.begin(), nums.end(), 3);  // sorted only
    int cnt = count(nums.begin(), nums.end(), 3);
    auto [mn, mx] = minmax_element(nums.begin(), nums.end());
    auto pos = lower_bound(nums.begin(), nums.end(), 3); // first ≥ 3 (sorted)
    auto pos2 = upper_bound(nums.begin(), nums.end(), 3);// first >  3 (sorted)

    // Remove-erase idiom — the CORRECT way to remove all matching elements
    // std::remove shifts matching elements to the end but does NOT resize
    nums.erase(remove(nums.begin(), nums.end(), 3), nums.end());

    return 0;
}
```

### emplace_back vs push_back — When It Actually Matters

```cpp
struct Point {
    int x, y;
    Point(int x, int y) : x(x), y(y) {}
};

vector<Point> pts;

// push_back: construct a temporary Point, then copy/move it into the vector
pts.push_back(Point(1, 2));   // 1 construction + 1 move = 2 ops

// push_back with brace-init (same cost)
pts.push_back({1, 2});        // same

// emplace_back: forward arguments directly, construct IN PLACE inside the vector
pts.emplace_back(1, 2);       // 1 construction only — no temporary

// For primitive types (int, double), there is ZERO difference.
// For complex objects with expensive constructors, emplace_back wins.
// Default: use emplace_back. It is never worse, sometimes better.
```

---

## 7. Common Patterns & Techniques

### The Erase-Remove Idiom

The single most commonly misused vector operation. `std::remove` does NOT actually remove elements — it rearranges them and returns a new end iterator. You must pair it with `erase`.

```cpp
vector<int> v = {1, 2, 3, 2, 4, 2, 5};

// Wrong: std::remove alone does not resize the vector
remove(v.begin(), v.end(), 2);   // v might be {1,3,4,5,?,?,?} — size still 7!

// Correct: erase-remove idiom
v.erase(remove(v.begin(), v.end(), 2), v.end());
// Now v = {1, 3, 4, 5}, size = 4

// C++20 shorthand:
erase(v, 2);                                // erase all equal to 2
erase_if(v, [](int x){ return x % 2 == 0; }); // erase all even elements
```

### Sorting with Custom Comparators

```cpp
vector<pair<int,int>> intervals = {{3,5},{1,4},{2,6}};

// Sort by start time, then by end time descending
sort(intervals.begin(), intervals.end(), [](const auto& a, const auto& b) {
    if (a.first != b.first) return a.first < b.first;
    return a.second > b.second;
});

// Sort indices without moving elements (argsort)
vector<int> idx(v.size());
iota(idx.begin(), idx.end(), 0);          // fill with 0,1,2,...
sort(idx.begin(), idx.end(), [&](int a, int b) {
    return v[a] < v[b];
});
// idx now gives access order from smallest to largest element of v
```

### Flatten / Build Results Pattern

```cpp
// Collecting results — classic pattern
vector<int> result;
result.reserve(n);  // if you know the max size
for (int i = 0; i < n; i++) {
    if (condition(i)) result.emplace_back(i);
}

// Building from multiple sources
vector<int> merged;
merged.insert(merged.end(), a.begin(), a.end());  // append vector a
merged.insert(merged.end(), b.begin(), b.end());  // append vector b
```

### Using vector as a Stack

```cpp
// vector is the best stack implementation in C++ for interviews
// push = push_back (O(1) amortized)
// pop  = pop_back  (O(1))
// top  = back()    (O(1))

vector<int> stk;
stk.emplace_back(5);     // push
stk.emplace_back(3);
int top = stk.back();    // peek
stk.pop_back();          // pop
```

### 2D Vector Initialisation

```cpp
int rows = 3, cols = 4;

// Method 1: vector of vectors (non-contiguous but flexible)
vector<vector<int>> grid(rows, vector<int>(cols, 0));
grid[1][2] = 7;

// Method 2: flat vector (contiguous, cache-friendly — prefer for large matrices)
vector<int> flat(rows * cols, 0);
auto at = [&](int r, int c) -> int& { return flat[r * cols + c]; };
at(1, 2) = 7;

// Method 3: std::array for fixed-size 2D (fastest, zero overhead)
array<array<int, 4>, 3> arr2d = {};
arr2d[1][2] = 7;
```

---

## 8. Interview Problems

### Problem 1: Merge Intervals

**Problem:** Given a list of intervals `[start, end]`, merge all overlapping intervals. Return the list of non-overlapping intervals after merging.

**Example:** `[[1,3],[2,6],[8,10],[15,18]]` → `[[1,6],[8,10],[15,18]]`

**Thought process:**

> "If I sort by start time, I can process intervals left to right. The current interval either extends the last merged interval (overlap) or starts a new one (no overlap). Overlap condition: `current.start <= last_merged.end`."

```cpp
// Time: O(n log n) for sort | Space: O(n) for output
vector<vector<int>> merge(vector<vector<int>>& intervals) {
    if (intervals.empty()) return {};

    // Sort by start time
    sort(intervals.begin(), intervals.end());

    vector<vector<int>> merged;
    merged.push_back(intervals[0]);  // seed with first interval

    for (int i = 1; i < (int)intervals.size(); i++) {
        auto& last = merged.back();

        if (intervals[i][0] <= last[1]) {
            // Overlap: extend the end if necessary
            last[1] = max(last[1], intervals[i][1]);
        } else {
            // No overlap: start a new merged interval
            merged.push_back(intervals[i]);
        }
    }
    return merged;
}
```

**Edge cases to mention:**
- Empty input → return `{}`
- Single interval → return it as-is
- All intervals overlap → one giant merged interval
- Nested intervals like `[1,10],[2,3]` — sorting + `max(end)` handles this correctly
- Adjacent intervals like `[1,3],[3,5]` — `3 <= 3` is true, so they DO merge to `[1,5]`

**Pattern:** Sort + linear scan. Sort makes an otherwise O(n²) comparison into a single pass. Extremely common pattern — also appears in meeting rooms, non-overlapping intervals, and scheduling problems.

---

### Problem 2: Product of Array Except Self

**Problem:** Given array `nums`, return an array `output` where `output[i]` = product of all elements except `nums[i]`. Do it in O(n) time without using division.

**Example:** `[1, 2, 3, 4]` → `[24, 12, 8, 6]`

**Thought process:**

> "For each position i, I need: (product of everything to the left of i) × (product of everything to the right of i). I can compute left-products in one pass, right-products in another — both O(n). Combine them in-place."

```cpp
// Time: O(n) | Space: O(1) excluding output array
vector<int> productExceptSelf(vector<int>& nums) {
    int n = nums.size();
    vector<int> result(n, 1);

    // Pass 1: result[i] = product of all elements LEFT of i
    // result[0] = 1 (nothing to the left)
    // result[1] = nums[0]
    // result[2] = nums[0] * nums[1]  ... etc.
    int prefix = 1;
    for (int i = 0; i < n; i++) {
        result[i] = prefix;
        prefix *= nums[i];
    }

    // Pass 2: multiply by product of all elements RIGHT of i
    // Traverse right to left, maintaining a running suffix product
    int suffix = 1;
    for (int i = n - 1; i >= 0; i--) {
        result[i] *= suffix;
        suffix *= nums[i];
    }

    return result;
}

/*
Dry run for [1, 2, 3, 4]:

After Pass 1 (prefix products):
  result = [1, 1, 2, 6]
  (result[i] = product of nums[0..i-1])

After Pass 2 (multiply by suffix products):
  i=3: result[3] = 6 * 1 = 6,  suffix = 4
  i=2: result[2] = 2 * 4 = 8,  suffix = 12
  i=1: result[1] = 1 * 12 = 12, suffix = 24
  i=0: result[0] = 1 * 24 = 24, suffix = 24

Final: [24, 12, 8, 6] ✓
*/
```

**Edge cases:**
- Array with one zero: all results are 0 except the position of the zero
- Array with two zeros: all results are 0 (no position has a nonzero product)
- Array with negative numbers: signs work out correctly — no special handling needed
- Single element: result is `[1]`

**Why no division?** The naive approach divides total product by `nums[i]`. Division fails if `nums[i] == 0`. The prefix-suffix approach handles zeros elegantly without any division.

---

### Problem 3: Next Permutation

**Problem:** Rearrange `nums` into its lexicographically next permutation in-place. If no next permutation exists (i.e., array is in descending order), rearrange to the smallest permutation (ascending order).

**Example:** `[1,2,3]`→`[1,3,2]` | `[3,2,1]`→`[1,2,3]` | `[1,1,5]`→`[1,5,1]`

**Thought process (the hardest part — figuring out the algorithm):**

> "Look at `[1,2,3]`. The next permutation is `[1,3,2]`. We changed the 2 → 3 and put 2 after it. Why 3? It's the smallest number greater than 2 that appears to its right. Then we sort everything to the right of the swap position to get the smallest possible suffix."
>
> "Algorithm: scan right to left to find the first position i where `nums[i] < nums[i+1]` (the pivot). Everything to the right of i is in descending order. Find the smallest element to the right of i that is greater than `nums[i]` — that's j. Swap i and j. Reverse the suffix starting at i+1 (it was descending, now it becomes ascending = smallest)."

```cpp
// Time: O(n) | Space: O(1)
void nextPermutation(vector<int>& nums) {
    int n = nums.size();

    // Step 1: Find the pivot — rightmost position where nums[i] < nums[i+1]
    int pivot = -1;
    for (int i = n - 2; i >= 0; i--) {
        if (nums[i] < nums[i + 1]) {
            pivot = i;
            break;
        }
    }

    // If no pivot: entire array is descending (e.g., [3,2,1])
    // Next permutation wraps around to [1,2,3]
    if (pivot == -1) {
        reverse(nums.begin(), nums.end());
        return;
    }

    // Step 2: Find the rightmost element just greater than nums[pivot]
    // (the suffix is descending, so scan from right)
    for (int j = n - 1; j > pivot; j--) {
        if (nums[j] > nums[pivot]) {
            swap(nums[pivot], nums[j]);
            break;
        }
    }

    // Step 3: Reverse the suffix after pivot to make it ascending (smallest)
    reverse(nums.begin() + pivot + 1, nums.end());
}

/*
Dry run for [1, 3, 5, 4, 2]:
  Step 1: Scan right-to-left for first dip
    i=3: nums[3]=4 > nums[4]=2 — still descending
    i=2: nums[2]=5 > nums[3]=4 — still descending
    i=1: nums[1]=3 < nums[2]=5 — FOUND pivot at index 1

  Step 2: Find rightmost > nums[1]=3 in suffix [5,4,2]
    j=4: nums[4]=2 < 3 — skip
    j=3: nums[3]=4 > 3 — swap! → [1, 4, 5, 3, 2]
                                          ↑pivot ↑

  Step 3: Reverse suffix after pivot [5,3,2] → [2,3,5]
    Result: [1, 4, 2, 3, 5] ✓ (next permutation after [1,3,5,4,2])
*/
```

**Why this works:** The suffix to the right of the pivot is in descending order (by definition — we scanned for the first violation from the right). After swapping the pivot with the next-larger element in the suffix, the suffix is still mostly descending. Reversing it gives the smallest possible arrangement, ensuring we get exactly the next permutation.

---

## 9. Real-World Uses

| Domain | Use Case | Why Vector |
|---|---|---|
| C++ STL internally | Default return type from most algorithms | Contiguous, growable, interoperable |
| Game development | Entity component lists, vertex buffers | Cache-efficient sequential processing |
| Compilers | AST node children, token lists | Unknown count at parse time |
| Database engines | Query result sets, column data (DuckDB) | Contiguous for SIMD scans |
| Machine learning | Feature vectors, batch data, embedding matrices | Flat memory for BLAS routines |
| Operating systems | Free page lists, process tables | Dynamic, random-access |
| Web servers | HTTP request headers, routing tables | Built once, read many times |
| Competitive programming | Almost everything | Fastest in practice, reserve eliminates overhead |

**Real example — how `std::vector` enables SIMD:**  
Modern compilers auto-vectorise loops over contiguous memory. A loop summing `vector<float>` can use AVX2 to process 8 floats per instruction — 8× speedup. This only works because vector is contiguous. `std::list` gets zero benefit from SIMD.

---

## 10. Edge Cases & Pitfalls

### Pitfall 1: Comparing size() with Signed Integers

```cpp
vector<int> v = {1, 2, 3};

// DANGEROUS: v.size() returns size_t (unsigned)
// If i = -1, the comparison -1 < v.size() compares signed with unsigned
// -1 gets converted to a HUGE unsigned number → loop runs incorrectly
for (int i = v.size() - 1; i >= 0; i--) {  // ok here, but read on

// Actually dangerous:
if (v.size() - 1 == -1) { ... }  // v.size()-1 when size=0 wraps to SIZE_MAX!

// CORRECT: cast to int, or use ssize_t, or use ptrdiff_t
for (int i = (int)v.size() - 1; i >= 0; i--) { ... }  // safe
```

### Pitfall 2: Accessing Empty Vector

```cpp
vector<int> v;
v.back();             // UB — undefined behaviour on empty vector
v.front();            // UB
v.pop_back();         // UB
v[0];                 // UB

// Always guard:
if (!v.empty()) v.pop_back();
if (!v.empty()) cout << v.back();
```

### Pitfall 3: Reference Invalidation in the Same Expression

```cpp
vector<int> v = {1, 2, 3};

// Classic bug: push_back with a reference INTO the same vector
v.push_back(v[0]);   // v[0] is a reference; push_back may reallocate,
                     // invalidating the reference BEFORE reading it → UB

// Safe: copy the value first
int val = v[0];
v.push_back(val);    // safe
```

### Pitfall 4: The Initialiser List Constructor vs Size Constructor

```cpp
vector<int> v1(3, 5);    // 3 elements, all = 5: {5, 5, 5}
vector<int> v2{3, 5};    // 2 elements: {3, 5}   ← initialiser list!

// This distinction matters:
vector<int> v3(1);        // {0}  — one zero-initialised element
vector<int> v4{1};        // {1}  — one element with value 1
```

### Pitfall 5: Modifying a Vector While Iterating with Range-For

```cpp
vector<int> v = {1, 2, 3, 4, 5};

// WRONG: modifying container during range-for
for (int x : v) {
    if (x == 3) v.push_back(99);  // may reallocate → UB
}

// CORRECT approach 1: index-based loop
for (int i = 0; i < (int)v.size(); i++) { ... }

// CORRECT approach 2: collect changes, apply after
vector<int> toAdd;
for (int x : v) if (x == 3) toAdd.push_back(99);
v.insert(v.end(), toAdd.begin(), toAdd.end());
```

### Pitfall 6: Forgetting to Reserve (Performance)

```cpp
// This runs 20 reallocations for 1,000,000 elements:
vector<int> v;
for (int i = 0; i < 1000000; i++) v.push_back(i);

// This runs 0 reallocations:
vector<int> v;
v.reserve(1000000);
for (int i = 0; i < 1000000; i++) v.push_back(i);

// Benchmark difference: 2-5× faster with reserve in tight loops
```

### Pitfall 7: Using vector<bool> — It Is Not a Vector of Bools

```cpp
// vector<bool> is a SPECIALISATION that packs bits.
// It does NOT store actual bool values — it stores bits.
// As a consequence, operator[] returns a PROXY object, not a bool&.

vector<bool> vb = {true, false, true};
auto x = vb[0];       // x is NOT bool — it's a proxy. auto breaks here.
bool* p = &vb[0];     // COMPILE ERROR — cannot take address of proxy

// Solutions:
vector<char> vc = {1, 0, 1};   // use char instead
vector<uint8_t> vi = {1,0,1};  // or uint8_t
// Or explicitly:
bool b = vb[0];                 // safe — explicit bool conversion
```

---

## 11. Implementing a Dynamic Array from Scratch

This is asked in senior-level interviews. Building it yourself reveals deep understanding of memory management, placement new, and amortized complexity. This is production-quality implementation.

```cpp
#include <cstddef>
#include <stdexcept>
#include <utility>
#include <algorithm>

template<typename T>
class DynamicArray {
private:
    T*      data_;      // pointer to heap-allocated block
    size_t  size_;      // number of elements stored
    size_t  capacity_;  // total allocated slots

    void grow() {
        size_t new_cap = (capacity_ == 0) ? 1 : capacity_ * 2;
        reallocate(new_cap);
    }

    void reallocate(size_t new_cap) {
        // Allocate raw memory — no construction
        T* new_data = static_cast<T*>(::operator new(new_cap * sizeof(T)));

        // Move-construct existing elements into new memory
        for (size_t i = 0; i < size_; i++) {
            new (new_data + i) T(std::move(data_[i]));  // placement new + move
            data_[i].~T();                               // explicitly destroy old
        }

        ::operator delete(data_);   // free old raw memory
        data_     = new_data;
        capacity_ = new_cap;
    }

public:
    // ── Constructors ──────────────────────────────────────────────────────────

    DynamicArray() : data_(nullptr), size_(0), capacity_(0) {}

    explicit DynamicArray(size_t n, const T& val = T{}) : data_(nullptr), size_(0), capacity_(0) {
        reserve(n);
        for (size_t i = 0; i < n; i++) emplace_back(val);
    }

    // Copy constructor — deep copy
    DynamicArray(const DynamicArray& other) : data_(nullptr), size_(0), capacity_(0) {
        reserve(other.size_);
        for (size_t i = 0; i < other.size_; i++) emplace_back(other.data_[i]);
    }

    // Move constructor — O(1), steals the pointer
    DynamicArray(DynamicArray&& other) noexcept
        : data_(other.data_), size_(other.size_), capacity_(other.capacity_) {
        other.data_     = nullptr;
        other.size_     = 0;
        other.capacity_ = 0;
    }

    // Destructor — must destroy each element, then free raw memory
    ~DynamicArray() {
        for (size_t i = 0; i < size_; i++) data_[i].~T();
        ::operator delete(data_);
    }

    // Copy assignment — copy-and-swap idiom
    DynamicArray& operator=(DynamicArray other) {
        swap(*this, other);  // other holds our old data, gets destroyed on exit
        return *this;
    }

    friend void swap(DynamicArray& a, DynamicArray& b) noexcept {
        std::swap(a.data_,     b.data_);
        std::swap(a.size_,     b.size_);
        std::swap(a.capacity_, b.capacity_);
    }

    // ── Capacity ──────────────────────────────────────────────────────────────

    size_t size()     const { return size_; }
    size_t capacity() const { return capacity_; }
    bool   empty()    const { return size_ == 0; }

    void reserve(size_t new_cap) {
        if (new_cap > capacity_) reallocate(new_cap);
    }

    // ── Element Access ────────────────────────────────────────────────────────

    T& operator[](size_t i)       { return data_[i]; }
    const T& operator[](size_t i) const { return data_[i]; }

    T& at(size_t i) {
        if (i >= size_) throw std::out_of_range("index out of range");
        return data_[i];
    }

    T& front() { return data_[0]; }
    T& back()  { return data_[size_ - 1]; }
    T* data()  { return data_; }

    // ── Modifiers ─────────────────────────────────────────────────────────────

    template<typename... Args>
    void emplace_back(Args&&... args) {
        if (size_ == capacity_) grow();
        new (data_ + size_) T(std::forward<Args>(args)...);  // construct in-place
        size_++;
    }

    void push_back(const T& val) { emplace_back(val); }
    void push_back(T&& val)      { emplace_back(std::move(val)); }

    void pop_back() {
        if (empty()) return;
        data_[--size_].~T();  // explicitly destroy last element
    }

    void clear() {
        for (size_t i = 0; i < size_; i++) data_[i].~T();
        size_ = 0;
        // capacity unchanged — no reallocation
    }

    // ── Iterators ─────────────────────────────────────────────────────────────

    T* begin()  { return data_; }
    T* end()    { return data_ + size_; }
    const T* begin() const { return data_; }
    const T* end()   const { return data_ + size_; }
};

// Usage:
int main() {
    DynamicArray<int> arr;
    arr.reserve(10);
    for (int i = 0; i < 5; i++) arr.emplace_back(i * i);  // {0,1,4,9,16}
    arr.pop_back();                                          // {0,1,4,9}
    for (int x : arr) cout << x << " ";                    // 0 1 4 9
    return 0;
}
```

**Key design decisions to explain in an interview:**
1. **`::operator new` not `malloc`** — we want raw memory without construction. `new T[n]` would default-construct n elements, which we don't want.
2. **Placement new** — constructs an object at an already-allocated address. The only way to construct into raw memory.
3. **Explicit destructor calls** — since we used placement new, the compiler won't auto-destroy. We must call `data_[i].~T()` manually.
4. **Copy-and-swap idiom** — makes assignment exception-safe and DRY.
5. **`noexcept` on move** — critical for STL compatibility. `std::vector` will move your objects during reallocation only if the move constructor is `noexcept`. Otherwise it falls back to copying for exception safety.

---

## 12. Comparison

| | `int arr[N]` | `std::array<int,N>` | `std::vector<int>` | `std::deque<int>` | `std::list<int>` |
|---|---|---|---|---|---|
| Size | Fixed, compile-time | Fixed, compile-time | Dynamic | Dynamic | Dynamic |
| Memory | Stack or static | Stack | Heap (contiguous) | Heap (chunked) | Heap (nodes) |
| Random access | O(1) | O(1) | O(1) | O(1) | O(n) |
| push_back | N/A | N/A | O(1) amortized | O(1) amortized | O(1) |
| push_front | N/A | N/A | O(n) | O(1) amortized | O(1) |
| Insert middle | O(n) | O(n) | O(n) | O(n) | O(1) |
| Cache friendly | ✓✓✓ | ✓✓✓ | ✓✓✓ | Partial | ✗ |
| Iterator invalidation | N/A | N/A | On realloc | Never (but pointers invalid) | Never |
| Overhead | 0 | 0 | 24 bytes | ~48 bytes | 16 bytes/node |
| **When to use** | Fixed size, stack | Fixed size, modern | Default choice | Need fast front insertion | Frequent mid-insert |

**Rule of thumb:** Use `vector` by default. Switch to `deque` only if you need fast `push_front`. Switch to `list` only if you insert/delete in the middle using iterators (this is rarer than people think — profile first).

---

## 13. Self-Test Questions

1. **What are the three internal pointers of std::vector? What does each represent?**

2. **push_back is O(1) amortized. Walk me through the proof using the aggregate method.**

3. **What is the difference between `size()` and `capacity()`? Can capacity ever be less than size?**

4. **When does a `push_back` call invalidate all iterators? When does it NOT?**

5. **Why is `vector<bool>` special and dangerous? What should you use instead?**

6. **Write the erase-remove idiom. Why does `std::remove` alone not actually remove elements?**

7. **What is placement new? Why does a hand-rolled dynamic array need it?**

8. **If you need to store 1 million integers and the final count is known upfront, what is the single most important optimisation you should make?**

9. **Why is `vector<vector<int>>` cache-unfriendly for large matrices? How do you fix it?**

10. **Explain why `emplace_back(v[0])` is dangerous. Write the safe version.**

---

## Quick Reference Card

```
std::vector — the workhorse of C++ containers.

Internally: contiguous heap block + 3 pointers (begin, end, cap_end)
Growth:     double capacity on overflow → O(1) amortized push_back

Key operations:
  push_back / emplace_back  O(1) amortized  ← use emplace_back always
  pop_back                  O(1)
  operator[]                O(1), no bounds check
  at()                      O(1), throws on OOB
  insert / erase (middle)   O(n)            ← avoid in hot paths
  reserve(n)                O(n)            ← call this if size is known

Critical rules:
  1. Any reallocation invalidates ALL iterators/pointers/references
  2. v.size() is unsigned — cast to int before subtracting
  3. vector<bool> is broken — use vector<char> instead
  4. Erase-remove idiom for removing elements: v.erase(remove(...), v.end())
  5. emplace_back > push_back for non-trivial types

When to reserve:
  → You know (or can bound) the final size
  → You're in a hot loop
  → Profiling shows excessive reallocations
```

---

*Previous: [01 — Static Array](./01_static_array.md)*  
*Next: [03 — Singly Linked List](./03_singly_linked_list.md)*  
*See also: [Stack (vector-backed)](./05_stack.md) | [Merge Intervals pattern](./algorithms/interval_problems.md)*

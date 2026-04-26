# Static Array

> **Curriculum position:** Linear Structures → #1  
> **Interview weight:** ★★★★★ Critical — every other data structure builds on this  
> **Difficulty:** Beginner concept, expert-level mastery

---

## Table of Contents

1. [Intuition First — What Is It Really?](#1-intuition-first)
2. [Internal Working — How the Machine Sees It](#2-internal-working)
3. [Memory Layout & Cache Behaviour](#3-memory-layout--cache-behaviour)
4. [Time & Space Complexity](#4-time--space-complexity)
5. [Core Operations in C++](#5-core-operations-in-c)
6. [Common Patterns & Techniques](#6-common-patterns--techniques)
7. [Interview Problems](#7-interview-problems)
8. [Real-World Uses](#8-real-world-uses)
9. [Edge Cases & Pitfalls](#9-edge-cases--pitfalls)
10. [Comparison with Dynamic Array](#10-comparison-with-dynamic-array)
11. [Self-Test Questions](#11-self-test-questions)

---

## 1. Intuition First

Imagine you rent 10 consecutive seats in a cinema hall. You know exactly which seat is which (seat 3 is the 3rd from the left), you can walk directly to any seat without asking anyone, and all seats are right next to each other — no gaps, no jumping around.

That is a static array.

- **Fixed size** — you decide at creation time. You cannot add an 11th seat without booking a whole new hall.
- **Contiguous memory** — all elements sit side-by-side in RAM, like houses on a straight road with no gaps.
- **Index = address** — the compiler computes the exact RAM address of any element in one arithmetic operation. There is no searching, no following pointers.

This directness is why arrays are the backbone of almost every other data structure. Hash tables are arrays under the hood. Heaps are arrays. Graphs can be represented as arrays of arrays. Even CPU caches are essentially arrays.

**The fundamental trade-off:** You get blazing O(1) access to any element, but you give up flexibility — size is fixed, and inserting or deleting in the middle is expensive.

---

## 2. Internal Working

### Memory Allocation

```
int arr[5];
```

When the compiler sees this, it reserves exactly `5 × sizeof(int) = 5 × 4 = 20 bytes` of contiguous memory — either on the **stack** (for local arrays) or the **heap** (for `new int[5]`).

```
Stack frame (grows downward):
┌─────────────────────────────────────────────┐
│  arr[0] │ arr[1] │ arr[2] │ arr[3] │ arr[4] │
│  1000   │  1004  │  1008  │  1012  │  1016  │  ← memory addresses
└─────────────────────────────────────────────┘
           base address = 1000
```

### Index-to-Address Formula

This is the key formula the CPU uses every single time you write `arr[i]`:

```
address(arr[i]) = base_address + i × sizeof(element_type)
```

For `int arr[5]` with base at address `1000`:
- `arr[0]` → `1000 + 0×4 = 1000`
- `arr[3]` → `1000 + 3×4 = 1012`
- `arr[i]` → computed in **one multiplication + one addition** — this is why access is O(1)

### Stack vs Heap Arrays

```cpp
// Stack array — automatic storage duration
// Allocated at function entry, freed at function exit
// Size must be a compile-time constant (in standard C++)
int stack_arr[100];

// Heap array — dynamic storage duration
// You control lifetime, must manually free
// Size can be a runtime variable
int* heap_arr = new int[n];
// ... use it ...
delete[] heap_arr;   // ← MUST do this. Forgetting = memory leak.
```

**Why does stack allocation matter?** Stack memory is pre-allocated per-thread by the OS (typically 1–8 MB). It is extremely fast — just moving a stack pointer. But large arrays on the stack cause **stack overflow**. Rule of thumb: anything over ~1 MB should go on the heap.

### What "Static" Actually Means

The word "static" here means **fixed-size at creation time**, not that it uses C's `static` keyword. A static array's size cannot change after it is created. The memory block is one solid chunk — you cannot "extend" it in place because there may be other data immediately after it in memory.

---

## 3. Memory Layout & Cache Behaviour

This section separates engineers who understand arrays from those who merely use them.

### Cache Lines

Modern CPUs do not fetch individual bytes from RAM. They fetch **cache lines** — typically **64 bytes** at a time — into L1/L2/L3 cache. For a 4-byte `int` array:

```
One cache line = 64 bytes = 16 integers loaded at once
                                                        
arr: [ 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11 | 12 | 13 | 14 | 15 | 16 | 17 ... ]
     |←————————— cache line 1 (loaded on first access) ——————————→|←— cache line 2 —→|
```

When you access `arr[0]`, the CPU loads `arr[0]` through `arr[15]` into cache simultaneously. Accessing `arr[1]` through `arr[15]` next is essentially free — they are already in cache. This is called **spatial locality**.

### Sequential vs Random Access

```cpp
// Sequential access — FAST. Every cache line fully used.
long long sum = 0;
for (int i = 0; i < N; i++) sum += arr[i];  // cache hit rate ≈ 100%

// Random access — SLOW. Cache lines loaded but mostly wasted.
for (int i = 0; i < N; i++) sum += arr[rand() % N];  // many cache misses
```

On a modern machine, accessing L1 cache costs ~4 cycles. A cache miss to RAM costs ~200 cycles. This is a **50× difference**. In performance-critical code, cache behaviour often matters more than algorithmic complexity.

### Row-Major vs Column-Major (2D Arrays)

```cpp
int matrix[N][M];  // C++ stores this in ROW-MAJOR order

// FAST: iterating row by row (sequential in memory)
for (int i = 0; i < N; i++)
    for (int j = 0; j < M; j++)
        process(matrix[i][j]);  // ✓ sequential, cache-friendly

// SLOW: iterating column by column (jumping M elements each step)
for (int j = 0; j < M; j++)
    for (int i = 0; i < N; i++)
        process(matrix[i][j]);  // ✗ stride = M, causes cache thrashing
```

For large matrices (e.g., 1000×1000), the row-major loop can be **5–10× faster** than column-major — purely due to cache effects, with identical algorithmic complexity.

---

## 4. Time & Space Complexity

### Operation Complexities

| Operation | Time Complexity | Notes |
|---|---|---|
| Access by index `arr[i]` | **O(1)** | One address computation |
| Search (unsorted) | **O(n)** | Must check every element |
| Search (sorted, binary search) | **O(log n)** | Requires sorted order |
| Insert at end (if space) | **O(1)** | Just write to next slot |
| Insert at middle/beginning | **O(n)** | Must shift all elements right |
| Delete at end | **O(1)** | Just decrement a counter |
| Delete at middle/beginning | **O(n)** | Must shift all elements left |
| Update `arr[i] = v` | **O(1)** | Direct address write |
| Traversal | **O(n)** | Visit every element once |
| Prefix sum query (after build) | **O(1)** | Requires O(n) preprocessing |

### Space Complexity

| | |
|---|---|
| Space for n elements | **O(n)** |
| Auxiliary space (most operations) | **O(1)** |
| Space for 2D array n×m | **O(n×m)** |

### Why Insert/Delete at Middle is O(n) — Visualised

```
Delete arr[2] from [10, 20, 30, 40, 50]:

Before: [ 10 | 20 | 30 | 40 | 50 ]
                    ↑
                 delete this

Step 1: [ 10 | 20 |    | 40 | 50 ]   ← hole created
Step 2: [ 10 | 20 | 40 |    | 50 ]   ← shift arr[3] left
Step 3: [ 10 | 20 | 40 | 50 |    ]   ← shift arr[4] left
After:  [ 10 | 20 | 40 | 50 ]        ← done, size = 4

Worst case: delete from index 0 → shift ALL n-1 elements → O(n)
```

---

## 5. Core Operations in C++

### Declaration and Initialisation

```cpp
#include <iostream>
#include <algorithm>
#include <numeric>
using namespace std;

int main() {
    // ── Stack arrays ──────────────────────────────────────────────

    int a[5];                        // uninitialised — contains garbage values!
    int b[5] = {1, 2, 3, 4, 5};     // fully initialised
    int c[5] = {1, 2};              // partial init: c = {1, 2, 0, 0, 0}
    int d[5] = {};                   // zero-initialised: {0, 0, 0, 0, 0}
    int e[] = {10, 20, 30};         // size inferred: e has 3 elements

    // 2D array
    int grid[3][4] = {
        {1,  2,  3,  4},
        {5,  6,  7,  8},
        {9, 10, 11, 12}
    };

    // ── Heap arrays ───────────────────────────────────────────────

    int n = 10;
    int* heap_arr = new int[n]();    // () = zero-initialises
    // ... use heap_arr ...
    delete[] heap_arr;               // always pair new[] with delete[]

    // ── std::array (preferred in modern C++) ──────────────────────
    // Fixed size like C array, but with bounds checking, iterators, etc.
    #include <array>
    array<int, 5> arr = {1, 2, 3, 4, 5};
    cout << arr.size() << "\n";      // 5
    cout << arr.at(2)  << "\n";      // 3, throws if out of bounds
    cout << arr[2]     << "\n";      // 3, no bounds check (faster)
}
```

### Core Operations with Explanation

```cpp
#include <iostream>
#include <algorithm>
#include <numeric>
using namespace std;

// ── Linear Search ─────────────────────────────────────────────────────────────
// When: unsorted array, or only 1 search needed
// Time: O(n) | Space: O(1)
int linearSearch(int arr[], int n, int target) {
    for (int i = 0; i < n; i++) {
        if (arr[i] == target) return i;  // return index
    }
    return -1;  // not found
}

// ── Binary Search ─────────────────────────────────────────────────────────────
// When: SORTED array, multiple searches needed
// Time: O(log n) | Space: O(1)
// Key insight: each comparison halves the search space
int binarySearch(int arr[], int n, int target) {
    int lo = 0, hi = n - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;  // NOT (lo+hi)/2 — avoids integer overflow!
        if      (arr[mid] == target) return mid;
        else if (arr[mid] <  target) lo = mid + 1;
        else                          hi = mid - 1;
    }
    return -1;
}

// ── Insert at Position ────────────────────────────────────────────────────────
// Shift elements right to make room
// Time: O(n) | Space: O(1) in-place
// arr has capacity >= n+1
void insertAt(int arr[], int& n, int pos, int val) {
    // Shift right from the end to avoid overwriting
    for (int i = n; i > pos; i--) {
        arr[i] = arr[i - 1];
    }
    arr[pos] = val;
    n++;
}

// ── Delete at Position ────────────────────────────────────────────────────────
// Shift elements left to fill the hole
// Time: O(n) | Space: O(1)
void deleteAt(int arr[], int& n, int pos) {
    for (int i = pos; i < n - 1; i++) {
        arr[i] = arr[i + 1];
    }
    n--;
}

// ── Reverse in Place ─────────────────────────────────────────────────────────
// Two-pointer technique — the most fundamental array pattern
// Time: O(n) | Space: O(1)
void reverse(int arr[], int n) {
    int left = 0, right = n - 1;
    while (left < right) {
        swap(arr[left], arr[right]);
        left++;
        right--;
    }
}

// ── Prefix Sum Array ─────────────────────────────────────────────────────────
// Build once in O(n), then answer range-sum queries in O(1)
// prefix[i] = arr[0] + arr[1] + ... + arr[i]
// Sum of arr[l..r] = prefix[r] - prefix[l-1]
void buildPrefixSum(int arr[], int prefix[], int n) {
    prefix[0] = arr[0];
    for (int i = 1; i < n; i++) {
        prefix[i] = prefix[i - 1] + arr[i];
    }
}

int rangeSum(int prefix[], int l, int r) {
    if (l == 0) return prefix[r];
    return prefix[r] - prefix[l - 1];
}

// ── Rotate Array ─────────────────────────────────────────────────────────────
// Rotate left by k positions using three-reverse trick
// Time: O(n) | Space: O(1)
void rotateLeft(int arr[], int n, int k) {
    k = k % n;                   // handle k > n
    reverse(arr,     k);         // reverse first k elements
    reverse(arr + k, n - k);     // reverse remaining
    reverse(arr,     n);         // reverse entire array
}

// ── Kadane's Algorithm (Maximum Subarray Sum) ─────────────────────────────────
// Classic DP on array. See Interview Problem #1 for full explanation.
// Time: O(n) | Space: O(1)
int maxSubarraySum(int arr[], int n) {
    int maxSoFar = arr[0];
    int maxEndingHere = arr[0];
    for (int i = 1; i < n; i++) {
        maxEndingHere = max(arr[i], maxEndingHere + arr[i]);
        maxSoFar      = max(maxSoFar, maxEndingHere);
    }
    return maxSoFar;
}

int main() {
    int arr[10] = {3, 1, 4, 1, 5, 9, 2, 6, 5, 3};
    int n = 10;

    cout << "Linear search for 9: index " << linearSearch(arr, n, 9) << "\n";

    sort(arr, arr + n);
    cout << "Binary search for 5: index " << binarySearch(arr, n, 5) << "\n";

    int prefix[10];
    buildPrefixSum(arr, prefix, n);
    cout << "Sum of indices 2..6: " << rangeSum(prefix, 2, 6) << "\n";

    return 0;
}
```

---

## 6. Common Patterns & Techniques

These are the building blocks you will use in 80% of array interview problems.

### Pattern 1: Two Pointers

Use two indices — typically `left` and `right` — that move toward each other or in the same direction. Eliminates nested loops in many problems.

```cpp
// Template: opposite ends moving inward
// Used for: palindrome check, pair sum, container with most water
int left = 0, right = n - 1;
while (left < right) {
    // process arr[left] and arr[right]
    if (condition_to_move_left)  left++;
    else                          right--;
}

// Template: fast/slow pointers (same direction)
// Used for: remove duplicates, move zeroes, partition
int slow = 0;
for (int fast = 0; fast < n; fast++) {
    if (keep(arr[fast])) {
        arr[slow++] = arr[fast];
    }
}
```

### Pattern 2: Sliding Window

A contiguous subarray of variable or fixed size that "slides" across the array. Converts O(n²) brute force to O(n).

```cpp
// Fixed-size window of length k
// Template for: max sum subarray of size k, etc.
int windowSum = 0;
for (int i = 0; i < k; i++) windowSum += arr[i];   // build first window

int maxSum = windowSum;
for (int i = k; i < n; i++) {
    windowSum += arr[i];        // add new element on right
    windowSum -= arr[i - k];   // remove old element on left
    maxSum = max(maxSum, windowSum);
}

// Variable-size window
// Template for: smallest subarray with sum >= target, longest substring, etc.
int left = 0, current = 0;
for (int right = 0; right < n; right++) {
    current += arr[right];               // expand window
    while (current >= target) {
        // record answer using (right - left + 1) as window size
        current -= arr[left++];          // shrink window
    }
}
```

### Pattern 3: Prefix Sum

Build a prefix sum array once in O(n), then answer any range-sum query in O(1). Non-obvious but critical.

```cpp
// 1D prefix sum (see implementation above)

// 2D prefix sum — for rectangle sum queries on a matrix
// prefix[i][j] = sum of all elements in rectangle (0,0) to (i-1,j-1)
int prefix[N+1][M+1] = {};
for (int i = 1; i <= N; i++)
    for (int j = 1; j <= M; j++)
        prefix[i][j] = arr[i-1][j-1]
                     + prefix[i-1][j]
                     + prefix[i][j-1]
                     - prefix[i-1][j-1];  // inclusion-exclusion

// Query: sum of rectangle (r1,c1) to (r2,c2) in 0-indexed arr
auto rectSum = [&](int r1, int c1, int r2, int c2) {
    return prefix[r2+1][c2+1]
         - prefix[r1][c2+1]
         - prefix[r2+1][c1]
         + prefix[r1][c1];
};
```

### Pattern 4: In-Place Manipulation

Many problems ask you to rearrange an array without extra space. Key insight: use the array's own indices to encode state.

```cpp
// Mark visited elements by negating them (works when values are positive)
// Used in: find missing number, find duplicate (Floyd-style)
for (int i = 0; i < n; i++) {
    int idx = abs(arr[i]) - 1;        // use value as index
    if (arr[idx] > 0) arr[idx] *= -1; // mark as seen
    else { /* arr[idx] already negative → duplicate found */ }
}
```

---

## 7. Interview Problems

### Problem 1: Maximum Subarray Sum (Kadane's Algorithm)

**Problem:** Given an integer array (may contain negatives), find the contiguous subarray with the largest sum.

**Example:** `[-2, 1, -3, 4, -1, 2, 1, -5, 4]` → Answer: `6` (subarray `[4, -1, 2, 1]`)

**Thought process in an interview:**

> "Let me start with brute force: try all O(n²) subarrays, compute each sum in O(n) → O(n³) total. Can we do better?"
>
> "Prefix sums reduce it to O(n²): precompute prefix, then sum(l,r) = prefix[r] - prefix[l-1]. Try all pairs."
>
> "Can we go O(n)? Key insight: at each index i, the best subarray ending at i is either (1) just arr[i] alone, or (2) extend the best subarray ending at i-1. So: `maxEndingHere = max(arr[i], maxEndingHere + arr[i])`."

```cpp
// Time: O(n) | Space: O(1)
// Returns the maximum sum AND the subarray indices
pair<int,int> kadane(int arr[], int n) {
    int maxSum = arr[0];
    int curSum = arr[0];
    int start = 0, end = 0, tempStart = 0;

    for (int i = 1; i < n; i++) {
        if (arr[i] > curSum + arr[i]) {
            curSum = arr[i];
            tempStart = i;                 // new subarray starts here
        } else {
            curSum += arr[i];
        }

        if (curSum > maxSum) {
            maxSum = curSum;
            start = tempStart;
            end = i;
        }
    }
    // maxSum is the answer; arr[start..end] is the subarray
    return {start, end};
}

// Simpler version (just the sum):
int maxSubarraySum(int arr[], int n) {
    int maxSoFar = arr[0], cur = arr[0];
    for (int i = 1; i < n; i++) {
        cur = max(arr[i], cur + arr[i]);
        maxSoFar = max(maxSoFar, cur);
    }
    return maxSoFar;
}
```

**Edge cases to mention:**
- All negatives → answer is the single largest element (NOT zero!)
- Single element array → return it
- All positives → sum of entire array

**Pattern:** This is the most fundamental DP-on-array problem. The recurrence `dp[i] = max(arr[i], dp[i-1] + arr[i])` appears in many forms. Recognise it.

---

### Problem 2: Two Sum (and its variants)

**Problem:** Given an array and a target, find two indices `i, j` such that `arr[i] + arr[j] == target`.

**Thought process:**

> "Brute force: try every pair → O(n²). Can we reduce to O(n)?"
>
> "For each arr[i], I need to find `target - arr[i]` in the array. If I use a hash map, I can look that up in O(1). Store `(value → index)` as I go."

```cpp
// Approach 1: Hash Map — O(n) time, O(n) space
// Works on unsorted arrays. Most general solution.
pair<int,int> twoSum(int arr[], int n, int target) {
    unordered_map<int,int> seen;  // value → index
    for (int i = 0; i < n; i++) {
        int complement = target - arr[i];
        if (seen.count(complement)) {
            return {seen[complement], i};
        }
        seen[arr[i]] = i;
    }
    return {-1, -1};  // no solution
}

// Approach 2: Two Pointers — O(n log n) time, O(1) space
// Requires sorted array (or sorting first)
pair<int,int> twoSumSorted(int arr[], int n, int target) {
    int left = 0, right = n - 1;
    while (left < right) {
        int sum = arr[left] + arr[right];
        if      (sum == target) return {left, right};
        else if (sum <  target) left++;
        else                     right--;
    }
    return {-1, -1};
}

// ── Variant: Three Sum (find all triplets summing to 0) ───────────────────────
// Fix one element, run two-pointer on the rest
// Time: O(n²) | Space: O(1) (excluding output)
vector<vector<int>> threeSum(vector<int>& nums) {
    sort(nums.begin(), nums.end());
    vector<vector<int>> result;
    int n = nums.size();

    for (int i = 0; i < n - 2; i++) {
        if (i > 0 && nums[i] == nums[i-1]) continue;  // skip duplicates!

        int left = i + 1, right = n - 1;
        while (left < right) {
            int sum = nums[i] + nums[left] + nums[right];
            if (sum == 0) {
                result.push_back({nums[i], nums[left], nums[right]});
                while (left < right && nums[left]  == nums[left+1])  left++;  // skip dups
                while (left < right && nums[right] == nums[right-1]) right--; // skip dups
                left++; right--;
            } else if (sum < 0) left++;
            else                 right--;
        }
    }
    return result;
}
```

**Key insight to communicate in interviews:** The two-pointer approach only works on a sorted array. The hash map approach is O(n) and works on anything. Know when to use which.

---

### Problem 3: Find Missing Number / Duplicate (In-Place Trick)

**Problem:** Array of size n contains numbers from 1 to n with one number missing. Find it in O(n) time and O(1) space.

**Multiple approaches — present all of them:**

```cpp
// ── Approach 1: Math (sum formula) ───────────────────────────────────────────
// Time: O(n) | Space: O(1)
// Sum of 1..n = n*(n+1)/2. Missing = expected_sum - actual_sum.
// PITFALL: can overflow for large n. Use long long.
int missingMath(int arr[], int n) {
    long long expected = (long long)n * (n + 1) / 2;
    long long actual   = 0;
    for (int i = 0; i < n - 1; i++) actual += arr[i];
    return (int)(expected - actual);
}

// ── Approach 2: XOR ──────────────────────────────────────────────────────────
// Time: O(n) | Space: O(1)
// XOR all indices 1..n with all arr values → missing number survives
// Key property: a XOR a = 0, a XOR 0 = a
int missingXOR(int arr[], int n) {
    int xorAll = 0;
    for (int i = 1; i <= n; i++)       xorAll ^= i;       // XOR 1..n
    for (int i = 0; i < n - 1; i++)    xorAll ^= arr[i];  // XOR array
    return xorAll;  // remaining value = missing number
}

// ── Approach 3: Cyclic Sort (index-based placement) ───────────────────────────
// Time: O(n) | Space: O(1) — modifies the array
// Place each number at its correct index (num 3 → index 2)
int missingCyclicSort(int arr[], int n) {
    // Sort: put arr[i] at index arr[i]-1
    for (int i = 0; i < n; i++) {
        while (arr[i] != i + 1 && arr[i] <= n) {
            swap(arr[i], arr[arr[i] - 1]);
        }
    }
    // Find the index where arr[i] != i+1
    for (int i = 0; i < n; i++) {
        if (arr[i] != i + 1) return i + 1;
    }
    return n;
}

// ── Extension: Find the duplicate (same technique) ────────────────────────────
// Array has n+1 elements, values 1..n, one value appears twice
// Floyd's cycle detection — no array modification, O(1) space
int findDuplicate(vector<int>& nums) {
    // Treat array as a linked list: index → next = nums[index]
    // A duplicate creates a cycle. Find cycle entry = duplicate.
    int slow = nums[0], fast = nums[0];
    do {
        slow = nums[slow];
        fast = nums[nums[fast]];
    } while (slow != fast);

    slow = nums[0];  // reset slow to start
    while (slow != fast) {
        slow = nums[slow];
        fast = nums[fast];
    }
    return slow;  // cycle entry = duplicate
}
```

**Interview tip:** When asked "O(1) space and O(n) time", your options are usually: math (sum/XOR), cyclic sort, or in-place sign-marking. Know all three. The interviewer is testing whether you know more than one trick.

---

## 8. Real-World Uses

| Domain | Where Arrays Are Used | Why Array Specifically |
|---|---|---|
| Image processing | Pixel buffers (RGB/RGBA values) | Direct index → O(1) pixel access by (x,y) coordinate |
| Audio processing | PCM audio samples | Sequential access with minimal overhead |
| GPU/SIMD | All vectorised computation | CPU can process 4/8/16 elements in one instruction |
| Database engines | Column-store databases (e.g. DuckDB) | Column = contiguous array → cache-efficient analytics |
| Operating systems | Page tables (physical memory mapping) | Direct index from page number → frame number |
| Networking | Packet buffers, ring buffers in NIC drivers | Lock-free circular array for producer-consumer |
| Compilers | Stack frames, bytecode instruction arrays | Fixed layout, offset-addressable |
| Competitive programming | Everything | Fastest constant factor; STL vector is just a dynamic array |

**Insight for system design interviews:** When asked to design a high-throughput system (message queue, time-series DB, log ingestion), think arrays first. Sequential writes to a memory-mapped array (or file) are the fastest possible I/O pattern — it's how Kafka's log storage works.

---

## 9. Edge Cases & Pitfalls

These are the things that cause bugs in real code and lost points in interviews.

### Pitfall 1: Off-by-One Errors

```cpp
int arr[5] = {1, 2, 3, 4, 5};

// WRONG: reading past the end
for (int i = 0; i <= 5; i++)   // should be i < 5
    cout << arr[i];             // arr[5] is undefined behaviour!

// WRONG: missing the last element
for (int i = 0; i < 4; i++)    // should be i < 5
    cout << arr[i];             // misses arr[4]

// Correct binary search bounds:
int lo = 0, hi = n - 1;        // hi is the last valid index, NOT n
```

### Pitfall 2: Integer Overflow in Index/Sum Calculations

```cpp
// WRONG — can overflow when lo and hi are both large
int mid = (lo + hi) / 2;

// CORRECT — equivalent, but never overflows
int mid = lo + (hi - lo) / 2;

// Sum of large array — use long long accumulator
long long sum = 0;
for (int i = 0; i < n; i++) sum += arr[i];  // not int sum!
```

### Pitfall 3: Uninitialised Arrays

```cpp
int arr[5];           // on the stack — contains GARBAGE (random values)
int arr[5] = {};      // zero-initialised — all zeros
int arr[5] = {0};     // also zero-initialised

// Heap:
int* p = new int[5];     // uninitialised
int* p = new int[5]();   // zero-initialised (the () matters!)
```

### Pitfall 4: Array Decay in Function Parameters

```cpp
// Arrays decay to pointers when passed to functions.
// sizeof DOES NOT work as expected inside the function.
void wrong(int arr[]) {
    int size = sizeof(arr) / sizeof(arr[0]);  // WRONG! sizeof(arr) = sizeof(pointer) = 8
}

// CORRECT: always pass size explicitly
void correct(int arr[], int n) { /* use n */ }

// Or use std::array which carries its size:
void better(array<int, 5>& arr) { /* arr.size() works */ }
```

### Pitfall 5: Out-of-Bounds Access (Undefined Behaviour)

```cpp
int arr[5] = {1, 2, 3, 4, 5};
cout << arr[-1];   // UB — may crash, may read garbage, may corrupt memory
cout << arr[5];    // UB — same issue, off the end
cout << arr[100];  // UB — likely segfault

// Use arr.at(i) from std::array for bounds-checked access in debug builds
// Use sanitizers: compile with -fsanitize=address,undefined during testing
```

### Pitfall 6: Modifying Array While Iterating

```cpp
// Removing elements while iterating → skips elements
for (int i = 0; i < n; i++) {
    if (arr[i] == 0) deleteAt(arr, n, i);  // WRONG: i++ skips the next element
}

// CORRECT: iterate backwards, or use slow/fast pointer pattern
for (int i = n - 1; i >= 0; i--) {
    if (arr[i] == 0) deleteAt(arr, n, i);
}
```

### Pitfall 7: Stack Overflow with Large Arrays

```cpp
// This will crash (stack is typically 1-8 MB)
int huge[10000000];   // 40 MB on stack → STACK OVERFLOW

// CORRECT: allocate on heap
int* huge = new int[10000000]();
// Or use vector: vector<int> huge(10000000, 0);
```

---

## 10. Comparison with Dynamic Array (std::vector)

| Feature | Static Array `int arr[N]` | Dynamic Array `std::vector<int>` |
|---|---|---|
| Size | Fixed at compile time | Grows at runtime |
| Memory location | Stack (or static segment) | Heap |
| Overhead | Zero | ~3 pointers (begin, end, capacity) |
| Access time | O(1), no indirection | O(1), one pointer dereference |
| Random access speed | Slightly faster (no indirection) | Nearly identical in practice |
| Cache behaviour | Identical — both contiguous | Identical — both contiguous |
| Bounds checking | None by default | `at()` throws, `[]` doesn't |
| Passing to functions | Decays to pointer (loses size) | Passed by reference, size preserved |
| When to use | Size known at compile time, stack is fine | Size dynamic, or size > ~1MB |

**Recommendation:** In modern C++, prefer `std::array<int, N>` over raw `int arr[N]`. It has zero runtime overhead, carries its size, works with all STL algorithms, and doesn't decay to a pointer.

---

## 11. Self-Test Questions

Test your understanding before moving on. These are the kinds of questions interviewers ask to probe depth.

1. **Why is `arr[i]` O(1) and not O(n)?** Explain using the address formula.

2. **What is the difference between these two?**
   ```cpp
   int arr[5] = {1};
   int arr[5];
   ```

3. **Why should you write `lo + (hi - lo) / 2` instead of `(lo + hi) / 2`?**

4. **You have a 1000×1000 integer matrix. Would you iterate row-first or column-first for better performance? Why?**

5. **Write Kadane's algorithm from scratch in 5 minutes. What is the key recurrence?**

6. **A function receives `int arr[]` as a parameter. Can you find its length using `sizeof`? Why or why not?**

7. **In Two Sum, when would you choose the hash map approach over the two-pointer approach?**

8. **What is undefined behaviour in C++ and why is `arr[n]` dangerous even if it doesn't crash?**

9. **Explain the three-reverse trick for rotating an array. Why does it work?**

10. **Design a system to answer 10⁶ range sum queries on an array of 10⁶ elements. What is the optimal preprocessing and query time?**

---

## Quick Reference Card

```
Static Array — O(1) access is the entire point.

  Access:   O(1)     ← direct address computation
  Search:   O(n) unsorted | O(log n) sorted
  Insert:   O(1) end | O(n) middle (shift)
  Delete:   O(1) end | O(n) middle (shift)
  Space:    O(n)

Key patterns to master:
  1. Two pointers        — eliminate O(n²) to O(n)
  2. Sliding window      — subarray problems
  3. Prefix sum          — O(1) range queries
  4. In-place indexing   — O(1) space tricks
  5. Binary search       — O(log n) on sorted arrays

Always ask: sorted or unsorted? What are the value constraints?
```

---

*Next in curriculum: [02 — Dynamic Array (std::vector)](./02_dynamic_array.md)*  
*See also: [Prefix Sum](./05_fenwick_tree.md) | [Binary Search](./algorithms/binary_search.md)*

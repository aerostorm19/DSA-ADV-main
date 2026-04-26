# Priority Queue

> **Curriculum position:** Linear Structures → #13 (Final linear structure before hash-based)
> **Interview weight:** ★★★★★ Critical — Dijkstra, A\*, Huffman, top-K, median maintenance, event simulation all run on priority queues
> **Difficulty:** Intermediate — the heap internals require care; the applications span nearly every problem domain
> **Prerequisite:** [07 — Stack](./07_stack.md) | [09 — Queue](./09_queue.md) | [Binary Heap (upcoming)](./heaps/binary_heap.md)

---

## Table of Contents

1. [Intuition First — Not All Elements Are Equal](#1-intuition-first)
2. [The Priority Queue ADT Contract](#2-the-priority-queue-adt-contract)
3. [Internal Working — The Binary Heap Engine](#3-internal-working)
4. [Heap Operations — Sift Up and Sift Down](#4-heap-operations)
5. [Time & Space Complexity](#5-time--space-complexity)
6. [Complete C++ Implementation](#6-complete-c-implementation)
7. [Core Operations — Visualised](#7-core-operations--visualised)
8. [Common Patterns & Techniques](#8-common-patterns--techniques)
9. [Interview Problems](#9-interview-problems)
10. [Real-World Uses](#10-real-world-uses)
11. [Edge Cases & Pitfalls](#11-edge-cases--pitfalls)
12. [std::priority\_queue — Complete API Reference](#12-stdpriority_queue)
13. [Priority Queue Variants](#13-priority-queue-variants)
14. [Comparison: Priority Queue vs Sorted Array vs BST vs Monotonic Deque](#14-comparison)
15. [Self-Test Questions](#15-self-test-questions)

---

## 1. Intuition First

An ordinary queue serves elements in arrival order — the oldest request is processed first regardless of its importance. This is fair but not always efficient. A hospital emergency room cannot serve patients by arrival time if someone is having a cardiac arrest while a bruised knee waits from three hours ago.

The **priority queue** breaks FIFO. Instead of ordering by arrival, it always serves the element with the highest priority — or equivalently, with the smallest (min-heap) or largest (max-heap) key value.

Think of a hospital triage system:
- Patients arrive continuously and are assigned a severity score.
- The most severely injured patient is always treated next, regardless of when they arrived.
- New critical patients can be added at any time and immediately move to the front.
- After treating a patient, the next most critical is instantly identified.

This is a **max-priority queue** (highest priority number = served first). A **min-priority queue** serves the element with the smallest key — used in Dijkstra's algorithm where the node with the smallest tentative distance is always processed next.

Three properties make a priority queue different from everything studied so far:

1. **Order by value, not position** — elements have no fixed "place"; their effective position is determined by their key.
2. **Dynamic** — new elements arrive continuously, and the priority ordering is maintained automatically.
3. **Efficient extremum** — finding and removing the min/max is the core operation, and it must be fast.

The priority queue is implemented almost universally with a **binary heap** — a complete binary tree stored in an array that maintains the heap property in O(log n) time per insertion or deletion, with O(1) access to the minimum or maximum.

---

## 2. The Priority Queue ADT Contract

```
push(x)     / insert(x)    — add element x with its priority       O(log n)
pop()       / extract()    — remove and return highest-priority element  O(log n)
top()       / peek()       — return highest-priority without removing    O(1)
empty()                    — true if no elements                         O(1)
size()                     — number of elements                          O(1)
```

Two variants:
- **Max-priority queue**: `top()` returns the LARGEST element. Used when "highest" priority means greatest value.
- **Min-priority queue**: `top()` returns the SMALLEST element. Used in Dijkstra (smallest distance), A\* (smallest f-score), Huffman coding (least frequent character).

**C++ default**: `std::priority_queue` is a **max-heap** by default. To get a min-heap, use `std::priority_queue<int, vector<int>, greater<int>>`.

---

## 3. Internal Working — The Binary Heap Engine

The priority queue is almost always implemented as a **binary heap**. Understanding the heap is understanding the priority queue.

### The Binary Heap

A binary heap is a **complete binary tree** (all levels filled except possibly the last, which is filled left to right) satisfying the **heap property**:

- **Max-heap**: every node's value ≥ its children's values. Root = maximum.
- **Min-heap**: every node's value ≤ its children's values. Root = minimum.

```
Max-heap example:
           100
          /    \
        80      70
       /  \    /  \
      50   60 30   40
     /  \
    20   10

Heap property: every parent ≥ both children. Root (100) = maximum. ✓
```

### The Array Representation — The Key Insight

A complete binary tree can be stored in an array with **zero wasted space** using index arithmetic:

```
Array: [100, 80, 70, 50, 60, 30, 40, 20, 10]
Index:    0   1   2   3   4   5   6   7   8

For node at index i (0-based):
  Parent:       (i - 1) / 2       = floor((i-1)/2)
  Left child:   2*i + 1
  Right child:  2*i + 2

Verify:
  Node 80 (i=1): parent=(1-1)/2=0 → 100 ✓; left=3 → 50 ✓; right=4 → 60 ✓
  Node 50 (i=3): parent=(3-1)/2=1 → 80 ✓; left=7 → 20 ✓; right=8 → 10 ✓
  Node 40 (i=6): parent=(6-1)/2=2 → 70 ✓; left=13 → none ✓; right=14 → none ✓

Visual mapping (array → tree):
  idx 0             → root (100)
  idx 1, 2          → level 1 (80, 70)
  idx 3, 4, 5, 6    → level 2 (50, 60, 30, 40)
  idx 7, 8          → level 3 (20, 10)
```

This is why heaps are stored in arrays and not as pointer-based trees:
- **Zero pointer overhead**: no `left`, `right`, or `parent` pointers — pure arithmetic.
- **Cache-friendly**: the entire tree is one contiguous block of memory.
- **Implicit structure**: the "shape" (complete binary tree) is maintained by keeping the array contiguous.

---

## 4. Heap Operations — Sift Up and Sift Down

Every heap operation reduces to one of two primitives:

### Sift Up (Bubble Up) — Used After push()

When a new element is added to the end of the array, it may violate the heap property with its parent. Fix by repeatedly swapping with the parent until the heap property is restored.

```
Insert 90 into the max-heap:
Before: [100, 80, 70, 50, 60, 30, 40, 20, 10]

Step 1: Append 90 at index 9.
  [100, 80, 70, 50, 60, 30, 40, 20, 10, 90]
  Parent of 9 = (9-1)/2 = 4 → value 60. 90 > 60 → swap.

Step 2: 90 now at index 4. Parent of 4 = (4-1)/2 = 1 → value 80. 90 > 80 → swap.
  [100, 90, 70, 50, 80, 30, 40, 20, 10, 60]

Step 3: 90 now at index 1. Parent of 1 = (1-1)/2 = 0 → value 100. 90 < 100 → STOP.
  [100, 90, 70, 50, 80, 30, 40, 20, 10, 60]  ← heap property restored ✓

Sift up path: idx 9 → 4 → 1. Height of tree = log n. Max steps = O(log n).
```

### Sift Down (Bubble Down) — Used After pop()

When the root is removed, replace it with the last element (to maintain the complete tree shape) and sift it down to its correct position by swapping with the larger child (max-heap) or smaller child (min-heap).

```
Remove root (100) from [100, 90, 70, 50, 80, 30, 40, 20, 10, 60]:

Step 1: Replace root with last element (60). Remove last slot.
  [60, 90, 70, 50, 80, 30, 40, 20, 10]

Step 2: Sift down from root (idx 0, value 60).
  Children: left=idx 1 (90), right=idx 2 (70). Max child = 90 at idx 1.
  60 < 90 → swap.
  [90, 60, 70, 50, 80, 30, 40, 20, 10]

Step 3: 60 now at idx 1.
  Children: left=idx 3 (50), right=idx 4 (80). Max child = 80 at idx 4.
  60 < 80 → swap.
  [90, 80, 70, 50, 60, 30, 40, 20, 10]

Step 4: 60 now at idx 4.
  Children: left=idx 9 (out of bounds), right=idx 10 (out of bounds). No children → STOP.
  [90, 80, 70, 50, 60, 30, 40, 20, 10]  ← heap property restored ✓

Sift down: O(log n) — at most the height of the tree.
```

### Build Heap — O(n) Not O(n log n)

Given an arbitrary array, you can build a heap in **O(n)** by sifting down from the last non-leaf node to the root. This is counterintuitive but provably correct.

```cpp
// Build heap from an arbitrary array: O(n)
void buildHeap(vector<int>& arr) {
    int n = arr.size();
    // Start from last non-leaf: index (n/2 - 1)
    // Leaves (indices n/2 to n-1) are already trivially valid heaps.
    for (int i = n / 2 - 1; i >= 0; i--) {
        siftDown(arr, n, i);
    }
}
// Why O(n) and not O(n log n)?
// Nodes at depth d require at most (height - d) sift-down steps.
// Most nodes are near the leaves (depth ≈ log n) and require few steps.
// Summing across all nodes: Σ k * n/2^k = O(n). (geometric series)
```

---

## 5. Time & Space Complexity

### Operation Complexities

| Operation | Binary Heap (array) | Sorted Array | Balanced BST | Fibonacci Heap |
|---|---|---|---|---|
| `push(x)` | **O(log n)** | O(n) | O(log n) | **O(1) amortized** |
| `pop()` | **O(log n)** | O(1) | O(log n) | **O(log n) amortized** |
| `top()` | **O(1)** | O(1) | O(log n) | **O(1)** |
| `build` from n elements | **O(n)** | O(n log n) | O(n log n) | O(n) |
| `decrease_key` | O(log n) | O(n) | O(log n) | **O(1) amortized** |
| Search | O(n) | O(log n) | O(log n) | O(n) |
| Space | O(n) | O(n) | O(n) + pointers | O(n) |

**Why Fibonacci Heap is rarely used despite theoretical superiority:**
- `decrease_key` in O(1) amortized makes Dijkstra's algorithm O(E + V log V) instead of O((E+V) log V).
- In practice, the constant factors are enormous and the implementation is notoriously complex.
- For most real workloads, the binary heap's cache-friendliness wins.
- `std::priority_queue` uses a binary heap.

### Space Complexity

| | |
|---|---|
| n elements | O(n) — one contiguous array |
| Per-element overhead | 0 bytes — no pointers |
| Auxiliary (heap ops) | O(1) — in-place sift operations |
| vs pointer-based BST | Heap: n × sizeof(T); BST: n × (sizeof(T) + 3 pointers) |

---

## 6. Complete C++ Implementation

### Implementation 1: Max-Heap from Scratch

```cpp
#include <iostream>
#include <vector>
#include <stdexcept>
#include <functional>
using namespace std;

template<typename T, typename Compare = less<T>>
class PriorityQueue {
    // Default Compare = less<T> → max-heap (largest element at top)
    // Use greater<T>           → min-heap (smallest element at top)
private:
    vector<T> heap_;
    Compare   cmp_;   // cmp_(a, b) = true if b has higher priority than a
                      // For max-heap (less<T>): cmp_(a,b) means a < b → b wins
                      // For min-heap (greater<T>): cmp_(a,b) means a > b → b wins

    int parent(int i)     const { return (i - 1) / 2; }
    int leftChild(int i)  const { return 2 * i + 1; }
    int rightChild(int i) const { return 2 * i + 2; }

    // Sift element at index i upward until heap property is restored
    void siftUp(int i) {
        while (i > 0) {
            int p = parent(i);
            // If current node has higher priority than parent → swap
            if (cmp_(heap_[p], heap_[i])) {
                swap(heap_[p], heap_[i]);
                i = p;
            } else {
                break;
            }
        }
    }

    // Sift element at index i downward until heap property is restored
    void siftDown(int i) {
        int n = (int)heap_.size();
        while (true) {
            int best = i;          // index of highest-priority among i and its children
            int left  = leftChild(i);
            int right = rightChild(i);

            if (left  < n && cmp_(heap_[best], heap_[left]))  best = left;
            if (right < n && cmp_(heap_[best], heap_[right])) best = right;

            if (best == i) break;  // heap property satisfied

            swap(heap_[i], heap_[best]);
            i = best;
        }
    }

public:
    // ── Constructors ──────────────────────────────────────────────────────────

    PriorityQueue() = default;

    // Build from a range in O(n) using Floyd's heap construction
    explicit PriorityQueue(vector<T> elems, Compare cmp = Compare{})
        : heap_(move(elems)), cmp_(cmp) {
        // Sift down from last non-leaf to root
        for (int i = (int)heap_.size() / 2 - 1; i >= 0; i--) {
            siftDown(i);
        }
    }

    // ── Core Operations ───────────────────────────────────────────────────────

    // O(log n) — add element
    void push(const T& val) {
        heap_.push_back(val);
        siftUp((int)heap_.size() - 1);
    }

    void push(T&& val) {
        heap_.push_back(move(val));
        siftUp((int)heap_.size() - 1);
    }

    template<typename... Args>
    void emplace(Args&&... args) {
        heap_.emplace_back(forward<Args>(args)...);
        siftUp((int)heap_.size() - 1);
    }

    // O(log n) — remove highest-priority element
    void pop() {
        if (heap_.empty()) throw underflow_error("pop on empty priority queue");
        // Replace root with last element, remove last, sift down root
        heap_[0] = move(heap_.back());
        heap_.pop_back();
        if (!heap_.empty()) siftDown(0);
    }

    // O(1) — peek at highest-priority element
    const T& top() const {
        if (heap_.empty()) throw underflow_error("top on empty priority queue");
        return heap_[0];
    }

    // O(1)
    bool   empty() const { return heap_.empty(); }
    size_t size()  const { return heap_.size(); }

    // O(n) — expose underlying array (for inspection/custom algorithms)
    const vector<T>& data() const { return heap_; }

    // O(log n) — decrease key for min-heap / increase key for max-heap
    // Only safe if you know the index (not exposed in std::priority_queue)
    void update(int i, const T& new_val) {
        heap_[i] = new_val;
        siftUp(i);
        siftDown(i);
    }

    void print() const {
        if (heap_.empty()) { cout << "(empty priority queue)\n"; return; }
        cout << "heap array: [";
        for (int i = 0; i < (int)heap_.size(); i++) {
            cout << heap_[i] << (i < (int)heap_.size()-1 ? ", " : "");
        }
        cout << "]  top=" << heap_[0] << "\n";
    }
};
```

### Implementation 2: std::priority_queue Patterns

```cpp
#include <queue>
#include <vector>
#include <functional>
using namespace std;

// ── Basic usage ───────────────────────────────────────────────────────────────

// MAX heap (default)
priority_queue<int> maxPQ;
maxPQ.push(3); maxPQ.push(1); maxPQ.push(4); maxPQ.push(1); maxPQ.push(5);
cout << maxPQ.top(); // 5

// MIN heap
priority_queue<int, vector<int>, greater<int>> minPQ;
minPQ.push(3); minPQ.push(1); minPQ.push(4);
cout << minPQ.top(); // 1

// ── Custom comparator for pairs/structs ───────────────────────────────────────

// Min-heap of pairs sorted by first element (distance in Dijkstra)
priority_queue<pair<int,int>,
               vector<pair<int,int>>,
               greater<pair<int,int>>> dijkstraPQ;
// pair<dist, node> — smaller dist = higher priority
dijkstraPQ.push({10, 0});
dijkstraPQ.push({5,  1});
dijkstraPQ.push({15, 2});
cout << dijkstraPQ.top().second; // 1 (node with smallest distance = 5)

// Custom struct with lambda comparator
struct Task {
    int priority;
    string name;
};
auto cmp = [](const Task& a, const Task& b) {
    return a.priority < b.priority;  // LESS means lower priority → max-heap by priority
};
priority_queue<Task, vector<Task>, decltype(cmp)> taskPQ(cmp);
taskPQ.push({3, "low"});
taskPQ.push({10, "critical"});
taskPQ.push({7, "medium"});
cout << taskPQ.top().name; // "critical" (priority=10, highest)

// ── Build from vector in O(n) ─────────────────────────────────────────────────
vector<int> nums = {3, 1, 4, 1, 5, 9, 2, 6};
// Method 1: construct directly — uses O(n) heap construction internally
priority_queue<int> pq1(nums.begin(), nums.end());

// Method 2: construct with custom comparator
priority_queue<int, vector<int>, greater<int>> pq2(nums.begin(), nums.end());

// ── Pop all elements in sorted order ─────────────────────────────────────────
// Max-heap gives elements in descending order
while (!maxPQ.empty()) {
    cout << maxPQ.top() << " ";  // 5 4 3 1 1
    maxPQ.pop();
}

// ── Heap operations NOT in std::priority_queue ────────────────────────────────
// std::priority_queue does NOT support:
//   - Random access / iteration
//   - decrease_key / increase_key
//   - Merging two heaps
//   - Access by index
// For these, use the raw heap algorithms:
//   std::make_heap, std::push_heap, std::pop_heap on a vector
```

### Implementation 3: Lazy Deletion Priority Queue

```cpp
// When you need to remove arbitrary elements (not just the top),
// "lazy deletion" marks elements as deleted and skips them on pop.
// Used in Dijkstra when edge relaxations make old entries obsolete.

template<typename T, typename Compare = less<T>>
class LazyPQ {
private:
    priority_queue<T, vector<T>, Compare> pq_;
    unordered_set<T>                      deleted_;

public:
    void push(const T& val) { pq_.push(val); }

    void remove(const T& val) { deleted_.insert(val); }  // O(1) mark

    void pop() {
        // Skip deleted elements from the top
        while (!pq_.empty() && deleted_.count(pq_.top())) {
            deleted_.erase(pq_.top());
            pq_.pop();
        }
        if (pq_.empty()) throw underflow_error("empty");
        pq_.pop();
    }

    const T& top() {
        while (!pq_.empty() && deleted_.count(pq_.top())) {
            deleted_.erase(pq_.top());
            pq_.pop();
        }
        if (pq_.empty()) throw underflow_error("empty");
        return pq_.top();
    }

    bool empty() {
        while (!pq_.empty() && deleted_.count(pq_.top())) {
            deleted_.erase(pq_.top());
            pq_.pop();
        }
        return pq_.empty();
    }
};
```

---

## 7. Core Operations — Visualised

### The Heap as a Tree and an Array — Side by Side

```
Max-heap after push(10), push(20), push(15), push(30), push(5), push(25):

Insertion order: 10, 20, 15, 30, 5, 25
After each push + sift-up:

push(10):  [10]
push(20):  [10, 20] → sift up: 20>10 → swap → [20, 10]
push(15):  [20, 10, 15] → sift up: 15>20? No → [20, 10, 15]
push(30):  [20, 10, 15, 30] → sift up: 30>10 → swap → [20, 30, 15, 10]
                                           30>20 → swap → [30, 20, 15, 10]
push(5):   [30, 20, 15, 10, 5] → sift up: 5>20? No → [30, 20, 15, 10, 5]
push(25):  [30, 20, 15, 10, 5, 25] → sift up: 25>15 → swap → [30, 20, 25, 10, 5, 15]
                                                25>30? No → [30, 20, 25, 10, 5, 15]

Final tree:
           30
          /  \
        20    25
       /  \  /
      10   5 15

Array: [30, 20, 25, 10, 5, 15]
Index:   0   1   2   3  4   5

Verify parent relationships:
  15 (idx 5): parent=(5-1)/2=2 → 25 ✓ (25 ≥ 15)
  10 (idx 3): parent=(3-1)/2=1 → 20 ✓ (20 ≥ 10)
  5  (idx 4): parent=(4-1)/2=1 → 20 ✓ (20 ≥ 5)
```

### Pop — Replace Root, Sift Down

```
Pop from [30, 20, 25, 10, 5, 15]:

Step 1: Save root (30). Move last element (15) to root. Remove last slot.
  [15, 20, 25, 10, 5]

Step 2: Sift down from root (idx 0, value 15).
  Left child = idx 1 (20), right child = idx 2 (25).
  Max child = idx 2 (25). 15 < 25 → swap.
  [25, 20, 15, 10, 5]

Step 3: 15 now at idx 2.
  Left child = idx 5 (out of bounds). No children → STOP.
  [25, 20, 15, 10, 5]

Tree after pop:
           25
          /  \
        20    15
       /  \
      10   5

Returned: 30. Remaining top: 25. ✓
```

### Build Heap in O(n) — Floyd's Algorithm

```
Input array: [4, 10, 3, 5, 1, 8, 7, 2, 9, 6]
n=10, last non-leaf = n/2-1 = 4

Start sifting down from index 4 toward 0:

i=4: value=1. Children: idx 9 (6). 1 < 6 → swap.
  [4, 10, 3, 5, 6, 8, 7, 2, 9, 1]

i=3: value=5. Children: idx 7 (2), idx 8 (9). Max=9 at idx 8. 5 < 9 → swap.
  [4, 10, 3, 9, 6, 8, 7, 2, 5, 1]

i=2: value=3. Children: idx 5 (8), idx 6 (7). Max=8. 3 < 8 → swap.
  [4, 10, 8, 9, 6, 3, 7, 2, 5, 1]
  3 now at idx 5. No children. Stop.

i=1: value=10. Children: idx 3 (9), idx 4 (6). Max=9. 10 > 9 → STOP. (already satisfied)

i=0: value=4. Children: idx 1 (10), idx 2 (8). Max=10. 4 < 10 → swap.
  [10, 4, 8, 9, 6, 3, 7, 2, 5, 1]
  4 now at idx 1. Children: idx 3 (9), idx 4 (6). Max=9. 4 < 9 → swap.
  [10, 9, 8, 4, 6, 3, 7, 2, 5, 1]
  4 now at idx 3. Children: idx 7 (2), idx 8 (5). Max=5 at idx 8. 4 < 5 → swap.
  [10, 9, 8, 5, 6, 3, 7, 2, 4, 1]
  4 now at idx 8. No children. Stop.

Final: [10, 9, 8, 5, 6, 3, 7, 2, 4, 1]
           10
          /  \
        9      8
       / \    / \
      5   6  3   7
     / \ /
    2  4 1

Valid max-heap. ✓  Built in O(n).
```

---

## 8. Common Patterns & Techniques

### Pattern 1: Top-K Elements

Find the K largest (or smallest) elements in an array or stream.

```cpp
// K LARGEST elements — use a MIN-heap of size k
// Invariant: the heap always holds the k largest seen so far.
// When a new element is larger than the smallest in the heap, it displaces it.
// Time: O(n log k) | Space: O(k)
vector<int> topKLargest(vector<int>& nums, int k) {
    priority_queue<int, vector<int>, greater<int>> minHeap;  // min-heap

    for (int num : nums) {
        minHeap.push(num);
        if ((int)minHeap.size() > k) minHeap.pop();  // evict the smallest
    }

    vector<int> result;
    while (!minHeap.empty()) {
        result.push_back(minHeap.top());
        minHeap.pop();
    }
    return result;   // in ascending order (reverse for descending)
}

// K SMALLEST elements — use a MAX-heap of size k
vector<int> topKSmallest(vector<int>& nums, int k) {
    priority_queue<int> maxHeap;  // max-heap (default)

    for (int num : nums) {
        maxHeap.push(num);
        if ((int)maxHeap.size() > k) maxHeap.pop();  // evict the largest
    }

    vector<int> result;
    while (!maxHeap.empty()) {
        result.push_back(maxHeap.top());
        maxHeap.pop();
    }
    return result;
}

// Counter-intuitive but correct:
// To keep the K LARGEST: maintain a MIN-heap of size k.
//   The top (minimum) of this heap is the smallest of the k largest.
//   Any new element larger than this minimum displaces it.
// To keep the K SMALLEST: maintain a MAX-heap of size k.
//   The top (maximum) of this heap is the largest of the k smallest.
//   Any new element smaller than this maximum displaces it.
```

### Pattern 2: Merge K Sorted Lists/Arrays

```cpp
// Merge k sorted arrays using a min-heap.
// Each heap entry: (value, array_index, element_index_within_array)
// Always extract the global minimum; push the next element from the same array.
// Time: O(n log k) where n = total elements | Space: O(k)

vector<int> mergeKSorted(vector<vector<int>>& arrays) {
    using T = tuple<int,int,int>;  // (value, array_idx, pos_in_array)
    priority_queue<T, vector<T>, greater<T>> minHeap;

    // Seed with the first element of each array
    for (int i = 0; i < (int)arrays.size(); i++) {
        if (!arrays[i].empty()) {
            minHeap.push({arrays[i][0], i, 0});
        }
    }

    vector<int> result;
    while (!minHeap.empty()) {
        auto [val, arr, pos] = minHeap.top();
        minHeap.pop();
        result.push_back(val);

        // Push the next element from the same array, if available
        if (pos + 1 < (int)arrays[arr].size()) {
            minHeap.push({arrays[arr][pos + 1], arr, pos + 1});
        }
    }
    return result;
}
```

### Pattern 3: Dijkstra's Shortest Path (Priority Queue Core Algorithm)

```cpp
// Dijkstra: single-source shortest path on a weighted graph with non-negative edges.
// Min-heap of (distance, node). Always process the closest unvisited node.
// Time: O((V + E) log V) | Space: O(V)

vector<int> dijkstra(int src, int V, vector<vector<pair<int,int>>>& adj) {
    // adj[u] = {(v, weight), ...}
    vector<int> dist(V, INT_MAX);
    priority_queue<pair<int,int>,
                   vector<pair<int,int>>,
                   greater<pair<int,int>>> minHeap;  // (dist, node)

    dist[src] = 0;
    minHeap.push({0, src});

    while (!minHeap.empty()) {
        auto [d, u] = minHeap.top();
        minHeap.pop();

        // Stale entry: we already found a shorter path to u
        if (d > dist[u]) continue;

        for (auto [v, w] : adj[u]) {
            if (dist[u] + w < dist[v]) {
                dist[v] = dist[u] + w;
                minHeap.push({dist[v], v});  // push updated distance
                // Old (larger) distance for v remains in heap → lazy deletion
            }
        }
    }
    return dist;
}
// Note: we use lazy deletion — old stale entries for a node remain in the heap
// but are skipped when popped (if d > dist[u]).
// Alternative: decrease-key (Fibonacci heap) but lazy deletion is simpler and fast in practice.
```

### Pattern 4: Median Maintenance (Two Heaps)

Maintain the running median of a stream using a max-heap for the lower half and a min-heap for the upper half.

```cpp
class MedianFinder {
private:
    priority_queue<int>                              lowerHalf_;  // max-heap: left half
    priority_queue<int,vector<int>,greater<int>>     upperHalf_;  // min-heap: right half
    // Invariant:
    //   lowerHalf.top() <= upperHalf.top()  (lower half ≤ upper half)
    //   |lowerHalf| == |upperHalf| or |lowerHalf| == |upperHalf| + 1

public:
    void addNum(int num) {
        // Step 1: push to lower half first
        lowerHalf_.push(num);

        // Step 2: balance — ensure lowerHalf.top() <= upperHalf.top()
        if (!upperHalf_.empty() && lowerHalf_.top() > upperHalf_.top()) {
            upperHalf_.push(lowerHalf_.top());
            lowerHalf_.pop();
        }

        // Step 3: rebalance sizes — lower can have at most 1 more than upper
        if (lowerHalf_.size() > upperHalf_.size() + 1) {
            upperHalf_.push(lowerHalf_.top());
            lowerHalf_.pop();
        } else if (upperHalf_.size() > lowerHalf_.size()) {
            lowerHalf_.push(upperHalf_.top());
            upperHalf_.pop();
        }
    }

    double findMedian() const {
        if (lowerHalf_.size() > upperHalf_.size()) {
            return lowerHalf_.top();   // odd total: median is top of lower half
        }
        return (lowerHalf_.top() + upperHalf_.top()) / 2.0;  // even: average of two middles
    }
};

/*
Trace: addNum(1), addNum(2), addNum(3), addNum(4)

addNum(1): lower=[1], upper=[].  Sizes: 1,0. OK.  median=1.0
addNum(2): lower=[2,1], upper=[].
  lower.top()=2 > upper.top()?  upper empty, no.
  Size: 2 > 0+1=1. Move 2 to upper. lower=[1], upper=[2]. median=(1+2)/2=1.5
addNum(3): lower=[3,1], upper=[2].
  lower.top()=3 > upper.top()=2. Move 3 to upper. lower=[1], upper=[2,3].
  Size: 1 < 2. Move 2 from upper to lower. lower=[2,1], upper=[3]. median=2.0
addNum(4): lower=[4,2,1], upper=[3].
  lower.top()=4 > upper.top()=3. Move 4 to upper. lower=[2,1], upper=[3,4].
  Sizes equal. median=(2+3)/2=2.5
*/
```

---

## 9. Interview Problems

### Problem 1: K Closest Points to Origin

**Problem:** Given an array of points `(x, y)` and integer `k`, return the k points closest to the origin `(0,0)`. Distance metric: Euclidean squared (`x² + y²`).

**Thought process:**

> "Brute force: compute all distances, sort, take first k. O(n log n). Can we do O(n log k)?"
>
> "Top-K pattern: keep a MAX-heap of size k. The heap always holds the k closest seen so far. The top is the FARTHEST of the k closest. If a new point is closer than the heap top, it displaces it. At the end, heap contains the answer."

```cpp
// Time: O(n log k) | Space: O(k)
vector<vector<int>> kClosest(vector<vector<int>>& points, int k) {
    // Max-heap: top = farthest among k closest candidates
    // Comparator: if dist(a) < dist(b), then b has higher "priority" in max-heap
    auto cmp = [](const vector<int>& a, const vector<int>& b) {
        return a[0]*a[0] + a[1]*a[1] < b[0]*b[0] + b[1]*b[1];
    };
    priority_queue<vector<int>, vector<vector<int>>, decltype(cmp)> maxHeap(cmp);

    for (auto& point : points) {
        maxHeap.push(point);
        if ((int)maxHeap.size() > k) maxHeap.pop();  // evict the farthest
    }

    vector<vector<int>> result;
    while (!maxHeap.empty()) {
        result.push_back(maxHeap.top());
        maxHeap.pop();
    }
    return result;
}

/*
points = [[1,3],[-2,2],[5,8],[1,1]], k=2

Distances²: [1,3]→10, [-2,2]→8, [5,8]→89, [1,1]→2

Process [1,3]: heap=[[1,3]]. size=1≤2.
Process [-2,2]: heap=[[1,3],[-2,2]]. size=2≤2.
  heap top = [1,3] (dist 10, farthest of the 2).
Process [5,8]: heap=[[5,8],[1,3],[-2,2]]. size=3>2.
  Pop top = [5,8] (dist 89, farthest). heap=[[1,3],[-2,2]].
Process [1,1]: dist=2 < heap.top=[1,3] dist=10.
  heap=[[1,1],[1,3],[-2,2]]. size=3>2.
  Pop top = [1,3] (dist 10). heap=[[1,1],[-2,2]].

Result: {[-2,2], [1,1]}. Distances: 8, 2. ✓ (2 closest points)
*/
```

**Why not O(n log n) sort?** When n is large and k is small, O(n log k) is significantly faster. For n=10⁶ and k=10: heap approach does ~2×10⁶×20 = 4×10⁷ operations; sort does ~2×10⁶×20 = 4×10⁷ too — similar here. But for k=3 and n=10⁸: heap does ~8×10⁸ but sorting would need to handle 10⁸ elements; in streaming contexts the heap shines because it never materialises all elements.

**Edge cases:**
- k = n: return all points (heap grows to n and nothing is evicted)
- Multiple points at the same distance: any k of them is a valid answer
- k = 1: return the single closest point
- Points on the axes: `x²+y²` handles correctly without sqrt

---

### Problem 2: Find Median from Data Stream

**Problem:** Design a data structure that supports:
- `addNum(int num)` — add a number to the stream
- `findMedian()` — return the median of all numbers added so far

Both operations should be efficient.

**Thought process:**

> "If I maintain a sorted structure, insertion is O(log n) and median is O(1) (middle element). But maintaining perfect sorting is expensive."
>
> "Key insight: I don't need the full sorted order — I only need the MIDDLE elements. Partition the numbers into a lower half and an upper half. The median is determined by the top of each half. A MAX-heap for the lower half gives me the largest of the lower half in O(1). A MIN-heap for the upper half gives me the smallest of the upper half in O(1). The median is one or both of these tops."
>
> "Invariant: lower half max ≤ upper half min (always). Sizes: equal or lower is 1 larger (for odd total)."

```cpp
class MedianFinder {
private:
    priority_queue<int>                             lower_;  // max-heap (left half)
    priority_queue<int,vector<int>,greater<int>>    upper_;  // min-heap (right half)

public:
    // O(log n) — add number and rebalance
    void addNum(int num) {
        // Always insert into lower first
        lower_.push(num);

        // Ensure cross-half ordering: lower's max ≤ upper's min
        if (!upper_.empty() && lower_.top() > upper_.top()) {
            upper_.push(lower_.top());
            lower_.pop();
        }

        // Ensure size balance: |lower| - |upper| ∈ {0, 1}
        if (lower_.size() > upper_.size() + 1) {
            upper_.push(lower_.top());
            lower_.pop();
        } else if (upper_.size() > lower_.size()) {
            lower_.push(upper_.top());
            upper_.pop();
        }
    }

    // O(1) — get median
    double findMedian() const {
        if (lower_.size() > upper_.size())
            return lower_.top();
        return (lower_.top() + (long long)upper_.top()) / 2.0;
    }
};
```

**Why the two-heap approach works:**

```
After adding [1, 2, 3, 4, 5]:

lower (max-heap): [3, 2, 1]   ← left half
upper (min-heap): [4, 5]      ← right half

lower.top() = 3 ≤ upper.top() = 4  ✓ (ordering maintained)
|lower| = 3, |upper| = 2 → odd total, median = lower.top() = 3 ✓

After adding [6]:
  Insert 6 into lower: lower=[6,3,2,1]
  6 > upper.top()=4 → move 6 to upper: lower=[3,2,1], upper=[4,5,6]
  |upper|=3 > |lower|=3? No, equal. Hmm — rebalance:
  |upper|=3 > |lower|=3? upper.size() > lower.size() → move 4 to lower.
  lower=[4,3,2,1], upper=[5,6]
  median = (lower.top() + upper.top()) / 2 = (4+5)/2 = 4.5 ✓
```

**Edge cases:**
- Single element: lower has 1, upper has 0. Median = lower.top().
- Two elements: one in each half. Median = average of both tops.
- All same elements (e.g., addNum(5) five times): lower=[5,5,5], upper=[5,5]. Median=5.0 ✓
- Integer overflow: when both tops are large ints, their sum may overflow `int`. Cast to `long long` before adding.

---

### Problem 3: Reorganise String (Greedy + Priority Queue)

**Problem:** Given a string, rearrange its characters so no two adjacent characters are the same. Return any valid rearrangement, or `""` if impossible.

**Example:** `"aab"` → `"aba"` | `"aaab"` → `""`

**Thought process:**

> "Greedy: always place the most frequent remaining character that is different from the last placed character. A max-heap of (count, char) gives the most frequent in O(1)."
>
> "Algorithm: extract the most frequent character, place it. Then extract the second most frequent, place it. Re-insert both with decremented counts. Continue until done or stuck."
>
> "Impossibility condition: if any character appears more than ⌈n/2⌉ times, it cannot be placed without adjacency conflicts."

```cpp
// Time: O(n log k) where k = number of distinct chars (≤26) = O(n)
// Space: O(k) = O(1) for the heap (at most 26 chars)
string reorganizeString(string s) {
    // Count frequencies
    unordered_map<char,int> freq;
    for (char c : s) freq[c]++;

    // Max-heap of (count, char)
    priority_queue<pair<int,char>> maxHeap;
    for (auto& [ch, cnt] : freq) {
        maxHeap.push({cnt, ch});
    }

    string result;
    result.reserve(s.size());

    while (maxHeap.size() >= 2) {
        // Take the two most frequent characters
        auto [cnt1, ch1] = maxHeap.top(); maxHeap.pop();
        auto [cnt2, ch2] = maxHeap.top(); maxHeap.pop();

        result += ch1;
        result += ch2;

        if (cnt1 - 1 > 0) maxHeap.push({cnt1 - 1, ch1});
        if (cnt2 - 1 > 0) maxHeap.push({cnt2 - 1, ch2});
    }

    // At most one character left in heap
    if (!maxHeap.empty()) {
        auto [cnt, ch] = maxHeap.top();
        if (cnt > 1) return "";   // more than 1 of last char → impossible

        result += ch;
    }

    return result;
}

/*
s = "aab": freq = {a:2, b:1}
heap = [(2,'a'), (1,'b')]

Iter 1: pop (2,'a'), (1,'b'). result="ab". push (1,'a'). heap=[(1,'a')]
Iter done (size<2). heap has (1,'a'): cnt=1, result="aba". ✓

s = "aaab": freq = {a:3, b:1}
heap = [(3,'a'), (1,'b')]

Iter 1: pop (3,'a'), (1,'b'). result="ab". push (2,'a'). heap=[(2,'a')]
Iter done. heap has (2,'a'): cnt=2>1 → return "". ✓
*/
```

**A cleaner alternative — interleave approach:**

```cpp
string reorganizeString(string s) {
    int freq[26] = {};
    for (char c : s) freq[c - 'a']++;

    // Find the most frequent character
    int maxFreq = *max_element(begin(freq), end(freq));
    if (maxFreq > (s.size() + 1) / 2) return "";   // impossible

    // Place most frequent character at even indices first, then fill odds
    string result(s.size(), ' ');
    int idx = 0;

    for (int i = 0; i < 26; i++) {
        while (freq[i] > 0) {
            result[idx] = 'a' + i;
            idx += 2;                    // place at even indices first
            if (idx >= (int)s.size()) idx = 1;  // switch to odd indices
            freq[i]--;
        }
    }
    return result;
}
// This O(n) approach works for this specific problem but does not generalise.
// The heap approach generalises to any "place characters with constraints" problem.
```

**Edge cases:**
- Single character: if s="a", return "a". freq[a]=1 ≤ ⌈1/2⌉=1. Valid.
- All same character of length 1: valid.
- All same character of length > 1: impossible. `cnt > 1` catches this.
- Empty string: return "". Handle before building the heap.
- Two distinct characters with equal counts: heap takes turns placing each. Always valid.

---

## 10. Real-World Uses

| Domain | Use Case | Priority Queue Role |
|---|---|---|
| **Networking** | Dijkstra / OSPF routing | Min-heap finds next closest router in O(log V) |
| **Game engines** | A\* pathfinding | Min-heap ordered by f = g + h score |
| **OS kernel** | Process scheduler (priority scheduling) | Max-heap by priority; highest-priority process runs next |
| **Databases** | Sort-merge join, external sorting | Min-heap merges k sorted runs from disk |
| **Event simulation** | Discrete event simulator | Min-heap of (timestamp, event); process in chronological order |
| **Data compression** | Huffman coding | Min-heap builds the optimal prefix-free code tree |
| **Real-time systems** | Task scheduler with deadlines | Min-heap by deadline (EDF scheduling) |
| **Machine learning** | K-nearest neighbours | Max-heap of size k maintains k closest training examples |
| **Streaming analytics** | Top-K heavy hitters | Min-heap of size k; new elements displace if larger |
| **Load balancers** | Least-connections load balancing | Min-heap of (active_connections, server); route to minimum |
| **Cloud infrastructure** | Kubernetes pod scheduling | Min-heap of (available_resources, node); place on most available |

**Huffman Coding — The Min-Heap in Action:**

```
Input: "aabbcccddddeeeeef"
Frequencies: a=2, b=2, c=3, d=4, e=5, f=1

Build Huffman tree using min-heap:
Initial heap: [(1,f), (2,a), (2,b), (3,c), (4,d), (5,e)]

Step 1: Extract f(1) and a(2). Merge → node(3). Push (3, fa_tree).
  heap: [(2,b), (3,c), (3,fa_tree), (4,d), (5,e)]

Step 2: Extract b(2) and c(3). Merge → node(5). Push (5, bc_tree).
  heap: [(3,fa_tree), (4,d), (5,e), (5,bc_tree)]

Step 3: Extract fa_tree(3) and d(4). Merge → node(7). Push (7, fad_tree).
  heap: [(5,e), (5,bc_tree), (7,fad_tree)]

Step 4: Extract e(5) and bc_tree(5). Merge → node(10). Push.
  heap: [(7,fad_tree), (10,ebc_tree)]

Step 5: Extract both remaining. Merge → root(17).

Result: optimal prefix-free codes.
  e: 0 (most frequent → shortest code)
  d: 10, b: 110, c: 111
  a: 1100, f: 1101  (least frequent → longest codes)

Each step: extract 2 minimum nodes O(log n), insert 1 O(log n).
Total: O(n log n) for n distinct characters.
```

**Kubernetes Scheduler:**

The Kubernetes scheduler maintains a priority queue of pending pods sorted by scheduling priority (user-defined) and arrival time. When a node reports available resources, the scheduler pops the highest-priority pod from the queue and binds it to the node. Custom schedulers implement this directly with `heap.Interface` in Go — the same binary heap structure underlying C++'s `std::priority_queue`.

---

## 11. Edge Cases & Pitfalls

### Pitfall 1: Default std::priority_queue Is Max-Heap, Not Min-Heap

```cpp
// WRONG assumption: thinking std::priority_queue gives smallest element first
priority_queue<int> pq;
pq.push(3); pq.push(1); pq.push(4); pq.push(1); pq.push(5);
cout << pq.top();   // 5 — this is MAX, not min!

// For Dijkstra and Top-K smallest, you need min-heap:
priority_queue<int, vector<int>, greater<int>> minPQ;
// Now minPQ.top() gives the smallest element.

// Common mistake in Dijkstra:
priority_queue<pair<int,int>> wrongPQ;   // max-heap → processes farthest node first!
priority_queue<pair<int,int>,
               vector<pair<int,int>>,
               greater<pair<int,int>>> correctPQ;  // min-heap ✓
```

### Pitfall 2: std::priority_queue Has No decrease_key

```cpp
// std::priority_queue does NOT support changing the priority of an element already in the heap.
// This is the main limitation vs a more sophisticated heap (Fibonacci, indexed heap).

// WRONG: attempting to update an existing element
priority_queue<pair<int,int>,vector<pair<int,int>>,greater<pair<int,int>>> pq;
pq.push({10, 0});
// Later: distance to node 0 improved to 5
// pq.update({5, 0})?  ← DOES NOT EXIST

// CORRECT: use lazy deletion — push the new (better) entry and ignore the stale one
pq.push({5, 0});   // push updated entry
// When {10, 0} is eventually popped, check if dist[0] is still 10.
// If dist[0] < 10 (i.e., we already processed it), skip it:
auto [d, u] = pq.top(); pq.pop();
if (d > dist[u]) continue;   // stale entry — skip
```

### Pitfall 3: Comparing pairs — First Element Is Primary Key

```cpp
// For priority_queue<pair<int,int>>:
// Comparison is lexicographic: first compare first element, then second.

priority_queue<pair<int,int>,vector<pair<int,int>>,greater<pair<int,int>>> pq;
pq.push({5, 100});
pq.push({5, 50});
pq.push({3, 200});

// Order popped: {3,200}, {5,50}, {5,100}
// Ties in first element broken by second element (smaller second pops first in min-heap).

// In Dijkstra: pair<dist, node>. If two nodes have same distance, the one with smaller
// node index pops first — this is fine for correctness but may affect which path is returned.
```

### Pitfall 4: Integer Overflow in Distance Calculations

```cpp
// When using INT_MAX as "infinity" for unvisited nodes in Dijkstra:
int dist[V];
fill(dist, dist + V, INT_MAX);

// WRONG: integer overflow
if (dist[u] + weight < dist[v]) ...   // dist[u] == INT_MAX → INT_MAX + weight overflows!

// CORRECT: guard against overflow before adding
if (dist[u] != INT_MAX && dist[u] + weight < dist[v]) {
    dist[v] = dist[u] + weight;
    pq.push({dist[v], v});
}
// Or use long long for dist array.
```

### Pitfall 5: Median Maintenance — Integer Overflow in Average

```cpp
// WRONG: overflow when both tops are large positive ints
double findMedian() {
    return (lower_.top() + upper_.top()) / 2.0;  // overflow if both near INT_MAX
}

// CORRECT: cast before arithmetic
double findMedian() {
    return ((long long)lower_.top() + upper_.top()) / 2.0;
}
```

### Pitfall 6: Not Handling Empty Queue Before top()

```cpp
// std::priority_queue::top() on an empty queue is UNDEFINED BEHAVIOUR (not an exception)
priority_queue<int> pq;
int x = pq.top();   // UB! May crash, may return garbage, may corrupt memory.

// ALWAYS check before top():
if (!pq.empty()) {
    int x = pq.top();
    pq.pop();
}
```

### Pitfall 7: Using Priority Queue When Monotonic Deque Is Better

```cpp
// Sliding window MAXIMUM of fixed size k:
// Priority queue approach: O(n log k) — push each element, use lazy deletion for eviction
// Monotonic deque approach: O(n) — each element pushed and popped exactly once

// If k is fixed and you process a continuous stream, use monotonic deque.
// If the window is not contiguous or elements can be removed arbitrarily, use priority queue.

// Symptom: "sliding window max/min with fixed k" → monotonic deque (faster).
// Symptom: "top-K from arbitrary position" → priority queue.
```

### Pitfall 8: Build Heap Complexity Confusion

```cpp
// WRONG claim: building a heap from n elements is O(n log n)
// This is what you'd get if you called push() n times.

// CORRECT: Floyd's build-heap algorithm is O(n)
// Use the range constructor of std::priority_queue for O(n) build:
vector<int> data = {5, 3, 8, 1, 9, 2};

// O(n) build — use range constructor
priority_queue<int> pq(data.begin(), data.end());

// O(n log n) build — push one by one (unnecessarily slow)
priority_queue<int> pq2;
for (int x : data) pq2.push(x);   // 6 × O(log 6) operations

// The range constructor internally calls std::make_heap, which uses Floyd's O(n) algorithm.
```

---

## 12. std::priority_queue

```cpp
#include <queue>

// Declaration:
// priority_queue<T, Container, Compare>
// T         = element type
// Container = underlying container (default: vector<T>)
// Compare   = comparison function (default: less<T> → MAX heap)

// Common instantiations:
priority_queue<int>                                         maxPQ;     // max-heap
priority_queue<int, vector<int>, greater<int>>              minPQ;     // min-heap
priority_queue<pair<int,int>>                               maxPairPQ; // pairs, max by first
priority_queue<pair<int,int>,
               vector<pair<int,int>>, greater<pair<int,int>>> minPairPQ;  // min by first

// Member functions — complete list:
pq.push(x)      // O(log n) — insert x
pq.emplace(...)  // O(log n) — construct in place, prefer over push for complex types
pq.pop()         // O(log n) — remove top, returns VOID
pq.top()         // O(1)     — reference to top element, no removal
pq.empty()       // O(1)     — true if no elements
pq.size()        // O(1)     — number of elements
pq.swap(other)   // O(1)     — swap contents

// What std::priority_queue does NOT have:
// - Random access or iteration (no begin/end)
// - Search (no find)
// - decrease_key or increase_key
// - Merge (no merge operation)
// - clear() — use pq = priority_queue<T>() to empty it

// The std::heap algorithms (for more control):
#include <algorithm>
vector<int> v = {3, 1, 4, 1, 5, 9};
make_heap(v.begin(), v.end());           // O(n)  — heapify in-place (max-heap)
push_heap(v.begin(), v.end());           // O(log n) — after push_back, restore heap
pop_heap(v.begin(), v.end());            // O(log n) — move max to back; then pop_back
is_heap(v.begin(), v.end());             // O(n)  — check if range is a max-heap
sort_heap(v.begin(), v.end());           // O(n log n) — heap sort (destroys heap property)

// Custom comparator:
make_heap(v.begin(), v.end(), greater<int>());  // min-heap
```

---

## 13. Priority Queue Variants

| Variant | Key Feature | Use Case | C++ |
|---|---|---|---|
| **Binary Heap** | Simple, cache-friendly, O(log n) ops | Dijkstra, Top-K, Median | `std::priority_queue` |
| **Fibonacci Heap** | O(1) amortized decrease_key | Theoretical Dijkstra O(E + V log V) | Custom (no STL) |
| **Binomial Heap** | O(log n) merge | Mergeable priority queues | Custom |
| **Pairing Heap** | Simpler than Fibonacci, fast in practice | Decrease-key intensive workloads | Custom |
| **Indexed Priority Queue** | O(log n) decrease_key + O(1) key lookup by index | Dijkstra with mutable priorities | Custom |
| **Double-Ended PQ** | O(log n) access to both min and max | Interval problems, median | Min-max heap |
| **Lazy Deletion PQ** | Soft delete + skip on pop | Dijkstra, removing stale entries | `std::priority_queue` + set |
| **Monotonic Deque** | O(1) window max/min | Fixed-size sliding window | `std::deque` |

**The Indexed Priority Queue** — the structure used in production Dijkstra implementations:

```cpp
// Standard priority_queue: when you update a node's distance, you push a new entry.
// Old entry remains as a stale "ghost." Works but wastes space and time.

// Indexed PQ: maps node indices to heap positions. Allows O(log n) decrease_key.
// Implemented as a binary heap + inverse index array.
class IndexedPQ {
    vector<int> heap_;       // heap[i] = node index at heap position i
    vector<int> pos_;        // pos[v]  = heap position of node v (-1 if not present)
    vector<int> dist_;       // dist[v] = current distance of node v

    void decrease_key(int v, int new_dist) {
        dist_[v] = new_dist;
        siftUp(pos_[v]);     // O(log n) — fix heap property upward
    }
};
// Used in all serious graph algorithm implementations (Boost.Graph, competitive programming).
```

---

## 14. Comparison

| Feature | Priority Queue (heap) | Sorted Array | Balanced BST (`std::set`) | Monotonic Deque |
|---|---|---|---|---|
| Insert | O(log n) | O(n) | O(log n) | O(1) amortized |
| Extract min/max | O(log n) | O(1) | O(log n) | O(1) |
| Peek min/max | O(1) | O(1) | O(log n) | O(1) |
| Search | O(n) | O(log n) | O(log n) | O(n) |
| Delete arbitrary | O(n) | O(n) | O(log n) | O(n) |
| decrease_key | O(n) naive / O(log n) indexed | O(n) | O(log n) | N/A |
| Merge | O(n) | O(n) | O(n) | N/A |
| Build from n elements | O(n) | O(n log n) | O(n log n) | O(n) |
| Cache performance | ★★★ Excellent (array) | ★★★ Excellent | ★ Poor (pointers) | ★★★ Excellent |
| Ordered iteration | Destroys heap | Yes (free) | Yes (free) | Destroys deque |
| Supports duplicates | Yes | Yes | No (use multiset) | Yes |
| Window eviction | With lazy deletion | N/A | O(log n) | O(1) amortized |

**The critical decision between priority queue and BST:**

```
Priority Queue wins when:
  → You only need min or max (not both or arbitrary rank)
  → Cache performance matters (no pointer chasing)
  → You need O(n) build from known data
  → You do not need ordered traversal

BST (std::set/multiset) wins when:
  → You need arbitrary deletion (not just min/max)
  → You need both min AND max in O(log n)
  → You need O(log n) rank queries (k-th smallest)
  → You need ordered iteration

Example: Dijkstra with decrease_key
  Binary heap + lazy deletion: simpler, cache-friendly, fast in practice
  BST (std::set of {dist, node}): supports true decrease_key via erase+insert
  Choice: use lazy deletion with binary heap for most problems.
```

---

## 15. Self-Test Questions

1. **State the heap property for a max-heap. For a min-heap. Is [10, 8, 9, 7, 6, 5, 4] a valid max-heap? Verify every parent-child relationship.**

2. **Trace the sift-up operation when inserting 95 into the max-heap [100, 80, 70, 50, 60, 30, 40]. Show every swap and the final array.**

3. **Prove that Floyd's build-heap algorithm runs in O(n) rather than O(n log n). Your proof must use the geometric series argument on the sum of work across all levels.**

4. **Why is `std::priority_queue` a max-heap by default and how do you make it a min-heap? What is the type signature for a min-heap of pairs sorted by first element?**

5. **In Dijkstra's algorithm with a binary heap, what is lazy deletion and why is it used instead of decrease_key? What is the worst-case number of elements in the heap at any time with lazy deletion?**

6. **Trace the two-heap median maintenance algorithm on the stream: 6, 2, 10, 3, 8, 5. Show the state of both heaps and the median after each insertion.**

7. **The K Closest Points problem uses a MAX-heap of size k to find the k SMALLEST distances. Why not a min-heap? What invariant does the max-heap maintain?**

8. **`std::priority_queue::top()` on an empty queue is undefined behaviour, not an exception. Why? What does this imply about caller responsibilities?**

9. **When would you use a BST (std::set) instead of a priority_queue? Give a concrete problem where a priority_queue cannot be used but a set can.**

10. **Design a system that processes customer support tickets with priorities 1-10. High-priority tickets must be handled first, but within the same priority, they should be handled in arrival order (FIFO). What data structure would you use? How would you implement it?**

---

## Quick Reference Card

```
Priority Queue — serves the highest-priority element next. O(log n) push/pop. O(1) peek.

C++ instantiation:
  Max-heap (default):  priority_queue<int> pq;
  Min-heap:            priority_queue<int, vector<int>, greater<int>> pq;
  Custom:              priority_queue<T, vector<T>, decltype(cmp)> pq(cmp);

Core operations (all via binary heap):
  push(x):   append to array, sift UP    → O(log n)
  pop():     move last to root, sift DOWN → O(log n)
  top():     return heap_[0]              → O(1)
  build n:   Floyd's sift-down from n/2  → O(n)

Array index formulas (0-based):
  parent(i)       = (i-1) / 2
  left_child(i)   = 2*i + 1
  right_child(i)  = 2*i + 2

Key patterns (master all four):
  1. Top-K largest:   MIN-heap of size k (min at top = smallest of k largest)
  2. Top-K smallest:  MAX-heap of size k
  3. Dijkstra:        MIN-heap of (dist, node), lazy deletion for stale entries
  4. Median stream:   MAX-heap (lower half) + MIN-heap (upper half), balanced sizes

Pitfall checklist:
  ✗ Assuming default PQ is min-heap — it is MAX-heap
  ✗ Calling top() on empty PQ — undefined behaviour, not exception
  ✗ Using INT_MAX as dist and adding to it — integer overflow
  ✗ Not using lazy deletion for Dijkstra — stale entries accumulate
  ✗ Using PQ for sliding window max — use monotonic deque (O(n) vs O(n log k))
  ✗ Building with n pushes — use range constructor for O(n) build

std::priority_queue limitations:
  - No decrease_key / increase_key
  - No iteration / random access
  - No clear() — use pq = priority_queue<T>()
  - No merge

When to use PQ vs alternatives:
  Fixed sliding window max/min  → monotonic deque O(n)
  Arbitrary deletion            → std::set O(log n)
  Both min and max              → std::set or min-max heap
  Dijkstra / A* / Huffman       → priority_queue (binary heap) ← default choice
```

---

*Previous: [12 — Monotonic Deque](./12_monotonic_deque.md)*
*Next: [14 — Hash Map / Hash Table](./14_hash_map.md)*
*See also: [Binary Heap (deep dive)](./heaps/binary_heap.md) | [Fibonacci Heap](./heaps/fibonacci_heap.md) | [Dijkstra's Algorithm](./algorithms/dijkstra.md)*

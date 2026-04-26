# Deque (Double-Ended Queue)

> **Curriculum position:** Linear Structures → #11
> **Interview weight:** ★★★★☆ High — sliding window maximum, palindrome check, monotonic deque, and BFS variants all depend on it
> **Difficulty:** Intermediate — the ADT is simple; std::deque's internal chunked layout and the monotonic deque pattern require depth
> **Prerequisite:** [09 — Queue](./09_queue.md) | [10 — Circular Queue](./10_circular_queue.md) | [07 — Stack](./07_stack.md)

---

## Table of Contents

1. [Intuition First — The Two-Ended Container](#1-intuition-first)
2. [The Deque ADT Contract](#2-the-deque-adt-contract)
3. [Internal Working — How std::deque Is Actually Built](#3-internal-working)
4. [The Chunked Array Layout — Deep Dive](#4-the-chunked-array-layout)
5. [Time & Space Complexity](#5-time--space-complexity)
6. [Complete C++ Implementation](#6-complete-c-implementation)
7. [Core Operations — Visualised](#7-core-operations--visualised)
8. [The Monotonic Deque — The Killer Pattern](#8-the-monotonic-deque)
9. [Interview Problems](#9-interview-problems)
10. [Real-World Uses](#10-real-world-uses)
11. [Edge Cases & Pitfalls](#11-edge-cases--pitfalls)
12. [Comparison: Deque vs Vector vs List vs Queue vs Stack](#12-comparison)
13. [Self-Test Questions](#13-self-test-questions)

---

## 1. Intuition First

Every container you have studied so far sacrifices something at one end:

- A **stack** gives O(1) push and pop only at the top — the bottom is inaccessible.
- A **queue** gives O(1) enqueue at the back and O(1) dequeue at the front — inserting at the front or removing from the back is O(n).
- A **vector** gives O(1) push/pop at the back — but O(n) at the front because every element must shift.

The **deque** — double-ended queue — removes all these restrictions. It supports O(1) push and pop at **both** the front **and** the back simultaneously. It is, in a sense, the most general linear access structure: a stack is a deque with the front sealed; a queue is a deque with push-front and pop-back sealed.

Think of a train carriage that can load and unload passengers from both the front door and the back door at the same time. New passengers can board either end. Departing passengers can leave from either end. The train body stays fixed — only the doors matter.

This generality makes the deque the natural backing structure for several higher-level abstractions:
- `std::stack` and `std::queue` in C++ both use `std::deque` by default
- The **monotonic deque** — the structure that solves sliding window maximum in O(n) — is built directly on deque operations
- **0-1 BFS** uses a deque to push weight-0 edges to the front and weight-1 edges to the back
- **Palindrome checking** reads from both ends simultaneously

The price of this generality: the internal layout is more complex than a vector (chunked arrays instead of one contiguous block), and random access, while O(1), has a larger constant factor than vector's direct pointer arithmetic.

---

## 2. The Deque ADT Contract

```
push_front(x)   — insert x at the front        O(1) amortized
push_back(x)    — insert x at the back         O(1) amortized
pop_front()     — remove from the front        O(1) amortized
pop_back()      — remove from the back         O(1) amortized
front()         — peek at front element        O(1)
back()          — peek at back element         O(1)
operator[](i)   — random access by index       O(1)
size()          — number of elements           O(1)
empty()         — true if no elements          O(1)
```

**The key guarantee that makes it different from everything else:**
O(1) at BOTH ends. This is the defining property. Any implementation that provides this is a valid deque.

---

## 3. Internal Working — How std::deque Is Actually Built

### Why a Circular Array Alone Is Not Enough

A circular array achieves O(1) push/pop at both ends — so why does `std::deque` not just use one? Two reasons:

1. **Fixed capacity**: a circular array has a fixed size. When full, it must reallocate and copy all n elements — O(n). For a deque, this should be avoidable.
2. **Contiguous reallocation is expensive**: when a `vector` grows, it copies everything to a new block. For a structure that supports growth at *both* ends, doing this on every resize would make push_front consistently expensive.

### The Chunked Array (Segmented Array) Layout

`std::deque` solves both problems by using a **map of fixed-size chunks**:

```
std::deque internal structure:

   map (array of pointers to chunks):
   ┌──────┬──────┬──────┬──────┬──────┐
   │ ptr0 │ ptr1 │ ptr2 │ ptr3 │ ptr4 │   ← the "map" (array of chunk pointers)
   └──────┴──────┴──────┴──────┴──────┘
      │       │       │       │
      ▼       ▼       ▼       ▼
   [C0 C1] [C2 C3] [C4 C5] [C6 C7]       ← chunks (fixed-size arrays)
    chunk0   chunk1   chunk2   chunk3

   start_chunk = 1    (first occupied chunk index in the map)
   start_pos   = 1    (first occupied slot within start_chunk)
   end_chunk   = 3    (last occupied chunk index in the map)
   end_pos     = 2    (one past last occupied slot in end_chunk)
```

**push_back**: write to `end_pos` in `end_chunk`. Advance `end_pos`. If `end_pos` reaches the chunk end, allocate a new chunk, append its pointer to the map, reset `end_pos = 0`. The map itself only needs reallocation when all map slots are used — a rare event.

**push_front**: write to `start_pos - 1` in `start_chunk`. Retreat `start_pos`. If `start_pos` goes below 0, allocate a new chunk, prepend its pointer to the map. Crucially: prepending to the map only copies map pointers (small), not element data — O(1) amortized.

**Random access `deque[i]`**: compute which chunk and which offset within that chunk in O(1) using division and modulo:

```
global_index = start_pos + i
chunk_index  = global_index / chunk_size
slot_in_chunk = global_index % chunk_size
element = map[start_chunk + chunk_index][slot_in_chunk]
```

Two pointer dereferences instead of one (map → chunk → element). This is why `deque[i]` has a larger constant factor than `vector[i]` despite both being O(1).

### Iterator Invalidation Rules

Unlike `vector` (where push_back may invalidate all iterators), `std::deque` has more nuanced rules:

| Operation | Iterators/References Invalidated |
|---|---|
| `push_back` | All iterators invalidated (map may reallocate). References to elements remain valid. |
| `push_front` | All iterators invalidated. References to elements remain valid. |
| `pop_back` | Iterator to erased element and `end()` invalidated only. |
| `pop_front` | Iterator to erased element and `begin()` invalidated only. |
| `insert` / `erase` in middle | All iterators and references invalidated. |

**The key subtlety**: push_front/push_back invalidate iterators (because the map may reallocate, changing the iterator's internal map pointer) but do NOT invalidate references/pointers to existing elements (the elements themselves do not move). This is different from `vector`, where reallocation moves elements.

---

## 4. The Chunked Array Layout — Deep Dive

Understanding this layout at a concrete level separates candidates who know deque from candidates who truly understand it.

### Concrete Example: chunk_size = 4

```
Initial empty deque:
  map: [ ptr0 ]         (one chunk pre-allocated, start in the middle)
  chunk0: [_ _ _ _]
  start_chunk=0, start_pos=2, end_chunk=0, end_pos=2
  (start_pos = end_pos = middle of chunk → empty)

After push_back(A, B, C):
  map: [ ptr0 ]
  chunk0: [_ _ A B]    A at pos 2, B at pos 3
  Wait — B at pos 3 fills the chunk. C needs a new chunk.

  map: [ ptr0 | ptr1 ]
  chunk0: [_ _ A B]
  chunk1: [C _ _ _]
  start_chunk=0, start_pos=2, end_chunk=1, end_pos=1

After push_front(Z):
  Z goes before A. start_pos retreats from 2 to 1.
  map: [ ptr0 | ptr1 ]
  chunk0: [_ Z A B]
  chunk1: [C _ _ _]
  start_chunk=0, start_pos=1, end_chunk=1, end_pos=1

After push_front(Y, X):  (Y fills slot 0 of chunk0; X needs a new front chunk)
  map: [ ptr2 | ptr0 | ptr1 ]   ← ptr2 prepended
  chunk2: [_ _ _ X]             ← X at the LAST slot of the new front chunk
  chunk0: [Y Z A B]
  chunk1: [C _ _ _]
  start_chunk=0 (in map), start_pos=3 (in chunk2), end_chunk=2, end_pos=1

Random access: deque[0] = X
  global = start_pos + 0 = 3
  chunk  = 3 / 4 = 0 → chunk2 (map[start_chunk + 0] = map[0] = ptr2 = chunk2)
  slot   = 3 % 4 = 3
  element = chunk2[3] = X ✓

Random access: deque[3] = B
  global = 3 + 3 = 6
  chunk  = 6 / 4 = 1 → chunk0 (map[start_chunk + 1] = map[1] = ptr0 = chunk0)
  slot   = 6 % 4 = 2
  element = chunk0[2] = A   Wait — let me recount.

  Actually: logical order is X Y Z A B C
  deque[0]=X, deque[1]=Y, deque[2]=Z, deque[3]=A, deque[4]=B, deque[5]=C
  deque[3]=A:
    global = start_pos(=3 in chunk2) + 3 = 6  (in absolute slot terms)
    Within map: map_offset = 6 / 4 = 1  → chunk0 (map[1] = ptr0)
    slot in chunk = 6 % 4 = 2
    chunk0[2] = A ✓
```

This two-level addressing is the entire secret of `std::deque`. Growing at either end only ever requires allocating one new chunk and (rarely) reallocating the small map array — never moving existing elements.

---

## 5. Time & Space Complexity

### Operation Complexities

| Operation | `std::deque` | `std::vector` | Linked List | Notes |
|---|---|---|---|---|
| `push_front(x)` | **O(1) amortized** | O(n) — shift all | O(1) | Deque's main advantage over vector |
| `push_back(x)` | **O(1) amortized** | O(1) amortized | O(1) | Same as vector |
| `pop_front()` | **O(1) amortized** | O(n) — shift all | O(1) | Deque's main advantage over vector |
| `pop_back()` | **O(1) amortized** | O(1) | O(n) SLL / O(1) DLL | Same as vector |
| `operator[](i)` | **O(1)** — 2 ptr deref | O(1) — 1 ptr deref | O(n) | Vector is faster in practice |
| `front()` | **O(1)** | O(1) | O(1) | All same |
| `back()` | **O(1)** | O(1) | O(1) | All same |
| `insert` (middle) | O(n) | O(n) | O(1) + O(n) search | All O(n) |
| `erase` (middle) | O(n) | O(n) | O(1) + O(n) search | All O(n) |
| `size()` | O(1) | O(1) | O(1) | All same |
| Iteration | O(n) | O(n) | O(n) | Vector fastest (cache) |

### Space Complexity

| | |
|---|---|
| n elements | O(n) |
| Per-element overhead | Near zero (chunk array, no per-element pointers) |
| Internal map overhead | O(n/chunk_size) — proportional to number of chunks |
| Wasted capacity | Up to 2 × chunk_size slots at front and back chunks |
| vs vector | Slightly more overhead; less contiguous; worse cache behaviour |

### The Cache-Performance Reality

`std::deque` is not cache-friendly like `std::vector`. Sequential iteration traverses across chunk boundaries, each requiring a new pointer dereference into the map. Benchmarks consistently show:

```
Sequential sum of n=1,000,000 integers:
  vector:  ~1.2 ms   (one contiguous block, hardware prefetcher excels)
  deque:   ~3.5 ms   (chunk boundaries cause periodic cache misses)

Ratio: deque is ~3× slower for sequential access.
```

Use `std::deque` when you need O(1) push/pop at both ends. Use `std::vector` when you only need O(1) at the back and optimal cache performance matters.

---

## 6. Complete C++ Implementation

### Implementation 1: Circular-Array-Backed Deque (Interview / Competitive Use)

For interviews, implement a deque with a circular array — it is simpler than chunked arrays and sufficient for any bounded problem.

```cpp
#include <iostream>
#include <vector>
#include <stdexcept>
using namespace std;

template<typename T>
class Deque {
private:
    vector<T> data_;
    int front_;     // index of the front element
    int back_;      // index of the back element
    int size_;
    int cap_;

    int advance(int i) const { return (i + 1) % cap_; }
    int retreat(int i) const { return (i - 1 + cap_) % cap_; }

public:
    explicit Deque(int capacity)
        : data_(capacity), front_(capacity / 2), back_(capacity / 2 - 1),
          size_(0), cap_(capacity) {
        // front_ and back_ initialised symmetrically around the middle.
        // push_front writes to front_; push_back advances back_ then writes.
        // This convention: front_ points TO the front element (or next front slot when empty).
        //                  back_ points TO the back element (or prev back slot when empty).
    }

    // ── State ─────────────────────────────────────────────────────────────────
    bool empty()    const { return size_ == 0; }
    bool full()     const { return size_ == cap_; }
    int  size()     const { return size_; }
    int  capacity() const { return cap_; }

    // ── Front operations ──────────────────────────────────────────────────────

    // O(1) — insert at front
    void push_front(const T& val) {
        if (full()) throw overflow_error("deque is full");
        if (!empty()) front_ = retreat(front_);  // move front back one slot
        data_[front_] = val;
        size_++;
    }

    // O(1) — remove from front
    T pop_front() {
        if (empty()) throw underflow_error("deque is empty");
        T val = data_[front_];
        if (size_ > 1) front_ = advance(front_);
        size_--;
        return val;
    }

    T& front() {
        if (empty()) throw underflow_error("deque is empty");
        return data_[front_];
    }

    // ── Back operations ───────────────────────────────────────────────────────

    // O(1) — insert at back
    void push_back(const T& val) {
        if (full()) throw overflow_error("deque is full");
        if (!empty()) back_ = advance(back_);    // move back forward one slot
        data_[back_] = val;
        size_++;
    }

    // O(1) — remove from back
    T pop_back() {
        if (empty()) throw underflow_error("deque is empty");
        T val = data_[back_];
        if (size_ > 1) back_ = retreat(back_);
        size_--;
        return val;
    }

    T& back() {
        if (empty()) throw underflow_error("deque is empty");
        return data_[back_];
    }

    // ── Random access O(1) ────────────────────────────────────────────────────
    T& operator[](int i) {
        if (i < 0 || i >= size_) throw out_of_range("index out of range");
        return data_[(front_ + i) % cap_];
    }

    void print() const {
        if (empty()) { cout << "(empty deque)\n"; return; }
        cout << "front ↔ ";
        for (int i = 0; i < size_; i++) {
            cout << data_[(front_ + i) % cap_];
            if (i < size_ - 1) cout << " ↔ ";
        }
        cout << " ↔ back\n";
    }
};
```

### Implementation 2: std::deque Full API Reference

```cpp
#include <deque>
#include <algorithm>
#include <numeric>
using namespace std;

int main() {
    // ── Construction ──────────────────────────────────────────────────────────
    deque<int> d1;                         // empty
    deque<int> d2(5, 42);                  // {42,42,42,42,42}
    deque<int> d3 = {1, 2, 3, 4, 5};      // initialiser list
    deque<int> d4(d3);                     // copy — O(n)
    deque<int> d5(move(d3));               // move — O(1)

    // ── Front/Back Operations ─────────────────────────────────────────────────
    d1.push_front(10);         // insert at front — O(1) amortized
    d1.push_back(20);          // insert at back  — O(1) amortized
    d1.emplace_front(5);       // construct in-place at front — prefer over push_front
    d1.emplace_back(25);       // construct in-place at back  — prefer over push_back

    d1.pop_front();            // remove front — O(1) amortized, returns void
    d1.pop_back();             // remove back  — O(1) amortized, returns void

    d1.front();                // peek front — O(1), reference
    d1.back();                 // peek back  — O(1), reference

    // ── Random Access ─────────────────────────────────────────────────────────
    d2[2];                     // O(1), no bounds check — UB if out of range
    d2.at(2);                  // O(1), throws std::out_of_range if OOB

    // ── Size & Capacity ───────────────────────────────────────────────────────
    d2.size();                 // O(1) — number of elements
    d2.empty();                // O(1)
    d2.max_size();             // implementation-defined maximum
    // Note: std::deque has NO reserve() or capacity() — unlike vector.
    // You cannot pre-allocate chunks in advance.
    d2.shrink_to_fit();        // hint to release unused chunks

    // ── Modifiers ─────────────────────────────────────────────────────────────
    d2.clear();                // O(n) — destroy all elements
    d2.resize(8);              // O(n) — grow/shrink (zero-init new elements)
    d2.resize(3, 99);          // O(n) — grow with fill value

    deque<int> d6 = {1, 2, 3};
    auto it = d6.begin() + 1;
    d6.insert(it, 99);         // O(n) — insert before iterator position
    d6.erase(it);              // O(n) — erase at iterator position
    d6.erase(d6.begin(), d6.begin() + 2);  // O(n) — erase range

    d6.assign(5, 7);           // O(n) — replace all contents
    d6.swap(d2);               // O(1) — swap internal pointers

    // ── Iteration ─────────────────────────────────────────────────────────────
    for (int x : d6) cout << x << " ";    // range-for — O(n)
    for (auto it = d6.rbegin(); it != d6.rend(); ++it)  // reverse iteration
        cout << *it << " ";

    // ── Algorithms ────────────────────────────────────────────────────────────
    sort(d6.begin(), d6.end());
    reverse(d6.begin(), d6.end());
    int s = accumulate(d6.begin(), d6.end(), 0);
    auto pos = find(d6.begin(), d6.end(), 3);

    // ── Using deque as a stack ─────────────────────────────────────────────────
    deque<int> stk;
    stk.push_back(1);   stk.push_back(2);   stk.push_back(3);
    while (!stk.empty()) { cout << stk.back(); stk.pop_back(); }   // 3 2 1

    // ── Using deque as a queue ─────────────────────────────────────────────────
    deque<int> q;
    q.push_back(1);   q.push_back(2);   q.push_back(3);
    while (!q.empty()) { cout << q.front(); q.pop_front(); }        // 1 2 3

    return 0;
}
```

### Implementation 3: The Monotonic Deque (The Most Important Pattern)

```cpp
// Monotonic deque: maintains elements in sorted order (increasing or decreasing)
// by popping from back when a new element violates the order.
// Front gives the maximum (decreasing) or minimum (increasing) of the window.

// Decreasing Monotonic Deque (front = max of current window)
class MonotonicDeque {
private:
    deque<int> dq_;   // stores INDICES, not values

public:
    // Add index i (value = nums[i]) to the back
    // Pop all indices from back whose values are ≤ nums[i] (they can never be max)
    void push(const vector<int>& nums, int i) {
        while (!dq_.empty() && nums[dq_.back()] <= nums[i]) {
            dq_.pop_back();
        }
        dq_.push_back(i);
    }

    // Remove index i from the front if it has fallen outside the window [i-k+1, i]
    void evict(int i, int k) {
        if (!dq_.empty() && dq_.front() < i - k + 1) {
            dq_.pop_front();
        }
    }

    // Get the maximum value in the current window
    int getMax(const vector<int>& nums) const {
        return nums[dq_.front()];
    }

    bool empty() const { return dq_.empty(); }
};
```

---

## 7. Core Operations — Visualised

### push_front and push_back with Circular Array

```
Deque with capacity=8, initially empty:
  data: [_][_][_][_][_][_][_][_]
                front=4, back=3, size=0
                (front and back initialised to middle; both "before" any element)

push_back(A):   back = advance(3) = 4; data[4] = A; size=1
  [_][_][_][_][A][_][_][_]
                ↑
           front=back=4

push_back(B):   back = advance(4) = 5; data[5] = B; size=2
  [_][_][_][_][A][B][_][_]
                ↑   ↑
             front  back

push_front(Z):  front = retreat(4) = 3; data[3] = Z; size=3
  [_][_][_][Z][A][B][_][_]
             ↑           ↑
           front         back

push_front(Y):  front = retreat(3) = 2; data[2] = Y; size=4
push_front(X):  front = retreat(2) = 1; data[1] = X; size=5
  [_][X][Y][Z][A][B][_][_]
      ↑               ↑
    front             back

Logical order: X Y Z A B   (front to back, left to right)
operator[](2) = data[(front + 2) % 8] = data[(1+2)%8] = data[3] = Z ✓

pop_back():   val=data[5]=B; back=retreat(5)=4; size=4
pop_front():  val=data[1]=X; front=advance(1)=2; size=3

Logical order: Y Z A
  data: [_][_][Y][Z][A][_][_][_]
               ↑       ↑
             front     back
```

### The Chunked Layout of std::deque — Operator[] Computation

```
chunk_size=4, map has 3 chunks:
  map: [ *chunk0 | *chunk1 | *chunk2 ]
  chunk0: [_ _ W X]   (start_pos=2, so W is at slot 2, X at slot 3)
  chunk1: [Y Z A B]   (full)
  chunk2: [C D _ _]   (end_pos=2, so C at slot 0, D at slot 1)

start_chunk=0, start_pos=2
Logical order: W X Y Z A B C D   (indices 0..7)

deque[5] = ?   (should be 'B')
  absolute_pos = start_pos + 5 = 2 + 5 = 7
  chunk_offset = 7 / 4 = 1         → map[start_chunk + 1] = map[1] = chunk1
  slot_in_chunk = 7 % 4 = 3        → chunk1[3] = B ✓

deque[6] = ?   (should be 'C')
  absolute_pos = 2 + 6 = 8
  chunk_offset = 8 / 4 = 2         → map[2] = chunk2
  slot_in_chunk = 8 % 4 = 0        → chunk2[0] = C ✓
```

---

## 8. The Monotonic Deque — The Killer Pattern

This section deserves the most attention. The monotonic deque is to the deque what the monotonic stack is to the stack — a constrained variant that unlocks a whole class of O(n) solutions to seemingly O(n²) problems.

### What Is a Monotonic Deque?

A deque that maintains its elements in sorted order (either non-increasing or non-decreasing from front to back) by enforcing an invariant on every push:

```
Decreasing Monotonic Deque (front = maximum):
  Before push(x): pop all elements from the BACK that are ≤ x
  Invariant: front > ... > back (non-increasing from front to back)
  Front always holds the maximum of all elements currently in the deque.

Increasing Monotonic Deque (front = minimum):
  Before push(x): pop all elements from the BACK that are ≥ x
  Invariant: front < ... < back (non-decreasing from front to back)
  Front always holds the minimum of all elements currently in the deque.
```

### Why It Gives O(n) for Sliding Window Problems

The naïve approach to sliding window maximum: for each window position, scan all k elements to find the max → O(nk).

The monotonic deque approach:
- Each element is pushed onto the deque exactly once → n push operations
- Each element is popped from the deque at most once → at most n pop operations
- Total work: O(n) across all window positions, regardless of k

This is the same amortised argument as the monotonic stack: each element's lifetime consists of one push and at most one pop.

### Why Store Indices, Not Values?

```
WRONG: store values
  deque stores: [5, 3, 1]
  Problem: when the window slides, you need to evict elements that have
  fallen outside the window. But you cannot tell which value corresponds
  to which position — you lose positional information.

CORRECT: store indices
  deque stores: [2, 5, 7]   (indices into the array)
  Access value: nums[dq.front()] — still O(1)
  Evict check:  dq.front() < window_left → pop_front()
  This cleanly separates "is this index still in the window?" from "what is its value?"
```

### The Complete Sliding Window Maximum Algorithm

```
nums = [1, 3, -1, -3, 5, 3, 6, 7], k = 3

Maintain a decreasing monotonic deque of indices.
Window: [i-k+1, i]. Evict front if out of window.

i=0: val=1  deque=[]       → push 0.           deque=[0]        window<3, no output
i=1: val=3  deque=[0]      → 1>nums[0]=1, pop 0. push 1.        deque=[1]        no output
i=2: val=-1 deque=[1]      → -1≤nums[1]=3, push 2.              deque=[1,2]
     window [0,2] complete. max = nums[dq.front()] = nums[1] = 3  output: 3

i=3: val=-3 deque=[1,2]    → -3≤nums[2]=-1, push 3.             deque=[1,2,3]
     evict? front=1 ≥ 3-3+1=1. No eviction.
     max = nums[1] = 3   output: 3, 3

i=4: val=5  deque=[1,2,3]  → 5>nums[3]=-3, pop 3.
                             5>nums[2]=-1, pop 2.
                             5>nums[1]=3,  pop 1.
                             push 4.                              deque=[4]
     evict? front=4 ≥ 4-3+1=2. No eviction.
     max = nums[4] = 5   output: 3, 3, 5

i=5: val=3  deque=[4]      → 3≤nums[4]=5, push 5.               deque=[4,5]
     evict? front=4 ≥ 5-3+1=3. No eviction.
     max = nums[4] = 5   output: 3, 3, 5, 5

i=6: val=6  deque=[4,5]    → 6>nums[5]=3, pop 5.
                             6>nums[4]=5, pop 4.
                             push 6.                              deque=[6]
     evict? front=6 ≥ 6-3+1=4. No eviction.
     max = nums[6] = 6   output: 3, 3, 5, 5, 6

i=7: val=7  deque=[6]      → 7>nums[6]=6, pop 6. push 7.        deque=[7]
     evict? front=7 ≥ 7-3+1=5. No eviction.
     max = nums[7] = 7   output: 3, 3, 5, 5, 6, 6, 7

Final output: [3, 3, 5, 5, 6, 6, 7] ✓

Total pushes: 8 (one per element). Total pops: 7 (at most one per element).
Total work: O(n). Window size k does NOT appear in the complexity. ✓
```

---

## 9. Interview Problems

### Problem 1: Sliding Window Maximum (The Canonical Monotonic Deque Problem)

**Problem:** Given an array `nums` and an integer `k`, return an array of the maximum value in each sliding window of size `k`.

**Example:** `nums = [1,3,-1,-3,5,3,6,7]`, `k = 3` → `[3,3,5,5,6,6,7]`

**Thought process:**

> "Brute force: for each of the n-k+1 windows, scan k elements for the max → O(nk). For n=10⁵ and k=10⁴, that's 10⁹ operations — too slow."
>
> "I need a structure that gives me the max of a sliding window in O(1) and updates in O(1) amortized as the window slides. The monotonic deque maintains a decreasing sequence — front is always the max. When the window slides: add the new element (popping smaller elements from the back), evict the front if it has slid out of the window."

```cpp
// Time: O(n) | Space: O(k) — deque holds at most k indices
vector<int> maxSlidingWindow(vector<int>& nums, int k) {
    int n = nums.size();
    vector<int> result;
    result.reserve(n - k + 1);

    deque<int> dq;   // stores indices; maintains decreasing order of nums[dq[i]]

    for (int i = 0; i < n; i++) {
        // Step 1: Evict indices that have slid out of the window [i-k+1, i]
        // Only need to check the front (deque is ordered by index)
        while (!dq.empty() && dq.front() < i - k + 1) {
            dq.pop_front();
        }

        // Step 2: Maintain decreasing invariant
        // Pop from back all indices whose values are ≤ nums[i]
        // (they can never be the window max while nums[i] is in the window)
        while (!dq.empty() && nums[dq.back()] <= nums[i]) {
            dq.pop_back();
        }

        dq.push_back(i);

        // Step 3: Record result once the first full window is complete
        if (i >= k - 1) {
            result.push_back(nums[dq.front()]);
        }
    }
    return result;
}
```

**Edge cases:**
- k=1: every element is its own window's max — output equals input
- k=n: one window, max of entire array
- All elements equal: deque never pops (condition `<= nums[i]` pops equal elements; use `<` to keep them but that changes semantics — discuss with interviewer)
- Strictly decreasing array `[5,4,3,2,1]`: every new element is smaller, so nothing is ever popped from the back; the front is evicted every step
- Strictly increasing array `[1,2,3,4,5]`: every new element pops all previous; deque always has exactly one element — the most recent

**Variant — Sliding Window Minimum:** Use an increasing monotonic deque (pop from back when `nums[back] >= nums[i]`).

---

### Problem 2: Shortest Subarray with Sum at Least K (Hard)

**Problem:** Given an integer array `nums` (may contain negatives) and an integer `k`, return the length of the shortest subarray with a sum of at least `k`. Return `-1` if no such subarray exists.

**Why this needs a deque (not a sliding window):** Negative numbers mean the window cannot simply expand/shrink — a longer subarray might have a smaller sum. Sliding window only works for non-negative numbers. This problem requires prefix sums + monotonic deque.

**Thought process:**

> "Brute force: check all O(n²) subarrays — too slow."
>
> "Build prefix sum array P where P[i] = sum of first i elements. Sum of subarray [l,r] = P[r+1] - P[l]. We want the minimum (r-l) such that P[r+1] - P[l] ≥ k, i.e., P[r+1] ≥ P[l] + k."
>
> "Key insight: as we sweep r from left to right, we want to find the largest l < r such that P[l] ≤ P[r+1] - k. If we maintain a monotonic increasing deque of prefix sum indices (front = smallest prefix sum), we can pop from the front while P[front] ≤ P[r+1] - k, recording the minimum length each time."
>
> "After recording, we pop from the front because: if P[front] already satisfies the condition with the current r, it will never give a shorter subarray with a future larger r (since future r only increases r-front)."

```cpp
// Time: O(n) | Space: O(n)
int shortestSubarray(vector<int>& nums, int k) {
    int n = nums.size();

    // Build prefix sums (use long long to avoid overflow)
    vector<long long> prefix(n + 1, 0);
    for (int i = 0; i < n; i++) prefix[i + 1] = prefix[i] + nums[i];

    int result = INT_MAX;
    deque<int> dq;   // increasing monotonic deque of prefix sum indices

    for (int i = 0; i <= n; i++) {
        // Try to find valid left endpoints: prefix[i] - prefix[dq.front()] >= k
        while (!dq.empty() && prefix[i] - prefix[dq.front()] >= k) {
            result = min(result, i - dq.front());
            dq.pop_front();   // pop front — it can't give shorter subarray later
        }

        // Maintain increasing order in deque (remove indices with larger prefix sums)
        // If prefix[dq.back()] >= prefix[i], then dq.back() can never be a better
        // left endpoint than i for any future right endpoint r > i
        while (!dq.empty() && prefix[dq.back()] >= prefix[i]) {
            dq.pop_back();
        }

        dq.push_back(i);
    }

    return result == INT_MAX ? -1 : result;
}

/*
nums = [2, -1, 2], k = 3
prefix = [0, 2, 1, 3]

i=0: dq=[]. push 0.         dq=[0]
i=1: prefix[1]=2. 2-prefix[0]=2<3. No pop. 2>prefix[0]=0? No pop from back. push 1. dq=[0,1]
i=2: prefix[2]=1. 1-0=1<3. No pop. 1<prefix[1]=2? pop 1. dq=[0]. 1>prefix[0]=0? No pop. push 2. dq=[0,2]
i=3: prefix[3]=3. 3-prefix[0]=3>=3 → result=min(INF,3-0)=3. pop 0. dq=[2].
     3-prefix[2]=3-1=2<3. Stop.
     3>prefix[2]=1? No. push 3. dq=[2,3].

result=3. Subarray = nums[0..2] = [2,-1,2], sum=3 ✓

Shortest is 3 (the entire array).
*/
```

**Why pop from front after finding a valid answer?** Once `dq.front()` gives a valid left endpoint for right endpoint `i`, any future right endpoint `j > i` would give a subarray of length `j - dq.front() > i - dq.front()` — longer, not shorter. So `dq.front()` is useless for future queries and can be safely removed.

**Edge cases:**
- All negative numbers + k ≤ 0: any single element suffices — return 1 (check carefully)
- k is very large: prefix sum never reaches k — return -1
- Single element ≥ k: return 1
- Large prefix sums: use `long long` for prefix sum array to prevent overflow

---

### Problem 3: Design a Sliding Window Rate Limiter

**Problem (System Design + Coding):** Design a rate limiter that allows at most `maxReqs` requests per sliding `windowMs` milliseconds. Implement `isAllowed(timestamp)` which returns `true` if the request at the given timestamp is allowed, `false` otherwise.

**Why a deque:** A deque of timestamps maintains the sliding window. Old timestamps are evicted from the front (they have expired). New timestamps are added to the back. Count of elements in the deque is the current request count.

```cpp
#include <deque>

class SlidingWindowRateLimiter {
private:
    deque<long long> window_;   // timestamps of allowed requests
    int     maxReqs_;           // max requests per window
    long long windowMs_;        // window duration in milliseconds

public:
    SlidingWindowRateLimiter(int maxReqs, long long windowMs)
        : maxReqs_(maxReqs), windowMs_(windowMs) {}

    // O(n) amortized (each timestamp enters and leaves the deque exactly once)
    // O(1) in the best case (no evictions needed)
    bool isAllowed(long long timestamp) {
        // Step 1: Evict timestamps outside the current window
        // Window = [timestamp - windowMs_, timestamp]
        while (!window_.empty() &&
               window_.front() <= timestamp - windowMs_) {
            window_.pop_front();
        }

        // Step 2: Check if current window is at capacity
        if ((int)window_.size() >= maxReqs_) {
            return false;   // rate limit exceeded
        }

        // Step 3: Allow request — record its timestamp
        window_.push_back(timestamp);
        return true;
    }

    int currentCount() const { return (int)window_.size(); }
};

/*
Trace: SlidingWindowRateLimiter(3, 1000)   // 3 reqs per 1000ms

isAllowed(100):  window=[100].  size=1<3 → true
isAllowed(200):  window=[100,200]. size=2<3 → true
isAllowed(300):  window=[100,200,300]. size=3 → true (exactly at limit)
isAllowed(400):  evict? 100 ≤ 400-1000=-600? No. size=3 ≥ 3 → false
isAllowed(1100): evict? 100 ≤ 1100-1000=100? Yes, pop 100.
                 evict? 200 ≤ 100? No.
                 window=[200,300,400? No — 400 was rejected].
                 Wait — 400 was rejected, never added.
                 window=[200,300]. size=2<3 → true. window=[200,300,1100]
isAllowed(1200): evict? 200 ≤ 1200-1000=200? Yes, pop 200.
                 window=[300,1100]. size=2<3 → true. window=[300,1100,1200]

Window always contains exactly the timestamps of allowed requests
that fall within [current_time - windowMs_, current_time].
*/

// Enhanced version: with request metadata
struct Request {
    long long timestamp;
    string    user_id;
    string    endpoint;
};

class PerUserRateLimiter {
private:
    unordered_map<string, deque<long long>> userWindows_;
    int      maxReqs_;
    long long windowMs_;

public:
    PerUserRateLimiter(int maxReqs, long long windowMs)
        : maxReqs_(maxReqs), windowMs_(windowMs) {}

    bool isAllowed(const Request& req) {
        auto& window = userWindows_[req.user_id];

        while (!window.empty() &&
               window.front() <= req.timestamp - windowMs_) {
            window.pop_front();
        }

        if ((int)window.size() >= maxReqs_) return false;
        window.push_back(req.timestamp);
        return true;
    }
};
```

**Why a deque and not a circular queue?** The number of requests in the window is unbounded — a burst of traffic could fill any fixed-size circular queue. The deque is unbounded and allows any number of timestamps within the window. The amortised O(1) per operation still holds because each timestamp is added once and removed once.

**Alternative approaches and their tradeoffs:**

| Approach | Time | Space | Notes |
|---|---|---|---|
| Deque of timestamps | O(1) amortized | O(maxReqs) | Exact; simple; used here |
| Fixed window counter | O(1) | O(1) | Inaccurate at window boundaries |
| Token bucket | O(1) | O(1) | Approximate; allows short bursts |
| Sorted set + eviction | O(log n) | O(n) | Exact; supports deletion by ID |

**Edge cases:**
- `maxReqs = 0`: all requests denied — `window.size() >= 0` always true
- Timestamps not in order: if the rate limiter receives out-of-order timestamps, the front-eviction logic breaks — add an assertion or sort requirement
- Same timestamp for multiple requests: all count separately — correct behaviour
- Very long window (`windowMs_ = ∞`): window grows unboundedly — consider a maximum

---

## 10. Real-World Uses

| Domain | Use Case | Why Deque |
|---|---|---|
| **C++ STL** | Default backing for `std::stack` and `std::queue` | O(1) at both ends; no reallocation when growing at front |
| **Browser** | Back/forward tab history | push_front new page; pop_front on forward; pop_back on back |
| **Scheduling** | Work-stealing thread pool (Java ForkJoinPool) | Workers push/pop own tasks from back; steal from other workers' fronts |
| **Networking** | Packet reordering buffer | Out-of-order packets inserted at correct position; in-order packets dequeued from front |
| **Undo/Redo with limit** | Text editor undo stack with max depth | push_back new action; pop_back for undo; pop_front when over the limit |
| **Sliding analytics** | Moving average, rate limiting, anomaly detection | pop_front expired entries; push_back new entries |
| **Game dev** | Replay buffer for replays or AI training | Fixed-size deque; oldest frames dropped from front, newest added to back |
| **Compilers** | Token buffer for look-ahead parsing | Push tokens to back as lexed; pop from front as consumed; push back on unget |
| **A\* pathfinding** | Priority queue with efficient re-ordering | 0-1 BFS uses deque for 0/1-weight edges |
| **Database** | B-tree node split buffer | Temporarily holds keys during node split at either end |

**Work-Stealing Thread Pool (Java ForkJoinPool) — The Deque as a Scheduler:**

This is one of the most sophisticated uses of a deque in production systems:

```
Each worker thread has its own deque of tasks (a "work-stealing deque" / "deque").

Normal execution (no stealing):
  Worker pushes new subtasks to its OWN BACK.
  Worker pops from its OWN BACK (LIFO — better cache locality for subtasks).
  The worker's own deque behaves like a stack.

Stealing (idle worker):
  Idle worker looks at another worker's deque.
  Steals from the FRONT of the victim's deque (FIFO for stolen tasks).
  This takes the OLDEST, largest task — likely at the root of the task tree.
  Minimises coordination: thief touches front; victim touches back. No conflict.

Why a deque and not two separate stacks?
  The stealing is O(1) from the front.
  Using two stacks would require O(n) to reverse the inbox for stealing.
  The deque's O(1) at both ends is exactly what makes work-stealing efficient.

This is used in Java's ForkJoinPool (Java 7+), Rust's Rayon, Intel TBB.
Every time you use parallel streams in Java, a work-stealing deque runs under the hood.
```

---

## 11. Edge Cases & Pitfalls

### Pitfall 1: Using Values Instead of Indices in Monotonic Deque

```cpp
// WRONG: pushing values loses positional information — cannot evict by window position
deque<int> dq;
while (!dq.empty() && nums[i] >= dq.back()) dq.pop_back();
dq.push_back(nums[i]);              // stored VALUE
// Problem: when evicting expired window elements:
if (dq.front() == nums[i - k]) dq.pop_front();  // brittle — fails with duplicates!
// If nums = [3, 3, 3] and k=2, dq.front()==nums[i-k] is true for wrong reasons.

// CORRECT: always push INDICES
while (!dq.empty() && nums[dq.back()] <= nums[i]) dq.pop_back();
dq.push_back(i);
// Evict: if (dq.front() < i - k + 1) dq.pop_front();
// This correctly handles duplicates — index comparison is unambiguous.
```

### Pitfall 2: Wrong Comparison Operator in Monotonic Deque (< vs <=)

```cpp
// For sliding window MAXIMUM, which comparison pops from back?

// Using strict < (keep equal elements):
while (!dq.empty() && nums[dq.back()] < nums[i]) dq.pop_back();
// Effect: equal elements are retained. Multiple copies of the max may be in the deque.
// Correct — harmless, just slightly more elements stored.

// Using <= (pop equal elements):
while (!dq.empty() && nums[dq.back()] <= nums[i]) dq.pop_back();
// Effect: only the most RECENT occurrence of the maximum is kept.
// Also correct, and slightly more memory-efficient.

// The wrong operator to use: > or >= (would maintain INCREASING instead of DECREASING)
// Would give the MINIMUM instead of the maximum — common direction error.
```

### Pitfall 3: Eviction Check Missing or in Wrong Place

```cpp
// The eviction of expired window elements must happen BEFORE reading the max.
// If you read the max first, you might return an expired element's value.

// WRONG order:
result.push_back(nums[dq.front()]);         // read max FIRST
while (dq.front() < i - k + 1) dq.pop_front(); // evict AFTER — too late!

// CORRECT order:
while (!dq.empty() && dq.front() < i - k + 1) dq.pop_front();  // evict FIRST
if (i >= k - 1) result.push_back(nums[dq.front()]);              // then read
```

### Pitfall 4: std::deque Has No reserve() — No Pre-Allocation

```cpp
// std::vector: reserve(n) prevents reallocation
vector<int> v;
v.reserve(1000000);   // single allocation, no future reallocations

// std::deque: NO reserve(), NO capacity()
deque<int> d;
d.reserve(1000000);   // COMPILE ERROR — method does not exist

// Implication: std::deque always allocates chunk by chunk.
// You cannot pre-allocate for performance.
// If you need pre-allocation AND O(1) at both ends, use a circular array deque.
```

### Pitfall 5: Iterator Invalidation After push_front / push_back

```cpp
deque<int> d = {1, 2, 3, 4, 5};
auto it = d.begin() + 2;   // iterator to element 3

d.push_back(6);   // ALL iterators invalidated — map may have reallocated
cout << *it;      // undefined behaviour — iterator may be dangling

// Safe: use indices, not iterators, across modifying operations
int idx = 2;
d.push_back(6);
cout << d[idx];   // safe — references/values at existing positions are stable
```

### Pitfall 6: Confusing pop_front Return with std::queue/stack

```cpp
// std::deque::pop_front() and pop_back() RETURN void
// (same design decision as std::stack::pop and std::queue::pop)

deque<int> d = {1, 2, 3};
int val = d.pop_front();   // COMPILE ERROR — returns void

// Correct:
int val = d.front();
d.pop_front();
```

### Pitfall 7: Using std::deque for Cache-Critical Sequential Processing

```cpp
// WRONG choice: using deque for a tight numerical loop that only needs back access
deque<double> data;
for (double x : input) data.push_back(x);
double sum = 0;
for (double x : data) sum += x;   // chunk-boundary cache misses every 512 bytes

// CORRECT: use vector when you only push/pop at the back
vector<double> data;
data.reserve(input.size());
for (double x : input) data.push_back(x);
double sum = 0;
for (double x : data) sum += x;   // single contiguous block — ~3× faster
```

---

## 12. Comparison

| Feature | Deque | Vector | List | Stack | Queue |
|---|---|---|---|---|---|
| push_front | **O(1) amortized** | O(n) | O(1) | N/A | N/A |
| push_back | **O(1) amortized** | O(1) amortized | O(1) | O(1) | O(1) |
| pop_front | **O(1) amortized** | O(n) | O(1) | N/A | O(1) |
| pop_back | **O(1) amortized** | O(1) | O(1) DLL | O(1) | N/A |
| Random access | **O(1)** 2-ptr | O(1) 1-ptr | O(n) | N/A | N/A |
| Insert middle | O(n) | O(n) | O(1)+search | N/A | N/A |
| Memory layout | Chunked | Contiguous | Scattered nodes | Contiguous | Chunked |
| Cache (sequential) | Good | **Excellent** | Poor | **Excellent** | Good |
| reserve() | No | Yes | No | N/A | No |
| Iterator stability on push | Invalidated | Invalidated | **Stable** | N/A | N/A |
| Reference stability on push | **Stable** | Invalidated | Stable | N/A | N/A |
| Monotonic pattern | ★ Natural | Possible | Poor | For stack variant | For queue variant |
| 0-1 BFS | ★ Natural | No | Poor | No | No |
| Work-stealing | ★ Natural | No | Possible | No | No |

**The decision flowchart:**

```
Need O(1) at both ends?
  YES → Deque or Circular Queue (fixed) or Linked List (unbounded)
  NO  → continue...

Need O(1) only at back?
  YES → Vector (best cache) or Stack
  NO  → continue...

Need O(1) at front and back with FIFO ordering?
  YES → Queue (backed by deque)
  NO  → continue...

Need O(1) middle insert/delete?
  YES → List
  NO  → Vector (default)
```

---

## 13. Self-Test Questions

1. **What is the key difference between a deque and a queue? Between a deque and a stack? What can a deque do that neither can?**

2. **Explain `std::deque`'s chunked array layout. How does `operator[](i)` achieve O(1) access without a single contiguous array? Trace `deque[5]` through the two-level addressing for a deque with chunk_size=4.**

3. **Why does `std::deque` invalidate iterators on `push_back` but NOT references to existing elements? What is the difference?**

4. **In the monotonic deque for sliding window maximum, why do we store indices rather than values? Give a concrete example where storing values would produce a wrong answer.**

5. **Trace the monotonic deque algorithm on `nums = [4, 3, 5, 4, 3, 3, 6, 7]`, `k = 3`. Show the deque state after every element. What is the output?**

6. **In the Shortest Subarray with Sum ≥ K problem, why do we pop from the front after finding a valid answer? What is the invariant being maintained?**

7. **`std::deque` has no `reserve()` while `std::vector` does. What implication does this have for performance-critical code that knows the final size in advance? What would you use instead?**

8. **In a work-stealing thread pool, why does each thread pop from its OWN BACK but steal from OTHER threads' FRONTS? What property of the deque makes this efficient?**

9. **Design a browser history system that supports: `visit(url)`, `back(steps)`, `forward(steps)`. Would you use a deque or two stacks? Justify the choice and show the implementation.**

10. **The sliding window rate limiter uses `pop_front` for eviction and `push_back` for new requests. If requests can arrive out of order (earlier timestamp after a later one), how does this break the algorithm? How would you fix it?**

---

## Quick Reference Card

```
Deque — O(1) at BOTH ends. The most general linear access structure.

Core ADT (all O(1) amortized):
  push_front  pop_front  front()
  push_back   pop_back   back()
  operator[]  size()     empty()

Internal: chunked array ("map" of pointers to fixed-size chunk arrays)
  push_back/front: write to existing chunk or allocate new chunk — O(1) amortized
  operator[i]:     two pointer dereferences (map → chunk → element) — O(1)
  No reserve() — cannot pre-allocate
  Iterator stability: iterators invalidated on push; REFERENCES remain stable

The Monotonic Deque pattern (memorise this):
  // Sliding window maximum (decreasing deque):
  for (int i = 0; i < n; i++) {
      // 1. Evict expired indices from front
      while (!dq.empty() && dq.front() < i - k + 1) dq.pop_front();
      // 2. Maintain decreasing invariant — pop smaller from back
      while (!dq.empty() && nums[dq.back()] <= nums[i]) dq.pop_back();
      dq.push_back(i);
      // 3. Record result
      if (i >= k - 1) result.push_back(nums[dq.front()]);
  }
  Time: O(n). Each element pushed once, popped at most once.
  Always push INDICES, not values.

Pitfall checklist:
  ✗ Storing values (not indices) in monotonic deque
  ✗ Wrong comparison operator (> instead of <= for max deque)
  ✗ Reading max before evicting expired elements
  ✗ Calling reserve() on std::deque (does not exist)
  ✗ Using deque for cache-critical sequential processing (use vector)
  ✗ Holding iterators across push_front/push_back (they are invalidated)
  ✗ Treating pop_front() as returning the value (it returns void)

Use deque when:
  ✓ O(1) push/pop at both ends required
  ✓ Sliding window maximum/minimum (monotonic deque)
  ✓ 0-1 BFS (push_front for weight-0, push_back for weight-1)
  ✓ Rate limiting / sliding window analytics
  ✓ Work-stealing or double-sided scheduling
  ✗ Cache-critical sequential access → use vector
  ✗ Fixed bounded capacity → use circular queue
  ✗ Frequent middle insert/delete → use list
```

---

*Previous: [10 — Circular Queue](./10_circular_queue.md)*
*Next: [12 — Priority Queue](./12_priority_queue.md)*
*See also: [08 — Monotonic Stack](./08_monotonic_stack.md) | [09 — Queue](./09_queue.md) | [Sliding Window algorithms](./algorithms/sliding_window.md)*

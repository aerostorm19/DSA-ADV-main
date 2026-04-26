# Circular Queue

> **Curriculum position:** Linear Structures → #10
> **Interview weight:** ★★★☆☆ Medium — implementation asked directly; underlies ring buffers, streaming, OS scheduling
> **Difficulty:** Beginner-Intermediate — the concept is simple, the index arithmetic has subtle traps
> **Prerequisite:** [09 — Queue](./09_queue.md)

---

## Table of Contents

1. [Intuition First — Why a Queue Needs to Be Circular](#1-intuition-first)
2. [Internal Working — The Ring of Slots](#2-internal-working)
3. [The Full vs Empty Ambiguity — Three Solutions](#3-the-full-vs-empty-ambiguity)
4. [Time & Space Complexity](#4-time--space-complexity)
5. [Complete C++ Implementation](#5-complete-c-implementation)
6. [Core Operations — Visualised](#6-core-operations--visualised)
7. [Common Patterns & Techniques](#7-common-patterns--techniques)
8. [Interview Problems](#8-interview-problems)
9. [Real-World Uses](#9-real-world-uses)
10. [Edge Cases & Pitfalls](#10-edge-cases--pitfalls)
11. [Circular Queue vs Ring Buffer vs Deque](#11-comparison)
12. [Self-Test Questions](#12-self-test-questions)

---

## 1. Intuition First

Recall the problem with a naive array-backed queue: `front` and `tail` indices only ever advance rightward. After enough enqueue/dequeue cycles, `tail` reaches the end of the array and the queue appears full — even though the left portion of the array, vacated by previous dequeues, is entirely unused.

```
After many enqueue/dequeue operations on a naive array:

[  ][  ][  ][  ][  ][ C ][ D ][ E ][  ][  ]
 ↑  ↑  ↑  ↑  ↑  ↑                    ↑
dead space (wasted)  front             tail

tail hits the end → "full" even though 5 slots are free on the left.
```

The fix is conceptually simple: **treat the array as a ring**. When `tail` reaches the last index, wrap it back to index 0. When `front` reaches the last index, wrap it back to 0 as well. The array becomes a circle where freed slots at the front are immediately reusable by the tail.

```
The same array, as a ring:

        0   1   2   3   4   5   6   7   8   9
      [  ][  ][  ][  ][  ][ C ][ D ][ E ][  ][  ]
                            ↑               ↑
                          front            tail

After enqueue(F), enqueue(G), enqueue(H), enqueue(I), enqueue(J):
tail wraps: indices 8, 9, 0, 1, 2

      [ I ][ J ][  ][  ][  ][ C ][ D ][ E ][ F ][ G ]
        ↑           ↑
       tail=2     front=5

The "ring" reuses slots 0 and 1 that were freed by earlier dequeues.
Physical array is unchanged; only the mental model changes.
```

This is a **circular queue** (also called a ring buffer when used for streaming). One fixed-size array. Two indices that wrap using modulo arithmetic. Zero wasted memory. All operations O(1) with no dynamic allocation.

The circular queue is the canonical answer to: *"implement a bounded queue with optimal fixed memory."*

---

## 2. Internal Working — The Ring of Slots

### The Array as a Clock Face

Visualise the array indices arranged around a clock face. `front` is the hour hand pointing to the next element to dequeue. `tail` is the minute hand pointing to the next empty slot to write.

```
           Capacity = 8 slots, arranged as a ring:

                    idx 0
                   [   ]
           idx 7         idx 1
          [   ]            [   ]

     idx 6  [   ]    [   ]  idx 2
              (ring interior)
          [   ]            [   ]
           idx 5         idx 3
                   [   ]
                    idx 4

front and tail both start at 0 (12 o'clock).
Enqueue advances tail clockwise.
Dequeue advances front clockwise.
Queue is empty when front == tail.
Queue is full when (tail + 1) % capacity == front (one gap convention).
```

### The Modulo Formula

Every index advancement uses the same formula:

```
new_index = (old_index + 1) % capacity
```

This single expression handles the wrap-around:
- For indices 0 through capacity-2: `(i + 1) % capacity = i + 1` (normal advance)
- For index capacity-1: `(capacity-1 + 1) % capacity = 0` (wrap to start)

No `if` statement needed. No branch. Pure arithmetic.

### The Three Pointers (with size counter)

```cpp
struct CircularQueue {
    vector<T> data;    // the fixed-size ring
    int front;         // index of the oldest element (next to dequeue)
    int tail;          // index where the next element will be written
    int sz;            // current number of elements (resolves full/empty ambiguity)
    int cap;           // maximum number of elements
};
```

State invariants that must hold at all times:
1. `0 ≤ front < cap`
2. `0 ≤ tail < cap`
3. `0 ≤ sz ≤ cap`
4. The i-th element (0-indexed from front) lives at `data[(front + i) % cap]`
5. `tail == (front + sz) % cap` — tail is always sz steps ahead of front

---

## 3. The Full vs Empty Ambiguity — Three Solutions

This is the central design challenge of circular queues. When `front == tail`, is the queue empty or full? Both states produce the same index relationship. You must choose one of three strategies.

### Strategy A: Size Counter (Recommended)

Maintain an explicit `size` variable alongside `front` and `tail`.

```
Empty: sz == 0
Full:  sz == cap

Advantages:
  + No wasted slot
  + Unambiguous — sz always tells you the truth
  + Simple conditionals
  + Can query size in O(1)

Disadvantage:
  + One extra integer to maintain (trivial cost)
```

```cpp
bool empty() const { return sz == 0; }
bool full()  const { return sz == cap; }
```

### Strategy B: Gap of One (Most Common in Systems)

Reserve one slot as a permanent gap between `tail` and `front`. The array has `cap` slots but stores at most `cap - 1` elements.

```
Empty: front == tail
Full:  (tail + 1) % cap == front   ← one slot always wasted

Capacity = 8 → stores at most 7 elements.

Advantages:
  + No extra variable
  + Simple implementation
  + Used in many embedded / systems implementations

Disadvantage:
  + Wastes one slot (minor)
  + Capacity and storage capacity differ (confusing)
```

```cpp
bool empty() const { return front == tail; }
bool full()  const { return (tail + 1) % cap == front; }
```

### Strategy C: Full Flag

Maintain a boolean `is_full` flag. Toggle it when `front == tail` changes state.

```
Empty: !is_full && front == tail
Full:   is_full && front == tail

Advantages:
  + No wasted slot

Disadvantages:
  + Extra state to update in every enqueue and dequeue
  + Easy to get flag updates wrong
  + Less elegant than size counter
```

**Recommendation:** Use Strategy A (size counter) in interviews and production. It is the clearest, least error-prone, and most flexible. Strategy B is acceptable and widely seen in embedded systems code. Strategy C is rarely worth the complexity.

---

## 4. Time & Space Complexity

### Operation Complexities

| Operation | Time | Notes |
|---|---|---|
| `enqueue(x)` | **O(1)** | Write `data[tail]`, advance tail with modulo |
| `dequeue()` | **O(1)** | Read `data[front]`, advance front with modulo |
| `front()` / `peek()` | **O(1)** | Direct index access `data[front]` |
| `back()` | **O(1)** | `data[(tail - 1 + cap) % cap]` |
| `empty()` | **O(1)** | Check `sz == 0` |
| `full()` | **O(1)** | Check `sz == cap` |
| `size()` | **O(1)** | Return `sz` |
| Access index i | **O(1)** | `data[(front + i) % cap]` — random access possible! |
| Search | **O(n)** | Traverse all occupied slots |
| Clear | **O(1)**\* | Reset front, tail, sz — elements not destructed |
| Resize | **O(n)** | Allocate new array, copy in order, remap indices |

\*O(1) for primitive types; O(n) if elements have non-trivial destructors that must be called.

### Space Complexity

| | |
|---|---|
| Total allocation | **O(capacity)** — fixed, determined at construction |
| Per-element overhead | **0 bytes** — pure value storage, no pointers |
| Bookkeeping | **3 integers** (front, tail, sz/cap) — O(1) |
| Comparison to linked queue | Linked queue: 8 bytes overhead per element; circular queue: 0 |
| Memory fragmentation | **None** — single contiguous allocation |

The circular queue has the lowest memory overhead of any queue implementation. The entire structure is one contiguous block — optimal for cache behaviour.

---

## 5. Complete C++ Implementation

### Production-Quality Generic Circular Queue

```cpp
#include <iostream>
#include <vector>
#include <stdexcept>
#include <initializer_list>
using namespace std;

template<typename T>
class CircularQueue {
private:
    vector<T> data_;     // fixed-size ring storage
    int front_;          // index of next element to dequeue
    int tail_;           // index where next element will be written
    int size_;           // current number of elements
    int capacity_;       // maximum number of elements

    // Advance an index by one step with wrap-around
    int advance(int idx) const {
        return (idx + 1) % capacity_;
    }

    // Retreat an index by one step with wrap-around (for back())
    int retreat(int idx) const {
        return (idx - 1 + capacity_) % capacity_;
    }

public:
    // ── Construction ──────────────────────────────────────────────────────────

    explicit CircularQueue(int capacity)
        : data_(capacity), front_(0), tail_(0), size_(0), capacity_(capacity) {
        if (capacity <= 0) throw invalid_argument("capacity must be positive");
    }

    CircularQueue(int capacity, initializer_list<T> vals)
        : CircularQueue(capacity) {
        for (const T& v : vals) enqueue(v);
    }

    // ── State Queries ─────────────────────────────────────────────────────────

    bool empty()    const { return size_ == 0; }
    bool full()     const { return size_ == capacity_; }
    int  size()     const { return size_; }
    int  capacity() const { return capacity_; }

    // ── Element Access ────────────────────────────────────────────────────────

    // O(1) — peek at front (oldest element)
    T& front() {
        if (empty()) throw underflow_error("front() on empty queue");
        return data_[front_];
    }
    const T& front() const {
        if (empty()) throw underflow_error("front() on empty queue");
        return data_[front_];
    }

    // O(1) — peek at back (newest element)
    T& back() {
        if (empty()) throw underflow_error("back() on empty queue");
        return data_[retreat(tail_)];
    }
    const T& back() const {
        if (empty()) throw underflow_error("back() on empty queue");
        return data_[retreat(tail_)];
    }

    // O(1) — random access by logical index (0 = front, size-1 = back)
    // This is unique to circular queue vs std::queue — std::queue has no operator[]
    T& operator[](int i) {
        if (i < 0 || i >= size_) throw out_of_range("index out of range");
        return data_[(front_ + i) % capacity_];
    }
    const T& operator[](int i) const {
        if (i < 0 || i >= size_) throw out_of_range("index out of range");
        return data_[(front_ + i) % capacity_];
    }

    // ── Modifiers ─────────────────────────────────────────────────────────────

    // O(1) — add element to the back
    void enqueue(const T& val) {
        if (full()) throw overflow_error("enqueue on full queue");
        data_[tail_] = val;
        tail_ = advance(tail_);
        size_++;
    }

    void enqueue(T&& val) {
        if (full()) throw overflow_error("enqueue on full queue");
        data_[tail_] = move(val);
        tail_ = advance(tail_);
        size_++;
    }

    // O(1) — remove element from the front and return it
    T dequeue() {
        if (empty()) throw underflow_error("dequeue on empty queue");
        T val = move(data_[front_]);
        front_ = advance(front_);
        size_--;
        return val;
    }

    // O(1) reset — does not call destructors for primitive types
    void clear() {
        front_ = tail_ = size_ = 0;
    }

    // ── Overwrite Mode (Ring Buffer Behaviour) ────────────────────────────────
    // Used when the queue should act as a ring buffer: overwrite the oldest
    // element when full instead of throwing. Essential for streaming use cases.
    void enqueue_overwrite(const T& val) {
        if (full()) {
            // Overwrite oldest: advance front to discard it
            front_ = advance(front_);
            size_--;
        }
        data_[tail_] = val;
        tail_ = advance(tail_);
        size_++;
    }

    // ── Resize (O(n)) ─────────────────────────────────────────────────────────
    // Grows or shrinks the queue. Elements are re-linearised in the new array.
    void resize(int new_capacity) {
        if (new_capacity < size_)
            throw invalid_argument("new capacity too small for current elements");

        vector<T> new_data(new_capacity);
        for (int i = 0; i < size_; i++) {
            new_data[i] = move(data_[(front_ + i) % capacity_]);
        }
        data_     = move(new_data);
        front_    = 0;
        tail_     = size_;          // tail is exactly size_ slots from front_=0
        capacity_ = new_capacity;
    }

    // ── Utilities ─────────────────────────────────────────────────────────────

    void print() const {
        if (empty()) { cout << "(empty circular queue)\n"; return; }
        cout << "front → ";
        for (int i = 0; i < size_; i++) {
            cout << data_[(front_ + i) % capacity_];
            if (i < size_ - 1) cout << " → ";
        }
        cout << " ← back   [size=" << size_ << " cap=" << capacity_
             << " front=" << front_ << " tail=" << tail_ << "]\n";
    }

    // Print the raw array including empty slots (for debugging wrap-around)
    void print_raw() const {
        cout << "raw: [";
        for (int i = 0; i < capacity_; i++) {
            bool occupied = false;
            // Check if index i is an occupied slot
            if (size_ > 0) {
                if (front_ < tail_) {
                    occupied = (i >= front_ && i < tail_);
                } else if (front_ > tail_) {
                    occupied = (i >= front_ || i < tail_);
                } else {
                    occupied = false;  // front == tail and size > 0: full
                    if (full()) occupied = true;
                }
            }
            cout << (occupied ? to_string(data_[i]) : "_");
            if (i < capacity_ - 1) cout << "|";
        }
        cout << "]  front=" << front_ << " tail=" << tail_ << "\n";
    }
};
```

### LeetCode-Style Design Circular Queue

```cpp
// Matches LeetCode #622 exactly — int-specialised, gap-of-one strategy
class MyCircularQueue {
private:
    vector<int> data_;
    int front_, tail_, cap_;

public:
    explicit MyCircularQueue(int k)
        : data_(k + 1), front_(0), tail_(0), cap_(k + 1) {}
    //              ^                                  ^
    //        allocate k+1 slots              cap = k+1 for gap-of-one

    bool isEmpty()  const { return front_ == tail_; }
    bool isFull()   const { return (tail_ + 1) % cap_ == front_; }

    bool enQueue(int value) {
        if (isFull()) return false;
        data_[tail_] = value;
        tail_ = (tail_ + 1) % cap_;
        return true;
    }

    bool deQueue() {
        if (isEmpty()) return false;
        front_ = (front_ + 1) % cap_;
        return true;
    }

    int Front() const {
        if (isEmpty()) return -1;
        return data_[front_];
    }

    int Rear() const {
        if (isEmpty()) return -1;
        return data_[(tail_ - 1 + cap_) % cap_];  // one step behind tail_
    }
};
```

### Lock-Free Circular Queue (Single-Producer Single-Consumer)

```cpp
// The SPSC (Single Producer, Single Consumer) circular queue
// Used in embedded systems, audio engines, and OS kernel ring buffers.
// No mutex needed — atomic operations only.

#include <atomic>
#include <vector>

template<typename T>
class SPSCQueue {
private:
    vector<T>          data_;
    int                capacity_;
    atomic<int>        head_{0};   // consumer reads from here
    atomic<int>        tail_{0};   // producer writes to here

public:
    explicit SPSCQueue(int capacity)
        : data_(capacity + 1), capacity_(capacity + 1) {}

    // Called by PRODUCER thread only
    bool push(const T& val) {
        int tail = tail_.load(memory_order_relaxed);
        int next = (tail + 1) % capacity_;
        if (next == head_.load(memory_order_acquire)) return false;  // full
        data_[tail] = val;
        tail_.store(next, memory_order_release);
        return true;
    }

    // Called by CONSUMER thread only
    bool pop(T& val) {
        int head = head_.load(memory_order_relaxed);
        if (head == tail_.load(memory_order_acquire)) return false;  // empty
        val = data_[head];
        head_.store((head + 1) % capacity_, memory_order_release);
        return true;
    }

    bool empty() const {
        return head_.load(memory_order_acquire) ==
               tail_.load(memory_order_acquire);
    }
};

// Why no mutex?
// In SPSC: head is only written by the consumer, tail is only written by producer.
// memory_order_acquire/release pairs ensure the data write (push) is visible
// before the tail update, and the data read (pop) sees the committed write.
// This is safe without locks because there is exactly ONE writer per index.
```

---

## 6. Core Operations — Visualised

### Enqueue and Dequeue with Wrap-Around

```
Capacity = 6 (using size counter strategy)

Step 1 — Initial state:
  data: [ _ | _ | _ | _ | _ | _ ]
          0   1   2   3   4   5
  front=0, tail=0, size=0

Step 2 — enqueue(A, B, C, D):
  data: [ A | B | C | D | _ | _ ]
          0   1   2   3   4   5
  front=0, tail=4, size=4

Step 3 — dequeue(), dequeue():  (remove A, then B)
  data: [ _ | _ | C | D | _ | _ ]
          0   1   2   3   4   5
               front=2, tail=4, size=2
  (slots 0 and 1 are now free but front cannot "go back")

Step 4 — enqueue(E, F, G):  (tail wraps around)
  enqueue(E): data[4]=E, tail=(4+1)%6=5
  enqueue(F): data[5]=F, tail=(5+1)%6=0   ← WRAP
  enqueue(G): data[0]=G, tail=(0+1)%6=1   ← reusing slot 0

  data: [ G | _ | C | D | E | F ]
          0   1   2   3   4   5
         tail=1       front=2, size=5

Step 5 — dequeue(): removes C (front=2)
  front = (2+1)%6 = 3, size=4

  data: [ G | _ | _ | D | E | F ]
          0   1   2   3   4   5
         tail=1      front=3, size=4

Logical queue order: D → E → F → G
Physical array:      [G][_][_][D][E][F]
                      ↑           ↑
                     tail=1     front=3

The physical layout is NOT the same as the logical order.
Always traverse using: data[(front + i) % capacity] for i in 0..size-1
```

### The `back()` Formula

```
tail_ points to the NEXT WRITE SLOT (one past the last element).
The last written element is one step BEHIND tail_.

back() = data_[(tail_ - 1 + capacity_) % capacity_]
                          ↑
                    +capacity_ prevents negative modulo
                    (-1 + 6) % 6 = 5  ✓  when tail_=0 (wrapped)
                    (4  - 1) % 6 = 3  ✓  when tail_=4 (normal)
```

### Resize — The One O(n) Operation

```
Before resize (capacity=6, front=3, tail=1, size=4):
  physical: [ G | _ | _ | D | E | F ]
  logical:    D → E → F → G   (front=3, reads at indices 3,4,5,0)

After resize to capacity=10 (re-linearise):
  Copy logical order to new array starting at index 0:
  new_data[0] = data[(3+0)%6] = data[3] = D
  new_data[1] = data[(3+1)%6] = data[4] = E
  new_data[2] = data[(3+2)%6] = data[5] = F
  new_data[3] = data[(3+3)%6] = data[0] = G

  physical: [ D | E | F | G | _ | _ | _ | _ | _ | _ ]
  front=0, tail=4, capacity=10

  Clean layout: front always at 0 after resize.
```

---

## 7. Common Patterns & Techniques

### Pattern 1: Sliding Window with a Circular Queue

Maintain the last k elements for running statistics (average, max, min). The circular queue's O(1) enqueue and O(1) dequeue of the oldest element makes it ideal.

```cpp
// Moving average of a stream of integers over a window of size k
class MovingAverage {
private:
    CircularQueue<int> window_;
    long long sum_;

public:
    explicit MovingAverage(int size) : window_(size), sum_(0) {}

    double next(int val) {
        if (window_.full()) {
            sum_ -= window_.front();     // remove oldest from sum
            window_.dequeue();
        }
        window_.enqueue(val);
        sum_ += val;
        return static_cast<double>(sum_) / window_.size();
    }
};

/*
MovingAverage(3):
  next(1): window=[1],     sum=1,  avg=1.0
  next(10): window=[1,10], sum=11, avg=5.5
  next(3): window=[1,10,3], sum=14, avg=4.67
  next(5): full! remove 1. window=[10,3,5], sum=17, avg=5.67
*/
```

### Pattern 2: Ring Buffer for Streaming (Overwrite Mode)

In streaming contexts (audio, video, network), a full buffer should overwrite the oldest data rather than blocking or throwing.

```cpp
// Producer-consumer with overwrite: newest data always preserved
// Oldest data silently dropped when buffer is full
class StreamBuffer {
private:
    CircularQueue<float> buf_;
    int overwritten_ = 0;

public:
    explicit StreamBuffer(int capacity) : buf_(capacity) {}

    void write(float sample) {
        if (buf_.full()) {
            buf_.dequeue();  // drop oldest
            overwritten_++;
        }
        buf_.enqueue(sample);
    }

    float read() {
        if (buf_.empty()) return 0.0f;
        return buf_.dequeue();
    }

    int dropped() const { return overwritten_; }
};
```

### Pattern 3: Round-Robin with a Circular Queue

The circular queue naturally models round-robin scheduling — dequeue the front, process, re-enqueue at the back.

```cpp
// Round-robin task scheduler: each task gets one "quantum" of time
void roundRobin(vector<pair<string, int>>& tasks, int quantum) {
    // tasks: {name, remaining_time}
    CircularQueue<pair<string,int>> q(tasks.size());
    for (auto& t : tasks) q.enqueue(t);

    int time = 0;
    while (!q.empty()) {
        auto [name, remaining] = q.dequeue();
        int run = min(remaining, quantum);
        time += run;
        remaining -= run;
        cout << "t=" << time << ": " << name
             << (remaining > 0 ? " (preempted)" : " (done)") << "\n";
        if (remaining > 0) q.enqueue({name, remaining});  // re-enqueue if not done
    }
}
```

---

## 8. Interview Problems

### Problem 1: Design Circular Queue (LeetCode #622)

**Problem:** Implement `MyCircularQueue` with the following operations:
- `MyCircularQueue(k)` — initialise with capacity k
- `enQueue(value)` — insert to the rear; return `false` if full
- `deQueue()` — delete from the front; return `false` if empty
- `Front()` — get the front element; return `-1` if empty
- `Rear()` — get the rear element; return `-1` if empty
- `isEmpty()` — return `true` if empty
- `isFull()` — return `true` if full

All operations must be **O(1)**.

**Thought process:**

> "Fixed capacity, O(1) all operations — circular array is the right structure. The core question is how to distinguish full from empty when `front == tail`. I'll use the gap-of-one strategy: allocate k+1 slots, waste one slot as the gap. Full when `(tail + 1) % (k+1) == front`. Empty when `front == tail`."

```cpp
class MyCircularQueue {
private:
    vector<int> data_;
    int front_;
    int tail_;
    int cap_;   // = k + 1 (one extra for gap-of-one strategy)

public:
    explicit MyCircularQueue(int k)
        : data_(k + 1, 0), front_(0), tail_(0), cap_(k + 1) {}

    bool enQueue(int value) {
        if (isFull()) return false;
        data_[tail_] = value;
        tail_ = (tail_ + 1) % cap_;
        return true;
    }

    bool deQueue() {
        if (isEmpty()) return false;
        front_ = (front_ + 1) % cap_;
        return true;
    }

    int Front() const {
        return isEmpty() ? -1 : data_[front_];
    }

    int Rear() const {
        // tail_ points to the next write slot; one step back is the last element
        return isEmpty() ? -1 : data_[(tail_ - 1 + cap_) % cap_];
    }

    bool isEmpty() const { return front_ == tail_; }
    bool isFull()  const { return (tail_ + 1) % cap_ == front_; }
};

/*
MyCircularQueue(3): cap=4, data=[0,0,0,0], front=0, tail=0

enQueue(1): data=[1,0,0,0], tail=1 → true
enQueue(2): data=[1,2,0,0], tail=2 → true
enQueue(3): data=[1,2,3,0], tail=3 → true
enQueue(4): isFull() → (3+1)%4=0==front=0 → true → return false
Rear():  data[(3-1+4)%4] = data[2] = 3 ✓
deQueue(): front=(0+1)%4=1 → true
enQueue(4): data=[1,2,3,4], tail=0 → true
                                  ↑ tail wraps to 0
Front(): data[front=1] = 2 ✓
Rear():  data[(0-1+4)%4] = data[3] = 4 ✓
*/
```

**Edge cases:**
- k=1: capacity=2 (one real slot + one gap); can hold exactly 1 element
- Enqueue on full: return false — never throw in this design
- Dequeue on empty: return false
- Rear() when tail=0: `(0 - 1 + cap_) % cap_ = cap_ - 1` — the last slot

---

### Problem 2: Design Circular Deque (LeetCode #641)

**Problem:** Design a circular double-ended queue (deque) where insertions and deletions are O(1) at both the front and the back. Capacity is fixed at construction.

**Why this tests circular queue mastery:** You must support push and pop at BOTH ends. The modulo arithmetic for `front` must now go in BOTH directions — forward for `push_back`/`pop_front` and backward for `push_front`/`pop_back`.

```cpp
class MyCircularDeque {
private:
    vector<int> data_;
    int front_;    // index of the front element
    int back_;     // index of the back element (NOT one-past-end — convention change!)
    int size_;
    int cap_;

    int advance(int i) const { return (i + 1) % cap_; }
    int retreat(int i) const { return (i - 1 + cap_) % cap_; }

public:
    explicit MyCircularDeque(int k)
        : data_(k), front_(0), back_(k - 1), size_(0), cap_(k) {
        // Convention: front_ and back_ point TO elements (not past-end).
        // Empty: size_ == 0. Full: size_ == cap_.
        // Initial back_ = cap_-1 so that insertFront(x) writes to data_[0].
    }

    bool insertFront(int value) {
        if (isFull()) return false;
        front_ = retreat(front_);   // move front one step back
        data_[front_] = value;
        size_++;
        return true;
    }

    bool insertLast(int value) {
        if (isFull()) return false;
        back_ = advance(back_);     // move back one step forward
        data_[back_] = value;
        size_++;
        return true;
    }

    bool deleteFront() {
        if (isEmpty()) return false;
        front_ = advance(front_);   // abandon the front slot
        size_--;
        return true;
    }

    bool deleteLast() {
        if (isEmpty()) return false;
        back_ = retreat(back_);     // abandon the back slot
        size_--;
        return true;
    }

    int getFront() const { return isEmpty() ? -1 : data_[front_]; }
    int getRear()  const { return isEmpty() ? -1 : data_[back_]; }
    bool isEmpty() const { return size_ == 0; }
    bool isFull()  const { return size_ == cap_; }
};

/*
MyCircularDeque(3): cap=3, front=0, back=2, size=0

insertLast(1):  back=advance(2)=0, data[0]=1, size=1 → true
insertLast(2):  back=advance(0)=1, data[1]=2, size=2 → true
insertFront(3): front=retreat(0)=2, data[2]=3, size=3 → true
  data=[1,2,_], actually data=[1,2,3]
  front=2, back=1, size=3
  Logical order (from front): data[2]=3, data[0]=1, data[1]=2

getFront(): data[front=2] = 3  ✓
getRear():  data[back=1]  = 2  ✓
deleteLast(): back=retreat(1)=0, size=2
insertFront(4): front=retreat(2)=1, data[1]=4, size=3 → true
  data=[1,4,3], front=1, back=0
  Logical order: data[1]=4, data[2]=3, data[0]=1
getFront(): data[1] = 4  ✓
*/
```

**The key insight:** When you think of `front` and `back` as pointing TO elements (not past-end), the convention for `insertFront` becomes "retreat first, then write" and `insertLast` becomes "advance first, then write." Deletions just move the pointer without writing. The size counter cleanly handles empty/full without the gap.

---

### Problem 3: Task Scheduler with Cooling Time

**Problem:** Given a list of CPU tasks (characters `'A'` to `'Z'`), where each task takes 1 unit of time, and a non-negative integer `n` representing the cooldown period between identical tasks — find the minimum time to complete all tasks. The CPU can be idle during cooling.

**Example:** tasks=`['A','A','A','B','B','B']`, n=2 → output: 8

**Why a circular queue is relevant here:**

The tasks cycle in a round-robin fashion across `n+1` "slots" in each cycle. A circular queue with capacity `n+1` models each cycle naturally — fill the queue with the most frequent tasks for this cycle (or idle if no task available), then advance.

**Thought process:**

> "In each cycle of length `n+1`, the most frequent remaining task must go first to minimise idle time. Greedy: always schedule the top `n+1` most frequent tasks per cycle (sorted by frequency descending). If fewer than `n+1` task types remain, the rest of the cycle is idle. Count total cycles × (n+1), but subtract the tail (last cycle may be shorter)."

```cpp
// Time: O(n log n) | Space: O(1) — at most 26 task types
int leastInterval(vector<char>& tasks, int n) {
    // Count frequency of each task
    int freq[26] = {};
    for (char t : tasks) freq[t - 'A']++;
    sort(begin(freq), end(freq));           // ascending sort

    int maxFreq = freq[25];                 // highest frequency task
    // Count how many tasks share the maximum frequency
    int maxCount = 0;
    for (int f : freq) if (f == maxFreq) maxCount++;

    // Formula derivation:
    // maxFreq-1 "full" cycles each of length n+1, plus a final partial cycle
    // Final cycle has exactly maxCount tasks (all max-frequency tasks)
    //
    // Example: AAABBB, n=2
    //   maxFreq=3, maxCount=2
    //   Arrangement: [A B _][A B _][A B]
    //   Time = (3-1)*(2+1) + 2 = 8
    //
    // But if tasks are numerous enough, no idle time is needed:
    //   Time = tasks.size() (every slot is filled)

    int time = (maxFreq - 1) * (n + 1) + maxCount;
    return max(time, (int)tasks.size());
}

/*
AAABBB, n=2:
  freq: A=3, B=3, others=0
  maxFreq=3, maxCount=2
  time = (3-1)*(2+1) + 2 = 4+4 = 8
  max(8, 6) = 8 ✓

  Actual schedule:
  Cycle 1: A B idle    (t=1,2,3)
  Cycle 2: A B idle    (t=4,5,6)
  Cycle 3: A B         (t=7,8)    ← partial cycle, no idle needed

AAABBBCCCD, n=2:
  A=3, B=3, C=3, D=1
  maxFreq=3, maxCount=3
  time = (3-1)*3 + 3 = 9
  max(9, 10) = 10 ✓  (10 tasks, enough variety, no idle)
*/

// Alternative: Priority Queue + Circular Queue simulation
// More intuitive, same complexity — useful when you need the actual schedule
int leastIntervalSimulation(vector<char>& tasks, int n) {
    int freq[26] = {};
    for (char t : tasks) freq[t - 'A']++;

    priority_queue<int> pq;           // max-heap of frequencies
    for (int f : freq) if (f > 0) pq.push(f);

    int time = 0;
    while (!pq.empty()) {
        vector<int> cycle;            // tasks scheduled in this cycle of n+1 slots
        int slots = n + 1;

        while (slots > 0 && !pq.empty()) {
            cycle.push_back(pq.top() - 1);   // schedule this task, decrement count
            pq.pop();
            slots--;
        }

        // Re-enqueue tasks that still have remaining count
        for (int remaining : cycle) {
            if (remaining > 0) pq.push(remaining);
        }

        // If queue is now empty, final cycle may be shorter
        time += pq.empty() ? (int)cycle.size() : n + 1;
    }
    return time;
}
```

**Edge cases:**
- n=0: no cooldown, all tasks can be scheduled back-to-back → answer is `tasks.size()`
- All tasks identical (e.g., `['A','A','A']`, n=2): answer is `(3-1)*3 + 1 = 7`
- Only one task type: answer is `count + (count-1)*n`
- Many different task types: cooling never forces idle → answer is `tasks.size()`

---

## 9. Real-World Uses

| Domain | Use Case | Details |
|---|---|---|
| **OS kernel** | CPU scheduling (round-robin) | Each process gets a time quantum; circular queue of ready processes |
| **Audio engines** | PCM audio ring buffer | Producer (codec) writes; consumer (soundcard) reads; fixed latency |
| **Network I/O** | NIC receive ring (Linux `sk_buff` ring) | DMA writes packets to ring; kernel reads and processes them |
| **Video streaming** | Frame buffer between decoder and renderer | Circular queue of decoded frames; renderer reads at display rate |
| **Serial communication** | UART receive buffer | Interrupt handler writes bytes; application reads; fixed buffer |
| **Embedded RTOS** | Task message queues (FreeRTOS `xQueueCreate`) | Inter-task communication with fixed memory, no heap allocation |
| **Database** | Write-ahead log (WAL) buffer | Transactions written to ring; flushed to disk when full or on commit |
| **Browser** | Event loop task queue (JavaScript) | Macrotasks in a FIFO ring; microtasks have their own queue |
| **Trading systems** | Low-latency order book update ring | Lock-free SPSC ring between market feed thread and strategy thread |
| **Logging** | High-throughput lock-free log ring | Producers write log entries; background thread drains to disk |

**Linux Network Stack — The NIC Ring Buffer in Detail:**

```
Network card (NIC) hardware:
  TX ring (transmit): kernel writes packets here; NIC DMA-reads and sends
  RX ring (receive):  NIC DMA-writes incoming packets; kernel reads and processes

Structure of one ring descriptor:
  [buffer_address | length | status_flags | vlan_tag | ... ]
  Each descriptor points to a pre-allocated packet buffer.

Why circular and not a linked list?
  1. DMA operates on contiguous physical addresses — a ring array is contiguous
  2. No pointer chasing — the NIC's DMA engine advances its own index
  3. Fixed size — NIC hardware has a fixed number of descriptors (e.g., 4096)
  4. Lock-free: producer (NIC) and consumer (kernel) use separate indices

At 10 Gbps: ~14.88 million 64-byte packets per second.
Each packet = one ring slot. Ring must be emptied every ~67 microseconds.
Any lock or heap allocation in the hot path would miss this deadline.
The circular queue's O(1) with zero allocation is not a design preference —
it is a hard requirement.
```

**JavaScript Event Loop — A Circular Queue in Disguise:**

```
The browser's event loop processes macrotasks (setTimeout, I/O, UI events)
from a FIFO queue. The queue is implemented as a circular buffer in V8/SpiderMonkey.

  while (true) {
      task = macrotask_queue.dequeue()  // FIFO circular queue
      execute(task)
      drain(microtask_queue)            // Promise callbacks, MutationObserver
      render_if_needed()
  }

Why circular and not a linked list?
  - Tasks are short-lived and numerous — heap allocation per task is too expensive
  - Fixed maximum queue depth is acceptable (tasks are throttled by execution time)
  - Contiguous storage improves CPU cache hit rate in the hot dequeue path
```

---

## 10. Edge Cases & Pitfalls

### Pitfall 1: The Negative Modulo Trap

```cpp
// When computing back() = (tail_ - 1) % capacity_:
// In C++, if tail_ == 0, then (0 - 1) % capacity_ is implementation-defined
// and typically returns -1 on most systems (negative result).

// WRONG:
int back_idx = (tail_ - 1) % capacity_;   // -1 when tail_ = 0 → data[-1] = UB!

// CORRECT: add capacity_ before taking modulo to ensure non-negative result
int back_idx = (tail_ - 1 + capacity_) % capacity_;   // always in [0, capacity_-1]
//                          ↑
//              This +capacity_ is the fix. Forget it and you get UB.
```

### Pitfall 2: Confusing Physical Index with Logical Index

```cpp
// Physical layout after wrap-around:
// data = [E | F | G | _ | B | C | D]
//         0   1   2   3   4   5   6
//                         ↑
//                       front=4

// WRONG: iterating physical indices 0..size-1
for (int i = 0; i < size; i++) cout << data[i];   // prints E F G _ B — garbage!

// CORRECT: iterate logical indices and map to physical
for (int i = 0; i < size; i++) {
    cout << data[(front + i) % capacity];   // prints B C D E F G ✓
}
```

### Pitfall 3: Off-by-One in Gap-of-One Strategy

```cpp
// You want a queue of capacity k. With gap-of-one, allocate k+1 slots.
// If you allocate k slots and use gap-of-one, actual capacity is k-1.

// WRONG: allocate k slots for a queue of capacity k with gap-of-one
MyCircularQueue(int k) : data_(k), cap_(k) {}
// This queue holds at most k-1 elements! Off by one.

// CORRECT:
MyCircularQueue(int k) : data_(k + 1), cap_(k + 1) {}
// cap_ = k+1, actual capacity = cap_-1 = k ✓

// Or use size counter and allocate exactly k:
MyCircularQueue(int k) : data_(k), cap_(k), size_(0) {}
// isFull() checks size_ == cap_, so full at exactly k elements ✓
```

### Pitfall 4: Using `int` for Indices on Large Queues

```cpp
// If capacity > INT_MAX / 2, modulo arithmetic on int overflows.
// (tail + 1) overflows before the % is applied.

// For large queues, use size_t or int64_t:
size_t front_ = 0, tail_ = 0;
size_t advance(size_t i) const { return (i + 1) % (size_t)capacity_; }

// In practice, circular queues rarely exceed millions of elements,
// so int is fine for most use cases. Be aware for hardware-level rings.
```

### Pitfall 5: Not Re-Linearising During Resize

```cpp
// WRONG: just extend the array without moving elements
void resize_wrong(int new_cap) {
    data_.resize(new_cap);
    capacity_ = new_cap;
    // front_ and tail_ are unchanged — correct IF front_ < tail_
    // But if the queue is wrapped (tail_ < front_), the logical order is broken:
    // elements that should be contiguous are now split by uninitialised slots
}

// CORRECT: copy in logical order to a fresh array
void resize(int new_cap) {
    vector<T> new_data(new_cap);
    for (int i = 0; i < size_; i++)
        new_data[i] = move(data_[(front_ + i) % capacity_]);
    data_     = move(new_data);
    front_    = 0;
    tail_     = size_;    // always (size_ ≤ new_cap is guaranteed by caller)
    capacity_ = new_cap;
}
```

### Pitfall 6: Thread Safety in Multi-Producer/Consumer Scenarios

```cpp
// The SPSC (single producer, single consumer) circular queue is lock-free.
// MPSC (multi-producer) or MPMC (multi-consumer) queues REQUIRE synchronisation.

// WRONG for MPMC: two consumers read head_ simultaneously
bool pop(T& val) {
    int head = head_;                    // both threads read same value
    if (head == tail_) return false;
    val = data_[head];                   // both read the same element!
    head_ = (head + 1) % capacity_;     // one write wins, other is lost
    return true;
}

// For MPMC: use mutex, or atomic compare-and-swap (CAS):
bool pop_mpmc(T& val) {
    int head, next;
    do {
        head = head_.load(memory_order_relaxed);
        if (head == tail_.load(memory_order_acquire)) return false;
        next = (head + 1) % capacity_;
    } while (!head_.compare_exchange_weak(head, next,
                 memory_order_release, memory_order_relaxed));
    val = data_[head];
    return true;
}
```

### Pitfall 7: Dequeue Returns a Copy, Not a Reference

```cpp
// WRONG: taking a reference to data that is about to be "freed"
T& dequeue_ref() {
    T& ref = data_[front_];
    front_ = (front_ + 1) % capacity_;
    size_--;
    return ref;  // dangling — the slot may be overwritten by next enqueue!
}

// CORRECT: return by value (or move)
T dequeue() {
    T val = move(data_[front_]);
    front_ = (front_ + 1) % capacity_;
    size_--;
    return val;
}
```

---

## 11. Comparison

| Feature | Circular Queue | Linear Queue (naive array) | Linked Queue | `std::queue` (deque) |
|---|---|---|---|---|
| Enqueue | O(1) | O(1) until full | O(1) | O(1) amortised |
| Dequeue | O(1) | O(1) (front index) | O(1) | O(1) |
| Memory reuse | ✓ Slots reused | ✗ Dead space grows | ✓ Node freed | ✓ Chunk freed |
| Fixed capacity | ✓ (set at construction) | Sort of (array size) | ✗ Unbounded | ✗ Unbounded |
| Per-element overhead | 0 bytes | 0 bytes | 8 bytes (pointer) | ~0 amortised |
| Cache behaviour | ★★★ Excellent | ★★★ Excellent | ★ Poor | ★★ Good |
| Random access | ✓ O(1) via `(front+i)%cap` | ✓ O(1) | ✗ O(n) | ✗ O(1) only front/back |
| Lock-free possible | ✓ (SPSC) | Partially | ✗ (allocation) | ✗ |
| Resize | O(n) re-linearise | N/A | N/A | O(n) automatic |
| Use in hardware (DMA) | ✓ Contiguous | ✓ Contiguous | ✗ Scattered | ✗ Scattered |
| Suitable for RTOS | ✓ No heap allocation | ✓ | ✗ | ✗ |

**When to choose circular queue:**
- Fixed capacity is acceptable or required
- Zero per-element overhead is important (embedded, kernel, high-frequency trading)
- Lock-free single-producer/consumer pattern is needed
- Hardware DMA requires contiguous memory
- Cache performance is critical (audio, video, networking)

**When NOT to use circular queue:**
- Capacity is unpredictable and unbounded → use `std::queue` or linked queue
- You need to insert or remove from the middle → use `std::deque` or linked list
- You need to search → the circular structure does not help
- Multiple producers and consumers with complex synchronisation → use a dedicated concurrent queue library

---

## 12. Self-Test Questions

1. **Why does a naive array-backed queue fail after many enqueue/dequeue operations? Draw the state of a 6-slot array after 4 enqueues and 4 dequeues, showing the wasted space.**

2. **What are the three strategies for resolving the full-vs-empty ambiguity in a circular queue? Which do you recommend and why?**

3. **Derive the formula for `back()` when `tail_` points to the next write slot. Why must you add `capacity_` before taking modulo?**

4. **Trace `MyCircularQueue(3)` through: `enQueue(1), enQueue(2), enQueue(3), deQueue(), enQueue(4), Rear()`. Show `front_`, `tail_`, and the physical array after each operation.**

5. **In the circular deque implementation, `insertFront` retreats first then writes, while `insertLast` advances first then writes. Why are these directions opposite? What would go wrong if both advanced?**

6. **A circular queue has `front=5, tail=2, capacity=8`. Is it empty, full, or partially filled (assume size counter is not available and you are using gap-of-one strategy)? How many elements does it contain?**

7. **Explain why the lock-free SPSC circular queue uses `memory_order_acquire` when loading the other thread's index and `memory_order_release` when storing your own. What data race would occur without these orderings?**

8. **During resize of a wrapped circular queue (`front=4, tail=2, size=6, cap=8`), write the exact loop that copies elements into the new linearised array. What are `front_` and `tail_` after the resize?**

9. **In the Task Scheduler problem, why is the answer `max((maxFreq-1)*(n+1)+maxCount, tasks.size())` and not just the formula without the max? Give a concrete example where `tasks.size()` is larger.**

10. **Why do NIC ring buffers use circular queues rather than linked lists? Name three specific properties of circular queues that make them suitable for DMA-based hardware interfaces.**

---

## Quick Reference Card

```
Circular Queue — fixed-capacity FIFO where the array wraps around like a ring.

Core invariant:    tail_ = (front_ + size_) % capacity_
Advance formula:   new_idx = (old_idx + 1) % capacity_
Retreat formula:   new_idx = (old_idx - 1 + capacity_) % capacity_

Three full/empty strategies:
  A) Size counter:   full → sz==cap,         empty → sz==0        ← recommended
  B) Gap of one:     full → (tail+1)%cap==front, empty → front==tail
  C) Bool flag:      is_full flag toggled on transition              ← avoid

All operations: O(1) — no dynamic allocation, no shifting
  enqueue: data[tail] = val; tail = (tail+1)%cap; size++
  dequeue: val = data[front]; front = (front+1)%cap; size--; return val
  front():  data[front]
  back():   data[(tail-1+cap)%cap]    ← the +cap is critical
  at(i):    data[(front+i)%cap]       ← O(1) random access

Pitfall checklist:
  ✗ (tail-1) % cap  when tail=0 → returns -1  (use (tail-1+cap)%cap)
  ✗ Iterating data[0..size-1] instead of data[(front+i)%cap]
  ✗ Allocating k slots for capacity k with gap-of-one (need k+1)
  ✗ Not re-linearising during resize when queue is wrapped
  ✗ Using reference returned by dequeue (slot reusable after dequeue)

Ring buffer mode (streaming):
  When full: dequeue oldest, then enqueue new.
  Never blocks. Always holds the most recent `capacity` items.

Real-world: NIC rings, audio buffers, round-robin schedulers,
            JavaScript event loop, UART buffers, trading feed rings.
```

---

*Previous: [09 — Queue](./09_queue.md)*
*Next: [11 — Deque](./11_deque.md)*
*See also: [Ring Buffer (specialized)](./specialized/ring_buffer.md) | [Priority Queue](./12_priority_queue.md) | [Lock-free data structures](./systems/lock_free.md)*

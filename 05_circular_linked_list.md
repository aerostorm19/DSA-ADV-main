# Circular Linked List

> **Curriculum position:** Linear Structures → #5
> **Interview weight:** ★★★☆☆ Medium — appears in scheduling, round-robin, and Josephus-class problems
> **Difficulty:** Beginner-Intermediate — builds directly on SLL/DLL, adds one invariant
> **Prerequisite:** [03 — Singly Linked List](./03_singly_linked_list.md) | [04 — Doubly Linked List](./04_doubly_linked_list.md)

---

## Table of Contents

1. [Intuition First — When the End Loops Back](#1-intuition-first)
2. [Internal Working — Singly vs Doubly Circular](#2-internal-working)
3. [The Tail Pointer Convention](#3-the-tail-pointer-convention)
4. [Time & Space Complexity](#4-time--space-complexity)
5. [Complete C++ Implementation](#5-complete-c-implementation)
6. [Core Operations — Visualised](#6-core-operations--visualised)
7. [Common Patterns & Techniques](#7-common-patterns--techniques)
8. [Interview Problems](#8-interview-problems)
9. [Real-World Uses](#9-real-world-uses)
10. [Edge Cases & Pitfalls](#10-edge-cases--pitfalls)
11. [Comparison: CLL vs SLL vs DLL vs Ring Buffer](#11-comparison)
12. [Self-Test Questions](#12-self-test-questions)

---

## 1. Intuition First

Every linked list you have studied so far has a clean beginning and a definitive end — the last node's `next` pointer sits at `nullptr`, a hard wall that stops all traversal. The circular linked list removes that wall. The last node's `next` pointer loops back to the **first node**, forming a closed ring with no beginning and no end.

Think of a round-robin tournament. Every team plays every other team in sequence, and after the last team plays, the cycle restarts from the first team. There is no "end" of the schedule — it wraps around. Or think of a clock face: after 12 comes 1 again, seamlessly and indefinitely.

That is the defining invariant of a circular linked list:

```
last_node->next == first_node       (always, for every non-empty list)
```

This single change — replacing `nullptr` with a back-pointer — unlocks three things the linear list cannot offer:

1. **Infinite traversal** — you can visit nodes continuously without hitting a wall, which is exactly what a scheduler or media player needs.
2. **O(1) access to both ends from a single pointer** — if you store only a `tail` pointer, `tail` gives you the tail directly and `tail->next` gives you the head in O(1). No separate `head` variable needed.
3. **Natural rotation** — advancing the "start" of the list by one step is a single pointer update, not a traversal.

The cost is constant vigilance: traversal loops must terminate on a condition other than `nullptr`, and every operation must preserve the circular invariant. Break it once — accidentally point the last node's `next` to `nullptr` — and any subsequent traversal hangs in an infinite loop or crashes.

---

## 2. Internal Working — Singly vs Doubly Circular

There are two mainstream variants. Both share the circular invariant; they differ in how many pointers each node carries.

### Singly Circular Linked List (SCLL)

Each node has one `next` pointer. The last node's `next` points to the first node.

```
        ┌──────────────────────────────────────────────┐
        ↓                                              │
   ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
   │   10    │    │   20    │    │   30    │    │   40    │
   │ next ───┼───▶│ next ───┼───▶│ next ───┼───▶│ next ───┼──┐
   └─────────┘    └─────────┘    └─────────┘    └─────────┘  │
        ↑                                                      │
        └──────────────────────────────────────────────────────┘

   tail pointer → node[40]   (the last inserted / "current" node)
   head         → tail->next = node[10]
```

**Key design choice — store `tail`, not `head`:**
- `head` = `tail->next` — free, O(1)
- `tail` = `tail` — direct, O(1)
- Insert at front: create node, wire it as `tail->next` — O(1)
- Insert at back: create node, wire it after `tail`, advance `tail` — O(1)

If you stored only `head` instead, reaching the tail would cost O(n).

### Doubly Circular Linked List (DCLL)

Each node has both `next` and `prev` pointers. Both the last→first and first→last edges are doubly linked.

```
   ┌──────────────────────────────────────────────────────────────────┐
   │    ┌─────────────────────────────────────────────────────────┐   │
   ↓    ↑                                                         ↓   ↑
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│    10    │    │    20    │    │    30    │    │    40    │
│ ←prev   │    │ ←prev    │    │ ←prev    │    │ ←prev    │
│  next→  │    │  next→   │    │  next→   │    │  next→   │
└──────────┘    └──────────┘    └──────────┘    └──────────┘
     ↑                                                ↑
   head                                             tail

head->prev == tail    (backward wrap)
tail->next == head    (forward wrap)
```

The DCLL is the basis of `std::list`'s internal layout in most implementations (with a sentinel node where `sentinel->next == head` and `sentinel->prev == tail`, forming a closed ring through the sentinel).

---

## 3. The Tail Pointer Convention

This is an architectural decision that shapes every operation. Get it wrong and you spend the rest of the implementation fighting your own code.

### Why Tail Is Better Than Head

```
Store head:                          Store tail:
  head → [10]→[20]→[30]→[40]→         tail → [40]→ (wraps to [10])
  
  Access head:  head         O(1)      Access head:  tail->next    O(1)
  Access tail:  traverse     O(n) ✗    Access tail:  tail          O(1) ✓
  Insert front: O(1)         ✓         Insert front: O(1)          ✓
  Insert back:  O(n) traverse ✗        Insert back:  O(1)          ✓
```

Storing `tail` gives O(1) access to both ends. Storing `head` forces O(n) access to the tail on every back-insertion. The tail pointer convention is used in all production implementations.

### The Single-Node Edge Case

```
Single node [42] with tail pointer:
  tail → [42]
  [42]->next == [42]   ← points to itself

Empty list:
  tail == nullptr
```

A single-node circular list is a node that points to itself. This is the minimal closed ring. Many bugs arise from forgetting this state.

---

## 4. Time & Space Complexity

### Operation Complexities (Singly Circular, tail pointer)

| Operation | Time | Notes |
|---|---|---|
| Access head | **O(1)** | `tail->next` |
| Access tail | **O(1)** | `tail` directly |
| Access index i | **O(n)** | Traverse from head |
| Search by value | **O(n)** | Full loop traversal |
| Insert at front | **O(1)** | Wire new node as `tail->next`, update `tail->next` |
| Insert at back | **O(1)** | Wire new node after tail, advance `tail` |
| Insert at index i | **O(n)** | Traverse to position |
| Delete front | **O(1)** | Update `tail->next` to `head->next`; free old head |
| Delete back | **O(n)** | Must find node before tail — no `prev` pointer |
| Delete back (DCLL) | **O(1)** | `tail->prev` gives predecessor directly |
| Delete given node (SCLL) | **O(n)** | Must find predecessor |
| Delete given node (DCLL) | **O(1)** | Use `node->prev` and `node->next` |
| Rotate by k | **O(k)** | Advance `tail` k steps; `tail->next` is new head |
| Split into two halves | **O(n)** | Find midpoint, rewire |
| Detect it is circular | **O(1)** | Check `last_node->next == head` |

### Space Complexity

| | |
|---|---|
| n nodes | **O(n)** |
| Per-node overhead (SCLL) | 1 pointer = 8 bytes |
| Per-node overhead (DCLL) | 2 pointers = 16 bytes |
| Auxiliary space (most ops) | **O(1)** |
| No sentinel overhead | Circular structure naturally eliminates null-end sentinels |

---

## 5. Complete C++ Implementation

Both variants are implemented below. The SCLL is the foundation; the DCLL extends it with backward pointers.

### Singly Circular Linked List

```cpp
#include <iostream>
#include <stdexcept>
#include <initializer_list>
using namespace std;

struct SCLLNode {
    int       val;
    SCLLNode* next;
    explicit SCLLNode(int v) : val(v), next(nullptr) {}
};

class SinglyCircularList {
private:
    SCLLNode* tail_;   // points to last node; tail_->next == head (first node)
    int       size_;

public:
    // ── Construction & Destruction ────────────────────────────────────────────

    SinglyCircularList() : tail_(nullptr), size_(0) {}

    SinglyCircularList(initializer_list<int> vals) : SinglyCircularList() {
        for (int v : vals) push_back(v);
    }

    ~SinglyCircularList() {
        if (!tail_) return;
        // Break the circle first, then traverse like a linear list
        SCLLNode* head = tail_->next;
        tail_->next = nullptr;    // break the circle — now it's a linear list
        SCLLNode* curr = head;
        while (curr) {
            SCLLNode* next = curr->next;
            delete curr;
            curr = next;
        }
    }

    SinglyCircularList(const SinglyCircularList&)            = delete;
    SinglyCircularList& operator=(const SinglyCircularList&) = delete;

    // ── Capacity ──────────────────────────────────────────────────────────────

    int  size()  const { return size_; }
    bool empty() const { return tail_ == nullptr; }

    // ── Access ────────────────────────────────────────────────────────────────

    int front() const {
        if (!tail_) throw runtime_error("list is empty");
        return tail_->next->val;   // head = tail->next
    }

    int back() const {
        if (!tail_) throw runtime_error("list is empty");
        return tail_->val;
    }

    // Expose raw pointers for algorithms
    SCLLNode* tail_node() const { return tail_; }
    SCLLNode* head_node() const { return tail_ ? tail_->next : nullptr; }

    // ── Insertion ─────────────────────────────────────────────────────────────

    // O(1) — insert at front (new node becomes new head)
    void push_front(int val) {
        SCLLNode* node = new SCLLNode(val);
        if (!tail_) {
            // Empty list: single node points to itself
            node->next = node;
            tail_ = node;
        } else {
            node->next  = tail_->next;   // new node → old head
            tail_->next = node;          // tail → new node (new head)
        }
        size_++;
    }

    // O(1) — insert at back (new node becomes new tail)
    void push_back(int val) {
        push_front(val);        // insert at front first
        tail_ = tail_->next;   // advance tail to the new node
        // Explanation: push_front makes the new node the head (tail->next).
        // Moving tail one step forward makes that new node the tail instead.
    }

    // O(n) — insert at index (0 = before current head)
    void insert(int index, int val) {
        if (index < 0 || index > size_) throw out_of_range("index out of range");
        if (index == 0)     { push_front(val); return; }
        if (index == size_) { push_back(val);  return; }

        SCLLNode* prev = tail_->next;   // start at head
        for (int i = 0; i < index - 1; i++) prev = prev->next;

        SCLLNode* node = new SCLLNode(val);
        node->next  = prev->next;
        prev->next  = node;
        size_++;
    }

    // ── Deletion ──────────────────────────────────────────────────────────────

    // O(1) — remove front (head)
    void pop_front() {
        if (!tail_) throw runtime_error("list is empty");
        if (tail_->next == tail_) {
            // Single node: list becomes empty
            delete tail_;
            tail_ = nullptr;
        } else {
            SCLLNode* old_head = tail_->next;
            tail_->next = old_head->next;   // tail now points to new head
            delete old_head;
        }
        size_--;
    }

    // O(n) — remove back (tail) — must find predecessor
    void pop_back() {
        if (!tail_) throw runtime_error("list is empty");
        if (tail_->next == tail_) {
            // Single node
            delete tail_;
            tail_ = nullptr;
            size_--;
            return;
        }
        // Find node just before tail_
        SCLLNode* prev = tail_->next;    // start at head
        while (prev->next != tail_) prev = prev->next;

        prev->next = tail_->next;        // bypass old tail
        delete tail_;
        tail_ = prev;                    // prev is now the new tail
        size_--;
    }

    // O(n) — remove first node with given value
    bool remove(int val) {
        if (!tail_) return false;

        // Special case: removing the head
        if (tail_->next->val == val) {
            pop_front();
            return true;
        }

        SCLLNode* prev = tail_->next;    // start at head
        while (prev->next != tail_->next && prev->next->val != val) {
            prev = prev->next;
        }
        if (prev->next->val != val) return false;  // not found

        SCLLNode* to_del = prev->next;
        if (to_del == tail_) tail_ = prev;   // removing tail — update tail_
        prev->next = to_del->next;
        delete to_del;
        size_--;
        return true;
    }

    // ── Rotation ──────────────────────────────────────────────────────────────

    // O(k) — rotate left by k: advance the "start" by k steps
    // Equivalent to: the node that was at index k becomes the new head
    void rotate(int k) {
        if (!tail_ || size_ == 1) return;
        k = ((k % size_) + size_) % size_;   // handle negative k and k > size_
        if (k == 0) return;
        for (int i = 0; i < k; i++) tail_ = tail_->next;
        // After advancing tail_ by k, tail_->next is the new head
    }

    // ── Utilities ─────────────────────────────────────────────────────────────

    void print() const {
        if (!tail_) { cout << "(empty)\n"; return; }
        SCLLNode* curr = tail_->next;   // start at head
        do {
            cout << curr->val;
            curr = curr->next;
            if (curr != tail_->next) cout << " → ";
        } while (curr != tail_->next);
        cout << " → (back to head)\n";
    }

    // O(n) — split into two halves; returns head of second half
    // Useful for merge sort on circular lists
    SCLLNode* split_half() {
        if (!tail_ || size_ <= 1) return nullptr;

        // Use fast/slow pointers to find the midpoint
        SCLLNode* slow = tail_->next;   // head
        SCLLNode* fast = tail_->next;

        // Stop when fast completes the circle or reaches tail
        while (fast != tail_ && fast->next != tail_) {
            slow = slow->next;
            fast = fast->next->next;
        }
        // slow is now at the midpoint; slow->next starts second half
        SCLLNode* second_head = slow->next;
        slow->next = tail_->next;   // first half wraps to original head

        // The caller must handle the second half's circular wiring
        return second_head;
    }
};
```

### Doubly Circular Linked List

```cpp
struct DCLLNode {
    int       val;
    DCLLNode* prev;
    DCLLNode* next;
    explicit DCLLNode(int v) : val(v), prev(nullptr), next(nullptr) {}
};

class DoublyCircularList {
private:
    DCLLNode* head_;   // first node; head_->prev == tail_
    DCLLNode* tail_;   // last node;  tail_->next == head_
    int       size_;

    // O(1): insert `node` between `before` and `after`
    void link_between(DCLLNode* before, DCLLNode* node, DCLLNode* after) {
        node->prev   = before;
        node->next   = after;
        before->next = node;
        after->prev  = node;
        size_++;
    }

    // O(1): remove `node` from the ring; does NOT free memory
    void unlink(DCLLNode* node) {
        node->prev->next = node->next;
        node->next->prev = node->prev;
        size_--;
    }

public:
    DoublyCircularList() : head_(nullptr), tail_(nullptr), size_(0) {}

    DoublyCircularList(initializer_list<int> vals) : DoublyCircularList() {
        for (int v : vals) push_back(v);
    }

    ~DoublyCircularList() {
        if (!head_) return;
        tail_->next  = nullptr;   // break the circle
        head_->prev  = nullptr;
        DCLLNode* curr = head_;
        while (curr) {
            DCLLNode* next = curr->next;
            delete curr;
            curr = next;
        }
    }

    int  size()  const { return size_; }
    bool empty() const { return head_ == nullptr; }

    int front() const {
        if (!head_) throw runtime_error("list is empty");
        return head_->val;
    }
    int back() const {
        if (!tail_) throw runtime_error("list is empty");
        return tail_->val;
    }

    DCLLNode* head_node() const { return head_; }
    DCLLNode* tail_node() const { return tail_; }

    // O(1) — insert at front
    void push_front(int val) {
        DCLLNode* node = new DCLLNode(val);
        if (!head_) {
            node->next = node->prev = node;   // self-loop
            head_ = tail_ = node;
            size_++;
        } else {
            link_between(tail_, node, head_);
            head_ = node;
        }
    }

    // O(1) — insert at back
    void push_back(int val) {
        DCLLNode* node = new DCLLNode(val);
        if (!head_) {
            node->next = node->prev = node;
            head_ = tail_ = node;
            size_++;
        } else {
            link_between(tail_, node, head_);
            tail_ = node;
        }
    }

    // O(1) — remove front
    void pop_front() {
        if (!head_) throw runtime_error("list is empty");
        if (head_ == tail_) {
            delete head_;
            head_ = tail_ = nullptr;
            size_--;
            return;
        }
        DCLLNode* old_head = head_;
        head_ = head_->next;
        unlink(old_head);
        delete old_head;
    }

    // O(1) — remove back — DLL advantage: no traversal needed
    void pop_back() {
        if (!tail_) throw runtime_error("list is empty");
        if (head_ == tail_) {
            delete tail_;
            head_ = tail_ = nullptr;
            size_--;
            return;
        }
        DCLLNode* old_tail = tail_;
        tail_ = tail_->prev;
        unlink(old_tail);
        delete old_tail;
    }

    // O(1) — remove an arbitrary node given its pointer
    void remove_node(DCLLNode* node) {
        if (!node) return;
        if (node == head_) head_ = (size_ == 1) ? nullptr : head_->next;
        if (node == tail_) tail_ = (size_ == 1) ? nullptr : tail_->prev;
        unlink(node);
        delete node;
    }

    // O(k) — rotate: advance head/tail by k steps
    void rotate(int k) {
        if (!head_ || size_ == 1) return;
        k = ((k % size_) + size_) % size_;
        for (int i = 0; i < k; i++) {
            tail_ = tail_->next;
            head_ = head_->next;
        }
    }

    void print_forward() const {
        if (!head_) { cout << "(empty)\n"; return; }
        DCLLNode* curr = head_;
        do {
            cout << curr->val;
            curr = curr->next;
            if (curr != head_) cout << " ⇄ ";
        } while (curr != head_);
        cout << " ⇄ (back to head)\n";
    }

    void print_backward() const {
        if (!tail_) { cout << "(empty)\n"; return; }
        DCLLNode* curr = tail_;
        do {
            cout << curr->val;
            curr = curr->prev;
            if (curr != tail_) cout << " ⇄ ";
        } while (curr != tail_);
        cout << " ⇄ (back to tail)\n";
    }
};
```

---

## 6. Core Operations — Visualised

### Insert at Back (SCLL, tail pointer)

```
Before: tail → [40] → [10] → [20] → [30] → [40] (circular)
Goal: insert [50] at back

Step 1: Create node [50]
Step 2: new_node->next = tail_->next        [50] → [10]  (point to current head)
Step 3: tail_->next    = new_node           [40] → [50]  (old tail points to new node)
Step 4: tail_          = new_node           tail → [50]  (advance tail)

After:  tail → [50] → [10] → [20] → [30] → [40] → [50] (circular)

Head = tail_->next = [10] ← unchanged ✓
Tail = tail_       = [50] ← new tail  ✓
```

### Delete Front (SCLL)

```
Before: tail → [40] → [10] → [20] → [30] → [40]
                              ↑ head

Step 1: old_head = tail_->next = [10]
Step 2: tail_->next = old_head->next = [20]   (tail now points to [20] = new head)
Step 3: delete old_head

After:  tail → [40] → [20] → [30] → [40]
Head = tail_->next = [20] ✓
```

### Rotate Left by 2

```
Before: tail → [40] → [10] → [20] → [30] → [40]
        (Head = [10])

rotate(2): advance tail_ by 2 steps
  Step 1: tail_ = tail_->next = [10]
  Step 2: tail_ = tail_->next = [20]

After:  tail → [20] → [30] → [40] → [10] → [20]
        Head = tail_->next = [30]

The same nodes, same links — only the "start" pointer moved.
Rotation is O(k) pointer walks with ZERO allocation. ✓
```

### The Josephus Elimination — Visualised

```
n=6 people, every 3rd eliminated. Circle: [1]→[2]→[3]→[4]→[5]→[6]→[1]

Round 1: count 1,2,3 → eliminate [3]. tail advances past [3].
  Circle: [1]→[2]→[4]→[5]→[6]→[1]

Round 2: count 4,5,6 → eliminate [6].
  Circle: [1]→[2]→[4]→[5]→[1]

Round 3: count 1,2,4 → eliminate [4].
  Circle: [1]→[2]→[5]→[1]

Round 4: count 5,1,2 → eliminate [2].
  Circle: [1]→[5]→[1]

Round 5: count 5,1,5 → eliminate [5].
  Winner: [1]
```

---

## 7. Common Patterns & Techniques

### Pattern 1: Termination Condition for Circular Traversal

The most fundamental pattern. In a linear list, you stop at `nullptr`. In a circular list, you stop when you have returned to the starting node.

```cpp
// Traverse every node exactly once
void traverse(SCLLNode* tail) {
    if (!tail) return;
    SCLLNode* curr = tail->next;    // start at head
    do {
        process(curr);
        curr = curr->next;
    } while (curr != tail->next);  // stop when we reach head again

    // Alternatively — equivalent and sometimes cleaner:
    SCLLNode* start = tail->next;
    curr = start;
    do {
        process(curr);
        curr = curr->next;
    } while (curr != start);
}
```

**Why `do-while` and not `while`?**  
A `while` loop with the circular condition checks the condition before the first iteration. For a single-node list, `curr == start` is immediately true — the loop body never executes, and the single node is skipped. A `do-while` always executes at least once, which is the correct semantics for visiting every node including in the single-element case.

### Pattern 2: Counting Steps Without a Length

When you do not know the list length (or do not want to maintain `size_`), you can count steps until you complete a full revolution:

```cpp
int count_length(SCLLNode* tail) {
    if (!tail) return 0;
    int count = 1;
    SCLLNode* curr = tail->next;     // start at head
    while (curr != tail) {           // stop when we reach tail
        curr = curr->next;
        count++;
    }
    return count;
}
```

### Pattern 3: Two-Pointer on a Circular List

Fast/slow pointers work on circular lists with one modification: the loop terminates when fast equals slow (cycle detection), but you must ensure fast starts one step ahead or you get an immediate false positive at the start.

```cpp
// Find if a circular list has been corrupted into containing
// an extra internal cycle (pathological case)
bool has_internal_cycle(SCLLNode* head) {
    if (!head) return false;
    SCLLNode* slow = head;
    SCLLNode* fast = head->next;
    while (fast && fast->next) {
        if (slow == fast) return true;
        slow = slow->next;
        fast = fast->next->next;
    }
    return false;
}
```

### Pattern 4: Josephus Step Simulation

Advance a pointer exactly k-1 steps (not k), delete the next node, repeat. The off-by-one is the most common Josephus bug.

```cpp
// Eliminate every k-th node, return the last surviving node's value
int josephus(int n, int k) {
    // Build circular list 1..n
    SCLLNode* head = new SCLLNode(1);
    SCLLNode* curr = head;
    for (int i = 2; i <= n; i++) {
        curr->next = new SCLLNode(i);
        curr = curr->next;
    }
    curr->next = head;    // close the circle

    curr = head;
    while (curr->next != curr) {    // while more than one node remains
        // Advance k-1 steps (curr is "1st count", curr->next is kth)
        for (int i = 0; i < k - 1; i++) curr = curr->next;

        // curr->next is the node to eliminate
        SCLLNode* to_del = curr->next;
        curr->next = to_del->next;
        delete to_del;
        curr = curr->next;    // continue counting from the next node
    }

    int survivor = curr->val;
    delete curr;
    return survivor;
}
```

---

## 8. Interview Problems

### Problem 1: Josephus Problem

**Problem:** `n` people stand in a circle numbered 1 to n. Starting from person 1, every k-th person is eliminated. Find the position of the last person standing.

**Example:** n=6, k=3 → survivor is person 1 (see visualisation above).

**Two approaches — simulation and mathematics:**

```cpp
// ── Approach 1: Circular List Simulation — O(nk) time, O(n) space ─────────────
// Intuitive, works for any k, directly models the problem
int josephus_simulation(int n, int k) {
    // Build: [1]→[2]→...→[n]→[1]
    SCLLNode* head = new SCLLNode(1);
    SCLLNode* prev = head;
    for (int i = 2; i <= n; i++) {
        prev->next = new SCLLNode(i);
        prev = prev->next;
    }
    prev->next = head;   // close the ring

    SCLLNode* curr = head;
    while (curr->next != curr) {
        // Advance k-1 steps: curr is position 1, curr->next is position k
        for (int i = 0; i < k - 1; i++) curr = curr->next;

        SCLLNode* eliminated = curr->next;
        cout << "Eliminated: " << eliminated->val << "\n";
        curr->next = eliminated->next;
        delete eliminated;

        curr = curr->next;   // next round starts from node after eliminated
    }

    int winner = curr->val;
    delete curr;
    return winner;
}

// ── Approach 2: Mathematical Recurrence — O(n) time, O(1) space ──────────────
// Based on the recurrence: J(1,k) = 0
//                           J(n,k) = (J(n-1,k) + k) % n
// Result is 0-indexed; add 1 for 1-indexed answer.
//
// Derivation intuition:
//   After the first elimination, the remaining n-1 people form a new circle.
//   The new numbering is shifted: old position p maps to new position (p-k-1) mod n.
//   Reversing this shift recovers the original position from the subproblem answer.

int josephus_math(int n, int k) {
    int pos = 0;                          // survivor position in 1-person circle
    for (int i = 2; i <= n; i++) {
        pos = (pos + k) % i;              // scale up to i-person circle
    }
    return pos + 1;                       // convert to 1-indexed
}

/*
Trace for n=6, k=3:
  i=2: pos = (0 + 3) % 2 = 1
  i=3: pos = (1 + 3) % 3 = 1
  i=4: pos = (1 + 3) % 4 = 0
  i=5: pos = (0 + 3) % 5 = 3
  i=6: pos = (3 + 3) % 6 = 0
  return 0 + 1 = 1  ✓  (person 1 survives)
*/
```

**Complexity comparison:**

| Approach | Time | Space | Notes |
|---|---|---|---|
| Simulation | O(nk) | O(n) | Intuitive; models the problem exactly |
| Recurrence | **O(n)** | **O(1)** | Optimal; interview gold if you know it |

**When to use which in an interview:** If the interviewer asks for the answer, derive the recurrence — it demonstrates mathematical thinking. If they ask you to "simulate" or "model the problem", use the circular list — it demonstrates data structure knowledge. Ideally, present simulation first (brute force) then optimise to recurrence.

**Edge cases:**
- k=1: every person eliminates the next; survivor is always person n
- k=n: survivor is the last person who would be counted
- n=1: the single person is the survivor, answer is 1
- Very large n and k: the recurrence is O(n) regardless of k size

---

### Problem 2: Split a Circular Linked List into Two Equal Halves

**Problem:** Given a circular singly linked list, split it into two circular linked lists of equal halves. If the number of nodes is odd, the extra node goes to the first half.

**Example:** `[1→2→3→4]` → `[1→2→(→1)]` and `[3→4→(→3)]`

**Thought process:**

> "I need to find the midpoint. Fast/slow pointers work, but I must account for the circular structure — fast stops at the tail node, not at nullptr. After finding the midpoint, I need to rewire two circular chains: the first half ends at slow and wraps to the original head, the second half ends at the original tail and wraps to slow->next."

```cpp
pair<SCLLNode*, SCLLNode*> split_circular(SCLLNode* tail) {
    if (!tail) return {nullptr, nullptr};

    SCLLNode* head = tail->next;

    // Single node: cannot split
    if (head == tail) return {tail, nullptr};

    // Two nodes: split directly
    if (head->next == tail) {
        head->next = head;   // first half: single node, self-loop
        tail->next = tail;   // second half: single node, self-loop
        return {head, tail};
    }

    // Use fast/slow pointers to find midpoint
    SCLLNode* slow = head;
    SCLLNode* fast = head;

    // Fast moves 2 steps per iteration; stops when it completes the circle
    while (fast->next != head && fast->next->next != head) {
        slow = slow->next;
        fast = fast->next->next;
    }

    // Handle even-length: advance fast one more step so slow is at first-half tail
    if (fast->next->next == head) fast = fast->next;

    // slow is now at the midpoint (end of first half)
    // fast is at the end of the second half (original tail)
    SCLLNode* head2 = slow->next;   // start of second half

    // Rewire: close first half into a circle
    slow->next = head;

    // Rewire: close second half into a circle
    // fast->next was head; now it should be head2
    fast->next = head2;

    // Return tails of each half
    return {slow, fast};
}

/*
Trace for [1→2→3→4→1] (tail points to [4]):
  head = [1]
  slow = [1], fast = [1]

  Iteration 1:
    fast->next = [2], fast->next->next = [3] — both not head → continue
    slow = [2], fast = [3]

  Iteration 2:
    fast->next = [4], fast->next->next = [1] = head → STOP (even length)
    fast = fast->next = [4]  (advance once more)

  slow = [2] (end of first half)
  fast = [4] (end of second half)
  head2 = slow->next = [3]

  Rewire first half:  [2]->next = [1]   → [1]→[2]→[1] ✓
  Rewire second half: [4]->next = [3]   → [3]→[4]→[3] ✓

Return: (tail1=[2], tail2=[4])
*/
```

**Edge cases:**
- Odd length `[1→2→3]`: slow lands at [1], extra node goes to first half → `[1→2→1]` and `[3→3]`
- Two nodes: special-cased above — each becomes a single-node self-loop
- Single node: cannot split — return (node, nullptr)

---

### Problem 3: Circular Tour (Gas Station)

**Problem:** There are n gas stations along a circular route. Station i has `gas[i]` litres and costs `cost[i]` litres to travel to the next station. Find the starting station from which you can complete the full circle. If no solution exists return -1. The solution is guaranteed to be unique if it exists.

**Why this is a circular list problem:**  
The route is a circle — after station n-1 comes station 0. You need to find a starting point such that the running fuel never goes negative over a full traversal of the circle.

**Thought process:**

> "Brute force: try each station as start, simulate. O(n²)."
>
> "Key observation: if total(gas) >= total(cost), a solution always exists. Why? Because the net surplus is non-negative — there is enough fuel globally. The question is only where to start."
>
> "Greedy insight: if running from station `start` the tank goes negative at station `i`, then no station between `start` and `i` can be the answer — they all start with less surplus than `start` did (since `start` already accumulated some surplus before failing). So: reset `start` to `i+1` whenever the tank goes negative."

```cpp
// O(n) time, O(1) space — single pass greedy
int canCompleteCircuit(vector<int>& gas, vector<int>& cost) {
    int total_surplus = 0;    // total gas - total cost across all stations
    int tank          = 0;    // current tank level from the current start
    int start         = 0;    // candidate starting station

    for (int i = 0; i < (int)gas.size(); i++) {
        int net = gas[i] - cost[i];
        total_surplus += net;
        tank          += net;

        if (tank < 0) {
            // Cannot reach station i+1 from current start.
            // Reset: try starting from i+1.
            start = i + 1;
            tank  = 0;
        }
    }

    // If total surplus >= 0, a unique solution exists at `start`.
    return (total_surplus >= 0) ? start : -1;
}

/*
Dry run: gas=[1,2,3,4,5], cost=[3,4,5,1,2]
  net = [-2,-2,-2, 3, 3]

  i=0: tank=-2 < 0 → start=1, tank=0
  i=1: tank=-2 < 0 → start=2, tank=0
  i=2: tank=-2 < 0 → start=3, tank=0
  i=3: tank= 3 ≥ 0 → continue
  i=4: tank= 6 ≥ 0 → continue

  total_surplus = -2-2-2+3+3 = 0 ≥ 0 → solution exists
  return start = 3 ✓

Verify: start at station 3 (0-indexed)
  At 3: tank = 4-1 = 3
  At 4: tank = 3 + 5-2 = 6
  At 0: tank = 6 + 1-3 = 4
  At 1: tank = 4 + 2-4 = 2
  At 2: tank = 2 + 3-5 = 0  ← exactly enough, never negative ✓
*/
```

**Why greedy works — the proof:**

If `total_surplus >= 0`, a valid start exists. The greedy algorithm finds it because:
- When the tank goes negative at station i, all stations from `start` to i are disqualified.
- Disqualified because: starting at any of them, you'd have even less fuel at i than starting from `start` did (since `start` accumulated some positive surplus before reaching those stations).
- Therefore, the only remaining candidates are stations i+1 onwards.
- The last `start` that survives to the end, combined with `total_surplus >= 0`, is the answer.

**Edge cases:**
- All stations have 0 net gain: total_surplus = 0, any valid start (usually 0) works
- n=1: if `gas[0] >= cost[0]`, return 0; else return -1
- No valid start: `total_surplus < 0`, return -1

---

## 9. Real-World Uses

| Domain | Use Case | Why Circular |
|---|---|---|
| OS kernel | Round-robin CPU scheduler | Continuously cycle through processes; no "last" process |
| OS kernel | Timer interrupt ring buffer | Circular write pointer; oldest entry overwritten |
| Multimedia | Audio/video streaming buffer | Producer writes, consumer reads; wraps around seamlessly |
| Networking | Token ring protocol (IEEE 802.5) | Token circulates forever; nodes grab it when it passes |
| Games | Turn management in multiplayer games | Player order cycles: P1→P2→P3→P1→... |
| Compiler / Interpreter | Circular dependency detection in module graphs | Not a linked list directly, but same traversal pattern |
| CPU architecture | Hardware instruction queues (ring queues) | Circular buffer of fixed size for pipeline stages |
| Cryptography | RC4 stream cipher key schedule | 256-element circular array permuted and cycled |
| Database | PostgreSQL VACUUM circular buffer | Pages in a fixed-size circular work queue |
| Embedded systems | Event loop in RTOS (FreeRTOS tasks) | Task list is a circular DLL; scheduler advances the pointer |

**The OS Round-Robin Scheduler — in depth:**

```
Ready queue (circular DLL):
  HEAD ←→ [Process A] ←→ [Process B] ←→ [Process C] ←→ HEAD

  Tick interrupt fires:
    1. Suspend current process (A), add to back of queue
    2. Load next process (B) from front, run for 1 quantum

  After one full cycle:  A → B → C → A → B → C → A → ...

  The scheduler never "ends" — the circular structure
  makes the round-robin semantics trivially correct.
  No special case for "we've reached the end of the list".

  Inserting a new process: push_back — O(1)
  Blocking a process (I/O wait): unlink from ready queue — O(1)
  Unblocking: push_back to ready queue — O(1)
```

---

## 10. Edge Cases & Pitfalls

### Pitfall 1: Infinite Loop from Missing Termination Condition

```cpp
// WRONG — this never terminates on a circular list
SCLLNode* curr = head;
while (curr) {               // curr is never null — infinite loop!
    cout << curr->val;
    curr = curr->next;
}

// CORRECT — check against the starting node
SCLLNode* start = head;
do {
    cout << curr->val;
    curr = curr->next;
} while (curr != start);
```

This is the #1 circular list bug. Any code borrowed from a linear list that terminates on `nullptr` will hang forever on a circular list.

### Pitfall 2: Off-by-One in Josephus Counting

```cpp
// WRONG: advance k steps (curr ends up AT the k-th node — it survives, not eliminated)
for (int i = 0; i < k; i++) curr = curr->next;
// curr is now the k-th node — but we want to eliminate it, not the node after it

// CORRECT: advance k-1 steps (curr is at position k-1; curr->next is the k-th)
for (int i = 0; i < k - 1; i++) curr = curr->next;
SCLLNode* eliminated = curr->next;
curr->next = eliminated->next;
```

The distinction: after advancing `k-1` steps, `curr` is the predecessor of the node to eliminate. You need `curr->next` to delete the right node and `curr->next = curr->next->next` to bypass it.

### Pitfall 3: Not Maintaining the Circular Invariant After Insertion/Deletion

```cpp
// WRONG: push_back that forgets to close the circle
void push_back_wrong(SCLLNode*& tail, int val) {
    SCLLNode* node = new SCLLNode(val);
    tail->next = node;   // connect old tail to new node
    tail = node;         // advance tail
    // MISSING: node->next = head — the new tail must wrap to head!
    // tail->next is now nullptr — the list is no longer circular
}

// CORRECT:
void push_back(SCLLNode*& tail, int val) {
    SCLLNode* node = new SCLLNode(val);
    node->next  = tail->next;   // new node → head (close the circle first)
    tail->next  = node;         // old tail → new node
    tail        = node;         // advance tail
}
```

### Pitfall 4: Single-Node Self-Loop

```cpp
// A single-node circular list points to itself
// Operations must handle this without corrupting the structure

// WRONG: delete the only node without updating tail_
void pop_front_wrong(SCLLNode*& tail) {
    SCLLNode* head = tail->next;
    tail->next = head->next;   // tail->next = tail (still points to deleted memory!)
    delete head;
    // tail now dangles — undefined behaviour on next access
}

// CORRECT:
void pop_front(SCLLNode*& tail, int& size) {
    if (tail->next == tail) {   // single node
        delete tail;
        tail = nullptr;
    } else {
        SCLLNode* old_head = tail->next;
        tail->next = old_head->next;
        delete old_head;
    }
    size--;
}
```

### Pitfall 5: Destructor Not Breaking the Circle

```cpp
// WRONG: naive destructor on circular list loops forever
~SinglyCircularList() {
    SCLLNode* curr = head_;
    while (curr) {             // curr never becomes null — infinite loop!
        SCLLNode* next = curr->next;
        delete curr;
        curr = next;
    }
}

// CORRECT: break the circle first, then traverse like a linear list
~SinglyCircularList() {
    if (!tail_) return;
    SCLLNode* head = tail_->next;
    tail_->next = nullptr;      // break the circle HERE
    SCLLNode* curr = head;
    while (curr) {
        SCLLNode* next = curr->next;
        delete curr;
        curr = next;
    }
}
```

### Pitfall 6: Using `while` Instead of `do-while` for Single-Element Lists

```cpp
// WRONG: misses the single-element case
SCLLNode* curr = head;
while (curr != head) {    // immediately false for single node — skips it entirely!
    process(curr);
    curr = curr->next;
}

// CORRECT:
SCLLNode* curr = head;
do {
    process(curr);
    curr = curr->next;
} while (curr != head);   // checks AFTER the first iteration — always visits at least once
```

### Pitfall 7: Fast Pointer Termination in Circular Context

```cpp
// WRONG: standard null-check termination for fast pointer — never triggers
SCLLNode* fast = head;
while (fast && fast->next) {        // fast is never null in a circular list
    fast = fast->next->next;        // infinite loop
}

// CORRECT: terminate when fast completes the circle
SCLLNode* fast = head;
while (fast->next != head && fast->next->next != head) {
    fast = fast->next->next;
}
// fast is now at the last or second-to-last node
```

---

## 11. Comparison

| Feature | SCLL | DCLL | Linear SLL | Linear DLL | Ring Buffer (array) |
|---|---|---|---|---|---|
| Backward traversal | No | Yes | No | Yes | Yes (index arithmetic) |
| Insert at front | O(1) | O(1) | O(1) | O(1) | O(1) amortized |
| Insert at back | O(1) with tail | O(1) | O(1) with tail | O(1) | O(1) |
| Delete front | O(1) | O(1) | O(1) | O(1) | O(1) |
| Delete back | O(n) | O(1) | O(n) | O(1) | O(1) |
| Delete given pointer | O(n) | O(1) | O(n) | O(1) | N/A |
| Rotation | O(k) no alloc | O(k) no alloc | O(k) no alloc | O(k) no alloc | O(1) index math |
| Termination condition | `curr != start` | `curr != start` | `curr == null` | `curr == null` | index wrap |
| Memory layout | Scattered (heap) | Scattered (heap) | Scattered | Scattered | **Contiguous** |
| Cache performance | Poor | Poor | Poor | Poor | **Excellent** |
| Fixed size | No | No | No | No | Yes (or dynamic) |
| Natural for infinite cycle | Yes | Yes | No | No | Yes |
| LRU Cache | Awkward | Natural | Awkward | **Natural** | Not suitable |
| Round-robin scheduling | **Natural** | Natural | Awkward | Awkward | Natural |

**When to choose a circular linked list:**
- You need semantically infinite cyclic traversal (scheduler, token ring)
- Rotation by k positions is a frequent operation (O(k), zero allocation)
- You are simulating a circular process (Josephus, card games, playlists)
- You need O(1) access to both head and tail from a single pointer (tail convention)

**When NOT to use it:**
- You need O(1) random access → use an array or ring buffer
- You need cache-efficient traversal → use a ring buffer (circular array)
- You need O(1) tail deletion without a doubly-linked variant → use DLL
- The circular structure adds complexity with no semantic benefit → use linear list

---

## 12. Self-Test Questions

1. **Why is the `tail` pointer preferred over the `head` pointer in a singly circular list? What operation becomes O(n) if you store only `head`?**

2. **Draw a three-node SCLL [A→B→C→A] and trace every pointer change during `push_back(D)`. How many pointer writes happen?**

3. **Why must `do-while` be used for circular list traversal instead of `while`? Give a concrete example where `while` produces wrong output.**

4. **In the Josephus simulation, advancing `k-1` steps and deleting `curr->next` is correct. What goes wrong if you advance `k` steps and delete `curr`?**

5. **Prove that the greedy gas station algorithm is correct. Specifically: why is every station between `start` and the failure point i also an invalid starting point?**

6. **Trace `split_circular` on the five-node list `[1→2→3→4→5→1]`. Which node does `slow` land on? Which nodes end up in each half?**

7. **Write the destructor for a SCLL. What is the single most important step, and what infinite loop does it prevent?**

8. **A DCLL has `head->prev == tail` and `tail->next == head`. You call `pop_back()` on a two-node list. List every pointer that changes and its new value.**

9. **What is the time complexity of rotating a SCLL of n nodes by k positions? Could it ever be reduced to O(1)?**

10. **Design a music playlist that supports: play-next (O(1)), play-previous (O(1)), add song at end (O(1)), remove current song (O(1)), and loop forever. Which circular structure would you use and why?**

---

## Quick Reference Card

```
Circular Linked List — the ring with no end.

Invariant:  last_node->next == first_node   (always, without exception)
Convention: store `tail` pointer, not `head`
  → head = tail->next    O(1)
  → tail = tail          O(1)
  → both ends in O(1) from a single pointer

Key complexities (SCLL with tail ptr):
  Insert front/back:  O(1)
  Delete front:       O(1)
  Delete back:        O(n) ← no prev pointer in SCLL
  Delete back (DCLL): O(1)
  Rotate by k:        O(k) zero allocation
  Traverse all n:     O(n)

Termination rules:
  Traversal:     do { ... curr = curr->next; } while (curr != start)
  Destructor:    tail->next = nullptr FIRST, then traverse like linear list
  Fast pointer:  stop at tail/head, NOT at nullptr

Canonical applications:
  Round-robin scheduler   → SCLL, advance tail one step per quantum
  Josephus problem        → SCLL simulation O(nk), or math O(n)
  Media playlist          → DCLL, O(1) prev/next/add/remove
  Gas station / Tour      → logical circle, greedy O(n) O(1) space

The #1 bug: forgetting to set new_node->next = head when inserting.
The #2 bug: using while(curr) instead of do-while with start comparison.
The #3 bug: destructor that never breaks the circle → infinite loop.
```

---

*Previous: [04 — Doubly Linked List](./04_doubly_linked_list.md)*
*Next: [05 — Stack](./05_stack.md)*
*See also: [Ring Buffer / Circular Buffer](./specialized/ring_buffer.md) | [Round-Robin Scheduling](./systems/scheduling.md)*

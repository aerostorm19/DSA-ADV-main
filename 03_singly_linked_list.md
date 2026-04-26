# Singly Linked List

> **Curriculum position:** Linear Structures → #3
> **Interview weight:** ★★★★★ Critical — pointer manipulation is tested in virtually every top-tier interview
> **Difficulty:** Beginner concept, surprisingly deep in practice
> **Prerequisite:** [01 — Static Array](./01_static_array.md) | [02 — Dynamic Array](./02_dynamic_array_vector.md)

---

## Table of Contents

1. [Intuition First — What Problem Does It Solve?](#1-intuition-first)
2. [Internal Working — Nodes and Pointers](#2-internal-working)
3. [The Fundamental Trade-off vs Arrays](#3-the-fundamental-trade-off-vs-arrays)
4. [Memory Layout — The Cache Penalty](#4-memory-layout--the-cache-penalty)
5. [Time & Space Complexity](#5-time--space-complexity)
6. [Complete C++ Implementation](#6-complete-c-implementation)
7. [Core Operations — Visualised](#7-core-operations--visualised)
8. [Common Patterns & Techniques](#8-common-patterns--techniques)
9. [Interview Problems](#9-interview-problems)
10. [Real-World Uses](#10-real-world-uses)
11. [Edge Cases & Pitfalls](#11-edge-cases--pitfalls)
12. [Comparison: SLL vs DLL vs Array vs std::list](#12-comparison)
13. [Self-Test Questions](#13-self-test-questions)

---

## 1. Intuition First

Imagine a treasure hunt. Each clue does not tell you the final destination — it only tells you where the next clue is hidden. You start at clue #1, which leads to clue #2, which leads to clue #3, and so on until you reach the treasure. You cannot jump directly to clue #5 without first following the chain from clue #1.

That is a singly linked list.

Each **node** holds two things: the actual data (the clue text) and the address of the next node (where to find the next clue). The list itself only stores a pointer to the first node — called the **head**. To find any node, you must start at the head and follow the chain.

**Why does this exist when arrays are faster at almost everything?**

Arrays require contiguous memory. Inserting an element in the middle of an array requires shifting all subsequent elements — O(n) work. A linked list sidesteps this entirely: inserting between two nodes means updating two pointers. If you already have a pointer to the insertion point, that is O(1) — regardless of list size.

The linked list is the first data structure where **shape** (how nodes reference each other) carries meaning. Mastering it trains the pointer reasoning that underpins trees, graphs, and virtually every advanced data structure you will learn next.

---

## 2. Internal Working — Nodes and Pointers

### The Node

```
┌──────────────────────┐
│  data  │    next     │
│  (int) │  (pointer)  │
└──────────────────────┘
```

Each node is a separately heap-allocated object. `next` holds the memory address of the next node, or `nullptr` for the last node.

### A List in Memory

```
head
 │
 ▼
┌────────┐    ┌────────┐    ┌────────┐    ┌────────┐
│  10    │    │  20    │    │  30    │    │  40    │
│  next──┼───▶│  next──┼───▶│  next──┼───▶│  null  │
└────────┘    └────────┘    └────────┘    └────────┘
 0x5060        0x71A0        0x3F20        0x9B10
```

Key observations:
- **Non-contiguous:** the four nodes sit at addresses `0x5060`, `0x71A0`, `0x3F20`, `0x9B10` — scattered across the heap wherever the allocator found free space.
- **No index:** there is no formula to compute the address of node `i`. You must traverse from `head`, following `next` pointers, i times.
- **Ownership chain:** each node owns nothing directly — it just knows the address of its neighbour. The list owns only `head`.
- **Termination:** the last node's `next == nullptr`. Every traversal loop must stop here.

### What the List Object Stores

```cpp
struct ListNode {
    int val;
    ListNode* next;
    ListNode(int v) : val(v), next(nullptr) {}
};

class SinglyLinkedList {
    ListNode* head;  // pointer to first node (nullptr if empty)
    int size_;       // optional — tracking size avoids O(n) length queries
};
```

The `SinglyLinkedList` object itself is tiny — just one or two pointers on the stack. All the data lives on the heap, allocated one node at a time.

---

## 3. The Fundamental Trade-off vs Arrays

This table is the conceptual core of this entire document. Every design decision in this structure flows from it.

```
                    Array               Singly Linked List
                 ──────────────────────────────────────────
Access by index    O(1)  ✓✓✓           O(n)  ✗
Search             O(n)                O(n)
Insert at front    O(n) — shift all    O(1)  ✓✓✓  ← SLL wins decisively
Insert at back     O(1) amortized      O(n) — traverse to tail
                   (with tail ptr: O(1))
Insert at middle   O(n) — shift        O(1) — if you have a pointer to prev node
Delete at front    O(n) — shift all    O(1)  ✓✓✓  ← SLL wins decisively
Delete at back     O(1)                O(n) — cannot find prev node
Delete at middle   O(n) — shift        O(1) — if you have a pointer to prev node
Memory overhead    0 per element       1 pointer (8 bytes) per node
Cache behaviour    Excellent ✓✓✓       Poor — every node is a cache miss
Resize             Realloc + copy      N/A — nodes added individually
```

**The decisive insight:** The linked list's O(1) insert/delete advantage only materialises *when you already have a pointer to the relevant position*. If you must first search for that position, the search costs O(n), eliminating the advantage. In practice, this makes linked lists most valuable in scenarios where positions are tracked externally — LRU caches, OS scheduler queues, undo stacks.

---

## 4. Memory Layout — The Cache Penalty

This is what separates engineers who understand linked lists from those who merely use them.

### Each Node Access is a Potential Cache Miss

```
CPU Cache (64-byte lines):
┌───────────────────────────────────────────┐
│ node1 data @ 0x5060 — loaded when we      │
│ access node1. 64 bytes fetched.           │
│ But node2 is at 0x71A0 — NOT in this line │
└───────────────────────────────────────────┘

To traverse: head→node1→node2→node3→node4
Each arrow = potential cache miss (200 cycles vs 4 cycles for hit)

Compare with array sequential scan:
arr[0]→arr[1]→arr[2]→arr[3]
arr[0] load → 64 bytes loaded → arr[0]..arr[15] ALL in cache
arr[1] through arr[15]: FREE (cache hit)
```

**Practical consequence:** On modern hardware, traversing a linked list of n integers can be **10–40× slower** than scanning an equivalent array — not because of algorithmic complexity (both O(n)) but purely due to cache behaviour. For n = 1,000,000 with random node allocation, linked list traversal can take hundreds of milliseconds; array traversal takes a few milliseconds.

**When does the cache penalty not matter?**
- Node processing is expensive (database row, complex object) — cache miss is negligible relative to processing cost
- List is small (< ~1000 nodes) — fits in cache anyway
- Allocation is sequential (pool allocator, arena) — nodes end up contiguous, cache performance recovers

This is why `std::list` is rarely the right answer in practice, despite its theoretical O(1) middle-insert advantage.

---

## 5. Time & Space Complexity

### Operation Complexities

| Operation | Time | Notes |
|---|---|---|
| Access by index i | **O(n)** | Must traverse from head |
| Search by value | **O(n)** | No shortcut; must walk the chain |
| Insert at head | **O(1)** | Update head pointer only |
| Insert at tail | **O(n)** | Must traverse to find last node |
| Insert at tail (with tail ptr) | **O(1)** | Maintain a `tail` pointer |
| Insert after a given node* | **O(1)** | Just pointer rewiring |
| Insert before a given node | **O(n)** | Must find previous node first |
| Delete head | **O(1)** | Update head, free old head |
| Delete tail | **O(n)** | Must find second-to-last node |
| Delete a given node* | **O(1)** | If prev pointer is known |
| Delete by value | **O(n)** | Must search first |
| Get length | **O(n)** unless tracked | Walk entire list |
| Reverse | **O(n)** | Single pass, O(1) space |
| Detect cycle | **O(n)** | Floyd's algorithm, O(1) space |

*\* Only if you already hold a pointer to that node or its predecessor.*

### Space Complexity

| | |
|---|---|
| n nodes | **O(n)** total |
| Per-node overhead | **1 pointer = 8 bytes** (64-bit system) |
| vs array of same data | Array: 0 overhead; SLL: 8 bytes × n overhead |
| Auxiliary space (most ops) | **O(1)** |
| Recursive operations | **O(n)** stack space |

For `int` nodes (4-byte data), the pointer is twice the size of the data — 67% of each node is overhead. For large structs, this ratio inverts and overhead becomes negligible.

---

## 6. Complete C++ Implementation

```cpp
#include <iostream>
#include <stdexcept>
#include <initializer_list>
using namespace std;

// ── Node definition ───────────────────────────────────────────────────────────
struct ListNode {
    int       val;
    ListNode* next;

    explicit ListNode(int v, ListNode* n = nullptr) : val(v), next(n) {}
};

// ── Singly Linked List ────────────────────────────────────────────────────────
class SinglyLinkedList {
private:
    ListNode* head_;
    ListNode* tail_;   // maintained for O(1) push_back
    int       size_;

public:
    // ── Constructors & Destructor ─────────────────────────────────────────────

    SinglyLinkedList() : head_(nullptr), tail_(nullptr), size_(0) {}

    // Initialiser-list constructor: SinglyLinkedList lst = {1, 2, 3, 4};
    SinglyLinkedList(initializer_list<int> vals) : SinglyLinkedList() {
        for (int v : vals) push_back(v);
    }

    // Destructor: must free every node to avoid memory leak
    ~SinglyLinkedList() {
        ListNode* curr = head_;
        while (curr) {
            ListNode* next = curr->next;  // save next BEFORE deleting curr
            delete curr;
            curr = next;
        }
    }

    // Disable copy (implement deep copy if needed)
    SinglyLinkedList(const SinglyLinkedList&) = delete;
    SinglyLinkedList& operator=(const SinglyLinkedList&) = delete;

    // ── Size & State ──────────────────────────────────────────────────────────

    int  size()  const { return size_; }
    bool empty() const { return size_ == 0; }

    // ── Access ────────────────────────────────────────────────────────────────

    int front() const {
        if (!head_) throw runtime_error("list is empty");
        return head_->val;
    }

    int back() const {
        if (!tail_) throw runtime_error("list is empty");
        return tail_->val;
    }

    // O(n) — index-based access; prefer not to use (defeats the purpose)
    int at(int index) const {
        if (index < 0 || index >= size_) throw out_of_range("index out of range");
        ListNode* curr = head_;
        for (int i = 0; i < index; i++) curr = curr->next;
        return curr->val;
    }

    // ── Insertion ─────────────────────────────────────────────────────────────

    // O(1): insert at front
    void push_front(int val) {
        ListNode* node = new ListNode(val, head_);
        head_ = node;
        if (!tail_) tail_ = head_;  // first element — tail == head
        size_++;
    }

    // O(1): insert at back (maintained with tail pointer)
    void push_back(int val) {
        ListNode* node = new ListNode(val, nullptr);
        if (!tail_) {
            head_ = tail_ = node;
        } else {
            tail_->next = node;
            tail_       = node;
        }
        size_++;
    }

    // O(n): insert at index (0 = before current head)
    void insert(int index, int val) {
        if (index < 0 || index > size_) throw out_of_range("index out of range");
        if (index == 0)     { push_front(val); return; }
        if (index == size_) { push_back(val);  return; }

        ListNode* prev = head_;
        for (int i = 0; i < index - 1; i++) prev = prev->next;

        ListNode* node = new ListNode(val, prev->next);
        prev->next = node;
        size_++;
    }

    // O(1): insert after a specific node — given a pointer to that node
    void insert_after(ListNode* prev, int val) {
        if (!prev) return;
        ListNode* node = new ListNode(val, prev->next);
        prev->next = node;
        if (prev == tail_) tail_ = node;  // update tail if inserting at end
        size_++;
    }

    // ── Deletion ──────────────────────────────────────────────────────────────

    // O(1): remove front node
    void pop_front() {
        if (!head_) throw runtime_error("list is empty");
        ListNode* old_head = head_;
        head_ = head_->next;
        if (!head_) tail_ = nullptr;  // list became empty
        delete old_head;
        size_--;
    }

    // O(n): remove back node (need to find the second-to-last)
    void pop_back() {
        if (!head_) throw runtime_error("list is empty");
        if (head_ == tail_) {  // single element
            delete head_;
            head_ = tail_ = nullptr;
            size_--;
            return;
        }
        // Find second-to-last node
        ListNode* prev = head_;
        while (prev->next != tail_) prev = prev->next;
        delete tail_;
        tail_ = prev;
        tail_->next = nullptr;
        size_--;
    }

    // O(n): remove first node with given value; returns true if found
    bool remove(int val) {
        if (!head_) return false;

        // Special case: removing the head
        if (head_->val == val) {
            pop_front();
            return true;
        }

        ListNode* prev = head_;
        while (prev->next && prev->next->val != val) {
            prev = prev->next;
        }
        if (!prev->next) return false;  // not found

        ListNode* to_delete = prev->next;
        prev->next = to_delete->next;
        if (to_delete == tail_) tail_ = prev;  // update tail if needed
        delete to_delete;
        size_--;
        return true;
    }

    // O(n): remove node at index
    void erase(int index) {
        if (index < 0 || index >= size_) throw out_of_range("index out of range");
        if (index == 0) { pop_front(); return; }

        ListNode* prev = head_;
        for (int i = 0; i < index - 1; i++) prev = prev->next;

        ListNode* to_delete = prev->next;
        prev->next = to_delete->next;
        if (to_delete == tail_) tail_ = prev;
        delete to_delete;
        size_--;
    }

    // ── Utilities ─────────────────────────────────────────────────────────────

    // O(n): reverse the list in-place using three-pointer technique
    void reverse() {
        ListNode* prev = nullptr;
        ListNode* curr = head_;
        tail_ = head_;            // old head becomes new tail

        while (curr) {
            ListNode* next = curr->next;  // save next
            curr->next = prev;            // reverse the link
            prev = curr;                  // advance prev
            curr = next;                  // advance curr
        }
        head_ = prev;             // prev ended on old tail = new head
    }

    // O(n): print the list
    void print() const {
        ListNode* curr = head_;
        while (curr) {
            cout << curr->val;
            if (curr->next) cout << " -> ";
            curr = curr->next;
        }
        cout << " -> null\n";
    }

    // O(n): search; returns pointer to first node with val, or nullptr
    ListNode* find(int val) const {
        ListNode* curr = head_;
        while (curr) {
            if (curr->val == val) return curr;
            curr = curr->next;
        }
        return nullptr;
    }

    // Expose head for algorithms that need raw pointer access
    ListNode* head() const { return head_; }
};
```

---

## 7. Core Operations — Visualised

Every pointer manipulation becomes obvious once you draw it. Train yourself to draw before coding.

### Insert at Head

```
Before: head → [20] → [30] → [40] → null

Step 1: Allocate new node [10]
Step 2: new_node->next = head          [10] → [20] → [30] → [40] → null
Step 3: head = new_node       head → [10] → [20] → [30] → [40] → null

Code:
    ListNode* node = new ListNode(val, head_);
    head_ = node;
```

### Insert After a Node

```
Before: head → [10] → [20] → [40] → null
Goal: insert [30] after [20]

Have: pointer `prev` pointing to [20]

Step 1: new_node->next = prev->next    [30] → [40]
Step 2: prev->next = new_node          [20] → [30] → [40]

Result: head → [10] → [20] → [30] → [40] → null

Order matters! Always set new_node->next FIRST.
If you do step 2 first, you lose the pointer to [40].
```

### Delete a Node (given predecessor)

```
Before: head → [10] → [20] → [30] → [40] → null
Goal: delete [30] — we have pointer `prev` pointing to [20]

Step 1: target = prev->next           target points to [30]
Step 2: prev->next = target->next     [20] → [40]  (bypass [30])
Step 3: delete target                 free [30]'s memory

Result: head → [10] → [20] → [40] → null
```

### Reverse — Three-Pointer Technique

```
Initial state:
  prev=null  curr=[10]  next=?
  null  [10]→[20]→[30]→[40]→null

Iteration 1:
  next = curr->next = [20]
  curr->next = prev = null      [10]→null
  prev = curr = [10]
  curr = next = [20]

Iteration 2:
  next = curr->next = [30]
  curr->next = prev = [10]      [20]→[10]→null
  prev = [20],  curr = [30]

Iteration 3:
  next = [40]
  curr->next = [20]             [30]→[20]→[10]→null
  prev = [30],  curr = [40]

Iteration 4:
  next = null
  curr->next = [30]             [40]→[30]→[20]→[10]→null
  prev = [40],  curr = null

Loop ends (curr == null). head = prev = [40].
Result: head → [40] → [30] → [20] → [10] → null  ✓
```

---

## 8. Common Patterns & Techniques

These patterns cover roughly 90% of all linked list interview problems.

### Pattern 1: Fast and Slow Pointers (Floyd's Algorithm)

Two pointers advancing at different speeds. The tortoise (slow) moves one step at a time; the hare (fast) moves two. Used for cycle detection, finding the middle, and finding the kth-from-last node.

```cpp
// Find the middle node — used in merge sort and palindrome check
// For even-length list, returns the SECOND middle (upper median)
ListNode* findMiddle(ListNode* head) {
    ListNode* slow = head;
    ListNode* fast = head;
    while (fast && fast->next) {
        slow = slow->next;         // advance 1
        fast = fast->next->next;   // advance 2
    }
    return slow;  // slow is at middle when fast reaches end
}

// Detect cycle — Floyd's cycle detection
bool hasCycle(ListNode* head) {
    ListNode* slow = head;
    ListNode* fast = head;
    while (fast && fast->next) {
        slow = slow->next;
        fast = fast->next->next;
        if (slow == fast) return true;  // they met inside the cycle
    }
    return false;  // fast reached null → no cycle
}

// Find cycle entry point (see Interview Problem #2 for full explanation)
ListNode* detectCycle(ListNode* head) {
    ListNode* slow = head, *fast = head;
    bool hasCycle = false;
    while (fast && fast->next) {
        slow = slow->next;
        fast = fast->next->next;
        if (slow == fast) { hasCycle = true; break; }
    }
    if (!hasCycle) return nullptr;
    slow = head;                          // reset slow to head
    while (slow != fast) {               // advance both one step at a time
        slow = slow->next;
        fast = fast->next;
    }
    return slow;                          // meeting point = cycle entry
}
```

### Pattern 2: Dummy Head Node

A dummy (sentinel) node before the real head eliminates special cases for operations on the head. Drastically simplifies code. Use this in almost every linked list problem.

```cpp
// Without dummy: must handle "head deletion" as a special case
// With dummy:    head deletion is just a regular node deletion
ListNode* removeElements(ListNode* head, int val) {
    ListNode dummy(0, head);   // dummy->next = real head
    ListNode* curr = &dummy;

    while (curr->next) {
        if (curr->next->val == val) {
            ListNode* to_del = curr->next;
            curr->next = to_del->next;
            delete to_del;
        } else {
            curr = curr->next;
        }
    }
    return dummy.next;  // new real head
}
```

### Pattern 3: Two Pointers (Fixed Gap)

Maintain two pointers with a fixed gap of k nodes between them. When the front pointer reaches the end, the back pointer is at the correct position.

```cpp
// Remove kth node from the end in one pass
// Key: advance fast pointer k steps first, then move both until fast reaches null
ListNode* removeNthFromEnd(ListNode* head, int k) {
    ListNode dummy(0, head);
    ListNode* fast = &dummy;
    ListNode* slow = &dummy;

    for (int i = 0; i <= k; i++) fast = fast->next;  // gap = k+1 (so slow is at prev)

    while (fast) {
        slow = slow->next;
        fast = fast->next;
    }
    // slow->next is the kth node from end
    ListNode* to_del = slow->next;
    slow->next = to_del->next;
    delete to_del;

    return dummy.next;
}
```

### Pattern 4: Reversal (In-Place and Partial)

```cpp
// Reverse a sublist from position left to right (1-indexed)
ListNode* reverseBetween(ListNode* head, int left, int right) {
    ListNode dummy(0, head);
    ListNode* prev = &dummy;

    // Move prev to node just before position `left`
    for (int i = 1; i < left; i++) prev = prev->next;

    ListNode* curr = prev->next;  // curr starts at position `left`

    // Reverse (right - left) times using the "insert-at-front" trick
    for (int i = 0; i < right - left; i++) {
        ListNode* next = curr->next;
        curr->next     = next->next;     // unlink next
        next->next     = prev->next;     // relink next before curr
        prev->next     = next;           // update front of reversed section
    }
    return dummy.next;
}

/*
Example: reverseBetween([1→2→3→4→5], left=2, right=4)

Initial:  dummy → [1] → [2] → [3] → [4] → [5]
                   ↑prev  ↑curr

i=0: next=[3], curr->next=[4], next->next=[2], prev->next=[3]
     dummy → [1] → [3] → [2] → [4] → [5]

i=1: next=[4], curr->next=[5], next->next=[3], prev->next=[4]
     dummy → [1] → [4] → [3] → [2] → [5]

Result: [1] → [4] → [3] → [2] → [5]  ✓
*/
```

### Pattern 5: Merge Two Sorted Lists

The building block of merge sort on linked lists. Master this before attempting anything more complex.

```cpp
// Merge two sorted linked lists — O(n+m) time, O(1) space
ListNode* mergeTwoLists(ListNode* l1, ListNode* l2) {
    ListNode dummy(0);
    ListNode* tail = &dummy;

    while (l1 && l2) {
        if (l1->val <= l2->val) {
            tail->next = l1;
            l1 = l1->next;
        } else {
            tail->next = l2;
            l2 = l2->next;
        }
        tail = tail->next;
    }
    tail->next = l1 ? l1 : l2;  // attach remaining nodes
    return dummy.next;
}
```

---

## 9. Interview Problems

### Problem 1: Reverse a Linked List

**Problem:** Reverse a singly linked list in-place. Return the new head.

**Example:** `1 → 2 → 3 → 4 → 5 → null` → `5 → 4 → 3 → 2 → 1 → null`

**Thought process in an interview:**

> "I need to reverse all the `next` pointers. After reversal, each node's `next` should point to what was previously its predecessor. I'll maintain three pointers: `prev` (starts null), `curr` (starts at head), `next` (temporary). For each node: save `next`, flip `curr->next` to point to `prev`, advance both `prev` and `curr`."

```cpp
// Iterative — O(n) time, O(1) space — ALWAYS prefer this
ListNode* reverseList(ListNode* head) {
    ListNode* prev = nullptr;
    ListNode* curr = head;

    while (curr) {
        ListNode* next_node = curr->next;  // 1. save next
        curr->next = prev;                 // 2. reverse the link
        prev = curr;                       // 3. advance prev
        curr = next_node;                  // 4. advance curr
    }
    return prev;  // prev is now pointing to the last node = new head
}

// Recursive — O(n) time, O(n) space (call stack)
// Less preferred but interviewers may ask for it to test recursion understanding
ListNode* reverseListRecursive(ListNode* head) {
    // Base case: empty or single node
    if (!head || !head->next) return head;

    // Recurse on the rest of the list
    ListNode* new_head = reverseListRecursive(head->next);

    // head->next is now the LAST node of the reversed sublist
    // Make it point back to head
    head->next->next = head;
    head->next = nullptr;

    return new_head;
}

/*
Recursive dry run for [1→2→3]:

reverseList(1):
  reverseList(2):
    reverseList(3):
      return 3  ← base case, new_head=3
    3->next = 2  → [3→2]
    2->next = null
    return 3
  3->next was 2, now: 2->next = 1  → [3→2→1]
  1->next = null
  return 3

Result: 3 → 2 → 1 → null  ✓
*/
```

**Edge cases:**
- `nullptr` → return `nullptr`
- Single node → return it unchanged
- Two nodes → must work correctly: `[1→2]` → `[2→1]`

**Follow-up questions interviewers ask:**
- "Can you do it recursively?" — yes, shown above, but note O(n) stack space
- "Can you reverse in groups of k?" — see LeetCode 25, uses the above as a subroutine
- "What if it's a doubly linked list?" — simpler: swap `next` and `prev` pointers for each node

---

### Problem 2: Linked List Cycle Detection and Entry Point

**Problem:** Given the head of a linked list, determine if it has a cycle. If it does, return the node where the cycle begins. If not, return `nullptr`.

**Why Floyd's algorithm works — the math:**

```
No cycle:
  fast pointer reaches null → no cycle

With cycle:
  Let:
    F = distance from head to cycle entry
    C = cycle length
    k = distance from entry to where slow and fast first meet

  When they meet:
    slow travelled: F + k
    fast travelled: F + k + m×C  (fast did m extra laps of the cycle)

  fast = 2 × slow:
    F + k + m×C = 2(F + k)
    m×C = F + k
    F = m×C - k

  Reset slow to head. Both advance one step at a time:
    slow travels F steps to reach entry
    fast travels F = m×C - k steps from meeting point
    = one full cycle (m=1) minus k steps from entry
    = arrives at entry exactly

  They meet at the cycle entry.  ✓
```

```cpp
// Time: O(n) | Space: O(1)
ListNode* detectCycle(ListNode* head) {
    if (!head || !head->next) return nullptr;

    ListNode* slow = head;
    ListNode* fast = head;

    // Phase 1: Detect if cycle exists and find meeting point
    while (fast && fast->next) {
        slow = slow->next;
        fast = fast->next->next;
        if (slow == fast) break;
    }

    // No cycle
    if (!fast || !fast->next) return nullptr;

    // Phase 2: Find cycle entry
    // Reset slow to head. Advance both one step at a time.
    slow = head;
    while (slow != fast) {
        slow = slow->next;
        fast = fast->next;
    }
    return slow;  // cycle entry point
}

/*
Visualised example:
  head → [1] → [2] → [3] → [4] → [5]
                              ↑           |
                              └───────────┘ (5 points back to 3)
  F = 2 (head to entry [3])
  C = 3 ([3]→[4]→[5]→[3])

Phase 1 trace:
  step 1: slow=[2], fast=[3]
  step 2: slow=[3], fast=[5]
  step 3: slow=[4], fast=[4]  ← MEET at [4], so k=1

Phase 2:
  slow reset to [1], fast stays at [4]
  step 1: slow=[2], fast=[5]
  step 2: slow=[3], fast=[3]  ← MEET at [3] = cycle entry ✓
*/
```

**Edge cases:**
- Empty list → `nullptr`
- Single node, no self-loop → `nullptr`
- Single node with self-loop (`head->next = head`) → `head`
- Cycle at the very head — works correctly with the math
- Cycle at the very last node — works correctly

**Alternative approach (not O(1) space but often mentioned):** Use a hash set of visited node addresses. First repeated address is the cycle entry. O(n) time, O(n) space.

---

### Problem 3: Merge K Sorted Linked Lists

**Problem:** Given an array of k sorted linked lists, merge them all into one sorted linked list.

**Example:** `[[1→4→5], [1→3→4], [2→6]]` → `[1→1→2→3→4→4→5→6]`

**Thought process — build from simpler subproblems:**

> "I already know how to merge 2 sorted lists in O(n+m) time. For k lists of total n nodes:"
>
> "Naive: merge lists one by one. Merging list 2 into list 1 costs O(n). Merging list 3 costs O(2n). Total: O(kn). Too slow for large k."
>
> "Better: use a min-heap. Keep the current front node of each list in the heap. Pop the minimum, add it to result, push the next node from that list. Every node is pushed and popped once → O(n log k)."
>
> "Alternatively: divide and conquer — merge pairs of lists, then merge the results. Like merge sort. Also O(n log k) with O(log k) extra space for recursion."

```cpp
#include <queue>
#include <vector>

// ── Approach 1: Min-Heap — O(n log k) time, O(k) space ───────────────────────
ListNode* mergeKLists(vector<ListNode*>& lists) {
    // Min-heap: always gives us the node with the smallest current value
    auto cmp = [](ListNode* a, ListNode* b) { return a->val > b->val; };
    priority_queue<ListNode*, vector<ListNode*>, decltype(cmp)> pq(cmp);

    // Seed the heap with the head of each list
    for (ListNode* head : lists) {
        if (head) pq.push(head);
    }

    ListNode dummy(0);
    ListNode* tail = &dummy;

    while (!pq.empty()) {
        ListNode* smallest = pq.top(); pq.pop();
        tail->next = smallest;
        tail = tail->next;
        if (smallest->next) pq.push(smallest->next);  // push next from same list
    }
    tail->next = nullptr;
    return dummy.next;
}

// ── Approach 2: Divide and Conquer — O(n log k) time, O(log k) stack space ───
// Merge pairs: [L0+L1], [L2+L3], ... then merge the results — like merge sort
ListNode* mergeTwoLists(ListNode* l1, ListNode* l2) {
    ListNode dummy(0);
    ListNode* tail = &dummy;
    while (l1 && l2) {
        if (l1->val <= l2->val) { tail->next = l1; l1 = l1->next; }
        else                    { tail->next = l2; l2 = l2->next; }
        tail = tail->next;
    }
    tail->next = l1 ? l1 : l2;
    return dummy.next;
}

ListNode* mergeKListsDC(vector<ListNode*>& lists) {
    if (lists.empty()) return nullptr;
    int n = lists.size();

    // Repeatedly halve the number of lists
    for (int step = 1; step < n; step *= 2) {
        for (int i = 0; i + step < n; i += 2 * step) {
            lists[i] = mergeTwoLists(lists[i], lists[i + step]);
        }
    }
    return lists[0];
}

/*
Divide & conquer trace for k=4 lists [L0,L1,L2,L3]:

step=1:  L0=merge(L0,L1),  L2=merge(L2,L3)
step=2:  L0=merge(L0,L2)

Total merge operations: like a tournament bracket.
Log2(4) = 2 rounds. Each round processes all n nodes → O(n log k). ✓
*/
```

**Complexity comparison:**

| Approach | Time | Space | Notes |
|---|---|---|---|
| Merge one-by-one | O(kn) | O(1) | Too slow for large k |
| Min-heap | O(n log k) | O(k) | Best for streaming data |
| Divide and conquer | O(n log k) | O(log k) | Best overall; clean code |

**Edge cases:**
- Empty input array → `nullptr`
- All lists are `nullptr` → `nullptr`
- k=1 → return the single list unchanged
- Lists of very different lengths — both approaches handle correctly

---

## 10. Real-World Uses

| Domain | Use Case | Why Linked List |
|---|---|---|
| OS kernel | Process/thread scheduler run queue | O(1) insert/remove from any position with a pointer |
| OS kernel | Free memory block list (buddy allocator) | Nodes are the free blocks themselves — zero overhead |
| Browser | Back/forward navigation history | Doubly linked list; O(1) navigation |
| Text editors | Undo/redo history chains | Each edit is a node; O(1) undo |
| Database | WAL (Write-Ahead Log) buffer chain | Sequential append, O(1) prepend of log records |
| Compiler | Symbol table chaining (open hashing) | Each hash bucket is a linked list |
| Networking | TCP/IP packet reassembly buffer | Out-of-order packets linked by sequence number |
| Java / JVM | `LinkedList<T>`, HashMap bucket chains | Standard library use |
| Hardware | CPU instruction pipeline stages | Conceptually a linked list of stages |
| Game dev | Entity update lists | Entities added/removed without shifting |

**The deepest real-world use — OS memory allocator:**  
The kernel's free list for memory pages is literally a linked list where the `next` pointer is stored inside the free page itself. There is no separate node allocation — the free memory block contains its own pointer. This gives O(1) allocation and deallocation with zero overhead per block. This is the intrusive linked list pattern.

---

## 11. Edge Cases & Pitfalls

### Pitfall 1: Losing the Next Pointer Before Reversing a Link

```cpp
// WRONG — you lost curr->next before saving it
curr->next = prev;    // reversed the link
prev = curr;
curr = curr->next;    // curr->next is now prev — infinite loop or null!

// CORRECT — always save next first
ListNode* next_node = curr->next;   // save
curr->next = prev;                  // reverse
prev = curr;                        // advance prev
curr = next_node;                   // advance curr using saved pointer
```

### Pitfall 2: Forgetting to Handle the Empty List

```cpp
// Every function that accesses head should guard against nullptr
ListNode* head = nullptr;
cout << head->val;    // CRASH — segmentation fault

// Pattern: always check before dereferencing
if (!head) return nullptr;
if (!head || !head->next) return head;  // common two-node guard
```

### Pitfall 3: Memory Leaks — Not Deleting Removed Nodes

```cpp
// WRONG: node is unlinked but memory is never freed
void pop_front() {
    head_ = head_->next;   // old head is now unreachable — LEAKED
}

// CORRECT: always delete what you unlink
void pop_front() {
    ListNode* old = head_;
    head_ = head_->next;
    delete old;             // free the memory
}
```

### Pitfall 4: Not Updating the Tail Pointer

```cpp
// If you maintain a tail_ pointer, every operation that touches the last node
// must update tail_ correctly. Easy to forget in:
//   - pop_back (tail_ must become the second-to-last)
//   - insert_after when inserting at the end
//   - reverse (old head becomes new tail)

// Missing tail_ update causes silent bugs — tail_ points to freed memory
```

### Pitfall 5: Off-by-One in Two-Pointer Gap Problems

```cpp
// "Remove nth node from end"
// You want slow to land on the node BEFORE the target (so you can unlink it)
// That means the gap should be k+1, not k

// WRONG: gap = k (slow lands ON the target, can't unlink)
for (int i = 0; i < k; i++) fast = fast->next;

// CORRECT: start both at dummy, gap = k+1 (slow is at predecessor of target)
ListNode dummy(0, head);
ListNode* fast = &dummy, *slow = &dummy;
for (int i = 0; i <= k; i++) fast = fast->next;
```

### Pitfall 6: Cycle Causes Infinite Loop in Traversal

```cpp
// Any traversal assuming null-termination will loop forever on a cyclic list
void print(ListNode* head) {
    while (head) {          // INFINITE LOOP if there's a cycle
        cout << head->val;
        head = head->next;
    }
}

// Always detect cycles before traversing untrusted input
// Or use Floyd's algorithm proactively (see Pattern 1)
```

### Pitfall 7: Returning the Wrong Head After Operations

```cpp
// If you modified the head inside a function, you MUST return the new head.
// Classic mistake: function modifies head locally but the caller still uses the old one.

// WRONG approach (modifying in-place without dummy or returning):
void reverseList(ListNode* head) {  // head is a local copy — original unchanged!
    // ...
}

// CORRECT: return the new head, or pass head by reference/pointer-to-pointer
ListNode* reverseList(ListNode* head) { /* ... */ return new_head; }
// OR
void reverseList(ListNode** head) { /* ... */ *head = new_head; }
```

### Pitfall 8: Null-Dereferencing in Fast Pointer Checks

```cpp
// When using fast/slow pointers, check BOTH fast AND fast->next before advancing
while (fast->next && fast->next->next) {     // WRONG if fast itself can be null
    fast = fast->next->next;
}

// CORRECT — check fast first, then fast->next
while (fast && fast->next) {
    fast = fast->next->next;
}
```

---

## 12. Comparison

| Feature | Singly Linked List | Doubly Linked List | `std::vector` | `std::list` |
|---|---|---|---|---|
| Memory per node | data + 1 ptr (8B) | data + 2 ptr (16B) | data + 0 | data + 2 ptr (16B) |
| Random access | O(n) | O(n) | O(1) | O(n) |
| Push front | O(1) | O(1) | O(n) | O(1) |
| Push back | O(1) with tail | O(1) | O(1) amortized | O(1) |
| Delete front | O(1) | O(1) | O(n) | O(1) |
| Delete back | O(n) | O(1) | O(1) | O(1) |
| Delete middle | O(n) search + O(1) | O(n) search + O(1) | O(n) | O(n) search + O(1) |
| Backward traversal | Not possible | O(1) per step | O(1) per step | O(1) per step |
| Cache performance | Poor | Poor | Excellent | Poor |
| Can find prev node | No | O(1) via `prev` | N/A | O(1) via iterator |
| Cycle possible | Yes | Yes | No | No |

**When to choose singly linked list:**
- You only traverse forward
- You frequently insert/remove at the front
- Memory is tight (save one pointer vs doubly linked)
- Building stacks or queues with O(1) operations at one end
- Interview problems (almost all linked list problems use singly linked nodes)

---

## 13. Self-Test Questions

1. **Draw the state of the list and all pointers after inserting 25 between nodes 20 and 30 in `[10→20→30→40]`. Which pointer must be set first and why?**

2. **Why is deleting the tail of a singly linked list O(n) even when you have a tail pointer?**

3. **Trace Floyd's algorithm on `[1→2→3→4→5→3]` (cycle at node 3). Show every step of both phases.**

4. **What is the dummy node technique? Write `removeAllOccurrences(head, val)` using it.**

5. **What happens to `tail_` when you reverse a singly linked list in-place? Write the code that handles this correctly.**

6. **You call `reverseList(head)` and the original list is unchanged after the call. What went wrong?**

7. **Why is `std::list` (doubly linked list) almost never the right choice in C++ despite its O(1) middle-insert? What data would change your mind?**

8. **Write a function to check if a linked list is a palindrome in O(n) time and O(1) space.** *(Hint: find middle, reverse second half, compare, restore.)*

9. **Merge K sorted lists: if k=100 and each list has 1000 nodes, compare the time complexity of the naive approach vs the heap approach. Give actual numbers.**

10. **What is an intrusive linked list? How does the Linux kernel use it for the process scheduler?**

---

## Quick Reference Card

```
Singly Linked List — trade O(1) access for O(1) front insert/delete.

Node:  [data | next→]   Last node: next = nullptr
List holds: head pointer (+ optional tail pointer + size)

Key complexities:
  Access i-th element:  O(n)    ← no index formula
  Search:               O(n)
  Insert at head:       O(1)    ← main advantage over array
  Insert at tail:       O(1) with tail ptr
  Insert after node*:   O(1)    ← if you have the pointer
  Delete head:          O(1)
  Delete tail:          O(n)    ← singly linked's main weakness
  Delete after node*:   O(1)

Interview patterns (master these 5):
  1. Fast/slow pointers  — cycle detection, find middle, kth from end
  2. Dummy head node     — eliminates head special cases
  3. Fixed-gap two ptr   — kth from end, remove nth from end
  4. In-place reversal   — three pointers: prev, curr, next
  5. Merge pattern       — dummy tail, advance smaller head

Pointer rules (never break these):
  → Save next BEFORE reversing any link
  → Always check for nullptr before dereferencing
  → Update tail_ in every operation that touches the last node
  → delete every node you unlink (no memory leaks)
  → Return the new head from any function that may change it
```

---

*Previous: [02 — Dynamic Array / std::vector](./02_dynamic_array_vector.md)*
*Next: [04 — Doubly Linked List](./04_doubly_linked_list.md)*
*See also: [Stack](./05_stack.md) | [Queue](./06_queue.md) | [Floyd's Cycle Detection](./algorithms/cycle_detection.md)*

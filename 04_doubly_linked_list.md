# Doubly Linked List

> **Curriculum position:** Linear Structures → #4
> **Interview weight:** ★★★★☆ High — appears directly in LRU Cache, browser history, and text editor design problems
> **Difficulty:** Beginner-Intermediate — same pointer logic as SLL, doubled in each node
> **Prerequisite:** [03 — Singly Linked List](./03_singly_linked_list.md)

---

## Table of Contents

1. [Intuition First — What Does the Second Pointer Buy You?](#1-intuition-first)
2. [Internal Working — The Bidirectional Chain](#2-internal-working)
3. [Sentinel Nodes — The Architecture Decision That Changes Everything](#3-sentinel-nodes)
4. [Time & Space Complexity](#4-time--space-complexity)
5. [Complete C++ Implementation](#5-complete-c-implementation)
6. [Core Operations — Visualised](#6-core-operations--visualised)
7. [Common Patterns & Techniques](#7-common-patterns--techniques)
8. [Interview Problems](#8-interview-problems)
9. [Real-World Uses](#9-real-world-uses)
10. [Edge Cases & Pitfalls](#10-edge-cases--pitfalls)
11. [Comparison: DLL vs SLL vs std::list vs deque](#11-comparison)
12. [Self-Test Questions](#12-self-test-questions)

---

## 1. Intuition First

A singly linked list is a one-way street. You can drive forward, but if you miss your turn you have to go all the way around the block to get back. That one restriction — no backward movement — is the source of all its limitations: O(n) tail delete, no backward traversal, inability to delete a node given only a pointer to it.

A doubly linked list adds a second pointer to each node: `prev`. Now every node knows both its successor and its predecessor. You have turned a one-way street into a two-way street.

That single addition unlocks four capabilities that the singly linked list simply cannot provide:

1. **O(1) deletion given only a pointer to the node** — you no longer need to find the predecessor by traversal. The node itself knows who comes before it.
2. **O(1) tail deletion** — the tail knows its predecessor via `prev`. Remove it, update `tail`, done.
3. **Backward traversal** — iterate from tail to head in O(n).
4. **Bidirectional iteration** — used by `std::list` iterators and all doubly linked list-based caches.

The cost: one extra pointer per node (8 bytes on a 64-bit system), and every insert/delete operation must now maintain two links instead of one — doubling the pointer bookkeeping and the opportunity for bugs.

This trade-off makes the DLL the structure of choice for any problem that requires **O(1) arbitrary deletion with a known pointer** — the defining requirement of the LRU Cache, the OS page replacement policy, text editor cursor mechanics, and the `std::list` container.

---

## 2. Internal Working — The Bidirectional Chain

### The Node

```
┌──────────────────────────────────────────┐
│   prev   │    data    │       next       │
│ (pointer)│   (int)    │    (pointer)     │
└──────────────────────────────────────────┘
```

Each node has three fields:
- `prev` — address of the previous node (or `nullptr` for the head)
- `data` — the stored value
- `next` — address of the next node (or `nullptr` for the tail)

### A List in Memory

```
nullptr
   ↑
┌────────┐   ←prev─   ┌────────┐   ←prev─   ┌────────┐   ←prev─   ┌────────┐
│  10    │            │  20    │            │  30    │            │  40    │
│prev=null│  ─next→   │prev=10 │  ─next→   │prev=20 │  ─next→   │prev=30 │
└────────┘            └────────┘            └────────┘            └────────┘
    ↑                                                                   ↑
   head                                                               tail
                                                                   next=null
```

Every node participates in two chains simultaneously:
- The forward chain: `head → 10 → 20 → 30 → 40 → null`
- The backward chain: `tail → 40 → 30 → 20 → 10 → null`

Both chains must be kept consistent at all times. This invariant — **every forward link has a matching backward link** — is the source of most DLL bugs.

### What the List Object Stores

```cpp
struct DLLNode {
    int      val;
    DLLNode* prev;
    DLLNode* next;
    DLLNode(int v) : val(v), prev(nullptr), next(nullptr) {}
};

class DoublyLinkedList {
    DLLNode* head;   // first real node
    DLLNode* tail;   // last real node
    int      size_;
};
```

---

## 3. Sentinel Nodes — The Architecture Decision That Changes Everything

This is the most important design concept in DLL implementation. Every production DLL uses this. Most interview candidates do not know it exists.

### The Problem with Null-Terminated Lists

Without sentinels, every operation has to handle special cases for empty lists, single-node lists, operations at the head, and operations at the tail. The code becomes a forest of `if (!head)`, `if (head == tail)`, `if (!prev)`, `if (!next)` guards. Each guard is a potential bug.

```cpp
// Inserting after node 'p' — null-terminated version
void insert_after(DLLNode* p, int val) {
    DLLNode* node = new DLLNode(val);
    node->next = p->next;
    node->prev = p;
    if (p->next) p->next->prev = node;  // ← special case: p was the tail
    else tail = node;                    // ← another special case
    p->next = node;
    size_++;
}
```

### The Sentinel Solution

Add two **dummy nodes** — a `head sentinel` and a `tail sentinel` — that are always present, even in an "empty" list. They never hold real data. They exist purely to eliminate boundary conditions.

```
Empty list with sentinels:
   ┌──────────────┐            ┌──────────────┐
   │  HEAD GUARD  │   ←──→    │  TAIL GUARD  │
   │  prev=null   │            │  next=null   │
   └──────────────┘            └──────────────┘

List with data [10, 20, 30]:
   ┌──────┐   ←──→   ┌──────┐   ←──→   ┌──────┐   ←──→   ┌──────┐   ←──→   ┌──────┐
   │ HEAD │          │  10  │          │  20  │          │  30  │          │ TAIL │
   │guard │          │      │          │      │          │      │          │guard │
   └──────┘          └──────┘          └──────┘          └──────┘          └──────┘
```

**The key invariant:** The first real node is always `head_guard->next`. The last real node is always `tail_guard->prev`. The "empty" check is simply `head_guard->next == tail_guard`.

Now `insert_after(p, val)` has **zero special cases**:

```cpp
void insert_after(DLLNode* p, int val) {
    DLLNode* node    = new DLLNode(val);
    node->next       = p->next;   // could be tail_guard — handled correctly
    node->prev       = p;
    p->next->prev    = node;      // no null check needed — p->next is always valid
    p->next          = node;
    size_++;
}
```

This works whether `p` is the head guard, the tail guard's predecessor, a middle node, or any other position. The guards absorb all boundary conditions.

**Sentinel nodes are used in:** `std::list` (GCC's implementation uses them), Linux kernel list (`list_head`), every production LRU cache implementation.

From this point forward, the implementation in this document uses sentinel nodes.

---

## 4. Time & Space Complexity

### Operation Complexities

| Operation | Time | Notes |
|---|---|---|
| Access by index i | **O(n)** | Traverse from head or tail (whichever is closer) |
| Search by value | **O(n)** | Walk the chain |
| Insert at head | **O(1)** | `insert_after(head_guard, val)` |
| Insert at tail | **O(1)** | `insert_before(tail_guard, val)` |
| Insert after node* | **O(1)** | Four pointer updates, no traversal |
| Insert before node* | **O(1)** | Use `node->prev` — singly LL would need O(n) |
| Delete head | **O(1)** | `remove(head_guard->next)` |
| Delete tail | **O(1)** | `remove(tail_guard->prev)` — **DLL advantage over SLL** |
| Delete node given pointer* | **O(1)** | No need to find predecessor — **DLL's defining advantage** |
| Delete by value | **O(n)** | Must search first |
| Get length | **O(1)** | With tracked `size_` counter |
| Reverse traversal | **O(n)** | Follow `prev` pointers |
| Reverse in-place | **O(n)** | Swap `next` and `prev` for every node |

*\* Requires a direct pointer to the node. If you need to search first, add O(n) for the search.*

### Space Complexity

| | |
|---|---|
| n nodes | **O(n)** |
| Per-node overhead | **2 pointers = 16 bytes** (vs 8 bytes for SLL) |
| Sentinel overhead | **2 nodes** (constant, amortised to zero) |
| Auxiliary space (most ops) | **O(1)** |

For `int` data (4 bytes), each DLL node is 4 + 8 + 8 = 20 bytes — the two pointers are 4× the data size. For large structs, this ratio becomes negligible.

---

## 5. Complete C++ Implementation

```cpp
#include <iostream>
#include <stdexcept>
#include <initializer_list>
using namespace std;

struct DLLNode {
    int      val;
    DLLNode* prev;
    DLLNode* next;
    explicit DLLNode(int v = 0) : val(v), prev(nullptr), next(nullptr) {}
};

class DoublyLinkedList {
private:
    DLLNode* head_guard_;   // sentinel: always before the first real node
    DLLNode* tail_guard_;   // sentinel: always after the last real node
    int      size_;

    // ── Core primitive: splice node out of the chain ──────────────────────────
    // Removes node from wherever it is. Caller is responsible for delete.
    // This is the fundamental DLL operation — all deletions use it.
    void unlink(DLLNode* node) {
        node->prev->next = node->next;
        node->next->prev = node->prev;
        node->prev = node->next = nullptr;
        size_--;
    }

    // ── Core primitive: insert node between two adjacent nodes ────────────────
    // Inserts `node` between `before` and `after`.
    // Caller ensures before->next == after (i.e., they are adjacent).
    void link_between(DLLNode* before, DLLNode* node, DLLNode* after) {
        node->prev   = before;
        node->next   = after;
        before->next = node;
        after->prev  = node;
        size_++;
    }

public:
    // ── Constructors & Destructor ─────────────────────────────────────────────

    DoublyLinkedList() : size_(0) {
        head_guard_ = new DLLNode(0);
        tail_guard_ = new DLLNode(0);
        head_guard_->next = tail_guard_;
        tail_guard_->prev = head_guard_;
        // head_guard_->prev and tail_guard_->next remain nullptr
    }

    DoublyLinkedList(initializer_list<int> vals) : DoublyLinkedList() {
        for (int v : vals) push_back(v);
    }

    ~DoublyLinkedList() {
        DLLNode* curr = head_guard_->next;
        while (curr != tail_guard_) {
            DLLNode* next = curr->next;
            delete curr;
            curr = next;
        }
        delete head_guard_;
        delete tail_guard_;
    }

    // Disable copy — add deep copy if your use case needs it
    DoublyLinkedList(const DoublyLinkedList&)            = delete;
    DoublyLinkedList& operator=(const DoublyLinkedList&) = delete;

    // ── Capacity ──────────────────────────────────────────────────────────────

    int  size()  const { return size_; }
    bool empty() const { return head_guard_->next == tail_guard_; }

    // Expose sentinels for algorithms that iterate externally
    DLLNode* head_sentinel() const { return head_guard_; }
    DLLNode* tail_sentinel() const { return tail_guard_; }
    DLLNode* front_node()    const { return empty() ? nullptr : head_guard_->next; }
    DLLNode* back_node()     const { return empty() ? nullptr : tail_guard_->prev; }

    // ── Element Access ────────────────────────────────────────────────────────

    int front() const {
        if (empty()) throw runtime_error("list is empty");
        return head_guard_->next->val;
    }

    int back() const {
        if (empty()) throw runtime_error("list is empty");
        return tail_guard_->prev->val;
    }

    // O(n) — bidirectional: approach from nearer end
    int at(int index) const {
        if (index < 0 || index >= size_) throw out_of_range("index out of range");
        DLLNode* curr;
        if (index <= size_ / 2) {
            curr = head_guard_->next;
            for (int i = 0; i < index; i++) curr = curr->next;
        } else {
            curr = tail_guard_->prev;
            for (int i = size_ - 1; i > index; i--) curr = curr->prev;
        }
        return curr->val;
    }

    // ── Insertion ─────────────────────────────────────────────────────────────

    // O(1) — insert at front
    void push_front(int val) {
        link_between(head_guard_, new DLLNode(val), head_guard_->next);
    }

    // O(1) — insert at tail
    void push_back(int val) {
        link_between(tail_guard_->prev, new DLLNode(val), tail_guard_);
    }

    // O(1) — insert new node immediately after `node`
    // `node` must be a real node (not tail_guard_)
    DLLNode* insert_after(DLLNode* node, int val) {
        DLLNode* new_node = new DLLNode(val);
        link_between(node, new_node, node->next);
        return new_node;
    }

    // O(1) — insert new node immediately before `node`
    // `node` must be a real node (not head_guard_)
    DLLNode* insert_before(DLLNode* node, int val) {
        DLLNode* new_node = new DLLNode(val);
        link_between(node->prev, new_node, node);
        return new_node;
    }

    // ── Deletion ──────────────────────────────────────────────────────────────

    // O(1) — remove front
    void pop_front() {
        if (empty()) throw runtime_error("list is empty");
        DLLNode* node = head_guard_->next;
        unlink(node);
        delete node;
    }

    // O(1) — remove back (DLL advantage: SLL needs O(n) for this)
    void pop_back() {
        if (empty()) throw runtime_error("list is empty");
        DLLNode* node = tail_guard_->prev;
        unlink(node);
        delete node;
    }

    // O(1) — remove a specific node given its pointer
    // The defining DLL advantage: no need to find the predecessor
    void remove_node(DLLNode* node) {
        // Safety: do not remove sentinels
        if (node == head_guard_ || node == tail_guard_) return;
        unlink(node);
        delete node;
    }

    // O(n) — remove first occurrence of value
    bool remove_value(int val) {
        DLLNode* curr = head_guard_->next;
        while (curr != tail_guard_) {
            if (curr->val == val) {
                remove_node(curr);
                return true;
            }
            curr = curr->next;
        }
        return false;
    }

    // ── Move Operations (O(1), no allocation) ────────────────────────────────
    // Move an existing node to the front — used in LRU Cache
    void move_to_front(DLLNode* node) {
        unlink(node);
        size_++;                          // unlink decremented; link_between increments
        link_between(head_guard_, node, head_guard_->next);
    }

    // Move an existing node to the back
    void move_to_back(DLLNode* node) {
        unlink(node);
        size_++;
        link_between(tail_guard_->prev, node, tail_guard_);
    }

    // ── Reversal ─────────────────────────────────────────────────────────────
    // O(n) — swap prev and next for every node including sentinels
    void reverse() {
        DLLNode* curr = head_guard_;
        while (curr) {
            swap(curr->prev, curr->next);
            curr = curr->prev;            // was curr->next before the swap
        }
        swap(head_guard_, tail_guard_);   // swap sentinel roles
    }

    // ── Utilities ─────────────────────────────────────────────────────────────

    void print_forward() const {
        DLLNode* curr = head_guard_->next;
        while (curr != tail_guard_) {
            cout << curr->val;
            if (curr->next != tail_guard_) cout << " ⇄ ";
            curr = curr->next;
        }
        cout << "\n";
    }

    void print_backward() const {
        DLLNode* curr = tail_guard_->prev;
        while (curr != head_guard_) {
            cout << curr->val;
            if (curr->prev != head_guard_) cout << " ⇄ ";
            curr = curr->prev;
        }
        cout << "\n";
    }

    DLLNode* find(int val) const {
        DLLNode* curr = head_guard_->next;
        while (curr != tail_guard_) {
            if (curr->val == val) return curr;
            curr = curr->next;
        }
        return nullptr;
    }

    void clear() {
        DLLNode* curr = head_guard_->next;
        while (curr != tail_guard_) {
            DLLNode* next = curr->next;
            delete curr;
            curr = next;
        }
        head_guard_->next = tail_guard_;
        tail_guard_->prev = head_guard_;
        size_ = 0;
    }
};
```

---

## 6. Core Operations — Visualised

### The Two Fundamental Primitives

Every DLL operation — insert, delete, move — reduces to one or both of these two primitives. Master these and the rest is bookkeeping.

#### Primitive 1: `link_between(before, node, after)`

```
Goal: insert [NEW] between [BEFORE] and [AFTER]

Before:  [BEFORE] ←──→ [AFTER]

Step 1: NEW->prev = BEFORE
Step 2: NEW->next = AFTER
Step 3: BEFORE->next = NEW
Step 4: AFTER->prev  = NEW

After:   [BEFORE] ←──→ [NEW] ←──→ [AFTER]

Steps 1 & 2 MUST precede steps 3 & 4.
If you do step 3 first, you lose the pointer to AFTER.
```

#### Primitive 2: `unlink(node)`

```
Goal: remove [NODE] from between [PREV] and [NEXT]

Before:  [PREV] ←──→ [NODE] ←──→ [NEXT]

Step 1: PREV->next = NODE->next = NEXT
Step 2: NEXT->prev = NODE->prev = PREV

After:   [PREV] ←──→ [NEXT]
         [NODE]  (dangling — caller must delete)

Order here is flexible — both steps are reads of NODE's fields,
which are not modified until we are done reading them.
```

### Deleting the Tail — The DLL Advantage

```
List: HEAD_G ←──→ [10] ←──→ [20] ←──→ [30] ←──→ TAIL_G
                                         ↑
                                       tail_guard_->prev

SLL: must traverse from head to find [20] (predecessor of [30]) → O(n)

DLL: [30]->prev == [20] directly. No traversal needed.

Step 1: node = tail_guard_->prev              → node points to [30]
Step 2: unlink(node)
          [20]->next = tail_guard_
          tail_guard_->prev = [20]
Step 3: delete node

Result: HEAD_G ←──→ [10] ←──→ [20] ←──→ TAIL_G    O(1) ✓
```

### Move-to-Front (the LRU Cache operation)

```
List: HEAD_G ←──→ [A] ←──→ [B] ←──→ [C] ←──→ TAIL_G
Goal: access [C], move it to the front

Step 1: unlink([C])
  HEAD_G ←──→ [A] ←──→ [B] ←──→ TAIL_G    [C] is detached

Step 2: link_between(HEAD_G, [C], HEAD_G->next=[A])
  HEAD_G ←──→ [C] ←──→ [A] ←──→ [B] ←──→ TAIL_G

Total: 4 pointer updates, O(1), zero allocation, zero traversal ✓
```

---

## 7. Common Patterns & Techniques

### Pattern 1: The LRU Node — Data + Position in One Object

The canonical DLL pattern: store the node pointer in a hash map so you can jump directly to any node in O(1), then use the DLL to maintain order. See Interview Problem 1 for full implementation.

```cpp
struct LRUNode {
    int key, val;
    LRUNode *prev, *next;
    LRUNode(int k, int v) : key(k), val(v), prev(nullptr), next(nullptr) {}
};
unordered_map<int, LRUNode*> cache;   // key → node pointer
// The DLL order = recency of access (MRU at head, LRU at tail)
```

### Pattern 2: Reverse Traversal

```cpp
// Traverse backward from tail — only possible in DLL
DLLNode* curr = tail_guard_->prev;
while (curr != head_guard_) {
    process(curr->val);
    curr = curr->prev;
}
```

### Pattern 3: Splice — Moving a Range of Nodes

Remove a contiguous range from one DLL and insert it into another, or at a different position in the same DLL, all in O(1) — no allocation, no traversal, just pointer surgery.

```cpp
// Splice nodes [first..last] out of their list and insert them after `pos` in another list.
// Precondition: [first..last] is a valid range, pos is in the target list.
// This is how std::list::splice() works internally.
void splice(DLLNode* pos, DLLNode* first, DLLNode* last) {
    // Detach [first..last] from their current chain
    DLLNode* before_first = first->prev;
    DLLNode* after_last   = last->next;
    before_first->next = after_last;
    after_last->prev   = before_first;

    // Insert [first..last] after pos
    DLLNode* after_pos = pos->next;
    pos->next    = first;
    first->prev  = pos;
    last->next   = after_pos;
    after_pos->prev = last;
}
```

### Pattern 4: XOR Doubly Linked List (Memory Optimisation)

Store `prev XOR next` in a single pointer field instead of two separate pointers. Halves the pointer overhead to match a singly linked list, while retaining O(1) bidirectional traversal if you track the previous node during iteration.

```cpp
// XOR list node — stores prev^next in one pointer
struct XORNode {
    int      val;
    XORNode* both;   // prev XOR next
};

XORNode* xor_ptr(XORNode* a, XORNode* b) {
    return reinterpret_cast<XORNode*>(
        reinterpret_cast<uintptr_t>(a) ^ reinterpret_cast<uintptr_t>(b)
    );
}

// Traversal: given current node and previous node, find next
// next = both XOR prev
XORNode* get_next(XORNode* curr, XORNode* prev) {
    return xor_ptr(curr->both, prev);
}
```

**When to use:** Embedded systems where RAM is a hard constraint. Not used in application code — it breaks garbage collectors and ASAN. Mentioning it in an interview signals deep systems knowledge.

---

## 8. Interview Problems

### Problem 1: LRU Cache

**Problem:** Design a data structure that supports two operations on a capacity-bounded cache:
- `get(key)` → return value if key exists, else `-1`. Mark as most recently used.
- `put(key, value)` → insert or update. If at capacity, evict the least recently used entry first.

Both operations must run in **O(1)** time.

**Why this problem belongs here:**

The O(1) eviction requirement means you need O(1) access to the LRU entry (least recently used = the one that has not been touched the longest). You also need O(1) promotion (move any accessed entry to "most recently used" position) and O(1) removal. A doubly linked list where the head is MRU and the tail is LRU, combined with a hash map for O(1) key lookup, satisfies all three requirements simultaneously.

```
Data structure:
  HashMap: key → DLL node pointer      ← O(1) lookup
  DLL: MRU ←──→ ... ←──→ LRU          ← O(1) move-to-front and tail eviction

get(key):
  if key in map: move node to front, return val   O(1)
  else: return -1                                   O(1)

put(key, val):
  if key in map: update val, move to front         O(1)
  else:
    if full: evict tail node, remove from map      O(1)
    create new node, insert at front, add to map   O(1)
```

```cpp
#include <unordered_map>
using namespace std;

class LRUCache {
private:
    struct Node {
        int   key, val;
        Node *prev, *next;
        Node(int k, int v) : key(k), val(v), prev(nullptr), next(nullptr) {}
    };

    int capacity_;
    unordered_map<int, Node*> map_;   // key → node
    Node* head_;   // head sentinel (MRU side)
    Node* tail_;   // tail sentinel (LRU side)

    // ── Two fundamental primitives ─────────────────────────────────────────
    void remove(Node* node) {
        node->prev->next = node->next;
        node->next->prev = node->prev;
    }

    void insert_at_front(Node* node) {
        node->next       = head_->next;
        node->prev       = head_;
        head_->next->prev = node;
        head_->next      = node;
    }

public:
    explicit LRUCache(int capacity) : capacity_(capacity) {
        head_ = new Node(0, 0);
        tail_ = new Node(0, 0);
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

    int get(int key) {
        if (!map_.count(key)) return -1;
        Node* node = map_[key];
        remove(node);               // detach from current position
        insert_at_front(node);      // promote to MRU
        return node->val;
    }

    void put(int key, int val) {
        if (map_.count(key)) {
            // Update existing
            Node* node = map_[key];
            node->val = val;
            remove(node);
            insert_at_front(node);
        } else {
            if ((int)map_.size() == capacity_) {
                // Evict LRU: the real node just before tail sentinel
                Node* lru = tail_->prev;
                remove(lru);
                map_.erase(lru->key);
                delete lru;
            }
            Node* node = new Node(key, val);
            insert_at_front(node);
            map_[key] = node;
        }
    }
};

/*
Dry run: LRUCache(2)
  put(1,1): DLL=[1], map={1→node1}
  put(2,2): DLL=[2,1], map={1,2}
  get(1):   DLL=[1,2], map={1,2}, return 1   ← 1 promoted to MRU
  put(3,3): full → evict LRU=2. DLL=[3,1], map={1,3}
  get(2):   not in map → return -1
  put(4,4): full → evict LRU=1. DLL=[4,3], map={3,4}
  get(1):   return -1
  get(3):   DLL=[3,4], return 3
  get(4):   DLL=[4,3], return 4
*/
```

**Why DLL and not just a sorted structure?**  
A heap or sorted set would give O(log n) promotion. A DLL gives O(1) because `remove` and `insert_at_front` are pure pointer updates — no comparison, no traversal, no restructuring.

**Edge cases:**
- Capacity 1: put evicts on every new key
- put with existing key: must update value AND promote — both steps required
- get on empty cache: return -1
- put same key twice: second put updates, does not create a duplicate

---

### Problem 2: Design Browser History

**Problem:** Implement a browser history system:
- `BrowserHistory(homepage)` — start at homepage
- `visit(url)` — navigate to url; clear all forward history
- `back(steps)` — move back up to `steps` pages; return current url
- `forward(steps)` — move forward up to `steps` pages; return current url

```cpp
// A doubly linked list where the current node is the "cursor"
// back()    → follow prev pointers
// forward() → follow next pointers
// visit()   → insert after current, delete everything after that node

class BrowserHistory {
private:
    struct PageNode {
        string    url;
        PageNode* prev;
        PageNode* next;
        PageNode(const string& u) : url(u), prev(nullptr), next(nullptr) {}
    };

    PageNode* current_;

    // Delete all nodes from `start` to end of chain
    void clear_forward(PageNode* start) {
        while (start) {
            PageNode* next = start->next;
            delete start;
            start = next;
        }
    }

public:
    explicit BrowserHistory(const string& homepage) {
        current_ = new PageNode(homepage);
    }

    ~BrowserHistory() {
        // Rewind to beginning then delete forward
        while (current_->prev) current_ = current_->prev;
        clear_forward(current_);
    }

    void visit(const string& url) {
        PageNode* page = new PageNode(url);
        // Sever forward history from current_ before linking the new page
        clear_forward(current_->next);
        current_->next = page;
        page->prev     = current_;
        current_       = page;
    }

    string back(int steps) {
        while (steps > 0 && current_->prev) {
            current_ = current_->prev;
            steps--;
        }
        return current_->url;
    }

    string forward(int steps) {
        while (steps > 0 && current_->next) {
            current_ = current_->next;
            steps--;
        }
        return current_->url;
    }
};

/*
Trace:
  BrowserHistory("leetcode.com")  current=leetcode
  visit("google.com")             leetcode ←→ google,  current=google
  visit("facebook.com")           leetcode ←→ google ←→ facebook, current=facebook
  back(1)                         current=google,   return "google.com"
  back(1)                         current=leetcode, return "leetcode.com"
  forward(1)                      current=google,   return "google.com"
  visit("youtube.com")            leetcode ←→ google ←→ youtube (facebook DELETED)
                                  current=youtube
  forward(2)                      current=youtube (no forward history), return "youtube.com"
  back(2)                         current=leetcode, return "leetcode.com"
  back(7)                         can't go back further, return "leetcode.com"
*/
```

**Edge cases:**
- `back(steps)` with steps > history length: return the earliest page, do not crash
- `forward(steps)` after `visit()`: forward history was cleared, return current page
- `back(0)` or `forward(0)`: return current page unchanged
- Visiting a page you are already on: treated as a new visit (new node, clears forward)

---

### Problem 3: Flatten a Multilevel Doubly Linked List

**Problem:** A doubly linked list may have nodes with a `child` pointer to another doubly linked list. Flatten it into a single-level DLL, inserting each child list immediately after its parent node.

**Example:**
```
1 ←→ 2 ←→ 3 ←→ 4 ←→ 5 ←→ 6
          |
          7 ←→ 8 ←→ 9 ←→ 10
                    |
                    11 ←→ 12

Flattened: 1 ←→ 2 ←→ 3 ←→ 7 ←→ 8 ←→ 9 ←→ 11 ←→ 12 ←→ 10 ←→ 4 ←→ 5 ←→ 6
```

**Thought process:**

> "At each node with a child: find the tail of the child list, then splice the child list between the current node and current->next. After splicing, continue traversing from the current node's next (which is now the first child node). This naturally handles multiple levels because after we splice a child list, we will encounter its children during normal traversal."

```cpp
struct Node {
    int   val;
    Node* prev;
    Node* next;
    Node* child;
    Node(int v) : val(v), prev(nullptr), next(nullptr), child(nullptr) {}
};

// Time: O(n) where n = total nodes across all levels
// Space: O(1) — pure pointer manipulation, no extra allocation
Node* flatten(Node* head) {
    Node* curr = head;

    while (curr) {
        if (!curr->child) {
            curr = curr->next;   // no child: just advance
            continue;
        }

        // curr has a child — splice the child list in
        Node* child      = curr->child;
        Node* after_curr = curr->next;

        // Find the tail of the child list
        Node* child_tail = child;
        while (child_tail->next) child_tail = child_tail->next;

        // Connect curr → child
        curr->next  = child;
        child->prev = curr;
        curr->child = nullptr;   // clear child pointer

        // Connect child_tail → after_curr
        child_tail->next = after_curr;
        if (after_curr) after_curr->prev = child_tail;

        // Continue traversal — curr->next is now the first child node
        // Its children (if any) will be handled when we reach them
        curr = curr->next;
    }
    return head;
}

/*
Trace on the example above:

curr=1: no child → advance
curr=2: no child → advance
curr=3: has child [7→8→9→10]
  child=7, after_curr=4
  child_tail = 10
  3→7, 7←3, 3.child=null
  10→4, 4←10
  List: 1←→2←→3←→7←→8←→9←→10←→4←→5←→6
  curr = 7 (curr->next)

curr=7: no child → advance
curr=8: no child → advance
curr=9: has child [11→12]
  child=11, after_curr=10
  child_tail=12
  9→11, 11←9, 9.child=null
  12→10, 10←12
  List: 1←→2←→3←→7←→8←→9←→11←→12←→10←→4←→5←→6
  curr = 11

curr=11: no child → advance
curr=12: no child → advance
... continue to end

Final: 1←→2←→3←→7←→8←→9←→11←→12←→10←→4←→5←→6  ✓
*/
```

**Edge cases:**
- Node whose child list itself has children — handled automatically by the traversal
- Last node has a child: `after_curr = nullptr`, so `child_tail->next = nullptr` — correct
- All nodes have children (deeply nested) — O(n) total because every node is visited exactly once
- Child list longer than the remaining main list — handled correctly

---

## 9. Real-World Uses

| Domain | Use Case | Why DLL Specifically |
|---|---|---|
| OS kernel | `linux/list.h` — the intrusive circular DLL used for *everything* | O(1) arbitrary delete; splice for work-stealing scheduler |
| OS kernel | Page replacement (Clock / LRU variant) | O(1) promote recently-used page to head |
| Browser | Tab history, back/forward navigation | Bidirectional traversal, O(1) truncate forward history |
| Text editors | Cursor movement, undo/redo | O(1) insert/delete at cursor; prev for undo chain |
| `std::list<T>` | C++ doubly linked list container | Iterator stability: iterators survive insert/erase |
| Database | Buffer pool manager page list (Postgres) | O(1) evict LRU page from buffer pool |
| JVM / .NET GC | Free list for memory regions | O(1) coalescing of adjacent free blocks |
| Music players | Playlist with prev/next track | O(1) prev and next, wraps around |
| IDE / Vim | Jump list (Ctrl+O / Ctrl+I history) | Bidirectional navigation through code locations |
| Networking | TCP receive buffer reordering | Insert out-of-order segments; in-order delivery from front |

**The Linux Kernel's `list_head` — a masterclass in intrusive DLLs:**

```c
// From linux/list.h — the most widely used DLL in existence
struct list_head {
    struct list_head *next, *prev;
};

// Embed list_head inside your struct — no separate node allocation
struct task_struct {         // process descriptor
    // ... hundreds of fields ...
    struct list_head tasks;  // embedded DLL node for the task list
    struct list_head children;
    struct list_head sibling;
    // A process is simultaneously in MULTIPLE linked lists via different list_head fields
};

// Navigate: given list_head*, recover the containing struct via container_of()
#define list_entry(ptr, type, member) \
    container_of(ptr, type, member)
```

**Why intrusive?** The node storage is inside the object being listed, not separate. There is zero allocation overhead for list membership. A process can be in ten different lists simultaneously (runqueue, wait queue, zombie list, etc.) by embedding ten `list_head` fields — no extra allocations, no indirection. This is the production pattern for high-performance systems code.

---

## 10. Edge Cases & Pitfalls

### Pitfall 1: The Four-Pointer Update Order for Insertion

```cpp
// Inserting NEW between BEFORE and AFTER

// WRONG: step 3 before step 1 & 2
before->next = new_node;   // now before->next == new_node, but new_node->next is garbage
new_node->next = after;    // ok, but before->next already lost the original after

// CORRECT: set NEW's pointers before rewiring existing nodes
new_node->prev = before;   // 1. point NEW backward
new_node->next = after;    // 2. point NEW forward
before->next   = new_node; // 3. rewire BEFORE
after->prev    = new_node; // 4. rewire AFTER
// Steps 1 & 2 can be in any order relative to each other.
// Steps 3 & 4 can be in any order relative to each other.
// But 3 & 4 must come after 1 & 2.
```

### Pitfall 2: Forgetting to Clear the Child Pointer (Flatten Problem)

```cpp
// When splicing a child list into the main list:
curr->next = child;
child->prev = curr;
// MISSING: curr->child = nullptr;
// If you don't clear child pointer, algorithms that check for children
// will process the same subtree again → infinite loop or corruption.
```

### Pitfall 3: Both Sentinels Must Be Deleted in the Destructor

```cpp
// Correct destructor — traverse real nodes AND delete both sentinels
~DoublyLinkedList() {
    DLLNode* curr = head_guard_->next;
    while (curr != tail_guard_) {
        DLLNode* next = curr->next;
        delete curr;
        curr = next;
    }
    delete head_guard_;   // ← must delete sentinel
    delete tail_guard_;   // ← must delete sentinel
}
// Missing either sentinel delete = 2-node memory leak per list instance
```

### Pitfall 4: Decrementing Size in `move_to_front` / `move_to_back`

```cpp
// move_to_front calls unlink() then link_between()
// unlink() decrements size_. link_between() increments size_.
// Net effect: size_ unchanged. ✓ — but only if you implement it this way.
// If you inline the pointer updates without touching size_, you get no bug.
// The danger is mixing unlink() with raw pointer updates inconsistently.
```

### Pitfall 5: Treating Sentinels as Real Nodes

```cpp
// If you expose the raw DLL nodes to callers, they might accidentally
// remove or read a sentinel node.
void remove_node(DLLNode* node) {
    // ALWAYS guard against sentinel removal
    if (node == head_guard_ || node == tail_guard_) return;
    unlink(node);
    delete node;
}

// Similarly: never call front() or back() on an empty list
// head_guard_->next == tail_guard_ when empty — reading its val is meaningless
```

### Pitfall 6: Using a Raw Pointer After `remove_node`

```cpp
DLLNode* node = list.find(42);
list.remove_node(node);   // node is deleted inside this call
cout << node->val;        // USE-AFTER-FREE — undefined behaviour
// Solution: treat the pointer as invalid immediately after removal.
```

### Pitfall 7: Comparing Against `nullptr` Instead of Sentinel

```cpp
// With sentinel-based DLL, the list never has real nullptr in the chain.
// Traversal must stop at the sentinel, NOT at nullptr.

// WRONG (null-terminated logic applied to sentinel-based DLL):
DLLNode* curr = head_guard_->next;
while (curr) {           // curr is NEVER null — tail_guard_ is a real node!
    process(curr->val);  // will process the sentinel and loop on null->next
    curr = curr->next;
}

// CORRECT: stop at the sentinel
while (curr != tail_guard_) {
    process(curr->val);
    curr = curr->next;
}
```

---

## 11. Comparison

| Feature | Singly LL | Doubly LL | `std::vector` | `std::list` | `std::deque` |
|---|---|---|---|---|---|
| Node overhead | 1 ptr (8B) | 2 ptr (16B) | 0 | 2 ptr (16B) | ~0 per element |
| Random access | O(n) | O(n) | **O(1)** | O(n) | **O(1)** |
| push_front | **O(1)** | **O(1)** | O(n) | **O(1)** | **O(1)** |
| push_back | **O(1)** w/ tail | **O(1)** | **O(1)** amort | **O(1)** | **O(1)** |
| pop_front | **O(1)** | **O(1)** | O(n) | **O(1)** | **O(1)** |
| pop_back | O(n) | **O(1)** ← DLL wins | **O(1)** | **O(1)** | **O(1)** |
| Delete given pointer | O(n) find prev | **O(1)** ← DLL wins | N/A | **O(1)** | N/A |
| Back traversal | Not possible | **O(n)** | O(n) | **O(n)** | O(n) |
| Iterator stability on insert | N/A | Stable except erased | **Invalidated** | Stable | Partly |
| Cache performance | Poor | Poor | **Excellent** | Poor | Good |
| Sentinel pattern | Optional | Recommended | N/A | Yes (impl) | N/A |
| LRU Cache use | With workarounds | **Natural fit** | Not suitable | Can work | Not suitable |

**Decision guide:**
- Default: `std::vector`
- Need O(1) push/pop at both ends: `std::deque`
- Need stable iterators through insert/erase, or O(1) splice: `std::list`
- Need O(1) delete with external node pointer (LRU, OS scheduler): **DLL directly**
- Implementing LRU Cache in an interview: **DLL + HashMap**
- Memory-constrained, forward-only: `SLL`

---

## 12. Self-Test Questions

1. **Without running code: what is the exact state of all pointers after calling `push_back(50)` on a DLL containing `[10 ←→ 20 ←→ 30]` with sentinels?** Draw the before and after.

2. **Why is `pop_back` O(1) in a DLL but O(n) in a SLL? Which pointer makes the difference?**

3. **What are sentinel nodes? What bug do they prevent? Draw an empty DLL with and without sentinels.**

4. **In the LRU Cache implementation, why is the DLL ordered MRU-at-head rather than LRU-at-head? Could it work the other way?**

5. **`move_to_front` calls `unlink` then `link_between`. Why does `size_` end up correct? Would it break if you inlined the pointer updates instead?**

6. **In `flatten`, why does `curr = curr->next` (not `curr = child`) after splicing the child list in? What would go wrong with `curr = child`?**

7. **Why does the sentinel-based traversal stop at `curr != tail_guard_` rather than `curr != nullptr`? What happens if you use the wrong condition?**

8. **What is an intrusive doubly linked list? How does the Linux kernel's `list_head` allow a single `task_struct` to be a member of multiple lists simultaneously?**

9. **Implement `reverse()` for a sentinel-based DLL. What happens to the sentinels?**

10. **You are designing a text editor. The cursor is at some position. Design the data structure to support O(1) insert character, O(1) delete character, O(1) move cursor left/right, and O(n) undo (reverse last n operations). Which parts use DLL?**

---

## Quick Reference Card

```
Doubly Linked List — the SLL with a second pointer that buys O(1) deletion anywhere.

Node:  [←prev | data | next→]
Sentinels: HEAD_GUARD ←→ [real nodes] ←→ TAIL_GUARD

Key complexities vs SLL:
  pop_back:               O(n) SLL → O(1) DLL   ← main win
  delete given pointer:   O(n) SLL → O(1) DLL   ← defining advantage
  insert before node:     O(n) SLL → O(1) DLL
  backward traversal:     impossible → O(n) DLL

Cost: 8 extra bytes per node (the prev pointer)

Two fundamental primitives — everything else is built from these:
  link_between(before, node, after):    4 pointer updates, O(1)
  unlink(node):                         2 pointer updates, O(1)

The interview application:
  LRU Cache = DLL (for O(1) promotion and eviction) + HashMap (for O(1) lookup)

Pointer update rules:
  Insertion: set NEW->prev and NEW->next BEFORE rewiring neighbours
  Deletion:  prev->next and next->prev are independent — any order fine
  Always:    clear child/extra pointers after splice operations
  Always:    do not dereference a node after remove_node() — it is deleted
```

---

*Previous: [03 — Singly Linked List](./03_singly_linked_list.md)*
*Next: [05 — Stack](./05_stack.md)*
*See also: [LRU Cache design](./specialized/lru_cache.md) | [LFU Cache](./specialized/lfu_cache.md) | [Linux list.h](https://github.com/torvalds/linux/blob/master/include/linux/list.h)*

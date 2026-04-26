# Queue

> **Curriculum position:** Linear Structures → #9
> **Interview weight:** ★★★★★ Critical — BFS, level-order traversal, and scheduling all run on queues
> **Difficulty:** Beginner concept, deep in BFS applications
> **Prerequisite:** [07 — Stack](./07_stack.md)

---

## Table of Contents

1. [Intuition First — The FIFO Principle](#1-intuition-first)
2. [Abstract Data Type Contract](#2-abstract-data-type-contract)
3. [Internal Working — Four Implementations](#3-internal-working)
4. [The Circular Array Implementation — Why It Exists](#4-the-circular-array-implementation)
5. [Time & Space Complexity](#5-time--space-complexity)
6. [Complete C++ Implementation](#6-complete-c-implementation)
7. [Core Operations — Visualised](#7-core-operations--visualised)
8. [Common Patterns & Techniques](#8-common-patterns--techniques)
9. [Interview Problems](#9-interview-problems)
10. [Real-World Uses](#10-real-world-uses)
11. [Edge Cases & Pitfalls](#11-edge-cases--pitfalls)
12. [Queue Variants at a Glance](#12-queue-variants-at-a-glance)
13. [std::queue — The Standard Library Version](#13-stdqueue)
14. [Comparison: Queue vs Stack vs Deque vs Priority Queue](#14-comparison)
15. [Self-Test Questions](#15-self-test-questions)

---

## 1. Intuition First

A queue is a line at a ticket counter. The first person to join the line is the first person served. New arrivals join the back. Service happens at the front. Nobody cuts in the middle. Nobody serves the person at the back first.

This is **FIFO: First In, First Out**. The element that has waited the longest is always the next to be served.

Where the stack tracked the most *recent* unresolved thing (LIFO — useful for reversal, backtracking, depth), the queue tracks the most *ancient* unresolved thing (FIFO — useful for fairness, ordering, breadth).

That single distinction — oldest first vs newest first — separates two entirely different algorithmic universes:

```
Stack (LIFO) → Depth-First Search  → explores as deep as possible before backtracking
Queue (FIFO) → Breadth-First Search → explores all neighbours before going deeper
```

BFS is the queue's defining application. Every shortest-path problem on an unweighted graph, every level-order tree traversal, every "minimum steps to reach" problem runs on a queue. The queue is what makes BFS explore level by level — the nodes discovered earliest (closest to the source) are processed first, guaranteeing that the first time you reach any node it is via the shortest possible path.

Beyond BFS, queues model any system where order of arrival must be preserved: OS process scheduling, network packet buffers, printer job queues, message brokers. Anywhere fairness and ordering matter, a queue is the right structure.

---

## 2. Abstract Data Type Contract

Like the stack, a queue is defined by its behavioural contract, not its internal implementation.

```
enqueue(x)  / push(x)   — add element x to the back        must be O(1)
dequeue()   / pop()     — remove element from the front     must be O(1)
front()     / peek()    — return front element, no remove   must be O(1)
back()                  — return back element, no remove    must be O(1)
empty()                 — true if no elements               must be O(1)
size()                  — number of elements                must be O(1)
```

Any implementation satisfying O(1) enqueue, O(1) dequeue, and O(1) front is a valid queue.

---

## 3. Internal Working — Four Implementations

### Implementation 1: Naive Array (Broken — Do Not Use)

```
enqueue at back:  arr[tail++] = x        O(1) ✓
dequeue at front: return arr[front++]    O(1) ✓ (just advance front index)

Problem: both front and tail march rightward forever.
After n operations: front = n, tail = n.
The array has n unused slots at the left and appears "full"
even though it has capacity for n more elements.

front                    tail
  ↓                        ↓
[  |  |  |  |  |  |  |  |  |  ]    ← slots 0..front-1 wasted forever
  dead space (can't reuse)
```

This is why you cannot implement a queue efficiently with a plain array and two advancing indices. The naive approach wastes memory and hits a false-full condition.

### Implementation 2: Circular Array (The Correct Array-Based Approach)

Wrap the indices around using modulo arithmetic. When `tail` reaches the end of the array, it wraps back to index 0 and reuses the freed space from previous dequeues.

```
Capacity = 8 slots, front=0, tail=0, size=0

After enqueue(A,B,C,D):
  [A][B][C][D][ ][ ][ ][ ]
   ↑               ↑
 front=0          tail=4

After dequeue(), dequeue():
  [_][_][C][D][ ][ ][ ][ ]
           ↑               ↑
         front=2           tail=4

After enqueue(E,F,G,H,I) — tail wraps around:
  [I][_][C][D][E][F][G][H]
   ↑   ↑
 tail=1 front=2

Now front=2, tail=1. Full! (size = capacity - 1 in the gap-of-one convention)
```

This is the canonical fixed-capacity queue implementation. `std::queue` uses `std::deque` internally (which is more sophisticated) but the circular array is what you implement from scratch in interviews.

### Implementation 3: Linked-List-Backed (Unbounded)

Maintain `head` (front — dequeue here) and `tail` (back — enqueue here).
- Enqueue: allocate new node, append to tail — O(1)
- Dequeue: remove head node — O(1)

```
head                                 tail
 ↓                                    ↓
[A] → [B] → [C] → [D] → [E] → null

enqueue(F): new node [F]; tail->next = [F]; tail = [F]
dequeue():  val = head->val; head = head->next; delete old head; return val
```

No false-full condition. No wasted memory. Unbounded. The cost is one heap allocation per enqueue — slower than circular array for small elements.

### Implementation 4: Two-Stack Queue (Interview Classic)

Build a queue from two stacks. This is a famous interview question (and appears again as a problem below).

```
inbox stack (for enqueue)    outbox stack (for dequeue)
    [C]                          (empty)
    [B]
    [A]   ← top

enqueue(X): push X onto inbox. Always O(1).

dequeue(): if outbox empty, pour entire inbox into outbox (reverses order).
           pop from outbox.
           Amortized O(1) — each element moves at most once.

After pour:
inbox (empty)    outbox:
                     [A]   ← top (A was oldest, now first to come out)
                     [B]
                     [C]
```

This demonstrates that a queue can be built from stacks, and that amortized O(1) dequeue is achievable with this trick.

---

## 4. The Circular Array Implementation — Why It Exists

This deserves its own section because most candidates implement it incorrectly.

### The Full vs Empty Ambiguity

With a circular array of capacity `n`, using indices `front` and `tail`:
- **Empty condition:** `front == tail`
- **Full condition:** also `front == tail` after wrapping!

These conditions are indistinguishable with just two indices. Three solutions exist:

**Solution A — Gap of one (capacity wastes one slot):**
- Full: `(tail + 1) % capacity == front`
- Empty: `tail == front`
- Usable capacity: `capacity - 1` slots
- Simple, most common

**Solution B — Explicit size counter:**
- Full: `size == capacity`
- Empty: `size == 0`
- Full capacity used, one extra integer maintained
- Most robust — preferred in production

**Solution C — Use a boolean `is_full` flag:**
- Toggle on enqueue when `front == tail` after writing
- Less elegant, not recommended

```cpp
// Solution B — size counter (recommended):
struct CircularQueue {
    vector<int> data;
    int front, tail, sz, cap;

    bool empty() { return sz == 0; }
    bool full()  { return sz == cap; }

    void enqueue(int x) {
        if (full()) throw overflow_error("queue full");
        data[tail] = x;
        tail = (tail + 1) % cap;
        sz++;
    }

    int dequeue() {
        if (empty()) throw underflow_error("queue empty");
        int val = data[front];
        front = (front + 1) % cap;
        sz--;
        return val;
    }
};
```

### Why Modulo Arithmetic Works

```
capacity = 8

tail = 7, enqueue(X):
  data[7] = X
  tail = (7 + 1) % 8 = 0    ← wraps to front of array

front = 7, dequeue():
  val = data[7]
  front = (7 + 1) % 8 = 0   ← wraps identically

The modulo ensures indices never exceed capacity-1 and
automatically reuse freed slots at the beginning of the array.
```

---

## 5. Time & Space Complexity

### Operation Complexities

| Operation | Circular Array | Linked List | Two-Stack |
|---|---|---|---|
| `enqueue(x)` | **O(1)** | **O(1)** | **O(1)** |
| `dequeue()` | **O(1)** | **O(1)** | **O(1) amortized** |
| `front()` | **O(1)** | **O(1)** | **O(1) amortized** |
| `back()` | **O(1)** | **O(1)** | **O(1)** |
| `empty()` | **O(1)** | **O(1)** | **O(1)** |
| `size()` | **O(1)** | **O(1)** | **O(1)** |
| Search | **O(n)** | **O(n)** | **O(n)** |
| Max capacity | Fixed | Unbounded | Unbounded |
| Cache behaviour | **Excellent** | Poor | Moderate |

### Space Complexity

| Implementation | Space | Overhead |
|---|---|---|
| Circular array (fixed) | O(capacity) | 0 per element |
| Linked list | O(n) | 8 bytes per element (next pointer) |
| Two-stack (vector) | O(n) | ~2× vector capacity (2 stacks) |
| `std::queue` (deque) | O(n) | Chunked allocation overhead |

---

## 6. Complete C++ Implementation

### Implementation 1: Circular Array Queue (Fixed Capacity)

```cpp
#include <iostream>
#include <vector>
#include <stdexcept>
using namespace std;

class CircularQueue {
private:
    vector<int> data_;
    int front_;     // index of the front element
    int tail_;      // index where next element will be written
    int size_;      // current number of elements
    int capacity_;  // maximum elements

public:
    explicit CircularQueue(int cap)
        : data_(cap), front_(0), tail_(0), size_(0), capacity_(cap) {}

    // ── Core Operations ───────────────────────────────────────────────────────

    // O(1) — add to back
    void enqueue(int val) {
        if (full()) throw overflow_error("queue is full");
        data_[tail_] = val;
        tail_ = (tail_ + 1) % capacity_;
        size_++;
    }

    // O(1) — remove from front
    int dequeue() {
        if (empty()) throw underflow_error("queue is empty");
        int val = data_[front_];
        front_ = (front_ + 1) % capacity_;
        size_--;
        return val;
    }

    // O(1) — peek at front
    int front() const {
        if (empty()) throw underflow_error("queue is empty");
        return data_[front_];
    }

    // O(1) — peek at back
    int back() const {
        if (empty()) throw underflow_error("queue is empty");
        // tail_ points to next write slot; element before it is the back
        return data_[(tail_ - 1 + capacity_) % capacity_];
    }

    bool empty() const { return size_ == 0; }
    bool full()  const { return size_ == capacity_; }
    int  size()  const { return size_; }

    void print() const {
        if (empty()) { cout << "(empty)\n"; return; }
        cout << "front → ";
        for (int i = 0; i < size_; i++) {
            cout << data_[(front_ + i) % capacity_];
            if (i < size_ - 1) cout << " → ";
        }
        cout << " ← back\n";
    }
};
```

### Implementation 2: Linked-List Queue (Unbounded)

```cpp
class LinkedQueue {
private:
    struct Node {
        int   val;
        Node* next;
        Node(int v) : val(v), next(nullptr) {}
    };

    Node* front_;   // dequeue from here
    Node* back_;    // enqueue here
    int   size_;

public:
    LinkedQueue() : front_(nullptr), back_(nullptr), size_(0) {}

    ~LinkedQueue() {
        while (front_) {
            Node* tmp = front_->next;
            delete front_;
            front_ = tmp;
        }
    }

    LinkedQueue(const LinkedQueue&)            = delete;
    LinkedQueue& operator=(const LinkedQueue&) = delete;

    // O(1) — add to back
    void enqueue(int val) {
        Node* node = new Node(val);
        if (!back_) {
            front_ = back_ = node;
        } else {
            back_->next = node;
            back_ = node;
        }
        size_++;
    }

    // O(1) — remove from front
    int dequeue() {
        if (!front_) throw underflow_error("queue is empty");
        int val = front_->val;
        Node* tmp = front_;
        front_ = front_->next;
        if (!front_) back_ = nullptr;   // queue became empty
        delete tmp;
        size_--;
        return val;
    }

    int  front_val() const { if (!front_) throw underflow_error("empty"); return front_->val; }
    int  back_val()  const { if (!back_)  throw underflow_error("empty"); return back_->val;  }
    bool empty()     const { return front_ == nullptr; }
    int  size()      const { return size_; }
};
```

### Implementation 3: Dynamic Queue (std::vector-backed, auto-resizing)

```cpp
// For interviews where capacity is unknown: use deque<T> or vector with
// front tracking. This is effectively what std::queue does.
#include <deque>

class DynamicQueue {
private:
    deque<int> data_;

public:
    void enqueue(int val) { data_.push_back(val); }
    int  dequeue()        {
        if (data_.empty()) throw underflow_error("queue is empty");
        int val = data_.front();
        data_.pop_front();
        return val;
    }
    int  front() const { return data_.front(); }
    int  back()  const { return data_.back();  }
    bool empty() const { return data_.empty(); }
    int  size()  const { return (int)data_.size(); }
};
```

### Implementation 4: Two-Stack Queue (Classic Interview Implementation)

```cpp
class TwoStackQueue {
private:
    stack<int> inbox_;    // receives all enqueues
    stack<int> outbox_;   // provides all dequeues

    // Pour inbox into outbox, reversing order
    // This is the amortized-O(1) trick: each element moves at most once
    void pour() {
        if (outbox_.empty()) {
            while (!inbox_.empty()) {
                outbox_.push(inbox_.top());
                inbox_.pop();
            }
        }
    }

public:
    // O(1) always
    void enqueue(int val) {
        inbox_.push(val);
    }

    // O(1) amortized — pour happens at most once per element's lifetime
    int dequeue() {
        if (inbox_.empty() && outbox_.empty())
            throw underflow_error("queue is empty");
        pour();
        int val = outbox_.top();
        outbox_.pop();
        return val;
    }

    // O(1) amortized
    int front() {
        if (inbox_.empty() && outbox_.empty())
            throw underflow_error("queue is empty");
        pour();
        return outbox_.top();
    }

    bool empty() const { return inbox_.empty() && outbox_.empty(); }
    int  size()  const { return (int)(inbox_.size() + outbox_.size()); }
};
```

---

## 7. Core Operations — Visualised

### BFS Level-Order Traversal — The Canonical Queue Algorithm

```
Tree:
          1
        /   \
       2     3
      / \   / \
     4   5 6   7

Queue state during level-order BFS:

Init:     enqueue(1)          queue: [1]
Level 0:  dequeue → 1         queue: []
          enqueue children:   queue: [2, 3]         output: 1
Level 1:  dequeue → 2         queue: [3]
          enqueue children:   queue: [3, 4, 5]      output: 1, 2
          dequeue → 3         queue: [4, 5]
          enqueue children:   queue: [4, 5, 6, 7]   output: 1, 2, 3
Level 2:  dequeue → 4         queue: [5, 6, 7]      output: 1, 2, 3, 4
          no children
          dequeue → 5         queue: [6, 7]          output: 1, 2, 3, 4, 5
          dequeue → 6         queue: [7]             output: 1, 2, 3, 4, 5, 6
          dequeue → 7         queue: []              output: 1, 2, 3, 4, 5, 6, 7
Done.

Key observation: nodes are dequeued in EXACTLY the order they were enqueued.
The queue preserves discovery order → guarantees level-by-level processing.
```

### Circular Array Wrap-Around

```
capacity=6, size counter tracks actual elements

Initial: front=0, tail=0, size=0
  [ ][ ][ ][ ][ ][ ]
   ↑
front=tail=0

enqueue(A,B,C,D): front=0, tail=4, size=4
  [A][B][C][D][ ][ ]
   ↑           ↑
 front=0      tail=4

dequeue(),dequeue(): front=2, tail=4, size=2
  [ ][ ][C][D][ ][ ]
           ↑       ↑
         front=2  tail=4

enqueue(E,F,G): tail wraps — (4+1)%6=5, (5+1)%6=0, (0+1)%6=1
  [G][ ][C][D][E][F]
   ↑   ↑
  tail=1 front=2     size=5

Indices crossed: tail < front. Still correct via modulo. ✓
Iterating: data[(front+i)%cap] for i in 0..size-1
  i=0: data[2]=C, i=1: data[3]=D, i=2: data[4]=E,
  i=3: data[5]=F, i=4: data[0]=G
```

---

## 8. Common Patterns & Techniques

### Pattern 1: BFS Template — The Most Important Queue Pattern

```cpp
// Generic BFS template — memorise this structure
// Works for: graphs, trees, grids, state-space search
void bfs(Node* start) {
    queue<Node*> q;
    unordered_set<Node*> visited;

    q.push(start);
    visited.insert(start);

    while (!q.empty()) {
        Node* curr = q.front(); q.pop();
        process(curr);

        for (Node* neighbour : curr->neighbours) {
            if (!visited.count(neighbour)) {
                visited.insert(neighbour);
                q.push(neighbour);
            }
        }
    }
}

// BFS with level tracking — essential for "minimum steps" and level-order problems
void bfsLevels(Node* start) {
    queue<Node*> q;
    q.push(start);
    int level = 0;

    while (!q.empty()) {
        int levelSize = q.size();   // snapshot: how many nodes are in this level

        for (int i = 0; i < levelSize; i++) {
            Node* curr = q.front(); q.pop();
            process(curr, level);   // process all nodes at this level together

            for (Node* neighbour : curr->neighbours) {
                q.push(neighbour);
            }
        }
        level++;
    }
}
```

**The level-size snapshot trick** is critical: capture `q.size()` BEFORE the inner loop. During the inner loop, you enqueue next-level nodes. By only iterating `levelSize` times (the count from the start of the level), you process exactly the current level's nodes before incrementing `level`.

### Pattern 2: Multi-Source BFS

When BFS starts from multiple sources simultaneously — all sources enqueued at the start as "level 0."

```cpp
// Classic: Rotting Oranges — all rotten oranges spread simultaneously
// All rotten oranges are sources enqueued at level 0
void multiSourceBFS(vector<vector<int>>& grid) {
    queue<pair<int,int>> q;
    int rows = grid.size(), cols = grid[0].size();

    // Enqueue ALL sources before starting BFS
    for (int r = 0; r < rows; r++)
        for (int c = 0; c < cols; c++)
            if (grid[r][c] == 2)          // rotten orange = source
                q.push({r, c});

    int minutes = 0;
    int dirs[4][2] = {{0,1},{0,-1},{1,0},{-1,0}};

    while (!q.empty()) {
        int sz = q.size();
        bool spread = false;

        for (int i = 0; i < sz; i++) {
            auto [r, c] = q.front(); q.pop();
            for (auto& d : dirs) {
                int nr = r + d[0], nc = c + d[1];
                if (nr >= 0 && nr < rows && nc >= 0 && nc < cols
                    && grid[nr][nc] == 1) {
                    grid[nr][nc] = 2;
                    q.push({nr, nc});
                    spread = true;
                }
            }
        }
        if (spread) minutes++;
    }
}
```

### Pattern 3: 0-1 BFS (Deque-based BFS for 0/1 Edge Weights)

When edges have weight 0 or 1, use a deque instead of a priority queue. Weight-0 edges push to the front (free); weight-1 edges push to the back (costs 1 step).

```cpp
// Shortest path where edges cost 0 or 1
// Runs in O(V+E) vs Dijkstra's O((V+E) log V)
vector<int> zeroOneBFS(int n, vector<vector<pair<int,int>>>& adj, int src) {
    vector<int> dist(n, INT_MAX);
    deque<int> dq;

    dist[src] = 0;
    dq.push_front(src);

    while (!dq.empty()) {
        int u = dq.front(); dq.pop_front();

        for (auto [v, w] : adj[u]) {
            if (dist[u] + w < dist[v]) {
                dist[v] = dist[u] + w;
                if (w == 0) dq.push_front(v);    // free edge: insert at front
                else        dq.push_back(v);      // cost-1 edge: insert at back
            }
        }
    }
    return dist;
}
```

### Pattern 4: State-Space BFS (Implicit Graph)

BFS on an implicit graph where "nodes" are states and "edges" are valid transitions. The queue holds states, not tree/graph nodes.

```cpp
// Minimum moves to solve a puzzle (e.g., sliding tiles, word ladder)
int minMoves(string start, string target, unordered_set<string>& wordList) {
    if (!wordList.count(target)) return -1;

    queue<string> q;
    unordered_set<string> visited;

    q.push(start);
    visited.insert(start);
    int steps = 0;

    while (!q.empty()) {
        int sz = q.size();
        for (int i = 0; i < sz; i++) {
            string curr = q.front(); q.pop();
            if (curr == target) return steps;

            // Generate all valid one-step transitions
            for (int j = 0; j < (int)curr.size(); j++) {
                string next = curr;
                for (char c = 'a'; c <= 'z'; c++) {
                    next[j] = c;
                    if (wordList.count(next) && !visited.count(next)) {
                        visited.insert(next);
                        q.push(next);
                    }
                    next[j] = curr[j];
                }
            }
        }
        steps++;
    }
    return -1;
}
```

---

## 9. Interview Problems

### Problem 1: Binary Tree Level Order Traversal

**Problem:** Given the root of a binary tree, return the level-order traversal of its nodes' values — a list of lists, where each inner list contains the values at that depth level.

**Example:**
```
Tree:     3
         / \
        9  20
          /  \
         15   7

Output: [[3], [9, 20], [15, 7]]
```

**Thought process:**

> "I need to process nodes level by level, left to right within each level. BFS naturally processes nodes in the order they were discovered. The challenge is knowing where one level ends and the next begins. Solution: snapshot `q.size()` at the start of each level — it tells me exactly how many nodes belong to this level."

```cpp
// Time: O(n) | Space: O(n) — at most n/2 nodes in the queue at once (bottom level)
vector<vector<int>> levelOrder(TreeNode* root) {
    vector<vector<int>> result;
    if (!root) return result;

    queue<TreeNode*> q;
    q.push(root);

    while (!q.empty()) {
        int levelSize = (int)q.size();   // ← snapshot: nodes in current level
        vector<int> currentLevel;
        currentLevel.reserve(levelSize);

        for (int i = 0; i < levelSize; i++) {
            TreeNode* node = q.front(); q.pop();
            currentLevel.push_back(node->val);

            if (node->left)  q.push(node->left);
            if (node->right) q.push(node->right);
        }
        result.push_back(currentLevel);
    }
    return result;
}

/*
Trace on example tree:

Init: q=[3], result=[]

Level 0: levelSize=1
  dequeue 3 → currentLevel=[3]. Enqueue 9, 20.
  q=[9,20]. result=[[3]]

Level 1: levelSize=2
  dequeue 9 → currentLevel=[9]. No children.
  dequeue 20 → currentLevel=[9,20]. Enqueue 15, 7.
  q=[15,7]. result=[[3],[9,20]]

Level 2: levelSize=2
  dequeue 15 → currentLevel=[15]. No children.
  dequeue 7  → currentLevel=[15,7]. No children.
  q=[]. result=[[3],[9,20],[15,7]]

Done. ✓
*/
```

**Common variants — know all of them:**

```cpp
// Zigzag level order: alternate left-to-right and right-to-left
vector<vector<int>> zigzagLevelOrder(TreeNode* root) {
    vector<vector<int>> result;
    if (!root) return result;
    queue<TreeNode*> q;
    q.push(root);
    bool leftToRight = true;

    while (!q.empty()) {
        int sz = q.size();
        deque<int> level;                       // deque for O(1) front and back insert
        for (int i = 0; i < sz; i++) {
            TreeNode* node = q.front(); q.pop();
            if (leftToRight) level.push_back(node->val);
            else             level.push_front(node->val); // reverse by inserting at front
            if (node->left)  q.push(node->left);
            if (node->right) q.push(node->right);
        }
        result.push_back(vector<int>(level.begin(), level.end()));
        leftToRight = !leftToRight;
    }
    return result;
}

// Maximum value per level
vector<int> largestValues(TreeNode* root) {
    vector<int> result;
    if (!root) return result;
    queue<TreeNode*> q;
    q.push(root);
    while (!q.empty()) {
        int sz = q.size(), levelMax = INT_MIN;
        for (int i = 0; i < sz; i++) {
            TreeNode* node = q.front(); q.pop();
            levelMax = max(levelMax, node->val);
            if (node->left)  q.push(node->left);
            if (node->right) q.push(node->right);
        }
        result.push_back(levelMax);
    }
    return result;
}

// Right side view: last node at each level
vector<int> rightSideView(TreeNode* root) {
    vector<int> result;
    if (!root) return result;
    queue<TreeNode*> q;
    q.push(root);
    while (!q.empty()) {
        int sz = q.size();
        for (int i = 0; i < sz; i++) {
            TreeNode* node = q.front(); q.pop();
            if (i == sz - 1) result.push_back(node->val);  // last in level = rightmost
            if (node->left)  q.push(node->left);
            if (node->right) q.push(node->right);
        }
    }
    return result;
}
```

**Edge cases:**
- Empty tree → return `[]`
- Single node → `[[root->val]]`
- Left-skewed tree (every node has only left child) → n levels, each with one element
- Complete binary tree → bottom level may have up to n/2 nodes — queue size can reach O(n)

---

### Problem 2: Shortest Path in a Binary Matrix

**Problem:** Given an `n×n` binary matrix `grid`, find the length of the shortest clear path from top-left `(0,0)` to bottom-right `(n-1,n-1)`. A clear path only moves through cells with value `0` and moves in 8 directions (horizontally, vertically, or diagonally). Return -1 if no path exists.

**Why BFS and not DFS?** BFS guarantees the first time it reaches any cell is via the shortest path. DFS would find *a* path, but not necessarily the shortest one. BFS is the right tool for unweighted shortest path in any graph — grid or otherwise.

```cpp
// Time: O(n²) — each cell visited at most once
// Space: O(n²) — queue may hold all cells in worst case
int shortestPathBinaryMatrix(vector<vector<int>>& grid) {
    int n = grid.size();

    // Edge cases: start or end is blocked
    if (grid[0][0] == 1 || grid[n-1][n-1] == 1) return -1;

    // Single cell grid
    if (n == 1) return 1;

    // 8-directional movement
    int dirs[8][2] = {{-1,-1},{-1,0},{-1,1},{0,-1},{0,1},{1,-1},{1,0},{1,1}};

    queue<tuple<int,int,int>> q;    // {row, col, path_length}
    q.push({0, 0, 1});
    grid[0][0] = 1;                 // mark visited by setting to 1 (avoid separate visited array)

    while (!q.empty()) {
        auto [r, c, dist] = q.front(); q.pop();

        for (auto& d : dirs) {
            int nr = r + d[0], nc = c + d[1];

            if (nr < 0 || nr >= n || nc < 0 || nc >= n) continue;   // out of bounds
            if (grid[nr][nc] == 1) continue;                          // blocked or visited

            if (nr == n-1 && nc == n-1) return dist + 1;             // reached target

            grid[nr][nc] = 1;       // mark visited
            q.push({nr, nc, dist + 1});
        }
    }
    return -1;   // no path found
}

/*
Grid:
  [0][0][0]
  [1][1][0]
  [1][1][0]

BFS from (0,0), dist=1:
  Explore 8 neighbours. Valid unblocked: (0,1) dist=2, (1,... blocked)
  Enqueue: (0,1,2)

From (0,1), dist=2:
  Valid: (0,2,3)
  Enqueue: (0,2,3)

From (0,2), dist=3:
  Valid: (1,2,4)
  Enqueue: (1,2,4)

From (1,2), dist=4:
  Valid: (2,2,5) — this is (n-1,n-1)! Return 5. ✓
*/
```

**Alternative — store distance in a separate array (do not modify input):**

```cpp
int shortestPathBinaryMatrix(vector<vector<int>>& grid) {
    int n = grid.size();
    if (grid[0][0] || grid[n-1][n-1]) return -1;

    vector<vector<int>> dist(n, vector<int>(n, -1));
    dist[0][0] = 1;

    queue<pair<int,int>> q;
    q.push({0, 0});

    int dirs[8][2] = {{-1,-1},{-1,0},{-1,1},{0,-1},{0,1},{1,-1},{1,0},{1,1}};

    while (!q.empty()) {
        auto [r, c] = q.front(); q.pop();

        if (r == n-1 && c == n-1) return dist[r][c];

        for (auto& d : dirs) {
            int nr = r + d[0], nc = c + d[1];
            if (nr < 0 || nr >= n || nc < 0 || nc >= n) continue;
            if (grid[nr][nc] == 1 || dist[nr][nc] != -1) continue;
            dist[nr][nc] = dist[r][c] + 1;
            q.push({nr, nc});
        }
    }
    return -1;
}
```

**Edge cases:**
- `grid[0][0] == 1` → no path, return -1 immediately
- `grid[n-1][n-1] == 1` → no path, return -1 immediately
- n=1, `grid[0][0] == 0` → path length is 1 (start = end)
- All cells are 0 → BFS finds the shortest diagonal path
- Already visited cells must be marked — forgetting this causes infinite loops on grids with cycles (any grid with 8-directional movement)

---

### Problem 3: Implement Queue using Two Stacks

**Problem:** Implement a queue's `push`, `pop`, `peek`, and `empty` operations using only two stacks. The `pop` and `peek` operations must be amortized O(1).

**This problem tests understanding of amortised analysis and the relationship between stacks and queues.**

**Thought process:**

> "A stack reverses order. Two reversals restore the original order. If I push everything into an 'inbox' stack, and pour it into an 'outbox' stack when I need to dequeue, the outbox gives elements in FIFO order — because pouring reverses the reversal."
>
> "Key insight for amortised O(1): once an element is in the outbox, it stays there until dequeued. It moves from inbox to outbox exactly once in its lifetime. Total moves for n operations = n. Amortised cost per operation = O(n)/n = O(1)."

```cpp
class MyQueue {
private:
    stack<int> inbox_;    // receives push operations
    stack<int> outbox_;   // serves pop/peek operations

    // Transfer from inbox to outbox if outbox is empty
    // O(n) worst case, but amortised O(1) per element
    void transfer() {
        if (outbox_.empty()) {
            while (!inbox_.empty()) {
                outbox_.push(inbox_.top());
                inbox_.pop();
            }
        }
    }

public:
    MyQueue() = default;

    // Always O(1) — just push to inbox
    void push(int x) {
        inbox_.push(x);
    }

    // Amortised O(1) — transfer only when outbox empty
    int pop() {
        transfer();
        if (outbox_.empty()) throw underflow_error("queue is empty");
        int val = outbox_.top();
        outbox_.pop();
        return val;
    }

    // Amortised O(1)
    int peek() {
        transfer();
        if (outbox_.empty()) throw underflow_error("queue is empty");
        return outbox_.top();
    }

    bool empty() const {
        return inbox_.empty() && outbox_.empty();
    }
};

/*
Trace:
push(1): inbox=[1], outbox=[]
push(2): inbox=[2,1], outbox=[]      (2 on top)
peek():  outbox empty → transfer.
         inbox=[2,1] → pour → outbox=[1,2]  (1 on top)
         return outbox.top() = 1  ✓  (FIFO: 1 was pushed first)
pop():   outbox=[1,2] non-empty, no transfer.
         return 1. outbox=[2]
push(3): inbox=[3], outbox=[2]
pop():   outbox non-empty. return 2. outbox=[]
pop():   outbox empty → transfer.
         inbox=[3] → outbox=[3]
         return 3.
*/
```

**The amortised analysis — bank account argument:**

Charge each `push` operation $2 (instead of the real $1):
- $1 pays for the push itself
- $1 is saved in the element's "account" for the future pour

When `transfer()` is called and n elements are poured:
- Cost: n pops from inbox + n pushes to outbox = 2n operations
- Payment: n elements × $1 saved = $n available
- Wait — that's $n, not $2n. Adjust: charge $3 per push ($1 push + $2 saved)
- Actually the pour costs $1 per element (one pop + one push), so save $1 per element
- Charge $2 per push: $1 push + $1 saved. Pour cost per element = $1. Bank covers it ✓

Every operation is paid for. No operation is ever "in debt". Hence amortised O(1).

**Edge cases:**
- `pop()` on empty queue → both stacks empty after transfer → throw
- `peek()` followed by `pop()` → outbox is non-empty after peek's transfer, pop uses it directly (no redundant transfer)
- Alternating push and pop: worst case is when outbox empties and inbox fills before each pop — but the total pours across all operations is still n

---

## 10. Real-World Uses

| Domain | Use Case | Why Queue |
|---|---|---|
| OS scheduler | CPU round-robin process scheduling | FIFO ensures fairness; each process gets a quantum in arrival order |
| Network routers | Packet buffering and forwarding | Packets queued in arrival order; FIFO prevents reordering |
| Web servers | HTTP request queue (thread pool) | Requests served in order; load balancing across worker threads |
| Message brokers | Kafka topics, RabbitMQ, SQS | Producers enqueue; consumers dequeue; FIFO ordering guarantees |
| Printers | Print job spooler | Jobs printed in submission order |
| Keyboard buffer | OS keyboard input buffer | Keystrokes enqueued as typed, dequeued by the application |
| BFS in compilers | Dependency resolution, build systems | Topological sort via BFS processes dependencies in FIFO order |
| CPU prefetcher | Memory prefetch request queue | Prefetch requests queued and served FIFO |
| Database | Write-ahead log (WAL) buffer | Transactions queued before writing to disk |
| Simulation | Discrete event simulation | Events processed in time order via a priority queue (queue variant) |
| Game servers | Player matchmaking queue | Players matched FIFO within skill brackets |
| Streaming | Video frame buffer | Frames decoded and queued for rendering at correct frame rate |

**Kafka — the world's most important queue:**

Apache Kafka is essentially a distributed, persistent, replicated queue. Producers append records to the back. Consumer groups read from the front at their own pace. Kafka retains records for days or weeks (unlike most queues that discard on consumption) and allows multiple independent consumers — each maintaining their own position (offset) in the queue. This makes Kafka simultaneously a message queue, an event log, and a stream processing backbone. At peak load, Kafka handles millions of messages per second per broker — all built on the fundamental FIFO principle.

---

## 11. Edge Cases & Pitfalls

### Pitfall 1: Off-by-One in Level-Size Snapshot

```cpp
// WRONG: q.size() changes as you enqueue children inside the loop
while (!q.empty()) {
    for (int i = 0; i < q.size(); i++) {   // q.size() grows during iteration!
        TreeNode* node = q.front(); q.pop();
        // ... enqueue children ...
    }
}
// If level has 3 nodes and you enqueue 6 children, q.size() goes 3→4→5→6
// i < q.size() remains true — you process next-level nodes in this iteration

// CORRECT: snapshot size BEFORE the inner loop
while (!q.empty()) {
    int levelSize = (int)q.size();         // snapshot once
    for (int i = 0; i < levelSize; i++) {  // fixed count
        TreeNode* node = q.front(); q.pop();
        // ... enqueue children ...
    }
}
```

### Pitfall 2: Forgetting to Mark Cells Visited Before Enqueuing

```cpp
// WRONG: mark visited when dequeuing — same cell gets enqueued multiple times
while (!q.empty()) {
    auto [r, c] = q.front(); q.pop();
    visited[r][c] = true;                    // too late!
    for (auto& dir : dirs) {
        int nr = r+dir[0], nc = c+dir[1];
        if (inBounds(nr,nc) && !visited[nr][nc])
            q.push({nr, nc});                // same cell pushed by multiple neighbours
    }
}

// CORRECT: mark visited when enqueuing — prevents duplicates in queue
// This can cause queue to hold O(n²) duplicate entries in worst case!
while (!q.empty()) {
    auto [r, c] = q.front(); q.pop();
    for (auto& dir : dirs) {
        int nr = r+dir[0], nc = c+dir[1];
        if (inBounds(nr,nc) && !visited[nr][nc]) {
            visited[nr][nc] = true;          // mark immediately
            q.push({nr, nc});
        }
    }
}
```

### Pitfall 3: Circular Queue False Full/Empty

```cpp
// WRONG: using front==tail for BOTH empty and full detection
bool empty() { return front == tail; }
bool full()  { return front == tail; }   // indistinguishable!

// CORRECT: use size counter
bool empty() { return size == 0; }
bool full()  { return size == capacity; }
```

### Pitfall 4: Modifying the Grid While BFS Is Running

```cpp
// Grid BFS: marking visited by modifying the grid is fine IF
// you mark BEFORE enqueuing, and IF modifying the grid is acceptable.
// If the grid must be preserved, use a separate visited matrix.

// Subtle bug: if the problem asks to restore the grid after BFS,
// you need the original values. Track what you changed.

vector<pair<int,int>> modified;   // track modified cells
grid[r][c] = 1;                   // mark visited
modified.push_back({r, c});
// ... BFS ...
for (auto [r, c] : modified) grid[r][c] = 0;   // restore
```

### Pitfall 5: Using BFS Where DFS (or vice versa) Is Needed

```cpp
// BFS finds SHORTEST path in unweighted graphs.
// DFS finds ANY path (not necessarily shortest).

// WRONG: using DFS to find shortest path
// DFS may find a very long path first. There is no guarantee.

// WRONG: using BFS to find all paths or detect back-edges
// BFS visits each node once. It does not naturally find all paths.
// For cycle detection, use DFS with visited + recursion-stack tracking.

// Rule:
// Shortest path (unweighted) → BFS
// Existence of path          → either (DFS often simpler)
// All paths                  → DFS with backtracking
// Detect cycles              → DFS
// Level-order / distance     → BFS
```

### Pitfall 6: Infinite Loop When Graph Has Cycles and No Visited Set

```cpp
// For graphs (not trees), BFS without visited tracking loops forever on cycles

// WRONG: no visited tracking on a cyclic graph
while (!q.empty()) {
    int node = q.front(); q.pop();
    for (int neighbour : adj[node]) {
        q.push(neighbour);   // re-enqueues already-processed nodes → infinite!
    }
}

// CORRECT: always track visited nodes in graphs
unordered_set<int> visited;
visited.insert(start);
q.push(start);
while (!q.empty()) {
    int node = q.front(); q.pop();
    for (int neighbour : adj[node]) {
        if (!visited.count(neighbour)) {
            visited.insert(neighbour);
            q.push(neighbour);
        }
    }
}
```

### Pitfall 7: `std::queue` Has No `clear()` — Clearing Requires a Workaround

```cpp
queue<int> q;
// ... fill q ...

// WRONG: no q.clear() in std::queue
q.clear();   // compile error

// Correct option 1: reassign an empty queue
q = queue<int>();

// Correct option 2: swap with an empty queue (classic C++ idiom)
queue<int> empty;
swap(q, empty);   // q is now empty; empty holds the old data and is destroyed
```

---

## 12. Queue Variants at a Glance

| Variant | Description | Use Case | C++ Type |
|---|---|---|---|
| **Simple Queue** | FIFO, single-ended | BFS, scheduling | `std::queue` |
| **Circular Queue** | FIFO, fixed-size ring | Ring buffer, audio buffer | Custom implementation |
| **Deque** | Double-ended: push/pop both ends | Sliding window, palindrome | `std::deque` |
| **Priority Queue** | Always dequeues min or max | Dijkstra, event scheduling | `std::priority_queue` |
| **Monotonic Deque** | Deque maintaining sorted invariant | Sliding window maximum | Custom with `std::deque` |
| **Circular Buffer** | Fixed-size FIFO with overwrite | Audio, video streaming | Circular array |
| **Blocking Queue** | Thread-safe queue with wait | Producer-consumer | `std::queue` + mutex |
| **0-1 BFS Deque** | Deque for 0/1 weight shortest path | Grid shortest path | `std::deque` |

Each variant has its own README in this curriculum. The simple queue is the foundation — master it first.

---

## 13. std::queue

```cpp
#include <queue>
#include <deque>   // default underlying container
#include <list>

// std::queue is a container adaptor — restricts a deque to FIFO access only
queue<int> q1;                          // default: deque<int> underneath
queue<int, list<int>> q2;              // list-backed: guaranteed O(1) no amortised
queue<int, deque<int>> q3;            // same as default, explicit

// Complete API:
q1.push(10);           // enqueue at back — O(1) amortised
q1.emplace(10);        // construct in place at back — prefer for complex types
q1.pop();              // dequeue from front — O(1), returns void!
q1.front();            // peek at front — O(1), reference
q1.back();             // peek at back — O(1), reference
q1.empty();            // O(1)
q1.size();             // O(1)
q1.swap(q3);           // O(1) swap

// What std::queue does NOT have:
// - iteration (no begin/end)
// - indexed access (no operator[])
// - clear() — use q = queue<int>() instead
// - search

// In competitive programming — use deque<int> directly:
deque<int> q;
q.push_back(val);      // enqueue
q.pop_front();         // dequeue
q.front();             // peek front
q.back();              // peek back
q.empty();             // empty check
// Bonus: q[i] works for indexed access, useful for debugging
```

---

## 14. Comparison: Queue vs Stack vs Deque vs Priority Queue

| Feature | Queue | Stack | Deque | Priority Queue |
|---|---|---|---|---|
| Access order | FIFO | LIFO | Both ends | By priority (min/max) |
| Insert | O(1) back | O(1) top | O(1) front & back | O(log n) |
| Remove | O(1) front | O(1) top | O(1) front & back | O(log n) |
| Peek | O(1) front | O(1) top | O(1) front & back | O(1) min/max |
| Search | O(n) | O(n) | O(n) | O(n) |
| Ordering | Arrival order | Reverse arrival | Flexible | Value-based |
| BFS | ★ Natural | Poor | Can simulate | — |
| DFS | Poor | ★ Natural | Can simulate | — |
| Dijkstra | Poor | Poor | 0-1 weights | ★ Natural |
| Sliding window | Poor | Poor | ★ Natural | Can (slower) |
| Undo/redo | Poor | ★ Natural | — | — |
| C++ type | `std::queue` | `std::stack` | `std::deque` | `std::priority_queue` |
| Underlying | `std::deque` | `std::deque` | Chunked array | Binary heap |

**The BFS vs DFS decision revisited:**

```
Both BFS and DFS visit every node exactly once: O(V + E) time, O(V) space.
The only difference is which node to process next:

    Use a queue (BFS):
        → Shortest path (unweighted)
        → Level-order processing
        → Minimum steps / moves
        → Connected components (either works, BFS gives level info)

    Use a stack (DFS):
        → Cycle detection
        → Topological sort
        → Finding all paths
        → Backtracking problems
        → Detecting strongly connected components

    The data structure IS the algorithm. Swap queue for stack:
    same code, entirely different traversal order, entirely different guarantees.
```

---

## 15. Self-Test Questions

1. **What is FIFO? Give three real-world examples that are not programming queues.**

2. **Why does a naive array implementation of a queue fail? What problem does the circular array solve, and what is the key formula for advancing indices?**

3. **In the level-order traversal, why must `levelSize = q.size()` be captured BEFORE the inner loop? Give a concrete example where forgetting this produces wrong output.**

4. **Trace the Two-Stack Queue on: `push(1), push(2), push(3), pop(), push(4), pop(), pop()`. Show the state of both stacks after each operation. What is amortised O(1) and why does it apply here?**

5. **BFS guarantees the shortest path on an unweighted graph. Prove this is true — why does DFS not have this guarantee?**

6. **In the Shortest Path in Binary Matrix problem, why must cells be marked visited when enqueued rather than when dequeued? What is the consequence of marking on dequeue?**

7. **What is multi-source BFS? Trace it on a 3×3 grid with rotten oranges at (0,0) and (2,2). Show the queue state after each level.**

8. **What is 0-1 BFS and when is it better than Dijkstra's algorithm? What does the deque replace and why?**

9. **Write the BFS template for a grid problem from memory. Include: initialisation, visited marking, direction vectors, bounds checking, and level tracking.**

10. **Design a system that processes web requests fairly across 1000 concurrent users. What queue properties matter? What happens if a priority queue is used instead of a FIFO queue?**

---

## Quick Reference Card

```
Queue — FIFO access. The oldest unresolved thing is always served next.

Core ADT:         enqueue(x)  dequeue()  front()  empty()  — all O(1)
Best backing:     std::queue (deque underneath) or std::deque directly
In competitive:   deque<int> q; q.push_back(x); q.pop_front(); q.front();

Implementations:
  Circular array  → O(1) all ops, fixed capacity, cache-friendly, zero overhead
  Linked list     → O(1) all ops, unbounded, 8B overhead per element
  Two stacks      → O(1) amortised, good when only stacks are available
  std::queue      → deque-backed, use in production and interviews

BFS template (memorise this):
  queue<Node*> q;
  visited.insert(start); q.push(start);
  while (!q.empty()) {
      int sz = q.size();              // ← level snapshot
      for (int i = 0; i < sz; i++) {
          Node* curr = q.front(); q.pop();
          for (Node* next : neighbours(curr))
              if (!visited.count(next)) { visited.insert(next); q.push(next); }
      }
      level++;
  }

BFS properties:
  → First arrival at any node = shortest path (unweighted)
  → Level = distance from source
  → Queue holds at most one or two levels at a time

Critical rules:
  → Mark visited WHEN ENQUEUING, not when dequeuing (prevents duplicates)
  → Snapshot q.size() before inner loop for level-order processing
  → std::queue::pop() returns void — use front() then pop()
  → No clear() in std::queue — use q = queue<T>() to clear
  → For grids, mark visited or you get infinite loops on cycles

Use queue when:
  ✓ Shortest path in unweighted graph/grid
  ✓ Level-order / BFS traversal
  ✓ Minimum steps / moves
  ✓ Fairness / arrival-order processing (scheduling, buffers)
  ✗ Need deepest path first → use stack (DFS)
  ✗ Need priority order → use priority_queue
  ✗ Need O(1) push/pop at both ends → use deque
```

---

*Previous: [07 — Stack](./07_stack.md)*
*Next: [10 — Deque](./10_deque.md)*
*See also: [08 — Monotonic Stack](./08_monotonic_stack.md) | [Priority Queue](./11_priority_queue.md) | [BFS algorithms](./algorithms/bfs.md)*

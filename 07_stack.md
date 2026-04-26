# Stack

> **Curriculum position:** Linear Structures → #7
> **Interview weight:** ★★★★★ Critical — underpins parsing, recursion, backtracking, and the monotonic stack pattern
> **Difficulty:** Beginner concept, expert-level interview applications
> **Prerequisite:** [01 — Static Array](./01_static_array.md) | [03 — Singly Linked List](./03_singly_linked_list.md)

---

## Table of Contents

1. [Intuition First — The LIFO Principle](#1-intuition-first)
2. [Abstract Data Type vs Concrete Implementation](#2-abstract-data-type-vs-concrete-implementation)
3. [Internal Working — Two Implementations](#3-internal-working)
4. [The Call Stack — The Stack You Use Every Second](#4-the-call-stack)
5. [Time & Space Complexity](#5-time--space-complexity)
6. [Complete C++ Implementation](#6-complete-c-implementation)
7. [Core Operations — Visualised](#7-core-operations--visualised)
8. [Common Patterns & Techniques](#8-common-patterns--techniques)
9. [Interview Problems](#9-interview-problems)
10. [Real-World Uses](#10-real-world-uses)
11. [Edge Cases & Pitfalls](#11-edge-cases--pitfalls)
12. [std::stack — The Standard Library Version](#12-stdstack)
13. [Comparison: Stack vs Queue vs Deque](#13-comparison)
14. [Self-Test Questions](#14-self-test-questions)

---

## 1. Intuition First

Imagine a stack of plates in a cafeteria. The kitchen adds clean plates to the top. Diners take plates from the top. Nobody reaches into the middle. Nobody takes from the bottom. The most recently added plate is always the first one taken. The plate added longest ago sits at the bottom, untouched, until every plate above it has been removed.

This is **LIFO: Last In, First Out**. The last item pushed onto the stack is the first item popped off.

This single access rule — only the top is ever visible or modifiable — sounds like a restriction. And it is. But restrictions create guarantees, and guarantees create power. Because you can only access the top, a stack naturally tracks **the most recent unresolved thing**. That is a surprisingly general concept:

- The most recent unclosed parenthesis that still needs a closing match
- The most recent function call that has not yet returned
- The most recent undo action that has not yet been undone
- The most recently visited webpage in a browser history
- The most recent directory entered in a file system traversal

Every one of these is a stack problem. The key pattern: whenever you encounter something that needs to be "resolved later" and resolutions happen in reverse order of encounters, reach for a stack.

**The stack is the first Abstract Data Type (ADT) in this curriculum.** An array is a concrete data structure — its identity is its memory layout. A stack is an ADT — its identity is its behaviour contract (LIFO), not its implementation. It can be built on an array, a linked list, or anything else that satisfies the contract.

---

## 2. Abstract Data Type vs Concrete Implementation

This distinction matters for interviews. When an interviewer asks "implement a stack", they want the ADT interface. When they ask "what is the underlying data structure of std::stack", they want the implementation.

### The Stack ADT Contract

```
push(x)   — add element x to the top               must be O(1)
pop()     — remove and return the top element        must be O(1)
top()     — return the top element without removing  must be O(1)
empty()   — return true if no elements               must be O(1)
size()    — return number of elements                must be O(1) or O(n)
```

Any implementation that satisfies these five operations with O(1) push/pop/top is a valid stack. The internal structure is irrelevant to the contract.

### Two Canonical Implementations

```
Implementation 1: Array-backed stack
  Pros: O(1) everything, cache-friendly, minimal overhead
  Cons: fixed capacity (static array) or occasional O(n) resize (dynamic array)
  Best for: bounded-size stacks, performance-critical code

Implementation 2: Linked-list-backed stack
  Pros: truly unbounded, O(1) guaranteed (no resize)
  Cons: per-node heap allocation, pointer overhead, cache misses
  Best for: stacks where size is completely unpredictable

In interviews: use std::vector as the backing store. Zero overhead, O(1) amortized
push, O(1) pop, and naturally expresses the "top = back of vector" convention.
```

---

## 3. Internal Working — Two Implementations

### Implementation 1: Array-Backed (the better one)

```
stack = [ 10 | 20 | 30 |    |    ]
          idx:  0    1    2    3    4
                               ↑
                             top (points to last occupied index)

push(40):  arr[++top] = 40
           [ 10 | 20 | 30 | 40 |    ]
                               ↑
                             top = 3

pop():     val = arr[top--]
           [ 10 | 20 | 30 |    |    ]  (40 is logically removed; memory unchanged)
                          ↑
                        top = 2

top():     return arr[top]   ← always index top, no movement
```

The `top` variable is an index (or a pointer for array-based stacks). The data never moves — only `top` changes. Push increments `top` then writes. Pop reads then decrements `top`. Both are a single memory access plus an integer increment/decrement: genuinely O(1) with essentially zero constant factor.

### Implementation 2: Linked-List-Backed

```
stack: top → [30] → [20] → [10] → null
              ↑
           new elements added here (push = insert at head)
           elements removed here  (pop  = remove head)

push(40):  new node [40]; [40]->next = top; top = [40]
           top → [40] → [30] → [20] → [10] → null

pop():     val = top->val; old = top; top = top->next; delete old; return val
           top → [30] → [20] → [10] → null
```

The head of the linked list is the top of the stack. Push = insert at head (O(1)). Pop = remove head (O(1)). No resizing ever occurs.

### Why Array-Backed Wins in Practice

```
For n = 1,000,000 push+pop operations:

Array-backed (vector):
  - Each push: one bounds check + one assignment + increment top
  - Cache: array is contiguous — top is always in the same cache line as the element below
  - Resizes: ~20 reallocations, each O(n) amortized to O(1)
  - Memory: n × sizeof(T) bytes

Linked-list-backed:
  - Each push: heap allocation (malloc/new) — typically 50-200 ns per call
  - Cache: each node is a separate heap allocation, scattered across memory
  - Memory: n × (sizeof(T) + sizeof(pointer)) bytes — 8 bytes overhead per element

Benchmark result (typical): array-backed is 3-10× faster for small element types.
For large elements (structs > 100 bytes), the difference shrinks.
```

**Rule of thumb:** Always use the array-backed (vector) implementation unless you have a specific reason for the linked list version (e.g., you need guaranteed O(1) with no amortized cost, which is rare in practice).

---

## 4. The Call Stack — The Stack You Use Every Second

The call stack is not a metaphor. It is a literal hardware-managed stack that tracks every function call in every running program. Understanding it deeply explains recursion, stack overflow, and how your code actually executes.

### How the Call Stack Works

```cpp
int multiply(int a, int b) {
    return a * b;                          // frame 3
}

int add_products(int x, int y, int z) {
    return multiply(x, y) + multiply(y, z); // frame 2
}

int main() {
    int result = add_products(2, 3, 4);    // frame 1
    return 0;
}
```

When `main` calls `add_products`, the CPU:
1. Pushes a **stack frame** onto the call stack containing: return address, parameters, local variables
2. Jumps to `add_products`

When `add_products` calls `multiply`, the CPU pushes another frame. The call stack at that moment:

```
High address (stack grows DOWN on x86)
┌──────────────────────────────┐  ← stack bottom (initial SP)
│  main's frame                │
│    local: result (uninit)    │
│    return addr: OS           │
├──────────────────────────────┤
│  add_products's frame        │
│    params: x=2, y=3, z=4     │
│    return addr: main+offset  │
│    local: (none)             │
├──────────────────────────────┤
│  multiply's frame            │
│    params: a=2, b=3          │
│    return addr: add_products │  ← SP (stack pointer) — current top
└──────────────────────────────┘
Low address
```

When `multiply` returns:
- Its frame is popped (SP moves back up)
- The return value is placed in a register
- Execution resumes at the saved return address

**Stack overflow** occurs when the stack grows so deep that SP reaches the memory reserved for the heap. Infinite recursion is the classic cause — each recursive call pushes a frame, and the frames accumulate until memory is exhausted.

```cpp
// This overflows the stack
int infinite(int n) {
    return infinite(n + 1);   // no base case — pushes frames forever
}

// Stack depth at overflow (typical): ~10,000-100,000 frames depending on OS
// Each frame is usually 16-256 bytes (parameters + locals + saved registers)
```

**Why does iterative code replace recursive code?** Converting recursion to iteration is just making the implicit call stack explicit with a manual stack on the heap. The heap has far more capacity than the call stack (usually 1-8 MB vs multiple GB).

---

## 5. Time & Space Complexity

### Operation Complexities

| Operation | Array-backed | Linked-list-backed | Notes |
|---|---|---|---|
| `push(x)` | **O(1) amortized** | **O(1)** | Array: occasional resize; LL: always O(1) |
| `pop()` | **O(1)** | **O(1)** | Array: decrement top; LL: remove head |
| `top()` | **O(1)** | **O(1)** | Array: index access; LL: head->val |
| `empty()` | **O(1)** | **O(1)** | top == -1 or head == nullptr |
| `size()` | **O(1)** | **O(1) with counter** | Maintain a size variable |
| `clear()` | **O(n)** | **O(n)** | Must destruct n elements |
| Search | **O(n)** | **O(n)** | Stacks are not for searching |

### Space Complexity

| | Array-backed | Linked-list-backed |
|---|---|---|
| n elements | O(n) | O(n) |
| Overhead | Up to 2× capacity waste (pre-resize) | 8 bytes per element (pointer) |
| Peak memory | O(n) amortized | O(n) |

---

## 6. Complete C++ Implementation

### Implementation 1: Array-Backed Stack (Recommended)

```cpp
#include <iostream>
#include <stdexcept>
#include <vector>
#include <initializer_list>
using namespace std;

template<typename T>
class Stack {
private:
    vector<T> data_;   // vector handles resizing automatically

public:
    Stack() = default;

    Stack(initializer_list<T> vals) {
        for (const T& v : vals) push(v);
    }

    // ── Core ADT Operations ───────────────────────────────────────────────────

    // O(1) amortized — add element to top
    void push(const T& val) {
        data_.push_back(val);
    }

    void push(T&& val) {
        data_.push_back(move(val));
    }

    // Construct in-place at top — no temporary, no copy
    template<typename... Args>
    void emplace(Args&&... args) {
        data_.emplace_back(forward<Args>(args)...);
    }

    // O(1) — remove top element
    void pop() {
        if (data_.empty()) throw underflow_error("pop on empty stack");
        data_.pop_back();
    }

    // O(1) — peek at top without removing
    T& top() {
        if (data_.empty()) throw underflow_error("top on empty stack");
        return data_.back();
    }

    const T& top() const {
        if (data_.empty()) throw underflow_error("top on empty stack");
        return data_.back();
    }

    // O(1) — state queries
    bool   empty() const { return data_.empty(); }
    size_t size()  const { return data_.size(); }

    // Reserve capacity when final size is known — eliminates all resizes
    void reserve(size_t n) { data_.reserve(n); }

    // Print from top to bottom (for debugging)
    void print() const {
        if (data_.empty()) { cout << "(empty stack)\n"; return; }
        cout << "top → ";
        for (int i = (int)data_.size() - 1; i >= 0; i--) {
            cout << data_[i];
            if (i > 0) cout << " → ";
        }
        cout << " → bottom\n";
    }
};

// ── Convenience: pop and return in one call ───────────────────────────────────
// std::stack doesn't provide this (for exception-safety reasons in generic code)
// Safe to add in our implementation for interview use
template<typename T>
T pop_top(Stack<T>& s) {
    T val = s.top();
    s.pop();
    return val;
}
```

### Implementation 2: Linked-List-Backed Stack

```cpp
template<typename T>
class LinkedStack {
private:
    struct Node {
        T     val;
        Node* next;
        Node(const T& v, Node* n) : val(v), next(n) {}
    };

    Node*  top_;
    size_t size_;

public:
    LinkedStack() : top_(nullptr), size_(0) {}

    ~LinkedStack() {
        while (top_) {
            Node* tmp = top_->next;
            delete top_;
            top_ = tmp;
        }
    }

    LinkedStack(const LinkedStack&)            = delete;
    LinkedStack& operator=(const LinkedStack&) = delete;

    // O(1) — guaranteed, no amortization
    void push(const T& val) {
        top_ = new Node(val, top_);   // insert at head of list
        size_++;
    }

    // O(1)
    void pop() {
        if (!top_) throw underflow_error("pop on empty stack");
        Node* tmp = top_;
        top_ = top_->next;
        delete tmp;
        size_--;
    }

    T& top() {
        if (!top_) throw underflow_error("top on empty stack");
        return top_->val;
    }

    bool   empty() const { return top_ == nullptr; }
    size_t size()  const { return size_; }
};
```

### Implementation 3: Fixed-Capacity Stack (for competitive programming)

```cpp
// When you know the maximum stack size in advance — fastest possible
// No heap allocation, no bounds checking in hot path
template<typename T, size_t CAPACITY>
class FixedStack {
private:
    T      data_[CAPACITY];
    int    top_ = -1;

public:
    void   push(const T& val) { data_[++top_] = val; }  // undefined if full
    void   pop()              { --top_; }                 // undefined if empty
    T&     top()              { return data_[top_]; }
    bool   empty()      const { return top_ == -1; }
    int    size()       const { return top_ + 1; }
    bool   full()       const { return top_ == (int)CAPACITY - 1; }
};

// Usage in competitive programming:
FixedStack<int, 100005> stk;   // global — zero-initialised, no runtime allocation
```

---

## 7. Core Operations — Visualised

### Push and Pop Sequence

```
Operation    Stack State (bottom→top)    top index
─────────────────────────────────────────────────
(empty)      []                          -1
push(10)     [10]                         0
push(20)     [10, 20]                     1
push(30)     [10, 20, 30]                 2
top()        returns 30 (no change)       2
pop()        [10, 20]        removed: 30  1
push(40)     [10, 20, 40]                 2
pop()        [10, 20]        removed: 40  1
pop()        [10]            removed: 20  0
pop()        []              removed: 10 -1
pop()        ← underflow_error!
```

### Bracket Matching (the canonical stack problem, visualised)

```
Input: "({[]})"

Process each character:
  '(' → push            stack: ['(']
  '{' → push            stack: ['(', '{']
  '[' → push            stack: ['(', '{', '[']
  ']' → closing bracket → pop '[' → match! ✓   stack: ['(', '{']
  '}' → closing bracket → pop '{' → match! ✓   stack: ['(']
  ')' → closing bracket → pop '(' → match! ✓   stack: []

End: stack empty → VALID ✓

Input: "({)}"
  '(' → push            stack: ['(']
  '{' → push            stack: ['(', '{']
  ')' → closing bracket → pop '{' → MISMATCH! '{' ≠ ')'  → INVALID ✗
```

### The Key Insight — Stack as "Unresolved Things"

```
Every push = "I have encountered something that needs a future resolution"
Every pop  = "I have now resolved the most recent unresolved thing"

Brackets:   push open brackets; pop on close bracket (resolution = matched pair)
Recursion:  push stack frame; pop on return (resolution = function completed)
Undo/redo:  push actions; pop on undo (resolution = action reversed)
DFS:        push nodes; pop to visit (resolution = node fully explored)
Parsing:    push operators; pop when precedence dictates (resolution = evaluated)
```

---

## 8. Common Patterns & Techniques

### Pattern 1: Bracket / Parenthesis Validation

Push opening brackets. On every closing bracket, check if the top of the stack is the matching opener.

```cpp
bool isValid(const string& s) {
    stack<char> stk;
    for (char c : s) {
        if (c == '(' || c == '[' || c == '{') {
            stk.push(c);
        } else {
            // Closing bracket: stack must be non-empty with matching opener
            if (stk.empty()) return false;
            char top = stk.top(); stk.pop();
            if ((c == ')' && top != '(') ||
                (c == ']' && top != '[') ||
                (c == '}' && top != '{')) return false;
        }
    }
    return stk.empty();   // stack must be empty at the end — all opened were closed
}
```

### Pattern 2: Monotonic Stack — The Most Powerful Stack Technique

A monotonic stack maintains elements in strictly increasing or strictly decreasing order. When a new element violates the order, pop elements from the stack until the order is restored. Each pop represents finding the answer for the popped element.

This converts a class of O(n²) brute-force problems into O(n). Every element is pushed once and popped once — total work is O(n).

```cpp
// Next Greater Element: for each element, find the next element to its right
// that is strictly greater. Return -1 if none exists.
// Example: [2, 1, 2, 4, 3] → [4, 2, 4, -1, -1]

vector<int> nextGreaterElement(vector<int>& nums) {
    int n = nums.size();
    vector<int> result(n, -1);
    stack<int> stk;     // stores INDICES, not values — indices let us fill result[]

    for (int i = 0; i < n; i++) {
        // While the current element is greater than what the stack top was "waiting" for:
        while (!stk.empty() && nums[i] > nums[stk.top()]) {
            int idx = stk.top(); stk.pop();
            result[idx] = nums[i];   // nums[i] is the next greater for nums[idx]
        }
        stk.push(i);
    }
    // Elements remaining in stack have no greater element to their right → -1 (default)
    return result;
}

/*
Trace on [2, 1, 2, 4, 3]:

i=0: stk empty → push 0.  stk:[0]        (waiting for NGE of nums[0]=2)
i=1: nums[1]=1 ≤ nums[0]=2 → push 1. stk:[0,1]   (waiting for NGE of 1)
i=2: nums[2]=2 > nums[1]=1 → pop 1, result[1]=2. stk:[0]
     nums[2]=2 ≤ nums[0]=2 → stop. push 2. stk:[0,2]
i=3: nums[3]=4 > nums[2]=2 → pop 2, result[2]=4. stk:[0]
     nums[3]=4 > nums[0]=2 → pop 0, result[0]=4. stk:[]
     push 3. stk:[3]
i=4: nums[4]=3 ≤ nums[3]=4 → push 4. stk:[3,4]

End: stk=[3,4] → result[3]=-1, result[4]=-1 (already default)
Result: [4, 2, 4, -1, -1] ✓
*/
```

### Pattern 3: Expression Evaluation

Two stacks — one for operands, one for operators. Process tokens left to right, respecting precedence.

```cpp
// Evaluate an arithmetic expression like "3 + 4 * 2 - 1"
// Handles: +, -, *, / with correct precedence; no parentheses version shown

int precedence(char op) {
    if (op == '+' || op == '-') return 1;
    if (op == '*' || op == '/') return 2;
    return 0;
}

int applyOp(int a, int b, char op) {
    switch (op) {
        case '+': return a + b;
        case '-': return a - b;
        case '*': return a * b;
        case '/': return a / b;
    }
    return 0;
}

int evaluate(const string& expr) {
    stack<int>  vals;   // operand stack
    stack<char> ops;    // operator stack
    int i = 0, n = expr.size();

    while (i < n) {
        if (expr[i] == ' ') { i++; continue; }

        if (isdigit(expr[i])) {
            int num = 0;
            while (i < n && isdigit(expr[i])) num = num * 10 + (expr[i++] - '0');
            vals.push(num);
        } else if (expr[i] == '(') {
            ops.push(expr[i++]);
        } else if (expr[i] == ')') {
            while (!ops.empty() && ops.top() != '(') {
                int b = vals.top(); vals.pop();
                int a = vals.top(); vals.pop();
                vals.push(applyOp(a, b, ops.top())); ops.pop();
            }
            if (!ops.empty()) ops.pop();   // remove '('
            i++;
        } else {
            // Operator: pop operators of higher or equal precedence first
            while (!ops.empty() && ops.top() != '(' &&
                   precedence(ops.top()) >= precedence(expr[i])) {
                int b = vals.top(); vals.pop();
                int a = vals.top(); vals.pop();
                vals.push(applyOp(a, b, ops.top())); ops.pop();
            }
            ops.push(expr[i++]);
        }
    }
    while (!ops.empty()) {
        int b = vals.top(); vals.pop();
        int a = vals.top(); vals.pop();
        vals.push(applyOp(a, b, ops.top())); ops.pop();
    }
    return vals.top();
}
```

### Pattern 4: Iterative DFS Using an Explicit Stack

Converting recursive DFS to iterative — the most important recursion-to-iteration conversion.

```cpp
// Iterative preorder tree traversal using explicit stack
// Equivalent to recursive DFS without the call stack overhead
void iterativePreorder(TreeNode* root) {
    if (!root) return;
    stack<TreeNode*> stk;
    stk.push(root);

    while (!stk.empty()) {
        TreeNode* node = stk.top(); stk.pop();
        process(node->val);

        // Push RIGHT first so LEFT is processed first (LIFO)
        if (node->right) stk.push(node->right);
        if (node->left)  stk.push(node->left);
    }
}

// The implicit call stack of recursive DFS mirrors this exactly:
// Each stack frame IS the loop iteration — local variables + "resume point"
```

---

## 9. Interview Problems

### Problem 1: Valid Parentheses (Foundation Problem)

**Problem:** Given a string containing only `(`, `)`, `{`, `}`, `[`, `]`, determine if the input string is valid. Valid means: open brackets are closed by the same type of bracket, in the correct order.

**Examples:**
- `"()"` → `true`
- `"()[]{}"` → `true`
- `"(]"` → `false`
- `"([)]"` → `false`
- `"{[]}"` → `true`

**Thought process in an interview:**

> "Every time I see an opening bracket, I want to remember it for later. When I see a closing bracket, I need to match it with the most recently seen opening bracket. 'Most recently seen' is LIFO — a stack is the natural fit. Push on open, pop and check on close. If they don't match, invalid. If the stack is non-empty at the end, some brackets were never closed."

```cpp
// Time: O(n) | Space: O(n) — stack holds at most n/2 elements
bool isValid(const string& s) {
    stack<char> stk;

    for (char c : s) {
        // Push all opening brackets
        if (c == '(' || c == '[' || c == '{') {
            stk.push(c);
            continue;
        }

        // Closing bracket: must match the most recent opener
        if (stk.empty()) return false;   // nothing to match against

        char top = stk.top(); stk.pop();

        if (c == ')' && top != '(') return false;
        if (c == ']' && top != '[') return false;
        if (c == '}' && top != '{') return false;
    }

    return stk.empty();   // all openers must have been matched
}
```

**Edge cases:**
- Empty string → `true` (vacuously valid — the stack is immediately empty)
- String with only closers like `")"` → fails at first character (stack is empty)
- String with only openers like `"((("` → stack is non-empty at end → `false`
- Odd-length string → can never be valid (each bracket needs a partner) — you can add an O(1) early return: `if (s.size() % 2 != 0) return false`
- Single character → always `false`

**Common mistake:** Forgetting `stk.empty()` before `stk.top()` — this causes undefined behaviour.

---

### Problem 2: Min Stack — Design a Stack with O(1) getMin()

**Problem:** Design a stack that, in addition to push, pop, and top, supports retrieving the minimum element in the stack — all in O(1) time.

**Why is this hard?** A naive approach stores the minimum as a single variable. When the minimum is popped, finding the new minimum requires scanning the entire stack — O(n). The stack must somehow remember the minimum *at every historical state*.

**Thought process:**

> "Every time I push an element, I want to remember 'what is the minimum of everything currently in the stack, including this new element?' That minimum is valid until something is popped. When I pop, I want to instantly know 'what was the minimum before I pushed the thing I just popped?'"
>
> "Key insight: pair each element with the minimum of the stack at the moment it was pushed. When an element is popped, the minimum-at-push-time of the element below it is the new minimum. No scanning needed."

```cpp
class MinStack {
private:
    // Each slot stores: (value, min_at_this_level)
    // min_at_this_level = min of all elements from bottom up to and including this one
    vector<pair<int,int>> data_;   // {value, running_min}

public:
    MinStack() = default;

    void push(int val) {
        int current_min = data_.empty() ? val : min(val, data_.back().second);
        data_.emplace_back(val, current_min);
    }

    void pop() {
        if (data_.empty()) throw underflow_error("pop on empty stack");
        data_.pop_back();
    }

    int top() {
        if (data_.empty()) throw underflow_error("top on empty stack");
        return data_.back().first;
    }

    int getMin() {
        if (data_.empty()) throw underflow_error("getMin on empty stack");
        return data_.back().second;   // O(1) — stored alongside the top element
    }
};

/*
Trace:
push(5): data = [(5, 5)]                       min=5
push(3): data = [(5, 5), (3, 3)]               min=3
push(7): data = [(5, 5), (3, 3), (7, 3)]       min=3 (7 doesn't beat 3)
push(1): data = [(5, 5), (3, 3), (7, 3), (1, 1)] min=1
getMin() → 1  ✓
pop():   data = [(5, 5), (3, 3), (7, 3)]       min=3 (1 is gone; state before push(1) restored)
getMin() → 3  ✓
pop():   data = [(5, 5), (3, 3)]               min=3
pop():   data = [(5, 5)]                       min=5
getMin() → 5  ✓
*/
```

**Space:** O(n) — each element carries a `min` alongside it. Total space is 2n integers.

**Space-optimised variant:** Only push onto a second `min_stack` when the new element is ≤ the current minimum. Pop from `min_stack` only when the popped element equals the current minimum. This saves space in the average case but not in the worst case.

```cpp
// Two-stack variant — saves space when many elements exceed current min
class MinStackOptimised {
private:
    stack<int> main_stk_;
    stack<int> min_stk_;   // top always holds the current minimum

public:
    void push(int val) {
        main_stk_.push(val);
        if (min_stk_.empty() || val <= min_stk_.top()) {
            min_stk_.push(val);   // push only if new minimum or tie
        }
    }

    void pop() {
        int val = main_stk_.top(); main_stk_.pop();
        if (val == min_stk_.top()) min_stk_.pop();  // remove from min_stk_ if it was the min
    }

    int top()    { return main_stk_.top(); }
    int getMin() { return min_stk_.top(); }
};
```

---

### Problem 3: Largest Rectangle in Histogram

**Problem:** Given an array of non-negative integers `heights` representing the histogram bar heights (each bar has width 1), find the area of the largest rectangle that can be formed.

**Example:** `[2, 1, 5, 6, 2, 3]` → `10` (the rectangle formed by bars at indices 2 and 3, height 5, width 2)

**This is the hardest canonical stack problem. Mastering it unlocks dozens of similar problems.**

**Thought process — build from brute force:**

> "Brute force: for every pair (i, j), the rectangle height is min(heights[i..j]) and width is j - i + 1. Try all pairs: O(n²). Too slow for n = 100,000."
>
> "Key insight: for every bar, what is the largest rectangle where THIS bar is the shortest bar (i.e., determines the height)? If I know the leftmost and rightmost boundary where this bar is still the shortest, I can compute its rectangle in O(1)."
>
> "For bar i to be the bottleneck: the left boundary is the first bar to the left that is shorter than heights[i]. The right boundary is the first bar to the right that is shorter than heights[i]. This is exactly the 'Previous Smaller Element' and 'Next Smaller Element' problem — solvable with a monotonic stack in O(n)."

```cpp
// Time: O(n) | Space: O(n)
int largestRectangleArea(vector<int>& heights) {
    int n = heights.size();
    int maxArea = 0;

    // Monotonic stack stores indices in increasing order of height.
    // When a bar shorter than the stack top is encountered, the stack top
    // has found its right boundary (current bar) and left boundary (new stack top).
    stack<int> stk;   // stores indices

    for (int i = 0; i <= n; i++) {
        // Treat index n as a sentinel bar of height 0 — forces all remaining
        // bars out of the stack so we don't need a separate cleanup loop.
        int currentHeight = (i == n) ? 0 : heights[i];

        while (!stk.empty() && currentHeight < heights[stk.top()]) {
            int height = heights[stk.top()]; stk.pop();

            // Width: from (new stack top + 1) to (i - 1)
            // If stack is empty after pop, the popped bar extends all the way to index 0
            int width = stk.empty() ? i : i - stk.top() - 1;

            maxArea = max(maxArea, height * width);
        }
        stk.push(i);
    }
    return maxArea;
}

/*
Dry run on [2, 1, 5, 6, 2, 3]:

i=0: h=2, stk empty → push 0.        stk:[0]
i=1: h=1 < h[0]=2 → pop 0:
       height=2, stk empty → width=1 (i=1, left=0..0)
       area=2×1=2. maxArea=2.
     h=1, push 1.                     stk:[1]
i=2: h=5 > h[1]=1 → push 2.          stk:[1,2]
i=3: h=6 > h[2]=5 → push 3.          stk:[1,2,3]
i=4: h=2 < h[3]=6 → pop 3:
       height=6, stk top=2 → width=4-2-1=1
       area=6×1=6. maxArea=6.
     h=2 < h[2]=5 → pop 2:
       height=5, stk top=1 → width=4-1-1=2
       area=5×2=10. maxArea=10.   ★ answer here
     h=2 ≥ h[1]=1 → stop. push 4.   stk:[1,4]
i=5: h=3 > h[4]=2 → push 5.         stk:[1,4,5]
i=6: sentinel h=0:
     pop 5: height=3, stk top=4 → width=6-4-1=1. area=3. maxArea=10.
     pop 4: height=2, stk top=1 → width=6-1-1=4. area=8. maxArea=10.
     pop 1: height=1, stk empty → width=6. area=6. maxArea=10.

Return 10. ✓
*/
```

**Why the width formula works:**

When we pop index `mid` because `heights[i] < heights[mid]`:
- Right boundary: index `i` (the first element shorter than `heights[mid]` to the right)
- Left boundary: index `stk.top() + 1` after the pop (the first element shorter than `heights[mid]` to the left)
- Width = `i - stk.top() - 1`
- If stack is empty: left boundary extends to index 0, so width = `i`

**Edge cases:**
- All bars same height: answer is `height × n`
- Single bar: answer is `heights[0]`
- Strictly increasing heights `[1,2,3,4,5]`: largest rectangle is found when the sentinel pops everything
- Strictly decreasing heights `[5,4,3,2,1]`: each bar's rectangle is computed when the next shorter bar arrives

**Pattern:** This exact technique — monotonic stack for finding previous/next smaller/larger element — appears in: Trapping Rain Water, Maximum Rectangle in Binary Matrix, Sum of Subarray Minimums, Online Stock Span, and many others. Master it here and you unlock the entire problem family.

---

## 10. Real-World Uses

| Domain | Use Case | Why Stack |
|---|---|---|
| CPU / OS | Call stack for function invocation | LIFO matches function return order exactly |
| Compilers | Expression parsing, operator precedence | Shunting-yard algorithm uses two stacks |
| Browsers | Back navigation history | Each visit pushed; back button pops |
| Text editors | Undo/redo functionality | Actions pushed; undo pops last action |
| IDE / editors | Matching brackets, XML/HTML tag validation | Push open tags, validate on close |
| JVM / Python VM | Bytecode interpreter evaluation stack | Stack-based VMs use explicit operand stacks |
| OS memory management | Stack segment of process address space | Local variables, function frames |
| Depth-First Search | Explicit stack for iterative DFS | Graph traversal without recursion overflow |
| Postfix calculators (RPN) | HP calculators, Forth language | Push operands, apply operators immediately |
| CSS layout engine | Style inheritance stacking contexts | CSS `z-index` uses a rendering stack |
| Version control | Git's stash (`git stash push/pop`) | LIFO stack of uncommitted work |
| Backtracking | Solving mazes, Sudoku, N-Queens | Push state on exploration, pop on failure |

**The JVM Bytecode Stack — most Java/Python engineers don't know this:**

```java
// Java source code:
int result = a + b * c;

// Compiled JVM bytecode (stack-based VM):
iload_1        // push a
iload_2        // push b
iload_3        // push c
imul           // pop b,c; push b*c
iadd           // pop a, b*c; push a+b*c
istore_0       // pop result; store in local var 0

// The JVM literally uses a stack machine to evaluate every expression.
// Every arithmetic operation pops its operands and pushes its result.
// This is why understanding stacks matters for understanding compilation.
```

---

## 11. Edge Cases & Pitfalls

### Pitfall 1: Calling top() or pop() on an Empty Stack

```cpp
stack<int> stk;
stk.top();    // undefined behaviour in std::stack — no exception thrown!
stk.pop();    // undefined behaviour in std::stack — no exception thrown!

// std::stack does NOT check for underflow — it is UB, not an exception.
// In debug builds, it may crash with an assertion failure.
// In release builds, it silently reads or modifies garbage memory.

// ALWAYS guard:
if (!stk.empty()) {
    int val = stk.top();
    stk.pop();
}

// Or in your custom stack: throw underflow_error explicitly.
```

### Pitfall 2: Forgetting That pop() Returns void in std::stack

```cpp
// WRONG — std::stack::pop() returns void, not the value
int val = stk.pop();   // compile error

// CORRECT — peek then pop
int val = stk.top();
stk.pop();

// Or use a helper:
template<typename T>
T pop_and_get(std::stack<T>& s) {
    T val = s.top();
    s.pop();
    return val;
}
```

**Why does `std::stack::pop()` return void?** This was a deliberate design decision by the C++ standard committee for exception safety. If `pop()` returned by value and the copy constructor of T threw an exception, the element would have been removed from the stack but the return value would be undeliverable — lost forever. Separating `top()` (copy) and `pop()` (remove) means neither operation alone can create an unrecoverable state.

### Pitfall 3: Using a Stack When You Need to Iterate (Stacks Are Not for Searching)

```cpp
// WRONG mindset: searching a stack
std::stack<int> stk;
// ... populate stk ...

// You cannot search a stack without destroying it:
while (!stk.empty()) {
    if (stk.top() == target) { found = true; break; }
    stk.pop();   // ← permanently removes elements
}

// If you need to search, use a vector directly (not wrapped in stack)
// or maintain a parallel data structure (like the hash map in Min Stack)
```

### Pitfall 4: Monotonic Stack — Pushing Values vs Indices

```cpp
// WRONG: push values when you need to know the position
stack<int> stk;
stk.push(heights[i]);   // stored value — cannot compute width later!

// CORRECT: push indices — you can always recover the value via heights[idx]
stack<int> stk;
stk.push(i);
// Access value:  heights[stk.top()]
// Access index:  stk.top()
// Compute width: i - stk.top() - 1
```

### Pitfall 5: Stack Overflow from Deep Recursion

```cpp
// This overflows for large inputs — each recursive call uses stack space
int fibonacci(int n) {
    if (n <= 1) return n;
    return fibonacci(n-1) + fibonacci(n-2);  // two recursive calls!
}
// fibonacci(50000) → stack overflow (depth ~50000, each frame ~32 bytes = 1.6 MB)

// Fix 1: iterative
int fibonacci_iter(int n) {
    if (n <= 1) return n;
    int a = 0, b = 1;
    for (int i = 2; i <= n; i++) { int c = a + b; a = b; b = c; }
    return b;
}

// Fix 2: explicit stack (simulate call stack on the heap)
// The heap is typically much larger than the stack (GBs vs MBs)
```

### Pitfall 6: Off-by-One in the Monotonic Stack Width Formula

```cpp
// Width when popping index `mid` with right boundary `i` and
// remaining stack top `left_idx`:

// WRONG:
int width = i - left_idx;        // off by one — includes left_idx itself

// CORRECT:
int width = stk.empty() ? i : i - stk.top() - 1;
//                              ↑           ↑
//                   right = i-1     left = stk.top()+1
//           width = (i-1) - (stk.top()+1) + 1 = i - stk.top() - 1
```

### Pitfall 7: Losing Data When stack::pop() Is Called in a Loop Without Checking empty()

```cpp
// Classic bug: forgot to check empty() inside while loop condition
while (!stk.empty() && condition(stk.top())) {
    // process stk.top()
    stk.pop();
    // BUG if condition() calls stk.top() again after stk may have become empty
    // through the pop above — but the while condition was true BEFORE the pop
}

// Safe pattern: re-check empty() inside the loop body if you pop multiple times
while (!stk.empty()) {
    if (!condition(stk.top())) break;
    process(stk.top());
    stk.pop();
    // safe: next iteration checks empty() again
}
```

---

## 12. std::stack

```cpp
#include <stack>
#include <vector>
#include <deque>   // default underlying container

// std::stack is a container adaptor — it wraps another container and
// restricts it to LIFO access only.

// Default: uses std::deque as the underlying container
stack<int> s1;

// Using std::vector — faster for most use cases (better cache performance)
stack<int, vector<int>> s2;

// Using std::list — guaranteed O(1) push with no resizing
stack<int, list<int>> s3;

// For interviews: always use stack<int, vector<int>> or just vector<int> directly
// (vector has push_back/pop_back/back which are push/pop/top for a stack)

// Complete API:
s1.push(10);        // add to top
s1.emplace(10);     // construct in place at top (prefer over push for complex types)
s1.pop();           // remove top (returns void!)
s1.top();           // peek at top (reference)
s1.empty();         // true if empty
s1.size();          // number of elements
s1.swap(s2);        // O(1) swap

// What std::stack does NOT have:
// - iteration (no begin/end)
// - indexed access (no operator[])
// - search (no find)
// - printing (no built-in)
// These are deliberate omissions — they would violate the LIFO contract.

// In competitive programming, just use vector<int> directly:
vector<int> stk;
stk.push_back(val);           // push
stk.pop_back();               // pop
stk.back();                   // top
stk.empty();                  // empty
// Advantage: you can also iterate stk, print it, index into it for debugging.
```

---

## 13. Comparison: Stack vs Queue vs Deque

| Feature | Stack | Queue | Deque |
|---|---|---|---|
| Access policy | LIFO — last in, first out | FIFO — first in, first out | Both ends — push/pop front and back |
| Insert end | O(1) push to top | O(1) enqueue at back | O(1) at either end |
| Remove end | O(1) pop from top | O(1) dequeue from front | O(1) at either end |
| Peek | O(1) top | O(1) front | O(1) front and back |
| Reverse order | Natural (LIFO gives reverse) | Requires extra structure | Possible with bidirectional access |
| DFS / backtracking | Natural fit | Not suitable | Not typically used |
| BFS | Not suitable | Natural fit | Can simulate both |
| Undo/redo | Natural fit | Not suitable | Not typically used |
| Sliding window max | Not suitable | Not suitable | Natural fit (monotonic deque) |
| Implementation | vector or linked list | deque or circular array | deque (std::deque) |
| std:: type | `std::stack` | `std::queue` | `std::deque` |

**The LIFO vs FIFO distinction in algorithms:**

```
DFS uses a stack:
  Push start node.
  Pop a node → process it → push its unvisited neighbours.
  The most recently discovered node is explored first → depth-first.

BFS uses a queue:
  Enqueue start node.
  Dequeue a node → process it → enqueue its unvisited neighbours.
  The earliest discovered node is explored first → breadth-first.

Same code structure. Different container. Completely different traversal order.
This is the single most important difference between stack and queue.
```

---

## 14. Self-Test Questions

1. **What does LIFO mean? Give three real-world examples of LIFO ordering that are NOT programming-related.**

2. **Why is `std::stack::pop()` void instead of returning the top element? What exception-safety issue does this prevent?**

3. **A colleague proposes using a linked-list-backed stack instead of a vector-backed stack for a system that processes 10 million elements. What would you measure to decide? What does each implementation optimise for?**

4. **Explain the call stack with a concrete example. What is in a stack frame? What causes a stack overflow?**

5. **Implement `getMin()` in O(1) from scratch without looking at the notes. What data structure do you pair with the main stack and why?**

6. **Trace the monotonic stack algorithm for Next Greater Element on input `[4, 5, 2, 10, 8]`. Show the stack state after every element. What is the output?**

7. **In the Largest Rectangle in Histogram problem, why is the width formula `i - stk.top() - 1` and not `i - stk.top()`? Draw the boundary positions to explain.**

8. **Convert this recursive function to iterative using an explicit stack:**
   ```cpp
   void dfs(TreeNode* node) {
       if (!node) return;
       process(node->val);
       dfs(node->left);
       dfs(node->right);
   }
   ```

9. **A stack of characters is used to reverse a string. Push each character, then pop all. Trace the operations for "hello". What is the space complexity?**

10. **Why does a monotonic stack process each element in O(1) amortised even though there's a `while` loop inside a `for` loop? Prove the total work is O(n).**

---

## Quick Reference Card

```
Stack — LIFO access. The most recent unresolved thing is always on top.

Core ADT:          push(x)  pop()  top()  empty()  — all O(1)
Best backing:      std::vector (push_back / pop_back / back)
In competitive:    Use vector<int> stk directly

Key complexities:
  push: O(1) amortized (vector) or O(1) guaranteed (linked list)
  pop:  O(1)
  top:  O(1)
  All others: O(n) or not applicable (stacks are not for searching)

Interview patterns (know all 4):
  1. Bracket matching:   push openers; pop and match on closers
  2. Monotonic stack:    maintain sorted stack; pop gives NGE/NSE
  3. Expression eval:    two stacks (values + operators)
  4. Explicit DFS:       replace call stack with heap-allocated stack

Monotonic stack:
  - Decreasing (stack top is max):  pop when current > top → find next greater
  - Increasing (stack top is min):  pop when current < top → find next smaller
  - Each element pushed once, popped once → O(n) total
  - Push INDICES not values (you need position to compute width/distance)

Critical rules:
  → Always check empty() before top() and pop()
  → std::stack::pop() returns void — peek first, then pop
  → For debugging, use vector<T> directly (supports iteration and printing)
  → Convert deep recursion to iterative with explicit stack to avoid overflow
  → In monotonic stack: push indices, not values

The call stack insight:
  Recursion IS a stack. Every recursive call pushes a frame.
  Converting recursion to iteration = making the implicit stack explicit.
  The heap is GBs; the call stack is MBs. Explicit stacks handle deeper problems.
```

---

*Previous: [06 — XOR Linked List](./06_xor_linked_list.md)*
*Next: [08 — Monotonic Stack](./08_monotonic_stack.md)*
*See also: [Queue](./09_queue.md) | [Deque](./10_deque.md) | [DFS algorithms](./algorithms/dfs.md)*

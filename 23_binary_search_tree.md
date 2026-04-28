# Binary Search Tree (BST)

> **Curriculum position:** Trees → #2
> **Interview weight:** ★★★★★ Critical — BST operations, validation, and conversion problems appear in every top-tier interview loop; BST is the conceptual foundation for AVL, Red-Black, B-Tree, and segment trees
> **Difficulty:** Intermediate — the property is simple; the deletion algorithm and degeneration analysis require depth
> **Prerequisite:** [22 — Binary Tree](./22_binary_tree.md)

---

## Table of Contents

1. [Intuition First — The Ordering That Enables Search](#1-intuition-first)
2. [The BST Property — Formal Definition](#2-the-bst-property)
3. [Internal Working — How the Property Enables O(log n)](#3-internal-working)
4. [Core Operations — Search, Insert, Delete](#4-core-operations)
5. [The Deletion Problem — The Three Cases](#5-the-deletion-problem)
6. [Inorder Traversal — The BST's Sorted Output](#6-inorder-traversal)
7. [Degeneration — When BST Becomes a Linked List](#7-degeneration)
8. [Time & Space Complexity](#8-time--space-complexity)
9. [Complete C++ Implementation](#9-complete-c-implementation)
10. [Core Operations — Visualised](#10-core-operations--visualised)
11. [Common Patterns & Techniques](#11-common-patterns--techniques)
12. [Interview Problems](#12-interview-problems)
13. [Real-World Uses](#13-real-world-uses)
14. [Edge Cases & Pitfalls](#14-edge-cases--pitfalls)
15. [BST vs Hash Map vs Sorted Array](#15-comparison)
16. [Self-Test Questions](#16-self-test-questions)

---

## 1. Intuition First

A binary tree is a structure for holding data hierarchically, but it imposes no constraint on *where* data is placed. Finding any value requires scanning up to all n nodes. The binary tree gives you hierarchy; it does not give you search efficiency.

The **Binary Search Tree** adds one rule that transforms everything: **for every node, all values in its left subtree are strictly less than the node's value, and all values in its right subtree are strictly greater.**

This single rule turns every node into a decision point. At any node with value `v`:
- The target is `v`? → Found.
- The target is less than `v`? → The answer is in the left subtree (if it exists at all).
- The target is greater than `v`? → The answer is in the right subtree.

At each step, half the remaining tree is eliminated. This is binary search, but in tree form. The search that requires O(n) comparisons in an unsorted list takes O(log n) comparisons in a balanced BST — the same improvement that binary search achieves over linear search, but now supporting dynamic insertion and deletion.

Think of it as a sorted phonebook where each page tells you: "Everything in the first half is to my left; everything in the second half is to my right." You always know which direction to go. You never backtrack. You never scan sideways. You just follow the ordering downward.

The BST is one of the most important data structures in computer science because it solves the fundamental tension between:
- **Arrays**: O(1) access but O(n) insertion/deletion
- **Linked lists**: O(1) insertion/deletion but O(n) search
- **BSTs**: O(log n) for ALL three operations when balanced

Every ordered data structure you will study — AVL tree, red-black tree, B-tree, skip list, `std::map`, `std::set` — is either a BST or a structure that achieves BST-like guarantees through different means.

---

## 2. The BST Property — Formal Definition

### The Property

For every node `N` in a BST:
1. All values in `N`'s **left subtree** are **strictly less than** `N->val`
2. All values in `N`'s **right subtree** are **strictly greater than** `N->val`
3. Both the left and right subtrees are themselves valid BSTs (recursive)

```
Valid BST:                  Invalid BST (value 3 violates property):
       5                           5
      / \                         / \
     3   7                       3   7
    / \ / \                     / \ / \
   2  4 6  8                   2  4 6  3  ← 3 is in the right subtree of 5
                                            but 3 < 5 → VIOLATION
```

### Strict vs Non-Strict Inequality

**Strict** (`<` and `>`): no duplicate values allowed. This is the most common definition in interviews and most implementations.

**Non-strict** (sometimes `≤` or `≥`): duplicates allowed by placing them consistently (all duplicates go left, or all go right, or use a count per node). This complicates deletion and validation; most interview problems assume strict inequality.

### The Whole-Subtree Constraint

The property is about the **entire subtree**, not just immediate children:

```
This looks valid at first glance but is NOT a valid BST:
        10
       /  \
      5    15
     / \
    2   12   ← 12 is in the LEFT subtree of 10, but 12 > 10 → VIOLATION!

A common bug: only checking the parent-child relationship, not the full subtree constraint.
```

This distinction is why BST validation is a classic interview problem (Problem 1 below).

---

## 3. Internal Working — How the Property Enables O(log n)

### The Search Invariant

At every step of a BST search for value `target` starting at node `curr`:
- `target` is in the subtree rooted at `curr` if and only if `target` exists in the tree.

This invariant is maintained by the routing rule:
- `target < curr->val` → recurse left (discard entire right subtree)
- `target > curr->val` → recurse right (discard entire left subtree)
- `target == curr->val` → found

### Why This Eliminates Half the Remaining Tree

```
BST of height h, perfectly balanced:
  n = 2^(h+1) - 1 nodes total

At root: compare target with root. One comparison eliminates roughly n/2 nodes.
At level 1: one comparison eliminates roughly n/4 nodes.
At level k: one comparison eliminates roughly n/2^(k+1) nodes.
After h+1 comparisons: 1 node remains (or we reach null → not found).
h+1 = log₂(n+1) comparisons.

This is the geometric series argument: binary search works because
each step halves the remaining search space.
```

### The Relationship to Binary Search on Arrays

```
Binary search on sorted array:
  arr = [2, 3, 4, 5, 6, 7, 8], target = 4
  Compare arr[3]=5: too big → search left half
  Compare arr[1]=3: too small → search right quarter
  Compare arr[2]=4: found!

BST search on equivalent tree:
       5
      / \
     3   7
    / \ / \
   2  4 6  8

  Compare 5: too big → go left
  Compare 3: too small → go right
  Compare 4: found!

Same decisions, same comparisons — just navigating a tree instead of index arithmetic.
The BST externalises the structure of binary search into an explicit data structure.
```

---

## 4. Core Operations — Search, Insert, Delete

### Search

```
search(node, target):
  if node is null: return null  // not found
  if target == node->val: return node  // found
  if target < node->val: return search(node->left, target)
  else: return search(node->right, target)
```

### Insert

New values are always inserted as **leaf nodes**. The path to the insertion point is determined by the BST property:

```
insert(node, val):
  if node is null: return new Node(val)  // insertion point found
  if val < node->val: node->left  = insert(node->left,  val)
  if val > node->val: node->right = insert(node->right, val)
  // val == node->val: duplicate, do nothing (or handle as needed)
  return node
```

### Minimum and Maximum

```
// Minimum: leftmost node (go left until null)
TreeNode* findMin(TreeNode* root) {
    while (root->left) root = root->left;
    return root;
}

// Maximum: rightmost node (go right until null)
TreeNode* findMax(TreeNode* root) {
    while (root->right) root = root->right;
    return root;
}
```

### Successor and Predecessor

The **inorder successor** of node `N` is the node with the smallest value greater than `N->val`:
1. If `N` has a right child: successor = minimum of right subtree
2. If `N` has no right child: successor = the lowest ancestor for which `N` is in the left subtree

The **inorder predecessor** is symmetric.

---

## 5. The Deletion Problem — The Three Cases

Deletion is the most complex BST operation. When removing a node, the BST property must be maintained for all remaining nodes. There are three cases based on the node's children.

### Case 1: Delete a Leaf Node (0 children)

Simply remove the node. No restructuring needed.

```
Delete 4:
Before:      After:
   5            5
  / \          / \
 3   7        3   7
/ \ / \      /   / \
2 4 6  8    2   6   8
```

### Case 2: Delete a Node with One Child

Replace the node with its only child. The child's entire subtree takes over.

```
Delete 3 (has only left child 2):
Before:      After:
   5            5
  / \          / \
 3   7        2   7
/   / \          / \
2  6   8        6   8
```

### Case 3: Delete a Node with Two Children — The Hard Case

Cannot simply remove the node — two subtrees would be disconnected. The solution:

**Find the inorder successor** (smallest value in the right subtree). Copy its value to the node being "deleted." Then delete the inorder successor from the right subtree (the successor has at most one child — a right child — so it falls into Case 1 or Case 2).

**Alternatively**, use the inorder **predecessor** (largest value in left subtree). Both approaches are valid.

```
Delete 5 (root, has two children):
Before:         After:
    5               6       ← 6 is the inorder successor (min of right subtree {6,7,8})
   / \             / \      Copy 6's value to root, delete 6 from right subtree.
  3   7           3   7
 / \ / \         / \   \
2  4 6  8       2   4   8

Why inorder successor works:
  - 6 > all values in left subtree (3,2,4) ✓ — so it can be the root
  - 6 < 7 and 6 < 8 ✓ — BST property maintained in right subtree
```

### The Deletion Algorithm

```cpp
TreeNode* deleteNode(TreeNode* root, int val) {
    if (!root) return nullptr;

    if (val < root->val) {
        root->left = deleteNode(root->left, val);
    } else if (val > root->val) {
        root->right = deleteNode(root->right, val);
    } else {
        // Found the node to delete
        if (!root->left) {
            TreeNode* tmp = root->right;
            delete root;
            return tmp;                    // Case 1 and Case 2 (left)
        }
        if (!root->right) {
            TreeNode* tmp = root->left;
            delete root;
            return tmp;                    // Case 2 (right)
        }
        // Case 3: two children
        TreeNode* successor = root->right;
        while (successor->left) successor = successor->left;   // find min of right subtree
        root->val   = successor->val;                           // copy successor's value
        root->right = deleteNode(root->right, successor->val); // delete the successor
    }
    return root;
}
```

---

## 6. Inorder Traversal — The BST's Sorted Output

The most important property of BST traversal:

> **The inorder traversal of a BST visits all nodes in sorted (ascending) order.**

This is not a coincidence — it is a direct consequence of the BST property. When you visit left subtree (all values less than current), then current, then right subtree (all values greater than current), you produce a sorted sequence.

```
BST:        Inorder traversal → [2, 3, 4, 5, 6, 7, 8]
    5
   / \
  3   7
 / \ / \
2  4 6  8

Sorted ascending. ✓
```

### Consequences

1. **Checking if an array is a BST inorder**: if the inorder traversal is strictly increasing, the tree is a valid BST.
2. **Kth smallest element**: the kth element visited during inorder is the kth smallest.
3. **Range queries**: inorder traversal from `low` to `high` gives all values in `[low, high]` in order.
4. **BST to sorted array**: inorder traversal.
5. **Sorted array to BST**: construct from sorted sequence.

---

## 7. Degeneration — When BST Becomes a Linked List

### The Degenerate Case

If keys are inserted in sorted order into a BST, every new key is larger than all existing keys and goes to the rightmost position. The tree grows as a straight right-leaning chain:

```
Insert 1, 2, 3, 4, 5 in order:
1
 \
  2
   \
    3
     \
      4
       \
        5

Height = n-1 = 4.
Search, insert, delete: O(n) instead of O(log n).
This is a linked list with extra pointer overhead.
```

### Why This Matters

```
Best case (perfectly balanced):    height = O(log n)
                                   search = O(log n)

Worst case (sorted insertion):     height = O(n)
                                   search = O(n)

Expected case (random insertion):  height = O(log n) with high probability
                                   (formal result: E[height] = 2.5 ln n for random permutations)
```

**This is why balanced BSTs (AVL, Red-Black) exist.** They add rebalancing operations (rotations) to guarantee O(log n) height regardless of insertion order. The plain BST is efficient in the average case but fragile in the worst case.

### Real-World Implication

If you cannot control the insertion order of keys (e.g., user-supplied data), a plain BST is risky. Use `std::map` or `std::set` (both use red-black trees in practice) for guaranteed O(log n) performance.

---

## 8. Time & Space Complexity

### Operation Complexities

| Operation | Average (balanced) | Worst (degenerate) | Notes |
|---|---|---|---|
| `search(val)` | **O(log n)** | O(n) | Compare and halve |
| `insert(val)` | **O(log n)** | O(n) | Search + leaf insert |
| `delete(val)` | **O(log n)** | O(n) | Search + case handling |
| `findMin()` | **O(log n)** | O(n) | Follow left pointers |
| `findMax()` | **O(log n)** | O(n) | Follow right pointers |
| `successor(val)` | **O(log n)** | O(n) | Right subtree min or ancestor |
| `predecessor(val)` | **O(log n)** | O(n) | Left subtree max or ancestor |
| Inorder traversal | **O(n)** | O(n) | Must visit all nodes |
| Range query [a,b] | **O(log n + k)** | O(n) | k = number of results |
| Build from n sorted | O(n log n) | O(n²) | n inserts |
| Build from sorted array | **O(n)** | O(n) | Special construction |
| Validate BST | O(n) | O(n) | Must check all nodes |

### Space Complexity

| Component | Space | Notes |
|---|---|---|
| n nodes | O(n) | One node per stored value |
| Per-node overhead | 2 pointers = 16 bytes (64-bit) | Plus the value |
| Recursive call stack (balanced) | O(log n) | Height of tree |
| Recursive call stack (degenerate) | O(n) | Stack overflow risk |
| Iterative operations | O(1) extra | No recursion needed |

### Why O(log n + k) for Range Queries?

```
Range query [a, b]: find all values in [a, b].

Step 1: Navigate to the position of 'a' in the BST: O(log n)
Step 2: Inorder traverse from 'a' until value > b: O(k) where k = count of results

Total: O(log n + k)
Compare to sorted array: O(log n + k) with binary search + linear scan
The BST achieves the same range query complexity as a sorted array,
while also supporting O(log n) insertions (array: O(n) for insertion).
```

---

## 9. Complete C++ Implementation

```cpp
#include <iostream>
#include <vector>
#include <stack>
#include <queue>
#include <climits>
#include <functional>
#include <optional>
using namespace std;

struct TreeNode {
    int       val;
    TreeNode* left;
    TreeNode* right;
    explicit TreeNode(int v) : val(v), left(nullptr), right(nullptr) {}
};

class BST {
private:
    TreeNode* root_ = nullptr;

    // ── Internal recursive helpers ────────────────────────────────────────────

    TreeNode* insert_(TreeNode* node, int val) {
        if (!node) return new TreeNode(val);
        if      (val < node->val) node->left  = insert_(node->left,  val);
        else if (val > node->val) node->right = insert_(node->right, val);
        // Equal: duplicate — do nothing (strict BST)
        return node;
    }

    TreeNode* search_(TreeNode* node, int val) const {
        if (!node || node->val == val) return node;
        if (val < node->val) return search_(node->left,  val);
        else                  return search_(node->right, val);
    }

    TreeNode* findMin_(TreeNode* node) const {
        if (!node) return nullptr;
        while (node->left) node = node->left;
        return node;
    }

    TreeNode* findMax_(TreeNode* node) const {
        if (!node) return nullptr;
        while (node->right) node = node->right;
        return node;
    }

    // Deletes node with given val from subtree rooted at node.
    // Returns the new root of the subtree.
    TreeNode* delete_(TreeNode* node, int val) {
        if (!node) return nullptr;

        if (val < node->val) {
            node->left  = delete_(node->left,  val);
        } else if (val > node->val) {
            node->right = delete_(node->right, val);
        } else {
            // Found the node to delete
            if (!node->left) {
                TreeNode* tmp = node->right;
                delete node;
                return tmp;                          // Case 1 or 2 (missing left)
            }
            if (!node->right) {
                TreeNode* tmp = node->left;
                delete node;
                return tmp;                          // Case 2 (missing right)
            }
            // Case 3: two children — replace with inorder successor
            TreeNode* succ = findMin_(node->right);  // smallest in right subtree
            node->val   = succ->val;                  // copy successor value
            node->right = delete_(node->right, succ->val);  // delete successor
        }
        return node;
    }

    void inorder_(TreeNode* node, vector<int>& result) const {
        if (!node) return;
        inorder_(node->left, result);
        result.push_back(node->val);
        inorder_(node->right, result);
    }

    void destroy_(TreeNode* node) {
        if (!node) return;
        destroy_(node->left);
        destroy_(node->right);
        delete node;
    }

    // Builds a balanced BST from sorted array [lo, hi]
    TreeNode* buildBalanced_(const vector<int>& sorted, int lo, int hi) {
        if (lo > hi) return nullptr;
        int mid = lo + (hi - lo) / 2;
        TreeNode* node = new TreeNode(sorted[mid]);
        node->left  = buildBalanced_(sorted, lo, mid - 1);
        node->right = buildBalanced_(sorted, mid + 1, hi);
        return node;
    }

    // Validate BST using min/max range
    bool isValid_(TreeNode* node, long long minVal, long long maxVal) const {
        if (!node) return true;
        if (node->val <= minVal || node->val >= maxVal) return false;
        return isValid_(node->left,  minVal, node->val)
            && isValid_(node->right, node->val, maxVal);
    }

    int height_(TreeNode* node) const {
        if (!node) return -1;
        return 1 + max(height_(node->left), height_(node->right));
    }

    // Collect all values in [lo, hi] in sorted order
    void rangeSearch_(TreeNode* node, int lo, int hi, vector<int>& result) const {
        if (!node) return;
        if (node->val > lo) rangeSearch_(node->left,  lo, hi, result);   // prune right
        if (node->val >= lo && node->val <= hi) result.push_back(node->val);
        if (node->val < hi) rangeSearch_(node->right, lo, hi, result);   // prune left
    }

public:
    // ── Constructors ──────────────────────────────────────────────────────────

    BST() = default;

    // Build balanced BST from sorted values
    explicit BST(const vector<int>& sorted) {
        root_ = buildBalanced_(sorted, 0, (int)sorted.size() - 1);
    }

    ~BST() { destroy_(root_); }

    BST(const BST&)            = delete;
    BST& operator=(const BST&) = delete;
    BST(BST&& other) noexcept : root_(other.root_) { other.root_ = nullptr; }

    // ── Core operations ───────────────────────────────────────────────────────

    // O(h) — insert value; ignores duplicates
    void insert(int val) { root_ = insert_(root_, val); }

    // O(h) — returns true if val exists
    bool contains(int val) const { return search_(root_, val) != nullptr; }

    // O(h) — remove val; does nothing if not present
    void remove(int val) { root_ = delete_(root_, val); }

    // O(h) — minimum value in BST
    optional<int> findMin() const {
        TreeNode* m = findMin_(root_);
        return m ? optional<int>(m->val) : nullopt;
    }

    // O(h) — maximum value in BST
    optional<int> findMax() const {
        TreeNode* m = findMax_(root_);
        return m ? optional<int>(m->val) : nullopt;
    }

    // O(h) — smallest value strictly greater than val
    optional<int> successor(int val) const {
        TreeNode* curr = root_;
        TreeNode* succ = nullptr;
        while (curr) {
            if (curr->val > val) {
                succ = curr;           // candidate: curr is greater than val
                curr = curr->left;     // try to find something smaller but still > val
            } else {
                curr = curr->right;    // curr <= val, successor must be to the right
            }
        }
        return succ ? optional<int>(succ->val) : nullopt;
    }

    // O(h) — largest value strictly less than val
    optional<int> predecessor(int val) const {
        TreeNode* curr = root_;
        TreeNode* pred = nullptr;
        while (curr) {
            if (curr->val < val) {
                pred = curr;           // candidate: curr is less than val
                curr = curr->right;    // try to find something larger but still < val
            } else {
                curr = curr->left;     // curr >= val, predecessor must be to the left
            }
        }
        return pred ? optional<int>(pred->val) : nullopt;
    }

    // ── Traversals ────────────────────────────────────────────────────────────

    // O(n) — returns all values in sorted ascending order
    vector<int> inorder() const {
        vector<int> result;
        inorder_(root_, result);
        return result;
    }

    // O(n) — iterative inorder (no recursion, safe for deep trees)
    vector<int> inorderIterative() const {
        vector<int> result;
        stack<TreeNode*> stk;
        TreeNode* curr = root_;
        while (curr || !stk.empty()) {
            while (curr) { stk.push(curr); curr = curr->left; }
            curr = stk.top(); stk.pop();
            result.push_back(curr->val);
            curr = curr->right;
        }
        return result;
    }

    // ── Queries ───────────────────────────────────────────────────────────────

    // O(n) — validate if tree satisfies BST property
    bool isValidBST() const {
        return isValid_(root_, (long long)INT_MIN - 1, (long long)INT_MAX + 1);
    }

    // O(h) — height of BST
    int height() const { return height_(root_); }

    // O(log n + k) — all values in [lo, hi], returned in sorted order
    vector<int> rangeSearch(int lo, int hi) const {
        vector<int> result;
        rangeSearch_(root_, lo, hi, result);
        return result;
    }

    // O(n log n) — check if given sorted array equals the BST's inorder
    bool matchesSorted(const vector<int>& sorted) const {
        return inorder() == sorted;
    }

    // O(h) — floor: largest value ≤ key
    optional<int> floor(int key) const {
        TreeNode* curr = root_;
        TreeNode* result = nullptr;
        while (curr) {
            if (curr->val == key) return key;
            if (curr->val < key) { result = curr; curr = curr->right; }
            else                  curr = curr->left;
        }
        return result ? optional<int>(result->val) : nullopt;
    }

    // O(h) — ceiling: smallest value ≥ key
    optional<int> ceil(int key) const {
        TreeNode* curr = root_;
        TreeNode* result = nullptr;
        while (curr) {
            if (curr->val == key) return key;
            if (curr->val > key) { result = curr; curr = curr->left; }
            else                  curr = curr->right;
        }
        return result ? optional<int>(result->val) : nullopt;
    }

    // ── Utilities ─────────────────────────────────────────────────────────────

    TreeNode* root() const { return root_; }

    void print() const {
        function<void(TreeNode*, string, bool)> pr = [&](TreeNode* n, string pre, bool isLeft) {
            if (!n) return;
            pr(n->right, pre + (isLeft ? "│   " : "    "), false);
            cout << pre << (isLeft ? "└── " : "┌── ") << n->val << "\n";
            pr(n->left, pre + (isLeft ? "    " : "│   "), true);
        };
        pr(root_, "", true);
    }
};
```

### Standalone BST Functions (LeetCode-style with TreeNode\*)

```cpp
// ── Iterative Search ──────────────────────────────────────────────────────────
TreeNode* searchBST(TreeNode* root, int val) {
    while (root) {
        if (val == root->val) return root;
        root = (val < root->val) ? root->left : root->right;
    }
    return nullptr;
}

// ── Iterative Insert ──────────────────────────────────────────────────────────
TreeNode* insertIntoBST(TreeNode* root, int val) {
    TreeNode* newNode = new TreeNode(val);
    if (!root) return newNode;

    TreeNode* curr = root, *parent = nullptr;
    while (curr) {
        parent = curr;
        curr = (val < curr->val) ? curr->left : curr->right;
    }
    if (val < parent->val) parent->left  = newNode;
    else                    parent->right = newNode;
    return root;
}

// ── Kth Smallest Element (inorder, k-counter) ─────────────────────────────────
int kthSmallest(TreeNode* root, int k) {
    // Iterative inorder with early exit when k-th element found
    stack<TreeNode*> stk;
    TreeNode* curr = root;
    int count = 0;

    while (curr || !stk.empty()) {
        while (curr) { stk.push(curr); curr = curr->left; }
        curr = stk.top(); stk.pop();
        if (++count == k) return curr->val;   // k-th element in sorted order
        curr = curr->right;
    }
    return -1;  // k > n
}

// ── LCA of BST (uses BST property for O(h) without hash map) ─────────────────
TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, TreeNode* q) {
    // Both p and q in left subtree
    if (p->val < root->val && q->val < root->val)
        return lowestCommonAncestor(root->left, p, q);
    // Both in right subtree
    if (p->val > root->val && q->val > root->val)
        return lowestCommonAncestor(root->right, p, q);
    // One in each subtree (or one IS the root) → root is LCA
    return root;
}

// ── Convert Sorted Array to Balanced BST ──────────────────────────────────────
// Always pick the middle element as root → guarantees height = O(log n)
TreeNode* sortedArrayToBST(vector<int>& nums, int lo, int hi) {
    if (lo > hi) return nullptr;
    int mid = lo + (hi - lo) / 2;
    TreeNode* node = new TreeNode(nums[mid]);
    node->left  = sortedArrayToBST(nums, lo, mid - 1);
    node->right = sortedArrayToBST(nums, mid + 1, hi);
    return node;
}

// ── Convert BST to Sorted Doubly Linked List (in-place) ──────────────────────
// Clever in-place transformation using the BST's existing pointers.
// After: each node's left = predecessor, right = successor.
TreeNode* bstToDoublyLinkedList(TreeNode* root) {
    if (!root) return nullptr;
    TreeNode* head = nullptr, *prev = nullptr;

    function<void(TreeNode*)> inorder = [&](TreeNode* node) {
        if (!node) return;
        inorder(node->left);
        if (prev) { prev->right = node; node->left = prev; }
        else       { head = node; }
        prev = node;
        inorder(node->right);
    };

    inorder(root);
    // Close the circle (optional — for circular DLL)
    // head->left = prev; prev->right = head;
    return head;
}
```

---

## 10. Core Operations — Visualised

### Search Path

```
BST:        5
           / \
          3   7
         / \ / \
        2  4 6  8

Search for 6:
  Compare 6 with 5: 6 > 5 → go right
  Compare 6 with 7: 6 < 7 → go left
  Compare 6 with 6: 6 = 6 → FOUND ✓
  Path: 3 nodes visited. For n=7 nodes: O(log 7) ≈ 2-3 comparisons.

Search for 9:
  Compare 9 with 5: 9 > 5 → go right
  Compare 9 with 7: 9 > 7 → go right
  Compare 9 with 8: 9 > 8 → go right
  null → NOT FOUND ✓
```

### Insert Sequence and Resulting Shape

```
Insert: 5, 3, 7, 2, 4, 6, 8 (balanced result)

Insert 5: root → [5]
Insert 3: 3 < 5 → left of 5
Insert 7: 7 > 5 → right of 5
Insert 2: 2 < 5 → left; 2 < 3 → left of 3
Insert 4: 4 < 5 → left; 4 > 3 → right of 3
Insert 6: 6 > 5 → right; 6 < 7 → left of 7
Insert 8: 8 > 5 → right; 8 > 7 → right of 7

Result:     5
           / \
          3   7
         / \ / \
        2  4 6  8    (height = 2, balanced) ✓

Insert: 1, 2, 3, 4, 5 (degenerate result)

Insert 1: root → [1]
Insert 2: 2 > 1 → right of 1
Insert 3: 3 > 1 → right; 3 > 2 → right of 2
...

Result: 1→2→3→4→5 (right-skewed, height = 4 = n-1) ✗
```

### Delete — All Three Cases

```
Original BST:
        8
       / \
      3   10
     / \    \
    1   6   14
       / \  /
      4  7 13

Case 1: Delete 7 (leaf). Simply remove.
        8
       / \
      3   10
     / \    \
    1   6   14
       /   /
      4   13

Case 2: Delete 10 (one child). Replace with 14.
        8
       / \
      3   14
     / \  /
    1   6 13
       /
      4

Case 3: Delete 3 (two children: 1 and 6).
  Successor = min of right subtree = min(6) = 4.
  Copy 4 to node: replace 3 with 4. Delete 4 from right subtree.
        8
       / \
      4   14
     / \  /
    1   6 13
       /
      (4 removed from here)
  
  Result:
        8
       / \
      4   14
     / \  /
    1   6 13
```

---

## 11. Common Patterns & Techniques

### Pattern 1: Using BST Property to Prune Search

Unlike general binary trees, BST algorithms can skip entire subtrees:

```cpp
// Find all values in range [lo, hi] — prune left/right based on BST property
void rangeSearch(TreeNode* root, int lo, int hi, vector<int>& result) {
    if (!root) return;
    // Prune: if root < lo, no need to search left subtree
    if (root->val > lo) rangeSearch(root->left, lo, hi, result);
    // Visit root if in range
    if (root->val >= lo && root->val <= hi) result.push_back(root->val);
    // Prune: if root > hi, no need to search right subtree
    if (root->val < hi) rangeSearch(root->right, lo, hi, result);
}
// Time: O(log n + k) where k = number of results in range.
// Without pruning: O(n).
```

### Pattern 2: Augmented BST

Standard BSTs can be augmented with extra information at each node to answer queries like "how many elements ≤ x?" in O(log n):

```cpp
struct AugmentedNode {
    int val, size;   // size = subtree node count
    AugmentedNode* left;
    AugmentedNode* right;
};

// With subtree sizes: kth smallest in O(log n) — no traversal needed
int kthSmallestAugmented(AugmentedNode* root, int k) {
    int leftSize = root->left ? root->left->size : 0;
    if (k <= leftSize)      return kthSmallestAugmented(root->left, k);
    if (k == leftSize + 1)  return root->val;
    return kthSmallestAugmented(root->right, k - leftSize - 1);
}
```

### Pattern 3: BST Validation with Bounds

The correct validation passes down valid ranges, not just parent values:

```cpp
// WRONG: only checks immediate parent-child relationships
bool isValidBST_wrong(TreeNode* root) {
    if (!root) return true;
    if (root->left && root->left->val >= root->val) return false;   // misses deep violations
    if (root->right && root->right->val <= root->val) return false;
    return isValidBST_wrong(root->left) && isValidBST_wrong(root->right);
}

// CORRECT: passes [min, max] bounds that must hold for entire subtree
bool isValidBST(TreeNode* root, long long minVal = LLONG_MIN, long long maxVal = LLONG_MAX) {
    if (!root) return true;
    if (root->val <= minVal || root->val >= maxVal) return false;
    return isValidBST(root->left,  minVal,    root->val)
        && isValidBST(root->right, root->val, maxVal);
}
```

### Pattern 4: Convert BST to/from Sorted Structures

```cpp
// BST → sorted array: inorder traversal, O(n)
// Sorted array → balanced BST: middle element as root, recurse, O(n)
// BST → doubly linked list: rewire left/right pointers during inorder
// Sorted list → BST: read elements left-to-right, advance pointer during construction
```

---

## 12. Interview Problems

### Problem 1: Validate Binary Search Tree

**Problem:** Given the root of a binary tree, determine if it is a valid BST. A valid BST is defined as: left subtree values are strictly less than the root; right subtree values are strictly greater; both subtrees are also valid BSTs.

**Why this tests depth:** The naive check (only comparing parent with immediate children) fails on cases like node 3 appearing in the right subtree of 5 in a deep subtree. The correct solution threads valid ranges down the tree.

**Thought process:**
> "At every node, the valid range is (min_allowed, max_allowed). Starting at the root: (-∞, +∞). Going left narrows the max: (min, root->val). Going right narrows the min: (root->val, max). Any node outside its range means the BST property is violated."

```cpp
// Time: O(n) | Space: O(h)
bool isValidBST(TreeNode* root, long long minVal = LLONG_MIN, long long maxVal = LLONG_MAX) {
    if (!root) return true;
    // Node must be strictly within (minVal, maxVal)
    if (root->val <= minVal || root->val >= maxVal) return false;
    // Left subtree: max is tightened to root->val
    // Right subtree: min is tightened to root->val
    return isValidBST(root->left,  minVal,    (long long)root->val)
        && isValidBST(root->right, (long long)root->val, maxVal);
}

// Alternative: inorder traversal — valid BST iff inorder is strictly increasing
bool isValidBSTInorder(TreeNode* root) {
    long long prev = LLONG_MIN;
    function<bool(TreeNode*)> inorder = [&](TreeNode* node) -> bool {
        if (!node) return true;
        if (!inorder(node->left)) return false;
        if (node->val <= prev) return false;   // not strictly increasing
        prev = node->val;
        return inorder(node->right);
    };
    return inorder(root);
}

/*
Counter-example that breaks naive parent-child check:
    5
   / \
  1   6
     / \
    3   7   ← 3 < 5 but 3 is in RIGHT subtree of 5 → INVALID!

With bounds:
  isValidBST(5, -∞, +∞):
    isValidBST(1, -∞, 5):   1 > -∞ and 1 < 5 ✓. Leaves: ✓.
    isValidBST(6, 5, +∞):
      isValidBST(3, 5, 6):  3 <= 5 → INVALID ✗ ← caught by bounds!
*/
```

**Edge cases:**
- Single node: always valid (no constraint to check)
- Tree with only left or only right children: valid as long as ordering is maintained
- INT_MIN or INT_MAX values: use `long long` bounds to handle boundary nodes
- Duplicates: equal values violate strict BST property → return false

---

### Problem 2: Kth Smallest Element in a BST

**Problem:** Given the root of a BST and an integer `k`, return the `k`th smallest value (1-indexed) among all nodes.

**Example:**
```
BST:    3
       / \
      1   4
       \
        2
k=1 → 1, k=2 → 2, k=3 → 3, k=4 → 4
```

**Thought process:**
> "Inorder traversal of a BST gives sorted order. The k-th element visited in inorder is the k-th smallest. Key optimisation: stop as soon as we've counted k nodes — don't traverse the entire tree."

```cpp
// Time: O(h + k) with early exit | Space: O(h)
int kthSmallest(TreeNode* root, int k) {
    // Iterative inorder with early exit — avoids O(n) traversal
    stack<TreeNode*> stk;
    TreeNode* curr = root;
    int count = 0;

    while (curr || !stk.empty()) {
        while (curr) { stk.push(curr); curr = curr->left; }
        curr = stk.top(); stk.pop();
        if (++count == k) return curr->val;   // EARLY EXIT: found k-th
        curr = curr->right;
    }
    return -1;  // k > number of nodes
}

/*
Trace on BST above, k=2:
  Go left to 1 (push 3, push 1). Pop 1. count=1. 1 ≠ 2. Go right to 2.
  Push 2. Go left (null). Pop 2. count=2. 2 = k=2. Return 2. ✓

Trace k=3:
  Go left to 1 (push 3, push 1). Pop 1. count=1. Go right to 2. Push 2.
  Pop 2. count=2. Go right (null). Pop 3. count=3. Return 3. ✓
*/
```

**Follow-up: BST with frequent inserts and kth-smallest queries.** If the tree is frequently modified, augment each node with a `subtree_size` field. Then k-th smallest can be answered in O(log n) without traversal:

```cpp
// With augmented subtree sizes: O(log n) kth smallest
int kthSmallestAugmented(AugmentedNode* root, int k) {
    while (root) {
        int leftSize = root->left ? root->left->size : 0;
        if      (k == leftSize + 1) return root->val;
        else if (k <= leftSize)     { root = root->left; }
        else                        { k -= leftSize + 1; root = root->right; }
    }
    return -1;
}
```

---

### Problem 3: Recover BST — Two Swapped Nodes

**Problem:** Two nodes of a BST were swapped by mistake. Recover the tree without changing its structure (in-place, O(1) space preferred).

**Example:**
```
Correct BST:  1,2,3,4,5,6  inorder
Corrupted:    1,6,3,4,5,2  ← 2 and 6 swapped

In the inorder of the corrupted tree:
  At position 1-2: 1 < 6 ✓
  At position 2-3: 6 > 3 ← FIRST violation (prev=6, curr=3)
  ...
  At position 5-6: 5 > 2 ← SECOND violation (prev=5, curr=2)

The first node of the first violation (prev=6) and
the second node of the second violation (curr=2) are the swapped nodes.
```

**Thought process:**
> "Inorder traversal gives sorted order. Two swapped nodes create either one violation (adjacent swap) or two violations (non-adjacent swap) in the inorder sequence. Track `prev` during inorder. At each violation, record the first node (prev) at the first violation and the second node (curr) at the second violation. Swap their values."

```cpp
// Time: O(n) | Space: O(h) for recursion (O(1) with Morris traversal)
void recoverTree(TreeNode* root) {
    TreeNode* first  = nullptr;   // first swapped node
    TreeNode* second = nullptr;   // second swapped node
    TreeNode* prev   = nullptr;   // previous node in inorder

    function<void(TreeNode*)> inorder = [&](TreeNode* node) {
        if (!node) return;
        inorder(node->left);

        // Detect violation: prev should be LESS than node
        if (prev && prev->val > node->val) {
            if (!first) first = prev;    // first violation: take the LARGER (prev)
            second = node;               // always update second to the SMALLER (node)
            // Don't return — need to keep traversing for the second violation
        }
        prev = node;

        inorder(node->right);
    };

    inorder(root);
    swap(first->val, second->val);   // fix the swap
}

/*
Inorder of corrupted BST: 1, 6, 3, 4, 5, 2

Inorder traversal:
  prev=null, node=1.  prev=1.
  prev=1,    node=6.  No violation (1 < 6). prev=6.
  prev=6,    node=3.  VIOLATION (6 > 3). first=6, second=3. prev=3.
  prev=3,    node=4.  No violation. prev=4.
  prev=4,    node=5.  No violation. prev=5.
  prev=5,    node=2.  VIOLATION (5 > 2). first still 6, second=2. prev=2.

Swap first(6) and second(2): tree now has values in correct positions.
Inorder now: 1, 2, 3, 4, 5, 6 ✓

Why always update second but only set first once?
  Adjacent swap: only ONE violation. first=prev, second=curr.
  Non-adjacent swap: TWO violations. first set at first violation, second at second.
  The pattern captures both cases: first is always the larger-of-the-swap, second the smaller.
*/
```

**O(1) space version using Morris traversal:**

```cpp
void recoverTreeMorris(TreeNode* root) {
    TreeNode *first = nullptr, *second = nullptr, *prev = nullptr;
    TreeNode *curr = root, *pred = nullptr;

    while (curr) {
        if (!curr->left) {
            // Process curr
            if (prev && prev->val > curr->val) {
                if (!first) first = prev;
                second = curr;
            }
            prev = curr;
            curr = curr->right;
        } else {
            // Find inorder predecessor
            pred = curr->left;
            while (pred->right && pred->right != curr) pred = pred->right;

            if (!pred->right) {
                pred->right = curr;   // create thread
                curr = curr->left;
            } else {
                pred->right = nullptr;  // remove thread
                // Process curr
                if (prev && prev->val > curr->val) {
                    if (!first) first = prev;
                    second = curr;
                }
                prev = curr;
                curr = curr->right;
            }
        }
    }
    swap(first->val, second->val);
}
// True O(1) space — no call stack, no explicit stack.
```

---

## 13. Real-World Uses

| Domain | Use | BST Detail |
|---|---|---|
| **C++ STL** | `std::map`, `std::set`, `std::multimap`, `std::multiset` | Red-black tree (a self-balancing BST) |
| **Java** | `TreeMap`, `TreeSet` | Red-black tree; O(log n) guaranteed |
| **Databases** | B-Tree indexes | Generalised BST on disk; O(log n) range queries |
| **File systems** | Extent trees (Btrfs, ext4) | BST-based tracking of disk block ranges |
| **Compilers** | Symbol tables (sorted) | `std::map` for identifier → attribute lookup |
| **Networking** | IP routing tables | BST on prefix → O(log n) longest prefix match |
| **Game development** | Spatial queries | BSP trees for visibility; BST for score leaderboards |
| **Operating systems** | Virtual memory management | Red-black trees for `vm_area_struct` in Linux kernel |
| **Text editors** | Piece table / rope structure | BST for character sequence management |
| **Statistics** | Order statistics tree | O(log n) rank, quantile, and percentile queries |

### Linux Kernel's Red-Black Tree — The Real-World BST

The Linux kernel uses red-black trees (a type of self-balancing BST) pervasively. The most notable use is in the **Completely Fair Scheduler (CFS)**:

```c
// From linux/include/linux/sched.h (simplified):
struct sched_entity {
    u64 vruntime;       // "virtual runtime" — the sorting key
    struct rb_node run_node;  // embedded red-black tree node
    // ...
};

// The run queue: a red-black tree sorted by vruntime.
// Leftmost node = process that has run the LEAST → scheduled next.
// O(log n) insert when a process becomes runnable.
// O(log n) deletion when it's scheduled or blocked.
// O(1) find-min (leftmost pointer cached) → O(1) to pick next process.
```

This is BST + min-augmentation in production, running on every Linux system. Every time you switch between applications, a BST lookup and rotation happens to pick the next process.

---

## 14. Edge Cases & Pitfalls

### Pitfall 1: Shallow Validation (Only Parent-Child Check)

```cpp
// WRONG: only checks each node against its immediate children
bool isValidBST_shallow(TreeNode* root) {
    if (!root) return true;
    if (root->left  && root->left->val  >= root->val) return false;
    if (root->right && root->right->val <= root->val) return false;
    return isValidBST_shallow(root->left) && isValidBST_shallow(root->right);
}
// FAILS for:
//     5
//    / \
//   1   4
//      / \
//     3   6   ← 3 is in right subtree of 5, but 3 < 5 → invalid
// shallow check says: 4 < 5 ✓, 3 < 4 ✓ → incorrectly returns true!

// CORRECT: use range bounds (see implementation above).
```

### Pitfall 2: Integer Overflow in Bounds Checking

```cpp
// If tree contains INT_MIN or INT_MAX values:
bool isValidBST_overflow(TreeNode* root, int minVal = INT_MIN, int maxVal = INT_MAX) {
    if (!root) return true;
    if (root->val <= minVal || root->val >= maxVal) return false;  // WRONG for INT_MIN nodes!
    // If root->val == INT_MIN: valid for left subtree, but <= INT_MIN is false.
    // But when called with minVal = INT_MIN: INT_MIN <= INT_MIN is TRUE → false positive rejection!
}

// CORRECT: use long long bounds
bool isValidBST(TreeNode* root, long long minVal = LLONG_MIN, long long maxVal = LLONG_MAX) {
    if (!root) return true;
    if ((long long)root->val <= minVal || (long long)root->val >= maxVal) return false;
    return isValidBST(root->left,  minVal, (long long)root->val)
        && isValidBST(root->right, (long long)root->val, maxVal);
}
```

### Pitfall 3: Delete with Single Child — Returning Wrong Pointer

```cpp
// WRONG: forgetting to return the remaining child after deletion
TreeNode* deleteNode_wrong(TreeNode* root, int val) {
    if (!root) return nullptr;
    if (val < root->val) { root->left = deleteNode_wrong(root->left, val); return root; }
    if (val > root->val) { root->right = deleteNode_wrong(root->right, val); return root; }
    // Found node to delete
    if (!root->left && !root->right) { delete root; return nullptr; }
    if (!root->left)  { delete root; return nullptr; }  // WRONG: should return root->right!
    if (!root->right) { delete root; return nullptr; }  // WRONG: should return root->left!
    // ...
}

// CORRECT: always return the surviving child
if (!root->left)  { TreeNode* tmp = root->right; delete root; return tmp; }
if (!root->right) { TreeNode* tmp = root->left;  delete root; return tmp; }
```

### Pitfall 4: Not Updating Parent on Iterative Insert

```cpp
// WRONG: forgetting to actually attach the new node to the parent
void insertIterative_wrong(TreeNode*& root, int val) {
    TreeNode* newNode = new TreeNode(val);
    if (!root) { root = newNode; return; }
    TreeNode* curr = root;
    while (curr) {
        if (val < curr->val) curr = curr->left;
        else curr = curr->right;
    }
    // curr is now null — but we lost the parent! newNode is never attached.
}

// CORRECT: track parent separately
void insertIterative(TreeNode*& root, int val) {
    TreeNode* newNode = new TreeNode(val);
    if (!root) { root = newNode; return; }
    TreeNode* curr = root, *parent = nullptr;
    while (curr) {
        parent = curr;
        curr = (val < curr->val) ? curr->left : curr->right;
    }
    if (val < parent->val) parent->left  = newNode;
    else                    parent->right = newNode;
}
```

### Pitfall 5: Stack Overflow on Degenerate Trees

```cpp
// Recursive operations on a degenerate BST (sorted insertion) hit O(n) depth.
// For n=100,000: stack overflow with typical 1-8 MB stack.

// Safe: use iterative versions for search, insert, min/max.
// Deletion is harder to make iterative; consider height check + switch to iterative.

// Or: always build BST from sorted array (guarantees O(log n) height).
// Or: use std::set / std::map (red-black tree, always balanced).
```

### Pitfall 6: Modifying Value vs Reconnecting Pointers

```cpp
// Two valid deletion strategies for Case 3:
// Strategy A: copy successor's value into the node being deleted, delete successor.
//   Pro: clean, simple.  Con: value changes (invalidates external pointers to the node).

// Strategy B: splice out the node, reconnect pointers.
//   Pro: node identity preserved.  Con: more pointer manipulation.

// In interviews: Strategy A is standard. Mention "this changes the node's value,
// which matters if external code holds pointers to nodes."
```

---

## 15. BST vs Hash Map vs Sorted Array

| Property | BST (balanced) | Hash Map | Sorted Array |
|---|---|---|---|
| Search | O(log n) | O(1) avg | O(log n) binary search |
| Insert | O(log n) | O(1) avg | O(n) shift |
| Delete | O(log n) | O(1) avg | O(n) shift |
| Min / Max | O(log n) | O(n) | O(1) |
| Successor / Predecessor | O(log n) | O(n) | O(1) adjacent |
| Range query [a,b] | O(log n + k) | O(n) scan | O(log n + k) |
| Sorted iteration | O(n) in-order | O(n log n) sort | O(n) scan |
| Space | O(n) + pointer overhead | O(n) + hash overhead | O(n) |
| Order maintained | ✓ Yes | ✗ No | ✓ Yes |
| Worst case (unbal.) | O(n) | O(n) hash flood | O(log n) always |
| C++ equivalent | `std::map` / `std::set` | `std::unordered_map` | `std::vector` sorted |

**Decision rule:**

```
Use HASH MAP when:
  ✓ Exact key lookup only (no ordering needed)
  ✓ O(1) average performance matters more than O(log n) worst case
  ✓ Keys are not ordered or comparable
  ✗ Need sorted output, range queries, min/max → use BST

Use BST when:
  ✓ Need sorted order, min/max, range queries, successor/predecessor
  ✓ O(log n) WORST CASE matters (hash map: O(n) worst)
  ✓ Keys are comparable and ordering has semantic meaning
  ✗ Only need exact lookup → hash map is faster on average

Use SORTED ARRAY when:
  ✓ Read-heavy, rare insertions (binary search is simpler + cache-friendly)
  ✓ Data fits in memory and known upfront
  ✗ Frequent insertions/deletions → O(n) shifts are too expensive

Rule of thumb: reach for std::unordered_map by default;
               switch to std::map when ordering or range queries are needed.
```

---

## 16. Self-Test Questions

1. **State the BST property for a node N. Why must the property apply to the ENTIRE subtree and not just immediate children? Construct an example where checking only parent-child pairs gives a wrong result.**

2. **Trace the insertion of keys [5, 3, 7, 1, 4, 6, 8] into an empty BST. Draw the tree after all insertions. What is its height? Is it balanced?**

3. **Trace the deletion of node 5 (the root) from the BST in question 2. Show the state of the tree after the three cases are handled. What is the inorder successor used?**

4. **Why does inorder traversal of a BST produce sorted output? Prove this claim recursively using the BST property and the definition of inorder traversal.**

5. **What is the worst-case insertion sequence for a BST? What is the resulting height? How does this compare to the best-case insertion sequence?**

6. **In the BST validation problem, why do we need `long long` for the bounds instead of `int`? Construct a specific test case where using `int` produces a wrong answer.**

7. **Implement `findSuccessor(root, val)` iteratively (no recursion). Trace it on a BST to find the successor of 6.**

8. **In the Recover BST problem, why is there only ONE violation in the inorder sequence for adjacent swaps, but TWO violations for non-adjacent swaps? Show an example of each.**

9. **Compare `std::map` and `std::unordered_map`. For which operations is `std::map` faster? For which is `std::unordered_map` faster? Give the exact time complexities for each.**

10. **The Linux CFS scheduler uses a red-black tree (balanced BST) to schedule processes. What is the key (sorting criterion)? What operation does it need to perform on every scheduling decision, and why is O(log n) acceptable but O(n) would not be?**

---

## Quick Reference Card

```
Binary Search Tree — ordered binary tree enabling O(log n) search/insert/delete.

BST Property: for every node N:
  ALL values in left subtree  < N->val
  ALL values in right subtree > N->val
  (not just immediate children — the ENTIRE subtree!)

Key insight: BST = binary search externalised into a data structure.
  Each node is a decision: go left if target < val, go right if target > val.

Core operations (all O(h)):
  search(val):   navigate left/right until found or null.
  insert(val):   navigate to position, insert as leaf.
  delete(val):   3 cases: leaf (remove), 1 child (replace), 2 children (use successor).
  findMin():     follow left pointers to bottom.
  findMax():     follow right pointers to bottom.
  successor(v):  min of right subtree, or lowest ancestor where v is in left subtree.
  predecessor(v): symmetric.

The golden property: INORDER traversal of BST = SORTED ascending order.
  Use this for: validation, kth smallest, sorted output, range queries.

Deletion — 3 cases:
  1. Leaf:         delete directly.
  2. One child:    replace node with its only child.
  3. Two children: find inorder successor (min of right subtree),
                   copy its value here, delete the successor (falls into case 1 or 2).

Validation — MUST use range bounds (not just parent-child check):
  isValid(node, minVal, maxVal):
    node->val ∈ (minVal, maxVal) must hold for every node.
    Use long long to handle INT_MIN/INT_MAX boundary nodes.

Height and complexity:
  Balanced: h = O(log n) → all ops O(log n).
  Degenerate (sorted insert): h = O(n) → all ops O(n) → use balanced BST!

When to use BST (std::map) vs Hash Map (std::unordered_map):
  BST:      need sorted order, range queries, min/max, O(log n) worst case.
  Hash map: need only exact lookup, O(1) average, no ordering needed.

Pitfall checklist:
  ✗ Shallow BST validation (only checks parent-child) → misses deep violations
  ✗ Using int bounds for validation → overflow on INT_MIN/INT_MAX nodes
  ✗ Forgetting to return child pointer after single-child deletion
  ✗ Recursive search/insert on degenerate tree → stack overflow at large n
  ✗ Not updating parent pointer in iterative insert
```

---

*Previous: [22 — Binary Tree](./22_binary_tree.md)*
*Next: [24 — AVL Tree](./24_avl_tree.md)*
*See also: [Red-Black Tree](./trees/red_black_tree.md) | [std::map vs std::unordered_map](./comparisons/map_vs_unordered_map.md) | [Linux CFS scheduler](https://www.kernel.org/doc/html/latest/scheduler/sched-design-CFS.html)*

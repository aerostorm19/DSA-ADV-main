# AVL Tree

> **Curriculum position:** Trees → #3
> **Interview weight:** ★★★★☆ High — rotation mechanics and balance factor logic are standard in systems design rounds; implementing from scratch is a senior-level question
> **Difficulty:** Advanced — the concept is clear; the four rotation cases and deletion rebalancing require careful reasoning
> **Prerequisite:** [23 — Binary Search Tree](./23_binary_search_tree.md)

---

## Table of Contents

1. [Intuition First — Enforcing Balance at Every Node](#1-intuition-first)
2. [The Balance Factor — Measuring Imbalance](#2-the-balance-factor)
3. [The Four Imbalance Cases and Their Rotations](#3-the-four-rotation-cases)
4. [Internal Working — Height, BF, and Rebalancing](#4-internal-working)
5. [Insertion — Rebalance on the Way Up](#5-insertion)
6. [Deletion — The Most Complex Operation](#6-deletion)
7. [Time & Space Complexity](#7-time--space-complexity)
8. [Complete C++ Implementation](#8-complete-c-implementation)
9. [Core Operations — Visualised](#9-core-operations--visualised)
10. [Interview Problems](#10-interview-problems)
11. [Real-World Uses](#11-real-world-uses)
12. [Edge Cases & Pitfalls](#12-edge-cases--pitfalls)
13. [AVL vs Red-Black vs BST vs Skip List](#13-comparison)
14. [Self-Test Questions](#14-self-test-questions)

---

## 1. Intuition First

The plain BST has a fatal flaw: inserting keys in sorted order produces a degenerate tree of height O(n), reducing every operation to O(n). The BST's O(log n) guarantee exists only for randomly ordered input.

The **AVL tree**, invented by Adelson-Velsky and Landis in 1962, fixes this by enforcing a structural invariant after every insertion and deletion: **the heights of the left and right subtrees of every node may differ by at most 1.**

This single constraint — called the **AVL property** or **height-balance property** — guarantees that an AVL tree of n nodes always has height at most 1.44 log₂(n). No matter what order you insert keys, you never get a degenerate tree. The O(log n) guarantee holds not just on average but in the worst case.

How does it maintain this invariant? Through **rotations** — local restructuring operations that change the shape of the tree without violating the BST property. When an insertion or deletion causes a node to become unbalanced (height difference > 1), one or two rotations restore balance. Rotations are O(1) — they change a constant number of pointers.

Think of an AVL tree as a self-correcting BST. Every time you perturb the structure, it automatically reshapes itself to stay balanced. You pay a constant extra cost per modification (the rotation), but gain the guarantee that all future operations will complete in O(log n) time.

This trade-off — O(1) extra work per modification to guarantee O(log n) for everything — is the foundational idea behind all self-balancing BSTs: AVL trees, red-black trees, splay trees, treaps. Understanding AVL trees deeply means understanding why balancing works and what the minimal restructuring necessary to restore balance looks like.

---

## 2. The Balance Factor — Measuring Imbalance

### Definition

The **balance factor (BF)** of a node is:

```
BF(node) = height(left subtree) - height(right subtree)

By convention: height(null) = -1

Valid AVL node: BF ∈ {-1, 0, 1}
  BF = -1: right subtree is one level taller
  BF =  0: both subtrees equal height
  BF = +1: left subtree is one level taller

Imbalanced (needs rotation): BF ∈ {-2, +2}
  BF = -2: right-heavy by 2 levels
  BF = +2: left-heavy by 2 levels
```

### Why BF Can Only Reach ±2 After One Modification

A single insertion or deletion changes the height of at most one path from root to leaf. Any node whose subtree changed in height has its BF change by at most 1. Before the modification, all nodes had BF ∈ {-1, 0, 1}. After, the BF of at most O(h) nodes on the insertion path change by 1, potentially reaching ±2. Rotations fix the ±2 cases and restore all BF values to {-1, 0, 1}.

### Storing Height vs Balance Factor

Two implementation approaches:

```
Approach 1: Store HEIGHT at each node (more flexible).
  BF computed on demand: BF(node) = height(left) - height(right).
  After rotations: recalculate heights bottom-up.
  Pro: height accessible in O(1) for other queries.
  Con: 4 bytes extra per node.

Approach 2: Store BALANCE FACTOR directly (2-3 bits per node).
  Pro: minimal extra memory.
  Con: updating BF after rotations requires careful bookkeeping.
  Used in some textbooks and memory-constrained systems.

This document uses Approach 1 (store height) — simpler to implement correctly.
```

---

## 3. The Four Rotation Cases

When a node becomes imbalanced (BF = ±2), exactly one of four cases applies. Each has a specific rotation remedy.

### Identifying the Case

For an imbalanced node Z with BF = ±2:
- **BF(Z) = +2** (left-heavy): the problem is in the left subtree. Let Y = Z's left child.
  - If BF(Y) ≥ 0 (left-left): **Right Rotation (LL case)**
  - If BF(Y) < 0 (left-right): **Left-Right Rotation (LR case)**
- **BF(Z) = -2** (right-heavy): the problem is in the right subtree. Let Y = Z's right child.
  - If BF(Y) ≤ 0 (right-right): **Left Rotation (RR case)**
  - If BF(Y) > 0 (right-left): **Right-Left Rotation (RL case)**

### Case 1: Right Rotation (LL — Left-Left)

Imbalance: Z is left-heavy (BF=+2) and Y = Z->left is left-heavy or balanced (BF ≥ 0).

```
Before:                After Right Rotation:
    Z  (BF=+2)              Y  (BF=0)
   / \                     / \
  Y   T3  (BF≥0)          X   Z  (BF=0)
 / \                      / \ / \
X   T2                   T1 T2 T2 T3

Right rotation: Y rises, Z falls. Y's right child becomes Z's left child.
```

```cpp
TreeNode* rotateRight(TreeNode* Z) {
    TreeNode* Y  = Z->left;
    TreeNode* T2 = Y->right;
    Y->right = Z;     // Y takes Z's place
    Z->left  = T2;    // T2 moves to Z's left
    updateHeight(Z);  // Z's height changes first (it's now lower)
    updateHeight(Y);  // Y's height changes after
    return Y;         // Y is the new root of this subtree
}
```

### Case 2: Left Rotation (RR — Right-Right)

Imbalance: Z is right-heavy (BF=-2) and Y = Z->right is right-heavy or balanced (BF ≤ 0).

```
Before:                After Left Rotation:
  Z  (BF=-2)               Y  (BF=0)
 / \                       / \
T1  Y  (BF≤0)             Z   X
   / \                   / \ / \
  T2  X                 T1 T2 T3 T4

Left rotation: Y rises, Z falls. Y's left child becomes Z's right child.
```

```cpp
TreeNode* rotateLeft(TreeNode* Z) {
    TreeNode* Y  = Z->right;
    TreeNode* T2 = Y->left;
    Y->left  = Z;     // Y takes Z's place
    Z->right = T2;    // T2 moves to Z's right
    updateHeight(Z);
    updateHeight(Y);
    return Y;
}
```

### Case 3: Left-Right Rotation (LR)

Imbalance: Z is left-heavy (BF=+2) and Y = Z->left is right-heavy (BF=-1). The imbalance is "bent" — in the inner subtree between Z and Y.

```
Before:           After Left on Y:      After Right on Z:
    Z (BF=+2)         Z                     X
   /                 /                    /   \
  Y  (BF=-1)        X                   Y     Z
   \               /                         
    X             Y                    

Fix: First rotate LEFT on Y, then rotate RIGHT on Z.
This converts the LR case into an LL case, then handles it as Case 1.
```

```cpp
// LR: left-rotate Y (Z->left), then right-rotate Z
Z->left = rotateLeft(Z->left);
return rotateRight(Z);
```

### Case 4: Right-Left Rotation (RL)

Imbalance: Z is right-heavy (BF=-2) and Y = Z->right is left-heavy (BF=+1). Symmetric to LR.

```
Before:           After Right on Y:     After Left on Z:
  Z (BF=-2)           Z                     X
   \                   \                  /   \
    Y (BF=+1)           X               Z     Y
   /                     \
  X                       Y

Fix: First rotate RIGHT on Y, then rotate LEFT on Z.
```

```cpp
// RL: right-rotate Y (Z->right), then left-rotate Z
Z->right = rotateRight(Z->right);
return rotateLeft(Z);
```

### The Decision Table

```
BF(Z) = +2 (left-heavy):
  BF(Z->left) ≥ 0: LL case → Right Rotation on Z
  BF(Z->left) < 0: LR case → Left on Z->left, then Right on Z

BF(Z) = -2 (right-heavy):
  BF(Z->right) ≤ 0: RR case → Left Rotation on Z
  BF(Z->right) > 0: RL case → Right on Z->right, then Left on Z
```

---

## 4. Internal Working — Height, BF, and Rebalancing

### Height Maintenance

After every rotation or structural change, heights must be updated bottom-up:

```cpp
int height(TreeNode* node) {
    return node ? node->height : -1;  // height of null = -1
}

void updateHeight(TreeNode* node) {
    if (node)
        node->height = 1 + max(height(node->left), height(node->right));
}

int balanceFactor(TreeNode* node) {
    return node ? height(node->left) - height(node->right) : 0;
}
```

### The Rebalance Function

```cpp
TreeNode* rebalance(TreeNode* node) {
    updateHeight(node);
    int bf = balanceFactor(node);

    // Left-heavy
    if (bf == 2) {
        if (balanceFactor(node->left) >= 0) {
            return rotateRight(node);           // LL
        } else {
            node->left = rotateLeft(node->left);// LR: fix left child first
            return rotateRight(node);
        }
    }
    // Right-heavy
    if (bf == -2) {
        if (balanceFactor(node->right) <= 0) {
            return rotateLeft(node);            // RR
        } else {
            node->right = rotateRight(node->right);// RL: fix right child first
            return rotateLeft(node);
        }
    }
    return node;  // already balanced
}
```

### Why Height Update Order Matters

After `rotateRight(Z)`: Z moves down, Y moves up. Z's new height depends on its new children (T2, T3). Y's new height depends on X and Z. So Z's height must be computed BEFORE Y's:

```
rotateRight(Z):
  1. Y->right = Z; Z->left = T2
  2. updateHeight(Z)   // Z is now a child of Y — must be updated first
  3. updateHeight(Y)   // Y depends on Z's updated height
```

---

## 5. Insertion — Rebalance on the Way Up

### The Algorithm

```
insert(node, val):
  1. Standard BST insert (navigate to the leaf position)
  2. On the WAY BACK UP (via recursion), call rebalance() at each ancestor
  3. Rotations fix imbalance at the first unbalanced ancestor

Key property: after inserting one node, at most ONE node becomes imbalanced
(the lowest ancestor of the new node that was previously balanced with BF=0
and now has BF=±1 has exactly one path lengthened).
Correction: AT MOST ONE ROTATION is needed after insertion.
```

```cpp
TreeNode* insert(TreeNode* node, int val) {
    // Standard BST insert
    if (!node) return new TreeNode(val);
    if      (val < node->val) node->left  = insert(node->left,  val);
    else if (val > node->val) node->right = insert(node->right, val);
    else return node;  // duplicate — ignore

    // Rebalance this node if needed
    return rebalance(node);
}
```

### Why Only One Rotation After Insertion

When a new leaf is added:
- Heights increase along the path from new leaf to root
- Height propagates upward only until a node absorbs the extra height (when BF changes from -1→0 or +1→0, the subtree height does NOT increase further)
- The FIRST node that becomes imbalanced (BF goes to ±2) is fixed by one rotation
- After the rotation, the subtree's height RETURNS to what it was before insertion — no further imbalance propagates upward

This "height restoration" property is unique to insertion. Deletion does NOT have this property — it may require multiple rotations all the way to the root.

---

## 6. Deletion — The Most Complex Operation

### Why Deletion Is Harder

After deleting a node, heights decrease along the path from the deleted node to the root. Unlike insertion, a rotation during deletion may NOT restore the subtree to its pre-deletion height. Height can continue to decrease upward, requiring rebalancing at multiple ancestors — up to O(log n) rotations for a single deletion.

### The Algorithm

```cpp
TreeNode* deleteMin(TreeNode* node) {
    if (!node->left) return node->right;  // node IS the minimum
    node->left = deleteMin(node->left);
    return rebalance(node);
}

TreeNode* deleteNode(TreeNode* node, int val) {
    if (!node) return nullptr;

    if      (val < node->val) node->left  = deleteNode(node->left,  val);
    else if (val > node->val) node->right = deleteNode(node->right, val);
    else {
        // Found the node to delete
        if (!node->left || !node->right) {
            // Case 1 or 2: zero or one child
            TreeNode* child = node->left ? node->left : node->right;
            delete node;
            return child;
        }
        // Case 3: two children — replace with inorder successor (min of right)
        TreeNode* succ = node->right;
        while (succ->left) succ = succ->left;
        node->val    = succ->val;
        node->right  = deleteNode(node->right, succ->val);
    }
    return rebalance(node);  // rebalance on every node on the way back up
}
```

### Deletion Rebalancing: Up to O(log n) Rotations

```
Delete causes height decrease in a subtree.
Parent of deleted subtree: BF may become ±2 → rotation.
After rotation: height of rotated subtree may decrease → grandparent's BF may change.
This cascades: up to O(log n) rotations may be needed along the path to root.

Contrast with insertion: at most 1 rotation needed.
This asymmetry is a key fact about AVL trees.
```

---

## 7. Time & Space Complexity

### Operation Complexities

| Operation | Time | Notes |
|---|---|---|
| `search(val)` | **O(log n)** | Height guaranteed O(log n) |
| `insert(val)` | **O(log n)** | BST insert + O(1) rotations |
| `delete(val)` | **O(log n)** | BST delete + O(log n) rotations |
| `findMin()` | **O(log n)** | Follow left pointers |
| `findMax()` | **O(log n)** | Follow right pointers |
| `successor(v)` | **O(log n)** | Right subtree min or ancestor |
| `rangeSearch` | **O(log n + k)** | k = results |
| Inorder traversal | O(n) | Visits all nodes |
| Rotation (single) | **O(1)** | Constant pointer updates |
| Build from n items | O(n log n) | n insertions |
| Build from sorted | O(n) | Special construction |

All primary operations are **O(log n) in the worst case** — this is the core guarantee AVL trees provide over plain BSTs.

### Space Complexity

| Component | Space | Notes |
|---|---|---|
| n nodes | O(n) | One node per entry |
| Per-node overhead | 4 bytes height + 2 × 8 bytes pointers | 20 bytes beyond the key+value |
| Call stack (ops) | O(log n) | Height is O(log n) |
| vs plain BST | +4 bytes per node | The height field is the only addition |

### The Height Guarantee

The AVL tree height guarantee is:

```
For an AVL tree with n nodes:
  h ≤ 1.44 × log₂(n + 2) - 0.328

For n=1,000,000: h ≤ 1.44 × 20 - 0.33 ≈ 28.4 → at most 29 levels.
For n=1,000,000,000: h ≤ 1.44 × 30 - 0.33 ≈ 43 levels.

Compare:
  Perfect BST: h = log₂(n) ≈ 20 (for n=10^6)
  AVL tree:    h ≤ 1.44 log₂(n) ≈ 29 (44% taller than perfect)
  Red-black:   h ≤ 2 log₂(n+1) ≈ 40 (100% taller than perfect)
  Plain BST:   h = O(n) in worst case

AVL is the MOST height-balanced self-balancing BST.
This comes at the cost of more rotations during insert/delete.
```

### Proof of the Height Bound (Fibonacci Argument)

The minimum number of nodes in an AVL tree of height h, N(h), satisfies:
```
N(-1) = 0 (empty)
N(0)  = 1 (single node)
N(h)  = N(h-1) + N(h-2) + 1

This is the Fibonacci recurrence! N(h) ≈ φ^h / √5 where φ = (1+√5)/2 ≈ 1.618.
Solving for h: h ≈ log_φ(n√5) = log₂(n√5) / log₂(φ) ≈ 1.44 log₂(n).
```

---

## 8. Complete C++ Implementation

```cpp
#include <iostream>
#include <algorithm>
#include <vector>
#include <functional>
#include <optional>
using namespace std;

struct AVLNode {
    int      val;
    int      height;    // height of subtree rooted here
    AVLNode* left;
    AVLNode* right;

    explicit AVLNode(int v)
        : val(v), height(0), left(nullptr), right(nullptr) {}
};

class AVLTree {
private:
    AVLNode* root_ = nullptr;

    // ── Height utilities ──────────────────────────────────────────────────────

    int height(AVLNode* n) const { return n ? n->height : -1; }

    void updateHeight(AVLNode* n) {
        if (n) n->height = 1 + max(height(n->left), height(n->right));
    }

    int bf(AVLNode* n) const {
        return n ? height(n->left) - height(n->right) : 0;
    }

    // ── Rotations ─────────────────────────────────────────────────────────────

    // Right rotation (LL case):
    //       Z              Y
    //      / \            / \
    //     Y  T3    →     X   Z
    //    / \                / \
    //   X  T2              T2  T3
    AVLNode* rotateRight(AVLNode* Z) {
        AVLNode* Y = Z->left;
        AVLNode* T2 = Y->right;
        Y->right = Z;
        Z->left  = T2;
        updateHeight(Z);   // Z is now lower — update first
        updateHeight(Y);   // Y is now higher — update after Z
        return Y;
    }

    // Left rotation (RR case):
    //   Z                  Y
    //  / \                / \
    // T1   Y    →        Z   X
    //     / \           / \
    //    T2  X         T1  T2
    AVLNode* rotateLeft(AVLNode* Z) {
        AVLNode* Y = Z->right;
        AVLNode* T2 = Y->left;
        Y->left  = Z;
        Z->right = T2;
        updateHeight(Z);
        updateHeight(Y);
        return Y;
    }

    // ── Rebalance ─────────────────────────────────────────────────────────────
    // Update height and apply rotation if BF = ±2. Returns new subtree root.
    AVLNode* rebalance(AVLNode* node) {
        if (!node) return nullptr;
        updateHeight(node);
        int b = bf(node);

        // Left-heavy (BF = +2)
        if (b == 2) {
            if (bf(node->left) >= 0) {
                // LL: single right rotation
                return rotateRight(node);
            } else {
                // LR: left-rotate left child, then right-rotate node
                node->left = rotateLeft(node->left);
                return rotateRight(node);
            }
        }

        // Right-heavy (BF = -2)
        if (b == -2) {
            if (bf(node->right) <= 0) {
                // RR: single left rotation
                return rotateLeft(node);
            } else {
                // RL: right-rotate right child, then left-rotate node
                node->right = rotateRight(node->right);
                return rotateLeft(node);
            }
        }

        return node;  // BF ∈ {-1, 0, 1}: already balanced
    }

    // ── Insert ────────────────────────────────────────────────────────────────

    AVLNode* insert_(AVLNode* node, int val) {
        if (!node) return new AVLNode(val);
        if      (val < node->val) node->left  = insert_(node->left,  val);
        else if (val > node->val) node->right = insert_(node->right, val);
        else return node;  // duplicate: no-op
        return rebalance(node);
    }

    // ── Find minimum in subtree ───────────────────────────────────────────────

    AVLNode* findMin_(AVLNode* node) const {
        if (!node) return nullptr;
        while (node->left) node = node->left;
        return node;
    }

    // ── Delete ────────────────────────────────────────────────────────────────

    AVLNode* delete_(AVLNode* node, int val) {
        if (!node) return nullptr;

        if (val < node->val) {
            node->left  = delete_(node->left,  val);
        } else if (val > node->val) {
            node->right = delete_(node->right, val);
        } else {
            // Found node to delete
            if (!node->left || !node->right) {
                AVLNode* child = node->left ? node->left : node->right;
                delete node;
                return child;                  // Case 1 or 2
            }
            // Case 3: two children — use inorder successor
            AVLNode* succ = findMin_(node->right);
            node->val    = succ->val;
            node->right  = delete_(node->right, succ->val);
        }
        return rebalance(node);  // rebalance on the way back up
    }

    // ── Search ────────────────────────────────────────────────────────────────

    AVLNode* search_(AVLNode* node, int val) const {
        if (!node || node->val == val) return node;
        return val < node->val ? search_(node->left, val) : search_(node->right, val);
    }

    // ── Inorder traversal ─────────────────────────────────────────────────────

    void inorder_(AVLNode* node, vector<int>& out) const {
        if (!node) return;
        inorder_(node->left, out);
        out.push_back(node->val);
        inorder_(node->right, out);
    }

    // ── Range search ─────────────────────────────────────────────────────────

    void range_(AVLNode* node, int lo, int hi, vector<int>& out) const {
        if (!node) return;
        if (node->val > lo) range_(node->left,  lo, hi, out);
        if (node->val >= lo && node->val <= hi) out.push_back(node->val);
        if (node->val < hi) range_(node->right, lo, hi, out);
    }

    // ── Validation ────────────────────────────────────────────────────────────

    bool isValidAVL_(AVLNode* node, long long minV, long long maxV) const {
        if (!node) return true;
        if (node->val <= minV || node->val >= maxV) return false;
        if (abs(bf(node)) > 1) return false;
        if (node->height != 1 + max(height(node->left), height(node->right))) return false;
        return isValidAVL_(node->left,  minV,         node->val)
            && isValidAVL_(node->right, node->val, maxV);
    }

    // ── Cleanup ───────────────────────────────────────────────────────────────

    void destroy_(AVLNode* node) {
        if (!node) return;
        destroy_(node->left);
        destroy_(node->right);
        delete node;
    }

public:
    // ── Constructors ──────────────────────────────────────────────────────────

    AVLTree() = default;

    ~AVLTree() { destroy_(root_); }

    AVLTree(const AVLTree&)            = delete;
    AVLTree& operator=(const AVLTree&) = delete;
    AVLTree(AVLTree&& o) noexcept : root_(o.root_) { o.root_ = nullptr; }

    // ── Public interface ──────────────────────────────────────────────────────

    void insert(int val)  { root_ = insert_(root_, val); }
    void remove(int val)  { root_ = delete_(root_, val); }

    bool   contains(int val) const { return search_(root_, val) != nullptr; }
    int    height()          const { return height(root_); }
    bool   empty()           const { return root_ == nullptr; }
    bool   isValidAVL()      const {
        return isValidAVL_(root_, (long long)INT_MIN - 1, (long long)INT_MAX + 1);
    }

    optional<int> findMin() const {
        AVLNode* m = findMin_(root_);
        return m ? optional<int>(m->val) : nullopt;
    }

    optional<int> findMax() const {
        AVLNode* n = root_;
        if (!n) return nullopt;
        while (n->right) n = n->right;
        return n->val;
    }

    vector<int> inorder() const {
        vector<int> out;
        inorder_(root_, out);
        return out;
    }

    vector<int> rangeSearch(int lo, int hi) const {
        vector<int> out;
        range_(root_, lo, hi, out);
        return out;
    }

    // ── Expose root for algorithms ────────────────────────────────────────────
    AVLNode* root() const { return root_; }

    // ── Pretty print ─────────────────────────────────────────────────────────
    void print() const {
        function<void(AVLNode*, string, bool)> pr =
            [&](AVLNode* n, string pre, bool isLeft) {
                if (!n) return;
                pr(n->right, pre + (isLeft ? "│   " : "    "), false);
                cout << pre << (isLeft ? "└── " : "┌── ")
                     << n->val << "(h=" << n->height
                     << ",bf=" << bf(n) << ")\n";
                pr(n->left, pre + (isLeft ? "    " : "│   "), true);
            };
        pr(root_, "", true);
    }
};

// ── Standalone rotation functions (LeetCode / interview style) ────────────────

// For use with raw TreeNode* (e.g., when interviewer provides node struct)
struct TreeNode {
    int val, height;
    TreeNode *left, *right;
    explicit TreeNode(int v) : val(v), height(0), left(nullptr), right(nullptr) {}
};

int ht(TreeNode* n) { return n ? n->height : -1; }
void upd(TreeNode* n) { if (n) n->height = 1 + max(ht(n->left), ht(n->right)); }
int  bfactor(TreeNode* n) { return n ? ht(n->left) - ht(n->right) : 0; }

TreeNode* rRight(TreeNode* z) {
    auto y = z->left, t2 = y->right;
    y->right = z; z->left = t2;
    upd(z); upd(y); return y;
}
TreeNode* rLeft(TreeNode* z) {
    auto y = z->right, t2 = y->left;
    y->left = z; z->right = t2;
    upd(z); upd(y); return y;
}
TreeNode* balance(TreeNode* n) {
    if (!n) return n;
    upd(n);
    int b = bfactor(n);
    if (b ==  2) return bfactor(n->left)  >= 0 ? rRight(n) : (n->left  = rLeft(n->left),  rRight(n));
    if (b == -2) return bfactor(n->right) <= 0 ? rLeft(n)  : (n->right = rRight(n->right), rLeft(n));
    return n;
}
TreeNode* avlInsert(TreeNode* n, int v) {
    if (!n) return new TreeNode(v);
    if      (v < n->val) n->left  = avlInsert(n->left,  v);
    else if (v > n->val) n->right = avlInsert(n->right, v);
    return balance(n);
}
```

---

## 9. Core Operations — Visualised

### Insertion with LL Rotation

```
Insert 3, 2, 1 into empty AVL tree:

After insert 3:
  3 (h=0, bf=0)

After insert 2 (2 < 3 → left of 3):
  3 (h=1, bf=1)
 /
2 (h=0, bf=0)
BF(3) = 1. Valid.

After insert 1 (1 < 3 → left; 1 < 2 → left of 2):
  3 (h=2, bf=2!) ← IMBALANCED!
 /
2 (h=1, bf=1)
/
1 (h=0, bf=0)

bf(3) = 2 (left-heavy). bf(3->left = 2) = 1 ≥ 0. → LL CASE: Right Rotation on 3.

rotateRight(3):
  Y = 3->left = 2
  T2 = Y->right = null
  Y->right = 3 (2's right becomes 3)
  3->left = T2 = null
  updateHeight(3): max(-1,-1)+1 = 0
  updateHeight(2): max(0,0)+1 = 1

Result:
  2 (h=1, bf=0)
 / \
1   3
(h=0)(h=0)

All BFs = 0. Valid AVL tree. ✓
```

### Insertion with LR Rotation

```
Insert 3, 1, 2 (causes Left-Right case):

After insert 3, 1:
  3 (h=1, bf=1)
 /
1 (h=0, bf=0)

After insert 2 (2 < 3 → left; 2 > 1 → right of 1):
  3 (h=2, bf=2!) ← IMBALANCED
 /
1 (h=1, bf=-1)
 \
  2 (h=0, bf=0)

bf(3)=2 (left-heavy). bf(3->left=1)=-1 < 0. → LR CASE.

Step 1: Left-rotate 1 (3->left):
  rotateLeft(1):
    Y = 1->right = 2
    T2 = Y->left = null
    Y->left = 1, 1->right = null
    updateHeight(1): 0, updateHeight(2): 1

  3 (h=2, bf=2)
 /
2 (h=1, bf=1)
/
1 (h=0)

Step 2: Right-rotate 3:
  rotateRight(3):
    Y = 3->left = 2, T2 = Y->right = null
    Y->right = 3, 3->left = null
    updateHeight(3): 0, updateHeight(2): 1

Result:
  2 (h=1, bf=0)
 / \
1   3
(h=0)(h=0)

Valid AVL tree. ✓ — Same shape as LL result but via a different rotation sequence.
```

### Deletion with Multiple Rebalances

```
AVL tree:
        8 (h=3, bf=0)
       / \
      4   12 (h=2, bf=0)
     / \  / \
    2  6 10  14
      /   \
     5     11

Delete 14 (leaf — simple removal):
        8 (h=3, bf=0)
       / \
      4   12 (h=2, bf=-1) ← height decreases
         / \
        10  ← 14 gone
         \
          11

Update heights up from 12:
  12: max(h(10),h(null)) + 1 = max(1,-1)+1 = 2? Wait:
  10 has right child 11 (h=0). h(10) = max(-1,0)+1 = 1.
  h(12) = max(1,-1)+1 = 2.
  bf(12) = h(10) - h(null) = 1 - (-1) = 2!  ← Wait, 14 was removed.

Actually:
  Before: 12 has left=10 (h=1, right child 11), right=14 (h=0).
  After: 12 has left=10 (h=1), right=null (h=-1).
  h(12) = max(1,-1)+1 = 2. bf(12) = 1-(-1) = 2. IMBALANCED!

bf(12) = 2 (left-heavy). bf(12->left = 10) = h(null) - h(11) = -1-0 = -1 < 0.
→ RL CASE? Wait: 12 is left-heavy with bf=+2, left child bf=-1 → LR case!

LR on 12:
  Left-rotate 10: (11 rises, 10 falls)
      12
     /
    11 (h=1)
   /
  10 (h=0)
  
  Right-rotate 12:
     11 (h=1, bf=0)
    / \
   10  12

Update heights up to 8: bf(8) = h(4) - h(11) = 2 - 1 = 1. Still valid!

Final:
        8 (h=3, bf=1)
       / \
      4   11 (h=1, bf=0)
     / \  / \
    2  6 10  12
      /
     5
```

---

## 10. Interview Problems

### Problem 1: Count Nodes in AVL Tree in O(log²n)

**Problem:** Given a **complete binary tree** (all levels full except possibly the last, which is filled left to right), count the number of nodes in less than O(n) time.

**Connection to AVL:** This problem appears in AVL contexts because: (a) it requires understanding tree height, and (b) the AVL height guarantee makes the O(log²n) bound tight and useful.

**Thought process:**
> "Naively traversing all nodes: O(n). Can we exploit the completeness? In a perfect binary tree of height h: count = 2^(h+1) - 1. For a complete tree: check if left and right heights are equal. If yes, it's perfect on the left — count the right side recursively. If no, it's perfect on the right — count the left side recursively. Each recursion level: O(log n) to compute height, O(log n) levels → O(log²n) total."

```cpp
// Time: O(log²n) | Space: O(log n)
int countNodes(TreeNode* root) {
    if (!root) return 0;

    // Compute left-most height (go all the way left)
    int leftHeight = 0;
    TreeNode* node = root;
    while (node->left) { node = node->left; leftHeight++; }

    // Compute right-most height (go all the way right)
    int rightHeight = 0;
    node = root;
    while (node->right) { node = node->right; rightHeight++; }

    // If left-most depth = right-most depth: perfect binary tree
    if (leftHeight == rightHeight) {
        return (1 << (leftHeight + 1)) - 1;  // 2^(h+1) - 1
    }

    // Not perfect: recursively count
    return 1 + countNodes(root->left) + countNodes(root->right);
}

/*
For a complete binary tree of n nodes:
  - Either the left subtree is perfect (and right is complete): recurse right
  - Or the right subtree is perfect (and left is complete): recurse left
  Each level: one subtree is perfect (O(1) count), other recurses.
  Recursion depth: O(log n). Each level: O(log n) for height computation.
  Total: O(log²n). ✓

AVL connection: an AVL tree of n nodes has height ≤ 1.44 log₂(n).
If you use this counting algorithm to verify a complete AVL tree,
the O(log²n) bound is tight because log(n) levels × log(n) height computation.
*/
```

---

### Problem 2: Balanced BST from Sorted Array — Prove It's AVL

**Problem:** Given a sorted array of n distinct integers, construct a balanced BST. Prove that the resulting tree is a valid AVL tree.

**Insight:** Always choose the middle element as the root. This guarantees that left and right subtrees have sizes ⌊(n-1)/2⌋ and ⌈(n-1)/2⌉ — they differ by at most 1. By induction, both subtrees are AVL trees. Their heights differ by at most 1, satisfying the AVL property.

```cpp
// O(n) construction | O(log n) height guaranteed
TreeNode* sortedToBST(vector<int>& nums, int lo, int hi) {
    if (lo > hi) return nullptr;
    int mid = lo + (hi - lo) / 2;   // always choose middle → balanced split

    TreeNode* node = new TreeNode(nums[mid]);
    node->left  = sortedToBST(nums, lo, mid - 1);
    node->right = sortedToBST(nums, mid + 1, hi);

    // Update height (if using AVL nodes)
    node->height = 1 + max(ht(node->left), ht(node->right));
    return node;
}

// Proof that this produces a valid AVL tree:
//
// Claim: sortedToBST([lo..hi]) produces a valid AVL tree.
// Proof by strong induction on n = hi - lo + 1.
//
// Base case: n=0 (null) and n=1 (single node). Both valid AVL trees trivially.
//
// Inductive step: Assume true for all arrays of length < n.
//   Choose mid = lo + (n-1)/2.
//   Left subarray:  [lo..mid-1], length = ⌊(n-1)/2⌋.
//   Right subarray: [mid+1..hi], length = ⌈(n-1)/2⌉.
//
//   By induction: both subtrees are valid AVL trees.
//   Heights: let h_L = height of left, h_R = height of right.
//   Left length  ≤ right length ≤ left length + 1.
//
//   For AVL validity: |h_L - h_R| ≤ 1.
//   This holds because both subtrees are "almost equal" in size.
//   For even n-1: left length = right length → h_L = h_R (perfect split).
//   For odd n-1:  right length = left length + 1 → h_R ≤ h_L + 1.
//   Either way, |h_L - h_R| ≤ 1. ✓
//
//   BST property: nums[lo..mid-1] < nums[mid] < nums[mid+1..hi] (sorted input). ✓
//   Therefore: sortedToBST produces a valid AVL tree. □

// Test: verify the result satisfies AVL invariants
bool verifyAVL(TreeNode* node, long long minV = LLONG_MIN, long long maxV = LLONG_MAX) {
    if (!node) return true;
    if (node->val <= minV || node->val >= maxV) return false;
    if (abs(bfactor(node)) > 1) return false;
    return verifyAVL(node->left,  minV, node->val)
        && verifyAVL(node->right, node->val, maxV);
}
```

---

### Problem 3: AVL Insertion Sequence — Predict the Final Tree

**Problem:** Given the insertion sequence [10, 20, 30, 40, 50, 25], trace the AVL tree construction. Show each rotation triggered and the final tree structure.

**This is the canonical hand-trace problem** — asked to verify understanding of all four rotation cases.

```cpp
/*
Start empty. Insert in order: 10, 20, 30, 40, 50, 25.

Step 1 — Insert 10:
  10 (h=0, bf=0) ✓

Step 2 — Insert 20 (20 > 10 → right of 10):
  10 (h=1, bf=-1) ✓
   \
    20 (h=0)

Step 3 — Insert 30 (30 > 10 → right; 30 > 20 → right of 20):
  10 (h=2, bf=-2) ← IMBALANCED!
   \
    20 (h=1, bf=-1)
     \
      30 (h=0)

bf(10)=-2, bf(10->right=20)=-1 ≤ 0. → RR CASE: Left Rotation on 10.

rotateLeft(10): Y=20, T2=20->left=null.
  20->left = 10, 10->right = null.
  h(10)=0, h(20)=1.

Result:
    20 (h=1, bf=0)
   /  \
  10   30

Step 4 — Insert 40 (40 > 20 → right; 40 > 30 → right of 30):
    20 (h=2, bf=-1) ✓
   /  \
  10   30 (h=1, bf=-1)
         \
          40 (h=0)

Step 5 — Insert 50 (50 > 20 → right; 50 > 30 → right; 50 > 40 → right of 40):
    20 (h=3, bf=-2) ← Check heights:
   /  \
  10   30 (h=2, bf=-2) ← ALSO IMBALANCED but we fix LOWEST first
         \
          40 (h=1, bf=-1)
            \
             50 (h=0)

Rebalance on way back up — rebalance(30) first:
  bf(30)=-2, bf(30->right=40)=-1. → RR: Left Rotation on 30.

rotateLeft(30): Y=40, T2=null.
  40->left=30, 30->right=null.
  h(30)=0, h(40)=1.

Now rebalance(20): left=10(h=0), right=40(h=1). h(20)=2, bf=-1. ✓

Tree:
    20 (h=2, bf=-1)
   /  \
  10   40 (h=1, bf=0)
      /  \
     30   50

Step 6 — Insert 25 (25 < 20? No. 25 > 20 → right; 25 < 40 → left; 25 < 30 → left of 30):
    20 (h=3, bf=-1) ... 
   /  \
  10   40 (h=2, bf=1) ← bf changes
      /  \
     30   50
    /
   25

Check balance on way up:
  rebalance(30): h=1, bf=1. ✓
  rebalance(40): left=30(h=1), right=50(h=0). h=2. bf = 1 - 0 = 1. ✓
  rebalance(20): left=10(h=0), right=40(h=2). h=3. bf = 0 - 2 = -2. ← IMBALANCED!

bf(20)=-2, bf(20->right=40)=+1 > 0. → RL CASE!

Step: Right-rotate 40's left child? Wait — RL case on node Z=20:
  Z=20, Z->right=40 (bf=+1 > 0) → RL case.
  First: right-rotate 40.
  
rotateRight(40): Y=40->left=30, T2=30->right=null.
  30->right=40, 40->left=null (T2).
  h(40)=max(h(null),h(50))+1 = max(-1,0)+1 = 1.
  h(30)=max(h(25),h(40))+1 = max(0,1)+1 = 2.

  Tree so far (after right-rotate 40):
      20 (h=3, bf=-2)
     /  \
    10   30 (h=2, bf=0)
        /  \
       25   40 (h=1, bf=-1)
              \
               50

  Then: left-rotate 20.

rotateLeft(20): Y=30, T2=30->left=25.
  30->left=20, 20->right=25.
  h(20)=max(h(10),h(25))+1 = max(0,0)+1 = 1.
  h(30)=max(h(20),h(40))+1 = max(1,1)+1 = 2.

FINAL AVL TREE:
      30 (h=2, bf=0)
     /  \
    20   40 (h=1, bf=-1)
   /  \    \
  10   25   50

Verify all balance factors:
  10: bf=0 ✓   25: bf=0 ✓   50: bf=0 ✓
  20: bf = h(10)-h(25) = 0-0 = 0 ✓
  40: bf = h(null)-h(50) = -1-0 = -1 ✓
  30: bf = h(20)-h(40) = 1-1 = 0 ✓

Inorder: 10,20,25,30,40,50 ✓ (sorted)
All BST and AVL properties satisfied. ✓
*/
```

---

## 11. Real-World Uses

| Domain | System | AVL Detail |
|---|---|---|
| **Database indexes** | Some DBMS implementations | AVL trees used for in-memory index structures needing strict balance |
| **Memory allocators** | jemalloc, Slab allocator | AVL trees for free-block size → address mapping with strict O(log n) |
| **Compiler symbol tables** | GCC, Clang (some components) | Scoped symbol tables using AVL for O(log n) lookup during parsing |
| **Set/Map implementations** | Java TreeMap (historically) | Some JVM implementations used AVL before standardising on Red-Black |
| **Computational geometry** | Interval trees, sweep line algorithms | Maintain active segments in O(log n) per event |
| **Real-time systems** | Embedded RTOS task schedulers | Strict O(log n) needed for deadline guarantees |
| **Text indexing** | Full-text search engines (AVL-based lexicons) | Sorted dictionary with O(log n) lookup |
| **Numerical computing** | k-d tree variants for nearest neighbour | AVL-balanced k-d trees for point set queries |

### Where AVL Beats Red-Black

AVL trees are preferred over red-black trees in **read-heavy workloads** because:

```
AVL maximum height:        1.44 × log₂(n)
Red-Black maximum height:  2.00 × log₂(n + 1)

For n=1,000,000:
  AVL:       max 29 levels
  Red-Black: max 40 levels

Lookup in AVL: at most 29 comparisons.
Lookup in Red-Black: at most 40 comparisons.
AVL lookups are ~38% faster in the worst case.

But: AVL insert/delete does MORE rotations than Red-Black.
  AVL delete: up to O(log n) rotations.
  Red-Black:  at most 3 rotations per delete.

Trade-off:
  AVL → better for read-heavy (databases, caches)
  Red-Black → better for write-heavy (Linux scheduler, std::map)
```

---

## 12. Edge Cases & Pitfalls

### Pitfall 1: Updating Height Before Checking Balance Factor

```cpp
// WRONG: checking BF before updating height gives stale BF
AVLNode* rebalance_wrong(AVLNode* node) {
    int b = bf(node);   // using stale height from before the insertion!
    updateHeight(node); // too late
    if (b == 2) { ... }
    return node;
}

// CORRECT: update height FIRST, then check BF
AVLNode* rebalance(AVLNode* node) {
    updateHeight(node);   // refresh height
    int b = bf(node);     // BF is now accurate
    if (b == 2) { ... }
    return node;
}
```

### Pitfall 2: Wrong Height Update Order in Rotations

```cpp
// WRONG: update Y before Z — Y depends on Z's height, which hasn't been updated yet
AVLNode* rotateRight_wrong(AVLNode* Z) {
    AVLNode* Y = Z->left;
    Y->right = Z;
    Z->left = Y->right;   // wait, this is already wrong too
    updateHeight(Y);      // WRONG ORDER: Y uses Z's height, but Z hasn't been updated
    updateHeight(Z);
    return Y;
}

// CORRECT: update Z first (it's now a child of Y), then Y
AVLNode* rotateRight(AVLNode* Z) {
    AVLNode* Y  = Z->left;
    AVLNode* T2 = Y->right;
    Y->right = Z;          // pointer wiring
    Z->left  = T2;
    updateHeight(Z);       // Z is now lower — update its height first
    updateHeight(Y);       // Y depends on Z's updated height
    return Y;
}
```

### Pitfall 3: Not Returning New Root After Rotation

```cpp
// WRONG: doesn't propagate new root back to parent
void insert_wrong(AVLNode* node, int val) {
    if (val < node->val) insert_wrong(node->left, val);
    else                  insert_wrong(node->right, val);
    rebalance(node);   // rotations create a new root but we don't return it!
    // The parent still points to the old node, not the new root.
}

// CORRECT: return the potentially-new root up the call stack
AVLNode* insert(AVLNode* node, int val) {
    if (!node) return new AVLNode(val);
    if (val < node->val) node->left  = insert(node->left,  val);
    else                  node->right = insert(node->right, val);
    return rebalance(node);   // rebalance returns new root of this subtree
}
// And the caller: root_ = insert(root_, val);
```

### Pitfall 4: Off-by-One in BF Comparison for LR/RL Cases

```cpp
// The LR case (left-right): BF(left child) < 0 (strictly negative)
// The LL case: BF(left child) >= 0 (zero or positive)

// WRONG: using > instead of >= for LL check
if (bf(node->left) > 0) {  // misses the BF=0 case!
    return rotateRight(node);  // BF=0 should also be LL
}

// What does BF=0 of the left child mean?
// Left child has equal-height subtrees. The LL rotation is correct here.
// Using rotateRight is fine: the rotation balances the tree correctly
// for BF=0 just as for BF=+1.

// CORRECT:
if (bf(node->left) >= 0) {
    return rotateRight(node);   // LL: BF of left child is 0 or +1
} else {
    node->left = rotateLeft(node->left);  // LR: BF of left child is -1
    return rotateRight(node);
}
```

### Pitfall 5: Memory Leak on Deletion

```cpp
// WRONG: removing node without freeing memory
AVLNode* delete_wrong(AVLNode* node, int val) {
    // ... navigate to node ...
    if (!node->left) {
        return node->right;  // ← node is disconnected but NOT deleted!
        // Memory leak: node's memory is never freed.
    }
    // ...
}

// CORRECT: always delete the removed node
AVLNode* delete_(AVLNode* node, int val) {
    // ... navigate ...
    if (!node->left || !node->right) {
        AVLNode* child = node->left ? node->left : node->right;
        delete node;         // ← explicit deletion
        return child;
    }
    // Case 3: don't delete the node itself — copy successor's value
    AVLNode* succ = findMin_(node->right);
    node->val    = succ->val;
    node->right  = delete_(node->right, succ->val);
    // succ is deleted inside the recursive call above
    return rebalance(node);
}
```

### Pitfall 6: Not Rebalancing All the Way to Root on Deletion

```cpp
// WRONG: only rebalancing at the deletion point, not up the path
AVLNode* delete_partial(AVLNode* node, int val) {
    if (!node) return nullptr;
    if (val < node->val) {
        node->left = delete_partial(node->left, val);
        return rebalance(node);   // rebalances here ✓
    } else if (val > node->val) {
        node->right = delete_partial(node->right, val);
        // return node;  ← WRONG: no rebalancing! Just return node without rebalancing.
        return rebalance(node);  // CORRECT: must rebalance here too
    }
    // ... handle found case
}

// The key: rebalance(node) must be called at EVERY node on the path back to root.
// Unlike insertion (one rotation fixes everything), deletion may need rotations
// at multiple levels.
```

---

## 13. Comparison: AVL vs Red-Black vs BST vs Skip List

| Property | Plain BST | AVL Tree | Red-Black Tree | Skip List |
|---|---|---|---|---|
| Height guarantee | O(n) worst | **1.44 log₂(n)** | 2 log₂(n+1) | O(log n) expected |
| Search | O(n) worst | **O(log n)** | O(log n) | O(log n) expected |
| Insert | O(n) worst | O(log n) | O(log n) | O(log n) expected |
| Delete | O(n) worst | O(log n) | O(log n) | O(log n) expected |
| Rotations per insert | 0 | **1 max** | ≤ 2 | N/A |
| Rotations per delete | 0 | **O(log n) max** | ≤ 3 | N/A |
| Balance strictness | None | Strict (BF ≤ 1) | Relaxed (2× height) | Probabilistic |
| Lookup efficiency | Poor worst | **Best** (strictest balance) | Good | Good expected |
| Write efficiency | N/A | Moderate | **Best** (fewest rebalancings) | Good |
| Memory per node | 2 ptrs | 2 ptrs + height | 2 ptrs + color bit | ptrs × levels |
| Implementation complexity | Simple | Moderate | **Most complex** | Moderate |
| Used in | Simple demos | Read-heavy indexes | `std::map`, Linux | Redis sorted set |
| Best use case | Static data | Read-heavy dynamic | General-purpose | Concurrent access |

**When to choose AVL over Red-Black:**
```
Choose AVL when:
  ✓ Read-heavy workload (lookups >> modifications)
  ✓ Height matters for predictable worst-case latency
  ✓ You need the tightest possible height bound
  ✓ Simpler to implement correctly (4 rotation cases vs Red-Black's 6)

Choose Red-Black when:
  ✓ Write-heavy workload (std::map use case)
  ✓ Amortised performance matters more than worst-case
  ✓ Fewer memory writes per modification
  ✓ Standard library implementation (most libraries use Red-Black)
```

---

## 14. Self-Test Questions

1. **State the AVL property precisely. If a node has BF = +2, which subtree is taller? By how many levels?**

2. **There are four rotation cases. For each, state: (a) the sign of BF at the imbalanced node, (b) the sign of BF at the child in the heavier direction, (c) the rotation(s) applied.**

3. **Trace the right rotation on node Z where Z has left child Y, and Y has right child T2. Draw before and after. Show which heights are updated first and why.**

4. **Why does insertion require at most ONE rotation while deletion may require O(log n) rotations? What structural property differs between the two cases?**

5. **Insert [15, 10, 20, 8, 12, 16, 25, 11] into an empty AVL tree. Draw the tree after each insertion. Identify every rotation triggered.**

6. **Prove that the minimum number of nodes in an AVL tree of height h satisfies N(h) = N(h-1) + N(h-2) + 1. What sequence does this resemble? What does it imply about the height bound?**

7. **In the LR case, why must you left-rotate the left child BEFORE right-rotating the root? What goes wrong if you skip the first rotation and try to right-rotate the root directly?**

8. **AVL trees store a height field at each node. An alternative stores only the balance factor (2 bits). What are the trade-offs? Under what constraints would you prefer the 2-bit approach?**

9. **You implement an AVL tree and the `isValidAVL` check fails after a deletion. List five things you would check in your implementation. Which is the most common bug?**

10. **AVL trees are more strictly balanced than Red-Black trees. Why does this make them faster for lookups but slower for insertions? Give the exact worst-case rotation counts for both.**

---

## Quick Reference Card

```
AVL Tree — self-balancing BST with height ≤ 1.44 log₂(n). All ops O(log n) worst case.

Invariant: for every node, |height(left) - height(right)| ≤ 1.
  Balance Factor BF = height(left) - height(right) ∈ {-1, 0, +1}.
  After modification: BF = ±2 triggers rotation to restore balance.

Four rotation cases (triggered when |BF| = 2):
  BF(Z)=+2, BF(left)≥0:  LL → Right Rotation on Z
  BF(Z)=+2, BF(left)<0:  LR → Left on Z->left, then Right on Z
  BF(Z)=-2, BF(right)≤0: RR → Left Rotation on Z
  BF(Z)=-2, BF(right)>0: RL → Right on Z->right, then Left on Z

Right Rotation (LL):                Left Rotation (RR):
    Z        →      Y                   Z        →      Y
   / \              / \                / \              / \
  Y  T3            X   Z             T1   Y            Z   X
 / \                  / \               / \            / \
X  T2                T2  T3            T2  X           T1  T2

Height update in rotation: ALWAYS update the LOWER node (Z) before the UPPER node (Y).

Insertion: O(log n). At most 1 rotation needed.
  Call insert_() recursively. Call rebalance() on each node on the way back up.
  After the first rotation, height is restored — no further rotations needed.

Deletion: O(log n). Up to O(log n) rotations may be needed.
  Call delete_() recursively. Call rebalance() at EVERY node on the way back up.
  Each rotation may decrease the subtree height, propagating imbalance upward.

Height guarantee: h ≤ 1.44 × log₂(n+2) − 0.328
  Derived from Fibonacci: N(h) = N(h-1) + N(h-2) + 1, like Fibonacci numbers.

Critical implementation rules:
  1. Always return the new root from insert/delete/rebalance (rotations create new roots)
  2. updateHeight(Z) BEFORE updateHeight(Y) in rotations (Z is now lower)
  3. Call rebalance() at EVERY node on the path back to root (not just the imbalanced one)
  4. Use long long for BST validation bounds (handles INT_MIN/INT_MAX nodes)
  5. Free deleted nodes to avoid memory leaks

vs BST: same structure + height field + rebalancing → guaranteed O(log n)
vs Red-Black: stricter balance (better reads), more rotations (slower writes)
vs std::map: std::map uses Red-Black; AVL is better for read-heavy workloads

Pitfall checklist:
  ✗ updateHeight after (not before) reading BF → stale balance factor
  ✗ Wrong height update order in rotation → wrong heights propagate up
  ✗ Not returning new root → parent still points to old displaced node
  ✗ BF comparison off-by-one (≥ vs >) for LL/LR boundary
  ✗ Only rebalancing at deletion point (not full path) → multiple imbalances missed
  ✗ Not deleting node memory → memory leak
```

---

*Previous: [23 — Binary Search Tree](./23_binary_search_tree.md)*
*Next: [25 — Red-Black Tree](./25_red_black_tree.md)*
*See also: [B-Tree](./trees/b_tree.md) | [Splay Tree](./trees/splay_tree.md) | [AVL rotation visualiser](https://www.cs.usfca.edu/~galles/visualization/AVLtree.html)*

# Binary Tree

> **Curriculum position:** Trees → #1
> **Interview weight:** ★★★★★ Critical — the foundation of every tree problem; traversals, path problems, and structural queries appear in virtually every top-tier interview
> **Difficulty:** Beginner concept, deep in recursive reasoning and traversal patterns
> **Prerequisite:** [03 — Singly Linked List](./03_singly_linked_list.md) | [07 — Stack](./07_stack.md) | [09 — Queue](./09_queue.md)

---

## Table of Contents

1. [Intuition First — From Linear to Hierarchical](#1-intuition-first)
2. [Formal Definition and Terminology](#2-formal-definition-and-terminology)
3. [Internal Working — Node Structure and Pointer Model](#3-internal-working)
4. [Tree Properties and Relationships](#4-tree-properties)
5. [The Four Traversals — The Most Important Concept in Trees](#5-the-four-traversals)
6. [Time & Space Complexity](#6-time--space-complexity)
7. [Complete C++ Implementation](#7-complete-c-implementation)
8. [Core Operations — Visualised](#8-core-operations--visualised)
9. [Recursive Thinking — The Pattern Behind Every Tree Problem](#9-recursive-thinking)
10. [Common Patterns & Techniques](#10-common-patterns--techniques)
11. [Interview Problems](#11-interview-problems)
12. [Real-World Uses](#12-real-world-uses)
13. [Edge Cases & Pitfalls](#13-edge-cases--pitfalls)
14. [Binary Tree Variants at a Glance](#14-variants)
15. [Self-Test Questions](#15-self-test-questions)

---

## 1. Intuition First

Every data structure studied so far has been **linear** — elements arranged in a sequence, with each element having at most one predecessor and one successor. Arrays, linked lists, stacks, queues, deques, hash tables — all are fundamentally one-dimensional.

The real world is rarely one-dimensional. A company has an executive, who manages several VPs, each of whom manages several directors, each of whom manages several engineers. A file system has a root folder containing subfolders, each containing more subfolders and files. A sentence has a subject, which contains a noun phrase, which contains an adjective and a noun. All of these structures are **hierarchical** — and the binary tree is the simplest, most fundamental way to represent hierarchy computationally.

A **binary tree** is a structure where:
- Each element (node) holds a value
- Each node has at most **two children**: a left child and a right child
- There is exactly one **root** node — the topmost node with no parent
- Every non-root node has exactly one parent
- The structure is recursive — every subtree is itself a binary tree

The "binary" in binary tree means each node branches into at most two paths. This constraint — at most two children — is what makes binary trees analytically tractable. With two children per node, a tree of height `h` can hold up to `2^(h+1) - 1` nodes. A perfectly balanced binary tree with `n` nodes has height `O(log n)`. This logarithmic height is the source of the efficiency of binary search trees, heaps, segment trees, and every other tree-based structure in this curriculum.

Think of a binary tree as a family tree that only ever has at most two children per parent. Every algorithm you learn on binary trees — traversal, search, construction, serialisation — generalises to the more complex tree structures that follow.

---

## 2. Formal Definition and Terminology

### Definition

A binary tree is either:
- **Empty** (null), or
- A **node** containing: a value, a left subtree (another binary tree), and a right subtree (another binary tree)

This recursive definition is not just elegant — it is the source of every recursive algorithm on trees.

### Terminology

```
                        ┌───────────────────────────────┐
                        │             1                 │  ← root
                        │           /   \               │
                        │          2     3              │  ← level 1
                        │         / \   / \             │
                        │        4   5 6   7            │  ← level 2
                        │       / \         \           │
                        │      8   9         10         │  ← level 3
                        └───────────────────────────────┘

Node:          Each circle is a node. Holds a value and up to 2 child pointers.
Root:          Node 1. The unique node with no parent.
Leaf:          Nodes 5, 6, 8, 9, 10. Nodes with no children.
Internal node: Nodes 1, 2, 3, 4, 7. Nodes with at least one child.
Parent:        Node 2 is the parent of nodes 4 and 5.
Child:         Nodes 4 and 5 are children of node 2.
Sibling:       Nodes 4 and 5 are siblings (same parent).
Ancestor:      Node 1 is an ancestor of every other node.
Descendant:    Every node is a descendant of node 1.

Height of a node:    Longest path from the node DOWN to any leaf.
                     height(8) = 0, height(4) = 1, height(2) = 2, height(1) = 3
Depth of a node:     Length of path from ROOT DOWN to the node.
                     depth(1) = 0, depth(2) = 1, depth(4) = 2, depth(8) = 3
Level:               Level = depth (sometimes level starts at 1 instead of 0).
Height of tree:      Height of the root = 3 in this example.
Size:                Total number of nodes = 10.

Edge:          The connection between a parent and a child.
Path:          Sequence of nodes connected by edges (e.g., 1→2→4→8).
Subtree:       A node and all its descendants (e.g., subtree rooted at 2: {2,4,5,8,9}).
```

### Special Types of Binary Trees

```
Full binary tree:     Every node has exactly 0 or 2 children (never 1).
Complete binary tree: All levels except possibly the last are fully filled;
                      last level is filled left to right.
Perfect binary tree:  All internal nodes have 2 children; all leaves at same level.
                      A perfect tree of height h has exactly 2^(h+1) - 1 nodes.
Balanced binary tree: Height is O(log n). The left and right subtree heights
                      of every node differ by at most 1.
Degenerate tree:      Every node has at most 1 child — essentially a linked list.
                      Height = n - 1. Worst case for most tree algorithms.
```

---

## 3. Internal Working — Node Structure and Pointer Model

### The Node

```cpp
struct TreeNode {
    int       val;
    TreeNode* left;
    TreeNode* right;

    TreeNode(int v) : val(v), left(nullptr), right(nullptr) {}
    TreeNode(int v, TreeNode* l, TreeNode* r) : val(v), left(l), right(r) {}
};
```

Each node is independently heap-allocated. The tree is navigated entirely through pointers. There is no implicit ordering (unlike BST), no balance guarantee (unlike AVL), and no structural constraint (unlike heap). A binary tree is the most general tree structure.

### Memory Layout

```
Node at address 0xA000:
┌───────────────────────────────────┐
│  val   (4 bytes)  │  [0xA000+0]  │
│  left* (8 bytes)  │  [0xA000+8]  │  → points to left child node or nullptr
│  right*(8 bytes)  │  [0xA000+16] │  → points to right child node or nullptr
└───────────────────────────────────┘

Heap layout (non-contiguous — each node is a separate allocation):
  Root → [1]  at 0xA000
         left  → [2] at 0xB100
                 left  → [4] at 0xC200
                 right → [5] at 0xD300
         right → [3] at 0xE400
```

Unlike arrays (contiguous), tree nodes are scattered across the heap. Each pointer dereference is a potential cache miss — this is the primary performance disadvantage of pointer-based trees vs flat array representations (heaps use a flat array for this reason).

### The Tree Object

Most implementations do not have a separate "tree" class — they just use the root pointer directly. The convention `TreeNode* root = nullptr` represents an empty tree.

```cpp
// A tree is just its root pointer. null = empty tree.
TreeNode* root = nullptr;

// Build:  1
//        / \
//       2   3
root = new TreeNode(1);
root->left  = new TreeNode(2);
root->right = new TreeNode(3);
```

---

## 4. Tree Properties

### Height and Size Relationships

```
For a binary tree with n nodes:
  Minimum height (perfect/complete):  h_min = ⌊log₂(n)⌋
  Maximum height (degenerate/skewed): h_max = n - 1

For a binary tree of height h:
  Minimum nodes (left-skewed line):   n_min = h + 1
  Maximum nodes (perfect tree):       n_max = 2^(h+1) - 1

Key relationship: height h and node count n are related by
  h ≥ ⌊log₂(n)⌋  (for any binary tree)
  h = O(log n)    (for balanced binary trees)
  h = O(n)        (for degenerate binary trees)
```

### Level-by-Level Node Count

```
Level 0 (root):   at most 2^0 = 1 node
Level 1:          at most 2^1 = 2 nodes
Level 2:          at most 2^2 = 4 nodes
Level k:          at most 2^k nodes
Level h (last):   at most 2^h nodes

Total nodes in perfect tree of height h:
  1 + 2 + 4 + ... + 2^h = 2^(h+1) - 1

This geometric series is why binary trees are O(log n) height when balanced.
```

### Leaf Count and Internal Node Relationships

```
For a FULL binary tree (every node has 0 or 2 children):
  If L = number of leaves, I = number of internal nodes:
  L = I + 1
  
  Proof: Total edges = n - 1 (n nodes, connected). 
         Each internal node contributes 2 edges (two children).
         Total edges = 2I. So 2I = n - 1 = L + I - 1, giving L = I + 1.
```

---

## 5. The Four Traversals — The Most Important Concept in Trees

Traversal is visiting every node exactly once in a specific order. There are four canonical traversals of a binary tree. Mastering them is the single most important skill for tree interview problems.

### The Three Depth-First Traversals

All three use the same recursive structure but differ in when the root is visited relative to its subtrees:

```
Inorder   (LEFT → ROOT → RIGHT): visits left subtree, then root, then right subtree.
Preorder  (ROOT → LEFT → RIGHT): visits root, then left subtree, then right subtree.
Postorder (LEFT → RIGHT → ROOT): visits left subtree, then right subtree, then root.
```

```
Tree:           1
               / \
              2   3
             / \
            4   5

Inorder   (L→N→R): 4, 2, 5, 1, 3    ← For BST: gives SORTED order!
Preorder  (N→L→R): 1, 2, 4, 5, 3    ← Useful for serialisation, tree copying.
Postorder (L→R→N): 4, 5, 2, 3, 1    ← Useful for tree deletion, evaluation.
```

### Breadth-First Traversal (Level Order)

Visits nodes level by level, left to right within each level. Uses a queue.

```
Level order: 1, 2, 3, 4, 5    ← processes nodes nearest to root first.
```

### Memory Aid — Which Traversal for What?

```
Inorder   → BST sorted output, range queries
Preorder  → Create a copy, serialise, prefix expression evaluation
Postorder → Delete a tree, evaluate expression trees (children before parent)
Level order → BFS, shortest path in unweighted trees, level-by-level processing
```

### Iterative vs Recursive

Every recursive traversal has an iterative equivalent using an explicit stack (for DFS) or queue (for BFS). This is a critical interview skill — interviewers often ask for iterative implementations explicitly.

```
Why iterative?
  1. Avoids O(h) call stack overhead (important for deep trees, h=O(n) in worst case)
  2. Demonstrates understanding of the implicit call stack
  3. Required when the tree is too deep for recursion (stack overflow risk)
  4. Necessary for resumable/generator-style traversal
```

---

## 6. Time & Space Complexity

### Traversal Complexities

| Traversal | Time | Space (call stack/queue) | Notes |
|---|---|---|---|
| Inorder (recursive) | O(n) | O(h) | h = tree height |
| Preorder (recursive) | O(n) | O(h) | h = log n for balanced |
| Postorder (recursive) | O(n) | O(h) | h = n-1 for skewed |
| Level order (BFS) | O(n) | O(w) | w = max width ≤ n/2 |
| All traversals | O(n) | O(n) worst case | Degenerate tree |

### Operation Complexities (Unordered Binary Tree)

| Operation | Time | Space | Notes |
|---|---|---|---|
| Search by value | O(n) | O(h) | Must traverse all nodes (no ordering) |
| Find height | O(n) | O(h) | Must visit every node |
| Count nodes | O(n) | O(h) | Must visit every node |
| Insert (arbitrary) | O(1) if node given | O(1) | Just set a pointer |
| Insert (BFS location) | O(n) | O(n) | Find first empty slot |
| Delete | O(n) | O(h) | Must find node first |
| Find LCA | O(n) | O(h) | Two-pass or recursive |
| Check symmetry | O(n) | O(h) | Compare left and right subtrees |
| Serialise | O(n) | O(n) | Visit and record all nodes |
| Deserialise | O(n) | O(n) | Build tree from sequence |

### Space for Balanced vs Skewed Trees

```
For n nodes:
  Balanced tree: height h = O(log n)
    Recursive call stack: O(log n) — fits comfortably
    Level order queue: O(n/2) = O(n) — bottom level

  Degenerate tree (linked list): height h = n - 1
    Recursive call stack: O(n) — may cause stack overflow for large n
    Level order queue: O(1) — only 1 node per level
```

---

## 7. Complete C++ Implementation

### Node Definition

```cpp
#include <iostream>
#include <queue>
#include <stack>
#include <vector>
#include <string>
#include <optional>
#include <functional>
#include <climits>
using namespace std;

struct TreeNode {
    int       val;
    TreeNode* left;
    TreeNode* right;

    explicit TreeNode(int v = 0, TreeNode* l = nullptr, TreeNode* r = nullptr)
        : val(v), left(l), right(r) {}
};
```

### Building a Tree

```cpp
// Build from level-order array (INT_MIN = null node)
// e.g., {1, 2, 3, INT_MIN, 5, 6, 7} → tree with root 1
TreeNode* buildFromLevelOrder(const vector<int>& vals) {
    if (vals.empty() || vals[0] == INT_MIN) return nullptr;

    TreeNode* root = new TreeNode(vals[0]);
    queue<TreeNode*> q;
    q.push(root);
    int i = 1;

    while (!q.empty() && i < (int)vals.size()) {
        TreeNode* node = q.front(); q.pop();

        if (i < (int)vals.size() && vals[i] != INT_MIN) {
            node->left = new TreeNode(vals[i]);
            q.push(node->left);
        }
        i++;

        if (i < (int)vals.size() && vals[i] != INT_MIN) {
            node->right = new TreeNode(vals[i]);
            q.push(node->right);
        }
        i++;
    }
    return root;
}
```

### The Four Traversals — Recursive

```cpp
// ── Inorder: Left → Root → Right ─────────────────────────────────────────────
void inorder(TreeNode* root, vector<int>& result) {
    if (!root) return;
    inorder(root->left, result);
    result.push_back(root->val);
    inorder(root->right, result);
}

// ── Preorder: Root → Left → Right ────────────────────────────────────────────
void preorder(TreeNode* root, vector<int>& result) {
    if (!root) return;
    result.push_back(root->val);
    preorder(root->left, result);
    preorder(root->right, result);
}

// ── Postorder: Left → Right → Root ───────────────────────────────────────────
void postorder(TreeNode* root, vector<int>& result) {
    if (!root) return;
    postorder(root->left, result);
    postorder(root->right, result);
    result.push_back(root->val);
}

// ── Level Order: BFS with queue ───────────────────────────────────────────────
vector<vector<int>> levelOrder(TreeNode* root) {
    vector<vector<int>> result;
    if (!root) return result;

    queue<TreeNode*> q;
    q.push(root);

    while (!q.empty()) {
        int levelSize = (int)q.size();          // snapshot current level size
        vector<int> level;
        level.reserve(levelSize);

        for (int i = 0; i < levelSize; i++) {
            TreeNode* node = q.front(); q.pop();
            level.push_back(node->val);
            if (node->left)  q.push(node->left);
            if (node->right) q.push(node->right);
        }
        result.push_back(move(level));
    }
    return result;
}
```

### The Four Traversals — Iterative

```cpp
// ── Iterative Inorder ─────────────────────────────────────────────────────────
// Key: push nodes as we go left; on popping, process and go right.
vector<int> inorderIterative(TreeNode* root) {
    vector<int> result;
    stack<TreeNode*> stk;
    TreeNode* curr = root;

    while (curr || !stk.empty()) {
        // Go as far left as possible, pushing each node
        while (curr) {
            stk.push(curr);
            curr = curr->left;
        }
        // Backtrack: process the node, then go right
        curr = stk.top(); stk.pop();
        result.push_back(curr->val);
        curr = curr->right;
    }
    return result;
}

// ── Iterative Preorder ────────────────────────────────────────────────────────
// Key: process on push; push RIGHT before LEFT (LIFO gives left-first).
vector<int> preorderIterative(TreeNode* root) {
    if (!root) return {};
    vector<int> result;
    stack<TreeNode*> stk;
    stk.push(root);

    while (!stk.empty()) {
        TreeNode* node = stk.top(); stk.pop();
        result.push_back(node->val);          // process ROOT
        if (node->right) stk.push(node->right);  // push RIGHT first
        if (node->left)  stk.push(node->left);   // push LEFT second (processes first)
    }
    return result;
}

// ── Iterative Postorder ───────────────────────────────────────────────────────
// Strategy 1: reverse of a modified preorder (root→right→left reversed = left→right→root)
vector<int> postorderIterative(TreeNode* root) {
    if (!root) return {};
    vector<int> result;
    stack<TreeNode*> stk;
    stk.push(root);

    while (!stk.empty()) {
        TreeNode* node = stk.top(); stk.pop();
        result.push_back(node->val);
        if (node->left)  stk.push(node->left);   // push LEFT first
        if (node->right) stk.push(node->right);  // push RIGHT second
    }
    reverse(result.begin(), result.end());  // reverse gives postorder
    return result;
}

// Strategy 2: single-stack with state tracking (more principled)
vector<int> postorderIterative2(TreeNode* root) {
    vector<int> result;
    stack<TreeNode*> stk;
    TreeNode* prev = nullptr;   // tracks last processed node
    TreeNode* curr = root;

    while (curr || !stk.empty()) {
        while (curr) {
            stk.push(curr);
            curr = curr->left;
        }
        curr = stk.top();

        // If right child exists and wasn't just processed, go right
        if (curr->right && curr->right != prev) {
            curr = curr->right;
        } else {
            // Both children processed (or don't exist) → process this node
            result.push_back(curr->val);
            stk.pop();
            prev = curr;
            curr = nullptr;
        }
    }
    return result;
}
```

### Core Tree Functions

```cpp
// ── Height ────────────────────────────────────────────────────────────────────
// Height = length of the longest path from node to a leaf.
// Null nodes have height -1 (so a single node has height 0).
int height(TreeNode* root) {
    if (!root) return -1;
    return 1 + max(height(root->left), height(root->right));
}

// ── Size (count nodes) ────────────────────────────────────────────────────────
int size(TreeNode* root) {
    if (!root) return 0;
    return 1 + size(root->left) + size(root->right);
}

// ── Depth of a specific node ──────────────────────────────────────────────────
int depth(TreeNode* root, int target, int d = 0) {
    if (!root) return -1;
    if (root->val == target) return d;
    int left = depth(root->left, target, d + 1);
    if (left != -1) return left;
    return depth(root->right, target, d + 1);
}

// ── Is Balanced? ──────────────────────────────────────────────────────────────
// Returns height if balanced, -2 as a sentinel for "unbalanced"
int checkBalanced(TreeNode* root) {
    if (!root) return -1;
    int lh = checkBalanced(root->left);
    int rh = checkBalanced(root->right);
    if (lh == -2 || rh == -2) return -2;              // propagate imbalance
    if (abs(lh - rh) > 1)     return -2;              // this node is unbalanced
    return 1 + max(lh, rh);                           // return height
}
bool isBalanced(TreeNode* root) { return checkBalanced(root) != -2; }

// ── Mirror / Invert Tree ──────────────────────────────────────────────────────
TreeNode* mirror(TreeNode* root) {
    if (!root) return nullptr;
    swap(root->left, root->right);
    mirror(root->left);
    mirror(root->right);
    return root;
}

// ── Is Symmetric? ─────────────────────────────────────────────────────────────
bool isMirror(TreeNode* left, TreeNode* right) {
    if (!left && !right) return true;
    if (!left || !right) return false;
    return left->val == right->val
        && isMirror(left->left,  right->right)
        && isMirror(left->right, right->left);
}
bool isSymmetric(TreeNode* root) {
    return !root || isMirror(root->left, root->right);
}

// ── Lowest Common Ancestor (LCA) ──────────────────────────────────────────────
// For a general binary tree (not BST). Returns LCA node.
TreeNode* lca(TreeNode* root, TreeNode* p, TreeNode* q) {
    if (!root || root == p || root == q) return root;
    TreeNode* left  = lca(root->left,  p, q);
    TreeNode* right = lca(root->right, p, q);
    if (left && right) return root;   // p and q are in different subtrees
    return left ? left : right;       // both in same subtree
}

// ── Serialise / Deserialise ───────────────────────────────────────────────────
// Preorder traversal with null markers.
string serialize(TreeNode* root) {
    if (!root) return "null,";
    return to_string(root->val) + ","
         + serialize(root->left)
         + serialize(root->right);
}

TreeNode* deserializeHelper(stringstream& ss) {
    string token;
    getline(ss, token, ',');
    if (token == "null") return nullptr;
    TreeNode* node = new TreeNode(stoi(token));
    node->left  = deserializeHelper(ss);
    node->right = deserializeHelper(ss);
    return node;
}
TreeNode* deserialize(const string& data) {
    stringstream ss(data);
    return deserializeHelper(ss);
}

// ── Memory cleanup ────────────────────────────────────────────────────────────
void deleteTree(TreeNode* root) {
    if (!root) return;
    deleteTree(root->left);
    deleteTree(root->right);
    delete root;
}

// ── Print tree (sideways, right branch on top) ────────────────────────────────
void printTree(TreeNode* root, string prefix = "", bool isLeft = true) {
    if (!root) return;
    printTree(root->right, prefix + (isLeft ? "│   " : "    "), false);
    cout << prefix << (isLeft ? "└── " : "┌── ") << root->val << "\n";
    printTree(root->left,  prefix + (isLeft ? "    " : "│   "), true);
}
```

---

## 8. Core Operations — Visualised

### The Four Traversals on the Same Tree

```
Tree:           1
               / \
              2   3
             / \   \
            4   5   6

Inorder   (L→N→R): 4 → 2 → 5 → 1 → 3 → 6

  Visit left subtree of 1:
    Visit left subtree of 2:
      Visit left subtree of 4: (empty)
      Visit 4 → output 4
      Visit right subtree of 4: (empty)
    Visit 2 → output 2
    Visit right subtree of 2:
      Visit left subtree of 5: (empty)
      Visit 5 → output 5
      Visit right subtree of 5: (empty)
  Visit 1 → output 1
  Visit right subtree of 1:
    Visit left subtree of 3: (empty)
    Visit 3 → output 3
    Visit right subtree of 3:
      Visit left subtree of 6: (empty)
      Visit 6 → output 6
      Visit right subtree of 6: (empty)

Result: [4, 2, 5, 1, 3, 6]

Preorder  (N→L→R): 1, 2, 4, 5, 3, 6
  Output 1, recurse left (output 2, recurse left (output 4), recurse right (output 5)),
  recurse right (output 3, recurse right (output 6))

Postorder (L→R→N): 4, 5, 2, 6, 3, 1
  Recurse left of 1 → recurse left of 2 → recurse left of 4 → empty, recurse right → empty
  Output 4. Back at 2: recurse right → output 5. Output 2.
  Back at 1: recurse right of 1 → output 6 (via 3's right), output 3.
  Output 1.

Level order:         1 | 2 3 | 4 5 6
  Enqueue 1. Process: output 1, enqueue 2, 3.
  Process 2: output 2, enqueue 4, 5. Process 3: output 3, enqueue 6.
  Process 4: output 4. Process 5: output 5. Process 6: output 6.
  Result: [[1], [2,3], [4,5,6]]
```

### Iterative Inorder — Call Stack Trace

```
Tree: 1 → left → 2 → left → 4 (leaf)

Iteration 1: curr=1. Push 1. Go left. curr=2. Push 2. Go left. curr=4. Push 4. Go left. curr=null.
Iteration 2: Pop 4. Output 4. curr = 4->right = null. (No right child)
Iteration 3: Pop 2. Output 2. curr = 2->right = 5.
Iteration 4: curr=5. Push 5. Go left. curr=null.
             Pop 5. Output 5. curr = 5->right = null.
Iteration 5: Pop 1. Output 1. curr = 1->right = 3.
Iteration 6: curr=3. Push 3. Go left. curr=null.
             Pop 3. Output 3. curr = 3->right = 6.
Iteration 7: curr=6. Push 6. Go left. curr=null.
             Pop 6. Output 6. curr = null.
Done. Stack empty. Result: [4, 2, 5, 1, 3, 6] ✓
```

### Height Computation — Recursive Call Tree

```
height(1):
  lh = height(2):
    lh = height(4): lh=height(null)=-1, rh=height(null)=-1. return 0.
    rh = height(5): similarly return 0.
    return 1 + max(0,0) = 1.
  rh = height(3):
    lh = height(null) = -1.
    rh = height(6): return 0.
    return 1 + max(-1, 0) = 1.
  return 1 + max(1, 1) = 2.

Height = 2. ✓ (longest path: 1→2→4 or 1→2→5 or 1→3→6)
```

---

## 9. Recursive Thinking — The Pattern Behind Every Tree Problem

The most important meta-skill for tree problems is recognising and applying the recursive structure. Every binary tree is either empty or a root node with two binary tree subtrees. This gives a universal template.

### The Universal Tree Recursion Template

```
solve(root):
  // Base case: empty tree
  if root is null: return base_value

  // Recursive cases: get answers from subtrees
  left_result  = solve(root.left)
  right_result = solve(root.right)

  // Combine: derive answer for this subtree from root + subtree answers
  return combine(root.val, left_result, right_result)
```

This template solves:
- Height: `combine = 1 + max(left_h, right_h)`
- Size: `combine = 1 + left_size + right_size`
- Sum of all values: `combine = root.val + left_sum + right_sum`
- Is symmetric: `combine = is_mirror(left, right)`
- Diameter: `combine = max(left_h + right_h + 2, left_diam, right_diam)`

### The "Return Multiple Values" Pattern

Many tree problems require returning more than one value from the recursion. Use a struct or pair:

```cpp
// Problem: diameter of binary tree
// For each subtree, we need BOTH: the diameter of the subtree AND the max path height
// to pass up to the parent.
pair<int,int> diameterHelper(TreeNode* root) {
    // returns {height, diameter}
    if (!root) return {-1, 0};

    auto [lh, ld] = diameterHelper(root->left);
    auto [rh, rd] = diameterHelper(root->right);

    int height   = 1 + max(lh, rh);
    int diameter = max({ld, rd, lh + rh + 2});  // path through root = lh+rh+2
    return {height, diameter};
}

int diameterOfBinaryTree(TreeNode* root) {
    return diameterHelper(root).second;
}
```

### The "Global Variable" Pattern

When the answer involves paths that don't necessarily pass through the root, use a global variable updated during traversal:

```cpp
int maxPathSum(TreeNode* root) {
    int globalMax = INT_MIN;

    function<int(TreeNode*)> dfs = [&](TreeNode* node) -> int {
        if (!node) return 0;
        int leftGain  = max(0, dfs(node->left));   // take path if positive
        int rightGain = max(0, dfs(node->right));

        // Path through this node = left + node + right
        globalMax = max(globalMax, node->val + leftGain + rightGain);

        // Return: max gain continuing UPWARD (can only go one direction from here)
        return node->val + max(leftGain, rightGain);
    };

    dfs(root);
    return globalMax;
}
```

### Recognising Recursive Substructure

The key question when approaching a tree problem: **"If I know the answer for both subtrees, can I compute the answer for the full tree?"**

```
If YES → bottom-up recursion works. Return value carries information upward.
If NO  → may need top-down (pass information DOWN via parameters).

Top-down pattern: pass running state into recursive calls.
// Example: path sum — pass remaining target down
bool hasPathSum(TreeNode* root, int remaining) {
    if (!root) return false;
    if (!root->left && !root->right) return root->val == remaining;
    return hasPathSum(root->left,  remaining - root->val)
        || hasPathSum(root->right, remaining - root->val);
}
```

---

## 10. Common Patterns & Techniques

### Pattern 1: Path Problems (Root to Leaf)

```cpp
// All root-to-leaf paths
void allPaths(TreeNode* root, vector<int>& path, vector<vector<int>>& result) {
    if (!root) return;
    path.push_back(root->val);

    if (!root->left && !root->right) {
        result.push_back(path);  // leaf reached: record path
    } else {
        allPaths(root->left,  path, result);
        allPaths(root->right, path, result);
    }

    path.pop_back();  // ← backtrack: remove current node before returning
}
// The push + recurse + pop_back pattern is the backtracking template.
```

### Pattern 2: Counting with Accumulation

```cpp
// Count nodes at depth exactly d
int countAtDepth(TreeNode* root, int d) {
    if (!root) return 0;
    if (d == 0) return 1;
    return countAtDepth(root->left, d-1) + countAtDepth(root->right, d-1);
}

// Count nodes satisfying a condition
int countMatching(TreeNode* root, function<bool(int)> pred) {
    if (!root) return 0;
    return (pred(root->val) ? 1 : 0)
         + countMatching(root->left, pred)
         + countMatching(root->right, pred);
}
```

### Pattern 3: Structural Queries

```cpp
// Does tree contain a value?
bool contains(TreeNode* root, int val) {
    if (!root) return false;
    if (root->val == val) return true;
    return contains(root->left, val) || contains(root->right, val);
}

// Are two trees structurally identical (same shape, same values)?
bool isSameTree(TreeNode* p, TreeNode* q) {
    if (!p && !q) return true;
    if (!p || !q) return false;
    return p->val == q->val
        && isSameTree(p->left, q->left)
        && isSameTree(p->right, q->right);
}

// Is tree t a subtree of tree s?
bool isSubtree(TreeNode* s, TreeNode* t) {
    if (!s) return false;
    if (isSameTree(s, t)) return true;
    return isSubtree(s->left, t) || isSubtree(s->right, t);
}
```

### Pattern 4: Morris Traversal — O(1) Space Inorder

```cpp
// Inorder traversal WITHOUT a stack or recursion. O(1) extra space.
// Uses the "threaded binary tree" trick: temporarily link nodes using null right pointers.
vector<int> morrisInorder(TreeNode* root) {
    vector<int> result;
    TreeNode* curr = root;

    while (curr) {
        if (!curr->left) {
            // No left child: process this node and go right
            result.push_back(curr->val);
            curr = curr->right;
        } else {
            // Find the inorder predecessor (rightmost node in left subtree)
            TreeNode* pred = curr->left;
            while (pred->right && pred->right != curr) pred = pred->right;

            if (!pred->right) {
                // First visit: create thread (temporary link back to curr)
                pred->right = curr;
                curr = curr->left;
            } else {
                // Second visit: remove thread, process curr, go right
                pred->right = nullptr;
                result.push_back(curr->val);
                curr = curr->right;
            }
        }
    }
    return result;
}
// Space: O(1) — no stack. Time: O(n) — each node visited at most twice.
// Used when stack space is a hard constraint.
```

---

## 11. Interview Problems

### Problem 1: Binary Tree Maximum Path Sum

**Problem:** A **path** in a binary tree is a sequence of nodes where each pair of adjacent nodes has an edge between them. A node can appear in the path at most once. The path does not need to pass through the root. Find the path with the maximum sum.

**Example:**
```
Tree:    -10
         /  \
        9   20
           /  \
          15   7

Answer: 42 (path 15 → 20 → 7)
```

**Thought process:**

> "For each node, the maximum path through it could be: left branch + node + right branch (a 'V'-shape). I need to update a global maximum with this V-path at every node. But when returning to the parent, I can only extend ONE branch (a path cannot fork). So the return value is: node + max(left_gain, right_gain), where gains are clamped at 0 (skip negative branches)."

```cpp
// Time: O(n) | Space: O(h) for recursion
int maxPathSum(TreeNode* root) {
    int result = INT_MIN;

    function<int(TreeNode*)> dfs = [&](TreeNode* node) -> int {
        if (!node) return 0;

        // max(0, ...) discards negative subtrees (better to not take them)
        int leftGain  = max(0, dfs(node->left));
        int rightGain = max(0, dfs(node->right));

        // Path through this node in V-shape: leftGain + node + rightGain
        result = max(result, node->val + leftGain + rightGain);

        // Return max gain we can contribute to parent: pick the better branch
        return node->val + max(leftGain, rightGain);
    };

    dfs(root);
    return result;
}

/*
Trace on [-10, 9, 20, null, null, 15, 7]:

dfs(9):  left=0, right=0. V-path=9. result=9. return 9.
dfs(15): left=0, right=0. V-path=15. result=15. return 15.
dfs(7):  left=0, right=0. V-path=7. result=15. return 7.
dfs(20): leftGain=15, rightGain=7. V-path=15+20+7=42. result=42. return 20+15=35.
dfs(-10): leftGain=max(0,9)=9, rightGain=max(0,35)=35. V-path=-10+9+35=34<42.
          result=42. return -10+35=25.

Answer: 42 ✓
*/
```

**Edge cases:**
- All negative values: the answer is the single largest node (V-shape with no branches, as both gains are clamped to 0)
- Single node: answer is that node's value
- Path must contain at least one node: `result = INT_MIN` handles the single negative node case correctly

---

### Problem 2: Construct Binary Tree from Preorder and Inorder Traversal

**Problem:** Given two integer arrays `preorder` and `inorder` representing the preorder and inorder traversal of the same binary tree, construct and return the binary tree.

**Example:**
```
preorder = [3, 9, 20, 15, 7]
inorder  = [9, 3, 15, 20, 7]

Tree:    3
        / \
       9  20
          / \
         15   7
```

**Thought process — the key insight:**

> "In preorder, the FIRST element is always the root. In inorder, the root splits the array: everything to the LEFT of the root in inorder is in the LEFT subtree, everything to the RIGHT is in the RIGHT subtree. Recursively apply this: the next `left_size` elements in preorder form the left subtree's preorder; the remaining form the right subtree's preorder."

```cpp
// Time: O(n) with hash map | O(n²) without | Space: O(n) for hash map + O(h) recursion
TreeNode* buildTree(vector<int>& preorder, vector<int>& inorder) {
    // Map from value to its index in inorder — O(1) root lookup
    unordered_map<int,int> inorderIdx;
    for (int i = 0; i < (int)inorder.size(); i++) inorderIdx[inorder[i]] = i;

    function<TreeNode*(int,int,int)> build = [&](int preStart, int inStart, int inEnd) -> TreeNode* {
        if (preStart >= (int)preorder.size() || inStart > inEnd) return nullptr;

        int rootVal   = preorder[preStart];
        int rootInIdx = inorderIdx[rootVal];
        int leftSize  = rootInIdx - inStart;  // number of nodes in left subtree

        TreeNode* root  = new TreeNode(rootVal);
        root->left  = build(preStart + 1,             inStart,         rootInIdx - 1);
        root->right = build(preStart + 1 + leftSize,  rootInIdx + 1,  inEnd);
        return root;
    };

    return build(0, 0, (int)inorder.size() - 1);
}

/*
Trace: preorder=[3,9,20,15,7], inorder=[9,3,15,20,7]
  inorderIdx: {9:0, 3:1, 15:2, 20:3, 7:4}

build(preStart=0, inStart=0, inEnd=4):
  rootVal=3, rootInIdx=1, leftSize=1.
  left  = build(1, 0, 0):   rootVal=9, leftSize=0. No children. return Node(9).
  right = build(2, 2, 4):   rootVal=20, rootInIdx=3, leftSize=1.
    left  = build(3, 2, 2): rootVal=15. return Node(15).
    right = build(4, 4, 4): rootVal=7.  return Node(7).
    return Node(20, left=15, right=7).
  return Node(3, left=9, right=Node(20,...)) ✓
*/
```

**Why the hash map?** Without it, finding `rootInIdx` requires O(n) search, making the overall algorithm O(n²). With the hash map, each `build` call is O(1), giving O(n) total.

**Variant — from Postorder and Inorder:** The last element of postorder is the root. Same logic applies; just read from the end of postorder.

**Edge cases:**
- Single element: preorder=[5], inorder=[5] → return Node(5)
- Left-skewed: all elements are in the left subtree; leftSize = inEnd - inStart each time

---

### Problem 3: Lowest Common Ancestor (LCA) — Binary Tree

**Problem:** Given a binary tree and two nodes `p` and `q`, find their lowest common ancestor. The LCA is defined as the deepest node that is an ancestor of both `p` and `q`. A node is allowed to be an ancestor of itself.

**Example:**
```
Tree:      3
         /   \
        5     1
       / \   / \
      6   2 0   8
         / \
        7   4

LCA(5, 1) = 3  (only the root is an ancestor of both)
LCA(5, 4) = 5  (5 is an ancestor of itself; 4 is in its subtree)
```

**Thought process:**

> "Post-order recursion: for each node, ask its left and right subtrees 'do you contain p or q?' If both subtrees return something non-null, this node is the LCA. If only one returns something, pass that result upward — it means both p and q are in that subtree."

```cpp
// Time: O(n) | Space: O(h)
// Key insight: base case handles both "found target" AND "null" simultaneously.
TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, TreeNode* q) {
    // Base case: if root is null OR root IS p OR root IS q
    // In all these cases, return root itself.
    // - null: nothing found in this branch
    // - root==p or root==q: found a target; return it (the LCA might be higher)
    if (!root || root == p || root == q) return root;

    // Recursively search both subtrees
    TreeNode* left  = lowestCommonAncestor(root->left,  p, q);
    TreeNode* right = lowestCommonAncestor(root->right, p, q);

    // If both subtrees found something → root is the LCA
    if (left && right) return root;

    // Otherwise, the LCA is in whichever subtree found something
    return left ? left : right;
}

/*
Trace LCA(5, 4) on the tree above:

lca(3, 5, 4):
  left  = lca(5, 5, 4):
    root == p (5). Return 5 immediately.
    [We don't recurse further — we'll still find 4 via the right call]
    Actually: the function returns 5 because root==p.
    This is correct: 5 IS an ancestor of 4.
  right = lca(1, 5, 4):
    left  = lca(0,...) = null (neither 5 nor 4 found)
    right = lca(8,...) = null
    return null (neither p nor q found in subtree rooted at 1)
  left=5, right=null. Return left=5.
  
Answer: Node(5) ✓

Trace LCA(5, 1):
  lca(3, 5, 1):
    left  = lca(5, 5, 1): root==5==p. Return 5.
    right = lca(1, 5, 1): root==1==q. Return 1.
    left=5, right=1. Both non-null → return root=3. ✓
*/
```

**Why is `root == p || root == q` a base case?** Because if one of our targets IS the current node, this node must be the LCA (either the other target is in its subtree — in which case this node IS the deepest common ancestor — or the other target is elsewhere, but that case is handled by the parent returning the deeper non-null result).

**Edge cases:**
- p or q is the root: LCA is the root (handled by `root == p || root == q`)
- p is an ancestor of q: the recursive call returns p (not q's location), correctly identifying p as the LCA
- p or q not in tree: the function returns the existing one or null — you'd need to add explicit existence checks for stricter requirements

---

## 12. Real-World Uses

| Domain | Use Case | Tree Detail |
|---|---|---|
| **Compilers** | Abstract Syntax Tree (AST) | Every program is a binary/n-ary tree; traversal = code generation |
| **File systems** | Directory hierarchy | File/folder structure; BFS = level listing |
| **Databases** | Query plan trees | SQL query optimiser represents execution plans as trees |
| **XML/HTML/JSON** | DOM (Document Object Model) | Every HTML page is a tree; CSS selectors traverse it |
| **Game development** | Scene graph, decision trees | Game objects arranged hierarchically; AI decision trees |
| **Computer graphics** | BSP (Binary Space Partitioning) trees | Spatial subdivision for rendering visibility |
| **Networking** | Routing trees, spanning trees | Network topology represented as trees |
| **Machine learning** | Decision trees, gradient boosting | Each split in a decision tree is a binary tree node |
| **Compression** | Huffman encoding tree | Optimal prefix codes via binary tree structure |
| **CPU architecture** | Expression evaluation | Arithmetic expression trees evaluated in hardware |

### The AST — The Tree You Write Code With

Every time you compile a program, the compiler builds a tree from your source code:

```
Expression: (a + b) * (c - d)

       *
      / \
     +   -
    / \ / \
   a  b c  d

Evaluation (postorder):
  - Evaluate left subtree: a + b
  - Evaluate right subtree: c - d
  - Apply operator: multiply results

Code generation (preorder):
  - Emit multiply instruction
  - Emit add instruction for operands a, b
  - Emit subtract instruction for operands c, d
```

The traversal order determines the computation order. This is why postorder is used for expression evaluation (children before parents), and why preorder is used for tree serialisation (parent before children).

---

## 13. Edge Cases & Pitfalls

### Pitfall 1: Not Handling the Null Tree

```cpp
// WRONG: assuming root is always non-null
int height_wrong(TreeNode* root) {
    return 1 + max(height_wrong(root->left),   // crashes if root is null!
                   height_wrong(root->right));
}

// CORRECT: always guard with null check first
int height(TreeNode* root) {
    if (!root) return -1;  // or -1 for "height of null"
    return 1 + max(height(root->left), height(root->right));
}

// Null check belongs at the BEGINNING of every recursive tree function.
// This is not optional — it is the base case that makes the recursion terminate.
```

### Pitfall 2: Single Node is a Leaf — Check Both Children

```cpp
// WRONG: checking only one child to determine if a node is a leaf
bool isLeaf_wrong(TreeNode* root) {
    return root->left == nullptr;   // single-child nodes are NOT leaves!
}

// CORRECT: a leaf has NO children — both must be null
bool isLeaf(TreeNode* root) {
    return root != nullptr && root->left == nullptr && root->right == nullptr;
}

// Common contexts where this matters:
// - Root-to-leaf path problems
// - Counting leaves
// - Checking path sum to leaves specifically
```

### Pitfall 3: Confusing Height and Depth

```cpp
// Height = distance FROM the node DOWN to the deepest leaf.
//   height(leaf) = 0
//   height(null) = -1 (by convention)
//   height(root) = length of longest root-to-leaf path

// Depth = distance FROM the root DOWN to the node.
//   depth(root) = 0
//   depth(child) = depth(parent) + 1

// Common bug: using height when depth is needed, or vice versa.
// Fix: always think "am I measuring from this node down, or from the root down?"
// If from node down → height. If from root down → depth (pass as parameter).
```

### Pitfall 4: Stack Overflow on Skewed Trees

```cpp
// For a degenerate (all-left or all-right) tree with n=100,000 nodes:
//   height = n-1 = 99,999
//   recursive call stack depth = 99,999 frames
//   Each frame ≈ 64 bytes → stack usage ≈ 6.4 MB
//   Default stack size: 1-8 MB → STACK OVERFLOW

// Solution 1: iterative implementation (uses heap instead of call stack)
// Solution 2: convert to BFS for problems that allow it
// Solution 3: increase stack size (OS-dependent, not portable)
// Solution 4: iterative deepening DFS (O(1) extra space, O(n) time per level)

// Rule: in interviews, mention "for balanced trees, recursion is safe.
// For general trees, iterative or explicit stack handles skewed cases."
```

### Pitfall 5: Modifying the Tree During Traversal

```cpp
// Deleting a node during postorder traversal is safe (delete after children are processed).
// Modifying a node's children during traversal can corrupt the traversal.

void deleteTree(TreeNode* root) {
    if (!root) return;
    deleteTree(root->left);
    deleteTree(root->right);
    delete root;   // postorder: safe — both children already freed
}

// DANGEROUS: changing root->left mid-traversal during inorder
void invertInorder_WRONG(TreeNode* root) {
    if (!root) return;
    invertInorder_WRONG(root->left);
    swap(root->left, root->right);  // NOW root->right is the old left!
    invertInorder_WRONG(root->right);  // ← visits old LEFT child again → infinite loop!
}

// CORRECT: preorder swap (before recursing) or postorder swap
void invertTree(TreeNode* root) {
    if (!root) return;
    swap(root->left, root->right);  // swap FIRST
    invertTree(root->left);          // then recurse (on the new left = old right)
    invertTree(root->right);
}
```

### Pitfall 6: Integer Overflow in Path Sums

```cpp
// For large trees with large values, path sums may overflow int.
// Use long long for path sum problems.

// WRONG:
int maxPathSum_wrong(TreeNode* root) {
    int result = INT_MIN;
    function<int(TreeNode*)> dfs = [&](TreeNode* node) -> int {
        if (!node) return 0;
        int lg = max(0, dfs(node->left));
        int rg = max(0, dfs(node->right));
        result = max(result, node->val + lg + rg);  // overflow if lg, rg are large ints
        return node->val + max(lg, rg);
    };
    dfs(root);
    return result;
}

// CORRECT: use long long when values can be large
int maxPathSum(TreeNode* root) {
    long long result = LLONG_MIN;
    function<long long(TreeNode*)> dfs = [&](TreeNode* node) -> long long {
        if (!node) return 0LL;
        long long lg = max(0LL, dfs(node->left));
        long long rg = max(0LL, dfs(node->right));
        result = max(result, (long long)node->val + lg + rg);
        return (long long)node->val + max(lg, rg);
    };
    dfs(root);
    return (int)result;
}
```

### Pitfall 7: Off-by-One in Level/Depth Counting

```cpp
// Convention inconsistency is a major source of bugs.
// ALWAYS establish your convention at the start:

// Convention A: root is at depth 0, leaves have depth ≥ 0
//   height(null) = -1, height(leaf) = 0

// Convention B: root is at level 1, leaves have level ≥ 1
//   height(null) = 0, height(leaf) = 1

// In LeetCode problems: depth/level starts at 1 UNLESS stated otherwise.
// In academic CS: depth of root = 0.

// Whichever convention you use — BE CONSISTENT throughout your solution.
```

---

## 14. Binary Tree Variants at a Glance

| Variant | Added Constraint | Key Property | Where in Curriculum |
|---|---|---|---|
| **Binary Tree** | None | General hierarchy | This document |
| **BST** | left < root < right | O(log n) search (balanced) | Next → BST |
| **AVL Tree** | BST + height-balanced | O(log n) all ops guaranteed | After BST |
| **Red-Black Tree** | BST + color rules | O(log n) amortised | After AVL |
| **Heap** | Complete + heap property | O(1) min/max, O(log n) insert | Heap section |
| **Segment Tree** | Built on array | O(log n) range queries | Advanced Trees |
| **Fenwick Tree** | Implicit binary tree | O(log n) prefix sums | Advanced Trees |
| **Trie** | Characters at edges | O(m) string lookup | String Trees |
| **B-Tree** | n-ary, disk-oriented | O(log n) with small constant | Database Trees |

---

## 15. Self-Test Questions

1. **What is the maximum number of nodes in a binary tree of height 4? What is the minimum? Show your reasoning using the height-size relationship.**

2. **Write the inorder, preorder, and postorder traversal of this tree from memory:**
   ```
        1
       / \
      3   2
     / \
    4   5
   ```

3. **What is the time complexity of the recursive height function? Why is it O(n) and not O(h)?**

4. **Implement iterative inorder traversal without looking at the notes. Trace it on a tree of your choice to verify correctness.**

5. **Explain why the base case `if (!root || root == p || root == q) return root` correctly handles the LCA when p is an ancestor of q.**

6. **In the "construct tree from preorder and inorder" problem, what information does the preorder array give you that the inorder array does not? Vice versa?**

7. **A tree has 15 nodes. What is its height if it is a perfect binary tree? What if it is completely left-skewed?**

8. **Why does the `maxPathSum` problem return `node->val + max(leftGain, rightGain)` to the parent rather than `node->val + leftGain + rightGain`? Draw the distinction.**

9. **Morris traversal achieves O(1) space for inorder traversal. Describe the mechanism. What temporary modification does it make to the tree? Is the tree restored to its original state?**

10. **For what tree shape would recursive traversal cause a stack overflow? What is the practical n threshold for this on a typical system? How would you handle it in production?**

---

## Quick Reference Card

```
Binary Tree — the foundational hierarchical data structure.
Every tree data structure in the curriculum is a binary tree with constraints.

Node: { val, left*, right* }
Tree: just a root pointer. null = empty tree.

Height: -1 for null; 0 for leaf; 1 + max(left_h, right_h) for internal.
Depth: 0 for root; parent_depth + 1 for children.
Size: 0 for null; 1 + size(left) + size(right) for any node.

The four traversals (master all of them):
  Inorder   (L→N→R): left subtree, root, right subtree
    → BST sorted order. Recursive + iterative (stack + curr pointer).
  Preorder  (N→L→R): root, left subtree, right subtree
    → Serialisation, copy. Iterative: push right before left.
  Postorder (L→R→N): left subtree, right subtree, root
    → Deletion, evaluation. Iterative: reverse preorder variant.
  Level order (BFS): queue-based, level by level.
    → Shortest path, level statistics.

Universal recursion template:
  f(null) = base_value
  f(node) = combine(node.val, f(node.left), f(node.right))

Key patterns:
  1. Path problems:       push + recurse + pop_back (backtracking)
  2. Return two values:   struct or pair for {height, answer}
  3. Global maximum:      capture by reference in lambda
  4. Top-down info:       pass extra parameter (depth, running sum)
  5. O(1) space inorder:  Morris traversal (threaded tree)

Complexity:
  All traversals: O(n) time, O(h) space (O(log n) balanced, O(n) skewed)
  Level order: O(n) time, O(n) space (max queue size = width)

Critical rules:
  → ALWAYS null-check at start of every recursive function
  → Leaf = !root->left && !root->right  (BOTH must be null)
  → Use long long for path sum problems to avoid overflow
  → For skewed trees: use iterative to avoid stack overflow
  → Establish height/depth convention (0-indexed root or 1-indexed) once and be consistent
```

---

*Previous: [21 — Robin Hood Hashing](./21_robin_hood_hashing.md)*
*Next: [23 — Binary Search Tree](./23_binary_search_tree.md)*
*See also: [Priority Queue / Heap](./13_priority_queue.md) | [Segment Tree](./trees/segment_tree.md) | [Trie](./trees/trie.md)*

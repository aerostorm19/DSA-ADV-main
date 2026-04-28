# Red-Black Tree

> **Curriculum position:** Trees → #4
> **Interview weight:** ★★★★☆ High — the five properties, rotation + recoloring mechanics, and comparison with AVL are standard depth questions; implementing from scratch is rare but the conceptual reasoning is expected
> **Difficulty:** Advanced — the invariant is more subtle than AVL's; the six insertion fixup cases and deletion complexity require genuine study
> **Prerequisite:** [23 — Binary Search Tree](./23_binary_search_tree.md) | [24 — AVL Tree](./24_avl_tree.md)

---

## Table of Contents

1. [Intuition First — Relaxed Balance for Write Efficiency](#1-intuition-first)
2. [The Five Red-Black Properties](#2-the-five-red-black-properties)
3. [Why the Properties Guarantee O(log n) Height](#3-height-guarantee)
4. [Internal Working — Nodes, Sentinel, and Colors](#4-internal-working)
5. [Insertion — Recolor Before You Rotate](#5-insertion)
6. [Insertion Fixup — The Six Cases](#6-insertion-fixup)
7. [Deletion — The Most Complex Operation in Any BST](#7-deletion)
8. [Time & Space Complexity](#8-time--space-complexity)
9. [Complete C++ Implementation](#9-complete-c-implementation)
10. [Core Operations — Visualised](#10-core-operations--visualised)
11. [Interview Problems](#11-interview-problems)
12. [Real-World Uses](#12-real-world-uses)
13. [Edge Cases & Pitfalls](#13-edge-cases--pitfalls)
14. [Red-Black vs AVL vs B-Tree vs Skip List](#14-comparison)
15. [Self-Test Questions](#15-self-test-questions)

---

## 1. Intuition First

AVL trees enforce strict balance: the height of any two sibling subtrees may differ by at most 1. This gives the tightest possible height bound — 1.44 log₂(n) — but forces O(log n) rotations during deletion, because every height decrease must be corrected all the way back to the root.

The **red-black tree** asks: what is the *minimum* structural constraint that still guarantees O(log n) height, while allowing O(1) rotations per modification?

The answer is a coloring scheme. Each node is painted red or black. Five invariants about these colors are maintained. Together they guarantee that the tree height never exceeds 2 log₂(n+1) — twice the height of a perfect tree. Less tight than AVL, but still O(log n), and achievable with at most **2 rotations per insertion** and at most **3 rotations per deletion** (amortized O(1)).

The key insight of red-black trees: instead of balancing by height, balance by **path length to null**. Every path from any node to any null descendant must pass through the same number of black nodes. Red nodes are "free" — they extend paths without increasing the black count, acting as flexible spacers that absorb structural irregularities without triggering rebalancing cascades.

This scheme was invented by Rudolf Bayer in 1972 and refined by Guibas and Sedgewick in 1978. It is the data structure behind `std::map`, `std::set`, `std::multimap`, `std::multiset` in every major C++ standard library, `TreeMap` and `TreeSet` in Java, the Linux kernel's process scheduler, virtual memory manager, and many other production systems. Understanding red-black trees deeply is understanding the default ordered container in modern programming.

---

## 2. The Five Red-Black Properties

Every valid red-black tree satisfies all five properties simultaneously:

```
Property 1 (Color):        Every node is either RED or BLACK.

Property 2 (Root):         The root is BLACK.

Property 3 (Null leaves):  Every null pointer (external/NIL node) is BLACK.
                            Null leaves are conceptually black leaf sentinels.

Property 4 (Red children): If a node is RED, both its children are BLACK.
                            (No two consecutive red nodes on any path.)

Property 5 (Black height): For any node, all paths from that node to
                            descendant null leaves contain the same number
                            of BLACK nodes.
                            (The "black-height" of every node is well-defined.)
```

### Visual Verification

```
Valid red-black tree (R=red, B=black):
         B:7
        /    \
      R:3    R:18
     /   \   /   \
   B:2  B:5 B:11  B:22
  / \  / \  / \   / \
 N  N N  N N  N  N  N   (N = null, black)

Property 1: All nodes colored. ✓
Property 2: Root (7) is black. ✓
Property 3: All null pointers are black. ✓
Property 4: Red nodes 3, 18 — their children (2,5,11,22) are all black. ✓
Property 5: Every path from root to null has 2 black nodes:
  7→3→2→N: blacks = {7, 2} = 2 ✓
  7→3→5→N: blacks = {7, 5} = 2 ✓
  7→18→11→N: blacks = {7, 11} = 2 ✓
  7→18→22→N: blacks = {7, 22} = 2 ✓
```

### The Black-Height

The **black-height** of a node is the number of black nodes on any path from that node down to a null leaf, **not** counting the node itself.

```
black_height(null) = 0
black_height(node) = black_height(child) + (child is BLACK ? 1 : 0)

For the tree above:
  black_height(2) = 1   (path 2→N: one black null)
  black_height(3) = 1   (path 3→2→N: one black node 2)
  black_height(7) = 2   (path 7→3→2→N: two black nodes 3? No — 3 is red!)
  Wait: black_height counts BLACK nodes on path to null, not including the node itself.
  black_height(7): path 7→3→2→N. Black nodes excluding 7: {2} and null... 
  
  Convention: bh(null) = 0. bh(node x) = # black nodes strictly below x on any root-to-null path.
  bh(2) = 1 (the null below it, but null has bh=0, so bh(2) = bh(null)+1 = 1)
  bh(3) = bh(2) = 1 (since 2 is black, bh(3) = bh(2) + 1... 
  
  Let me use the cleaner definition:
  bh(node) = number of BLACK nodes on path from node to any null leaf, NOT counting node itself.
  bh(null) = 0.
  bh(leaf black node like 2) = 0 + 1 = 1? No.
  
  Standard: bh(null) = 0. bh(x) = bh(x->left) + (x->left is BLACK ? 1 : 0).
  bh(2): x->left = null(BLACK). bh(null) + 1 = 0 + 1 = 1.
  bh(3): x->left = 2(BLACK). bh(2) + 1 = 1 + 1 = 2.
  bh(7): x->left = 3(RED). bh(3) + 0 = 2 + 0 = 2.
```

---

## 3. Why the Properties Guarantee O(log n) Height

### The Key Argument

**Lemma:** A red-black tree with black-height `bh` has at least `2^bh - 1` internal nodes.

*Proof by induction:*
- Base: bh=0 → 0 nodes (empty tree). 2^0 - 1 = 0. ✓
- Inductive step: Each child has bh ≥ bh-1 (at least bh-1 if the child is red, exactly bh-1 if the child is black). By induction, each child subtree has ≥ 2^(bh-1) - 1 nodes. Together with the root: total ≥ 2(2^(bh-1) - 1) + 1 = 2^bh - 1. ✓

**From Lemma to Height Bound:**

```
Let h = tree height, bh = black-height.
By Property 4: no two consecutive red nodes.
→ On any root-to-leaf path, at most half the nodes can be red.
→ bh ≥ h/2 (at least half the nodes on any path are black).

From the Lemma: n ≥ 2^bh - 1 ≥ 2^(h/2) - 1.

Solving: n + 1 ≥ 2^(h/2)
         log₂(n+1) ≥ h/2
         h ≤ 2 log₂(n+1)

Therefore: height ≤ 2 log₂(n+1) = O(log n). ✓

For n = 1,000,000:
  Lower bound (perfect tree): h = 20
  AVL upper bound: h = 29
  Red-Black upper bound: h = 40
  All three: O(log n). Red-Black is looser but still logarithmic.
```

---

## 4. Internal Working — Nodes, Sentinel, and Colors

### Node Structure

```cpp
enum Color { RED, BLACK };

struct RBNode {
    int     val;
    Color   color;
    RBNode* left;
    RBNode* right;
    RBNode* parent;  // parent pointer — essential for fixup operations

    RBNode(int v, Color c, RBNode* nil)
        : val(v), color(c), left(nil), right(nil), parent(nil) {}
};
```

**Parent pointers are required.** Unlike AVL trees (which use recursion to backtrack), red-black tree fixup iterates upward through the tree. Without parent pointers, you cannot reach the uncle or grandparent nodes that fixup logic needs.

### The Sentinel NIL Node

A crucial implementation detail: instead of `nullptr` for leaf pointers, use a **sentinel NIL node** that is always BLACK. This eliminates null-pointer checks in the fixup algorithms:

```cpp
// One sentinel shared by the entire tree
RBNode* NIL = new RBNode(0, BLACK, nullptr);
NIL->left = NIL->right = NIL->parent = NIL;

// Every empty child pointer points to NIL, not nullptr.
// Invariant: NIL->color == BLACK always.
// Benefit: fixup code can read NIL->color without null checks.

// Without sentinel: every rotation and fixup needs:
//   if (node->right != nullptr && node->right->color == RED) ...
// With sentinel:
//   if (node->right->color == RED) ...   ← always safe
```

### Rotations — Same Structure as AVL

Red-black rotations are identical to AVL rotations, but must also update parent pointers:

```cpp
void rotateLeft(RBNode*& root, RBNode* x) {
    RBNode* y = x->right;
    x->right = y->left;
    if (y->left != NIL) y->left->parent = x;
    y->parent = x->parent;
    if (x->parent == NIL) root = y;           // x was root
    else if (x == x->parent->left)  x->parent->left  = y;
    else                             x->parent->right = y;
    y->left   = x;
    x->parent = y;
}

void rotateRight(RBNode*& root, RBNode* x) {
    RBNode* y = x->left;
    x->left = y->right;
    if (y->right != NIL) y->right->parent = x;
    y->parent = x->parent;
    if (x->parent == NIL) root = y;
    else if (x == x->parent->right) x->parent->right = y;
    else                             x->parent->left  = y;
    y->right  = x;
    x->parent = y;
}
```

---

## 5. Insertion — Recolor Before You Rotate

### Basic Insertion

Insert as in a standard BST, but color the new node **RED**:

```
Why red?
  Inserting red does NOT change the black-height of any path.
  Therefore, Property 5 is NOT violated by a new red node.
  Properties 1, 3 are trivially preserved.
  Property 2 (root is black) may be violated if we insert into an empty tree.
  Property 4 (no two consecutive reds) may be violated if the parent is red.

We only need to fix Property 2 and Property 4 after insertion.
This is easier than fixing Property 5 (black-height change).
```

### The Strategy: Recolor First, Rotate Last

The fixup algorithm's philosophy: **try to fix violations by recoloring; only rotate if recoloring alone cannot fix it.** Recoloring is O(1) and moves the violation upward (toward the root) without structural change. Rotation fixes the final violation in O(1).

The violation to fix after insertion: the new red node `Z` has a red parent `P`. This violates Property 4. The fix depends on the color of the **uncle** (P's sibling):

---

## 6. Insertion Fixup — The Six Cases

The cases come in symmetric pairs (left/right mirror images). We describe three cases and their left-right mirrors.

### Setup

```
Terminology:
  Z = newly inserted red node (or the red node moving up during fixup)
  P = Z's parent (red — that's the violation)
  G = Z's grandparent (black — since P is red and we had a valid tree)
  U = Z's uncle = G's other child

Z's parent P is always red (the violation). G is always black (before P turned red, the tree was valid — so G was black to satisfy P's prior BLACK state... actually G must be black because Property 4 required P's parent to be black before we inserted Z). U can be red or black.
```

### Case 1: Uncle is RED

```
Before:
       G (BLACK)
      / \
     P   U     ← both P and U are RED
    /
   Z (RED)    ← violation: Z and P are both red

Fix: Recolor P and U to BLACK, G to RED.
After:
       G (RED)    ← G is now red
      / \
     P   U     ← both now BLACK
    /
   Z (RED)    ← no longer a violation at this level

Why correct: P→Z path: added one black (P became black). G→U path: added one black (U). G→P path: removed one black (G changed to red). The black-heights stay consistent IF we assume G's parent handled this new red G. But G is now red and its parent might be red → move violation upward to G.

Continue fixup with Z = G (G is the new potential violator).
If G reaches the root: recolor root BLACK (satisfies Property 2, adds 1 to overall black-height).
```

### Case 2: Uncle is BLACK, Z is the "Inner" Child (forms a bent path)

Inner child: Z is a right child and P is a left child (or Z is left child and P is right child).

```
Before:
       G (BLACK)
      / \
     P   U (BLACK)
      \
       Z (RED)   ← Z is right child of P (inner)

Fix: Left-rotate on P. This transforms into Case 3.
After rotate:
       G (BLACK)
      / \
     Z   U (BLACK)   ← Z is now left child of G
    /
   P (RED)           ← P is now left child of Z
   (Z is still RED, P is still RED → violation still exists but now in Case 3 form)

Now Z and P have swapped roles. Continue with Case 3 where "new Z" = P.
```

### Case 3: Uncle is BLACK, Z is the "Outer" Child (forms a straight path)

Outer child: Z is a left child and P is a left child (or Z is right child and P is right child).

```
Before (after Case 2 transform, or directly):
       G (BLACK)
      / \
     P   U (BLACK)
    /
   Z (RED)   ← Z is left child of P (outer, straight path)

Fix: Right-rotate on G, swap colors of P and G.
After:
       P (BLACK)   ← P takes G's place, becomes BLACK
      / \
     Z   G (RED)   ← G becomes RED, drops down
(RED)   /
        (old P->right, now G->left, color unchanged)
        U still G's right child (unchanged)

Property 4: P is black. Z (red) and G (red) are P's children — both red children of a black parent is fine IF Z and G don't have red children themselves. But Z is a leaf (just inserted) and G's right child is U (black). ✓
Property 5: black-height preserved. Before: G→P path had bh contributions from G and P. After rotation: P is the root of this subtree, P is black (was red, gained black status). ✓
```

### All Six Cases (Left + Right Mirror)

```
Case 1L / 1R: Uncle is RED → Recolor. Move up.
Case 2L:      Uncle BLACK, Z is inner (right child of left parent) → Left-rotate P → becomes 3L
Case 2R:      Uncle BLACK, Z is inner (left child of right parent) → Right-rotate P → becomes 3R
Case 3L:      Uncle BLACK, Z is outer (left child of left parent) → Right-rotate G, recolor
Case 3R:      Uncle BLACK, Z is outer (right child of right parent) → Left-rotate G, recolor

Total: at most 2 rotations per insertion (Case 2 + Case 3), plus O(log n) recolorings (Case 1).
```

---

## 7. Deletion — The Most Complex Operation in Any BST

### Why Deletion Is Hard

When deleting a black node, the black-height of paths through that node decreases by 1 — violating Property 5. The fixup must restore black-height without violating Properties 1, 2, 3, 4.

### The Transplant Operation

```cpp
// Replace subtree rooted at u with subtree rooted at v
void transplant(RBNode*& root, RBNode* u, RBNode* v) {
    if (u->parent == NIL) root = v;
    else if (u == u->parent->left)  u->parent->left  = v;
    else                             u->parent->right = v;
    v->parent = u->parent;   // always safe even if v == NIL (NIL->parent updated)
}
```

### The Three BST Deletion Cases

```
Case 1: Node has no left child → transplant with right child.
Case 2: Node has no right child → transplant with left child.
Case 3: Node has two children → find successor (min of right subtree),
        copy value to node, then delete the successor (Case 1 or 2).
```

### The "Double Black" Concept

When a black node is removed and replaced by a black child (or NIL), the replaced node is conceptually "double black" — it is carrying an extra unit of blackness that must be redistributed. The fixup algorithm works to move this extra black upward or eliminate it.

### Deletion Fixup — Four Cases

Assume `x` is the double-black node (could be NIL), `w` is x's sibling.

**Case 1: w is RED**
```
Recolor w to BLACK, x's parent P to RED.
Left-rotate on P (if x is left child) or right-rotate.
New sibling of x is the old w's left child (which was black) → proceed to Case 2, 3, or 4.
```

**Case 2: w is BLACK, both of w's children are BLACK**
```
Recolor w to RED.
If P is RED: recolor P to BLACK → done.
If P is BLACK: P becomes double black → move up (x = P, continue fixup).
```

**Case 3: w is BLACK, w's "inner" child is RED, w's "outer" child is BLACK**
```
Recolor w's inner child BLACK, w RED.
Rotate w in the direction of x.
New sibling is the old inner child → now in Case 4.
```

**Case 4: w is BLACK, w's "outer" child is RED**
```
Recolor w to x->parent's color.
Recolor x->parent to BLACK.
Recolor w's outer child to BLACK.
Rotate x->parent away from x.
x is no longer double black. Done.
```

### Deletion Properties

```
Insertion:  At most 2 rotations. O(log n) recolorings.
Deletion:   At most 3 rotations. O(log n) recolorings.

Compare to AVL:
  AVL insertion:  At most 1 rotation.
  AVL deletion:   Up to O(log n) rotations.
  
Red-Black is BETTER than AVL for deletions:
  bounded rotations means bounded write amplification to disk/cache.
```

---

## 8. Time & Space Complexity

### Operation Complexities

| Operation | Time | Rotations | Notes |
|---|---|---|---|
| `search(val)` | **O(log n)** | 0 | Path ≤ 2 log₂(n+1) |
| `insert(val)` | **O(log n)** | ≤ 2 | O(log n) recolorings + ≤ 2 rotations |
| `delete(val)` | **O(log n)** | ≤ 3 | O(log n) recolorings + ≤ 3 rotations |
| `findMin()` | **O(log n)** | 0 | Follow left pointers |
| `findMax()` | **O(log n)** | 0 | Follow right pointers |
| `successor(v)` | **O(log n)** | 0 | Right-subtree min or ancestor |
| `rangeSearch` | **O(log n + k)** | 0 | k = results in range |
| Inorder traversal | O(n) | 0 | Visits all nodes |
| Rotation (each) | **O(1)** | — | 3 pointer updates + parent updates |
| Build from n | O(n log n) | — | n insertions |

### Space Complexity

| Component | Space | Notes |
|---|---|---|
| n nodes | O(n) | One node per value |
| Per-node overhead | 1 color bit + 3 pointers | ~25 bytes on 64-bit systems |
| Parent pointer | +8 bytes vs AVL | Required for fixup algorithms |
| NIL sentinel | O(1) | One shared node |
| Call stack | O(log n) | From recursive operations |
| Total | **O(n)** | Same asymptotic as all BSTs |

### Height Comparison

```
n = 1,000 nodes:
  Perfect BST: h = 10
  AVL worst:   h ≤ 14
  RB worst:    h ≤ 20

n = 1,000,000 nodes:
  Perfect BST: h = 20
  AVL worst:   h ≤ 29
  RB worst:    h ≤ 40

n = 1,000,000,000 nodes:
  Perfect BST: h = 30
  AVL worst:   h ≤ 43
  RB worst:    h ≤ 60

All O(log n). RB is always within 2× of optimal.
The absolute difference (29 vs 40 for n=10^6) rarely matters in practice —
the O(log n) bound is what matters, not the constant.
```

---

## 9. Complete C++ Implementation

```cpp
#include <iostream>
#include <functional>
#include <vector>
#include <optional>
#include <cassert>
using namespace std;

enum class Color : uint8_t { RED = 0, BLACK = 1 };

struct RBNode {
    int     val;
    Color   color;
    RBNode* left;
    RBNode* right;
    RBNode* parent;

    RBNode(int v, Color c, RBNode* nil_sentinel)
        : val(v), color(c), left(nil_sentinel),
          right(nil_sentinel), parent(nil_sentinel) {}
};

class RedBlackTree {
private:
    RBNode* NIL_;   // sentinel: always BLACK, shared by all empty positions
    RBNode* root_;

    // ── Rotation helpers ─────────────────────────────────────────────────────

    void rotateLeft(RBNode* x) {
        RBNode* y = x->right;
        x->right  = y->left;
        if (y->left != NIL_) y->left->parent = x;
        y->parent = x->parent;
        if (x->parent == NIL_)       root_              = y;
        else if (x == x->parent->left) x->parent->left  = y;
        else                            x->parent->right = y;
        y->left   = x;
        x->parent = y;
    }

    void rotateRight(RBNode* x) {
        RBNode* y = x->left;
        x->left   = y->right;
        if (y->right != NIL_) y->right->parent = x;
        y->parent = x->parent;
        if (x->parent == NIL_)        root_              = y;
        else if (x == x->parent->right) x->parent->right = y;
        else                             x->parent->left  = y;
        y->right  = x;
        x->parent = y;
    }

    // ── Insertion fixup ───────────────────────────────────────────────────────
    // Fix Property 4 violation: z and z->parent are both RED.

    void insertFixup(RBNode* z) {
        while (z->parent->color == Color::RED) {
            if (z->parent == z->parent->parent->left) {
                // Parent is LEFT child of grandparent
                RBNode* uncle = z->parent->parent->right;

                if (uncle->color == Color::RED) {
                    // Case 1: uncle is RED → recolor, move up
                    z->parent->color          = Color::BLACK;
                    uncle->color              = Color::BLACK;
                    z->parent->parent->color  = Color::RED;
                    z = z->parent->parent;   // move violation up to grandparent
                } else {
                    if (z == z->parent->right) {
                        // Case 2: z is inner child (right child of left parent)
                        z = z->parent;
                        rotateLeft(z);   // now z is outer → Case 3
                    }
                    // Case 3: z is outer child (left child of left parent)
                    z->parent->color         = Color::BLACK;
                    z->parent->parent->color = Color::RED;
                    rotateRight(z->parent->parent);
                }
            } else {
                // Symmetric: parent is RIGHT child of grandparent
                RBNode* uncle = z->parent->parent->left;

                if (uncle->color == Color::RED) {
                    // Case 1 (mirror)
                    z->parent->color         = Color::BLACK;
                    uncle->color             = Color::BLACK;
                    z->parent->parent->color = Color::RED;
                    z = z->parent->parent;
                } else {
                    if (z == z->parent->left) {
                        // Case 2 (mirror)
                        z = z->parent;
                        rotateRight(z);
                    }
                    // Case 3 (mirror)
                    z->parent->color         = Color::BLACK;
                    z->parent->parent->color = Color::RED;
                    rotateLeft(z->parent->parent);
                }
            }
        }
        root_->color = Color::BLACK;   // Property 2: root is always black
    }

    // ── Deletion helpers ──────────────────────────────────────────────────────

    void transplant(RBNode* u, RBNode* v) {
        if (u->parent == NIL_)       root_              = v;
        else if (u == u->parent->left) u->parent->left  = v;
        else                            u->parent->right = v;
        v->parent = u->parent;
    }

    RBNode* minimum(RBNode* x) const {
        while (x->left != NIL_) x = x->left;
        return x;
    }

    // Fix double-black violation at x.
    void deleteFixup(RBNode* x) {
        while (x != root_ && x->color == Color::BLACK) {
            if (x == x->parent->left) {
                RBNode* w = x->parent->right;   // sibling

                if (w->color == Color::RED) {
                    // Case 1: sibling is red
                    w->color          = Color::BLACK;
                    x->parent->color  = Color::RED;
                    rotateLeft(x->parent);
                    w = x->parent->right;
                }
                if (w->left->color == Color::BLACK && w->right->color == Color::BLACK) {
                    // Case 2: sibling is black, both nephews are black
                    w->color = Color::RED;
                    x = x->parent;
                } else {
                    if (w->right->color == Color::BLACK) {
                        // Case 3: sibling black, left nephew red, right nephew black
                        w->left->color = Color::BLACK;
                        w->color       = Color::RED;
                        rotateRight(w);
                        w = x->parent->right;
                    }
                    // Case 4: sibling black, right nephew red
                    w->color          = x->parent->color;
                    x->parent->color  = Color::BLACK;
                    w->right->color   = Color::BLACK;
                    rotateLeft(x->parent);
                    x = root_;   // done: break the while loop
                }
            } else {
                // Symmetric: x is right child
                RBNode* w = x->parent->left;

                if (w->color == Color::RED) {
                    // Case 1 (mirror)
                    w->color         = Color::BLACK;
                    x->parent->color = Color::RED;
                    rotateRight(x->parent);
                    w = x->parent->left;
                }
                if (w->right->color == Color::BLACK && w->left->color == Color::BLACK) {
                    // Case 2 (mirror)
                    w->color = Color::RED;
                    x = x->parent;
                } else {
                    if (w->left->color == Color::BLACK) {
                        // Case 3 (mirror)
                        w->right->color = Color::BLACK;
                        w->color        = Color::RED;
                        rotateLeft(w);
                        w = x->parent->left;
                    }
                    // Case 4 (mirror)
                    w->color         = x->parent->color;
                    x->parent->color = Color::BLACK;
                    w->left->color   = Color::BLACK;
                    rotateRight(x->parent);
                    x = root_;
                }
            }
        }
        x->color = Color::BLACK;   // ensure x (possibly root) is black
    }

    // ── Cleanup ───────────────────────────────────────────────────────────────

    void destroy(RBNode* node) {
        if (node == NIL_) return;
        destroy(node->left);
        destroy(node->right);
        delete node;
    }

    // ── Validation ────────────────────────────────────────────────────────────

    bool validateBST(RBNode* node, long long minV, long long maxV) const {
        if (node == NIL_) return true;
        if (node->val <= minV || node->val >= maxV) return false;
        return validateBST(node->left,  minV,       node->val)
            && validateBST(node->right, node->val,  maxV);
    }

    // Returns black-height if valid, -1 if invalid
    int validateRB(RBNode* node) const {
        if (node == NIL_) return 1;   // null leaf counts as 1 black

        if (node->color == Color::RED) {
            // Property 4: red node's children must be black
            if (node->left->color != Color::BLACK) return -1;
            if (node->right->color != Color::BLACK) return -1;
        }

        int lbh = validateRB(node->left);
        int rbh = validateRB(node->right);
        if (lbh == -1 || rbh == -1) return -1;
        if (lbh != rbh) return -1;   // Property 5: black-heights must match

        return lbh + (node->color == Color::BLACK ? 1 : 0);
    }

    // ── Inorder ───────────────────────────────────────────────────────────────

    void inorder_(RBNode* node, vector<int>& out) const {
        if (node == NIL_) return;
        inorder_(node->left, out);
        out.push_back(node->val);
        inorder_(node->right, out);
    }

    // ── Range search ─────────────────────────────────────────────────────────

    void range_(RBNode* node, int lo, int hi, vector<int>& out) const {
        if (node == NIL_) return;
        if (node->val > lo) range_(node->left, lo, hi, out);
        if (node->val >= lo && node->val <= hi) out.push_back(node->val);
        if (node->val < hi) range_(node->right, lo, hi, out);
    }

public:
    // ── Construction ─────────────────────────────────────────────────────────

    RedBlackTree() {
        NIL_  = new RBNode(0, Color::BLACK, nullptr);
        NIL_->left = NIL_->right = NIL_->parent = NIL_;
        root_ = NIL_;
    }

    ~RedBlackTree() {
        destroy(root_);
        delete NIL_;
    }

    RedBlackTree(const RedBlackTree&)            = delete;
    RedBlackTree& operator=(const RedBlackTree&) = delete;

    // ── Insert — O(log n) ────────────────────────────────────────────────────

    void insert(int val) {
        RBNode* z = new RBNode(val, Color::RED, NIL_);
        RBNode* y = NIL_;
        RBNode* x = root_;

        // Standard BST descent to find insertion point
        while (x != NIL_) {
            y = x;
            if      (z->val < x->val) x = x->left;
            else if (z->val > x->val) x = x->right;
            else { delete z; return; }  // duplicate: no-op
        }

        z->parent = y;
        if      (y == NIL_)           root_    = z;
        else if (z->val < y->val)  y->left  = z;
        else                        y->right = z;

        // z's children are already NIL_ (set in constructor)
        insertFixup(z);
    }

    // ── Delete — O(log n) ────────────────────────────────────────────────────

    void remove(int val) {
        // Find node
        RBNode* z = root_;
        while (z != NIL_ && z->val != val) {
            z = (val < z->val) ? z->left : z->right;
        }
        if (z == NIL_) return;   // not found

        RBNode* y = z;           // node being removed (or its successor)
        RBNode* x;               // node that moves into y's position
        Color   yOriginalColor = y->color;

        if (z->left == NIL_) {
            x = z->right;
            transplant(z, z->right);
        } else if (z->right == NIL_) {
            x = z->left;
            transplant(z, z->left);
        } else {
            y = minimum(z->right);             // inorder successor
            yOriginalColor = y->color;
            x = y->right;
            if (y->parent == z) {
                x->parent = y;
            } else {
                transplant(y, y->right);
                y->right         = z->right;
                y->right->parent = y;
            }
            transplant(z, y);
            y->left          = z->left;
            y->left->parent  = y;
            y->color         = z->color;
        }

        delete z;

        if (yOriginalColor == Color::BLACK) {
            deleteFixup(x);   // fixup needed only when a BLACK node was removed
        }
    }

    // ── Search — O(log n) ────────────────────────────────────────────────────

    bool contains(int val) const {
        RBNode* x = root_;
        while (x != NIL_) {
            if      (val == x->val) return true;
            else if (val <  x->val) x = x->left;
            else                    x = x->right;
        }
        return false;
    }

    // ── Utility ───────────────────────────────────────────────────────────────

    bool empty() const { return root_ == NIL_; }

    optional<int> findMin() const {
        if (root_ == NIL_) return nullopt;
        return minimum(root_)->val;
    }

    optional<int> findMax() const {
        if (root_ == NIL_) return nullopt;
        RBNode* x = root_;
        while (x->right != NIL_) x = x->right;
        return x->val;
    }

    int blackHeight() const { return validateRB(root_); }

    bool isValidRBT() const {
        if (root_->color != Color::BLACK) return false;
        if (!validateBST(root_, (long long)INT_MIN - 1, (long long)INT_MAX + 1)) return false;
        return validateRB(root_) != -1;
    }

    vector<int> inorder() const {
        vector<int> out; inorder_(root_, out); return out;
    }

    vector<int> rangeSearch(int lo, int hi) const {
        vector<int> out; range_(root_, lo, hi, out); return out;
    }

    void print() const {
        function<void(RBNode*, string, bool)> pr =
            [&](RBNode* n, string pre, bool isLeft) {
                if (n == NIL_) return;
                pr(n->right, pre + (isLeft ? "│   " : "    "), false);
                cout << pre << (isLeft ? "└── " : "┌── ")
                     << n->val
                     << (n->color == Color::RED ? "(R)" : "(B)") << "\n";
                pr(n->left, pre + (isLeft ? "    " : "│   "), true);
            };
        pr(root_, "", true);
    }
};
```

---

## 10. Core Operations — Visualised

### Insertion Cases — All Three

```
Build tree by inserting: 10, 20, 30, 15, 25, 27, 50

Insert 10: root = 10(B). Root forced BLACK.
  10(B)

Insert 20: 20 > 10 → right. New node RED. Parent (10) is BLACK → no violation.
  10(B)
    \
    20(R)

Insert 30: 30 > 10 → right; 30 > 20 → right. New node RED. Parent 20 is RED.
  10(B)
    \
    20(R)
      \
      30(R) ← violation: 20 and 30 both red

fixup(30): parent=20(R), uncle=10->left=NIL(B).
  Uncle is BLACK, 30 is outer (right child of right parent) → Case 3:
  Recolor: 20→BLACK, 10→RED. Left-rotate 10.

After:
    20(B)
   /    \
  10(R) 30(R)

Insert 15: 15 > 10 (wait, 20 is root now). 15 < 20 → left; 15 > 10 → right. 
  New node 15 is RED. Parent 10 is RED.

    20(B)
   /    \
  10(R) 30(R)
    \
    15(R) ← violation: 10 and 15 both red

fixup(15): parent=10(R), grandparent=20(B), uncle=30(R).
  Uncle is RED → Case 1: recolor 10→B, 30→B, 20→R. Move up: z=20.
  z=20=root: root forced BLACK. Done.

After:
    20(B)
   /    \
  10(B) 30(B)
    \
    15(R) ← valid: 10 is black, 15 is red, 10's children OK

Insert 25: 25 > 20 → right; 25 < 30 → left. New node RED. Parent 30 is BLACK → no violation.
    20(B)
   /    \
  10(B) 30(B)
    \   /
   15(R)25(R)

Insert 27: 27 > 20 → right; 27 < 30 → left; 27 > 25 → right. Parent 25 is RED.

    20(B)
   /    \
  10(B) 30(B)
    \   /
   15(R)25(R)
          \
          27(R) ← violation

fixup(27): parent=25(R), gp=30(B), uncle=30->left=NIL? No: 30's left is 25, right is NIL(B).
  Wait: 30 is a node with left=25 and right=NIL(B). Uncle of 27 = 30->right = NIL(B).
  Uncle is BLACK. 27 is right child of 25 (right child of 30's left) → inner? No.
  25 is the LEFT child of 30 (parent 25->parent = 30, 25 = 30->left).
  27 is the RIGHT child of 25.
  So: parent=25 is left child; z=27 is right child → Case 2 (inner).

  Case 2: left-rotate parent (25). z moves up.
    30(B)                      
   /                           
  27(R)  ← 27 is now left child of 30
 /                            
25(R)   ← 25 is left child of 27

  Now z=25 (the old z=27 got swapped in Case 2). 
  Actually after Case 2, z becomes the parent: z=25 was parent, now z=old-z's parent.
  Per algorithm: after rotateLeft(z->parent), z = z->parent (= 25).
  
  Actually: z starts as 27. After rotateLeft(25): z becomes the parent of 25, which is z→parent.
  Per CLRS: "z = z->parent; rotateLeft(z);" → z becomes 25 after leftRotate(25).
  Then continue to Case 3 with z=25.

  Case 3 on z=25: z is left child of 27 which is left child of 30 → outer.
  Recolor 27(parent)→BLACK, 30(gp)→RED. Right-rotate 30.

  30(B)→30(R), 27(R)→27(B), right-rotate 30:

      20(B)
     /    \
   10(B)  27(B) ← was 30's position
     \   /    \
    15(R)25(R) 30(R)

Insert 50: 50>20→right; 50>27→right; 50>30→right. Parent 30 is RED. 
  Uncle = 27->left = 25(R).
  Uncle is RED → Case 1: recolor 30→B, 25→B, 27→R. z=27.
  27's parent = 20(B). 27(R) and 20(B): no violation! Done.

Final RB tree:
      20(B)
     /    \
   10(B)  27(R)
     \   /    \
    15(R)25(B) 30(B)
                 \
                 50(R)

Verify all 5 properties:
  1. All nodes colored ✓
  2. Root 20 is BLACK ✓
  3. All null leaves are BLACK (NIL sentinel) ✓
  4. Red nodes: 15,27,50. Their children: 15→(NIL,NIL)✓, 27→(25B,30B)✓, 50→(NIL,NIL)✓
  5. Black-heights from root:
     20→10→NIL: B{20,10,NIL}→2 black nodes (20,10)
     20→10→15→NIL: B{20,10,NIL}→2 (10 and NIL)? Wait:
     Count blacks on path to null NOT including the node itself.
     Path 20→10→NIL(left): 20 is black(+1), 10 is black(+1), NIL counted as 1. bh from root = 2.
     Path 20→10→15→NIL: 20(+1), 10(+1), 15 red(+0), NIL = bh from root = 2. ✓
     Path 20→27→25→NIL: 20(+1), 27 red(+0), 25(+1), NIL = 2. ✓
     Path 20→27→30→50→NIL: 20(+1), 27(0), 30(+1), 50(0), NIL = 2. ✓
  All black-heights = 2 ✓
```

---

## 11. Interview Problems

### Problem 1: Explain Why std::map Guarantees O(log n)

**Context:** This is a common conceptual interview question. The expected answer demonstrates understanding of red-black trees, not just "it's a balanced BST."

```cpp
/*
std::map<K,V> in every major C++ standard library (GCC's libstdc++, Clang's libc++,
MSVC's STL) is implemented as a red-black tree.

Proof of O(log n) for std::map operations:

1. Height bound: An RB tree of n nodes has height h ≤ 2 log₂(n+1).
   → Every operation that traverses a root-to-leaf path takes O(log n).

2. insert() — O(log n):
   - BST insert: O(h) = O(log n) to find insertion point.
   - insertFixup: O(log n) recolorings (moving up one level per Case 1 iteration)
                  + at most 2 rotations (O(1) each) = O(log n) total.

3. erase() — O(log n):
   - Find node: O(h) = O(log n).
   - BST deletion: O(log n) to find successor.
   - deleteFixup: O(log n) recolorings + at most 3 rotations = O(log n) total.

4. find()/count()/lower_bound()/upper_bound() — O(log n):
   - All are BST searches: O(h) = O(log n).

5. Iteration (begin to end) — O(n):
   - Inorder traversal of all n nodes: O(n).
   - single ++iterator: O(log n) amortised (O(1) amortised with parent pointers).

Why not hash map?
   std::unordered_map gives O(1) average but:
   - No ordering, so lower_bound/upper_bound/range queries are O(n).
   - Worst case O(n) per operation (hash flooding).
   - std::map provides O(log n) GUARANTEED, plus range operations.

API summary:
   map.insert({k,v})  O(log n)
   map.erase(k)       O(log n)
   map[k]             O(log n) (inserts if not present!)
   map.find(k)        O(log n)
   map.lower_bound(k) O(log n) — first key ≥ k
   map.upper_bound(k) O(log n) — first key > k
   map.begin()        O(1)     — iterator to smallest key
   map.rbegin()       O(1)     — iterator to largest key
*/

// Demonstrating lower_bound for range queries with std::map:
void demonstrateMapRangeQuery() {
    map<int,string> m;
    m[1]="a"; m[3]="b"; m[5]="c"; m[7]="d"; m[9]="e";

    // All keys in [3, 7] — O(log n + k)
    auto lo = m.lower_bound(3);   // first key ≥ 3
    auto hi = m.upper_bound(7);   // first key > 7

    for (auto it = lo; it != hi; ++it) {
        cout << it->first << ":" << it->second << " ";  // 3:b 5:c 7:d
    }
}
```

---

### Problem 2: Count of Range Sum Queries — RB Tree Application

**Problem:** Given an integer array `nums`, return the count of range sums that lie in `[lower, upper]`. A range sum `S(i,j)` = `nums[i] + ... + nums[j]`.

**Example:** `nums = [-2, 5, -1]`, lower=-2, upper=2 → 3

**Why this uses an ordered set (RB tree internally):**

```cpp
// Time: O(n log n) using a sorted multiset (std::multiset = red-black tree)
// Key insight: S(i,j) = prefix[j+1] - prefix[i]. We want lower ≤ prefix[j+1]-prefix[i] ≤ upper.
// Rearranging: prefix[j+1]-upper ≤ prefix[i] ≤ prefix[j+1]-lower.
// For each prefix[j+1], count how many EARLIER prefix sums lie in this range.
// std::multiset supports this via lower_bound/upper_bound in O(log n).

int countRangeSum(vector<int>& nums, int lower, int upper) {
    multiset<long long> prefixes;   // internally a red-black tree
    prefixes.insert(0);             // empty prefix has sum 0

    long long sum = 0;
    int count = 0;

    for (int num : nums) {
        sum += num;

        // Count prefix sums in [sum-upper, sum-lower]
        // These correspond to subarrays ending here with sum in [lower, upper]
        auto lo_it = prefixes.lower_bound(sum - upper);   // O(log n) — RB tree search
        auto hi_it = prefixes.upper_bound(sum - lower);   // O(log n) — RB tree search
        count += (int)distance(lo_it, hi_it);

        prefixes.insert(sum);   // O(log n) — RB tree insert
    }
    return count;
}

/*
Trace: nums=[-2,5,-1], lower=-2, upper=2.
Prefixes (running): 0, -2, 3, 2.

sum=0: insert 0. prefixes={0}.
sum=-2: 
  lo = lower_bound(0-2=-2)=0→it at -2. hi=upper_bound(0-(-2)=2)→after 2 (none yet).
  Actually: lo=lower_bound(-2-upper=-2-2=-4) = lower_bound(-4) → first ≥ -4 = 0.
  hi=upper_bound(-2-lower=-2-(-2)=0) = upper_bound(0) → after 0 in {0}.
  count += distance(lo,hi) = 1. (the prefix 0 satisfies: -2-0=-2 ∈ [-2,2] ✓)
  insert -2. prefixes={-2,0}.
sum=3:
  lo=lower_bound(3-2=1). First ≥ 1 in {-2,0} → end. hi=upper_bound(3-(-2)=5)=end.
  count += 0.
  insert 3. prefixes={-2,0,3}.
sum=2:
  lo=lower_bound(2-2=0). First ≥ 0 in {-2,0,3} → 0. 
  hi=upper_bound(2-(-2)=4). First > 4 in {-2,0,3} → end (3 < 4 ≤ hi... wait)
  Actually upper_bound(4) in {-2,0,3} → end (no element > 4... no, 3 < 4 but we want first > 4).
  All elements ≤ 4: -2,0,3 → upper_bound(4) = end.
  count += distance(first ≥ 0, end) = 2 (elements 0 and 3).

Wait, let me recount. lower_bound(0)→iterator to 0. upper_bound(4)→iterator past 3.
distance(0, end of {-2,0,3}) = 2 elements {0, 3}. count+=2. Total count=1+0+2=3. ✓

This problem uses std::multiset (= red-black tree) for O(log n) range queries.
Without an ordered structure, you'd need O(n) to count elements in each range → O(n²) total.
*/
```

---

### Problem 3: Verify Red-Black Tree Properties

**Problem:** Given a binary tree where each node has a color attribute, verify that it is a valid red-black tree. Return the black-height if valid, -1 if invalid.

**This tests:** deep understanding of all five properties and their interaction.

```cpp
// Returns black-height of subtree rooted at node, or -1 if invalid
int verifyRBT(RBNode* node, RBNode* NIL) {
    // Property 3: null leaves are black
    if (node == NIL) return 1;   // null counts as 1 black for black-height

    // Property 4: red node has black children
    if (node->color == Color::RED) {
        if (node->left->color  != Color::BLACK) return -1;
        if (node->right->color != Color::BLACK) return -1;
    }

    // Property 5: both subtrees have equal black-height
    int lbh = verifyRBT(node->left,  NIL);
    int rbh = verifyRBT(node->right, NIL);

    if (lbh == -1 || rbh == -1) return -1;   // subtree already invalid
    if (lbh != rbh)              return -1;   // black-heights don't match

    // Return this node's contribution to black-height
    return lbh + (node->color == Color::BLACK ? 1 : 0);
}

bool isValidRedBlackTree(RBNode* root, RBNode* NIL) {
    // Property 2: root is black
    if (root != NIL && root->color != Color::BLACK) return false;

    // Also verify BST property
    function<bool(RBNode*, long long, long long)> validBST =
        [&](RBNode* n, long long lo, long long hi) -> bool {
            if (n == NIL) return true;
            if (n->val <= lo || n->val >= hi) return false;
            return validBST(n->left, lo, n->val)
                && validBST(n->right, n->val, hi);
        };

    if (!validBST(root, LLONG_MIN, LLONG_MAX)) return false;
    return verifyRBT(root, NIL) != -1;
}

/*
Common invalid trees and which property they violate:

(1) Root is RED → Property 2 violated.

(2) RED node with RED child:
       B
      /
     R
    /
   R   ← Property 4 violated (two consecutive reds)

(3) Unequal black-heights:
     B
    / \
   R   B   ← left bh=1, right bh=2. Property 5 violated.
       \
        B

(4) Valid-looking but BST invalid:
     B:5
    /   \
   R:3  R:7   ← but R:3's right child could violate BST if R:3->right > 5
*/
```

---

## 12. Real-World Uses

| Domain | System | Red-Black Detail |
|---|---|---|
| **C++ STL** | `std::map`, `std::set`, `std::multimap`, `std::multiset` | All four use red-black trees internally |
| **Java** | `TreeMap`, `TreeSet` | Java SE documentation explicitly specifies red-black tree |
| **Linux kernel** | CFS scheduler (`task_struct.run_node`) | Red-black tree sorted by `vruntime`; O(log n) per context switch |
| **Linux kernel** | Virtual memory (`vm_area_struct`) | Each process's memory-mapped regions in an RB tree |
| **Linux kernel** | Ext3/Ext4 directory lookup | B-tree variant inspired by red-black for directory entries |
| **Nginx** | Timer management | Red-black tree for scheduling connection timeouts |
| **Java** | `HashMap` (collision handling) | When a bucket chain exceeds 8 entries, it converts to a red-black tree |
| **MongoDB** | Index structures | WiredTiger uses RB trees for in-memory indexes |
| **epoll** (Linux) | File descriptor ready set | Red-black tree for O(log n) fd management |
| **OpenJDK** | GC remembered sets | RB trees tracking inter-generational references |

### The Linux Kernel's Red-Black Tree — Deep Dive

```c
// From include/linux/rbtree.h (Linux kernel source):
struct rb_node {
    unsigned long __rb_parent_color;  // parent pointer + color bit packed together!
    struct rb_node *rb_right;
    struct rb_node *rb_left;
} __attribute__((aligned(sizeof(long))));
// sizeof(rb_node) = 24 bytes on 64-bit systems.
// The color bit is stored in the LOWEST bit of the parent pointer
// (always zero in a valid pointer due to alignment).
// This saves 8 bytes per node compared to a separate color field.
// For the Linux scheduler with millions of runnable tasks: significant memory savings.

struct rb_root {
    struct rb_node *rb_node;   // just the root pointer
};

// The CFS scheduler's red-black tree usage:
struct cfs_rq {
    struct rb_root_cached tasks_timeline;  // RB tree of runnable tasks by vruntime
    // ...
};

// On every scheduler tick:
// 1. leftmost(tasks_timeline) = next task to run  [O(1) with cached leftmost]
// 2. rb_erase(current_task)                        [O(log n)]
// 3. update current_task->vruntime += delta        [O(1)]
// 4. rb_insert(current_task)                       [O(log n)]
// Total: O(log n) per scheduling decision, 1000 times per second per CPU.
```

### Java's HashMap — RB Tree as Overflow Handler

Since Java 8, `HashMap` bins that exceed 8 entries convert their linked list to a **TreeMap** (red-black tree):

```java
// java/util/HashMap.java (simplified):
static final int TREEIFY_THRESHOLD   = 8;  // list → tree when chain ≥ 8
static final int UNTREEIFY_THRESHOLD = 6;  // tree → list when chain ≤ 6

// A bin of a HashMap:
// - 0-7 entries: singly-linked list of nodes, O(chain_length) lookup
// - 8+ entries: red-black tree of nodes, O(log(chain_length)) lookup

// This hybrid design means:
// - Normal operation: O(1) average with constant-small chains
// - Pathological case (hash collision flooding): O(log n) per lookup
//   instead of O(n). RB tree is the safety net.
```

---

## 13. Edge Cases & Pitfalls

### Pitfall 1: Not Using a NIL Sentinel

```cpp
// WRONG: using nullptr for empty child pointers
void insertFixup_wrong(RBNode* z) {
    while (z->parent && z->parent->color == RED) {  // ← crashes on root
        // z->parent->parent could be null if z->parent is root
        RBNode* uncle = z->parent->parent->right;   // ← null dereference!
        // ...
    }
}

// The sentinel NIL eliminates ALL null checks:
// NIL->color is always BLACK.
// NIL->parent, NIL->left, NIL->right are all NIL (self-referential).
// The while loop condition `z->parent->color == RED` is safe:
// NIL->color = BLACK → loop terminates before dereferencing NIL->parent.

// CORRECT: with NIL sentinel, no null checks needed in fixup
void insertFixup(RBNode* z) {
    while (z->parent->color == Color::RED) {  // safe: NIL->color = BLACK
        // z->parent->parent is either a real node or NIL (if parent is root)
        // → still safe because we check parent's color first,
        //   and root's parent = NIL (BLACK) terminates the loop
    }
}
```

### Pitfall 2: Forgetting to Reset Root to BLACK After Insertions

```cpp
// After insertFixup, the root might have been recolored RED in Case 1.
// If the recoloring propagates all the way to the root, the root is red → Property 2 violated.

void insertFixup(RBNode* z) {
    while (z->parent->color == Color::RED) {
        // ... fixup cases ...
    }
    root_->color = Color::BLACK;  // ← ALWAYS set root black after fixup
    // Even if no violation occurred, this line is a no-op (root was already black).
    // If Case 1 propagated to root, this fixes Property 2.
}
```

### Pitfall 3: Wrong Case Identification — Confusing Inner and Outer

```cpp
// The distinction between Case 2 and Case 3:
// Case 2 (inner): z is on the "inside" relative to g (bends toward g)
// Case 3 (outer): z is on the "outside" relative to g (straight line)

// If parent is LEFT child of grandparent:
//   Case 2: z is RIGHT child of parent (inner, because right bends toward g's right)
//   Case 3: z is LEFT child of parent (outer, straight line going left)

// WRONG: swapping Case 2 and Case 3 logic
// This applies rotations in the wrong order, often breaking the tree structure.

// CORRECT: always check which side parent is, then which side z is.
if (z->parent == z->parent->parent->left) {   // parent is left child
    RBNode* uncle = z->parent->parent->right;
    if (uncle->color == RED) {
        // Case 1: recolor
    } else {
        if (z == z->parent->right) {           // z is inner (right of left)
            z = z->parent;
            rotateLeft(z);                      // Case 2: left-rotate parent
        }
        // Now z is outer (left of left) → Case 3
        z->parent->color         = BLACK;
        z->parent->parent->color = RED;
        rotateRight(z->parent->parent);
    }
}
```

### Pitfall 4: Transplant Without Updating NIL's Parent

```cpp
// When transplanting with NIL (deleting a node with no children):
// NIL->parent must be updated to point to the deleted node's parent.
// This is needed by deleteFixup which may traverse upward from NIL.

void transplant(RBNode* u, RBNode* v) {
    if (u->parent == NIL_)       root_              = v;
    else if (u == u->parent->left) u->parent->left  = v;
    else                            u->parent->right = v;
    v->parent = u->parent;   // ← updates NIL_->parent when v=NIL_
    // Without this: deleteFixup(NIL) cannot find the parent to check the sibling.
}
```

### Pitfall 5: deleteFixup on Wrong Node

```cpp
// The double-black node after deletion is NOT always the node we deleted.
// It is the node that MOVED INTO the deleted position.

// Correct logic:
if (z->left == NIL_) {
    x = z->right;    // x moves up into z's position
    transplant(z, z->right);
} else if (z->right == NIL_) {
    x = z->left;
    transplant(z, z->left);
} else {
    y = minimum(z->right);       // y = inorder successor
    yOriginalColor = y->color;
    x = y->right;                // x is what moved into y's position
    // ...
}
// deleteFixup is called on x, not on z.
if (yOriginalColor == BLACK) deleteFixup(x);
// If y was RED: no black-height change (red node removed → no fixup needed).
// If y was BLACK: x (which replaced y) is double-black → fixup needed.
```

### Pitfall 6: Checking Color of Freed Node

```cpp
// WRONG: accessing z->color after deleting z
delete z;
if (yOriginalColor == BLACK) deleteFixup(x);  // safe: yOriginalColor saved before delete

// WRONG arrangement:
delete z;
if (z->color == BLACK) deleteFixup(x);   // USE AFTER FREE: z was deleted!

// CORRECT: save the original color before any deletion
Color yOriginalColor = y->color;   // save before anything changes
// ... delete z ...
if (yOriginalColor == BLACK) deleteFixup(x);
```

---

## 14. Red-Black vs AVL vs B-Tree vs Skip List

| Property | AVL Tree | Red-Black Tree | B-Tree | Skip List |
|---|---|---|---|---|
| Height bound | 1.44 log₂(n) | 2 log₂(n+1) | O(log_B n) | O(log n) expected |
| Insert rotations | ≤ 1 | ≤ 2 | O(B) splits | 0 |
| Delete rotations | O(log n) | ≤ 3 | O(B) merges | 0 |
| Lookup | Fewer levels | More levels | Fewer I/Os | Good expected |
| Read heavy | **★★★ Best** | ★★ Good | ★★★ Best (disk) | ★★ Good |
| Write heavy | ★★ Moderate | **★★★ Best** | ★★ Good | ★★★ Good |
| Concurrency | Hard (many rotations) | Moderate | **★★★ Best** | **★★★ Best** |
| Memory | +4B height/node | +1 bit color/node | Large pages | Variable |
| Implementation | Moderate | **Most complex** | Complex | Moderate |
| STL/stdlib | Rare | **`std::map`/`set`** | SQLite, ext4 | Redis |

**The Practical Decision:**

```
For general-purpose ordered containers in a single-threaded application:
  → std::map (Red-Black) is the default choice.
  → AVL is better if reads dominate writes by >10:1.

For concurrent ordered containers:
  → Skip lists (Redis sorted set, Java ConcurrentSkipListMap) are better.
    Skip lists allow lock-free concurrent operations more easily.
    B-trees/B+ trees with latch coupling also scale well.

For disk-based indexes (databases, file systems):
  → B-tree / B+ tree. Minimise I/O by maximising fan-out per page.
    Red-black and AVL have too much overhead per node for disk.

For in-memory with extremely tight memory budget:
  → Red-Black: only 1 color bit per node vs AVL's 4-byte height field.
  → Linux kernel: packs color bit into parent pointer (saves 8B/node vs naive impl).
```

---

## 15. Self-Test Questions

1. **State all five red-black properties from memory. For each, explain which structural invariant it enforces and which failure mode it prevents.**

2. **Prove that a red-black tree of n nodes has height at most 2 log₂(n+1). Your proof must use the black-height lemma as an intermediate step.**

3. **In insertion fixup Case 1, why does recoloring the uncle and grandparent move the violation upward without creating new violations lower in the tree?**

4. **Why must new nodes be colored RED and not BLACK on insertion? What property would be violated if we inserted BLACK nodes?**

5. **Trace the insertion of [41, 38, 31, 12, 19, 8] into an empty RB tree. Show the tree state and identify each fixup case triggered after each insertion.**

6. **In the Linux kernel's `rb_node`, the color bit is stored in the lowest bit of the parent pointer. Why is this safe? What alignment guarantee makes it possible?**

7. **Red-Black deletion may require up to 3 rotations, but AVL deletion may require O(log n). Explain why RB tree achieves fewer rotations. What property of the RB invariant enables this?**

8. **Java's HashMap uses linked lists for buckets with ≤ 8 entries and red-black trees for buckets with > 8 entries. What is the exact lookup time complexity when a key hashes to a bucket with n' entries?**

9. **Implement `verifyRBT(root)` without using recursion (iterative). What data structure do you need? What is the time and space complexity?**

10. **You have an AVL tree and a red-black tree, both with 10^6 entries. You perform 10^6 searches and 10^6 insertions, interleaved randomly. Which has better real-world performance and why? What measurements would you take to verify this empirically?**

---

## Quick Reference Card

```
Red-Black Tree — self-balancing BST with O(log n) guaranteed. Height ≤ 2 log₂(n+1).
                 Default implementation of std::map, std::set, Java TreeMap.

Five Properties:
  1. Every node is RED or BLACK.
  2. Root is BLACK.
  3. All null leaves are BLACK (use NIL sentinel).
  4. RED node's children are both BLACK (no two consecutive reds).
  5. All paths from any node to null leaves have the same black-height.

Key concepts:
  Black-height bh(x): # black nodes on path from x to null, not including x.
  NIL sentinel: single BLACK node shared by all null positions.
  Parent pointer: required for bottom-up fixup traversal.

Insertion: color new node RED (preserves black-height). Fix Property 4 violations.
  Three cases (+ mirrors):
    Case 1: Uncle RED → recolor uncle+parent BLACK, gp RED. Move up.
    Case 2: Uncle BLACK, z inner child → rotate parent. Becomes Case 3.
    Case 3: Uncle BLACK, z outer child → rotate gp, swap colors. Done.
  At most 2 rotations per insertion. O(log n) recolorings.
  Always: root->color = BLACK after fixup.

Deletion: save original color of removed/replaced node.
  Four cases for double-black fixup (+ mirrors):
    Case 1: Sibling RED → recolor+rotate. Transform to 2/3/4.
    Case 2: Sibling BLACK, both nephews BLACK → recolor sibling RED. Move up.
    Case 3: Sibling BLACK, inner nephew RED → rotate sibling, becomes Case 4.
    Case 4: Sibling BLACK, outer nephew RED → rotate+recolor. Done.
  At most 3 rotations per deletion.
  Call deleteFixup only if REMOVED node was BLACK.

Height: h ≤ 2 log₂(n+1) because bh ≥ h/2 (no consecutive reds) and n ≥ 2^bh - 1.

Pitfall checklist:
  ✗ No NIL sentinel → null dereferences in fixup code
  ✗ Not resetting root to BLACK after insertFixup
  ✗ Wrong Case 2/3 identification (inner vs outer confusion)
  ✗ transplant doesn't update NIL->parent → deleteFixup can't navigate
  ✗ deleteFixup called on z instead of x (the replacement node)
  ✗ Reading z->color after delete z → use-after-free

vs AVL: RB has looser balance (2x vs 1.44x) but fewer rotations per op.
        Prefer RB for write-heavy; prefer AVL for read-heavy.
        std::map uses RB. Most DB indexes use B-tree (not RB).
```

---

*Previous: [24 — AVL Tree](./24_avl_tree.md)*
*Next: [26 — Splay Tree](./26_splay_tree.md)*
*See also: [B-Tree](./trees/b_tree.md) | [Skip List](./structures/skip_list.md) | [Linux rbtree.h](https://github.com/torvalds/linux/blob/master/include/linux/rbtree.h)*

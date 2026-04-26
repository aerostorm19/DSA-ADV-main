# Monotonic Stack

> **Curriculum position:** Linear Structures → #8 (Advanced Pattern)
> **Interview weight:** ★★★★★ Critical — one of the highest-ROI patterns at FAANG interviews
> **Difficulty:** Intermediate-Advanced — the concept is simple; pattern recognition is the hard part
> **Prerequisite:** [07 — Stack](./07_stack.md)

---

## Table of Contents

1. [Intuition First — What Problem Does It Solve?](#1-intuition-first)
2. [The Core Invariant — What Makes It "Monotonic"](#2-the-core-invariant)
3. [The Four Variants](#3-the-four-variants)
4. [Why It Is O(n) — The Amortized Proof](#4-why-it-is-on)
5. [Time & Space Complexity](#5-time--space-complexity)
6. [Complete C++ Templates — All Four Variants](#6-complete-c-templates)
7. [Extended Patterns & Techniques](#7-extended-patterns--techniques)
8. [Interview Problems](#8-interview-problems)
9. [Real-World Uses](#9-real-world-uses)
10. [Edge Cases & Pitfalls](#10-edge-cases--pitfalls)
11. [Problem Recognition Guide](#11-problem-recognition-guide)
12. [Self-Test Questions](#12-self-test-questions)

---

## 1. Intuition First

Consider this problem: for each element in an array, find the nearest element to its right that is greater than it. Brute force is O(n²) — for each element, scan rightward until you find a larger one.

Can you do it in O(n)?

The key observation: when scanning left to right and you encounter a large element — say `10` after `[3, 1, 7, 2]` — you already know that `10` is the answer for every element to its left that is smaller than `10`. You do not need to come back and check them later. The moment `10` arrives, you can resolve all pending smaller elements simultaneously.

A **monotonic stack** is a stack that maintains a sorted invariant — either strictly increasing or strictly decreasing from bottom to top — by removing elements that violate the order the moment a new element arrives. Each removal corresponds to an answer being computed for the removed element.

The mental model: imagine people standing in a queue, each holding a number. Each person is waiting to find out "who is the next taller person behind me?" When a tall person arrives, they can immediately answer this question for every shorter person in front of them, all at once. Those shorter people leave the queue (get popped). The tall person then joins the queue and waits for their own answer.

**This is not a data structure you learn once and apply mechanically. It is a reasoning pattern.** The skill is recognising which of the four variants applies to a given problem, then applying the template. This document gives you all four templates and the recognition heuristic.

---

## 2. The Core Invariant — What Makes It "Monotonic"

A monotonic stack maintains one of two invariants at all times:

```
Monotonic increasing stack:   bottom → ... → top (values increase from bottom to top)
Monotonic decreasing stack:   bottom → ... → top (values decrease from bottom to top)
```

The invariant is **enforced on every push**: before pushing a new element, pop all elements that violate the invariant. Then push the new element.

```
Monotonic INCREASING stack — pop elements GREATER than or equal to the new element:

Push sequence: 3, 1, 4, 1, 5, 9, 2, 6

Push 3:  [3]
Push 1:  1 < 3 → pop 3; [1]          (3 violates increasing order)
Push 4:  4 > 1 → push; [1, 4]
Push 1:  1 ≤ 4 → pop 4; 1 ≤ 1 → pop 1; [] → push 1; [1]
Push 5:  [1, 5]
Push 9:  [1, 5, 9]
Push 2:  2 < 9 → pop 9; 2 < 5 → pop 5; 2 > 1 → push; [1, 2]
Push 6:  [1, 2, 6]

Monotonic DECREASING stack — pop elements LESS than or equal to the new element:

Push sequence: 3, 1, 4, 1, 5, 9, 2, 6

Push 3:  [3]
Push 1:  1 < 3 → no pop; [3, 1]
Push 4:  4 > 1 → pop 1; 4 > 3 → pop 3; [] → push; [4]
Push 1:  1 < 4 → no pop; [4, 1]
Push 5:  5 > 1 → pop 1; 5 > 4 → pop 4; [] → push; [5]
Push 9:  9 > 5 → pop 5; [] → push; [9]
Push 2:  2 < 9 → no pop; [9, 2]
Push 6:  6 > 2 → pop 2; 6 < 9 → stop; push; [9, 6]
```

**The critical insight: each pop is an answer.** When element `x` is popped because new element `y` arrived:
- In a decreasing stack: `y > x` → `y` is the **Next Greater Element** for `x`
- In an increasing stack: `y < x` → `y` is the **Next Smaller Element** for `x`

The element that caused the pop is always the answer for the popped element.

---

## 3. The Four Variants

Every monotonic stack problem is a combination of two choices:
- **Direction**: what is to the right of each element (Next) or left (Previous)?
- **Relation**: find the nearest Greater element or Smaller element?

This gives exactly four variants:

```
Variant                       Stack Type         Traversal    Answer Computed At
NGE — Next Greater Element    Decreasing         L → R        When element is POPPED
NSE — Next Smaller Element    Increasing         L → R        When element is POPPED
PGE — Previous Greater Elem   Decreasing         L → R        Stack TOP before PUSH
PSE — Previous Smaller Elem   Increasing         L → R        Stack TOP before PUSH
```

**The Unified Rule:**

```
To find Next Greater/Smaller → traverse L→R
  The element is answered when it gets POPPED by a newcomer
  The newcomer that caused the pop IS the answer

To find Previous Greater/Smaller → traverse L→R
  Inspect the STACK TOP before pushing the current element
  The surviving top IS the answer for the current element
```

### Visual Mapping

```
Array:    [2,  1,  5,  6,  2,  3]
Index:      0   1   2   3   4   5

NGE:      [5,  5,  6, -1,  3, -1]   (next element to right that is greater)
NSE:      [1, -1,  2,  2, -1, -1]   (next element to right that is smaller)
PGE:     [-1,  2,  6,  6,  6,  6]   (nearest element to left that is greater)
PSE:     [-1, -1,  1,  5,  1,  2]   (nearest element to left that is smaller)
```

---

## 4. Why It Is O(n) — The Amortized Proof

At first glance, the algorithm appears O(n²): for each of n elements, the inner `while` loop might pop O(n) elements. The key insight is that these costs are shared across iterations.

**Proof by accounting (token method):**

Assign each element two tokens when it is pushed: one for the push, one for a future pop.

- Every element is pushed exactly once → n push operations, n tokens spent
- Every element is popped at most once → at most n pop operations, n tokens spent
- Total tokens: 2n → Total operations: O(n)

```
Specifically: across the ENTIRE algorithm:
  Total pushes: exactly n (one per array element)
  Total pops:   at most n (each pushed element is popped at most once)
  The while loop's total work over the full algorithm = at most n iterations

The while loop does NOT run n times per outer iteration.
It runs O(1) amortised per outer iteration because each pop is "charged"
to the element being popped, and each element can only be charged once.
```

**Intuition:** Think of each element having a "budget" of 2 operations — one push and one pop. The while loop can only spend from existing budgets. Since total budget is 2n, total work is O(n).

---

## 5. Time & Space Complexity

| Variant | Time | Space | Notes |
|---|---|---|---|
| NGE (Next Greater Element) | **O(n)** | **O(n)** | Each element pushed and popped once |
| NSE (Next Smaller Element) | **O(n)** | **O(n)** | Same |
| PGE (Previous Greater Element) | **O(n)** | **O(n)** | Same |
| PSE (Previous Smaller Element) | **O(n)** | **O(n)** | Same |
| All four simultaneously | **O(n)** | **O(n)** | Two stacks, single pass |
| Circular array variant | **O(n)** | **O(n)** | Virtual doubling, one extra pass |
| Brute force (no stack) | O(n²) | O(1) | For comparison |

**Space breakdown:** Stack holds at most O(n) elements (worst case: sorted array). Result arrays: O(n). Total auxiliary: O(n). In practice, the stack rarely holds more than O(log n) elements for random input, but O(n) is the correct worst-case bound.

---

## 6. Complete C++ Templates — All Four Variants

```cpp
#include <vector>
#include <stack>
using namespace std;

// ── Universal skeleton — the core loop all variants share ─────────────────────
//
// for (int i = 0; i < n; i++) {
//     while (!stk.empty() && CONDITION(nums[stk.top()], nums[i])) {
//         int idx = stk.top(); stk.pop();
//         answer[idx] = nums[i];   // popped element's answer = current element
//     }
//     if (!stk.empty()) answer[i] = nums[stk.top()];   // for PGE/PSE only
//     stk.push(i);   // NEVER skip this line
// }
// Remaining elements in stk have no answer → keep default value (-1 or n).

// ── Variant 1: NGE — Next Greater Element ────────────────────────────────────
// Stack: decreasing. Pop when current > top. Answer set at pop time.
vector<int> nextGreaterElement(const vector<int>& nums) {
    int n = nums.size();
    vector<int> nge(n, -1);
    stack<int> stk;

    for (int i = 0; i < n; i++) {
        while (!stk.empty() && nums[i] > nums[stk.top()]) {
            nge[stk.top()] = nums[i];
            stk.pop();
        }
        stk.push(i);
    }
    return nge;
}

// ── Variant 2: NSE — Next Smaller Element ────────────────────────────────────
// Stack: increasing. Pop when current < top. Answer set at pop time.
vector<int> nextSmallerElement(const vector<int>& nums) {
    int n = nums.size();
    vector<int> nse(n, -1);
    stack<int> stk;

    for (int i = 0; i < n; i++) {
        while (!stk.empty() && nums[i] < nums[stk.top()]) {
            nse[stk.top()] = nums[i];
            stk.pop();
        }
        stk.push(i);
    }
    return nse;
}

// ── Variant 3: PGE — Previous Greater Element ────────────────────────────────
// Stack: decreasing. Answer = stack top AFTER cleanup, BEFORE push.
vector<int> previousGreaterElement(const vector<int>& nums) {
    int n = nums.size();
    vector<int> pge(n, -1);
    stack<int> stk;

    for (int i = 0; i < n; i++) {
        while (!stk.empty() && nums[stk.top()] <= nums[i]) stk.pop();
        if (!stk.empty()) pge[i] = nums[stk.top()];   // surviving top = PGE
        stk.push(i);
    }
    return pge;
}

// ── Variant 4: PSE — Previous Smaller Element ────────────────────────────────
// Stack: increasing. Answer = stack top AFTER cleanup, BEFORE push.
vector<int> previousSmallerElement(const vector<int>& nums) {
    int n = nums.size();
    vector<int> pse(n, -1);
    stack<int> stk;

    for (int i = 0; i < n; i++) {
        while (!stk.empty() && nums[stk.top()] >= nums[i]) stk.pop();
        if (!stk.empty()) pse[i] = nums[stk.top()];   // surviving top = PSE
        stk.push(i);
    }
    return pse;
}

// ── All Four in One Pass ──────────────────────────────────────────────────────
void allFour(const vector<int>& nums,
             vector<int>& nge, vector<int>& nse,
             vector<int>& pge, vector<int>& pse) {
    int n = nums.size();
    nge.assign(n,-1); nse.assign(n,-1);
    pge.assign(n,-1); pse.assign(n,-1);
    stack<int> dec, inc;

    for (int i = 0; i < n; i++) {
        // Decreasing stack handles NGE (on pop) and PGE (top before push)
        while (!dec.empty() && nums[i] > nums[dec.top()]) {
            nge[dec.top()] = nums[i]; dec.pop();
        }
        if (!dec.empty()) pge[i] = nums[dec.top()];
        dec.push(i);

        // Increasing stack handles NSE (on pop) and PSE (top before push)
        while (!inc.empty() && nums[i] < nums[inc.top()]) {
            nse[inc.top()] = nums[i]; inc.pop();
        }
        if (!inc.empty()) pse[i] = nums[inc.top()];
        inc.push(i);
    }
}

// ── Circular Array Variant ────────────────────────────────────────────────────
// Simulate the circle by iterating 0..2n-1 with index mod n.
// Only push real indices [0..n-1] on the first pass.
vector<int> nextGreaterCircular(const vector<int>& nums) {
    int n = nums.size();
    vector<int> nge(n, -1);
    stack<int> stk;

    for (int i = 0; i < 2 * n; i++) {
        while (!stk.empty() && nums[i % n] > nums[stk.top()]) {
            nge[stk.top()] = nums[i % n];
            stk.pop();
        }
        if (i < n) stk.push(i);   // only push real indices
    }
    return nge;
}
```

---

## 7. Extended Patterns & Techniques

### Technique 1: Always Push Indices, Not Values

```cpp
// WRONG: pushing values loses position information
stk.push(nums[i]);         // can't compute width = i - stk.top() - 1 later

// CORRECT: push indices; recover value as nums[stk.top()]
stk.push(i);
// Access value:    nums[stk.top()]
// Access index:    stk.top()
// Distance right:  i - stk.top()
// Width (exclusive): i - stk.top() - 1
```

### Technique 2: The Contribution Method

Instead of asking "what is the minimum of each subarray?", ask "for how many subarrays is `arr[i]` the minimum?" Then multiply.

```
For element arr[i]:
  L = index of nearest previous STRICTLY smaller element (PSE boundary)
  R = index of nearest next SMALLER OR EQUAL element (NSE boundary, exclusive)

  Subarrays where arr[i] is minimum:
    Left choices:  i - L          (endpoints from L+1 to i)
    Right choices: R - i          (endpoints from i to R-1)
    Count: (i - L) × (R - i)

  Contribution to total sum: arr[i] × (i - L) × (R - i)
```

Using strict `<` on one side and `<=` on the other prevents double-counting when duplicates exist.

### Technique 3: Monotonic Deque (Sliding Window Maximum)

When the problem involves a window that slides across the array, upgrade from stack to deque so elements can expire from the front.

```cpp
// Sliding window maximum — O(n)
vector<int> maxSlidingWindow(vector<int>& nums, int k) {
    deque<int> dq;   // indices; front = max in window (decreasing values)
    vector<int> result;

    for (int i = 0; i < (int)nums.size(); i++) {
        // Expire elements outside the window
        while (!dq.empty() && dq.front() <= i - k) dq.pop_front();
        // Maintain decreasing order (pop smaller elements from back)
        while (!dq.empty() && nums[dq.back()] <= nums[i]) dq.pop_back();
        dq.push_back(i);
        if (i >= k - 1) result.push_back(nums[dq.front()]);
    }
    return result;
}
```

### Technique 4: Stock Span — Online Processing With Accumulated Spans

When answering queries one at a time (streaming), store the span accumulated so far alongside each element. Popped spans are absorbed into the current span.

```cpp
class StockSpanner {
private:
    stack<pair<int,int>> stk;   // {price, span}
public:
    int next(int price) {
        int span = 1;
        while (!stk.empty() && stk.top().first <= price) {
            span += stk.top().second;   // absorb popped element's span
            stk.pop();
        }
        stk.push({price, span});
        return span;
    }
};

/*
Trace for [100,80,60,70,60,85,100]:
next(100): span=1. push {100,1}.     stk:[(100,1)]
next(80):  80≤100? no pop. span=1. push {80,1}.   stk:[(100,1),(80,1)]
next(60):  span=1. push {60,1}.     stk:[(100,1),(80,1),(60,1)]
next(70):  70≥60→pop {60,1} span=2. 70≤80? no pop. push {70,2}. → span=2
next(60):  span=1. push {60,1}.     stk:[(100,1),(80,1),(70,2),(60,1)]
next(85):  85≥60→pop span=2. 85≥70→pop span=4. 85≥80→pop span=5. 85≤100. push {85,5}. → span=5
next(100): 100≥85→pop span=6. 100≥100→pop span=7. push {100,7}. → span=7

Spans: [1,1,1,2,1,5,7]
*/
```

---

## 8. Interview Problems

### Problem 1: Daily Temperatures

**Problem:** For each day, find how many days until a warmer temperature. If no future day is warmer, output `0`.

**Example:** `[73,74,75,71,69,72,76,73]` → `[1,1,4,2,1,1,0,0]`

**Thought process:** For each day, find the nearest future day with strictly higher temperature — this is NGE by distance. Monotonic decreasing stack, traverse left to right, the index distance at pop time is the answer.

```cpp
// Time: O(n) | Space: O(n)
vector<int> dailyTemperatures(vector<int>& T) {
    int n = T.size();
    vector<int> result(n, 0);
    stack<int> stk;

    for (int i = 0; i < n; i++) {
        while (!stk.empty() && T[i] > T[stk.top()]) {
            int idx = stk.top(); stk.pop();
            result[idx] = i - idx;   // distance to warmer day
        }
        stk.push(i);
    }
    return result;
}

/*
Trace on [73,74,75,71,69,72,76,73]:

i=0: T=73. push 0.                    stk:[0(73)]
i=1: T=74>73 → pop 0: result[0]=1-0=1. push 1.      stk:[1(74)]
i=2: T=75>74 → pop 1: result[1]=2-1=1. push 2.      stk:[2(75)]
i=3: T=71<75. push 3.                 stk:[2(75),3(71)]
i=4: T=69<71. push 4.                 stk:[2(75),3(71),4(69)]
i=5: T=72>69 → pop 4: result[4]=5-4=1.
          72>71 → pop 3: result[3]=5-3=2.
          72<75. push 5.              stk:[2(75),5(72)]
i=6: T=76>72 → pop 5: result[5]=6-5=1.
          76>75 → pop 2: result[2]=6-2=4.
          push 6.                     stk:[6(76)]
i=7: T=73<76. push 7.                 stk:[6(76),7(73)]

Remaining: indices 6,7 → result stays 0.
Result: [1,1,4,2,1,1,0,0] ✓
*/
```

**Edge cases:**
- All decreasing `[5,4,3,2,1]` → all zeros; nothing ever pops anything
- All equal `[70,70,70]` → all zeros; `70 > 70` is false
- Single element → `[0]`
- All strictly increasing → each element is answered by the very next one: `[1,1,...,1,0]`

---

### Problem 2: Trapping Rain Water

**Problem:** Given elevation heights, compute how much water is trapped after rain.

**Example:** `[0,1,0,2,1,0,1,3,2,1,2,1]` → `6`

**Why a monotonic stack:** Water fills valleys. A valley exists between a left wall and a right wall. The moment we find a bar taller than the stack top, we've found the right wall of a trapped layer — the stack top is the valley bottom, and the new stack top (after popping) is the left wall.

```cpp
// Time: O(n) | Space: O(n)
// Computes water layer by layer as the stack unwinds
int trap(vector<int>& height) {
    int n = height.size(), water = 0;
    stack<int> stk;

    for (int i = 0; i < n; i++) {
        while (!stk.empty() && height[i] > height[stk.top()]) {
            int bottom_idx = stk.top(); stk.pop();
            if (stk.empty()) break;           // no left wall → no water

            int left_idx  = stk.top();
            int h = min(height[left_idx], height[i]) - height[bottom_idx];
            int w = i - left_idx - 1;
            water += h * w;
        }
        stk.push(i);
    }
    return water;
}

/*
Key trace points for [0,1,0,2,1,0,1,3,2,1,2,1]:

i=3 h=2: pops idx=2 (h=0). left=1(h=1), right=3(h=2).
  water += (min(1,2)-0) × (3-1-1) = 1×1 = 1.  total=1.

i=7 h=3: eventually pops idx=4 (h=1). left=3(h=2), right=7(h=3).
  water += (min(2,3)-1) × (7-3-1) = 1×3 = 3.  total=5.

i=10 h=2: pops idx=9 (h=1). left=8(h=2), right=10(h=2).
  water += (min(2,2)-1) × (10-8-1) = 1×1 = 1.  total=6.

Final water = 6 ✓
*/

// ── Two-pointer alternative — O(n) time, O(1) space ─────────────────────────
// For each position: water = min(maxLeft, maxRight) - height[i]
// Process the side with the lower max first — that max is definitively bounded.
int trapTwoPointers(vector<int>& height) {
    int left = 0, right = (int)height.size() - 1;
    int maxL = 0, maxR = 0, water = 0;
    while (left < right) {
        if (height[left] < height[right]) {
            maxL = max(maxL, height[left]);
            water += maxL - height[left++];
        } else {
            maxR = max(maxR, height[right]);
            water += maxR - height[right--];
        }
    }
    return water;
}
```

**Present both in interviews.** The stack solution demonstrates the monotonic pattern. The two-pointer solution shows space optimisation. Strong candidates explain both and state when each is preferable.

**Edge cases:**
- Monotonically increasing or decreasing → no trapped water, answer is 0
- All same height → no water
- Two elements → no water (need at least one valley between two walls)
- A single tall bar in the middle → water on both sides computed independently

---

### Problem 3: Sum of Subarray Minimums

**Problem:** Given array `arr`, find the sum of `min(subarray)` for every possible subarray. Return modulo `10^9 + 7`.

**Example:** `[3,1,2,4]` → 17

**Thought process — the contribution shift:**

> "Brute force enumerates O(n²) subarrays and finds each minimum: O(n³) or O(n²) with preprocessing. Too slow for n = 30,000."
>
> "Inversion: instead of 'what is the minimum of each subarray?', ask 'for how many subarrays is arr[i] the minimum?' Multiply arr[i] by that count to get its contribution. Sum all contributions."
>
> "arr[i] is the minimum of subarray [l,r] when it is the smallest element in that range. The left boundary is the previous strictly smaller element (PSE). The right boundary is the next smaller or equal element (NSE). Number of subarrays = (i - PSE) × (NSE - i)."

```cpp
// Time: O(n) | Space: O(n)
int sumSubarrayMins(vector<int>& arr) {
    int n = arr.size();
    const int MOD = 1e9 + 7;
    vector<long long> left(n), right(n);
    stack<int> stk;

    // left[i] = distance to previous STRICTLY smaller (or left edge if none)
    // Use >= to pop equals: leftmost duplicate "owns" the subarray
    for (int i = 0; i < n; i++) {
        while (!stk.empty() && arr[stk.top()] >= arr[i]) stk.pop();
        left[i] = stk.empty() ? i + 1 : i - stk.top();
        stk.push(i);
    }
    while (!stk.empty()) stk.pop();

    // right[i] = distance to next SMALLER OR EQUAL (or right edge if none)
    // Use > to pop only strictly greater: rightmost duplicate lets left one win
    for (int i = n - 1; i >= 0; i--) {
        while (!stk.empty() && arr[stk.top()] > arr[i]) stk.pop();
        right[i] = stk.empty() ? n - i : stk.top() - i;
        stk.push(i);
    }

    long long ans = 0;
    for (int i = 0; i < n; i++) {
        // arr[i] contributes to left[i] × right[i] subarrays as their minimum
        ans = (ans + (long long)arr[i] % MOD * left[i] % MOD * right[i]) % MOD;
    }
    return (int)ans;
}

/*
Trace on [3,1,2,4]:

left[] computation (PSE strictly smaller):
  i=0: arr=3. stk empty → left[0]=0+1=1. push 0.
  i=1: arr=1. 3≥1 → pop 0. stk empty → left[1]=1+1=2. push 1.
  i=2: arr=2. 1<2 → stop. left[2]=2-1=1. push 2.
  i=3: arr=4. 2<4 → stop. left[3]=3-2=1. push 3.
  left = [1, 2, 1, 1]

right[] computation (NSE smaller or equal):
  i=3: arr=4. stk empty → right[3]=4-3=1. push 3.
  i=2: arr=2. 4>2 → pop 3. stk empty → right[2]=4-2=2. push 2.
  i=1: arr=1. 2>1 → pop 2. stk empty → right[1]=4-1=3. push 1.
  i=0: arr=3. 1≤3 → stop. right[0]=1-0=1. push 0.
  right = [1, 3, 2, 1]

Contributions:
  i=0: 3 × 1 × 1 = 3
  i=1: 1 × 2 × 3 = 6
  i=2: 2 × 1 × 2 = 4
  i=3: 4 × 1 × 1 = 4
  Total = 17 ✓
*/
```

**The duplicate handling asymmetry — explained precisely:**

For `[2, 2, 2]`, subarray `[2,2,2]` should be counted once with minimum 2. Using `>=` on the left and `>` on the right assigns ownership to the leftmost copy:
- `left[0]=1, right[0]=3` → arr[0]=2 contributes to 3 subarrays
- `left[1]=1, right[1]=2` → arr[1]=2 contributes to 2 subarrays
- `left[2]=1, right[2]=1` → arr[2]=2 contributes to 1 subarray
- Total: 6 subarrays × 2 = 12 = 2+2+2+2+2+2 ✓ (all 6 subarrays of a 3-element array)

Using `>=` on both sides would double-count subarrays containing multiple equal minimums.

**Edge cases:**
- Single element `[x]` → answer = x
- All equal `[v,v,...,v]` → sum = v × n(n+1)/2 (each of n(n+1)/2 subarrays has min v)
- Very large values → every intermediate multiplication needs `% MOD` and `(long long)` cast

---

## 9. Real-World Uses

| Domain | Use Case | Variant |
|---|---|---|
| Financial systems | Online stock span: days since last higher price | Decreasing + span accumulation |
| Trading algorithms | Support/resistance: nearest higher/lower candle | All four variants |
| Compilers | Operator precedence in expression parsing | Decreasing (Shunting-Yard algorithm) |
| Database engines | Query plan pruning: dominant index elimination | NSE for range restriction |
| Image processing | Skyline generation, histogram segmentation | NSE/PSE for bar decomposition |
| Game engines | Field-of-view / line-of-sight in 2D maps | Decreasing (blocked by taller walls) |
| Weather systems | Temperature inversion detection | NGE on vertical temperature profiles |
| Cartographic tools | Horizon computation from an elevation profile | Decreasing stack sweepline |
| Video encoding | Scene change detection via frame difference | NSE on quality/difference sequences |
| Distributed systems | Rate limiting with sliding window maximum | Monotonic deque |

**The Shunting-Yard algorithm** — used in every expression parser and calculator — is a monotonic stack application. Operators are maintained on a stack in decreasing precedence order. A new operator pops all higher-or-equal-precedence operators (evaluating them) before being pushed, exactly like the NGE template. Every time you type `3 + 4 * 2` into a calculator and get 11 (not 14), a monotonic operator stack ran.

---

## 10. Edge Cases & Pitfalls

### Pitfall 1: Strict vs Non-Strict — The Duplicate Problem

```cpp
// The single most common bug in monotonic stack problems.

// For NGE (next STRICTLY greater): use strict >
while (!stk.empty() && nums[i] > nums[stk.top()])  // correct: 5 is not NGE of 5

// For "next greater or equal": use >=
while (!stk.empty() && nums[i] >= nums[stk.top()]) // correct

// For Sum of Subarray Minimums — asymmetry is intentional:
// Left (PSE): pop when >= (strict, so leftmost copy owns the range)
// Right (NSE): pop when >  (non-strict, so rightmost copy does not steal)
// Swapping these or making both sides equal causes double-counting.
```

### Pitfall 2: Forgetting stk.push(i) After the While Loop

```cpp
// This is the single most common implementation bug.
for (int i = 0; i < n; i++) {
    while (!stk.empty() && nums[i] > nums[stk.top()]) {
        nge[stk.top()] = nums[i]; stk.pop();
    }
    // MISSING: stk.push(i)
    // Without this, i never waits for ITS OWN answer.
    // The stack stays permanently empty after the first element is popped.
}

// CORRECT: always push after the while loop, unconditionally.
stk.push(i);
```

### Pitfall 3: Checking stk.empty() After pop() in Trapping Rain Water

```cpp
// After popping the valley bottom, the left wall is stk.top().
// But the pop may have emptied the stack — then there is no left wall.

while (!stk.empty() && height[i] > height[stk.top()]) {
    int bottom = stk.top(); stk.pop();
    if (stk.empty()) break;   // ← critical: missing this → crash on stk.top()
    int left = stk.top();
    water += ...;
}
```

### Pitfall 4: Off-by-One in Width Formula

```cpp
// Valley between left_idx and right_idx (the walls, not included in water):
// Width = right_idx - left_idx - 1

int width = right_idx - left_idx;       // WRONG: includes one wall
int width = right_idx - left_idx + 1;   // WRONG: includes both walls
int width = right_idx - left_idx - 1;   // CORRECT: only the valley between

// Mnemonic: "walls are not water" — subtract both wall indices from the span.
```

### Pitfall 5: Circular Array — Pushing on the Second Pass

```cpp
// WRONG: pushes indices 0..n-1 twice, corrupting the stack
for (int i = 0; i < 2 * n; i++) {
    while (!stk.empty() && nums[i%n] > nums[stk.top()]) { ... stk.pop(); }
    stk.push(i % n);   // pushes each index twice
}

// CORRECT: only push during the first pass (i < n)
for (int i = 0; i < 2 * n; i++) {
    while (!stk.empty() && nums[i%n] > nums[stk.top()]) { ... stk.pop(); }
    if (i < n) stk.push(i);   // real indices only, once
}
```

### Pitfall 6: Overflow in Contribution Calculation

```cpp
// WRONG: intermediate product overflows int before modulo is applied
ans += arr[i] * left[i] * right[i] % MOD;
// For arr[i]=30000, left[i]=30000, right[i]=30000:
// 30000 × 30000 = 9×10^8 → overflows int (max ~2.1×10^9 for unsigned, ~1×10^9 for signed)
// Then × 30000 = 2.7×10^13 → severe overflow → wrong answer

// CORRECT: cast to long long at the first multiplication
ans = (ans + (long long)arr[i] * left[i] % MOD * right[i] % MOD) % MOD;
```

### Pitfall 7: Confusing NGE with PGE — Which Direction?

```cpp
// NGE: the element is POPPED by its answer — answer comes from the RIGHT
// PGE: the stack TOP (before push) is the answer — answer comes from the LEFT

// WRONG for PGE: recording the answer when elements are popped
while (!stk.empty() && nums[i] > nums[stk.top()]) {
    pge[stk.top()] = nums[i];   // WRONG: this gives NGE, not PGE
    stk.pop();
}

// CORRECT for PGE: record answer before pushing (the surviving top is the left neighbour)
while (!stk.empty() && nums[stk.top()] <= nums[i]) stk.pop();  // clean up smaller
if (!stk.empty()) pge[i] = nums[stk.top()];   // surviving top IS the PGE
stk.push(i);
```

---

## 11. Problem Recognition Guide

```
Does each element need to know about the nearest element satisfying some comparison?
  YES → Monotonic stack

  Is the comparison with elements to the RIGHT?
    → NGE or NSE. Traverse L→R. Answer recorded when element is POPPED.
    → NGE (find larger): decreasing stack, pop when current > top
    → NSE (find smaller): increasing stack, pop when current < top

  Is the comparison with elements to the LEFT?
    → PGE or PSE. Traverse L→R. Answer is STACK TOP before push.
    → PGE (find larger): decreasing stack, pop ≤ current, then read top
    → PSE (find smaller): increasing stack, pop ≥ current, then read top

Does the problem sum something over all subarrays based on min/max?
  → Contribution method = PSE + NSE for each element

Does the problem involve a sliding window with max/min?
  → Monotonic DEQUE (not stack — need to expire front elements)

Signal words:
  "next greater/warmer/taller/higher"  → NGE
  "next smaller/lower/closer to zero"  → NSE
  "previous greater"                   → PGE
  "previous smaller"                   → PSE
  "days until", "distance to"          → NGE or NSE (return index difference)
  "water trapped", "fill", "rain"      → NSE on both sides for walls
  "rectangle area", "largest bar"      → NSE + PSE for width boundaries
  "sum of subarray min/max"            → contribution method
  "stock span", "consecutive days"     → PGE with span accumulation
  "sliding window max/min"             → monotonic deque
  "visibility", "skyline", "blocked"   → decreasing stack

Problem → variant mapping:
  Daily Temperatures                   → NGE (distance)
  Next Greater Element I, II           → NGE (II is circular)
  Trapping Rain Water                  → NSE on both sides
  Largest Rectangle in Histogram       → NSE + PSE (or combined)
  Maximum Rectangle in Binary Matrix   → per-row histogram + above
  Sum of Subarray Minimums             → contribution (PSE + NSE)
  Sum of Subarray Ranges               → contribution (all four)
  Online Stock Span                    → PGE with span
  Remove K Digits (smallest number)    → increasing stack
  132 Pattern                          → decreasing + running min
  Sliding Window Maximum               → monotonic deque
```

---

## 12. Self-Test Questions

1. **State the invariant of a monotonic decreasing stack. What is the pop condition when pushing a new element?**

2. **Prove that the total number of push + pop operations across the full algorithm is O(n). Why doesn't the inner while loop make it O(n²)?**

3. **Write the NGE template from memory. What are: (a) the stack type, (b) the traversal direction, (c) the pop condition, (d) when the answer is recorded?**

4. **What is the difference between NGE and PGE in terms of when the answer is computed? Trace both on `[4, 5, 2, 10, 8]`.**

5. **In Sum of Subarray Minimums, why must you use `>=` on the left and `>` on the right for duplicate handling? What goes wrong if you use `>=` on both sides? Demonstrate with `[2, 2, 2]`.**

6. **Trace the Trapping Rain Water stack algorithm on `[3, 0, 2, 0, 4]`. Show every pop and the water computed at each pop. What is the total?**

7. **Trace the circular NGE algorithm on `[1, 2, 1]`. Show the stack state after every iteration of the 2n loop. What is the output?**

8. **Why does the Stock Span algorithm accumulate spans instead of just counting pops? Trace it on `[3, 3, 3, 3]` and show why it produces `[1, 2, 3, 4]`.**

9. **You are given `[5, 4, 3, 2, 1]`. Trace the NGE algorithm fully. How many total push operations occur? How many total pop operations? What is in the result array?**

10. **Classify each of the following as NGE / NSE / PGE / PSE / contribution method / monotonic deque. Justify each:**
    - "For each bar, find the width of the largest rectangle where that bar is the height limiter."
    - "For each day, find the most recent previous day with a higher stock price."
    - "Find the maximum element in every sliding window of size k."
    - "Sum of the minimum element across all subarrays."

---

## Quick Reference Card

```
Monotonic Stack — enforced sorted invariant; each pop = one resolved answer.

Two stack types:
  Decreasing: pop when current > top  → for NGE, PGE
  Increasing: pop when current < top  → for NSE, PSE

Four variants (all O(n) time and space):
  NGE: L→R traverse. Pop when curr>top. Popped element's answer = curr value.
  NSE: L→R traverse. Pop when curr<top. Popped element's answer = curr value.
  PGE: L→R traverse. Pop when top≤curr. Stack top BEFORE push = answer for curr.
  PSE: L→R traverse. Pop when top≥curr. Stack top BEFORE push = answer for curr.

Universal template:
  for i in 0..n-1:
      while stk not empty and CONDITION(nums[stk.top()], nums[i]):
          idx = stk.pop()
          answer[idx] = nums[i]            // NGE or NSE
      if stk not empty:
          answer[i] = nums[stk.top()]      // PGE or PSE
      stk.push(i)                          // ← NEVER skip this

Always:
  → Push INDICES not values
  → check stk.empty() before stk.top() everywhere
  → stk.push(i) unconditionally after the while loop

Contribution method (subarray min/max sums):
  contribution[i] = arr[i] × (i - PSE[i]) × (NSE[i] - i)
  Use strict on one side, non-strict on other to avoid double-counting duplicates
  Always use (long long) and % MOD at each multiplication step

Amortized O(n) proof:
  Each element pushed once and popped at most once → total 2n operations
  The while loop cannot do more work than elements have been pushed

Problem keyword → variant:
  "next greater/warmer"     → NGE (decreasing, L→R, pop on >)
  "next smaller"            → NSE (increasing, L→R, pop on <)
  "previous greater"        → PGE (decreasing, read top before push)
  "previous smaller"        → PSE (increasing, read top before push)
  "trapped water / area"    → NSE+PSE for left/right boundaries
  "subarray min/max sum"    → contribution method
  "sliding window max/min"  → monotonic DEQUE (not stack)
```

---

*Previous: [07 — Stack](./07_stack.md)*
*Next: [09 — Queue](./09_queue.md)*
*See also: [Largest Rectangle in Histogram](./problems/histogram.md) | [Sliding Window Maximum — Deque](./10_deque.md)*

# Monotonic Deque

> **Curriculum position:** Linear Structures → #12
> **Interview weight:** ★★★★★ Critical — the only O(n) solution for sliding window extrema; appears in hard-tier problems at every top company
> **Difficulty:** Intermediate-Advanced — the concept is elegant once seen; the applications span a surprisingly wide problem family
> **Prerequisite:** [11 — Deque](./11_deque.md) | [08 — Monotonic Stack](./08_monotonic_stack.md)

---

## Table of Contents

1. [Intuition First — The Horizon Problem](#1-intuition-first)
2. [What Makes a Deque Monotonic](#2-what-makes-a-deque-monotonic)
3. [The Two Variants and Their Invariants](#3-the-two-variants-and-their-invariants)
4. [Why It Is O(n) — The Amortised Argument](#4-why-it-is-on)
5. [Time & Space Complexity](#5-time--space-complexity)
6. [Complete C++ Implementation](#6-complete-c-implementation)
7. [Core Operations — Visualised Step by Step](#7-core-operations--visualised)
8. [The Problem Family — Every Pattern Decoded](#8-the-problem-family)
9. [Interview Problems](#9-interview-problems)
10. [Real-World Uses](#10-real-world-uses)
11. [Edge Cases & Pitfalls](#11-edge-cases--pitfalls)
12. [Monotonic Deque vs Monotonic Stack vs Segment Tree vs Sparse Table](#12-comparison)
13. [Problem Recognition Guide](#13-problem-recognition-guide)
14. [Self-Test Questions](#14-self-test-questions)

---

## 1. Intuition First

Stand on a street and look east. The buildings you can see are those not blocked by a closer, taller building in front of them. As you walk east, old buildings disappear behind you and new ones appear ahead. At every step, the tallest visible building within your line of sight is the answer.

Now formalise: you have an array, a sliding window of size k, and a query: *what is the maximum value in the current window?*

The naïve approach re-scans k elements for every window position: O(nk). For n = 10⁵ and k = 10³, that is 10⁸ operations — too slow.

The key observation is that most of that work is redundant. When you slide the window one step to the right:
- One old element leaves the window from the left.
- One new element enters the window from the right.
- The maximum changes, but only because of these two events.

The monotonic deque exploits this structure. It maintains a deque of **candidate indices** — the elements that *could* be the maximum for some future window position. Its invariant: the deque is always sorted in **decreasing order of values** from front to back. The front is always the maximum of the current window.

When a new element arrives:
- Any candidate in the deque with a value **smaller than the new element** is eliminated — permanently. They can never be the maximum of any future window, because the new element is both larger *and* newer (it will outlast them in the window).
- The new element is added to the back.
- Any candidate at the front that has **exited the window** (its index is too old) is evicted.

Each element enters the deque exactly once and leaves exactly once. Total work across all n steps: O(n). The window size k does not appear in the complexity at all.

This is the monotonic deque: a deque maintained under a sorted invariant, where the front always answers the current window query in O(1), and every update is O(1) amortised.

---

## 2. What Makes a Deque Monotonic

A plain deque allows arbitrary push and pop at both ends. A **monotonic deque** adds one constraint: its contents are always in sorted order (either non-increasing or non-decreasing) from front to back.

This constraint is enforced by a specific action on every push:

```
Before pushing index i onto the back:
  Pop from the back any index j where nums[j] is "dominated" by nums[i].

  For a DECREASING deque (window maximum):
      Pop j from back while nums[j] <= nums[i]

  For an INCREASING deque (window minimum):
      Pop j from back while nums[j] >= nums[i]
```

After this pop-before-push, the new element is the smallest (for decreasing) or largest (for increasing) in the deque. The sorted order is maintained.

Additionally, on each step, the front is evicted if its index has fallen outside the current window:

```
If dq.front() < window_left:
    dq.pop_front()
```

These two rules — back-pop to maintain order, front-pop to maintain window — are the complete algorithm. Everything else is bookkeeping.

---

## 3. The Two Variants and Their Invariants

### Variant A: Decreasing Monotonic Deque (Sliding Window Maximum)

```
Invariant: nums[dq[0]] >= nums[dq[1]] >= ... >= nums[dq[back]]
           (values decrease from front to back — indices increase front to back)

Front = maximum of the current window
Back  = most recently added candidate

Push rule: pop from back while nums[back] <= nums[new]
Query:     nums[dq.front()]
Evict:     pop front if dq.front() < window_left
```

### Variant B: Increasing Monotonic Deque (Sliding Window Minimum)

```
Invariant: nums[dq[0]] <= nums[dq[1]] <= ... <= nums[dq[back]]
           (values increase from front to back — indices still increase front to back)

Front = minimum of the current window
Back  = most recently added candidate

Push rule: pop from back while nums[back] >= nums[new]
Query:     nums[dq.front()]
Evict:     pop front if dq.front() < window_left
```

### The Symmetric Relationship

```
Sliding window MAXIMUM → DECREASING deque → pop back when new element is LARGER
Sliding window MINIMUM → INCREASING deque → pop back when new element is SMALLER

Memory trick:
  MAX problem → maintain MAX at front → deque must be decreasing → pop when GREATER (>=)
  MIN problem → maintain MIN at front → deque must be increasing → pop when LESSER (<=)

Or more precisely:
  Pop from back when the back element would be DOMINATED by the new element.
  "Dominated" means: smaller value AND older index (for max); larger value AND older index (for min).
  Since we process left to right, older index is guaranteed — the only check needed is value.
```

---

## 4. Why It Is O(n) — The Amortised Argument

This is the most important analysis. Prove it rigorously, not just claim it.

### Claim: The total number of push + pop operations across all n steps is O(n).

**Proof:**
- Each of the n elements is pushed onto the deque **exactly once** — when we process index i, we push i.
- Each of the n elements is popped from the deque **at most once** — either via the back-pop (dominated by a later element) or the front-pop (evicted when it leaves the window).
- Total push operations: exactly n.
- Total pop operations: at most n (each element can be popped at most once — back-pop OR front-pop, never both at the back and both at the front).

Total work = push operations + pop operations ≤ n + n = 2n = **O(n)**.

This holds **regardless of k**. The window size affects how long an element stays in the deque before being front-evicted, but not the total number of pops across the entire array.

### Why the Inner while Loop Does Not Break the O(n) Claim

```cpp
for (int i = 0; i < n; i++) {
    while (!dq.empty() && nums[dq.back()] <= nums[i]) {  // inner while loop
        dq.pop_back();
    }
    dq.push_back(i);
    // ...
}
```

The inner `while` loop looks dangerous — it could run O(k) times in a single outer iteration. But across ALL outer iterations, the total number of back-pops is bounded by the total number of pushes, which is n. The inner loop is not O(k) per iteration on average — it is O(1) amortised.

**Concrete example showing why the worst case is O(n) total, not O(nk):**

```
nums = [5, 4, 3, 2, 1, 6], k = 6   (one big window, entire array)

i=0: push 0.                           dq=[0]       1 push, 0 pops
i=1: 4<5, push 1.                      dq=[0,1]     1 push, 0 pops
i=2: 3<4, push 2.                      dq=[0,1,2]   1 push, 0 pops
i=3: 2<3, push 3.                      dq=[0,1,2,3] 1 push, 0 pops
i=4: 1<2, push 4.                      dq=[0,1,2,3,4] 1 push, 0 pops
i=5: 6>1,pop4; 6>2,pop3; 6>3,pop2; 6>4,pop1; 6>5,pop0; push 5.
                                        dq=[5]       1 push, 5 pops

Total: 6 pushes, 5 pops = 11 operations for n=6. O(n). ✓
The 5 pops at i=5 "paid for" by the 5 prior pushes — each element popped at most once.
```

---

## 5. Time & Space Complexity

### Operation Complexities

| Operation | Time | Notes |
|---|---|---|
| Process single element (push + evict) | **O(1) amortized** | Each element pushed and popped at most once |
| Process all n elements | **O(n)** | 2n total push+pop operations |
| Query current window max/min | **O(1)** | Always at `dq.front()` |
| Evict front | **O(1)** | Check index + pop_front |
| Back pop (maintain invariant) | **O(1) amortized** | Amortised over all elements |
| Build for entire array | **O(n)** | One pass through the array |

### Space Complexity

| | |
|---|---|
| Deque storage | **O(k)** — at most k indices in the deque at any time |
| Why O(k) and not O(n) | Front-eviction removes elements older than the window |
| Worst case | k elements (strictly decreasing array — nothing is ever back-popped) |
| Best case | 1 element (strictly increasing array — every push pops all previous) |
| Result array | O(n - k + 1) — one max/min per window position |

### Comparison with Alternatives

| Structure | Build time | Query time | Space | Notes |
|---|---|---|---|---|
| Brute force scan | O(nk) | O(k) | O(1) | Too slow for large k |
| **Monotonic Deque** | **O(n)** | **O(1)** | **O(k)** | **Optimal for online/streaming** |
| Sparse Table | O(n log n) | O(1) | O(n log n) | Only for static arrays |
| Segment Tree | O(n) build | O(log n) | O(n) | Supports updates; overkill for fixed window |
| Heap (priority queue) | O(n log k) | O(log k) | O(k) | Handles non-consecutive windows |

The monotonic deque achieves **optimal time O(n) and optimal space O(k)** simultaneously, which no other structure does for the sliding window problem.

---

## 6. Complete C++ Implementation

### Core Monotonic Deque — Generic Template

```cpp
#include <deque>
#include <vector>
#include <functional>
#include <stdexcept>
using namespace std;

// Generic monotonic deque that maintains either max or min at the front.
// Stores indices into a reference array; values accessed via nums[idx].
template<typename T>
class MonotonicDeque {
private:
    deque<int>        dq_;          // stores indices
    const vector<T>&  nums_;        // reference to the underlying array
    function<bool(T,T)> dominated_; // returns true if 'a' is dominated by 'b'
                                    // i.e., 'a' can never be the answer while 'b' is present

public:
    // For MAX deque: dominated = [](T a, T b){ return a <= b; }
    // For MIN deque: dominated = [](T a, T b){ return a >= b; }
    MonotonicDeque(const vector<T>& nums, function<bool(T,T)> dominated)
        : nums_(nums), dominated_(dominated) {}

    // Add index i to the deque, maintaining the monotonic invariant
    // O(1) amortized
    void push(int i) {
        while (!dq_.empty() && dominated_(nums_[dq_.back()], nums_[i])) {
            dq_.pop_back();
        }
        dq_.push_back(i);
    }

    // Remove front if it has left the window [window_left, ...]
    // O(1)
    void evict(int window_left) {
        if (!dq_.empty() && dq_.front() < window_left) {
            dq_.pop_front();
        }
    }

    // Get the front value (max or min of current window)
    // O(1)
    T front_val() const {
        if (dq_.empty()) throw underflow_error("deque is empty");
        return nums_[dq_.front()];
    }

    int front_idx() const {
        if (dq_.empty()) throw underflow_error("deque is empty");
        return dq_.front();
    }

    bool empty() const { return dq_.empty(); }
    int  size()  const { return (int)dq_.size(); }
};
```

### Sliding Window Maximum — Clean Production Implementation

```cpp
// The canonical use case: sliding window maximum in O(n)
// Time: O(n) | Space: O(k)
vector<int> slidingWindowMax(const vector<int>& nums, int k) {
    int n = nums.size();
    if (n == 0 || k == 0) return {};

    vector<int> result;
    result.reserve(n - k + 1);
    deque<int> dq;   // decreasing monotonic deque — stores indices

    for (int i = 0; i < n; i++) {
        // --- Step 1: Evict expired front ---
        // Front index has slid outside the window [i-k+1, i]
        while (!dq.empty() && dq.front() < i - k + 1) {
            dq.pop_front();
        }

        // --- Step 2: Maintain decreasing invariant ---
        // Pop all back indices whose values are dominated by nums[i]
        // (they cannot be the max of any future window — nums[i] is larger AND newer)
        while (!dq.empty() && nums[dq.back()] <= nums[i]) {
            dq.pop_back();
        }

        dq.push_back(i);

        // --- Step 3: Record answer once the first full window is complete ---
        if (i >= k - 1) {
            result.push_back(nums[dq.front()]);
        }
    }
    return result;
}
```

### Sliding Window Minimum

```cpp
// Identical structure, opposite comparison direction
// Time: O(n) | Space: O(k)
vector<int> slidingWindowMin(const vector<int>& nums, int k) {
    int n = nums.size();
    vector<int> result;
    result.reserve(n - k + 1);
    deque<int> dq;   // INCREASING monotonic deque

    for (int i = 0; i < n; i++) {
        while (!dq.empty() && dq.front() < i - k + 1) {
            dq.pop_front();
        }
        // Only difference: pop when back value is >= nums[i] (not <=)
        while (!dq.empty() && nums[dq.back()] >= nums[i]) {
            dq.pop_back();
        }
        dq.push_back(i);
        if (i >= k - 1) {
            result.push_back(nums[dq.front()]);
        }
    }
    return result;
}
```

### Simultaneous Max and Min (Two Deques)

```cpp
// Useful for problems that need BOTH the window max and min simultaneously.
// Example: longest subarray where max - min <= limit.
// Time: O(n) | Space: O(n) worst case
pair<vector<int>, vector<int>> slidingWindowMaxMin(const vector<int>& nums, int k) {
    int n = nums.size();
    deque<int> maxDq, minDq;
    vector<int> maxVals, minVals;
    maxVals.reserve(n - k + 1);
    minVals.reserve(n - k + 1);

    for (int i = 0; i < n; i++) {
        // Evict both deques
        while (!maxDq.empty() && maxDq.front() < i - k + 1) maxDq.pop_front();
        while (!minDq.empty() && minDq.front() < i - k + 1) minDq.pop_front();

        // Maintain max deque (decreasing)
        while (!maxDq.empty() && nums[maxDq.back()] <= nums[i]) maxDq.pop_back();
        maxDq.push_back(i);

        // Maintain min deque (increasing)
        while (!minDq.empty() && nums[minDq.back()] >= nums[i]) minDq.pop_back();
        minDq.push_back(i);

        if (i >= k - 1) {
            maxVals.push_back(nums[maxDq.front()]);
            minVals.push_back(nums[minDq.front()]);
        }
    }
    return {maxVals, minVals};
}
```

### Variable-Window Monotonic Deque

```cpp
// For problems where the window size is not fixed but determined by a condition.
// The deque's front is evicted not by a fixed k but by a dynamic left pointer.
// Template: longest/shortest subarray satisfying some constraint on max or min.

int longestSubarrayMaxMinDiff(const vector<int>& nums, int limit) {
    deque<int> maxDq, minDq;
    int left = 0, result = 0;

    for (int right = 0; right < (int)nums.size(); right++) {
        // Maintain both deques
        while (!maxDq.empty() && nums[maxDq.back()] <= nums[right]) maxDq.pop_back();
        maxDq.push_back(right);
        while (!minDq.empty() && nums[minDq.back()] >= nums[right]) minDq.pop_back();
        minDq.push_back(right);

        // Shrink window from left until constraint is satisfied
        while (nums[maxDq.front()] - nums[minDq.front()] > limit) {
            left++;
            if (maxDq.front() < left) maxDq.pop_front();
            if (minDq.front() < left) minDq.pop_front();
        }

        result = max(result, right - left + 1);
    }
    return result;
}
```

---

## 7. Core Operations — Visualised Step by Step

### Full Trace: Sliding Window Maximum

```
nums = [2, 1, 5, 3, 6, 4, 8, 2],  k = 3
Expected output: [5, 5, 6, 6, 8, 8]

Deque stores INDICES. Values shown as nums[idx] for clarity.
Notation: dq = [idx(val), idx(val), ...]  front on left.

────────────────────────────────────────────────────────────
i=0, nums[0]=2
  Evict: front -1 < 0-3+1=-2? dq empty. Skip.
  Back-pop: dq empty. Skip.
  Push 0.
  dq = [0(2)]
  i < k-1=2: no output.
────────────────────────────────────────────────────────────
i=1, nums[1]=1
  Evict: front 0 < 1-3+1=-1? No.
  Back-pop: nums[0]=2 <= nums[1]=1? No. (2 > 1, back dominates new — keep back)
  Push 1.
  dq = [0(2), 1(1)]     ← decreasing: 2 ≥ 1 ✓
  i < k-1=2: no output.
────────────────────────────────────────────────────────────
i=2, nums[2]=5
  Evict: front 0 < 2-3+1=0? 0 < 0 is false. No eviction.
  Back-pop: nums[1]=1 <= nums[2]=5? YES → pop 1.
            nums[0]=2 <= nums[2]=5? YES → pop 0.
            dq empty.
  Push 2.
  dq = [2(5)]           ← only 5 remains; 1 and 2 were dominated
  i == k-1=2: OUTPUT nums[dq.front()] = nums[2] = 5
────────────────────────────────────────────────────────────
i=3, nums[3]=3
  Evict: front 2 < 3-3+1=1? No.
  Back-pop: nums[2]=5 <= nums[3]=3? No. (5 > 3, keep back)
  Push 3.
  dq = [2(5), 3(3)]     ← decreasing: 5 ≥ 3 ✓
  OUTPUT nums[dq.front()] = nums[2] = 5
────────────────────────────────────────────────────────────
i=4, nums[4]=6
  Evict: front 2 < 4-3+1=2? 2 < 2 is false. No eviction.
  Back-pop: nums[3]=3 <= nums[4]=6? YES → pop 3.
            nums[2]=5 <= nums[4]=6? YES → pop 2.
            dq empty.
  Push 4.
  dq = [4(6)]
  OUTPUT nums[dq.front()] = nums[4] = 6
────────────────────────────────────────────────────────────
i=5, nums[5]=4
  Evict: front 4 < 5-3+1=3? No.
  Back-pop: nums[4]=6 <= nums[5]=4? No. (6 > 4, keep back)
  Push 5.
  dq = [4(6), 5(4)]     ← decreasing: 6 ≥ 4 ✓
  OUTPUT nums[dq.front()] = nums[4] = 6
────────────────────────────────────────────────────────────
i=6, nums[6]=8
  Evict: front 4 < 6-3+1=4? 4 < 4 is false. No eviction.
  Back-pop: nums[5]=4 <= nums[6]=8? YES → pop 5.
            nums[4]=6 <= nums[6]=8? YES → pop 4.
            dq empty.
  Push 6.
  dq = [6(8)]
  OUTPUT nums[dq.front()] = nums[6] = 8
────────────────────────────────────────────────────────────
i=7, nums[7]=2
  Evict: front 6 < 7-3+1=5? No.
  Back-pop: nums[6]=8 <= nums[7]=2? No. (8 > 2, keep back)
  Push 7.
  dq = [6(8), 7(2)]     ← decreasing: 8 ≥ 2 ✓
  OUTPUT nums[dq.front()] = nums[6] = 8
────────────────────────────────────────────────────────────

Result: [5, 5, 6, 6, 8, 8] ✓

Push operations: 8 (one per element)
Pop  operations: 7 (total back-pops across all iterations)
Total work: 15 = O(n). k=3 never appears in the total count. ✓
```

### What the Deque Represents at Each Moment

```
After i=3, dq = [2(5), 3(3)]:

  This means:
  - The current window maximum is nums[2] = 5  (front)
  - If 5 exits the window in the future, the next candidate is nums[3] = 3  (back)
  - Element at index 1 (value=1) was already eliminated — it could never beat 5 OR 3
  - Element at index 0 (value=2) was already eliminated — it could never beat 5 OR 3

  The deque is a ranked list of "still-viable candidates" for future window queries.
  Front = current answer. Back = answer if everything before it exits the window.
```

---

## 8. The Problem Family — Every Pattern Decoded

The monotonic deque solves a wider family than just "sliding window max." Here is the complete taxonomy.

### Category 1: Fixed-Window Extremum Queries

Direct application. Window size k is fixed throughout.

```
Pattern: for each window [i-k+1, i], find max or min.
Template: standard sliding window max/min code above.
Examples: Sliding Window Maximum, Maximum of All Subarrays of Size K
```

### Category 2: Variable-Window Extremum Constraints

Window size is determined by a condition on the window's max or min. Use two pointers: `right` advances, `left` advances when the constraint is violated.

```
Pattern: find longest/shortest subarray where max(window) - min(window) <= limit
         OR where max(window) <= threshold
         OR similar constraint on the extremum.
Template: two deques (max + min) + two pointer shrinkage.

Key insight: when does the window need to shrink?
  → When the constraint on the extremum is violated.
  → The extremum is maintained by the deque front — O(1) check.
  → Shrink by advancing left; evict from deque fronts if they exit.
```

### Category 3: Prefix-Sum + Monotonic Deque

For range-sum problems with negative numbers where the window cannot be maintained by simple expansion.

```
Pattern: find minimum/maximum subarray sum, or shortest subarray with sum >= k.
Approach: build prefix sum array P.
          sum(l,r) = P[r+1] - P[l]
          Maintain monotonic deque over P indices.
          For "sum >= k": want minimum P[l] (use increasing deque over P).
          For right = i: pop front while P[i+1] - P[front] >= k (record length).

This is how "Shortest Subarray with Sum >= K" achieves O(n).
```

### Category 4: DP Optimisation with Sliding Window

This is the deepest application. Many DP recurrences have the form:

```
dp[i] = max(dp[j]) + cost(i)    for j in [i-k, i-1]

Without optimisation: O(n²) — for each i, scan k previous dp values.
With monotonic deque:  O(n)  — maintain deque of dp indices [i-k, i-1] in decreasing dp-value order.
                                dp[i] = dp[dq.front()] + cost(i) in O(1).

Template:
  for i in 0..n:
      // Evict dp indices outside [i-k, i-1]
      while dq.front() < i - k: dq.pop_front()
      // dp[i] uses the front (maximum dp value in window)
      dp[i] = dp[dq.front()] + cost(i)
      // Maintain decreasing dp-value deque
      while !dq.empty() and dp[dq.back()] <= dp[i]: dq.pop_back()
      dq.push_back(i)
```

Examples: Jump Game VI, Maximum Sum of Two Non-Overlapping Subarrays, Constrained Subsequence Sum.

### Category 5: Monotonic Deque in 2D (Matrix Problems)

Apply the 1D sliding window max/min row-wise, then column-wise (or vice versa).

```
For each row: compute sliding window max with window width k_cols → intermediate matrix.
For each column of intermediate: compute sliding window max with window height k_rows.

Result: max of every k_rows × k_cols submatrix.
Time: O(n × m) — two passes with monotonic deque.
```

---

## 9. Interview Problems

### Problem 1: Jump Game VI (DP + Monotonic Deque)

**Problem:** You are given a 0-indexed integer array `nums` and an integer `k`. You start at index 0 and at each step you can jump to any index in the range `[i+1, i+k]`. Your score is the sum of values at each index you visit. Return the maximum score to reach index n-1.

**Example:** `nums = [1,-1,-2,4,-7,3]`, `k = 2` → `7`  
(Path: 0 → 1 → 3 → 5, scores: 1 + (-1) + 4 + 3 = 7)

**Thought process:**

> "Define `dp[i]` = maximum score to reach index i. At each index i, we can have jumped from any index j in [i-k, i-1]. So `dp[i] = nums[i] + max(dp[j])` for j in [i-k, i-1]. This is a sliding window maximum on the dp array! Naïve: O(nk). With monotonic deque: O(n)."

```cpp
// Time: O(n) | Space: O(k) deque + O(n) dp array
int maxResult(vector<int>& nums, int k) {
    int n = nums.size();
    vector<int> dp(n);
    dp[0] = nums[0];

    deque<int> dq;    // decreasing monotonic deque over dp values
    dq.push_back(0);  // seed with index 0

    for (int i = 1; i < n; i++) {
        // Evict: index i-k-1 has left the window [i-k, i-1]
        while (!dq.empty() && dq.front() < i - k) {
            dq.pop_front();
        }

        // dp[i] uses the maximum dp value in [i-k, i-1]
        dp[i] = nums[i] + dp[dq.front()];

        // Maintain decreasing deque — add dp[i] to the back
        while (!dq.empty() && dp[dq.back()] <= dp[i]) {
            dq.pop_back();
        }
        dq.push_back(i);
    }

    return dp[n - 1];
}

/*
Trace: nums=[1,-1,-2,4,-7,3], k=2

dp[0]=1, dq=[0(dp=1)]

i=1: evict? front=0 < 1-2=-1? No.
     dp[1] = nums[1] + dp[dq.front()=0] = -1 + 1 = 0
     maintain: dp[0]=1 <= dp[1]=0? No. push 1.
     dq=[0(1), 1(0)]

i=2: evict? front=0 < 2-2=0? 0<0 false. No.
     dp[2] = nums[2] + dp[dq.front()=0] = -2 + 1 = -1
     maintain: dp[1]=0 <= dp[2]=-1? No. push 2.
     dq=[0(1), 1(0), 2(-1)]

i=3: evict? front=0 < 3-2=1? 0<1 YES → pop 0.
     front=1 < 1? No.
     dp[3] = nums[3] + dp[dq.front()=1] = 4 + 0 = 4
     maintain: dp[2]=-1 <= dp[3]=4? YES → pop 2.
               dp[1]=0  <= dp[3]=4? YES → pop 1.
               dq empty.
     push 3. dq=[3(4)]

i=4: evict? front=3 < 4-2=2? No.
     dp[4] = nums[4] + dp[3] = -7 + 4 = -3
     maintain: dp[3]=4 <= dp[4]=-3? No. push 4.
     dq=[3(4), 4(-3)]

i=5: evict? front=3 < 5-2=3? 3<3 false. No.
     dp[5] = nums[5] + dp[3] = 3 + 4 = 7
     maintain: dp[4]=-3 <= dp[5]=7? YES → pop 4.
               dp[3]=4  <= dp[5]=7? YES → pop 3.
     push 5. dq=[5(7)]

return dp[5] = 7 ✓
*/
```

**Edge cases:**
- k >= n: can jump to any index from index 0 in one step — dp[i] = nums[i] + dp[0] for all i, or more generally always take the global max dp from the window
- All negative: must still reach n-1, so the answer is the path with minimum total negative sum
- n=1: return nums[0], no jumps needed

**Why monotonic deque and not just max DP with re-scan?** For n=10⁵ and k=10⁴, naïve O(nk) is 10⁹ — too slow. The deque makes it O(n) = 10⁵ — fast enough.

---

### Problem 2: Longest Continuous Subarray with Absolute Diff ≤ Limit

**Problem:** Given an array `nums` and an integer `limit`, return the size of the longest non-empty subarray such that the absolute difference between any two elements is ≤ limit.

**Example:** `nums = [8, 2, 4, 7]`, `limit = 4` → `2`  
Subarrays: `[8,2]` has diff 6 > 4; `[2,4]` has diff 2 ≤ 4; `[4,7]` has diff 3 ≤ 4; longest is 2.

**Thought process:**

> "The condition `abs(any two elements) <= limit` is equivalent to `max(window) - min(window) <= limit`. So I need a window where the max minus the min is at most limit."
>
> "I'll use two monotonic deques simultaneously: one for the window max (decreasing) and one for the window min (increasing). The window is valid when `maxDq.front() - minDq.front() <= limit`. When it becomes invalid, shrink from the left. Use two pointers."

```cpp
// Time: O(n) | Space: O(n) worst case deques
int longestSubarray(vector<int>& nums, int limit) {
    deque<int> maxDq;   // decreasing — front = window max
    deque<int> minDq;   // increasing — front = window min
    int left = 0, result = 0;

    for (int right = 0; right < (int)nums.size(); right++) {
        // Maintain max deque
        while (!maxDq.empty() && nums[maxDq.back()] <= nums[right]) {
            maxDq.pop_back();
        }
        maxDq.push_back(right);

        // Maintain min deque
        while (!minDq.empty() && nums[minDq.back()] >= nums[right]) {
            minDq.pop_back();
        }
        minDq.push_back(right);

        // Shrink window from left until constraint is satisfied
        while (nums[maxDq.front()] - nums[minDq.front()] > limit) {
            left++;
            if (maxDq.front() < left) maxDq.pop_front();
            if (minDq.front() < left) minDq.pop_front();
        }

        result = max(result, right - left + 1);
    }
    return result;
}

/*
Trace: nums=[8,2,4,7], limit=4

right=0: maxDq=[0(8)], minDq=[0(8)]. max-min=0<=4. left=0. result=1.
right=1: maxDq: 8>2 → keep. push 1. maxDq=[0(8),1(2)].
         minDq: 8>2 → pop 0. push 1. minDq=[1(2)].
         max=8, min=2. 8-2=6>4 → shrink: left=1.
         maxDq.front()=0<1 → pop 0. maxDq=[1(2)].
         minDq.front()=1>=1 → keep.
         max=2, min=2. 2-2=0<=4. result=max(1,1-1+1)=1.

right=2: maxDq: 2<=4 → pop 1. push 2. maxDq=[2(4)].
         minDq: 2<=4 → keep. push 2. minDq=[1(2),2(4)].
         max=4, min=2. 4-2=2<=4. result=max(1,2-1+1)=2.

right=3: maxDq: 4<=7 → pop 2. push 3. maxDq=[3(7)].
         minDq: 4<=7 → keep. push 3. minDq=[1(2),2(4),3(7)].
         max=7, min=2. 7-2=5>4 → shrink: left=2.
         maxDq.front()=3>=2 → keep.
         minDq.front()=1<2 → pop 1. minDq=[2(4),3(7)].
         max=7, min=4. 7-4=3<=4. result=max(2,3-2+1)=2.

return 2. ✓

Valid subarrays of length 2: [2,4] (diff=2), [4,7] (diff=3). Both ≤ 4. ✓
*/
```

**The critical insight — why two deques?** You need both the window max and the window min simultaneously in O(1). No single structure gives you both. Two monotonic deques — one decreasing for max, one increasing for min — together give both in O(1) and O(n) total time.

**Edge cases:**
- `limit = 0`: only subarrays where all elements are equal; every element is its own valid subarray of length 1, and longer runs of equal elements extend it
- All elements equal: entire array is valid, return n
- `limit` is very large: entire array may be valid — return n
- Single element: return 1 always

---

### Problem 3: Maximum Sum of Subarray of Size K with at Most One Deletion

**Problem:** Given an array of integers and an integer k, find the maximum sum of a subarray of **exactly k** elements after deleting **at most one** element from the array (you may delete any element, reducing the subarray to k-1 original elements plus the deletion slot, or keep all k).

**A more precise version — Constrained Subsequence Sum (LeetCode 1425):**

Given `nums` (may be negative) and `k`, find the maximum sum of a non-empty subsequence of `nums` such that for every pair of consecutive elements in the subsequence, their indices satisfy `i - j <= k`.

**Thought process:**

> "Define `dp[i]` = maximum sum of a valid subsequence ending at index i. Base case: `dp[i] = nums[i]` (take only element i). Transition: optionally extend from some j in `[i-k, i-1]` with `dp[j] > 0`: `dp[i] = nums[i] + max(0, max(dp[j]))` for j in `[i-k, i-1]`."
>
> "The `max(dp[j])` over a sliding window of size k is exactly the monotonic deque problem. The `max(0, ...)` part handles the case where we start fresh at index i (not extending any previous subsequence)."

```cpp
// Time: O(n) | Space: O(k)
int constrainedSubsetSum(vector<int>& nums, int k) {
    int n = nums.size();
    vector<int> dp(n);
    deque<int> dq;   // decreasing deque over dp values
    int result = INT_MIN;

    for (int i = 0; i < n; i++) {
        // Evict front outside window [i-k, i-1]
        while (!dq.empty() && dq.front() < i - k) {
            dq.pop_front();
        }

        // dp[i] = nums[i] + max(0, best dp in window)
        // max(0, ...) = if best previous is negative, start fresh at i
        dp[i] = nums[i] + ((!dq.empty() && dp[dq.front()] > 0) ? dp[dq.front()] : 0);

        result = max(result, dp[i]);

        // Maintain decreasing deque over dp values
        while (!dq.empty() && dp[dq.back()] <= dp[i]) {
            dq.pop_back();
        }
        dq.push_back(i);
    }

    return result;
}

/*
Trace: nums=[10,2,-10,5,20], k=2

i=0: dq=[]. dp[0]=10+0=10. result=10. dq=[0(10)].
i=1: evict? 0 < 1-2=-1? No.
     dp[1]=2+dp[0]=2+10=12. result=12.
     12>10 → pop 0. dq=[1(12)].
i=2: evict? 1 < 0? No.
     dp[2]=-10+dp[1]=-10+12=2. result=12.
     2<12 → push 2. dq=[1(12),2(2)].
i=3: evict? 1 < 3-2=1? 1<1 false. No.
     dp[3]=5+dp[1]=5+12=17. result=17.
     17>2 → pop 2. 17>12 → pop 1. dq=[3(17)].
i=4: evict? 3 < 4-2=2? No.
     dp[4]=20+dp[3]=20+17=37. result=37.
     37>17 → pop 3. dq=[4(37)].

return 37. ✓
Subsequence: take index 0 (10), skip 2, take index 1 (2)? Wait —
Actually: dp[3]=17 came from dp[1]=12+nums[3]=5.
  dp[1]=12 came from dp[0]=10+nums[1]=2.
  So path: 0→1→3: 10+2+5=17. ✓
Then dp[4]=37: path 0→1→3→4: 10+2+5+20=37. ✓
*/
```

**Edge cases:**
- All negative: dp[i] = nums[i] for all i (never extend, always start fresh); result = max(nums)
- k=1: can only extend from the immediately preceding element
- k >= n: any valid subsequence is allowed

---

## 10. Real-World Uses

| Domain | Use Case | Connection to Monotonic Deque |
|---|---|---|
| **Financial systems** | Rolling max/min price over last n ticks | Direct sliding window max/min application |
| **Game AI** | Sliding window difficulty rating, enemy strength peaks | Track max threat in a time window |
| **Stream processing** | Apache Flink, Kafka Streams window aggregations | `TUMBLING WINDOW MAX(val)` compiled to monotonic deque |
| **Video encoding** | Scene change detection via rolling variance | Rolling max/min pixel intensity |
| **Network monitoring** | Peak bandwidth in last k seconds | Sliding window max on bandwidth measurements |
| **Sensor fusion** | Robust IMU data: reject outliers using rolling range | Window max - min > threshold → anomaly |
| **Database query optimisation** | `MAX() OVER (PARTITION BY ... ORDER BY ... ROWS BETWEEN k PRECEDING AND CURRENT ROW)` | Window functions in SQL engines use monotonic deque |
| **DP optimisation** | Compiler IR optimisation, sequence alignment | Recurrences of the form `dp[i] = max(dp[j]) + cost` |
| **Robotics** | Sliding window obstacle detection | Max height in a forward-looking LiDAR window |
| **OS scheduling** | Maximum request rate in a time slice | Sliding window max on request counters |

**SQL Window Functions — The Monotonic Deque in Databases:**

```sql
-- This SQL query:
SELECT
    timestamp,
    value,
    MAX(value) OVER (
        ORDER BY timestamp
        ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
    ) AS rolling_max_3
FROM sensor_data;

-- Is executed internally by PostgreSQL, DuckDB, and Spark using an algorithm
-- equivalent to the monotonic deque. The query planner recognises this as
-- a "sliding window aggregate" and applies the O(n) algorithm automatically.

-- Without the optimisation: O(nk) — scan k rows for every output row.
-- With the optimisation (monotonic deque): O(n) — each row pushed and popped once.

-- DuckDB's source code (src/execution/window_executor.cpp) explicitly implements
-- this as a monotone queue for min/max window aggregates.
-- PostgreSQL calls it a "running window aggregate" and uses a similar structure.
```

**Apache Flink — Stream Processing at Scale:**

```java
// Flink DataStream API: sliding window max over a 1-minute window
DataStream<Integer> maxPerMinute = stream
    .keyBy(event -> event.userId)
    .window(SlidingEventTimeWindows.of(Time.minutes(1), Time.seconds(1)))
    .aggregate(new MaxAggregateFunction());

// Internally, Flink's HeapWindowBuffer uses a variation of the monotonic
// deque for min/max aggregates over sliding windows. This is what allows
// Flink to process millions of events per second per core while maintaining
// rolling max/min statistics in real time.
```

---

## 11. Edge Cases & Pitfalls

### Pitfall 1: Pushing Values Instead of Indices

This is the #1 bug in monotonic deque implementations.

```cpp
// WRONG — loses positional information, breaks eviction
deque<int> dq;
while (!dq.empty() && dq.back() <= nums[i]) dq.pop_back();
dq.push_back(nums[i]);   // stored VALUE
// Eviction attempt (broken):
if (dq.front() == nums[i - k]) dq.pop_front();
// Bug: if nums[0] == nums[5] == 7 and k=3, at i=5:
//   nums[i-k] = nums[2]. If nums[2] != 7, we don't evict.
//   But if nums[2] == 7 too, we evict even though nums[0] (the actual old element) is still in deque!
// With duplicates, value-based eviction is fundamentally broken.

// CORRECT — always push indices
while (!dq.empty() && nums[dq.back()] <= nums[i]) dq.pop_back();
dq.push_back(i);            // stored INDEX
// Eviction (always correct):
if (dq.front() < i - k + 1) dq.pop_front();
// Index comparison is unambiguous — no issue with duplicate values.
```

### Pitfall 2: Wrong Eviction Bound

```cpp
// Eviction should remove indices that have LEFT the window [i-k+1, i].
// The window left boundary is i-k+1, so evict if front < i-k+1.

// WRONG: off-by-one
if (dq.front() <= i - k) dq.pop_front();   // equivalent to front < i-k+1 — actually correct!
if (dq.front() < i - k)  dq.pop_front();   // WRONG: keeps one extra old element

// The two correct forms:
if (dq.front() < i - k + 1) dq.pop_front();   // form 1: explicit left bound
if (dq.front() <= i - k)    dq.pop_front();    // form 2: equivalent

// Memory aid: window = [i-k+1, i]. Front is expired if front < window_left = i-k+1.
```

### Pitfall 3: Recording Result Before First Full Window

```cpp
// WRONG: recording result before the first k elements are processed
for (int i = 0; i < n; i++) {
    // ... maintain deque ...
    result.push_back(nums[dq.front()]);  // records before window is full!
}
// For k=3 and i=0: the "window" [0-3+1, 0] = [-2, 0] has only one real element.
// Front is valid, but the window isn't full yet — semantics depend on the problem.

// CORRECT: only record once i >= k-1 (first full window complete)
if (i >= k - 1) {
    result.push_back(nums[dq.front()]);
}
```

### Pitfall 4: Wrong Comparison Direction (Max vs Min Confusion)

```cpp
// For MAXIMUM: pop back while back value is SMALLER THAN OR EQUAL TO new value
// The deque should be DECREASING (large values at front).
while (!dq.empty() && nums[dq.back()] <= nums[i]) dq.pop_back();  // MAX deque ✓

// For MINIMUM: pop back while back value is LARGER THAN OR EQUAL TO new value
// The deque should be INCREASING (small values at front).
while (!dq.empty() && nums[dq.back()] >= nums[i]) dq.pop_back();  // MIN deque ✓

// WRONG (swapped): using >= for max deque gives the MIN at front — wrong answer
while (!dq.empty() && nums[dq.back()] >= nums[i]) dq.pop_back();  // would give MIN ✗

// Memory trick: MAX deque is DECREASING → pop when back is SMALLER (<=).
//               MIN deque is INCREASING → pop when back is LARGER  (>=).
```

### Pitfall 5: Evicting Both Fronts in Two-Deque Problems

```cpp
// When maintaining two deques (max + min) with a variable left pointer:
while (nums[maxDq.front()] - nums[minDq.front()] > limit) {
    left++;
    // WRONG: only evict one deque
    if (maxDq.front() < left) maxDq.pop_front();
    // Forgot to also evict minDq! The min deque front may now be outside window.

    // CORRECT: check and evict both deques
    if (maxDq.front() < left) maxDq.pop_front();
    if (minDq.front() < left) minDq.pop_front();
}
```

### Pitfall 6: DP Deque — Pushing Before Computing dp[i]

```cpp
// In DP + deque problems, the deque holds dp values for PREVIOUS indices [i-k, i-1].
// You must compute dp[i] FIRST (using dq.front()), THEN push i to the deque.

// WRONG: push i before computing dp[i]
dq.push_back(i);                                     // i is now in the window
dp[i] = nums[i] + dp[dq.front()];                   // dq.front() might be i itself!
// If dq.front() == i, dp[i] = nums[i] + dp[i] — circular dependency!

// CORRECT: compute dp[i] first, then push
dp[i] = nums[i] + (dq.empty() ? 0 : dp[dq.front()]); // use previous dp values
// NOW push i (so it is available for future dp[i+1], ..., dp[i+k])
while (!dq.empty() && dp[dq.back()] <= dp[i]) dq.pop_back();
dq.push_back(i);
```

### Pitfall 7: Not Handling the Empty Deque Before Querying Front

```cpp
// If the deque is unexpectedly empty when you call front(), it is UB (std::deque)
// or an exception (your custom deque). This can happen when:
// - k > n (window larger than array)
// - All elements have been evicted before the first valid window

// Always guard:
if (!dq.empty()) {
    int maxVal = nums[dq.front()];
    result.push_back(maxVal);
} else {
    // Handle appropriately — usually this indicates a logic error
}
```

---

## 12. Comparison

| Structure | Window Max/Min | Build Time | Query Time | Space | Supports Updates | Window Type |
|---|---|---|---|---|---|---|
| **Monotonic Deque** | Both | **O(n)** | **O(1)** | **O(k)** | Only append/evict | Fixed or variable |
| Brute Force | Both | — | O(k) per query | O(1) | Any | Any |
| Segment Tree | Both | O(n) | O(log n) | O(n) | Any (point update) | Any range |
| Sparse Table | Both | O(n log n) | O(1) | O(n log n) | No | Any static range |
| Monotonic Stack | Both (limited) | O(n) | O(1) per element | O(n) | Append only | Half-open (no eviction) |
| Priority Queue (heap) | Both | O(n log k) | O(log k) | O(k) | With lazy deletion | Non-contiguous ok |
| Deque (unsorted) | Neither | — | O(k) scan | O(k) | Any | Any |

**When to choose each:**

```
Fixed sliding window, stream of data:
  → Monotonic Deque ← ALWAYS the right answer here

Static array, arbitrary range queries (not just sliding):
  → Sparse Table (O(1) query, O(n log n) build, idempotent functions only)
  → Segment Tree (O(log n) query, handles non-idempotent functions)

Arbitrary range queries with point updates:
  → Segment Tree

Non-contiguous window (e.g., specific set of indices):
  → Priority Queue with lazy deletion

Need both max and min simultaneously:
  → Two Monotonic Deques
```

---

## 13. Problem Recognition Guide

When you see a problem in an interview, these signals should trigger "monotonic deque":

### Signal 1: "Sliding window" + "maximum" or "minimum"

```
"Find the maximum of each window of size k"
"Find the minimum in a rolling window"
"Maximum of all subarrays of size k"
→ Monotonic deque. Direct application.
```

### Signal 2: "Longest/shortest subarray" + "constraint on max or min"

```
"Longest subarray where max - min <= limit"
"Longest subarray where max <= k * min"
"Shortest subarray with max value < threshold"
→ Two monotonic deques + two pointers.
```

### Signal 3: "DP with jump constraint" + O(n) expected

```
"dp[i] = max(dp[j]) + something, for j in [i-k, i-1]"
"Jump to any index up to k steps forward, maximise score"
"Select elements with index gap constraint, maximise sum"
→ Monotonic deque over dp array. dp recurrence optimisation.
```

### Signal 4: "Prefix sum" + "minimum sum subarray" + negatives

```
"Shortest subarray with sum >= k" (with negative numbers)
"Minimum subarray sum" (variable window)
→ Prefix sum array + monotonic deque over prefix sums.
```

### Signal 5: "SQL window function MAX/MIN" or "rolling statistics"

```
"ROWS BETWEEN k PRECEDING AND CURRENT ROW with MAX or MIN"
"Rolling maximum over last n data points"
→ Monotonic deque. O(n) regardless of window size.
```

### The Decision Algorithm

```
Does the problem involve a range query on a subarray?
  YES:
    Is it a sliding window of FIXED size k?
      YES: Is the query MAX or MIN?
             YES → Monotonic deque. O(n).
             NO  → Depends (sum → prefix; count → sliding window counter)
      NO (variable window determined by condition on max/min):
             → Two monotonic deques + two pointers. O(n).
    Is it a DP where dp[i] depends on max/min of dp[j] in a window?
             → Monotonic deque over dp array. O(n).
    Is it an arbitrary range query (not sliding)?
             → Sparse Table or Segment Tree.
  NO: Not a monotonic deque problem.
```

---

## 14. Self-Test Questions

1. **State the invariant of a decreasing monotonic deque in one sentence. State it for an increasing monotonic deque. Which one gives you the sliding window maximum and which gives the minimum?**

2. **Prove that the total number of push and pop operations across all n elements is O(n). Your proof must bound both back-pops and front-evictions separately.**

3. **Why do we store indices in the deque rather than values? Construct a concrete example with duplicate values where storing values produces a wrong answer.**

4. **Trace the decreasing monotonic deque on `nums = [3, 1, 3, 2, 4, 1]` with `k = 3`. Show the full deque state (indices and values) after each element. What is the output?**

5. **In the Jump Game VI solution, why is `dp[i] = nums[i] + dp[dq.front()]` and not `dp[i] = max(dp[j] for j in [i-k,i-1])`? They seem equivalent — are they?**

6. **Why does the Longest Subarray with Absolute Diff ≤ Limit solution need TWO deques? Could you solve it with one?**

7. **Write the variable-window monotonic deque template for finding the longest subarray where `max(window) < 2 * min(window)`. What is the eviction condition?**

8. **Compare monotonic deque vs sparse table for sliding window max. When is each preferred? What does sparse table offer that monotonic deque cannot?**

9. **A system needs rolling 5-minute maximum CPU usage, updated every second. Which data structure would you use? What are the time and space requirements? How does the answer change if you need both max AND min simultaneously?**

10. **In a DP optimisation with the recurrence `dp[i] = max(dp[j] for j in [i-k, i-1]) + cost(i)`, you forget to evict expired indices from the deque front. Give a concrete example where this produces a wrong answer.**

---

## Quick Reference Card

```
Monotonic Deque — O(1) sliding window max/min. O(n) total for n elements.

Two variants:
  DECREASING deque → front = window MAXIMUM
    Push rule: pop back while nums[back] <= nums[new]
  INCREASING deque → front = window MINIMUM
    Push rule: pop back while nums[back] >= nums[new]

Always store INDICES (not values).
Always evict front before querying: pop front if front < i - k + 1

Canonical template (sliding window max):
  deque<int> dq;
  for (int i = 0; i < n; i++) {
      while (!dq.empty() && dq.front() < i - k + 1) dq.pop_front(); // evict
      while (!dq.empty() && nums[dq.back()] <= nums[i]) dq.pop_back(); // maintain
      dq.push_back(i);
      if (i >= k - 1) result.push_back(nums[dq.front()]); // record
  }

Complexity:
  Time:  O(n) — each element pushed once, popped at most once
  Space: O(k) — at most k indices in deque at any time

Problem family (recognise these triggers):
  "sliding window max/min of fixed size k"    → single deque, O(n)
  "longest subarray, constraint on max-min"   → two deques + two pointers, O(n)
  "DP: dp[i] = max(dp[j]) + cost, j in window" → deque over dp array, O(n)
  "shortest subarray sum >= k with negatives"  → prefix sum + increasing deque, O(n)

Pitfall checklist:
  ✗ Storing values (not indices) — breaks with duplicates
  ✗ Using wrong comparison (>= for max deque, <= for min deque)
  ✗ Evicting with wrong bound (< i-k vs < i-k+1)
  ✗ Recording result before first full window (need i >= k-1)
  ✗ In DP: pushing i before computing dp[i] (circular dependency)
  ✗ Forgetting to evict both deques in two-deque variable-window problems
  ✗ Not checking empty() before querying front()

vs alternatives:
  Sparse Table:    O(1) query but O(n log n) build, no updates, static only
  Segment Tree:    O(log n) query, handles updates, arbitrary ranges
  Priority Queue:  O(log k) per step, useful for non-contiguous windows
  Monotonic Deque: O(1) amortized, O(k) space, optimal for sliding windows
```

---

*Previous: [11 — Deque](./11_deque.md)*
*Next: [13 — Priority Queue](./13_priority_queue.md)*
*See also: [08 — Monotonic Stack](./08_monotonic_stack.md) | [Sliding Window algorithms](./algorithms/sliding_window.md) | [DP Optimisation techniques](./algorithms/dp_optimisation.md)*

# EOO Function Discussion

## Background: Issues with the Current Formula

### Current EOO Formula

```
EOO = On Hand
    + Actual Expected [Backlog + TLT/52wks]
    + Open Order [Backlog + TLT/52wks]
    - Demand [Backlog + TLT/52wks]
```

This formula uses demand as the baseline. Within a fixed time window (`TLT/52wks`), any inventory above the demand within that window is classified as excess.

---

## Core Problems

### Problem 1: Safety Stock Is Not Accounted For

The current baseline is demand only, with no Safety Stock included. This implicitly allows inventory to drop to zero, when in reality inventory should always remain above Safety Stock at any given point in time.

### Problem 2: The Time Window Does Not Align with the True Inventory Cycle

Within a replenishment cycle, inventory levels are well above zero — and well above Safety Stock — for most of the time, because we need to cover demand for the next cycle.

In the ideal scenario:
- Inventory only briefly touches Safety Stock right before a replenishment order arrives, then immediately jumps to `Safety Stock + Replenishment Order`
- This assumes the time window (`TLT/52wks`) perfectly aligns with the true replenishment cycle, or is an exact multiple of it

In practice:
- **The inventory cycle is almost always longer than TLT**
- The current time window is neither equal to the true cycle nor an integer multiple of it

### Problem 3: The Calculation Start Date Is Not the Beginning of a True Inventory Cycle

The EOO calculation starts from "today," which does not coincide with the beginning of any replenishment cycle — and the end of the window doesn't coincide with the end of a cycle either. This means the formula captures an incomplete inventory consumption interval, leading to two types of misclassification:

1. **Non-excess inventory flagged as EOO** → Canceling or reducing orders based on this result actually creates a shortage
2. **Shortage misidentified as EOO** → A part that is already short appears to have excess inventory

---

## Improvement Directions

### Approximate Fix (Scenario Analysis)

Two ways to tighten the EOO threshold:
1. **Include Safety Stock** in the baseline, raising the bar for what counts as excess
2. **Extend the time window** to better approximate a full replenishment cycle

Five scenarios have been calculated using this week's data:

| Scenario | Time Window |
|---|---|
| EOO within 3 months | 3 months |
| EOO within 6 months | 6 months |
| EOO within 9 months | 9 months |
| EOO within 12 months | 12 months |
| Firm Zone | Exact data pending |

> Note: These adjustments are approximations and do not fully resolve the root cause of the cycle misalignment issue.

---

## New Approach: Optimizer-Based EOO Calculation

### Problem Statement

> **Given that inventory must not fall below Safety Stock in any week, what is the minimum Open Order quantity (K) needed?**

The Excess Open Order is then: **EOO = H − K**, where H is the current Open Order quantity.

### Calculation Logic

1. **Compute weekly inventory levels assuming no Open Orders**
   Use only On Hand and Actual Expected (committed, non-cancellable inventory) to project inventory week by week.

2. **Find the inventory trough**
   Identify the week with the lowest projected inventory level across the planning horizon.

3. **Determine whether additional Open Order is needed**
   - If the trough ≥ Safety Stock → no Open Order needed; EOO = entire current Open Order
   - If the trough < Safety Stock → add just enough Open Order to bring that week up to Safety Stock

4. **Leverage the trough's propagation effect**
   The trough is the lowest point in the entire horizon. If it meets the Safety Stock constraint, all subsequent weeks — which carry higher on-hand levels — automatically satisfy the constraint as well. A single top-up at the trough is sufficient to secure the entire horizon.

5. **Derive the minimum Open Order quantity (K)**
   The top-up quantity is the minimum Open Order required to maintain inventory safety. The difference between the current Open Order (H) and this minimum (K) is the EOO.

### Key Advantages Over the Current Approach

- Answers "what is the minimum inventory needed" directly, rather than inferring excess from a demand baseline
- Fully incorporates the Safety Stock constraint
- Eliminates reliance on an arbitrary time window, avoiding cycle misalignment errors
- Highly interpretable: each part's EOO maps to a concrete "top-up at the trough" operation

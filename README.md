# EOO Function Discussion

## 背景：当前公式的问题

### 现行 EOO 公式

```
EOO = On Hand
    + Actual Expected [Backlog + TLT/52wks]
    + Open Order [Backlog + TLT/52wks]
    - Demand [Backlog + TLT/52wks]
```

该公式以 demand 为 baseline，在一个特定 time window（`TLT/52wks`）内，任何高于该窗口 demand 的 inventory 均被定义为 excess。

---

## 核心问题

### 问题 1：未计入 Safety Stock

当前公式的 baseline 是 demand，没有将 Safety Stock 纳入计算。这意味着公式允许 inventory 降至 0，而实际上任何时候 inventory 都应高于 Safety Stock。

### 问题 2：Time Window 与真实 Inventory 周期不匹配

在一个 inventory 消耗周期中，大部分时间点的 inventory level 都远高于 0，也远高于 Safety Stock——因为我们需要为下一个周期的 demand 做准备。

在最优情况下：
- 只有在收到 replenishment order 时，inventory 才会瞬间触及 Safety Stock，随即上升至 `Safety Stock + Replenishment Order`
- 该情况要求 time window（`TLT/52wks`）与真实 inventory 周期完全吻合，或是其整数倍

但实际情况是：
- **Inventory 周期几乎总是大于 TLT**
- 当前 time window 既不等于真实周期，也不是其整数倍

### 问题 3：计算起点不是真实 Inventory 周期的开始

EOO 计算的开始时间（"今天"）并不落在某个 inventory 周期的起点，结束时间也不落在周期终点。这导致公式截取了一个不完整的库存消耗区间，产生两类误判：

1. **把非 EOO 的 quantity 识别为 EOO** → 按此取消/减少订单后实际形成 shortage
2. **把 shortage 识别为 EOO** → 原本是短缺，公式却显示有 excess

---

## 优化方向

### 近似修正（Scenario Analysis）

通过以下两种方式 tighten EOO 标准：
1. **加入 Safety Stock**：将 Safety Stock 纳入 baseline，提高 EOO 认定门槛
2. **延长 Time Window**：用更长的窗口近似覆盖完整的 inventory 周期

本周数据已计算了五个 scenario：
| Scenario | Time Window |
|---|---|
| EOO within 3 months | 3 个月 |
| EOO within 6 months | 6 个月 |
| EOO within 9 months | 9 个月 |
| EOO within 12 months | 12 个月 |
| Firm Zone | 待确认准确数据 |

> 注：上述修正方式均为近似，无法完全解决周期不匹配的根本问题。

---

## 新方法：基于 Optimizer 的精确 EOO 计算

### 核心问题定义

> **在保证每周 inventory level 不低于 Safety Stock 的前提下，最小的 Open Order quantity 是多少（K）？**

用当前 Open Order quantity（H）减去该最小值（K），即为 **Excess Open Order（EOO）= H − K**。

### 计算逻辑

1. **计算每周"无 Open Order 时的 inventory level"**
   仅基于 On Hand 和 Actual Expected（已确认、不可取消的库存），计算每周净库存。

2. **找到 inventory 最低点**
   在所有周中，找到 inventory level 最低的那一周。

3. **判断是否需要补充 Open Order**
   - 如果最低点 ≥ Safety Stock → 无需任何 Open Order，EOO = 全部 Open Order
   - 如果最低点 < Safety Stock → 补充 Open Order，使该周 inventory 刚好等于 Safety Stock

4. **利用最低点的传导效应**
   该周是整个 horizon 的 inventory 最低点。只要它 ≥ Safety Stock，后续所有周的 on hand 均会相应提升，自然也 ≥ Safety Stock。因此，只需在最低点一次性补足，即可保证整个 horizon 的库存安全。

5. **得出最小 Open Order quantity（K）**
   上述补足量即为维持库存安全所需的最小 Open Order。与当前 Open Order（H）之差即为 EOO。

### 方法优势

- 直接回答"最少需要多少库存"，而非以 demand 为 baseline 间接推算
- 完整计入 Safety Stock 约束
- 不依赖 time window 的人工设定，避免周期错位问题
- 结果可解释性强：每个 part 的 EOO 对应一个具体的"最低点补足"操作

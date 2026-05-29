# FetchPredecoder — 取指预译码器设计方案

> 版本: 1.0 | 日期: 2026-05-13 | 模块: `v1.fetch.FetchPredecoder`

---

## 1. 问题描述

RISC-V RV64IMC 取指面临两个核心问题：

### 问题 1：指令段中有几条指令？

64-bit 取指块中可能包含 16-bit 压缩指令和 32-bit 标准指令的任意组合：

```
情况1: [16b][16b][16b][16b]  → 4条指令
情况2: [32b           ][32b           ]  → 2条指令
情况3: [16b][16b][32b           ]  → 3条指令
情况4: [16b][32b           ][16b]  → 3条指令
情况5: [32b           ][16b][16b]  → 3条指令
```

### 问题 2：32-bit 指令跨越两个取指块怎么办？

```
  取指块 N (64-bit)           取指块 N+1 (64-bit)
┌────────────────────┐    ┌────────────────────┐
│ 16b │ 32b      │ 跨│    │ 块 │ 32b           │
│     │          │ 块│    │    │               │
└────────────────────┘    └────────────────────┘
  0   2   4   6   8   10   ...
              ↑
         32-bit 指令只有低 16-bit 在当前块
         高 16-bit 在下一块
```

## 2. 方案概述

在取指阶段加入纯组合逻辑的**预译码器**，输入 64-bit 取指数据 + 上一块的残留半字（carry-in），输出指令边界信息 + 传递给下一块的残留半字（carry-out）。

```
                 ┌───────────────┐
  fetchData[63:0]│               │ instCount[2:0]
  carryIn[15:0]  │   FetchPre-   │ inst0-3 (valid/size/data)
  hasCarryIn     │   decoder     │
                 │  (组合逻辑)    │ carryOut[15:0]
                 │               │ hasCarryOut
                 └───────────────┘
```

## 3. 核心算法

### 3.1 半字分类

将 64-bit 块分割为 4 个固定 16-bit 半字（H0-H3），加上上一个 carry-in 半字 C：

```
   C (16b)     H0 (16b)    H1 (16b)    H2 (16b)    H3 (16b)
  [carryIn]   [15:0]      [31:16]     [47:32]     [63:48]
   slot 0      slot 1      slot 2      slot 3      slot 4
```

每个半字根据 `inst[1:0]` 分类：

```
halfword[1:0] == 11 && halfword[4:2] != 111  →  32-bit 指令起始
其他                                         →  16-bit 压缩指令
```

### 3.2 边界扫描

从起始 slot 开始（有 carry 则 slot 0，否则 slot 1），逐条确定：

```
i0: starts at startSlot
    if is32Start[i0Slot]: consumes 2 slots
    else:                consumes 1 slot

i1: starts at i0End
    if is32Start[i1Slot]: consumes 2 slots
    ...

i2: starts at i1End
    ...

i3: starts at i2End
    ...
```

### 3.3 跨块 (Straddle) 检测

```
一条 32-bit 指令跨越块边界：
  → 起始 slot ≤ 4
  → 结束 slot = 5 (需要下一块的 H0)
  → 当前块只包含该指令的低 16-bit
  → 低 16-bit 保存为 carryOut，下一拍拼到下一块前面
```

### 3.4 指令有效性

一条指令"完整"（valid）的条件：
- 结束 slot ≤ 4（完全在当前块内）
- 前一条指令不跨块

## 4. 时序示例

### 示例 1：无跨块（16+32+16）

```
取指块: [C.ADDI] [ADD x1,x2,x3] [C.ANDI]
         16b      32b             16b
         H0       H1+H2           H3

扫描:
  startSlot = 1 (no carry)
  H0: inst[1:0]=01 → 16-bit → i0Slot=1, i0End=2
  H1: inst[1:0]=11 → 32-bit → i1Slot=2, i1End=4
  H3: inst[1:0]=01 → 16-bit → i2Slot=4, i2End=5 → 超出! → i2 不完整

输出: instCount=2 (i0,i1 valid), hasCarryOut=0
```

### 示例 2：跨块（16+32 straddle）

```
取指块N:   [C.LI] [ADD x1,x2,x3  │
            16b    32b (低16b)   │
            H0     H1            │ H2 (低半字)

取指块N+1: │ x2,x3] [C.NOP]
            │ (高16b) 16b

扫描块N:
  startSlot = 1
  H0: 16-bit → i0Slot=1, i0End=2, i0Complete
  H1: 32-bit → i1Slot=2, i1End=4 → fit? i1End=4 ≤ 4 → yes, complete! 
  
  Wait — H1 starts at slot 2, needs slots 2+3=4, so i1End=4. This fits!
  
  Straddle happens when: 32-bit starts at H2 (slot 3) → needs slots 3+4=5 → slot 5 不存在
  或 starts at H3 (slot 4) → needs slots 4+5=6 → 跨块

真正跨块场景: [32b][16b][32b跨块]
  H0+H1: 32-bit → i0End=3
  H2: 16-bit → i1Slot=3, i1End=4
  H3: 32-bit start! → i2Slot=4, needs 4+2=6 → i2End=6 > 4 → STRADDLE
  i2 的低 16-bit (H3) → carryOut
  carryOut 与下一块的 H0 拼接 → 恢复完整 32-bit 指令

输出: instCount=2 (i0,i1), hasCarryOut=1, carryOut=H3
```

## 5. 硬件结构

```
  fetchData[63:0]
       │
  ┌────┴────┬────────┬────────┬────────┐
  │ H0[15:0]│H1[31:16]│H2[47:32]│H3[63:48]│  ← 固定位段提取(纯连线)
  └────┬────┴───┬────┴───┬────┴───┬────┘
       │        │        │        │
  ┌────▼────────▼────────▼────────▼────┐
  │    is32Start[0:4] 判定 (5-bit)     │  ← 每个半字的 inst[1:0] 检查
  └────────────────┬──────────────────┘
                   │
  ┌────────────────▼──────────────────┐
  │      边界扫描 & 跨块检测            │  ← Mux 级联 + 比较器
  │  i0Slot→i0End→i1Slot→...→i3Slot   │
  └────────────────┬──────────────────┘
                   │
  ┌────────────────▼──────────────────┐
  │   输出: instCount, inst0-3 info    │
  │         carryOut, hasCarryOut      │
  └───────────────────────────────────┘
```

## 6. 接口定义

| 信号 | 方向 | 位宽 | 说明 |
|------|------|------|------|
| `fetchData` | in | 64 | 当前对齐的取指数据 |
| `carryIn` | in | 16 | 上一块的残留半字 |
| `hasCarryIn` | in | 1 | carryIn 有效 |
| `instCount` | out | 3 | 完整指令数 (0-4) |
| `instNValid` | out | 1 | 第 N 条指令完整 |
| `instNSize` | out | 1 | 0=16b, 1=32b |
| `instNData` | out | 32 | 指令数据 (16b 时高 16 位为 0) |
| `carryOut` | out | 16 | 传递给下一块的残留半字 |
| `hasCarryOut` | out | 1 | carryOut 有效 |

## 7. 集成方式

```
  ┌─────────┐     ┌──────────────┐     ┌─────────┐
  │ ICache  │────►│ FetchPre-    │────►│ RVC      │────► 译码器
  │ / Bus   │ 64b │ decoder      │ 32b │ Expander │ 32b
  └─────────┘     └──────────────┘     └─────────┘
                       │
                  carryIn/Out
                       │
                  ┌────▼────┐
                  │ 16b Reg │  ← 流水线寄存器保存 carry
                  └─────────┘
```

1. ICache 返回 64-bit 对齐数据
2. Predecoder 在同一拍输出指令边界
3. 根据边界信息，并行取出 1-4 条指令送 RVC Expander
4. carryOut 存入流水线寄存器，下一拍作为 carryIn
5. 若 hasCarryOut=1，下一拍**不发起新的取指**——先消费 carry 完成跨块指令

## 8. 与 Rocket-Chip 方案的对比

| 方面 | Rocket-Chip RVCExpander | 本方案 FetchPredecoder |
|------|------------------------|----------------------|
| 功能 | 16b→32b 单条指令展开 | 64b 块中识别所有指令边界 |
| 输入 | 单条指令 (16b 或 32b) | 64-bit 取指块 + carry |
| 输出 | 展开后的 32b 指令 | 1-4 条指令的边界/数据 + carry |
| 跨块处理 | 不处理（依赖上层对齐） | 内置 carry 机制 |
| 架构层级 | 译码前 | 取指后、译码前 |

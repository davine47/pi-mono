# InstQueue 模块接口与设计文档

> 版本: 1.1 | 日期: 2026-05-18 | 模块路径: `v1.rvc.InstQueue`
> 依赖: SpinalHDL 1.13.0+, `v1.rvc.RVCDecoder`, `StreamFifo`

---

## 1. 概述

InstQueue 是指令取指后的**缓冲队列**，接收 RVCDecoder 输出的指令边界信息（每 64-bit 取指块最多 4 条指令），将原始指令位提取出来存入可配置深度的 FIFO，对外每次输出 1 条指令（含 isRVC 标识）。支持 `flush` 信号快速清空队列。

### 1.1 设计目标

| 目标 | 说明 |
|------|------|
| 速度匹配 | 取指端每拍最多产出 4 条 → 译码端每拍消费 1 条 |
| 原始位保留 | 16-bit 压缩指令存低 16 位，32-bit 存满 32 位，不预先展开 |
| 可配置深度 | `depth` 参数（默认 16），适配不同流水线深度 |
| Flush 支持 | `flush` 信号一拍清空所有内部状态（buffer + FIFO） |
| 反压安全 | `allEmpty` 检测确保 pushIdx 在空闲时复位，避免状态残留导致的乱序 |

### 1.2 参考

- `changbai/spinal/src/main/scala/v1/rvc/RVCDecoder.scala` — 指令边界扫描器
- `changbai/spinal/src/main/scala/v1/rvc/RVCExpander.scala` — 压缩指令展开（下游消费端）
- SpinalHDL `StreamFifo` — 内置流式 FIFO 组件

---

## 2. 参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `depth` | 16 | FIFO 深度（存储指令条数）。深度越大，吸收突发能力越强 |

深度选择参考：

| 场景 | 建议 depth |
|------|-----------|
| 单发射顺序流水线 | 4~8 |
| 双发射流水线 | 8~16 |
| 高 IPC 乱序前端 | 16~32 |

---

## 3. 接口定义

### 3.1 控制信号

| 信号 | 方向 | 位宽 | 说明 |
|------|------|------|------|
| `flush` | in | 1 | 高有效，一拍清空所有内部状态（buffer + FIFO + 推送 FSM） |
| `clk` | in | 1 | 时钟 |
| `reset` | in | 1 | 异步复位（高有效） |

### 3.2 输入（来自 RVCDecoder）

```
                   fetchData[63:0]
                   carryIn[15:0]
                   hasCarryIn
  RVCDecoder ──── inst0Valid, inst0Is32
                   inst1Valid, inst1Is32
                   inst2Valid, inst2Is32
                   inst3Valid, inst3Is32
                         │
                         ▼
                    InstQueue
```

| 信号 | 方向 | 位宽 | 说明 |
|------|------|------|------|
| `fetchData` | in | 64 | 64-bit 取指数据块 |
| `carryIn` | in | 16 | 上一块跨边界的低 16 位（只有 hasCarryIn=1 时有效） |
| `hasCarryIn` | in | 1 | 是否有来自上一块的 carryIn |
| `inst0Valid` | in | 1 | 第 0 条指令有效 |
| `inst0Is32` | in | 1 | 第 0 条是 32-bit 指令（=0 表示 16-bit 压缩指令） |
| `inst1Valid` | in | 1 | 第 1 条指令有效 |
| `inst1Is32` | in | 1 | 第 1 条是 32-bit |
| `inst2Valid` | in | 1 | 第 2 条指令有效 |
| `inst2Is32` | in | 1 | 第 2 条是 32-bit |
| `inst3Valid` | in | 1 | 第 3 条指令有效 |
| `inst3Is32` | in | 1 | 第 3 条是 32-bit |

> **注意**：`inst0Is32` 等信号仅在对应的 `instXValid=1` 时有效。这些信号可直接取自 `RVCDecoder.io`。

### 3.3 输出（1 条指令/周期）

| 信号 | 方向 | 位宽 | 说明 |
|------|------|------|------|
| `instValid` | out | 1 | 输出有效（高时 instBits/isRVC 有效） |
| `instBits` | out | 32 | 32-bit 容器：32-bit 指令存满；16-bit 指令存低 16 位（高 16 位为 0） |
| `isRVC` | out | 1 | 压缩指令标识（1=16-bit 压缩指令，0=32-bit 标准指令） |

### 3.4 完整端口表

| 端口 | 方向 | 位宽 | 分类 |
|------|------|------|------|
| `io_flush` | in | 1 | 控制 |
| `io_fetchData` | in | 64 | 取指 |
| `io_carryIn` | in | 16 | 取指 |
| `io_hasCarryIn` | in | 1 | 取指 |
| `io_inst0Valid` | in | 1 | 边界 |
| `io_inst0Is32` | in | 1 | 边界 |
| `io_inst1Valid` | in | 1 | 边界 |
| `io_inst1Is32` | in | 1 | 边界 |
| `io_inst2Valid` | in | 1 | 边界 |
| `io_inst2Is32` | in | 1 | 边界 |
| `io_inst3Valid` | in | 1 | 边界 |
| `io_inst3Is32` | in | 1 | 边界 |
| `io_instValid` | out | 1 | 输出 |
| `io_instBits` | out | 32 | 输出 |
| `io_isRVC` | out | 1 | 输出 |
| `clk` | in | 1 | 时钟 |
| `reset` | in | 1 | 复位 |

---

## 4. 微架构设计

### 4.1 整体框图

```
                   ┌─────────────────────────────────────────┐
                   │              InstQueue                  │
  flush ───────────┼──────────────────────────────┐          │
                   │                              ▼          │
  fetchData[63:0]  │  ┌──────────┐    ┌────────┐            │
  carryIn[15:0] ───┼─►│   Slot   │    │ 4-entry│            │
  hasCarryIn ──────┼─►│  位置     │───►│ Buffer │            │
                   │  │  计算    │    │ (暂存) │            │
  inst0Valid ──────┼─►│          │    └───┬────┘            │
  inst0Is32 ───────┼─►│ hwBySlot │        │                 │
  inst1Valid ──────┼─►│ raw32/16 │    ┌───▼────┐   ┌──────┐ │
  inst1Is32 ───────┼─►│          │    │  Push  │   │Stream│ │
  inst2Valid ──────┼─►└──────────┘    │  FSM   │──►│FIFO  │ │
  inst2Is32 ───────┼─                 │(轮询)  │   │(N深) │ │
  inst3Valid ──────┼─                  └────────┘   └──┬───┘ │
  inst3Is32 ───────┼─                     ▲             │     │
                   │                      │ flush       ▼     │
                   │                      ▼      instValid   │
                   │                   清空      instBits    │
                   │                   所有      isRVC       │
                   │                   状态                  │
                   └─────────────────────────────────────────┘
```

### 4.2 Slot 位置计算

与 RVCDecoder 内部逻辑一致，根据 `hasCarryIn` 和累积的指令大小计算每条指令在半字槽（halfword slot）中的起始位置：

```
i0Slot = hasCarryIn ? 0 : 1       // carry 有效时从 slot 0 开始，否则从 slot 1
i0Size = inst0Is32 ? 2 : 1        // 32-bit 占 2 个半字，16-bit 占 1 个
i1Slot = i0Slot + i0Size
i1Size = inst1Is32 ? 2 : 1
i2Slot = i1Slot + i1Size
i2Size = inst2Is32 ? 2 : 1
i3Slot = i2Slot + i2Size
```

### 4.3 Halfword 提取

5 个半字槽的来源：

| 槽 | 来源 | 说明 |
|----|------|------|
| hw0 | `carryIn[15:0]` | 来自上一取指块的跨边界低 16 位 |
| hw1 | `fetchData[15:0]` | 取指块 byte 0-1 |
| hw2 | `fetchData[31:16]` | 取指块 byte 2-3 |
| hw3 | `fetchData[47:32]` | 取指块 byte 4-5 |
| hw4 | `fetchData[63:48]` | 取指块 byte 6-7 |

`hwBySlot(slot)` 是一个组合逻辑 Mux（含默认值 0），根据 `slot`（0-4）选择对应半字。

### 4.4 指令位提取

```
32-bit 指令 (is32=1):  raw32 = {hw[slot+1], hw[slot]}     → 32 位满
16-bit 指令 (is32=0):  raw16 = {16'b0, hw[slot]}           → 低 16 位有效
```

每条指令用 `Mux(is32, raw32, raw16)` 得到 32-bit 容器值。16-bit 指令的高 16 位填 0。

### 4.5 暂存寄存器（Buffer）

4 条目暂存寄存器，每拍锁存 RVCDecoder 产出的指令。flush 时全部清空：

| 条目 | 数据来源 | 写入条件 | flush 行为 |
|------|---------|---------|------------|
| buf[0] | i0Bits, i0Rvc | `inst0Valid = 1` | `bufValid(0) := False` |
| buf[1] | i1Bits, i1Rvc | `inst1Valid = 1` | `bufValid(1) := False` |
| buf[2] | i2Bits, i2Rvc | `inst2Valid = 1` | `bufValid(2) := False` |
| buf[3] | i3Bits, i3Rvc | `inst3Valid = 1` | `bufValid(3) := False` |

每条目的有效位 `bufValid[i]` 在写入时置 1，被推入 FIFO 后清 0。

### 4.6 推送状态机

一个 2-bit 轮询状态机，顺序扫描 buf[0..3]，将有效条目逐个推入 StreamFifo：

```
  ┌──────────────────────────────┐
  │  allEmpty ?                  │
  │  (bufValid全部为0)           │──────► pushIdx := 0, pushing := False
  └──────────────────────────────┘
  ┌──────────────────────────────┐
  │  flush ?                     │──────► pushIdx := 0, pushing := False
  └──────────────────────────────┘
  ┌──────────────────────────────┐
  │  !pushing                    │
  │  bufValid[pushIdx] ?         │──────► pushing := True
  │  : pushIdx := pushIdx + 1    │         (循环 0→1→2→3→0)
  └──────────────────────────────┘
  ┌──────────────────────────────┐
  │  pushing                     │
  │  fifoIn.fire ?               │──────► bufValid[pushIdx] := False
  │  : wait                      │         pushIdx := pushIdx + 1
  └──────────────────────────────┘         pushing := False
```

**关键设计**：`allEmpty` 检测确保所有 buffer 条目为空时 `pushIdx` 复位为 0，避免跨测试/跨取指块的状态残留导致指令乱序输出。

### 4.7 StreamFifo

- **载荷**: 33 bits = `{instBits[31:0], isRVC}` 拼接
- **深度**: `depth`（可配置，默认 16）
- **写入**: 由推送状态机驱动，`fifoIn.valid := pushing`，每周期最多 1 次写入
- **读出**: `fifo.io.pop.ready := True`（始终就绪），输出直连 `io.instValid/instBits/isRVC`
- **Flush**: `fifo.io.flush := io.flush`，直接清空

### 4.8 关键设计决策

| 决策 | 理由 |
|------|------|
| 4-entry 暂存 + 1-wide 推送 FSM | 避免实现多写 FIFO（复杂度高），用简单状态机替代 |
| `allEmpty` 检测 + pushIdx 复位 | 防止 FSM 在多次入队/出队后 pushIdx 残留非零值导致乱序 |
| `fifoIn.valid := pushing` | 推送状态机直接控制 valid，仅在推送时有效 |
| 16-bit 指令不展开 | 保留原始位供下游 RVCExpander 处理，避免重复逻辑 |
| 33-bit 合并载荷 | 将 `instBits` 和 `isRVC` 打包为一个 FIFO 条目，简化控制 |
| `fifoPop.ready = True` | 下游随时可消费，反压由 FIFO 满信号间接体现（推送 FSM 等待） |

---

## 5. 时序行为

### 5.1 写入时序（取指块含 3 条指令）

```
          T0        T1        T2        T3        T4        T5
clk     __/‾‾\__/‾‾\__/‾‾\__/‾‾\__/‾‾\__/‾‾\__
             │         │         │         │         │
instXValid ──┘         │         │         │         │   取指块到达
             │         │         │         │         │
buf[0..2]   ───────────X─────────│─────────│─────────│   锁存 (T0上升沿)
             │         │         │         │         │
push FSM    ──────────► 检查 ──► 推送 ──► 检查 ──► 推送   逐个推入FIFO
             │       buf[0]   buf[0]   buf[1]   buf[1]
instValid   ─────────────────────────►├─────────│───   FIFO输出 (T3开始)
instBits    ─────────────────────────X── inst0  X── inst1
```

- T0 上升沿：RVCDecoder 输出有效，buf[0..2] 锁存
- T1 上升沿：推送 FSM 检测 buf[0] 有效 → `pushing = 1`
- T2 上升沿：`fifoIn.valid = 1`（pushing 已提交），FIFO 接受 → `fire`，buf[0] 推入
- T3 上升沿：buf[0] 出现在 FIFO 输出 → instValid=1，同时 FSM 处理 buf[1]
- T4-T5：后续指令依次输出

### 5.2 Flush 时序

```
          T0        T1        T2        T3
clk     __/‾‾\__/‾‾\__/‾‾\__/‾‾\__
             │         │         │
flush       ──┘         │         │    T0 拉起 flush
             │         │         │
bufValid    ──X─────────│─────────│   T0 上升沿：全部清0
pushIdx     ──X─────────│─────────│   pushIdx := 0
FIFO        ──X─────────│─────────│   FIFO 清空
             │         │         │
instValid   ──X─────────│─────────│   输出归0
             │         │         │
flush       ────────────┘         │   T1 拉低 flush
             │         │         │
正常工作    ─────────────────────X   可重新入队
```

`flush = 1` 在下一个时钟沿生效：
- `bufValid[*] := False`
- `pushIdx := 0`, `pushing := False`
- `fifo.io.flush := 1` → FIFO 立即清空
- `instValid` 变为 0

### 5.3 背压时序（FIFO 满）

```
          T0        T1        T2        T3
clk     __/‾‾\__/‾‾\__/‾‾\__/‾‾\__
             │         │         │
FIFO full    ├─────────┤         │    FIFO 满载
             │         │         │
push FSM     ├──►wait──┤──►push──│   等待 FIFO 空出位置
             │         │         │
instValid    ├─────────┤         │   输出暂停
```

推送 FSM 在 `fifoIn.fire = false` 时保持 `pushing = 1`，等待 FIFO 空间释放。下游消费（`fifoPop.ready = True`）会自然释放空间。

---

## 6. 与上下游集成

### 6.1 上游：RVCDecoder

```
  ┌──────────────┐        ┌──────────────┐
  │ RVCDecoder   │        │  InstQueue   │
  │              │        │              │
  │ fetchData ───┼───────►│ fetchData    │
  │ carryIn ─────┼───────►│ carryIn      │
  │ hasCarryIn───┼───────►│ hasCarryIn   │
  │ inst0Valid ──┼───────►│ inst0Valid   │
  │ inst0Is32 ───┼───────►│ inst0Is32    │
  │ ...          │        │ ...          │
  └──────────────┘        └──────┬───────┘
                                 │ instValid
                                 │ instBits[31:0]
                                 │ isRVC
                                 ▼
                          下游消费端
```

RVCDecoder 的所有输出直接连接到 InstQueue 的对应输入。

### 6.2 下游：RVCExpander + ScalarDecode

```
  InstQueue    instValid    ┌──────────────┐    ┌──────────────┐
  ────────────►├───────────►│ RVCExpander  │───►│ ScalarDecode │
               instBits     │ (if isRVC=1) │    │              │
               isRVC        │              │    │ 12-bit ctrl  │
                            └──────────────┘    └──────────────┘
```

- `isRVC = 1`：instBits[15:0] → RVCExpander → 32-bit → ScalarDecode
- `isRVC = 0`：instBits[31:0] → ScalarDecode（直接）

### 6.3 在 Frontend 中的位置

```
  ┌──────────────────────────────────────────────────────────┐
  │                       Frontend                           │
  │                                                          │
  │  fetchData ──► RVCDecoder ──► InstQueue ──► (下游译码)  │
  │                      │           │                       │
  │                      │      flush│ (分支预测失败等)       │
  │                      │           │                       │
  │  CPU req ───► Rw64Fetch ──────────────────► RW64 bus    │
  └──────────────────────────────────────────────────────────┘
```

---

## 7. 验证

### 7.1 测试环境

```bash
cd changbai/env/coco_tb/InstQueue
make          # 运行全部测试
make wave     # 查看波形
```

### 7.2 测试清单（7/7 通过，59623ns）

| # | 测试 | 说明 | 结果 |
|---|------|------|------|
| 1 | `test_single_32` | 1 条 32-bit 指令入队 → 出队 | PASS |
| 2 | `test_4x16` | 4 条 16-bit 压缩指令 → 依次出队 | PASS |
| 3 | `test_2x32` | 2 条 32-bit → 依次出队 | PASS |
| 4 | `test_mixed` | 混合 32/16-bit → 依次出队，验证 isRVC | PASS |
| 5 | `test_flush` | Flush 清空队列后输出归零 | PASS |
| 6 | `test_straddle` | hasCarryIn 跨块 32-bit 指令恢复 | PASS |
| 7 | `test_comprehensive` | 800 次随机混合 + flush，57861 cycles | PASS |

### 7.3 RTL 生成

```bash
# 默认 depth=16
make inst_queue

# 自定义深度
mill -i changbaiV1.spinal.runMain v1.rvc.GenInstQueue 32
```

---

## 8. 面积估算

| 组件 | 估算资源（depth=16） |
|------|---------------------|
| 4-entry 暂存器 | 4 × (1-bit valid + 32-bit data + 1-bit rvc) = ~136 FF |
| 推送 FSM | ~10 FF + 少量 LUT |
| `allEmpty` 逻辑 | 4-input OR + NOT = ~2 LUT |
| StreamFifo (33-bit × 16) | ~528 FF + 16 × 33 bit RAM (~1 BRAM 或分布式 RAM) |
| slot 计算 + 提取 | ~50 LUT（5:1 Mux + 加法器） |
| **总计** | **~700 FF + ~50 LUT + ~1 BRAM** |

---

## 9. 文件清单

| 路径 | 说明 |
|------|------|
| `changbai/spinal/src/main/scala/v1/rvc/InstQueue.scala` | SpinalHDL 源码 |
| `changbai/rtl/InstQueue.sv` | 生成的 Verilog |
| `changbai/rtl/StreamFifo.sv` | 依赖的 SpinalHDL 库组件 |
| `changbai/env/coco_tb/InstQueue/` | cocotb 测试环境 |
| `changbai/Makefile` | `inst_queue` 目标 |
| `changbai/assistant/pi-mono/work/instqueue-design-doc.md` | 本文档 |

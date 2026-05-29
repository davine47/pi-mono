# Regfile 模块接口与设计文档

> 版本: 1.0 | 日期: 2026-05-13 | 模块路径: `v1.regfile.Regfile`
> 依赖: SpinalHDL 1.13.0+

---

## 1. 概述

Regfile 模块实现 RISC-V RV64I 的 32 个体系结构整数寄存器（x0-x31）。采用可配置多端口设计，支持超标量处理器中同时进行多个读/写操作。

### 1.1 关键特性

| 特性 | 说明 |
|------|------|
| 寄存器宽度 | 64-bit (RV64)，可配置为 32-bit (RV32) |
| 寄存器数量 | 32 (RV32I/RV64I)，可配置为 16 (RV32E) |
| x0 硬连线 | 读始终返回 0，写被静默忽略 |
| 读端口数 | 可配置，默认 2（支持超标量双发射译码） |
| 写端口数 | 可配置，默认 1 |
| 读时序 | 组合逻辑读出（同周期返回数据） |
| 写时序 | 同步写入（时钟上升沿生效） |
| 多写冲突 | 高索引端口优先仲裁 |

---

## 2. 接口定义

### 2.1 默认配置 (2R1W)

生成 Verilog 后 `RegfileTop` 的顶层端口（SpinalHDL 将 `Vec` 展开为 `_N` 后缀）：

```
module RegfileTop (
  input  wire          io_clk,
  input  wire          io_reset,

  // Read port 0
  input  wire [4:0]    io_readAddr_0,
  output wire [63:0]   io_readData_0,

  // Read port 1
  input  wire [4:0]    io_readAddr_1,
  output wire [63:0]   io_readData_1,

  // Write port 0
  input  wire [4:0]    io_writeAddr_0,
  input  wire [63:0]   io_writeData_0,
  input  wire          io_writeEn_0
);
```

### 2.2 信号说明

| 信号 | 方向 | 位宽 | 说明 |
|------|------|------|------|
| `io_clk` | in | 1 | 时钟 |
| `io_reset` | in | 1 | 同步复位（高有效），复位后所有寄存器清零 |
| `io_readAddr_N` | in | 5 | 读地址，索引 0-31 对应 x0-x31 |
| `io_readData_N` | out | 64 | 读数据，组合逻辑输出，与 `readAddr` 同周期有效 |
| `io_writeAddr_N` | in | 5 | 写地址 |
| `io_writeData_N` | in | 64 | 写数据 |
| `io_writeEn_N` | in | 1 | 写使能，高有效 |

### 2.3 地址映射

```
readAddr/writeAddr  →  寄存器
    0               →  x0 (zero) — 读返回 0，写被忽略
    1               →  x1 (ra)
    2               →  x2 (sp)
    3               →  x3 (gp)
    ...
   31               →  x31 (t6)
```

---

## 3. 微架构设计

### 3.1 存储结构

```
┌─────────────────────────────────────────────────────────┐
│                    Vec(Reg) 寄存器阵列                    │
│                                                         │
│  regs(0)  = Reg[63:0]  ←→ x1 (ra)                      │
│  regs(1)  = Reg[63:0]  ←→ x2 (sp)                      │
│  regs(2)  = Reg[63:0]  ←→ x3 (gp)                      │
│  ...                                                    │
│  regs(30) = Reg[63:0]  ←→ x31 (t6)                     │
│                                                         │
│  x0 无物理存储，读路径直接返回 0                           │
└─────────────────────────────────────────────────────────┘
```

采用 **Vec(Reg)**（31 个独立触发器）而非 SRAM/Mem 宏单元，原因：

1. **多端口可配置**：`Mem` 的写端口数量在 SpinalHDL 中受限制且不能动态配置；触发器阵列天然支持任意数量的读写端口
2. **面积效率**：32×64 bit = 2048 bit 触发器，在超标量设计中通常可接受
3. **组合读路径**：触发器输出直接进入 Mux 树，无需 SRAM 读出使能时序

> 对面积敏感的设计可将 `Vec(Reg)` 替换为 `Mem`，但需固定端口数。

### 3.2 读数据路径

```
                    ┌──────────────┐
  io_readAddr_0 ───►│ one-hot 译码 │──► selBits[30:0]
                    └──────────────┘
                           │
     regs_0 (x1) ─────────┤
     regs_1 (x2) ─────────┤
     ...                  ├──► MuxOH(selBits, regs) ──► muxed
     regs_30 (x31) ───────┤
                           │
     io_readAddr == 0 ? ───┼──► Mux ──► io_readData_0
           0 ──────────────┘
```

关键设计决策：

| 组件 | 作用 |
|------|------|
| `UIntToOh(addr, 32)` | 将 5-bit 地址译码为 32-bit one-hot（addr=0 → bit0=1, addr=1 → bit1=1, ...） |
| `addrOh[31:1]` | 丢弃 bit0（x0），得到 31-bit 选择信号 |
| `MuxOH(selBits, regs)` | one-hot 多路选择器，选中的 reg 输出到 muxed |
| `Mux(addr===0, 0, muxed)` | 显式处理 x0：若地址为 0 则直接输出 0，绕过 MuxOH |

> **为什么需要显式 `Mux(addr===0)`？**  
> 当 `selBits` 全为 0 时（addr=0 或地址超出范围），`MuxOH` 的输出行为在 SpinalHDL 中是未定义的——它可能输出任意值或锁存器值。显式 Mux 确保 x0 始终返回 0。

### 3.3 写数据路径

```
  io_writeAddr_0 ─┬──► addrMatch(0) = en(0) & (addr(0)==i)
  io_writeData_0  │
  io_writeEn_0  ──┤
                  │
  io_writeAddr_1 ─┼──► addrMatch(1) = en(1) & (addr(1)==i)
  io_writeData_1  │
  io_writeEn_1  ──┘        ↓
                    ┌──────────────────┐
                    │ 优先级编码器       │
                    │ (高索引端口优先)   │
                    └──────────────────┘
                           │
                    winnerPort, hasWrite
                           │
                    ┌──────▼──────┐
                    │ hasWrite ?   │
                    │  regs(i-1) :=│──► D pin of flip-flop
                    │  writeData   │
                    │  (winnerPort)│
                    └─────────────┘
```

每个物理寄存器 x1-x31 拥有独立的写逻辑：

1. **地址匹配**：对每个写端口，检查 `writeEn(wp) && writeAddr(wp) == i`
2. **端口仲裁**：若多个端口匹配，高索引端口获胜（翻转 `addrMatch` 位向量后使用 `OHToUInt` 找最低置位 = 原始最高索引）
3. **条件赋值**：`when(hasWrite) { regs(i-1) := writeData(winnerPort) }`

> 没有写使能的寄存器保持当前值（`Reg` 默认行为——无驱动时保持）。

---

## 4. 时序行为详解

### 4.1 基本写后读时序

```
         T0      T1      T2      T3
clk    __/‾‾\__/‾‾\__/‾‾\__/‾‾\__
            │       │       │
writeEn  ───┘       │       │   写使能: T0上升沿采样
writeAddr ──────────┘       │   写地址: x5
writeData ──────────┘       │   写数据: 0xDEAD
                        　  │
readAddr  ──────────────────┘   读地址: x5
readData  ──────────xxx────────── 读数据: T3同周期返回 0xDEAD
                   (旧值)
```

- **T0 上升沿**：采样 writeEn/writeAddr/writeData，写入在 T0→T1 之间生效
- **T1 周期**：寄存器 x5 已更新为 0xDEAD，但 readAddr 若仍指向旧地址则读旧值
- **T2 上升沿**：采样 readAddr=x5
- **T2 周期**：组合读出 0xDEAD（因为 T1 周期寄存器已更新）

### 4.2 同周期读写同一地址（核心问题）

```
         T0      T1
clk    __/‾‾\__/‾‾\__
            │
writeEn  ───┘        写使能: T0上升沿采样
writeAddr ───┘        写地址: x5
writeData ───┘        写数据: 0xCAFE
readAddr  ───┘        读地址: x5 (同一周期!)
readData  ───┘──────── 读数据: T0周期返回 旧值 (不是 0xCAFE!)
            ↑
         旧值 (写入尚未生效)
```

**行为：读端口返回写入前的旧值，而非正在写入的新值。**

这是因为：

```
时间线 (T0 周期内):

1. 组合逻辑: readAddr=x5 → MuxOH选择 regs(4) → readData = regs(4)的当前值 (旧值)
                                                           ↑
                                                    此时还是旧值!
2. T0 上升沿: 采样 writeEn=1, writeAddr=x5, writeData=0xCAFE
3. T0→T1 之间: regs(4) 更新为 0xCAFE
4. T1 周期: readAddr=x5 → readData = 0xCAFE (新值)
```

**根本原因**：
- **读路径是组合逻辑**：数据直接来自寄存器触发器的 Q 端
- **写路径是时序逻辑**：数据在时钟上升沿写入触发器的 D 端，经过 clk-to-Q 延迟后才在 Q 端可见
- 同周期读发生在时钟沿**之前**的组合逻辑求值中，此时触发器 Q 端仍是旧值

### 4.3 多写端口同地址冲突

```
         T0
clk    __/‾‾\__
            │
writeEn_0 ───┘    Port0: 写 x5 ← 0xAAAA
writeEn_1 ───┘    Port1: 写 x5 ← 0xBBBB
                     │
                     ▼
              仲裁结果: Port1 获胜 (高索引优先)
              x5 最终值 = 0xBBBB
```

### 4.4 x0 写保护时序

```
writeEn=1, writeAddr=0, writeData=任意值
                    │
                    ▼
          addrMatch(0) 被硬件强制为 0
          (因为 x0 在写循环中从 i=1 开始，不匹配任何物理寄存器)
                    │
                    ▼
          hasWrite = False → regs 不受影响
          读路径: addr=0 → 显式 Mux 返回 0
```

x0 在写路径中被完全排除（写循环 `for (i <- 1 until numRegs)`），读路径有显式 `Mux(addr===0, 0, ...)` 双重保护。

---

## 5. 在超标量处理器中的使用

### 5.1 典型连接方式

```
    ┌──────────┐     ┌──────────────┐
    │ Decode0  │────►│ readAddr_0   │
    │  rs1=5   │    │ readData_0 ──► ALU0
    └──────────┘     │              │
                     │   Regfile    │
    ┌──────────┐     │              │
    │ Decode1  │────►│ readAddr_1   │
    │  rs1=8   │    │ readData_1 ──► ALU1
    └──────────┘     │              │
                     │ writeAddr_0◄── WB (写回级)
    ┌──────────┐     │ writeData_0◄── WB
    │ WB stage │────►│ writeEn_0  ◄── WB
    └──────────┘     └──────────────┘
```

### 5.2 RAW（Read-After-Write）冒险处理

由于同周期读返回旧值，RAW 冒险**必须由旁路（forwarding）逻辑解决**，不能依赖寄存器文件内部转发。

```
   Cycle N:   ALU 产生结果 → 写回 x5
   Cycle N+1: 后续指令读 x5 → Regfile 返回新值  ← 存在 1 周期气泡

   解决方案: 旁路网络 (forwarding mux)
   - 在 ALU 输入前插入 Mux
   - Mux 输入: Regfile 读数据, EX/MEM 结果, MEM/WB 结果
   - 比较源寄存器号与流水线寄存器中的目标寄存器号
```

### 5.3 多写端口扩展

```scala
// 2R2W 配置（双发射 + 双写回）
val config = RegfileConfig(
  readPorts  = 4,  // 双发射各需 rs1, rs2
  writePorts = 2,  // 双发射各需一条写回路径
  xlen       = 64
)
new RegfileTop(config)
```

此时若两个写端口同周期写同一寄存器，高索引端口（Port1）的数据被写入。

---

## 6. 配置参数

| 参数 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `readPorts` | Int | 2 | 读端口数量 |
| `writePorts` | Int | 1 | 写端口数量 |
| `xlen` | Int | 64 | 寄存器位宽 (32 or 64) |
| `numRegs` | Int | 32 | 寄存器数量 (16 or 32) |

命令行生成示例：
```bash
# 默认 2R1W, 64-bit
make regfile

# 4R2W 超标量配置
mill -i changbaiV1.spinal.runMain v1.regfile.GenRegfile 4 2 64 32
```

---

## 7. 面积估算 (2R1W, 64-bit)

| 组件 | 数量 | 位宽 | 等效门数 (估算) |
|------|------|------|-----------------|
| 触发器 (x1-x31) | 31 | 64 | ~12K gates |
| 读多路选择器 (2 ports) | 2 | 31→1×64 | ~8K gates |
| 写地址译码 & 仲裁 | 31 | — | ~2K gates |
| **总计** | | | **~22K gates** |

---

## 8. 验证

### 8.1 测试环境

```bash
cd changbai/env/coco_tb/Regfile
CHANGBAI_ROOT=/path/to/changbai make
make wave   # 打开 gtkwave 查看波形
```

### 8.2 测试清单

| 测试名称 | Cycles | 验证内容 |
|----------|--------|---------|
| test_x0_zero | ~10 | x0 读写保护，写入 x0 不改变任何状态 |
| test_basic_write_read | ~100 | 31 个寄存器逐次写后读 |
| test_dual_read | ~14 | 双端口同时读不同/相同地址 |
| test_write_read_forward | ~17 | 写后读同一寄存器，验证 1 周期后生效 |
| test_back_to_back_writes | ~69 | 连续快速写入 31 个寄存器不丢失 |
| test_no_crosstalk | ~388 | 重复写 x1，验证 x5/x10/x15/x20/x25/x31 不受影响 |
| test_bit_patterns | ~27 | 10 种边界位模式（全0/全1/交替/高位置1等） |
| test_address_masking | ~71 | 5-bit 地址覆盖全部 32 个寄存器 |
| test_comprehensive | ~2107 | 随机读写 + 软件模型比对，覆盖 2000+ cycles |
| test_x0_stress | ~458 | 反复写 x0 200 次，验证始终保持 0 |
| **总计** | **~3261** | |

---

## 9. 局限性

| 局限 | 说明 | 缓解方案 |
|------|------|---------|
| 无内部转发 | 同周期写后读返回旧值 | 依赖外部旁路网络 |
| Vec(Reg) 面积 | 31×64 触发器 ≈ 2048 FF | 可替换为 SRAM Mem（但需固定端口数） |
| 纯组合读 | 大 Mux 树可能影响关键路径 | 超标量通常 4-6 读端口，Mux 深度可控 |
| 无 ECC/奇偶校验 | 无软错误检测 | 如需可靠性，在外层包装 |

# RVC 模块接口与设计文档

> 版本: 1.0 | 日期: 2026-05-14 | 模块: `v1.rvc`
> 参考: rocket-chip RVC.scala, fetch-predecoder-design.md

---

## 1. 概述

RVC 模块实现 RISC-V 压缩指令扩展（C Extension）的硬件支持。它由两个子模块组成：

| 子模块 | 功能 | 状态 |
|--------|------|------|
| **RVCDecoder** | 64-bit 取指块指令边界扫描，识别每条指令的位置和大小 | RTL 已生成 |
| **RVCExpander** | 16-bit 压缩指令 → 32-bit 标准指令展开 | 源码完成，编译待调试 |

### 1.1 数据流

```
  取指 64-bit ──► RVCDecoder ──► inst0..3 (valid/size/data)
                                      │
                                      ▼
                               ┌──────────────┐
                               │ RVCExpander  │  (×4, parallel)
                               │ 16b → 32b    │
                               └──────────────┘
                                      │
                                      ▼
                               32-bit 标准指令 → 译码器
```

---

## 2. RVCDecoder — 指令边界扫描器

### 2.1 接口

| 信号 | 方向 | 位宽 | 说明 |
|------|------|------|------|
| `fetchData` | in | 64 | 8字节对齐的取指数据 |
| `carryIn` | in | 16 | 上一块的残留半字 |
| `hasCarryIn` | in | 1 | carryIn 有效标志 |
| `instCount` | out | 3 | 完整指令数 (0-4) |
| `inst[N]Valid` | out | 1 | 第 N 条指令完整 |
| `inst[N]Is32` | out | 1 | 0=16b, 1=32b |
| `carryOut` | out | 16 | 传递给下一块的残留半字 |
| `hasCarryOut` | out | 1 | carryOut 有效 |

### 2.2 算法

将 64-bit 块划分为 5 个固定 slot（carryIn + 4 个 fetch 半字）：

```
  Slot 0    Slot 1     Slot 2     Slot 3     Slot 4
 [carryIn] [15:0]     [31:16]    [47:32]    [63:48]
```

每个半字按 `inst[1:0]` 分类：
- `inst[1:0] == 11 && inst[4:2] != 111` → 32-bit 指令起始
- 其他 → 16-bit 压缩指令

从起始 slot 逐条扫描，确定每条指令的边界和大小。32-bit 指令跨越 slot 4 边界时产生 carryOut。

### 2.3 关键特性

- **纯组合逻辑**，当周期完成
- **5 种指令排列全覆盖**（16/16/16/16, 32/32, 16/16/32, 16/32/16, 32/16/16）
- **跨块处理**：32-bit 指令跨边界时产生 carryOut，下一拍拼接

---

## 3. RVCExpander — 压缩指令展开器

### 3.1 接口

| 信号 | 方向 | 位宽 | 说明 |
|------|------|------|------|
| `instIn` | in | 16 | 压缩指令（或 32-bit 指令的低 16 位） |
| `instOut.bits` | out | 32 | 展开后的 32-bit 指令 |
| `instOut.rd/rs1/rs2` | out | 5 | 解码出的寄存器字段 |
| `rvc` | out | 1 | 输入是压缩指令 (inst[1:0] ≠ 11) |
| `ill` | out | 1 | 输入是非法压缩编码 |

### 3.2 实现方式

采用 **when/elsewhen 链**（独立 when 块），以 5-bit opcode 索引 `{inst[1:0], inst[15:13]}` 区分 32 种情况：

```
索引 = {inst[1:0], inst[15:13]}

00xxx → 象限 0 (CIW/CL/CS): C.ADDI4SPN, C.LW, C.LD, C.SW, C.SD
01xxx → 象限 1 (CI/CB/CJ): C.ADDI, C.ADDIW, C.LI, C.LUI, C.J, C.BEQZ, C.BNEZ, ALU ops
10xxx → 象限 2 (CR/CI/CSS): C.SLLI, C.LWSP, C.LDSP, C.JR/MV/JALR/ADD, C.SWSP, C.SDSP
11xxx → 象限 3: 非压缩指令（直通）
```

### 3.3 展开示例

**C.ADDI (opIdx=8):** `c.addi x10, 5`
```
  输入: 0x4529 (0100_0101_0010_1001)
  
  funct3=000, rd=x10, nzimm=5
  → ADDI x10, x10, 5
  → 32-bit: {imm[11:0], 01010, 000, 01010, 0010011}
```

**C.J (opIdx=13):** `c.j 0x100`
```
  输入: offset 编码 in [12|8|10:9|6|7|2|11|5:3]
  → JAL x0, offset
  → 32-bit: {imm[20|10:1|11|19:12], 00000, 1101111}
```

### 3.4 非法指令检测

以下编码触发 `ill=1`：

| 指令 | 非法条件 |
|------|---------|
| C.ADDI4SPN | nzuimm = 0 |
| C.ADDIW | rd = x0 (RV64) |
| C.LUI | rd = x0 |
| C.LWSP / C.LDSP | rd = x0 |

---

## 4. 支持的指令全集

### 4.1 象限 0 (inst[1:0]=00)

| opIdx | 指令 | 展开为 |
|-------|------|--------|
| 0 | C.ADDI4SPN | ADDI rd', x2, nzuimm |
| 2 | C.LW | LW rd', offset(rs1') |
| 3 | C.LD | LD rd', offset(rs1') |
| 6 | C.SW | SW rs2', offset(rs1') |
| 7 | C.SD | SD rs2', offset(rs1') |

### 4.2 象限 1 (inst[1:0]=01)

| opIdx | 指令 | 展开为 |
|-------|------|--------|
| 8 | C.ADDI / C.NOP | ADDI rd, rd, imm |
| 9 | C.ADDIW | ADDIW rd, rd, imm |
| 10 | C.LI | ADDI rd, x0, imm |
| 11 | C.LUI / C.ADDI16SP | LUI rd, imm |
| 12 | C.SRLI/C.SRAI/C.ANDI/C.SUB... | 对应 ALU 指令 |
| 13 | C.J | JAL x0, offset |
| 14 | C.BEQZ | BEQ rs1', x0, offset |
| 15 | C.BNEZ | BNE rs1', x0, offset |

### 4.3 象限 2 (inst[1:0]=10)

| opIdx | 指令 | 展开为 |
|-------|------|--------|
| 16 | C.SLLI | SLLI rd, rd, shamt |
| 18 | C.LWSP | LW rd, offset(x2) |
| 19 | C.LDSP | LD rd, offset(x2) |
| 20 | C.JR/C.MV/C.JALR/C.ADD | 对应指令 |
| 22 | C.SWSP | SW rs2, offset(x2) |
| 23 | C.SDSP | SD rs2, offset(x2) |

---

## 5. 文件清单

| 文件 | 说明 |
|------|------|
| `spinal/.../v1/rvc/RVCDecoder.scala` | 指令边界扫描器（~140行） |
| `spinal/.../v1/rvc/RVCExpander.scala` | 压缩指令展开器（~210行） |
| `rtl/RVCDecoder.sv` | 生成的 RTL（8KB） |
| `work/fetch-predecoder-design.md` | 预译码器详细设计 |


## 6. 使用方式

```bash
# 生成 RTL
make rvc
# 输出: rtl/RVCDecoder.sv
```

在取指流水线中集成：

```scala
val predecoder = new RVCDecoder
predecoder.io.fetchData  := fetchData
predecoder.io.carryIn    := carryReg
predecoder.io.hasCarryIn := hasCarryReg

// 根据 instCount 和 inst*Valid/Is32 信号，
// 将 decoded 指令送入 RVCExpander 或直通译码器
```

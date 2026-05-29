# ALU 模块接口与设计文档

> 版本: 1.0 | 日期: 2026-05-13 | 模块路径: `v1.alu.Alu`
> 依赖: SpinalHDL 1.13.0+

---

## 1. 概述

ALU 模块实现 RV32I/RV64I 基础整数运算，包括算术、逻辑、移位和比较操作。

### 1.1 关键特性

| 特性 | 说明 |
|------|------|
| 数据宽度 | 64-bit (RV64)，可配置为 32-bit (RV32) |
| 组合逻辑 | 纯组合电路，无流水级，当周期输出结果 |
| 操作编码 | 5-bit `aluOp`，参考 RISC-V funct3 约定 |
| RV64 W后缀 | 支持 ADDW/SUBW/SLLW/SRLW/SRAW（32位运算+符号扩展） |
| 辅助操作 | LUI, AUIPC, JAL/JALR 地址计算 |

---

## 2. 接口定义

### 2.1 模块端口 (Alu)

```verilog
module Alu (
  input  wire [63:0]  io_src0,    // 操作数0 (rs1)
  input  wire [63:0]  io_src1,    // 操作数1 (rs2 / immediate)
  input  wire [4:0]   io_aluOp,   // 操作选择
  output wire [63:0]  io_result   // 运算结果
);
```

### 2.2 顶层包装 (AluTop) — 用于独立仿真

额外包含 `io_clk`, `io_reset` 端口（内部通过 ClockingArea 保持 Alu 纯组合）。

---

## 3. ALU 操作编码 (aluOp[4:0])

编码继承 RISC-V funct3 格式：`aluOp[2:0]` 对应指令 `funct3` 字段。

| aluOp | 助记符 | funct3 对应 | 功能 |
|-------|--------|------------|------|
| `0b00000` | ADD | 000 | rd = rs1 + rs2 |
| `0b00001` | SLL | 001 | rd = rs1 << rs2[5:0] |
| `0b00010` | SLT | 010 | rd = (signed(rs1) < signed(rs2)) ? 1 : 0 |
| `0b00011` | SLTU | 011 | rd = (rs1 < rs2) ? 1 : 0 |
| `0b00100` | XOR | 100 | rd = rs1 ^ rs2 |
| `0b00101` | SRL | 101 | rd = rs1 >> rs2[5:0] (逻辑右移) |
| `0b00110` | OR | 110 | rd = rs1 \| rs2 |
| `0b00111` | AND | 111 | rd = rs1 & rs2 |
| `0b01000` | ADDW | — | rd = sext32(rs1[31:0] + rs2[31:0]) |
| `0b01001` | SLLW | — | rd = sext32(rs1[31:0] << rs2[4:0]) |
| `0b01010` | LUI | — | rd = rs1 (U-immediate 通过 src1 传入) |
| `0b01011` | AUIPC | — | rd = rs1 + rs2 (PC + offset) |
| `0b01100` | JAL | — | rd = rs1 + rs2 (PC + 4，返回地址) |
| `0b01101` | SRLW | — | rd = sext32(rs1[31:0] >> rs2[4:0]) |
| `0b10000` | SUB | — | rd = rs1 - rs2 |
| `0b10101` | SRA | — | rd = rs1 >>> rs2[5:0] (算术右移) |
| `0b11000` | SUBW | — | rd = sext32(rs1[31:0] - rs2[31:0]) |
| `0b11101` | SRAW | — | rd = sext32(rs1[31:0] >>> rs2[4:0]) |

**未定义编码** (其余14个): 返回 0。

### 3.1 编码位字段

```
aluOp[4:0]
  [2:0] = funct3  (RISC-V 指令编码)
  [3]   = is32    (1 → 32-bit W-suffix 操作)
  [4]   = isAlt   (1 → funct3=000 时选 SUB, funct3=101 时选 SRA)
```

### 3.2 指令到 aluOp 映射示例

| RISC-V 指令 | opcode | funct3 | funct7[5] | aluOp |
|------------|--------|--------|-----------|-------|
| ADD | 0110011 | 000 | 0 | 00000 |
| SUB | 0110011 | 000 | 1 | 10000 |
| ADDI | 0010011 | 000 | — | 00000 |
| SLLI | 0010011 | 001 | 0 | 00001 |
| SLT | 0110011 | 010 | — | 00010 |
| XORI | 0010011 | 100 | — | 00100 |
| SRLI | 0010011 | 101 | 0 | 00101 |
| SRAI | 0010011 | 101 | 1 | 10101 |
| ORI | 0010011 | 110 | — | 00110 |
| ANDI | 0010011 | 111 | — | 00111 |
| ADDW | 0111011 | 000 | 0 | 01000 |
| SUBW | 0111011 | 000 | 1 | 11000 |
| LUI | 0110111 | — | — | 01010 |
| AUIPC | 0010111 | — | — | 01011 |

---

## 4. 微架构设计

### 4.1 数据路径

```
                          ┌──────────────┐
  io_src0 ────────────────┤              │
  io_src1 ────────────────┤  运算单元     │
                          │              │
  io_aluOp ──► 译码 ──────┤              ├──► io_result
                          │  ADD / SUB   │
                          │  SLL/SRL/SRA │
                          │  SLT / SLTU  │
                          │  AND/OR/XOR  │
                          │  ADDW/SUBW   │
                          │  LUI/AUIPC   │
                          └──────────────┘
```

### 4.2 实现方式

采用 **Vec(32) 查找表** 实现操作选择：

```scala
val resultMap = Vec.fill(32)(B(0, xlen bits))
resultMap(ADD)  := adderOut
resultMap(SUB)  := subberOut
// ... 每个操作预计算自己的结果存入对应槽位
result := resultMap(io.aluOp.asUInt)
```

优势：
- 所有运算并行计算，由 `aluOp` 选择输出
- 零开销添加新操作（只需增加一个 `resultMap(SLOT)` 赋值）
- SpinalHDL 自动生成完整的 case/mux 树

### 4.3 32-bit W-suffix 符号扩展

RV64 的 W-suffix 指令（ADDW/SUBW/SLLW/SRLW/SRAW）执行 32 位运算后将结果符号扩展到 64 位：

```
result[63:0] = {32{result32[31]}, result32[31:0]}
```

实现方式：

```scala
def sext32To64(hi: Bool, lo: Bits): Bits = {
  Mux(hi, B"32'hFFFFFFFF", B"32'h0") ## lo.asBits
}
resultMap(ADDW) := sext32To64(adderWOut(31), adderWOut.asBits)
```

> 注意：`B(bool, N bits)` 在 SpinalHDL 中只将 bool 放在最低位，高位补零，不是复制 N 次。正确做法是 `Mux(bit, allOnes, allZeros)`。

### 4.4 运算单元详情

| 运算 | 实现 |
|------|------|
| ADD/SUB | `src0 + src1`, `src0 - src1`（64-bit UInt，自然溢出截断） |
| SLL | `src0 \|<< shamt`（shamt 取 src1[5:0]） |
| SRL | `src0 \|>> shamt` |
| SRA | `src0.asSInt \|>> shamt`（算术移位） |
| SLT | `src0.asSInt < src1.asSInt` |
| SLTU | `src0.asUInt < src1.asUInt` |
| AND/OR/XOR | 按位逻辑操作 |
| ADDW/SUBW | `(src0[31:0] + src1[31:0]).resize(32)` 后符号扩展 |

---

## 5. 时序

ALU 是**纯组合逻辑**，无寄存器：

```
  io_src0  ──┐
  io_src1  ──┤ 组合逻辑云 → io_result
  io_aluOp ──┘

  延迟: 取决于综合结果，通常 < 1ns (成熟工艺)
  关键路径: 64-bit 加法器 + Mux 选择器
```

在 `AluTop` 中通过 `ClockingArea` 包装，输入在上升沿采样，输出在下一拍稳定。

---

## 6. 与译码器的接口

```
  指令 ──► 译码器 ──► aluOp[4:0]
                  ──► rs1 (→ src0)
                  ──► rs2/imm (→ src1)
                  ──► ALU ──► result → 写回 Regfile
```

译码器根据 `opcode` + `funct3` + `funct7[5]` 生成 `aluOp`：

```scala
// 伪代码示例
when(opcode === OP) {
  aluOp(2 downto 0) := funct3
  aluOp(3) := False  // 64-bit
  aluOp(4) := funct7(5)  // ADD/SUB or SRL/SRA
}
when(opcode === OP_32) {
  aluOp(2 downto 0) := funct3
  aluOp(3) := True   // 32-bit W-suffix
  aluOp(4) := funct7(5)
}
```

---

## 7. 验证

### 7.1 运行测试

```bash
cd changbai/env/coco_tb/ALU
CHANGBAI_ROOT=/path/to/changbai make
make wave   # gtkwave 查看波形
```

### 7.2 测试清单 (10/10 通过, ≥21050ns)

| 测试 | 说明 |
|------|------|
| test_add_sub | ADD/SUB/ADDW/SUBW 含溢出和边界 |
| test_shifts | SLL/SRL/SRA/SLLW/SRLW/SRAW 各种移位量 |
| test_logic | AND/OR/XOR 位模式 |
| test_slt | SLT/SLTU 有符号/无符号比较 |
| test_aux_ops | LUI/AUIPC/JAL 辅助操作 |
| test_edge_cases | 零操作数、最大/最小值、模64移位 |
| test_undefined_ops | 14个未定义 opcode 均返回 0 |
| test_back_to_back | 100次随机快速连续操作 |
| test_comprehensive | 21050ns 随机化全覆盖测试（>2000 cycles） |
| test_w_suffix_signext | W-suffix 符号扩展正确性 |

---

## 8. 面积估算

| 组件 | 估算门数 |
|------|---------|
| 64-bit 加法器 | ~400 gates |
| 64-bit 移位器 | ~600 gates |
| 64-bit 逻辑单元 (AND/OR/XOR) | ~200 gates |
| 比较器 (SLT/SLTU) | ~300 gates |
| 32-bit W-suffix 单元 | ~200 gates |
| 结果选择 Mux 树 (32→1 × 64bit) | ~1500 gates |
| **总计** | **~3200 gates** |

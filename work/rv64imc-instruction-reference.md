# RV64IMC 指令参考手册

> 参考源: RISC-V Unprivileged ISA Manual (riscv-isa-manual)
> 涵盖扩展: RV64I (Base Integer) + M (Integer Multiply/Divide) + C (Compressed)
> XLEN=64

---

## 1. 指令编码格式

### 1.1 32位指令格式 (RV64I / M)

```
R-type:  funct7[31:25] rs2[24:20] rs1[19:15] funct3[14:12] rd[11:7] opcode[6:0]
I-type:  imm[11:0][31:20]         rs1[19:15] funct3[14:12] rd[11:7] opcode[6:0]
S-type:  imm[11:5][31:25] rs2[24:20] rs1[19:15] funct3[14:12] imm[4:0][11:7] opcode[6:0]
B-type:  imm[12|10:5][31:25] rs2[24:20] rs1[19:15] funct3[14:12] imm[4:1|11][11:7] opcode[6:0]
U-type:  imm[31:12][31:12]                                   rd[11:7] opcode[6:0]
J-type:  imm[20|10:1|11|19:12][31:12]                        rd[11:7] opcode[6:0]
```

### 1.2 16位压缩指令格式 (C扩展)

```
CR:  funct4[15:12]  rd/rs1[11:7]  rs2[6:2]    op[1:0]
CI:  funct3[15:13]  imm[12]       rd/rs1[11:7] imm[6:2]     op[1:0]
CSS: funct3[15:13]  imm[12:7]                   rs2[6:2]     op[1:0]
CIW: funct3[15:13]  imm[12:5]                   rd′[4:2]     op[1:0]
CL:  funct3[15:13]  imm[12:10]    rs1′[9:7]    imm[6:5]     rd′[4:2]    op[1:0]
CS:  funct3[15:13]  imm[12:10]    rs1′[9:7]    imm[6:5]     rs2′[4:2]   op[1:0]
CB:  funct3[15:13]  imm[12:10]    rs1′[9:7]    imm[6:2]                  op[1:0]
CJ:  funct3[15:13]  imm[12:2]                                            op[1:0]
```

---

## 2. Opcode 映射表

| opcode[6:0] | 类别 | 指令格式 |
|-------------|------|---------|
| `0110111` | LUI | U-type |
| `0010111` | AUIPC | U-type |
| `1101111` | JAL | J-type |
| `1100111` | JALR | I-type |
| `1100011` | BRANCH | B-type |
| `0000011` | LOAD | I-type |
| `0100011` | STORE | S-type |
| `0010011` | OP-IMM | I-type |
| `0110011` | OP | R-type |
| `0011011` | OP-IMM-32 | I-type |
| `0111011` | OP-32 | R-type |
| `0001111` | MISC-MEM | I-type |
| `1110011` | SYSTEM | I-type |

---

## 3. RV64I 基础整数指令

### 3.1 整数寄存器-立即数指令 (OP-IMM, opcode=`0010011`)

| 指令 | funct3 | funct7/imm[11:5] | 格式 | 功能 |
|------|--------|------------------|------|------|
| ADDI | 000 | — | I | rd = rs1 + sext(imm[11:0]) |
| SLLI | 001 | imm[11:6]=`000000` | I | rd = rs1 << imm[5:0] (逻辑左移) |
| SLTI | 010 | — | I | rd = (signed(rs1) < sext(imm)) ? 1 : 0 |
| SLTIU | 011 | — | I | rd = (unsigned(rs1) < sext(imm)) ? 1 : 0 |
| XORI | 100 | — | I | rd = rs1 ^ sext(imm) |
| SRLI | 101 | imm[11:6]=`000000` | I | rd = rs1 >> imm[5:0] (逻辑右移) |
| SRAI | 101 | imm[11:6]=`010000` | I | rd = rs1 >>> imm[5:0] (算术右移) |
| ORI | 110 | — | I | rd = rs1 | sext(imm) |
| ANDI | 111 | — | I | rd = rs1 & sext(imm) |

> RV64I中移位量取 imm[5:0] (6位)，RV32I取 imm[4:0] (5位)

### 3.2 整数寄存器-立即数指令 (OP-IMM-32, opcode=`0011011`) — RV64I only

| 指令 | funct3 | funct7/imm[11:5] | 格式 | 功能 |
|------|--------|------------------|------|------|
| ADDIW | 000 | — | I | rd = sext32(rs1[31:0] + sext(imm)) |
| SLLIW | 001 | imm[11:5]=`0000000` | I | rd = sext32(rs1[31:0] << imm[4:0]) |
| SRLIW | 101 | imm[11:5]=`0000000` | I | rd = sext32(rs1[31:0] >> imm[4:0]) |
| SRAIW | 101 | imm[11:5]=`0100000` | I | rd = sext32(rs1[31:0] >>> imm[4:0]) |

> imm[5]必须为0，否则为保留编码

### 3.3 整数寄存器-寄存器指令 (OP, opcode=`0110011`)

| 指令 | funct3 | funct7 | 格式 | 功能 |
|------|--------|--------|------|------|
| ADD | 000 | `0000000` | R | rd = rs1 + rs2 |
| SUB | 000 | `0100000` | R | rd = rs1 - rs2 |
| SLL | 001 | `0000000` | R | rd = rs1 << rs2[5:0] |
| SLT | 010 | `0000000` | R | rd = (signed(rs1) < signed(rs2)) ? 1 : 0 |
| SLTU | 011 | `0000000` | R | rd = (rs1 < rs2) ? 1 : 0 |
| XOR | 100 | `0000000` | R | rd = rs1 ^ rs2 |
| SRL | 101 | `0000000` | R | rd = rs1 >> rs2[5:0] (逻辑右移) |
| SRA | 101 | `0100000` | R | rd = rs1 >>> rs2[5:0] (算术右移) |
| OR | 110 | `0000000` | R | rd = rs1 | rs2 |
| AND | 111 | `0000000` | R | rd = rs1 & rs2 |

### 3.4 整数寄存器-寄存器指令 (OP-32, opcode=`0111011`) — RV64I only

| 指令 | funct3 | funct7 | 格式 | 功能 |
|------|--------|--------|------|------|
| ADDW | 000 | `0000000` | R | rd = sext32(rs1[31:0] + rs2[31:0]) |
| SUBW | 000 | `0100000` | R | rd = sext32(rs1[31:0] - rs2[31:0]) |
| SLLW | 001 | `0000000` | R | rd = sext32(rs1[31:0] << rs2[4:0]) |
| SRLW | 101 | `0000000` | R | rd = sext32(rs1[31:0] >> rs2[4:0]) |
| SRAW | 101 | `0100000` | R | rd = sext32(rs1[31:0] >>> rs2[4:0]) |

### 3.5 上立即数指令 (U-type)

| 指令 | opcode | 格式 | 功能 |
|------|--------|------|------|
| LUI | `0110111` | U | rd = sext(imm[31:12] << 12) |
| AUIPC | `0010111` | U | rd = pc + sext(imm[31:12] << 12) |

### 3.6 无条件跳转指令

| 指令 | opcode | funct3 | 格式 | 功能 |
|------|--------|--------|------|------|
| JAL | `1101111` | — | J | rd = pc+4; pc += sext(offset[20:1]) |
| JALR | `1100111` | `000` | I | rd = pc+4; pc = (rs1 + sext(imm)) & ~1 |

> JAL目标为±1MiB范围，JALR目标低位置0保证对齐

### 3.7 条件分支指令 (BRANCH, opcode=`1100011`)

| 指令 | funct3 | 格式 | 功能 (条件成立则pc += offset) |
|------|--------|------|------|
| BEQ | `000` | B | rs1 == rs2 |
| BNE | `001` | B | rs1 != rs2 |
| BLT | `100` | B | signed(rs1) < signed(rs2) |
| BGE | `101` | B | signed(rs1) >= signed(rs2) |
| BLTU | `110` | B | rs1 < rs2 (unsigned) |
| BGEU | `111` | B | rs1 >= rs2 (unsigned) |

> 分支偏移范围为±4KiB

### 3.8 Load指令 (LOAD, opcode=`0000011`)

| 指令 | funct3 | 格式 | 功能 |
|------|--------|------|------|
| LB | `000` | I | rd = sext(M[rs1+offset][7:0]) |
| LH | `001` | I | rd = sext(M[rs1+offset][15:0]) |
| LW | `010` | I | rd = sext(M[rs1+offset][31:0]) |
| LD | `011` | I | rd = M[rs1+offset][63:0] (RV64I) |
| LBU | `100` | I | rd = M[rs1+offset][7:0] (零扩展) |
| LHU | `101` | I | rd = M[rs1+offset][15:0] (零扩展) |
| LWU | `110` | I | rd = M[rs1+offset][31:0] (零扩展, RV64I) |

### 3.9 Store指令 (STORE, opcode=`0100011`)

| 指令 | funct3 | 格式 | 功能 |
|------|--------|------|------|
| SB | `000` | S | M[rs1+offset][7:0] = rs2[7:0] |
| SH | `001` | S | M[rs1+offset][15:0] = rs2[15:0] |
| SW | `010` | S | M[rs1+offset][31:0] = rs2[31:0] |
| SD | `011` | S | M[rs1+offset][63:0] = rs2[63:0] (RV64I) |

### 3.10 内存排序指令 (MISC-MEM, opcode=`0001111`)

| 指令 | funct3 | rd | rs1 | imm[11:0] | 功能 |
|------|--------|----|-----|-----------|------|
| FENCE | `000` | 0 | 0 | {fm,PI,PO,PR,PW,SI,SO,SR,SW} | 内存/设备访问排序 |
| FENCE.I | `001` | 0 | 0 | 0 | 指令缓存同步 |
| FENCE.TSO | `000` | 0 | 0 | fm=`1000`, pred=`RW`, succ=`RW` | TSO兼容FENCE |

### 3.11 系统指令 (SYSTEM, opcode=`1110011`)

| 指令 | funct3 | func12 | 格式 | 功能 |
|------|--------|--------|------|------|
| ECALL | `000` | `000000000000` | I | 向执行环境发起服务请求 |
| EBREAK | `000` | `000000000001` | I | 调试断点 |
| CSRRW | `001` | CSR地址[11:0] | I | rd = CSR; CSR = rs1 |
| CSRRS | `010` | CSR地址[11:0] | I | rd = CSR; CSR |= rs1 |
| CSRRC | `011` | CSR地址[11:0] | I | rd = CSR; CSR &= ~rs1 |
| CSRRWI | `101` | CSR地址[11:0] | I | rd = CSR; CSR = uimm[4:0] |
| CSRRSI | `110` | CSR地址[11:0] | I | rd = CSR; CSR |= uimm[4:0] |
| CSRRCI | `111` | CSR地址[11:0] | I | rd = CSR; CSR &= ~uimm[4:0] |

> CSR指令的rs1字段在立即数形式下编码5位零扩展立即数uimm[4:0]

### 3.12 NOP

| 指令编码 | opcode | funct3 | rd | rs1 | imm | 功能 |
|----------|--------|--------|----|-----|-----|------|
| ADDI x0, x0, 0 | `0010011` | `000` | 0 | 0 | 0 | 空操作 |

---

## 4. M扩展 — 整数乘除法指令

### 4.1 乘法指令 (OP, funct7=`0000001`, opcode=`0110011`)

| 指令 | funct3 | 格式 | 功能 |
|------|--------|------|------|
| MUL | `000` | R | rd = (rs1 * rs2)[63:0] |
| MULH | `001` | R | rd = (signed(rs1) * signed(rs2))[127:64] |
| MULHSU | `010` | R | rd = (signed(rs1) * unsigned(rs2))[127:64] |
| MULHU | `011` | R | rd = (rs1 * rs2)[127:64] (unsigned) |

### 4.2 除法/取余指令 (OP, funct7=`0000001`, opcode=`0110011`)

| 指令 | funct3 | 格式 | 功能 |
|------|--------|------|------|
| DIV | `100` | R | rd = signed(rs1) / signed(rs2) (向零舍入) |
| DIVU | `101` | R | rd = rs1 / rs2 (unsigned) |
| REM | `110` | R | rd = signed(rs1) % signed(rs2) (符号=被除数) |
| REMU | `111` | R | rd = rs1 % rs2 (unsigned) |

### 4.3 RV64I专用乘除法 (OP-32, funct7=`0000001`, opcode=`0111011`)

| 指令 | funct3 | 格式 | 功能 |
|------|--------|------|------|
| MULW | `000` | R | rd = sext32(rs1[31:0] * rs2[31:0]) |
| DIVW | `100` | R | rd = sext32(signed(rs1[31:0]) / signed(rs2[31:0])) |
| DIVUW | `101` | R | rd = sext32(rs1[31:0] / rs2[31:0]) |
| REMW | `110` | R | rd = sext32(signed(rs1[31:0]) % signed(rs2[31:0])) |
| REMUW | `111` | R | rd = sext32(rs1[31:0] % rs2[31:0]) |

### 4.4 除零与溢出行为

| 条件 | 被除数 | 除数 | DIV[U] | REM[U] | DIV[U]W | REM[U]W |
|------|--------|------|--------|--------|---------|---------|
| 除零 | x | 0 | 2^L^-1 | x | 2^32^-1 | x |
| 有符号溢出 | -2^L-1^ | -1 | -2^L-1^ | 0 | -2^31^ | 0 |

> L = XLEN for DIV/DIVU/REM/REMU, L = 32 for W-suffixed

---

## 5. C扩展 — 16位压缩指令

### 5.1 编码象限 (由inst[1:0]决定)

| inst[1:0] | 象限 | 使用说明 |
|-----------|------|---------|
| `00` | Quadrant 0 | 栈指针相关加载/存储、寄存器加载/存储 |
| `01` | Quadrant 1 | 立即数运算、跳转、分支 |
| `10` | Quadrant 2 | 寄存器运算、栈指针加载/存储 |
| `11` | — | 非16位指令 (32位或更长) |

### 5.2 C0象限指令 (inst[1:0]=`00`)

**CIW格式 — C.ADDI4SPN**

| 指令 | funct3 | 功能 |
|------|--------|------|
| C.ADDI4SPN | `000` | rd' = sp + nzuimm[9:2] (nzuimm≠0，rd'为x8+x′) |

**CL格式 — 加载**

| 指令 | funct3 | 功能 |
|------|--------|------|
| C.LW | `010` | rd' = sext(M[rs1'+offset])(31:0) |
| C.LD | `011` | rd' = M[rs1'+offset](63:0) (RV64) |

> rd', rs1' 编码为 x8 + 3位索引 (寄存器x8-x15)

**CS格式 — 存储**

| 指令 | funct3 | 功能 |
|------|--------|------|
| C.SW | `110` | M[rs1'+offset] = rs2'(31:0) |
| C.SD | `111` | M[rs1'+offset] = rs2'(63:0) (RV64) |

### 5.3 C1象限指令 (inst[1:0]=`01`)

**CI格式 — 立即数运算**

| 指令 | funct3 | 条件 | 功能 |
|------|--------|------|------|
| C.NOP | `000` | nzimm=0, rd=0 | 空操作 |
| C.ADDI | `000` | rd≠0 | rd = rd + nzimm[5:0] |
| C.ADDIW | `001` | rd≠0 | rd = sext32(rd[31:0] + imm[5:0]) (RV64) |
| C.LI | `010` | rd≠0 | rd = imm[5:0] |
| C.LUI | `011` | rd≠0, rd≠2 | rd = sext(nzimm[17:12] << 12) |
| C.ADDI16SP | `011` | rd=2 | sp = sp + sext(nzimm[9:4] × 16) |

**CI/CB混合格式 — 移位与逻辑**

| 指令 | funct3 | funct2 | 条件 | 功能 |
|------|--------|--------|------|------|
| C.SRLI | `100` | `00` | — | rd' = rd' >> shamt[5:0] (逻辑) |
| C.SRAI | `100` | `01` | — | rd' = rd' >>> shamt[5:0] (算术) |
| C.ANDI | `100` | `10` | — | rd' = rd' & imm[5:0] |
| C.SUB | `100` | `11` | funct6=`100011` | rd' = rd' - rs2' |
| C.XOR | `100` | `11` | funct6=`100011` | rd' = rd' ^ rs2' |
| C.OR | `100` | `11` | funct6=`100011` | rd' = rd' | rs2' |
| C.AND | `100` | `11` | funct6=`100011` | rd' = rd' & rs2' |
| C.SUBW | `100` | `11` | funct6=`100111` | rd' = sext32(rd'[31:0] - rs2'[31:0]) (RV64) |
| C.ADDW | `100` | `11` | funct6=`100111` | rd' = sext32(rd'[31:0] + rs2'[31:0]) (RV64) |

> inst[12] (bit12) 编码 shamt[5] / imm[5]

**CJ格式 — 跳转**

| 指令 | funct3 | 功能 |
|------|--------|------|
| C.J | `101` | pc += sext(offset[11:1]) |
| C.JAL | `101` | ra = pc+2; pc += sext(offset[11:1]) (RV32) |

> RV64中C.JAL无效，保留用于其他用途

**CB格式 — 分支**

| 指令 | funct3 | 功能 |
|------|--------|------|
| C.BEQZ | `110` | if rs1'==0 pc += sext(offset[8:1]) |
| C.BNEZ | `111` | if rs1'!=0 pc += sext(offset[8:1]) |

### 5.4 C2象限指令 (inst[1:0]=`10`)

**CI格式 — 移位与加载**

| 指令 | funct3 | 条件 | 功能 |
|------|--------|------|------|
| C.SLLI | `000` | rd≠0, shamt≠0 | rd = rd << shamt[5:0] (RV32:shamt[5]=0) |
| C.LWSP | `010` | rd≠0 | rd = sext(M[sp+offset])(31:0) |
| C.LDSP | `011` | rd≠0 | rd = M[sp+offset](63:0) (RV64) |

**CR格式 — 寄存器操作与跳转**

| 指令 | funct4 | rs2 | 条件 | 功能 |
|------|--------|-----|------|------|
| C.JR | `1000` | 0 | rs1≠0 | pc = rs1 |
| C.MV | `1000` | ≠0 | rd≠0 | rd = rs2 |
| C.JALR | `1001` | 0 | rs1≠0 | ra = pc+2; pc = rs1 |
| C.ADD | `1001` | ≠0 | rd≠0 | rd = rd + rs2 |
| C.EBREAK | `1001` | 0 | rs1=0, rs2=0 | 调试断点 |

**CSS格式 — 栈指针存储**

| 指令 | funct3 | 功能 |
|------|--------|------|
| C.SWSP | `110` | M[sp+offset] = rs2(31:0) |
| C.SDSP | `111` | M[sp+offset] = rs2(63:0) (RV64) |

### 5.5 非法压缩指令编码

以下编码被定义为非法指令(所有位全零):

| inst[15:0] | 说明 |
|------------|------|
| `0000000000000000` | 全零16位编码为非法指令 |

### 5.6 压缩指令寄存器映射

压缩指令使用3位寄存器索引 rd′, rs1′, rs2′，映射到整数寄存器:

| 3位编码 | 寄存器 | ABI名称 |
|---------|--------|---------|
| 000 | x8 | s0/fp |
| 001 | x9 | s1 |
| 010 | x10 | a0 |
| 011 | x11 | a1 |
| 100 | x12 | a2 |
| 101 | x13 | a3 |
| 110 | x14 | a4 |
| 111 | x15 | a5 |

---

## 6. 汇编伪指令对照

| 伪指令 | 等价实际指令 | 功能 |
|--------|-------------|------|
| NOP | ADDI x0, x0, 0 | 空操作 |
| MV rd, rs | ADDI rd, rs, 0 | 寄存器复制 |
| NOT rd, rs | XORI rd, rs, -1 | 按位取反 |
| NEG rd, rs | SUB rd, x0, rs | 取负数 |
| SEQZ rd, rs | SLTIU rd, rs, 1 | rd = (rs==0)?1:0 |
| SNEZ rd, rs | SLTU rd, x0, rs | rd = (rs!=0)?1:0 |
| SLTZ rd, rs | SLT rd, rs, x0 | rd = (rs<0)?1:0 |
| SGTZ rd, rs | SLT rd, x0, rs | rd = (rs>0)?1:0 |
| J offset | JAL x0, offset | 无条件跳转 |
| JR rs | JALR x0, rs, 0 | 寄存器间接跳转 |
| RET | JALR x0, ra, 0 | 函数返回 |
| CALL offset | AUIPC ra, offset[31:12]; JALR ra, ra, offset[11:0] | 函数调用 |
| LI rd, imm | (组合指令) | 加载立即数 |
| LA rd, symbol | (组合指令) | 加载地址 |
| CSRR rd, csr | CSRRS rd, csr, x0 | 读CSR |
| CSRW csr, rs1 | CSRRW x0, csr, rs1 | 写CSR |
| CSRS csr, rs1 | CSRRS x0, csr, rs1 | 置位CSR |
| CSRC csr, rs1 | CSRRC x0, csr, rs1 | 清除CSR |

---

## 7. 指令数量统计

| 扩展 | 指令数 | 说明 |
|------|--------|------|
| RV32I | 40 | 基础整数指令 (含ECALL/EBREAK/FENCE) |
| RV64I | +15 | 增加的W后缀指令、LD/SD/LWU等 |
| M | 8 (RV32) + 5 (RV64) | 乘除法指令 |
| Zicsr | 6 | CSR操作指令 |
| Zifencei | 1 | FENCE.I |
| C (RV64) | ~29 | 16位压缩指令 |
| **合计 RV64IMC** | **~90** | 去重后独立指令数 |

---

## 8. 立即数编码说明

### 8.1 B-type立即数 (分支偏移)

```
inst[31]   inst[7]   inst[30:25]   inst[11:8]
imm[12]    imm[11]   imm[10:5]     imm[4:1]
偏移 = sext({imm[12:1], 0})  // 最低位始终为0，单位为2字节
```

### 8.2 J-type立即数 (JAL偏移)

```
inst[31]   inst[19:12]   inst[20]   inst[30:21]   inst[7]
imm[20]    imm[19:12]    imm[11]    imm[10:1]     (忽略，设0)
偏移 = sext({imm[20:1], 0})  // 最低位始终为0，单位为2字节
```

### 8.3 C扩展立即数

压缩指令的立即数字段是分散编码的，各指令格式不同。所有立即数均按要求进行符号扩展。

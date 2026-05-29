# ScalaDecode

## Introduction
此模块为ScalarDecoder中的译码映射表，详细描述了每条指令所对应的译码表

## Design
* 根据以下译码模式和指令译码表更新路径changbai/spinal/src/main/scala/v1/下的ScalarDecode的译码逻辑
* 严格参考spinal/src/test/scala/VectorDecodeDemo.scala的实现方式，使用spinal/src/main/scala/v1/utils/Decode.scala进行译码
* 严格参考knowledge/rocket-chip/src/main/scala/rocket/IDecode.scala
* 译码模式（各字段默认值是0）

| 字段名称       | 类型/宽度       | 解释                  |
|------------|-------------|---------------------|
| legel      | Bool/1 bit  | 代表指令译码合法，译码表中没有即为非法 |
| branch     | Bool/1 bit  | 指令是有条件分支            |
| jal        | Bool/1 bit  | 指令是jal              |
| jalr       | Bool/1 bit  | 指令是jalr             |
| rrf1       | Bool/1 bit  | 指令需要读取第一个操作数        |
| rrf2       | Bool/1 bit  | 指令需要读取第二个操作数        |
| wrf1       | Bool/1 bit  | 指令需要写回第一个操作数        |
| useALU     | Bool/1 bit  | 指令使用ALU             |
| aluOp      | Bits/5 bits | 指令操作ALU的操作码         |
| useMem     | Bool/1 bit  | 指令使用Memory          |
| memOp      | Bits/5 bits | 指令操作memory单元的操作码    |
| memResOp   | Bits/3 bits | 取数结果进行规整化操作         |
| useCsr     | Bool/1 bit  | 指令是CSR指令，使用csr单元    |
| csrOp      | Bits/3 bits | CSR指令对于CSR单元的操作码    |
| needImmExt | Bool/1 bit  | 指令包含立即数，需要做立即数拓展    |
| immExtType | Bits/3 bit  | 指令立即数拓展类型           |
| fence      | Bool/1 bit  | 指令是fence            |
| fenceI     | Bool/1 bit  | 指令是fence.i          |
| amo        | Bool/1 bit  | 指令是amo类型            |

* 指令译码（未标明字段为无关项，设置为默认值)，aluOp参考alu.md，immExt编码表参考signext.md

| 指令     | 译码表                                                                                  |
|--------|--------------------------------------------------------------------------------------|
| ADD    | ill=0, rrf1=1, rrf2=1, wrf1=1, useALU=1, aluOp=ADD                                   |
| SUB    | ill=0, rrf1=1, rrf2=1, wrf1=1, useALU=1, aluOp=SUB                                   |
| SLL    | ill=0, rrf1=1, rrf2=1, wrf1=1, useALU=1, aluOp=SLL                                   |
| SLT    | ill=0, rrf1=1, rrf2=1, wrf1=1, useALU=1, aluOp=SLT                                   |
| SLTU   | ill=0, rrf1=1, rrf2=1, wrf1=1, useALU=1, aluOp=SLTU                                  |
| XOR    | ill=0, rrf1=1, rrf2=1, wrf1=1, useALU=1, aluOp=XOR                                   |
| SRL    | ill=0, rrf1=1, rrf2=1, wrf1=1, useALU=1, aluOp=SRL                                   |
| SRA    | ill=0, rrf1=1, rrf2=1, wrf1=1, useALU=1, aluOp=SRA                                   |
| OR     | ill=0, rrf1=1, rrf2=1, wrf1=1, useALU=1, aluOp=OR                                    |
| AND    | ill=0, rrf1=1, rrf2=1, wrf1=1, useALU=1, aluOp=AND                                   |
| ADDW   | ill=0, rrf1=1, rrf2=1, wrf1=1, useALU=1, aluOp=ADDW                                  |
| SUBW   | ill=0, rrf1=1, rrf2=1, wrf1=1, useALU=1, aluOp=SUBW                                  |
| SLLW   | ill=0, rrf1=1, rrf2=1, wrf1=1, useALU=1, aluOp=SLLW                                  |
| SRLW   | ill=0, rrf1=1, rrf2=1, wrf1=1, useALU=1, aluOp=SRLW                                  |
| SRAW   | ill=0, rrf1=1, rrf2=1, wrf1=1, useALU=1, aluOp=SRAW                                  |
| ADDI   | ill=0, rrf1=1, wrf1=1, useALU=1, aluOp=ADD, needImmExt=1, immExtType=I               |
| SLLI   | ill=0, rrf1=1, wrf1=1, useALU=1, aluOp=SLL, needImmExt=1, immExtType=I               |
| SLTI   | ill=0, rrf1=1, wrf1=1, useALU=1, aluOp=SLT, needImmExt=1, immExtType=I               |
| SLTIU  | ill=0, rrf1=1, wrf1=1, useALU=1, aluOp=SLTU, needImmExt=1, immExtType=I              |
| XORI   | ill=0, rrf1=1, wrf1=1, useALU=1, aluOp=XOR, needImmExt=1, immExtType=I               |
| SRLI   | ill=0, rrf1=1, wrf1=1, useALU=1, aluOp=SRL, needImmExt=1, immExtType=I               |
| SRAI   | ill=0, rrf1=1, wrf1=1, useALU=1, aluOp=SRA, needImmExt=1, immExtType=I               |
| ORI    | ill=0, rrf1=1, wrf1=1, useALU=1, aluOp=OR, needImmExt=1, immExtType=I                |
| ANDI   | ill=0, rrf1=1, wrf1=1, useALU=1, aluOp=AND, needImmExt=1, immExtType=I               |
| ADDIW  | ill=0, rrf1=1, wrf1=1, useALU=1, aluOp=ADDW, needImmExt=1, immExtType=I              |
| SLLIW  | ill=0, rrf1=1, wrf1=1, useALU=1, aluOp=SLLW, needImmExt=1, immExtType=I              |
| SRLIW  | ill=0, rrf1=1, wrf1=1, useALU=1, aluOp=SRLW, needImmExt=1, immExtType=I              |
| SRAIW  | ill=0, rrf1=1, wrf1=1, useALU=1, aluOp=SRAW, needImmExt=1, immExtType=I              |
| LUI    | ill=0, rrf1=1, wrf1=1, useALU=1, aluOp=LUI, needImmExt=1, immExtType=U               |
| AUIPC  | ill=0, rrf1=1, wrf1=1, useALU=1, aluOp=AUIPC, needImmExt=1, immExtType=U             |
| LB     | ill=0, rrf1=1, wrf1=1, useMem=1, memOp=LB, memResOp=sExt, needImmExt=1, immExtType=I |
| LH     | ill=0, rrf1=1, wrf1=1, useMem=1, memOp=LH, memResOp=sExt, needImmExt=1, immExtType=I |
| LW     | ill=0, rrf1=1, wrf1=1, useMem=1, memOp=LW, memResOp=sExt, needImmExt=1, immExtType=I |
| LD     | ill=0, rrf1=1, wrf1=1, useMem=1, memOp=LW, memResOp=nop, needImmExt=1, immExtType=I  |
| LBU    | ill=0, rrf1=1, wrf1=1, useMem=1, memOp=LB, memResOp=uExt, needImmExt=1, immExtType=I |
| LHU    | ill=0, rrf1=1, wrf1=1, useMem=1, memOp=LH, memResOp=uExt, needImmExt=1, immExtType=I |
| LWU    | ill=0, rrf1=1, wrf1=1, useMem=1, memOp=LW, memResOp=uExt, needImmExt=1, immExtType=I |
| SB     | ill=0, rrf1=1, useMem=1, memOp=SB, memResOp=nop, needImmExt=1, immExtType=S          |
| SH     | ill=0, rrf1=1, useMem=1, memOp=SH, memResOp=nop, needImmExt=1, immExtType=S          |
| SW     | ill=0, rrf1=1, useMem=1, memOp=SW, memResOp=nop, needImmExt=1, immExtType=S          |
| SD     | ill=0, rrf1=1, useMem=1, memOp=SD, memResOp=nop, needImmExt=1, immExtType=S          |
| BEQ    | ill=0, rrf1=1, rrf2=1, branch=1                                                      |
| BNE    | ill=0, rrf1=1, rrf2=1, branch=1                                                      |
| BLT    | ill=0, rrf1=1, rrf2=1, branch=1, useALU=1, aluOp=SLT                                 |
| BGT    | ill=0, rrf1=1, rrf2=1, branch=1, useALU=1, aluOp=BGT                                 |
| BLTU   | ill=0, rrf1=1, rrf2=1, branch=1, useALU=1, aluOp=SLTU                                |
| BGTU   | ill=0, rrf1=1, rrf2=1, branch=1, useALU=1, aluOp=BGTU                                |
| JAL    | ill=0, wrf1=1, jal=1, useALU=1, aluOp=ADD, needImmExt=1, immExtType=UJ               |
| JALR   | ill=0, wrf1=1, jalr=1, useALU=1, aluOp=ADD, needImmExt=1, immExtType=I               |
| CSRRW  | ill=0, rrf1=1, useCsr=1, csrOp=rw                                                    |
| CSRRS  | ill=0, rrf1=1, useCsr=1, csrOp=rs                                                    |
| CSRRC  | ill=0, rrf1=1, useCsr=1, csrOp=rc                                                    |
| CSRRWI | ill=0, rrf1=1, useCsr=1, csrOp=rwi                                                   |
| CSRRSI | ill=0, rrf1=1, useCsr=1, csrOp=rsi                                                   |
| CSRRCI | ill=0, rrf1=1, useCsr=1, csrOp=rci                                                   |



### memOp操作类型编码

| 编码 | 类型               |
|----|------------------|
| LB | memOp = 5‘b00000 |
| LH | memOp = 5‘b00001 |
| LW | memOp = 5‘b00010 |
| LD | memOp = 5‘b00011 |
| SB | memOp = 5‘b00100 |
| SH | memOp = 5‘b00101 |
| SW | memOp = 5‘b00110 |
| SD | memOp = 5‘b00111 |

### memResOp操作类型编码

| 编码   | 类型                  |
|------|---------------------|
| nop  | memResOp = 5‘b00000 |
| sExt | memResOp = 5‘b00001 |
| uExt | memResOp = 5‘b00010 |

### csrOp操作类型编码

| 编码  | 类型               |
|-----|------------------|
| nop | csrOp = 5‘b00000 |
| rw  | csrOp = 5‘b00001 |
| rs  | csrOp = 5‘b00010 |
| rc  | csrOp = 5‘b00011 |
| rwi | csrOp = 5‘b00100 |
| rsi | csrOp = 5‘b00101 |
| rci | csrOp = 5‘b00110 |

## Verification
* 在changbai/env/verilator下创建decode文件夹，并使用verialtor作为verilog编译工具
* 要求生成可以手动执行的测试脚本
* 测试激励不少于2000cycles
* 生成可以用于gtkwave的vcd波形
* 所有的指令必须覆盖完全
# SignExt

## Introduction
本模块用于将指令中的立即数按照译码选择拓展成32/64位的擦作数

## Design
* 将knowledge/VexRiscv/src/main/scala/vexriscv/Riscv.scala中的IMM完整逻辑放入到SignExt模块
* 根据端口输入的immExtType决定当前指令是哪种指令类型
* 根据riscv-isa-manual rv64imc，对riscv指令中的立即数进行符号位拓展

### immExtType立即数类型编码

| 编码 | 类型                  |
|----|---------------------|
| S  | immExtType = 3‘b001 |
| SB | immExtType = 3‘b010 |
| U  | immExtType = 3‘b011 |
| UJ | immExtType = 3‘b100 |
| I  | immExtType = 3‘b101 |
| Z  | immExtType = 3‘b110 |

## Verification
* 在changbai/env/coco_tb下创建RVC文件夹
* 要求生成可以手动执行的测试脚本
* 测试激励不少于2000cycles
* 生成可以用于gtkwave的vcd波形
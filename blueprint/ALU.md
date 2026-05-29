# ALU

## Introduction
此模块为CPU组件中的ALU运算单元

## design
* 在路径changbai/spinal/src/main/scala/v1/下设计一个使用spinalhdl的ALU模块
* 生成的rtl放入changbai/rtl
* 在changbai/Makefile中增加makefile指令使得可以再次手动编译spinalhdl生成verilog
* ALU应实现rv32i/rv64i中基本的逻辑和加减法运算
* 按照riscv指令func3和opcode编码格式设计ALU的控制信号（alu_op）
* ALU的控制信号设计可参考knowledge/XiangShan/src/main/scala/xiangshan/package.scala
* ALU逻辑设计可参考knowledge/XiangShan/src/main/scala/xiangshan/backend/fu/Alu.scala
* 参考knowledge/riscv-isa-manual手册，实现支持rv32i/rv64i的ALU设计
* 生成详细的设计和接口文档

## verification
* 在changbai/env/coco_tb下创建ALU文件夹
* 要求生成可以手动执行的测试脚本
* 测试激励不少于2000cycles
* 生成可以用于gtkwave的vcd波形
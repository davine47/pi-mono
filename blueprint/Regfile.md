# Regfile

## Introduction
此模块用于实现所有的riscv，32个体系结构寄存器

## design
* 在路径changbai/spinal/src/main/scala/v1/下设计一个使用spinalhdl的regfile的模块
* 生成的rtl放入changbai/rtl
* 在changbai/Makefile中增加makefile指令使得可以再次手动编译spinalhdl生成verilog
* 实现riscv64中的32个体系结构寄存器，参考/changbai/knowledge/riscv-isa-manual
* 参考/changbai/knowledge/VexRiscv/src/main/scala/vexriscv/plugin/RegFilePlugin.scala
* 要考虑到此模块在超标量处理器中的使用，读写接口数量需要可配置，默认1读1写

## verification
* 在changbai/env/coco_tb下创建Regfile文件夹
* 要求生成可以手动执行的测试脚本
* 测试激励不少于2000cycles
* 生成可以用于gtkwave的vcd波形
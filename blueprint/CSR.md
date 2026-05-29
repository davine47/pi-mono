# CSR

## Introduction
此模块用于实现所有的riscv csr寄存器，并提供可以读写CSR的接口

## design
* 在路径changbai/spinal/src/main/scala/v1/下设计一个使用spinalhdl的CSR模块
* 生成的rtl放入changbai/rtl
* 在changbai/Makefile中增加makefile指令使得可以再次手动编译spinalhdl生成verilog
* 实现riscv64特权架构中的CSR，参考/changbai/knowledge/riscv-isa-manual
* 参考/changbai/knowledge/rocket-chip/src/main/scala/rocket/CSR.scala，模块接口第一版只保留CSR寄存器读和写，先不做任何其它信号的设计

## verification
* 在changbai/env/coco_tb下创建CSR文件夹
* 要求生成可以手动执行的测试脚本
* 测试激励不少于2000cycles
* 生成可以用于gtkwave的vcd波形
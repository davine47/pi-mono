# RVC

## Introduction
64-bit 取指块中可能包含 16-bit 压缩指令和 32-bit 标准指令的任意组合，32-bit指令也可能跨越两个取指块
RVC模块输入是64-bit取指块，输出是经过RVC expand后的32位riscv指令

RVC模块应包含两个模块，一个是RVCDecoder另一个是RVCExpander

## Design
* 在路径changbai/spinal/src/main/scala/v1/下设计一个使用spinalhdl的RVC的模块
* 生成的rtl放入changbai/rtl
* 在changbai/Makefile中增加makefile指令使得可以再次手动编译spinalhdl生成verilog
* 适配changbai/assistant/pi-mono/blueprint/Rw64Fetch.md的数据响应（resp）接口
* RVCDecoder可以参考刚刚生成的assistant/pi-mono/work/fetch-predecoder-design.md方案
* RVCExpander严格参考knowledge/rocket-chip/src/main/scala/rocket/RVC.scala
* 参考riscv-isa-manual中关于RVC指令的定义，实现所有RVC指令的转译
* 输入是Rw64Fetch的数据响应接口，输出是32-bit的指令，使用1bit标识是否是rvc expand指令

## verification
* 在changbai/env/coco_tb下创建RVC文件夹
* 要求生成可以手动执行的测试脚本
* 测试激励不少于2000cycles
* 生成可以用于gtkwave的vcd波形
* 所有译码场景（5种排列）必须覆盖到，包括跨指令块，数据resp连续回复等边界场景
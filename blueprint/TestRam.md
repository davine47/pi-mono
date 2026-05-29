# TestRam

## Introduction
此模块实现一个RAM，用来模拟主存功能，配合CPU进行功能验证

## Design
* 在路径changbai/spinal/src/main/scala/v1/下设计一个使用spinalhdl的TestRam模块
* 生成的rtl放入changbai/rtl
* 在changbai/Makefile中增加makefile指令使得可以再次手动编译spinalhdl生成verilog
* 参考knowledge/rocket-chip/src/main/scala/devices/tilelink/TestRAM.scala的逻辑设计
* 按照Rw64接口进行模块接口设计，参考assistant/pi-mono/work/rw64fetch-design-doc.md
* 参考已经设计好的src/main/scala/v1/rw64fetch/Rw64Fetch.scala的接口，后续要把两个模块集成在一起
* 要求以下参数可配置

| 名称       | 描述    | 
|----------|-------|
| width    | ram 一行的大小，以byte为单位，默认8bytes |
| depth    | ram深度，默认2048 |

## Verification
* 在changbai/env/coco_tb下创建TestRam文件夹
* 要求生成可以手动执行的测试脚本
* 测试激励不少于2000cycles
* 生成可以用于gtkwave的vcd波形
* 将knowledge/rocket-chip/bootrom中的Makefile和和链接脚本集成到coco_tb下的TestRam文件夹
* 使用TestRam/bootrom下的脚本编译bootrom.S，并将其bin文件读取到TestRam模块中
* 使用Rw64协议验证TestRam模块回复指令是否与bootrom.S一致
* riscv工具链在/Users/wenjunnan/riscv/bin下
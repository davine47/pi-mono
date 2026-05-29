# Gshare

## design
* 在路径changbai/spinal/src/main/scala/v1/下设计一个使用spinalhdl的Gshare分支预测器，
* 此预测器作为基础模块要求其通用性较强,接口与具体指令集实现无关
* 生成的rtl放入changbai/rtl
* 在changbai/Makefile中增加makefile指令使得可以再次手动编译spinalhdl生成verilog
* 参考changbai/knowledge/VexiiRiscv/src/main/scala/vexiiriscv/prediction/GSharePlugin.scala的设计思路
* 支持32、48、64位可配置宽度的pc，默认是64位宽度pc

## verification
* 在changbai/env/coco_tb下创建Gshare文件夹
* 要求生成可以手动执行的测试脚本
* 测试激励不少于2000cycles
* 生成可以用于gtkwave的vcd波形
* 至少30%的测试PC激励分布在0x80000000,0x10000000,0xffffffff80000100附近
# FrontendMonitor

## Introduction
此模块用于监测和统计CPU Frontend的取指密度、指令流以及Backend反馈给Frontend redirect信息，Frontend能做出反应的最大时间延迟等指标

## Design
接口

| 信号名       | 方向/位宽      | 说明   |
|-----------|------------|------|
| instValid | in/1 bit   | 当前指令有效 |
| instBits  | in/32 bits | 当前指令 |

### accGapCycles
64 bits

用来记录没有指令valid的周期数，也是两条有效指令之间的间隔cycle数的累加值

### bucketGapCycles0
64 bits

用来记录连续两条指令中间间隔为0 cycle发生的次数
### bucketGapCycles5
64 bits

用来记录连续两条指令中间间隔为大于等于1，小于等于5cycles发生的次数
### bucketGapCycles6_10
64 bits

用来记录连续两条指令中间间隔为大于等于6，小于等于10cycles发生的次数
### bucketGapCycles11_15
64 bits

用来记录连续两条指令中间间隔为大于等于11，小于等于15cycles发生的次数
### bucketGapCycles16_20
64 bits

用来记录连续两条指令中间间隔为大于等于16，小于等于20cycles发生的次数
### timeCounter
64 bits

复位后在时钟上升沿时+1


### validInstCounts
64 bits

指令有效时+1

### Verification

* 在changbai/env/coco_tb下创建FrontendMonitor文件夹
* 要求生成可以手动执行的测试脚本
* 测试激励不少于2000cycles
* 生成可以用于gtkwave的vcd波形

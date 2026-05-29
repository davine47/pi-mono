# Rw64Fetch

## Introduction
此模块用来将cpu流水线发出的标准请求转换成Rw64（64位读写）协议的请求发出

## Design
* 在路径changbai/spinal/src/main/scala/v1/下设计一个使用spinalhdl的Rw64Fetch模块，不用创建文件夹
* 生成的rtl放入changbai/rtl
* 在changbai/Makefile中增加makefile指令使得可以再次手动编译spinalhdl生成verilog
* RW64接口详情参照RW64一节
* CPU流水线请求标准参照CpuPipelineProtocol一节
* 地址位宽跟随接口可配置，默认64位
* 此模块Cpupipeline请求直接转译成RW请求

## Verification
* 在changbai/env/coco_tb下创建Rw64Fetch文件夹
* 要求生成可以手动执行的测试脚本
* 测试激励不少于2000cycles
* 生成可以用于gtkwave的vcd波形

## RW64
RW64和读写寄存器接口时序和实现相同，参考本级目录下Regfile读写接口

## CpuPipelineProtocol

| 信号名称     | 属性          | 位宽           | 说明                 |
|----------|-------------|--------------|--------------------|
| `addr`   | master req  | 可配置，默认64bits | cpu流水线发送的addr字段信息  |
| `valid`  | master req  | 1 bit        | cpu流水线发送请求的valid   |
| `ready`  | master req  | 1 bit        | cpu流水线发送请求的ready   |
| `len`    | master req  | 4 bit        | cpu流水线请求的数据长度      |
| `opcode` | master req  | 6 bit        | cpu流水线请求的操作码       |
| `valid`  | master resp | 1 bit        | cpu流水线收到回复的valid   |
| `ready`  | master resp | 1 bit        | cpu流水线收到回复的ready   |
| `data`   | master resp | 可配置，默认64bits | cpu流水线收到回复的data    |
| `msg`    | master resp | 可配置，默认8bits  | cpu流水线收到回复的message |

* msg保留不使用
* len字段代表cpu流水线请求的数据长度，如下：

| 值      | 字节数（bytes） |
|--------|------------|
| `0000` | 1          |
| `0001` | 2          |
| `0010` | 4          |
| `0011` | 8          |
| `0100` | 16         |
| `0101` | 32         |
| `0110` | 64         |
| `0111` | 128        |
| `1000` | 256        |
| `1001` | 512        |
| `1010` | 1024       |
| `1011` | 2048       |
| `1100` | 4096       |
| `1101` | 8192       |
| `1110` | 16384      |
| `1111` | 32768      |

* 目前CPU请求rw接口len长度值默认8bytes不变
* opcode代表cpu流水线请求的操作码，如下：

| 值        | 字节数（bytes）      |
|----------|-----------------|
| `000000` | 读数据请求           |
| `000001` | 写数据请求           |
| `11----` | msg有效，低三位代表消息类型 |







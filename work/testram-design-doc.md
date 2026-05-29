# TestRam 模块接口与设计文档

> 版本: 1.0 | 日期: 2026-05-15 | 模块路径: `v1.testram.TestRam`
> 依赖: SpinalHDL 1.13.0+, RW64 总线协议

---

## 1. 概述

TestRam 是一个可参数化的单端口 SRAM 仿真模块，用于模拟主存功能，配合 CPU 进行功能验证。它作为 RW64 总线协议的 **slave 端**，接收来自 CPU 侧（通过 Rw64Fetch 转换）的读写请求，返回读数据。

### 1.1 设计目标

| 目标 | 说明 |
|------|------|
| 主存仿真 | 模拟 CPU 可寻址的主存空间，支持任意地址读写 |
| RW64 协议 slave | 接收 waddr/wdata/wvalid/raddr/rvalid，返回 wready/rready/rdata/rresp |
| 可配置容量 | width（每行字节数，默认 8）和 depth（行数，默认 2048）可参数化 |
| 1 周期读延迟 | 读请求在 1 个时钟周期后返回数据，匹配典型 Block RAM 时序 |
| 0 周期写延迟 | 写操作在 valid/ready handshake 同一拍完成，无需等待 |
| 背靠背读写 | 支持连续读写，读通道有 busy 指示（rready=0）防止冲突 |
| bootROM 加载 | 可通过外部文件初始化内存内容（编译期 $readmemh 或测试期 Rw64 写入） |

### 1.2 参考

- `knowledge/rocket-chip/src/main/scala/devices/tilelink/TestRAM.scala` — TileLink 协议 RAM 设计
- `changbai/spinal/src/main/scala/v1/rw64fetch/Rw64Fetch.scala` — RW64 总线协议定义
- `assistant/pi-mono/work/rw64fetch-design-doc.md` — RW64 协议文档
- `knowledge/rocket-chip/bootrom/bootrom.S` — bootROM 汇编源码

---

## 2. 参数配置

### 2.1 TestRamConfig

```scala
case class TestRamConfig(
    width: Int = 8,    // 每行字节数（默认 8 → 64-bit）
    depth: Int = 2048  // 行数（默认 2048）
)
```

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `width` | 8 | 每行字节数。`dataWidth = width * 8` |
| `depth` | 2048 | 存储行数。`addrWidth = log2Up(depth * width)` |
| `dataWidth` | 64 | 衍生参数：数据位宽 = width × 8 |
| `addrWidth` | 14 | 衍生参数：地址位宽，`log2Up(2048 × 8) = 14` |
| `rowAddrWidth` | 11 | 衍生参数：行地址位宽，`log2Up(2048) = 11` |

### 2.2 容量计算

| width | depth | 容量 | addrWidth | dataWidth |
|-------|-------|------|-----------|-----------|
| 8 | 2048 | 16 KB | 14 | 64 |
| 8 | 4096 | 32 KB | 15 | 64 |
| 16 | 2048 | 32 KB | 15 | 128 |
| 4 | 8192 | 32 KB | 15 | 32 |

---

## 3. 接口定义

### 3.1 TestRam 核心接口（slave 侧）

TestRam 内部使用 RW64 slave 信号。信号名与 `Rw64Bus` bundle 一致，方向相反（slave 接收 master 的请求，返回响应）。

```
                   ┌─────────────┐
  waddr ──────────►│             │
  wdata ──────────►│             │
  wvalid ─────────►│             │
  wready ◄─────────│             │
                   │   TestRam   │
  raddr ──────────►│   (slave)   │
  rvalid ─────────►│             │
  rready ◄─────────│             │
                   │             │
  rdata ◄──────────│             │
  rresp ◄──────────│             │
                   └─────────────┘
```

#### 3.1.1 写通道（master → TestRam）

| 信号 | 方向 | 位宽 | 说明 |
|------|------|------|------|
| `waddr` | in | `addrWidth` (14) | 写地址（字节寻址） |
| `wdata` | in | `dataWidth` (64) | 写数据 |
| `wvalid` | in | 1 | 写请求有效 |
| `wready` | out | 1 | TestRam 准备好接收写请求（恒为 1） |

> **Handshake**: `wvalid && wready` 同时为高时，写入 `mem[waddr >> 3] <= wdata`。

#### 3.1.2 读请求通道（master → TestRam）

| 信号 | 方向 | 位宽 | 说明 |
|------|------|------|------|
| `raddr` | in | `addrWidth` (14) | 读地址（字节寻址） |
| `rvalid` | in | 1 | 读请求有效 |
| `rready` | out | 1 | TestRam 可接收新读请求（`!readPending`） |

> TestRam 一次只能处理一个未完成的读请求。当 `readPending=1` 时 `rready=0`，master 须等待。

#### 3.1.3 读响应通道（TestRam → master）

| 信号 | 方向 | 位宽 | 说明 |
|------|------|------|------|
| `rdata` | out | `dataWidth` (64) | 读回数据 |
| `rresp` | out | 1 | 读数据有效脉冲（高 1 拍） |

### 3.2 TestRamTop 顶层接口（扁平 I/O）

为 Verilog 生成方便，TestRamTop 将所有信号展平并加入 `clk`/`reset`：

| 信号 | 方向 | 位宽 | 对应 TestRam 信号 |
|------|------|------|-------------------|
| `io_clk` | in | 1 | 时钟 |
| `io_reset` | in | 1 | 异步复位（高有效） |
| `io_rw_waddr` | in | 14 | `waddr` |
| `io_rw_wdata` | in | 64 | `wdata` |
| `io_rw_wvalid` | in | 1 | `wvalid` |
| `io_rw_wready` | out | 1 | `wready` |
| `io_rw_raddr` | in | 14 | `raddr` |
| `io_rw_rvalid` | in | 1 | `rvalid` |
| `io_rw_rready` | out | 1 | `rready` |
| `io_rw_rdata` | out | 64 | `rdata` |
| `io_rw_rresp` | out | 1 | `rresp` |

---

## 4. 微架构设计

### 4.1 整体框图

```
                    ┌─────────────────────────────────┐
                    │           TestRam                │
                    │                                  │
  waddr ──────────►│─┐                                │
  wdata ──────────►│─┤──► mem.write(row, data, en)    │
  wvalid ─────────►│─┘       │                        │
  wready ◄─────────│  (always 1)                      │
                    │                                  │
  raddr ──────────►│─┐   ┌──────────┐                 │
  rvalid ─────────►│─┤──►│ read FSM │                 │
  rready ◄─────────│─┘   │(2-state) │                 │
                    │     └────┬─────┘                 │
                    │          │                       │
  rdata ◄──────────│──────────┤                       │
  rresp ◄──────────│──────────┘                       │
                    │                                  │
                    │  mem: Mem(Bits(64), 2048)        │
                    │  readAsync  combinational read   │
                    └─────────────────────────────────┘
```

### 4.2 存储阵列

```scala
val mem = Mem(Bits(dataWidth bits), depth)
```

使用 SpinalHDL `Mem` 类型，综合为 Block RAM（FPGA）或 SRAM 阵列（ASIC）。在 Verilator 仿真中展开为 `reg [63:0] mem [0:2047]`。

### 4.3 写路径

```scala
io.wready := True
mem.write(wRowAddr, io.wdata, enable = io.wvalid && io.wready)
```

- **行地址计算**: `wRowAddr = io_waddr[13:3]`（字节地址右移 3 位，即除以 8）
- **写使能**: `io.wvalid && io.wready`，使用 `Mem.write` 的 `enable` 参数直接传递，避免 SpinalHDL 生成中间组合逻辑 reg（Verilator 兼容性）
- **写延迟**: 0 周期。wvalid 拉起当拍，数据在下一个时钟沿写入 mem
- **wready**: 恒为 1，表示随时可接收写请求

### 4.4 读路径（2 状态 FSM）

```scala
val readPending = RegInit(False)       // 状态位：1=正在处理读请求
val readRowAddr = Reg(UInt(11 bits))   // 已接受的读地址（行索引）
val readDataReg  = Reg(Bits(64 bits))  // 读数据输出寄存器

io.rready := !readPending               // busy 时反压 master
io.rdata  := readDataReg
io.rresp  := readPending                // 响应脉冲 = 状态位
```

**状态转换**:

```
                    rvalid && rready
    ┌──────────┐ ──────────────────► ┌──────────┐
    │  IDLE    │                     │  BUSY    │
    │ rready=1 │                     │ rready=0 │
    │ rresp=0  │ ◄────────────────── │ rresp=1  │
    └──────────┘    (1 cycle later)  └──────────┘
                        自动返回
```

1. **IDLE** (`readPending=0`): 
   - `rready=1`，可接收新请求
   - 当 `rvalid && rready` 时：锁存读行地址 `readRowAddr <= byteAddrToRow(raddr)`，进入 BUSY
2. **BUSY** (`readPending=1`):
   - `rready=0`，拒绝新请求
   - 本拍：`readDataReg <= mem.readAsync(readRowAddr)`（组合逻辑读出，寄存到输出）
   - `rresp <= readPending = 1`（响应脉冲），自动返回 IDLE

**关键设计决策**: 使用 `mem.readAsync`（组合逻辑读）而非 `mem.readSync`（寄存器读）。这是因为 `readSync` 在每个时钟沿自动捕获 `mem[addr]`，当地址在本拍更新（非阻塞赋值）时，捕获的是**旧地址**的数据，导致读回数据错位一个地址。`readAsync` 是组合逻辑连线 `assign mem_spinal_port1 = mem[readRowAddr]`，在 BUSY 拍 `readRowAddr` 已经稳定（上一拍锁存），因此读到正确数据。

### 4.5 关键设计决策

| 决策 | 理由 |
|------|------|
| `Mem` 而非 `Vec(Reg)` | `Vec(Reg, 2048)` 展开为 2048 个独立 register + 2048 路 Mux，Verilog 超过 12000 行，编译极慢。`Mem` 生成标准 `reg [63:0] mem [0:2047]` |
| `mem.write(enable=...)` 而非 `when` | SpinalHDL 的 `when` 生成 `always @(*) reg` 中间变量，Verilator 在时钟沿前不更新该组合信号，导致写使能恒为 0 |
| `readAsync` 而非 `readSync` | `readSync` 在地址更新同一拍捕获旧地址数据，导致连续读时返回前一地址的数据 |
| 1 周期读延迟 | 匹配标准 Block RAM 的 read-first 模式，方便与 CPU 流水线集成 |
| `rready = !readPending` | 实现简单的反压机制，防止 master 在 BUSY 期间发出新读请求 |

---

## 5. 时序行为

### 5.1 写时序

```
          T0      T1      T2
clk     __/‾‾\__/‾‾\__/‾‾\__
             │       │
waddr   ─────X───────│────────   addr = A
wdata   ─────X───────│────────   data = D
wvalid  ─────┘       │
wready  ─────────────│────────   (always 1)
             │       │
             写入 mem[A>>3] ← D    (T0 上升沿)
```

- Master 在 T0 前设置 waddr/wdata，拉起 wvalid
- T0 上升沿：wvalid=1, wready=1 → `mem[wRowAddr] <= wdata`
- T0 后 deassert wvalid

### 5.2 读时序

```
          T0      T1      T2      T3
clk     __/‾‾\__/‾‾\__/‾‾\__/‾‾\__
             │       │       │
raddr   ─────X───────│─────────────   addr = A
rvalid  ─────┘       │
rready  ─────┘       └──────┐        (T0: 1→0, T2: 0→1)
             │       │       │
rdata   ─────────────X───────│───   data = mem[A>>3]
rresp   ─────────────┘       │       (1 拍脉冲)
             │       │       │
             │   readPending  │
             │   = 1 (BUSY)   │
```

- T0 上升沿: rvalid=1, rready=1 → 锁存 raddr，进入 BUSY
- T0 后: rready→0（反压）
- T1 上升沿: `readDataReg <= mem[锁存地址]`, rresp=1, 返回 IDLE
- T1 后: rready→1（可接受新请求）

### 5.3 背靠背读时序

```
          T0      T1      T2      T3      T4      T5
clk     __/‾‾\__/‾‾\__/‾‾\__/‾‾\__/‾‾\__/‾‾\__
rvalid  ──┐       ┌───────┐
raddr   ──A───────│─B─────│────────
rready  ──┘   ┌───┘   ┌───┘
rdata   ──────X──A────X──B──────
rresp   ──────┘       └────
             │   │   │   │
             读A  应  读B  应
             请求 答  请求 答
```

每隔 2 拍可发出一次读请求（1 拍请求 + 1 拍 BUSY）。

---

## 6. 与 CPU 流水线的集成

### 6.1 典型连接

```
  ┌──────────┐        ┌──────────────┐        ┌──────────┐
  │ LSU/MEM  │──req──►│              │──w──►  │          │
  │  (流水级) │◄─resp──│  Rw64Fetch   │◄─r──   │ TestRam  │
  └──────────┘        │  (协议转换)   │        │ (主存)   │
                      └──────────────┘        └──────────┘
                            CPU侧              存储侧
                         (master)            (slave)
```

- Rw64Fetch 的 `io.rw` 端口直连 TestRam 的 `io` 端口
- CPU 通过 Rw64Fetch 发出 READ(0x00)/WRITE(0x01) 操作码
- TestRam 对读写分别响应，MSG 操作不涉及 TestRam

### 6.2 地址映射

TestRam 是平坦地址空间。对上层 CPU 而言：
- 复位向量地址可映射到 TestRam 起始位置（加载 bootROM）
- 数据段、栈空间可分配至 TestRam 其他区域
- 地址范围: `0x0000 ~ 0x3FFF`（默认 16 KB）

### 6.3 bootROM 集成

bootROM 编译流程（独立于 RTL 生成）:

```bash
cd changbai/env/coco_tb/TestRam/bootrom
make                                    # 编译 bootrom.S → bootrom.img
```

生成 `bootrom.img`（128 字节），内容为 RISC-V bootROM 机器码。测试时通过 Rw64 写入 TestRam：

```python
BOOTROM_DATA = load_bootrom("bootrom/bootrom.img")
for i, word in enumerate(BOOTROM_DATA):
    await rw_write(dut, i * 8, word)
```

---

## 7. 验证

### 7.1 测试环境

```bash
# 1. 编译 bootROM
cd changbai/env/coco_tb/TestRam/bootrom && make

# 2. 生成 RTL
cd changbai && make testram

# 3. 运行测试
cd changbai/env/coco_tb/TestRam && make

# 4. 查看波形
make wave
```

### 7.2 测试清单（8/8 通过，49224ns）

| # | 测试 | 说明 | Cycles |
|---|------|------|--------|
| 1 | `test_bootrom` | 通过 Rw64 写入 bootROM 16 字，读回逐字校验 | ~52 |
| 2 | `test_write_read` | 10 个地址单次写后读，覆盖低/中/高地址段 | ~35 |
| 3 | `test_back_to_back` | 256 字连续写入，逐字回读校验 | ~770 |
| 4 | `test_raw_hazard` | 写后立即读同一地址 100 次（RAW 冒险） | ~300 |
| 5 | `test_strided` | 4 种地址跨步（8/16/32/64 字节），写入后校验 | ~210 |
| 6 | `test_data_integrity` | 16 种 64 位数据模式（全 0、全 1、交替位等） | ~50 |
| 7 | `test_read_latency` | 200 次读操作，验证延迟全部为精确 1 周期 | ~600 |
| 8 | `test_comprehensive` | 2000 次随机操作（读写混合），≥2000 cycles | 4979 |

### 7.3 测试基础设施

- **仿真器**: Verilator 5.043（默认），支持 Icarus Verilog
- **激励框架**: cocotb + Python 3.12
- **波形**: VCD 格式，`--trace --trace-structs`，可用 GTKWave 查看
- **工具链**: `riscv64-unknown-elf-gcc 15.2.0`（bootROM 编译）

### 7.4 已知限制

| 限制 | 说明 |
|------|------|
| 无字节写使能 | 写操作总是写入完整 64-bit，不支持 sub-word store |
| 无 $readmemh | 当前版本不使用 Verilog `$readmemh` 初始化（Verilator 路径问题），bootROM 通过测试代码写入 |
| 单读通道 | 一次只能处理一个未完成的读请求（`rready` 反压） |
| 无 ECC/校验 | 纯功能仿真 RAM，无错误检测与纠正 |

---

## 8. 面积估算

### 8.1 默认配置（width=8, depth=2048）

| 组件 | 估算资源 |
|------|---------|
| 存储阵列 | 2048 × 64 bit = 128 Kb Block RAM |
| 行地址解码 | ~11-bit 地址线 |
| 读 FSM | 2 个 1-bit 寄存器 + 少量组合逻辑 |
| 写使能 | 1 个 AND 门 |
| **总计** | **~1 个 128Kb BRAM + ~50 LUT/FF** |

### 8.2 扩展分析

| depth | Block RAM | 备注 |
|-------|-----------|------|
| 2048 | 1 × 128Kb | 默认配置，适合小型 SoC |
| 4096 | 2 × 128Kb 或 1 × 256Kb | 取决于 FPGA 架构 |
| 8192 | 4 × 128Kb | 可能需多个 BRAM 拼接 |
| 16384 | 8 × 128Kb | 接近中规模 FPGA BRAM 上限 |

---

## 9. RTL 生成

### 9.1 Mill 命令

```bash
# 默认配置（width=8, depth=2048）
make testram

# 自定义配置
mill -i changbaiV1.spinal.runMain v1.testram.GenTestRam 16 4096
```

### 9.2 生成文件

| 文件 | 说明 |
|------|------|
| `rtl/TestRamTop.sv` | 顶层模块（含 clk/reset，扁平 I/O） |
| `rtl/TestRam.sv` | 核心模块（RW64 slave 接口） |

---

## 10. 文件清单

| 路径 | 说明 |
|------|------|
| `changbai/spinal/src/main/scala/v1/testram/TestRam.scala` | SpinalHDL 源码 |
| `changbai/rtl/TestRam.sv` | 生成的 Verilog（核心） |
| `changbai/rtl/TestRamTop.sv` | 生成的 Verilog（顶层） |
| `changbai/env/coco_tb/TestRam/Makefile` | cocotb 测试 Makefile |
| `changbai/env/coco_tb/TestRam/test_testram.py` | cocotb Python 测试 |
| `changbai/env/coco_tb/TestRam/bootrom/` | bootROM 源码及编译脚本 |
| `changbai/Makefile` | 项目 Makefile（`testram` 目标） |

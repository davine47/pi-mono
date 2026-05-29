# Rw64Fetch 模块接口与设计文档

> 版本: 1.0 | 日期: 2026-05-13 | 模块路径: `v1.rw64fetch.Rw64Fetch`
> 依赖: SpinalHDL 1.13.0+

---

## 1. 概述

Rw64Fetch 是 CPU 流水线协议到 RW64 读写总线协议的**协议转换桥**。它将 CPU 流水线发出的标准内存访问请求（含地址、操作码、长度等字段）直接转译为 RW64 总线的读写事务。

### 1.1 设计目标

| 目标 | 说明 |
|------|------|
| 协议转换 | CPU Pipeline Protocol → RW64 Bus Protocol |
| 零延迟 | 纯组合逻辑，不引入额外时钟周期 |
| 读写分离 | 读/写通道独立，支持同时进行（分别占用不同总线通道） |
| 反压支持 | RW64 总线未就绪时，正确反压 CPU 流水线（reqReady=0） |
| 可配置位宽 | 地址位宽和数据位宽可参数化，默认 64-bit |

---

## 2. 接口定义

### 2.1 CPU Pipeline 协议 (CpuPipelineBus)

CPU 流水线侧接口，遵循 master 模式（CPU 流水线是 master，本模块是 slave）。

#### 2.1.1 请求通道 (CPU → 模块)

| 信号 | 方向 | 位宽 | 说明 |
|------|------|------|------|
| `reqAddr` | in | 可配置 (默认64) | 内存访问地址 |
| `reqValid` | in | 1 | 请求有效（CPU 流水线发起请求） |
| `reqReady` | out | 1 | 模块准备好接受请求（handshake 完成时置 1） |
| `reqLen` | in | 4 | 突发长度（当前版本单拍传输，保留用于未来扩展） |
| `reqOpcode` | in | 6 | 操作码，bit0 决定读/写：0=读, 1=写 |
| `reqWdata` | in | 可配置 (默认64) | 写数据（写操作时有效，读操作时忽略） |

> **Handshake 规则**：`reqValid && reqReady` 在同一拍为高时，请求被接受。

#### 2.1.2 响应通道 (模块 → CPU)

| 信号 | 方向 | 位宽 | 说明 |
|------|------|------|------|
| `respValid` | out | 1 | 响应有效（读操作返回数据时置 1） |
| `respReady` | in | 1 | CPU 流水线准备好接收响应 |
| `respData` | out | 可配置 (默认64) | 读回的数据 |
| `respMsg` | out | 8 | 消息/状态码（当前保留为 0） |

> 写操作无响应：写请求在 `reqValid && reqReady` 时即完成。

### 2.2 RW64 总线协议 (Rw64Bus)

模块对外（如 L1 缓存、总线矩阵）一侧接口，模块是 master。

#### 2.2.1 写通道 (模块 → RW64)

| 信号 | 方向 | 位宽 | 说明 |
|------|------|------|------|
| `waddr` | out | 可配置 | 写地址 |
| `wdata` | out | 可配置 | 写数据 |
| `wvalid` | out | 1 | 写请求有效 |
| `wready` | in | 1 | RW64 总线准备好接收写请求 |

#### 2.2.2 读请求通道 (模块 → RW64)

| 信号 | 方向 | 位宽 | 说明 |
|------|------|------|------|
| `raddr` | out | 可配置 | 读地址 |
| `rvalid` | out | 1 | 读请求有效 |
| `rready` | in | 1 | RW64 总线准备好接收读请求 |

#### 2.2.3 读响应通道 (RW64 → 模块)

| 信号 | 方向 | 位宽 | 说明 |
|------|------|------|------|
| `rdata` | in | 可配置 | 读回的数据 |
| `rresp` | in | 1 | 读响应有效脉冲（高 1 拍表示数据就绪） |

---

## 3. 微架构设计

### 3.1 整体框图

```
  ┌────────────── CPU Pipeline ──────────────┐
  │                                           │
  │  reqAddr ──────┐                          │
  │  reqValid ─────┤                          │
  │  reqOpcode ────┤    ┌──────────────┐      │
  │  reqWdata ─────┤    │              │      │
  │                ├───►│  Rw64Fetch   │      │
  │  respValid ◄───┤    │  (组合逻辑)   │      │
  │  respData  ◄───┤    │              │      │
  │  respMsg   ◄───┤    └──────┬───────┘      │
  │                │           │               │
  └────────────────┘           │               │
                               ▼               │
  ┌────────────── RW64 Bus ───────────────────┐
  │                                           │
  │          waddr, wdata, wvalid ──►         │
  │          raddr, rvalid        ──►         │
  │          ◄── wready, rready              │
  │          ◄── rdata, rresp                 │
  │                                           │
  └───────────────────────────────────────────┘
```

### 3.2 数据路径

```
                    opcode[0]=0 (读)
  reqAddr ───────────────────────► raddr
  reqValid ──────────────────────► rvalid  (当 opcode[0]=0)
  rready  ───────────────────────► reqReady (当 opcode[0]=0)
  rdata   ───────────────────────► respData
  rresp   ───────────────────────► respValid

                    opcode[0]=1 (写)
  reqAddr ───────────────────────► waddr
  reqWdata ──────────────────────► wdata
  reqValid ──────────────────────► wvalid  (当 opcode[0]=1)
  wready  ───────────────────────► reqReady (当 opcode[0]=1)
```

### 3.3 核心逻辑 (Scala)

```scala
val isRead  = !io.cpu.reqOpcode(0)   // bit0=0 → read
val isWrite = io.cpu.reqOpcode(0)     // bit0=1 → write

// Write channel
io.rw.waddr  := io.cpu.reqAddr
io.rw.wdata  := io.cpu.reqWdata
io.rw.wvalid := io.cpu.reqValid && isWrite

// Read channel
io.rw.raddr  := io.cpu.reqAddr
io.rw.rvalid := io.cpu.reqValid && isRead

// CPU ready: 取决于操作类型，选择对应通道的 ready 信号
io.cpu.reqReady := Mux(isWrite, io.rw.wready, io.rw.rready)

// CPU response: 直连 RW64 读响应
io.cpu.respValid := io.rw.rresp
io.cpu.respData  := io.rw.rdata
io.cpu.respMsg   := 0   // 保留
```

### 3.4 关键设计决策

| 决策 | 理由 |
|------|------|
| 纯组合逻辑 | 协议转换无需状态，面积最小，零延迟 |
| `opcode[0]` 区分读写 | 1-bit 即可区分，其余 opcode 位预留扩展 |
| wdata 放在请求侧 | 写数据与地址同属请求，与标准总线协议一致（AXI/AHB） |
| 写操作无响应 | 简化协议，写完成由 wready handshake 确认 |
| 读响应独立 rresp 脉冲 | 与 Regfile 读接口风格一致，简洁清晰 |

---

## 4. 时序行为

### 4.1 写操作时序

```
         T0      T1      T2
clk    __/‾‾\__/‾‾\__/‾‾\__
            │       │
reqValid  ──┘       │   CPU 发起写请求
reqOpcode ──┘       │   opcode[0]=1
reqAddr   ──┘       │   addr = A
reqWdata  ──┘       │   wdata = D
            │       │
wvalid    ──┘       │   模块驱动 wvalid=1
wready    ──┘       │   RW64 总线就绪
            │       │
reqReady  ──┘       │   handshake 完成 → CPU 请求被接受
            │       │
            T0 上升沿: wvalid=1 && wready=1 → 写事务完成
```

### 4.2 读操作时序

```
         T0      T1      T2      T3
clk    __/‾‾\__/‾‾\__/‾‾\__/‾‾\__
            │       │       │
reqValid  ──┘       │       │   CPU 发起读请求
reqOpcode ──┘       │       │   opcode[0]=0
reqAddr   ──┘       │       │   addr = A
            │       │       │
rvalid    ──┘       │       │   模块驱动 rvalid=1
rready    ──┘       │       │   RW64 总线就绪
            │       │       │
reqReady  ──┘       │       │   handshake 完成 → raddr 被接受
            │               │
rdata     ──────────┘       │   RW64 返回数据 (T1 上升沿后有效)
rresp     ──────────┘       │   脉冲 1 拍
            │               │
respValid ──────────┘       │   模块转发给 CPU 流水线
respData  ──────────┘       │   data = rdata
```

### 4.3 反压时序 (RW64 未就绪)

```
         T0      T1      T2      T3      T4
clk    __/‾‾\__/‾‾\__/‾‾\__/‾‾\__/‾‾\__
            │       │       │       │
reqValid  ──┘                       │   CPU 持续等待
wvalid    ──┘                       │   wvalid=1 (一直保持)
wready    ──────────────────────────┘   T3 才就绪
reqReady  ──────────────────────────┘   T3 handshake 完成
            │       │       │       │
            请求在 T0-T3 期间被阻塞
            CPU 流水线看到 reqReady=0 → 流水线停顿
```

---

## 5. 验证

### 5.1 测试环境

```bash
cd changbai/env/coco_tb/Rw64Fetch
CHANGBAI_ROOT=/path/to/changbai make
make wave   # gtkwave 查看波形
```

### 5.2 测试清单 (10/10 通过, 27799ns)

| 测试 | Cycles | 验证内容 |
|------|--------|---------|
| test_single_write | 5 | 单次写：addr/wdata 透传到 waddr/wdata，wvalid/wready handshake |
| test_single_read | 7 | 单次读：raddr/rvalid 发出 → rdata/rresp 返回 → CPU resp |
| test_back_to_back_writes | 55 | 50 次连续写，验证无数据错乱 |
| test_back_to_back_reads | 65 | 30 次连续读，每次验证返回数据正确 |
| test_interleaved_rw | 50 | 30 对读写交替，验证 opcode 切换无毛刺 |
| test_backpressure | 14 | 反压：wready=0 时 reqReady=0；rready=0 时 reqReady=0，恢复后正常 |
| test_write_patterns | 12 | 7 种位模式（全0/全1/交替/边界位等）写数据验证 |
| test_read_response_patterns | 15 | 5 种响应数据模式验证 |
| test_comprehensive | 2251 | 500 写 + 500 读 + 400 交替 + 填充至 >2000 cycles |
| test_concurrent_rw | 305 | 100 对紧密交替读写 |

---

## 6. 在 CPU 流水线中的集成

### 6.1 典型连接

```
  ┌──────────┐        ┌──────────────┐        ┌──────────┐
  │ LSU/MEM  │──req──►│              │──w──►  │          │
  │  (流水级) │◄─resp──│  Rw64Fetch   │◄─r──   │ L1 D$    │
  └──────────┘        │              │        │ / Bus    │
                      └──────────────┘        └──────────┘
```

### 6.2 opcode 编码约定

| opcode[5:0] | 操作 | bit0 | 说明 |
|-------------|------|------|------|
| `000000` | Load | — | 读操作 |
| `000001` | Store | — | 写操作 |
| `110000`-`111111` | MSG | — | 消息传递，高2位=11，低4位为消息类型（0-15），无总线事务 |

> 当前版本仅通过 bit0 区分读写，其余位透传（未在模块内使用）。

### 6.3 扩展可能

| 扩展方向 | 说明 |
|----------|------|
| 突发传输 | 利用 `reqLen` 字段支持多拍读写，需增加内部状态机 |
| 原子操作 | 利用 opcode 高 5 位编码 AMO 类型，通过 RW64 总线仲裁 |
| Fence/I/O | 利用 opcode 编码区分普通内存访问和 I/O 访问 |
| 非对齐访问 | 在模块内增加地址拆分逻辑，将非对齐请求拆为多次 RW64 事务 |

---

## 7. 面积估算 (64-bit 地址/数据)

| 组件 | 估算门数 |
|------|---------|
| 地址/数据直连通路 | ~200 gates |
| opcode 译码 (1-bit) | ~5 gates |
| Mux (read vs write ready) | ~10 gates |
| **总计** | **~215 gates** |

> 纯组合逻辑，面积极小。综合后可能被优化为若干连线 + 1 个 2:1 Mux。

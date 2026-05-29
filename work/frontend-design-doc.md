# Frontend 模块设计文档与时序分析

> 版本: 1.2 | 日期: 2026-05-18 | 模块路径: `v1.Frontend`
> 依赖: SpinalHDL 1.13.0+, `Rw64Fetch`, `RVCDecoder`, `InstQueue`, `RVCExpander`

---

## 1. 概述

Frontend 是 CPU 前端取指模块，负责将 CPU 的取指请求通过 RW64 总线发送到主存，接收响应数据后进行指令边界扫描、压缩指令展开、指令缓冲，最终逐条输出 32-bit 指令。同时维护 `nextPc` 寄存器追踪下一条指令地址。

### 1.1 设计目标

| 目标 | 说明 |
|------|------|
| 取指 | CPU 请求 → Rw64Fetch → RW64 总线 → TestRam |
| 指令边界扫描 | RVCDecoder 分析 64-bit 取指块中的指令边界（最多 4 条） |
| 跨块恢复 | carryOut/carryIn 寄存器处理 32-bit 指令跨 64-bit 块边界 |
| 指令缓冲 | InstQueue 将 0~4 条/周期的输入缓冲为 1 条/周期的输出 |
| 压缩展开 | RVCExpander（纯组合）将 16-bit 压缩指令展开为 32-bit |
| PC 追踪 | nextPc 寄存器：32-bit +4，16-bit 压缩 +2，flush 清零 |
| 反压支持 | Rw64Fetch handshake + InstQueue backpressure |

### 1.2 内部模块

```
  ┌──────────────────────────────────────────────────────────────┐
  │                        Frontend                              │
  │                                                              │
  │  cpu_req* ──► Rw64Fetch ──► RW64 bus (对外)                  │
  │                   │                                          │
  │             respData/respValid                               │
  │                   │                                          │
  │              ┌────▼────────────────────────────┐              │
  │              │  validReg (RegNext)             │              │
  │              │  carryReg / hasCarryReg (反馈)   │              │
  │              └────┬────────────────────────────┘              │
  │                   │                                          │
  │              ┌────▼────┐    ┌───────────┐    ┌──────────┐   │
  │              │RVCDecoder│───►│ InstQueue │───►│RVCExpander│  │
  │              └──────────┘    │ (depth=16)│    │ (组合)   │   │
  │                              └─────┬─────┘    └────┬─────┘   │
  │                                    │               │         │
  │                              instValid/instBits/isRVC        │
  │                                    │                         │
  │                              ┌─────▼─────┐                   │
  │                              │  nextPcReg │                   │
  │                              │  +2/+4     │                   │
  │                              └───────────┘                   │
  └──────────────────────────────────────────────────────────────┘
```

---

## 2. 接口定义

### 2.1 时钟与复位

| 信号 | 方向 | 位宽 | 说明 |
|------|------|------|------|
| `io_clk` | in | 1 | 时钟 |
| `io_reset` | in | 1 | 异步复位（高有效） |

### 2.2 CPU 请求接口

| 信号 | 方向 | 位宽 | 说明 |
|------|------|------|------|
| `io_cpu_reqAddr` | in | 64 | 取指地址（字节寻址） |
| `io_cpu_reqValid` | in | 1 | 请求有效 |
| `io_cpu_reqReady` | out | 1 | Frontend 可接受新请求 |
| `io_cpu_reqOpcode` | in | 6 | 操作码（通常 READ=0x00） |
| `io_cpu_reqWdata` | in | 64 | 写数据（取指时忽略） |
| `io_cpu_reqLen` | in | 4 | 长度编码（取指 8B=0b0011） |

> Handshake: `reqValid && reqReady` 同时为高时请求被接受。

### 2.3 指令输出接口

| 信号 | 方向 | 位宽 | 说明 |
|------|------|------|------|
| `io_instValid` | out | 1 | 指令有效（高时 instBits/isRVC 有效） |
| `io_instBits` | out | 32 | 32-bit 指令（16-bit 展开后） |
| `io_isRVC` | out | 1 | 原始指令为压缩指令 |
| `io_nextPc` | out | 64 | 下一条指令地址（每条指令后更新） |
| `io_flush` | in | 1 | 清空指令队列 + 重置 PC |

### 2.4 RW64 总线接口

| 信号 | 方向 | 位宽 | 说明 |
|------|------|------|------|
| `io_rw_waddr` | out | 64 | 写地址 |
| `io_rw_wdata` | out | 64 | 写数据 |
| `io_rw_wvalid` | out | 1 | 写有效 |
| `io_rw_wready` | in | 1 | 写就绪 |
| `io_rw_raddr` | out | 64 | 读地址 |
| `io_rw_rvalid` | out | 1 | 读有效 |
| `io_rw_rready` | in | 1 | 读就绪 |
| `io_rw_rdata` | in | 64 | 读数据 |
| `io_rw_rresp` | in | 1 | 读响应脉冲 |

---

## 3. 数据路径与时序

### 3.1 取指流水线

```
时钟周期:  T0    T1    T2    T3    T4    T5    T6    T7
          │     │     │     │     │     │     │     │
cpu_req   ├─req─┤     │     │     │     │     │     │  发起读 A0
rw64      │     ├─rreq►     │     │     │     │     │  raddr=A0
TestRam   │     │     ├─resp►     │     │     │     │  rdata/rresp
validReg  │     │     │     ├─1──►     │     │     │  RegNext 延迟
RVCDecoder│     │     │     │  scan     │     │     │  指令边界
InstQueue │     │     │     │  buf  │     │     │     │  入队
output    │     │     │     │     ├─I0─►├─I1─►├─I2─►  逐条输出
nextPc    │  0  │  0  │  0  │  0  │ +N  │ +N  │ +N    每条指令递增
```

**关键延迟**:

| 阶段 | 延迟 | 说明 |
|------|------|------|
| CPU 请求 → RW64 总线 | 0 周期 | Rw64Fetch 纯组合 |
| TestRam 读延迟 | 1 周期 | `rvalid/rready` → `rresp/rdata` |
| respValid → validReg | 1 周期 | `RegNext` 寄存单拍脉冲 |
| validReg → RVCDecoder | 0 周期 | 组合逻辑 |
| RVCDecoder → InstQueue 入队 | 1 周期 | Buffer + Push FSM |
| InstQueue 出队 | 可变 | 深度 16，约 2-3 周期/条 |
| RVCExpander | 0 周期 | 纯组合 |

**端到端延迟**: 取指请求 → 第一条指令输出 ≈ 4-5 周期

### 3.2 validReg 寄存的必要性

```
            T0        T1        T2
rresp     ───┐         │         │   单拍脉冲
               ├────────┤         │
respValid ───┐ │         │         │  组合跟随 rresp
               ├────────┤         │
validReg    ───┼────────┼───┐     │  RegNext 延迟 1 拍
               │         │   ├─────┤
RVCDecoder  ───┼────────┼───┤ scan │  validReg=1 时输出
               │         │   │     │
InstQueue   ───┼────────┼───┤ buf │  clk 沿采样 RVCDecoder
               │         │   │     │
```

- `rresp` 是单周期脉冲 → `respValid` 也是单周期脉冲
- InstQueue 在时钟沿采样 `instXValid`，若 `rresp` 与 InstQueue 时钟沿同时变化，可能错过
- `RegNext(respValid)` 将脉冲延长为 1 个完整周期，确保 InstQueue 稳定采样

### 3.3 Carry 跨块流水线

```
Chunk N                         Chunk N+1
  │                               │
  ├─ validReg=1                   ├─ validReg=1
  ├─ RVCDecoder 扫描              ├─ hasCarryIn=1 (来自上一块)
  ├─ hasCarryOut=1 ?              ├─ carryIn = carryOut_N
  ├─ carryOut = hw4               ├─ RVCDecoder 从 slot0 开始
  │   │                           │   正确恢复跨块 32-bit 指令
  │   └──► carryReg   锁存
  │        hasCarryReg 锁存        │
  │                               │
  └─ flush ? → 清零               └─ flush ? → 清零
```

### 3.4 nextPc 更新时序

```
            T0        T1        T2
instValid  ───┐         │         │  队列输出有效
               ├────────┤         │
nextPc     ──0─┼────────┼───4─────┤  T1 上升沿: 0+4=4 (NB)
               │         │         │  T1+delta: nextPc=4
读端口     ────┼────────┼─────────┤  外部在 T1+delta 读到 4
```

`nextPc` 更新与 `instValid` 在同一拍（非阻塞赋值），外部读取时比 `instBits` 晚一拍有效（因 NB 未提交）。TestHarness 中通过 `lastFetchPc` 锁存避免重复请求。

---

## 4. 各子模块设计要点

### 4.1 Rw64Fetch（组合逻辑）

- CPU 请求 → RW64 总线协议转换
- 读通道: `raddr = reqAddr`, `rvalid = reqValid && isRead`
- 写通道: `waddr = reqAddr`, `wvalid = reqValid && isWrite`
- `reqReady` 取决于 `rready`/`wready`

### 4.2 RVCDecoder（组合逻辑）

- 64-bit 取指块 → 指令边界扫描
- `valid` 门控输出（valid=0 时全部输出为 0）
- carryOut/hasCarryOut 标识跨块

### 4.3 InstQueue（时序逻辑，含 clk/reset）

- 4 条/周期入队 → 1 条/周期出队
- 内部: 4-entry Buffer → Push FSM → StreamFifo(16)
- `flush` 清空全部状态
- `allEmpty` 检测防止 pushIdx 残留

### 4.4 RVCExpander（纯组合逻辑）

- 16-bit → 32-bit 压缩指令展开
- 无时钟/无寄存器，`always @(*)`
- 覆盖 RV64IMC 全部合法指令
- `ill` 标识非法编码

### 4.5 nextPc 寄存器

```scala
val nextPcReg = Reg(UInt(64 bits)) init 0
when(io.flush) {
  nextPcReg := 0
}.elsewhen(queue.io.instValid) {
  nextPcReg := nextPcReg + Mux(queue.io.isRVC, U(2), U(4))
}
```

---

## 5. 集成方式

### 5.1 直连 TestRam（TestHarness）

```
Frontend.rw (master)  ←→  TestRam.rw (slave, 14-bit addr truncated)
Frontend.cpu_req*     ←   Fetch FSM (lastFetchPc 锁存, booted 启动)
Frontend.inst*/nextPc →   外部监控
```

### 5.2 外部 CPU 驱动

```
CPU 流水线 → Frontend.cpu_req* (handshake)
           ← Frontend.inst*/nextPc
```

外部需实现类似 TestHarness 的 Fetch FSM，避免同一地址重复请求。

### 5.3 关键约束

| 约束 | 说明 |
|------|------|
| `cpu_reqValid` 不应恒为 1 | 需用 FSM 在 `nextPc` 变化时才发起新请求 |
| `flush` 至少 1 拍 | 确保 carry/queue/nextPc 全部清零 |
| RW64 地址截断 | Frontend 64-bit → TestRam `addr[13:0]` |
| `cpu_reqLen = 0b0011` | 取指固定 8 字节 |

---

## 6. 测试

### 6.1 测试环境

```bash
cd changbai/env/coco_tb/Frontend
CHANGBAI_ROOT=../../.. make
```

### 6.2 测试清单

| # | 测试 | 说明 | 结果 |
|---|------|------|------|
| 1 | `test_valid_gating` | valid=0 → 输出全 0 | PASS |
| 2 | `test_fetch_decode` | 2×32-bit 取指 → 解码 | PASS |
| 3 | `test_flush` | flush 清空流水线 | PASS |
| 4 | `test_comprehensive` | 500 次随机取指，2002 cycles | PASS |
| 5 | `test_bootrom` (Harness) | bootROM $readmemh → 自动取指解码，100 条指令 | PASS |

---

## 7. 文件清单

| 路径 | 说明 |
|------|------|
| `changbai/spinal/src/main/scala/v1/Frontend.scala` | Frontend 源码 |
| `changbai/spinal/src/main/scala/v1/TestHarness.scala` | 集成 + Fetch FSM |
| `changbai/rtl/Frontend.sv` | 生成 Verilog |
| `changbai/rtl/TestHarness.sv` | 生成 Verilog |
| `changbai/env/coco_tb/Frontend/` | Frontend 测试 |
| `changbai/env/coco_tb/TestHarness/` | TestHarness 测试 |
| `changbai/assistant/pi-mono/work/frontend-design-doc.md` | 本文档 |

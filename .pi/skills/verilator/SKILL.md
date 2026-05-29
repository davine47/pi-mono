---
name: verilator
description: Verilator - 开源高性能 Verilog/SystemVerilog 仿真编译器。将 RTL 编译为 C++/SystemC 多线程模型，支持波形追踪、覆盖率分析、性能剖析。用于 RTL 仿真验证、lint 检查、波形调试、性能优化。
---

# Verilator

Verilator 将 Verilog/SystemVerilog 设计编译为 C++ 或 SystemC 模型，然后通过 C++ 编译器生成可执行仿真程序。它不是一个传统的事件驱动仿真器，而是一个编译器。

## 快速开始

### 安装

```bash
# Ubuntu 包管理器
sudo apt-get install verilator

# macOS
brew install verilator

# 从源码构建
git clone https://github.com/verilator/verilator
cd verilator
autoconf && ./configure && make -j $(nproc) && sudo make install
```

### 最简示例：Bin 模式

```bash
cat >our.v <<'EOF'
  module our;
     initial begin $display("Hello World"); $finish; end
  endmodule
EOF

# 一键编译+运行
verilator --binary -j 0 -Wall our.v
obj_dir/Vour
# 输出: Hello World
```

`--binary` 等价于 `--main --exe --build --timing`，是完整流程的一步到位选项。

### C++ 模式（自定义 testbench）

```bash
cat >sim_main.cpp <<'EOF'
  #include "Vour.h"
  #include "verilated.h"
  int main(int argc, char** argv) {
      VerilatedContext* contextp = new VerilatedContext;
      contextp->commandArgs(argc, argv);
      Vour* top = new Vour{contextp};
      while (!contextp->gotFinish()) { top->eval(); }
      delete top;
      delete contextp;
      return 0;
  }
EOF

verilator --cc --exe --build -j 0 -Wall sim_main.cpp our.v
obj_dir/Vour
```

## 核心工作模式

| 模式 | 选项 | 说明 |
|------|------|------|
| 生成可执行二进制 | `--binary` | 等价于 `--main --exe --build --timing` |
| 生成 C++ 源码 | `--cc` | 输出 C++ 文件，需自己写 wrapper 和 Makefile |
| 生成 SystemC 源码 | `--sc` | 输出 SystemC 文件 |
| 仅 Lint 检查 | `--lint-only` | 只检查警告，不生成输出文件 |
| 仅 JSON 输出 | `--json-only` | 生成 JSON 供其他工具使用 |
| 仅预处理 | `-E` | 预处理后输出到 stdout |

## 常用选项速查

### 输出控制

| 选项 | 说明 |
|------|------|
| `--Mdir <dir>` | 输出目录，默认 `obj_dir` |
| `--prefix <name>` | 输出文件前缀，默认为顶层模块名 |
| `--top-module <name>` | 指定顶层模块 |
| `-y <dir>` | 添加模块搜索目录 |
| `+incdir+<dir>` | 添加 include 搜索目录 |

### Lint 与警告

| 选项 | 说明 |
|------|------|
| `-Wall` | 启用额外 lint 警告（推荐） |
| `-Wno-<warning>` | 关闭特定警告，如 `-Wno-WIDTH` |
| `-Werror-<warning>` | 将特定警告升级为错误 |
| `--lint-only` | 仅做 lint 检查 |
| `--no-assert` | 禁用所有断言检查（提升性能） |

### 性能优化

| 选项 | 说明 |
|------|------|
| `-O3` | 最高优化级别 |
| `--x-assign fast` | 优化 X 赋值（可能有复位风险） |
| `--x-initial fast` | 优化 X 初始化 |
| `--output-split <n>` | 将输出拆分为 n 个 .cpp 文件，加速编译 |
| `--output-split-cfuncs <n>` | 按函数数拆分输出文件 |
| `--threads <n>` | 启用 n 线程多线程仿真 |

### 调试与分析

| 选项 | 说明 |
|------|------|
| `--trace` | 启用 VCD 波形追踪 |
| `--trace-fst` | 启用 FST 波形追踪（推荐，压缩率高） |
| `--coverage` | 启用所有覆盖率（line/toggle/fsm/expr/user） |
| `--coverage-line` | 仅行覆盖率 |
| `--coverage-toggle` | 仅翻转覆盖率 |
| `--prof-cfuncs` | 启用 C++ 函数性能剖析 |
| `--prof-exec` | 启用执行性能剖析（配合 verilator_gantt） |
| `--savable` | 启用仿真状态保存/恢复 |

### 编译控制

| 选项 | 说明 |
|------|------|
| `--build` | Verilate 后自动调用 make 编译 |
| `--exe` | 生成可执行文件（需提供 C++ wrapper） |
| `--main` | 自动生成 main 函数 |
| `-j <n>` | 并行度，`0` 表示使用所有 CPU 核心 |
| `--build-jobs <n>` | 编译阶段并行度 |
| `-CFLAGS <flags>` | 传递额外 C++ 编译选项 |
| `-LDFLAGS <flags>` | 传递额外链接选项 |

### SystemVerilog 语言支持

| 选项 | 说明 |
|------|------|
| `--language <lang>` | 指定语言标准（1364-2001/2005, 1800-2005/2012/2017/2023） |
| `--default-language <lang>` | 默认语言标准 |
| `+1364-2005ext+.v` | 指定 .v 文件按 Verilog 2005 解析 |

## 连接到 Verilated 模型

生成的模型类 `V{top}` 暴露以下接口：

```cpp
#include "Vtop.h"
#include "verilated.h"

// 创建上下文和顶层模块
VerilatedContext* contextp = new VerilatedContext;
contextp->commandArgs(argc, argv);
Vtop* top = new Vtop{contextp};

// 访问顶层 IO 端口（直接读写）
top->io_input = value;
int result = top->io_output;

// 仿真循环
while (!contextp->gotFinish()) {
    top->eval();                    // 评估组合逻辑
    top->clk ^= 1;                  // 翻转时钟
    top->eval();
}

// 访问内部信号（使用 rootp 指针）
#include "Vtop___024root.h"
top->rootp->internal_signal = value;

// 波形追踪
contextp->traceEverOn(true);         // 启用追踪
VerilatedVcdC* tfp = new VerilatedVcdC;
top->trace(tfp, 99);                 // 追踪 99 层
tfp->open("waveform.vcd");
// ... 仿真循环中 ...
tfp->dump(contextp->time());         // 每个时间步 dump
contextp->timeInc(1);
tfp->close();
```

## 多线程仿真

```bash
verilator --threads 4 --cc --exe --build sim_main.cpp top.v
```

关键注意事项：
- eval 线程和构造模型的线程必须是同一个
- 多核绑定建议使用 `numactl` 控制
- 可通过环境变量 `VERILATOR_NUMA_STRATEGY=none` 禁用自动 NUMA 绑定
- VPI 调用只能在主线程中进行

```bash
# 手动绑定到指定物理核心
numactl -m 0 -C 0,1,2,3 -- obj_dir/Vtop
```

## 覆盖率和分析

### 波形查看

```bash
# 生成 FST 波形（推荐）
verilator --trace-fst --cc --exe --build sim_main.cpp top.v
obj_dir/Vtop +verilator+trace+fst+file+wave.fst
gtkwave wave.fst
```

### 覆盖率收集

```bash
# 编译时启用覆盖率
verilator --coverage --cc --exe --build sim_main.cpp top.v

# 运行仿真，生成 coverage.dat
obj_dir/Vtop +verilator+coverage+file+logs/coverage.dat

# 分析和注释源文件
verilator_coverage --annotate logs/annotated logs/coverage.dat
```

### 性能剖析

```bash
# 执行剖析
verilator --prof-exec --cc --exe --build sim_main.cpp top.v
obj_dir/Vtop +verilator+prof+exec+file+prof.dat
verilator_gantt prof.dat

# 代码级剖析
verilator --prof-cfuncs --cc --exe --build sim_main.cpp top.v
obj_dir/Vtop  # 产出 gmon.out
gprof gmon.out > gprof.log
verilator_profcfunc gprof.log > profcfunc.log
```

## 层次化 Verilation（大设计）

适用于大型 SoC 设计（编译时间 >10 分钟或内存 >100GB）：

1. 在 HDL 中用 metacomment 标记层次块：
   ```systemverilog
   module cpu(/* ... */);
      /*verilator hier_block*/
   endmodule
   ```

2. 用 `--hierarchical` 选项编译：
   ```bash
   verilator --hierarchical --cc --exe --build sim_main.cpp top.v
   ```

Verilator 自动将层次块编译为独立库，顶层自动调用子层模型。支持嵌套层次块和 `-j N` 并行 Verilation。

## 项目中的典型用法（Changbai 项目）

### 对 Chisel/SpinalHDL 生成的 Verilog 进行仿真

```bash
# SpinalHDL 生成 Verilog 后仿真
verilator --binary -j 0 -Wall \
    -y ./rtl \
    --top-module MyCpuTop \
    ./rtl/MyCpuTop.v

# 带波形的仿真
verilator --cc --exe --build -j 0 -Wall \
    --trace-fst \
    -y ./rtl \
    --top-module MyCpuTop \
    sim_main.cpp ./rtl/MyCpuTop.v
```

### 使用 Makefile 增量编译

Verilator 自动生成 `V{top}.mk`，支持增量编译：

```bash
# 首次编译
make -C obj_dir -f Vtop.mk Vtop

# 增量编译（仅编译变更部分）
make -C obj_dir -f Vtop.mk Vtop
```

### 运行时参数

| 参数 | 说明 |
|------|------|
| `+verilator+debug` | 启用运行时调试输出 |
| `+verilator+quiet` | 禁用仿真报告 |
| `+verilator+rand+reset+<n>` | 设置随机种子 |
| `+verilator+seed+<n>` | 设置随机种子（同 rand+reset） |
| `+verilator+coverage+file+<name>` | 覆盖率输出文件 |
| `+verilator+prof+exec+file+<name>` | 执行剖析输出文件 |
| `+verilator+trace+vcd+file+<name>` | VCD 波形输出文件 |
| `+verilator+trace+fst+file+<name>` | FST 波形输出文件 |

## 环境变量

| 变量 | 说明 |
|------|------|
| `VERILATOR_ROOT` | Verilator 安装根目录 |
| `VERILATOR_SOLVER` | 约束求解器（如 z3） |
| `VERILATOR_NUMA_STRATEGY` | NUMA 线程绑定策略：`default`/`none` |
| `SYSTEMC_INCLUDE` | SystemC 头文件目录 |
| `SYSTEMC_LIBDIR` | SystemC 库目录 |
| `OBJCACHE` | C++ 编译器缓存（如 ccache） |

## 性能调优建议

1. 确保无 `UNOPTFLAT` 警告（修复可获高达 60% 性能提升）
2. 使用 `-O3 --x-assign fast --x-initial fast --no-assert` 获得最佳性能
3. 用最新 Clang（比 GCC 快约 10%）并静态链接
4. 对大设计使用 `--output-split` 加速并行编译
5. 编译器 PGO：先 `-fprofile-generate` 运行，再 `-fprofile-use` 重编译
6. 使用 `numactl` 手动绑定线程到独立物理核心
7. 设置 `OPT_FAST="-O2 -march=native"` 进一步优化

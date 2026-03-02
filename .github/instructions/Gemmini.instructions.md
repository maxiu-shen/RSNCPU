---
applyTo: '**'
---
# Gemmini 架构参考

在回答与 Gemmini、脉动阵列、DNN 加速器、RISC-V 自定义指令、RoCC 接口或 RSNCPU 项目设计相关的问题时，请始终参考 `doc/self_doc/README.md` 中的 Gemmini 文档。

## Gemmini 核心概念

### 架构概述
- Gemmini 是一个使用非标准 RISC-V 自定义指令的 **RoCC 加速器**。
- 核心：执行矩阵乘法的**脉动阵列**，同时支持**输出固定（output-stationary）**和**权重固定（weight-stationary）**数据流。
- 数据存储在显式管理的 **Scratchpad**（分组 SRAM）和**累加器（Accumulator）**（带加法器单元）中。
- **DMA 引擎**负责在主存和 Scratchpad 之间传输数据。
- Gemmini 采用**解耦访问/执行（decoupled access/execute）**架构，包含三个控制器：`ExecuteController`、`LoadController`、`StoreController`。
- **ROB（重排序缓冲区）**用于检测不同控制器之间指令的冒险。

### ISA 指令集概要
- **数据搬移**：`mvin`（DRAM → Scratchpad），`mvout`（Scratchpad → DRAM）。三种 mvin 变体（`mvin`、`mvin2`、`mvin3`）用于重叠加载。
- **配置指令**：`config_ex`（执行流水线）、`config_mvin`（加载流水线）、`config_mvout`（存储流水线）、`config_norm`（归一化）、`flush`（TLB 刷新）。
- **核心矩阵乘法**：`matmul.preload` + `matmul.compute.preloaded` / `matmul.compute.accumulated`。计算 C = A * B + D。
- **CISC 循环指令**：`gemmini_loop_ws`（大矩阵乘法）、`gemmini_loop_conv_ws`（支持池化的卷积）。这些指令自动分块并支持双缓冲。

### 生成器参数
- 脉动阵列维度：`tileRows`、`tileColumns`、`meshRows`、`meshColumns`（两级层次结构：组合逻辑 Tile + 流水线 Mesh）。
- 类型参数：`inputType`（如 SInt(8.W)）、`outputType`、`accType`（如 SInt(32.W)）。
- 队列参数：`ld_queue_length`、`st_queue_length`、`ex_queue_length`、`rob_entries`，用于访问-执行解耦。
- DMA 参数：`dma_maxbytes`、`dma_buswidth`。

### 内存寻址
- 按行寻址的私有存储器，每行宽度为 `DIM` 个元素。
- 32 位地址：第 31 位选择 Scratchpad（0）或累加器（1）。第 30 位控制覆写或累加。第 29 位控制缩放读取或原始读取。

### 与 SNCPU 的对比
- **Gemmini**：异构架构（Rocket CPU + 通过 RoCC 连接的独立加速器）。CPU 在整个执行过程中向加速器发送指令，DMA 负责数据搬移。
- **SNCPU**：统一架构（PE 阵列可在 CPU 和加速器之间重配置）。无需 DMA。双模双向数据流使数据保持本地化。
- **RSNCPU**（本项目）：SNCPU 的升级版本，旨在将 SNCPU 的数据局部性优势与 Gemmini 级别的 ISA 灵活性相结合。

---
title: LoRA and QLoRA
date: 2025-12-30 16:42:18
tags: [LLM,PEFT]
---

## 1. 概述 (Executive Summary)

随着大语言模型 (LLM) 参数量的爆炸式增长（7B 至 70B+），传统的**全量微调 (Full Fine-Tuning)** 因其高昂的算力成本（显存与计算资源）和存储成本，在实际应用中变得难以落地。

**LoRA (Low-Rank Adaptation)** 与 **QLoRA (Quantized LoRA)** 是目前最主流的**参数高效微调 (PEFT, Parameter-Efficient Fine-Tuning)** 技术。它们通过冻结预训练模型权重，仅训练少量新增参数，在大幅降低显存需求的同时，实现了与全量微调相当的性能。

本主要技术指标：

- **LoRA**：显存占用降低约 3 倍，模型文件缩小 1000+ 倍。
- **QLoRA**：在 LoRA 基础上引入 4-bit 量化，进一步降低显存需求（7B 模型仅需 6-8GB 显存即可微调）。

------

## 2. LoRA (Low-Rank Adaptation) 技术详解

### 2.1 核心数学原理

LoRA 基于一个核心假设：**过参数化的大模型在适应特定任务时，其权重更新矩阵具有极低的“内在秩” (Intrinsic Rank)。**

假设预训练权重矩阵为 $W_0 \in \mathbb{R}^{d \times k}$，微调后的权重为 $W$。

全量微调的目标是学习更新量 $\Delta W$，即 $W = W_0 + \Delta W$。

LoRA 将 $\Delta W$ 分解为两个低秩矩阵的乘积：

$$\Delta W = B \cdot A$$

其中：

- $B \in \mathbb{R}^{d \times r}$
- $A \in \mathbb{R}^{r \times k}$
- $r \ll \min(d, k)$，即**秩 (Rank)**，通常取 8, 16, 32, 64 等较小值。

#### 前向传播公式

对于输入 $x$，修正后的前向传播过程为：

$$h = W_0 x + \Delta W x = W_0 x + B A x$$

注意：$W_0$ 在训练过程中是冻结的 (Frozen)，仅 $A$ 和 $B$ 参与梯度更新。

#### 初始化策略 (Initialization)

为了保证训练开始时模型的输出与原模型一致：

- 矩阵 $A$ 使用**随机高斯分布**初始化。
- 矩阵 $B$ 使用**全零**初始化。
- 初始状态下 $BAx = 0$，模型行为完全等同于预训练模型。

#### 缩放系数 (Scaling Factor)

实际计算中包含一个缩放系数 $\alpha$：

$$\Delta W = \frac{\alpha}{r} (B A)$$

- $\alpha$ (Alpha) 是常数超参数，类似于学习率调节器。
- 在训练时，通常将 $\alpha$ 设置为 $r$ 的倍数（如 $r=16, \alpha=32$）。这使得调整 $r$ 时无需频繁重新调整学习率。

### 2.2 架构优势

1. **无推理延迟 (No Inference Latency)：** 训练完成后，可以将 $BA$ 显式地加回 $W_0$ 中，部署时不需要额外的计算步骤。
2. **存储高效：** 对于 GPT-3 (175B) 级别的模型，全量微调需要保存 350GB 的 checkpoint，而 LoRA 仅需保存约 350MB。
3. **任务切换灵活：** 基础模型只需部署一份，针对不同任务（如代码生成、文本摘要）只需动态加载对应的 LoRA 权重。

------

## 3. QLoRA (Quantized LoRA) 技术详解

### 3.1 核心痛点与解决方案

LoRA 虽然减少了可训练参数，但在训练过程中，**基础模型 $W_0$ 仍需以 16-bit (FP16/BF16) 精度驻留在显存中**。这对于显存有限的消费级显卡仍是瓶颈。

QLoRA 通过将 $W_0$ 量化为 **4-bit**，极大压缩了底模的显存占用，同时利用 LoRA 微调 16-bit 的 Adapter，实现了精度与效率的平衡。

### 3.2 三大核心技术创新

QLoRA 并不仅仅是简单的“量化+微调”，它引入了三项关键技术来保证在 4-bit 极端压缩下的性能不损失：

#### A. 4-bit NormalFloat (NF4) 数据类型

传统的 4-bit 整数 (Int4) 量化对于正态分布权重的拟合效果不佳。

- **原理：** 神经网络的权重通常服从均值为 0 的正态分布。
- **NF4：** 一种基于分位数（Quantile）量化的数据类型。它根据正态分布的累积分布函数 (CDF) 划分量化区间，确保每个量化桶中的数值数量大致相等。
- **效果：** NF4 在信息理论上是最优的 4-bit 数据类型，比 Int4 精度更高。

#### B. 双重量化 (Double Quantization)

量化过程本身需要存储“量化常数” (Quantization Constants)，这部分也会占用显存。

- **原理：** QLoRA 对量化常数本身再进行一次 8-bit 量化。
- **收益：** 每个参数平均节省约 0.37 bit。对于 65B 模型，这能额外节省约 3GB 显存。

#### C. 分页优化器 (Paged Optimizers)

- **原理：** 利用 NVIDIA 统一内存特性，当 GPU 显存不足时，自动将优化器状态 (Optimizer States) 逐页 (Page-by-Page) 转移到 CPU RAM 中。
- **收益：** 有效防止训练过程中的显存峰值导致 OOM (Out Of Memory)，允许在较小的 GPU 上训练较大的批次 (Batch Size)。

### 3.3 计算流程

在 QLoRA 的前向传播中：

1. 存储的权重是 4-bit 的。
2. 计算时，将 4-bit 权重**实时反量化 (De-quantize)** 为 BF16。
3. 在 BF16 精度下进行矩阵乘法。
4. 计算出的梯度用于更新 LoRA 的参数（LoRA 参数始终保持 BF16）。

------

## 4. 技术对比分析 (Comparison)

| **维度**          | **Full Fine-Tuning** | **LoRA**           | **QLoRA**                   |
| ----------------- | -------------------- | ------------------ | --------------------------- |
| **基础模型精度**  | 16-bit (FP16/BF16)   | 16-bit (FP16/BF16) | **4-bit (NF4)**             |
| **可训练参数量**  | 100%                 | 0.01% - 5%         | 0.01% - 5%                  |
| **显存需求 (7B)** | > 60 GB              | ~ 16-24 GB         | **~ 6-8 GB**                |
| **训练速度**      | 慢                   | 快                 | **较慢** (因涉及实时反量化) |
| **模型性能**      | 基准 (100%)          | ~99% - 100%        | ~98% - 99%                  |
| **存储空间**      | 巨大 (GB 级别)       | 极小 (MB 级别)     | 极小 (MB 级别)              |
| **硬件门槛**      | A100/H100 集群       | A10/3090/4090      | **3060/4060/T4**            |

------

## 5. 工程最佳实践 (Best Practices)

### 5.1 超参数推荐

在训练文本模型时，基于实战经验的推荐设置：

1. **Rank (r) 与 Alpha:**
   - **推荐：** $r=16, \alpha=32$ 或 $r=64, \alpha=128$。
   - **原则：** Alpha 通常设置为 $r$ 的 2 倍。如果数据量非常小，适当降低 $r$ 防止过拟合。
2. **Target Modules (目标模块):**
   - **保守策略：** 仅针对 `q_proj`, `v_proj` (Attention 部分)。
   - **进阶策略 (推荐)：** 针对 `all-linear` (所有线性层)，包括 `q_proj, k_proj, v_proj, o_proj, gate_proj, up_proj, down_proj`。
   - **效果：** 针对所有线性层训练通常能获得更好的效果，虽然参数量略有增加，但性价比极高。
3. **Learning Rate (学习率):**
   - LoRA/QLoRA 支持比全量微调更大的学习率。
   - 推荐范围：`2e-4` 到 `5e-5`。

### 5.2 常见问题排查 (Troubleshooting)

- **Loss 不下降：**
  - 检查数据格式是否正确（Prompt 模板是否匹配底模）。
  - 检查学习率是否过大导致梯度爆炸，或过小导致停滞。
  - 尝试增加 Rank 或更换 Target Modules。
- **训练后模型只会重复输出：**
  - 通常是因为 `EOS_TOKEN` (结束符) 没有正确添加到训练数据中，导致模型不知道何时停止。
- **推理速度慢：**
  - 如果使用的是 QLoRA 且未合并权重，推理时会有反量化开销。
  - **解决方案：** 将 LoRA 权重与底模合并 (Merge)，并导出为标准 FP16 模型进行服务部署。

------

## 6. 参考文献与工具

- **Paper (LoRA):** *LoRA: Low-Rank Adaptation of Large Language Models (Hu et al., 2021)*
- **Paper (QLoRA):** *QLoRA: Efficient Finetuning of Quantized LLMs (Dettmers et al., 2023)*
- **推荐库:**
  - Hugging Face `PEFT`
  - `bitsandbytes` (QLoRA 核心依赖)
  - `LLaMA-Factory` (一站式训练框架)
  - `Unsloth` (加速训练库)

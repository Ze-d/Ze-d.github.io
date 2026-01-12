---
title: CPU Microarchitecture & Hardware Implementation
date: 2026-01-05 17:42:49
tags: [cs,os]
---

## 1. 软件到硬件的映射

高级语言的锁操作最终会编译为特定的机器指令：

- **Java:** `Atomic.increment()` / `synchronized` (轻量级/偏向锁阶段)。
- **Assembly (x86):** **`LOCK` 前缀指令** (e.g., `lock cmpxchg`, `lock xadd`)。
- **Assembly (ARM):** `LDREX` (Load Exclusive) 和 `STREX` (Store Exclusive) 对。

## 2. LOCK 指令的工作原理

当 CPU 执行带 `LOCK` 前缀的指令时，必须确保“读-改-写”的原子性。硬件经历了两个阶段的演进：

### 阶段 A：总线锁 (Bus Lock) —— 粗粒度

- **机制：** CPU 通过引脚拉低 `LOCK#` 信号。
- **效果：** 物理封锁内存总线，所有其他核心无法访问**任何**内存地址。
- **缺点：** 性能断崖式下跌，多核退化为单核。

### 阶段 B：缓存锁 (Cache Lock) —— 细粒度

- **机制：** 利用缓存一致性协议 (如 MESI) 锁定特定的**缓存行 (Cache Line)**。
- **条件：** 数据必须已经存在于 CPU 缓存中且未跨越缓存行边界。
- **流程：**
  1. Core A 发出 **RFO (Read For Ownership)** 信号。
  2. Core B/C 嗅探到信号，将自己缓存中的对应行标记为 **I (Invalid)**。
  3. Core A 获得独占权，修改数据，状态变为 **M (Modified)**。
  4. Core B 下次读取时，强制发生 Cache Miss，重新拉取数据。

## 3. MESI 协议详解

- **M (Modified):** 已修改。数据是脏的，只在当前 Cache，且与内存不一致。
- **E (Exclusive):** 独占。数据是干净的，只在当前 Cache，与内存一致。
- **S (Shared):** 共享。数据是干净的，多个 Cache 都有。
- **I (Invalid):** 失效。数据不可用。

## 4. 性能悖论：为什么加锁会慢？

1. **通信开销：** 锁操作需要核心间广播消息 (RFO)，占用总线带宽。
2. **流水线停顿：** `LOCK` 指令具有**内存屏障 (Memory Barrier)** 效果，会禁止指令重排序，并强制清空 Store Buffer（写缓冲），导致 CPU 流水线停顿。
3. **缓存失效：** 竞争导致其他核心缓存失效，引发强制的内存读取 (Cache Miss)，耗时从 1ns 激增至 100ns。

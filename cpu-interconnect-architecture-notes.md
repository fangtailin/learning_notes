# 现代数据中心 CPU 互联架构学习笔记

## 0. 学习目标

这份笔记用于理解现代高性能 CPU 为什么越来越重视“互联架构”和“内存路径”，以及 NVIDIA Vera、AMD EPYC、Arm AGI、Microsoft Cobalt 200 这几类代表性设计分别在解决什么问题。

核心问题不是“谁的核心更多”，而是：

- CPU 核心之间如何通信？
- 核心如何访问缓存和内存？
- 数据在片内、片间、内存、I/O 之间走多远？
- 高核心数下如何控制延迟、带宽、功耗和成本？
- 云计算和 AI 基础设施为什么越来越需要定制 CPU？

一句话总结：

> 现代数据中心 CPU 的竞争，已经从“堆核心数”转向“在真实云负载中平衡带宽、延迟、功耗、安全、成本和平台协同”。

## 1. 先理解几个核心概念

### 1.1 Core、Cache、Memory 和 Fabric

CPU 不只是很多核心的集合。一个高性能服务器 CPU 通常包括：

- **CPU core**：执行程序指令的计算单元。
- **L1/L2 cache**：靠近核心的小容量高速缓存。
- **L3 / system-level cache**：多个核心共享或分布式共享的更大缓存。
- **Memory controller**：连接 DDR、LPDDR、HBM 等内存的控制器。
- **I/O controller**：连接 PCIe、CXL、NVMe、NIC、GPU 等外部设备。
- **Fabric / interconnect**：把核心、缓存、内存控制器、I/O 控制器连接起来的内部网络。

在单核时代，CPU 的关键是核心本身的性能；在几十核、上百核时代，关键变成了：

> 如何让大量核心高效地访问数据。

如果核心很多，但互联和内存跟不上，就会出现“核心等数据”的情况。

### 1.2 Monolithic vs Chiplet

现代 CPU 大致有两种物理实现方式。

**Monolithic die（单片大硅片）**

- 所有核心、缓存、内存控制器、I/O 尽量集成在一块大 die 上。
- 优点：路径更直接，拓扑更统一，延迟更可控。
- 缺点：die 越大，制造良率越低，成本越高，扩展核心数困难。

**Chiplet（小芯粒）**

- 把 CPU 拆成多个小 die，再通过封装互联组合成一颗处理器。
- 优点：良率友好，容易组合不同 SKU，适合堆核心数。
- 缺点：跨芯粒访问路径更复杂，延迟、带宽和 NUMA 拓扑更难管理。

### 1.3 NUMA

NUMA 是 Non-Uniform Memory Access，意思是“非一致内存访问”。

简单理解：

- 本地核心访问本地内存或本地缓存，比较快。
- 一个核心访问远端 chiplet、远端 socket 或远端内存，路径更长，延迟更高。

服务器 CPU 越多芯粒、越多 socket，NUMA 影响越明显。

### 1.4 为什么 AI 和云原生 workload 改变了 CPU 设计

过去很多 CPU benchmark 更关注单线程性能或纯计算吞吐。但云和 AI 场景中，CPU 经常要处理：

- 数据库查询；
- Web/API 请求；
- TLS/加密通信；
- 压缩和解压；
- 数据搬运；
- 分布式任务调度；
- 容器和虚拟机隔离；
- GPU/AI accelerator orchestration；
- agentic AI 的大量小任务、工具调用和状态管理。

这些负载不只是“算得快”就够了，还要求：

- 内存带宽高；
- 缓存命中率好；
- 核心之间通信快；
- I/O 路径短；
- 加密和压缩能卸载；
- 多租户隔离强；
- 单位功耗能做更多工作。

## 2. NVIDIA Vera / 第二代 SCF 路线

### 2.1 它想解决什么问题

NVIDIA Vera CPU 的核心目标是为 AI factory 时代的 agentic AI、reinforcement learning、数据分析、orchestration、storage 和 HPC 等负载提供高单线程性能、高并发能力和高内存带宽。

Vera 相比 Grace 更明确地面向“CPU for agents”：当 AI agent 需要调用工具、运行代码、执行 Python/JavaScript、访问数据库、做检索、执行 sandbox 任务时，CPU 会进入 AI loop 的关键路径。Vera 的设计目标就是减少这些 CPU-bound 环节对 GPU/accelerator 的拖慢。

### 2.2 SCF 是什么

SCF 是 **Scalable Coherency Fabric**。

可以把它理解为 Vera CPU 内部的高速一致性网络。Vera 使用第二代 NVIDIA SCF，连接：

- 88 个 NVIDIA custom Olympus cores；
- unified cache；
- memory subsystem；
- system I/O；
- NVLink-C2C。

SCF 不是普通总线，而是用于维持片内一致性和高带宽数据流动的 mesh/fabric 架构。Vera 的第二代 SCF 重点是让所有核心在高负载下仍能稳定访问 cache、memory、I/O 和 NVLink-C2C。

### 2.3 Vera CPU 的关键结构

| 项目 | 信息 |
|---|---|
| CPU core | 88 个 NVIDIA custom Olympus cores |
| ISA | Armv9.2 compatible |
| Threading | NVIDIA Spatial Multithreading，最多 176 threads |
| Compute topology | Single monolithic compute die，旁边 dielets 实现 memory 和 I/O subsystem |
| SCF | 第二代 NVIDIA Scalable Coherency Fabric |
| SCF 带宽 | 3.4TB/s bisection bandwidth |
| 内存 | LPDDR5X，通过 SOCAMM 提供可拆卸、可升级、可维护模块 |
| 内存带宽 | 最高 1.2TB/s |
| 内存容量 | 每 socket 最高 1.5TB |
| 每核心内存带宽 | 最高约 14GB/s per core |
| NVLink-C2C | 最高 1.8TB/s coherent bandwidth，可连接 Vera CPU 与 NVIDIA GPUs，也可用于 dual-socket Vera |
| TDP | 公开技术博客提到 configurable 250W-450W TDP range |

### 2.4 Vera 为什么继续强调 LPDDR5X

传统服务器 CPU 通常使用 DDR5 DIMM。Vera 使用 LPDDR5X，但通过 **SOCAMM** 把低功耗内存带入数据中心，同时保留一定的可维护性和可升级性。

这样做的好处：

- 内存带宽高；
- 单位带宽功耗低；
- 每核心可获得更高内存带宽；
- 支持大量并发 sandbox、tool calls、ETL、analytics 和 orchestration workload；
- SOCAMM 相比 Grace 时代的 soldered/on-module LPDDR 方案，更强调可拆卸和可服务性。

代价：

- 仍然比传统 DDR5 服务器平台更依赖特定平台设计；
- 生态开放度和配置自由度不如通用 DDR5 DIMM 服务器；
- 更适合 NVIDIA AI factory / Vera Rubin / MGX 等平台化部署。

### 2.5 Vera 的 Olympus core 为什么重要

Vera 不再使用 Arm Neoverse V2 core，而是使用 NVIDIA 自研的 **Olympus core**。

公开资料中，Olympus 的重点包括：

- 面向 branch-heavy、memory-sensitive、control-heavy 的 agentic code；
- 支持 Armv9.2 软件生态；
- 10-wide instruction fetch/decode frontend；
- neural branch predictor；
- deep out-of-order scheduling；
- 专门的 memory prefetching；
- 支持 NVIDIA Spatial Multithreading，在性能/线程数之间做运行时选择。

它的目标不是单纯提高核心数量，而是提高每个 agentic step 的完成速度，尤其是编译、解释器、脚本、代码运行、数据检索和工具调用这类 CPU-bound 环节。

### 2.6 Vera 路线的理解重点

Vera 的思路是：

> 用高 IPC 的 Olympus core、第二代 SCF、LPDDR5X/SOCAMM 和 NVLink-C2C，把 CPU 做成 AI factory 中服务 agentic AI 和 reinforcement learning 的高带宽执行平台。

可以把 NVIDIA Vera 理解为：

> 面向 agentic AI 时代的高单线程、高带宽、强 NVLink 生态 CPU。

## 3. AMD EPYC Chiplet 路线

### 3.1 它想解决什么问题

AMD EPYC 的核心思路是通过 chiplet 架构，把服务器 CPU 做到高核心数、高良率、高产品复用度，同时保持成熟 x86 生态。

它的重点不是单片统一拓扑，而是：

> 用模块化小芯粒组合出不同核心数和不同市场定位的服务器 CPU。

### 3.2 CCD、I/O Die 和 Infinity Fabric

AMD EPYC 通常由两类 die 构成：

- **CCD（Core Complex Die）**：主要放 CPU cores 和缓存。
- **I/O Die（IOD）**：放内存控制器、PCIe/CXL、I/O 等资源。

CCD 与 I/O Die 之间通过 **Infinity Fabric** 连接。

这种设计的好处是：

- CCD 可以复用；
- 坏掉一个小 die 不一定影响整颗产品设计；
- 可以灵活组合不同核心数；
- 容易扩展到 96、128、192 cores 这样的高核心数。

### 3.3 AMD 路线的优势

- **良率友好**：小 die 比超大 die 更容易制造。
- **成本效率高**：同一类 chiplet 可复用于多个 SKU。
- **扩展性强**：适合高核心数服务器 CPU。
- **x86 生态成熟**：企业软件、虚拟化、数据库、OS 支持都非常成熟。
- **产品覆盖广**：可以做高频、低功耗、高核心数、缓存增强等不同方向。

### 3.4 AMD 路线的代价

chiplet 架构的代价主要在拓扑复杂度。

需要注意：

- 跨 CCD 访问比本地访问更复杂；
- 远端 NUMA 访问延迟更高；
- 内存和 I/O 位于 I/O Die，路径不如单片直连简单；
- workload 调度、NUMA 亲和性、BIOS NPS 配置会影响性能。

不要简单说“所有跨核心通信都必须通过 cIOD 中转”。更准确的理解是：

> AMD EPYC 的跨 CCD、远端 NUMA、内存和 I/O 访问会受 chiplet 拓扑影响，具体延迟和带宽取决于 CPU 代际、系统配置和 workload。

### 3.5 AMD 路线的理解重点

AMD EPYC 是典型的“商业工程最优解”：

> 牺牲一定拓扑一致性，换取良率、成本效率、产品复用度、高核心数和成熟 x86 生态。

可以把 AMD EPYC 理解为：

> 最擅长规模化通用服务器部署的 chiplet 路线。

## 4. Arm AGI CPU 路线

### 4.1 它想解决什么问题

Arm AGI CPU 是 Arm 自己推出的 production silicon 数据中心 CPU，面向 agentic AI infrastructure。

这里的 agentic AI infrastructure 不是单指模型推理，而是指 AI 数据中心里大量 CPU 侧工作：

- 管理 accelerator；
- 处理 control plane；
- 调度 agent 任务；
- 处理 API 和工具调用；
- 运行应用和服务框架；
- 做数据搬运和 orchestration。

### 4.2 Arm AGI 的关键规格

| 项目 | 信息 |
|---|---|
| 产品属性 | Arm 自有成品数据中心 CPU |
| 核心 | 最高 136 Neoverse V3 cores |
| 架构 | Armv9.2 |
| L2 | 2MB/core |
| System-level cache | 128MB |
| 内存 | 12x DDR5，最高 8800MT/s |
| Memory throughput/core | 136C SKU 约 6GB/s per core |
| I/O | 96 lanes PCIe Gen6 |
| CXL | CXL 3.0 Type 3 |
| Base TDP | 300W |
| 2-socket support | Yes |

### 4.3 Arm AGI 的设计重点

Arm AGI 的重点是：

- 高核心密度；
- 高内存带宽；
- 标准 DDR5；
- PCIe Gen6；
- CXL 3.0；
- 面向 rack-scale AI 基础设施；
- 给 AI accelerator 提供 CPU orchestration layer。

它不是为了替代 GPU，而是为了让 GPU/accelerator 周边的 CPU 工作更高效。

### 4.4 Arm AGI 路线的理解重点

Arm AGI 可以理解为：

> Arm 从“提供 IP/CSS”走向“提供成品数据中心 CPU”的尝试，目标是吃到 agentic AI 基础设施中 CPU orchestration 的增长需求。

## 5. Microsoft Azure Cobalt 200 路线

### 5.1 它想解决什么问题

Cobalt 200 是 Azure 第二代自研 Arm CPU / SoC，面向 Azure 自己的数据中心和 VM 产品。

它不是 Arm AGI CPU 的改名版，也不是通用零售 CPU。它的核心目标是：

> 针对 Azure 真实云负载，从 silicon 到 server 到 service 做协同优化。

典型目标负载包括：

- cloud-native workload；
- Linux-based agentic AI workload；
- 数据库；
- Web/API；
- 缓存；
- 数据管道；
- 加密通信；
- 多租户 VM。

### 5.2 Cobalt 200 的关键规格

| 项目 | 信息 |
|---|---|
| 产品属性 | Microsoft Azure 自研/定制 SoC |
| 代际 | 第二代 Azure Cobalt Arm CPU |
| 核心架构 | Arm Neoverse CSS V3 |
| 制程 | TSMC 3nm / N3P |
| 核心数 | 132 active cores |
| L2 | 3MB/core |
| L3 | 192MB system-level L3 |
| 内存控制器 | Microsoft custom memory controller |
| 安全 | 默认内存加密，negligible performance impact |
| Confidential computing | Arm Confidential Compute Architecture |
| 加速器 | Compression、cryptography、data movement accelerators |
| 功耗控制 | Per-core Dynamic Voltage and Frequency Scaling |
| 平台能力 | Azure Boost networking / remote storage offload integration |
| HSM | Azure Integrated HSM |

### 5.3 Cobalt 200 为什么强调缓存和加速器

云负载中很多开销不是纯计算，而是：

- 压缩；
- 解压；
- 加密；
- 数据搬运；
- 网络和远程存储 I/O；
- 多租户隔离；
- VM 安全边界。

Cobalt 200 通过更大的 L2/L3 缓存和专用 accelerator 来减少 CPU core 被这些通用操作占用。

例如：

- 数据库可以把压缩/加密卸载出去；
- Web/API 可以提升加密通信效率；
- 缓存和数据管道可以受益于数据搬运和压缩加速；
- 多租户 VM 可以受益于默认内存加密和 Arm CCA。

### 5.4 Cobalt 200 的 VM 层能力

| 项目 | 信息 |
|---|---|
| 性能提升 | 相比 Cobalt 100，最高 50% CPU performance improvement |
| Remote storage IOPS | 最高 20% 提升 |
| Remote storage throughput | 最高 10% 提升 |
| Network bandwidth | 最高 15% 提升 |
| vCPU | VM 最高 128 vCPUs |
| VM 系列 | Dplsv7 / Dpldsv7、Dpsv7 / Dpdsv7、Epsv7 / Epdsv7、Mpsv4 / Mpdsv4、Lpsv5 |

### 5.5 不要混淆 Cobalt 200 和 Arm AGI

Arm AGI 和 Cobalt 200 都体现 Neoverse V3 / CSS V3 时代的数据中心 CPU 趋势，但不能说它们是同一颗芯片。

目前不建议把以下 Arm AGI 信息写到 Cobalt 200 上：

- DDR5-8800；
- PCIe Gen6；
- CXL 3.0；
- 136 cores；
- 2MB L2/core；
- 128MB system-level cache。

这些是 Arm AGI CPU 官方披露的信息，不是 Cobalt 200 官方重点披露的信息。

### 5.6 Cobalt 200 路线的理解重点

Cobalt 200 可以理解为：

> Azure 基于 Arm Neoverse CSS V3 做的云平台定制 CPU，用更强缓存、安全、加速器和 Azure Boost 协同来优化真实云负载。

它代表的是 hyperscaler custom silicon 思路：

> 云厂商不只是买通用 CPU，而是根据自己的数据中心 workload 定制 CPU。

## 6. Arm AGI 与 Cobalt 200 的关系

### 6.1 相同点

二者都体现同一类技术趋势：

- Arm Neoverse V3 / CSS V3；
- 高核心密度；
- 多通道 DDR5；
- 面向 AI 和云基础设施；
- 强调每核心内存带宽；
- 强调系统级能效；
- 倾向 chiplet / multi-die 扩展；
- 服务 agentic AI 时代的 CPU 侧负载。

### 6.2 不同点

| 维度 | Arm AGI CPU | Microsoft Azure Cobalt 200 |
|---|---|---|
| 产品属性 | Arm 自有成品数据中心 CPU | Microsoft Azure 自研/定制 SoC |
| 目标客户 | AI 数据中心、合作伙伴、ODM/OEM 生态 | Azure 自家云平台和 VM 产品 |
| 核心数 | 最高 136 cores | 132 active cores |
| L2 | 2MB/core | 3MB/core |
| System cache / L3 | 128MB system-level cache | 192MB system-level L3 |
| 内存 | 12x DDR5，最高 8800MT/s | 12 通道 DDR5 方向明确，但公开资料未强调 DDR5-8800 |
| I/O | 96 lanes PCIe Gen6，CXL 3.0 Type 3 | 公开资料未确认 PCIe Gen6 / CXL 3.0 |
| 安全 | 面向 AI 数据中心平台安全 | 默认内存加密、Arm CCA、Azure Integrated HSM |
| 专用加速器 | 官网重点较少 | Compression / cryptography / data movement accelerators |
| 主要场景 | Agentic AI infrastructure、accelerator orchestration | Azure 云原生、数据库、缓存、Web/API、agentic AI VM |

### 6.3 记忆方式

可以这样记：

- **Arm AGI**：Arm 自己下场做的一颗 agentic AI 数据中心 CPU。
- **Cobalt 200**：Azure 为自己云平台做的一颗 Neoverse CSS V3 定制 CPU。

二者是“同一技术浪潮下的不同落地方式”，不是“孪生芯片”。

## 7. 三种路线的对比理解

| 维度 | NVIDIA Vera / 第二代 SCF | AMD EPYC Chiplet | Arm AGI / Cobalt 200 |
|---|---|---|---|
| 核心思想 | 高 IPC Olympus cores + 第二代 SCF + LPDDR5X/SOCAMM + 强 NVLink 生态 | 模块化 chiplet + x86 生态 | Neoverse V3/CSS V3 + 云/AI 定制 |
| 互联重点 | 第二代 SCF、single monolithic compute die、NVLink-C2C | Infinity Fabric、CCD/IOD 拓扑 | Arm CMN/CSS mesh、chiplet/multi-die |
| 内存路线 | LPDDR5X + SOCAMM，最高 1.2TB/s | 插槽式 DDR5 | 多通道 DDR5 |
| 强项 | 高单线程性能、高内存带宽、agentic sandbox、CPU-GPU 协同 | 良率、扩展性、x86 软件生态 | 云原生、安全、agentic AI orchestration |
| 主要代价 | 更依赖 NVIDIA AI factory / Vera Rubin / MGX 平台生态 | NUMA 和跨 CCD 拓扑复杂 | 依赖具体云平台和生态落地 |
| 典型场景 | Agentic AI、RL sandbox、工具调用、数据分析、orchestration、HPC、CPU-GPU 协同 | 通用云计算、虚拟化、数据库 | 云原生、多租户、AI 控制面、缓存、数据库 |

## 8. 如何从架构图读懂一颗服务器 CPU

读 CPU 架构图时，可以按以下顺序看。

### 8.1 先看物理形态

问自己：

- 是单片 die 还是 chiplet？
- 有几个 compute die？
- 是否有单独 I/O die？
- 是否是 dual-socket 或 superchip？

这决定了数据路径的大框架。

### 8.2 再看核心和缓存

问自己：

- 有多少核心？
- 每核心 L2 多大？
- L3 是集中式还是 distributed？
- L3 / system cache 总量多少？

核心多但缓存小，容易被内存延迟拖累。

### 8.3 再看内存路径

问自己：

- 内存控制器在哪里？
- 是 DDR5 DIMM、LPDDR5X on-module，还是 HBM？
- 有多少内存通道？
- 每核心内存带宽够不够？

很多云负载和 AI orchestration workload 都很吃内存带宽。

### 8.4 再看片内/片间互联

问自己：

- 核心之间通过什么 fabric 通信？
- 跨 die 用什么协议？
- 跨 socket 用什么协议？
- 是否有 NUMA？
- 拓扑是否对称？

互联决定了多核心扩展时的真实效率。

### 8.5 最后看平台协同

问自己：

- 是否有网络/存储 offload？
- 是否有加密/压缩 accelerator？
- 是否支持 confidential computing？
- 是否与云平台调度、VM、容器生态协同？

现代云 CPU 的竞争已经不是单颗芯片，而是整个平台。

## 9. 常见误区

### 9.1 误区：核心越多越强

核心数只是上限。真实性能还取决于：

- 每核心性能；
- 缓存层级；
- 内存带宽；
- NUMA 拓扑；
- I/O 路径；
- workload 是否能并行；
- 软件调度是否正确。

### 9.2 误区：Chiplet 一定更慢

Chiplet 有跨 die 代价，但它带来了良率、成本和扩展性优势。对很多高并发、多实例、云虚拟化场景，chiplet 是非常合理的工程选择。

### 9.3 误区：单片一定最好

单片路径更统一，但超大 die 良率和成本压力很大，核心数扩展也有限。单片不是绝对最优，只是适合某些高带宽、低延迟、强耦合场景。

### 9.4 误区：Arm AGI 和 Cobalt 200 是同一颗芯片

它们都使用 Neoverse V3 / CSS V3 相关技术路线，但产品属性不同：

- Arm AGI 是 Arm 自有成品 CPU。
- Cobalt 200 是 Microsoft Azure 定制 SoC。

### 9.5 误区：Cobalt 200 已确认 PCIe Gen6 / CXL 3.0

这些是 Arm AGI CPU 官方披露的信息。Cobalt 200 公开资料目前没有把 PCIe Gen6 / CXL 3.0 作为明确规格披露，因此学习笔记中应保持区分。

## 10. 记忆口诀

### 10.1 三条路线

- **NVIDIA Vera**：agentic AI 执行、单线程性能、内存带宽和 NVLink 生态优先。
- **AMD EPYC**：chiplet 扩展和 x86 生态优先。
- **Arm AGI / Cobalt 200**：云原生、AI orchestration、安全和平台定制优先。

### 10.2 三个关键词

- Vera：**Olympus core + 第二代 SCF + LPDDR5X/SOCAMM + NVLink-C2C**
- EPYC：**CCD + I/O Die + Infinity Fabric**
- Cobalt 200：**CSS V3 + 3MB L2/core + Azure Boost**

### 10.3 一句话理解

> NVIDIA Vera 像 AI 工厂里的高速执行车间；AMD EPYC 像模块化城市路网；Cobalt 200 像 Azure 为自己云业务定制的智能交通系统。

## 11. 复习问题

1. 为什么服务器 CPU 不只看核心数？
2. SCF 与普通总线的区别是什么？
3. Vera 为什么选择 LPDDR5X/SOCAMM，而不是传统 DDR5 DIMM？
4. AMD EPYC 为什么要把 CCD 和 I/O Die 分开？
5. NUMA 对数据库和虚拟化 workload 有什么影响？
6. Arm AGI CPU 主要服务 AI 数据中心里的哪类 CPU 工作？
7. Cobalt 200 为什么强调 compression、cryptography 和 data movement accelerators？
8. 为什么不能把 Arm AGI 和 Cobalt 200 称为“孪生芯片”？
9. Cobalt 200 公开资料中哪些信息是确认的，哪些不应从 Arm AGI 推断？
10. 如果你要为云原生数据库选择 CPU，应重点关注哪些架构指标？

## 12. 来源与可信度说明

本笔记基于公开官网与一手资料整理：

- NVIDIA Vera CPU 官方资料；
- NVIDIA Vera CPU 技术博客；
- NVIDIA Grace CPU / Grace CPU Superchip 官方资料，用作 Vera 前代背景参考；
- AMD EPYC 官方产品资料；
- Arm Neoverse CSS V3 官方资料；
- Arm AGI CPU 官方资料与 Arm Newsroom；
- Microsoft Azure Cobalt 200 Azure Blog；
- Microsoft TechCommunity Azure Infrastructure Blog。

# Notes on Modern Data Center CPU Interconnect Architectures

## 0. Learning Goal

These notes are intended to help explain why modern high-performance CPUs are placing increasing emphasis on **interconnect architecture** and **memory paths**, and how representative designs such as **NVIDIA Vera**, **AMD EPYC**, **Arm AGI**, and **Microsoft Cobalt 200** differ in their trade-offs.

The core question is no longer simply “who has more cores,” but rather:

- How do CPU cores communicate with each other?
- How do cores access cache and memory?
- How far does data need to travel across on-die, cross-die, memory, and I/O paths?
- How do designs control latency, bandwidth, power, and cost at high core counts?
- Why are cloud computing and AI infrastructure increasingly driving demand for customized CPUs?

One-sentence summary:

> Competition in modern data center CPUs has shifted from simply “adding more cores” to balancing bandwidth, latency, power, security, cost, and platform-level coordination under real cloud workloads.

## 1. First, Understand Several Core Concepts

### 1.1 Core, Cache, Memory, and Fabric

A CPU is not just a collection of many cores. A high-performance server CPU typically includes:

- **CPU core**: the compute unit that executes program instructions.
- **L1/L2 cache**: small, high-speed caches located close to the core.
- **L3 / system-level cache**: a larger cache shared or distributed across multiple cores.
- **Memory controller**: the controller that connects to DDR, LPDDR, HBM, and other memory types.
- **I/O controller**: the controller that connects to PCIe, CXL, NVMe, NICs, GPUs, and other external devices.
- **Fabric / interconnect**: the internal network that connects cores, cache, memory controllers, and I/O controllers.

In the single-core era, the key concern was core performance itself. In the era of dozens or even hundreds of cores, the key question becomes:

> How do you allow a large number of cores to access data efficiently?

If there are many cores but the interconnect and memory system cannot keep up, cores end up waiting for data.

### 1.2 Monolithic vs. Chiplet

Modern CPUs are broadly implemented in two physical forms.

**Monolithic die**

- All cores, cache, memory controllers, and I/O are integrated as much as possible on one large die.
- Advantages: more direct paths, more uniform topology, and more predictable latency.
- Disadvantages: as the die grows larger, manufacturing yield drops, cost rises, and scaling core count becomes more difficult.

**Chiplet**

- The CPU is split into multiple smaller dies and then combined into a processor through package-level interconnects.
- Advantages: better manufacturing yield, easier SKU composition, and better scalability to high core counts.
- Disadvantages: cross-chiplet access paths become more complex, and latency, bandwidth, and NUMA topology are harder to manage.

### 1.3 NUMA

NUMA stands for **Non-Uniform Memory Access**.

A simple way to think about it:

- A local core accessing local memory or local cache is relatively fast.
- A core accessing a remote chiplet, remote socket, or remote memory takes a longer path and has higher latency.

The more chiplets and sockets a server CPU has, the more important NUMA effects become.

### 1.4 Why AI and Cloud-Native Workloads Are Changing CPU Design

Traditional CPU benchmarks often focused on single-thread performance or raw compute throughput. But in cloud and AI scenarios, CPUs often handle:

- database queries;
- Web/API requests;
- TLS and encrypted communication;
- compression and decompression;
- data movement;
- distributed task scheduling;
- container and virtual machine isolation;
- GPU / AI accelerator orchestration;
- large numbers of small agentic AI tasks, tool invocations, and state management.

These workloads do not just require “fast compute.” They also demand:

- high memory bandwidth;
- good cache hit rates;
- fast inter-core communication;
- short I/O paths;
- offload for encryption and compression;
- strong multi-tenant isolation;
- more work done per watt.

## 2. NVIDIA Vera / Second-Generation SCF

### 2.1 What Problem Is It Trying to Solve?

The main goal of the NVIDIA Vera CPU is to provide high single-thread performance, high concurrency, and high bandwidth for AI factory era workloads such as agentic AI, reinforcement learning, data analytics, orchestration, storage, and HPC.

Compared with Grace, Vera is more explicitly positioned as a **CPU for agents**: when AI agents need to call tools, run code, execute Python or JavaScript, access databases, perform retrieval, or run sandboxed tasks, the CPU becomes a critical part of the system.

### 2.2 What Is SCF?

SCF stands for **Scalable Coherency Fabric**.

It can be understood as the high-speed coherent on-chip network inside the Vera CPU. Vera uses a **second-generation NVIDIA SCF** to connect:

- 88 NVIDIA custom Olympus cores;
- unified cache;
- memory subsystem;
- system I/O;
- NVLink-C2C.

SCF is not just a conventional bus. It is a mesh/fabric architecture designed to maintain coherence and sustain high-bandwidth data movement across the chip. In Vera, the second-generation SCF is intended to let all cores continue to access cache, memory, and I/O efficiently even under heavy load.

### 2.3 Key Structural Characteristics of Vera CPU

| Item | Information |
|---|---|
| CPU core | 88 NVIDIA custom Olympus cores |
| ISA | Armv9.2 compatible |
| Threading | NVIDIA Spatial Multithreading, up to 176 threads |
| Compute topology | Single monolithic compute die, with adjacent dielets for memory and I/O subsystems |
| SCF | Second-generation NVIDIA Scalable Coherency Fabric |
| SCF bandwidth | 3.4TB/s bisection bandwidth |
| Memory | LPDDR5X via SOCAMM, providing removable, upgradeable, serviceable modules |
| Memory bandwidth | Up to 1.2TB/s |
| Memory capacity | Up to 1.5TB per socket |
| Per-core memory bandwidth | Up to about 14GB/s per core |
| NVLink-C2C | Up to 1.8TB/s coherent bandwidth, for connecting Vera CPUs to NVIDIA GPUs, and also for dual-socket Vera |
| TDP | Public technical blog mentions a configurable 250W–450W TDP range |

### 2.4 Why Vera Continues to Emphasize LPDDR5X

Traditional server CPUs usually use DDR5 DIMMs. Vera uses LPDDR5X, but through **SOCAMM**, it brings low-power memory into the data center while preserving a degree of serviceability and upgradability.

Benefits of this approach:

- high memory bandwidth;
- lower power consumption per unit of bandwidth;
- higher memory bandwidth per core;
- support for large numbers of concurrent sandbox, tool-call, ETL, analytics, and orchestration workloads;
- compared with the soldered/on-module LPDDR approach used in the Grace era, SOCAMM puts more emphasis on removability and serviceability.

Trade-offs:

- it still depends more on platform-specific design than a traditional DDR5 server platform;
- ecosystem openness and configuration flexibility are lower than in general-purpose DDR5 DIMM servers;
- it is better suited to platformized deployments such as NVIDIA AI factory, Vera Rubin, and MGX.

### 2.5 Why the Olympus Core Matters

Vera no longer uses the Arm Neoverse V2 core. Instead, it uses NVIDIA’s in-house **Olympus core**.

According to public materials, Olympus emphasizes:

- performance for branch-heavy, memory-sensitive, and control-heavy agentic code;
- compatibility with the Armv9.2 software ecosystem;
- a 10-wide instruction fetch/decode frontend;
- a neural branch predictor;
- deep out-of-order scheduling;
- specialized memory prefetching;
- support for NVIDIA Spatial Multithreading, enabling runtime trade-offs between performance and thread count.

Its goal is not simply to increase core count, but to improve the completion speed of each agentic step—especially in CPU-bound tasks such as compilation, interpreters, scripting, code execution, data retrieval, and tool invocation.

### 2.6 Key Takeaway for the Vera Approach

The Vera strategy can be summarized as:

> Using high-IPC Olympus cores, second-generation SCF, LPDDR5X/SOCAMM, and NVLink-C2C to turn the CPU into a high-bandwidth execution platform for agentic AI and reinforcement learning in AI factories.

You can think of NVIDIA Vera as:

> A high-single-thread-performance, high-bandwidth CPU with a strong NVLink ecosystem for the agentic AI era.

## 3. The AMD EPYC Chiplet Approach

### 3.1 What Problem Is It Trying to Solve?

The core idea of AMD EPYC is to use a chiplet architecture to deliver high core counts, high manufacturing yield, and high product reuse, while maintaining a mature x86 ecosystem.

Its focus is not on a monolithic unified topology, but on:

> Using modular small dies to assemble server CPUs with different core counts and market positions.

### 3.2 CCD, I/O Die, and Infinity Fabric

AMD EPYC typically consists of two categories of dies:

- **CCD (Core Complex Die)**: mainly contains CPU cores and cache.
- **I/O Die (IOD)**: contains memory controllers, PCIe/CXL, and other I/O resources.

The CCDs and I/O Die are connected through **Infinity Fabric**.

Advantages of this design include:

- CCD reuse;
- a defect in one small die does not necessarily invalidate the entire product design;
- flexible composition across different core counts;
- easier scaling to very high core counts such as 96, 128, or 192 cores.

### 3.3 Strengths of the AMD Approach

- **Yield-friendly**: smaller dies are easier to manufacture than a very large die.
- **Cost-efficient**: the same class of chiplet can be reused across multiple SKUs.
- **Highly scalable**: well suited to high-core-count server CPUs.
- **Mature x86 ecosystem**: enterprise software, virtualization, databases, and OS support are all very mature.
- **Broad product coverage**: can target high-frequency, low-power, high-core-count, and cache-enhanced segments.

### 3.4 Trade-Offs of the AMD Approach

The main trade-off of a chiplet architecture is topology complexity.

Important points to keep in mind:

- cross-CCD access is more complex than local access;
- remote NUMA access has higher latency;
- memory and I/O are located on the I/O Die, so the path is less direct than in a monolithic design;
- workload scheduling, NUMA affinity, and BIOS NPS configuration can all affect performance.

It is not accurate to say “all cross-core communication must be relayed through the cIOD.” A better understanding is:

> Cross-CCD, remote NUMA, memory, and I/O accesses in AMD EPYC are influenced by chiplet topology, and the exact latency and bandwidth depend on the CPU generation, system configuration, and workload.

### 3.5 Key Takeaway for the AMD Approach

AMD EPYC is a classic example of an “optimal commercial engineering solution”:

> It gives up some topology uniformity in exchange for yield, cost efficiency, product reuse, high core counts, and a mature x86 ecosystem.

You can think of AMD EPYC as:

> The chiplet approach best suited to large-scale general-purpose server deployment.

## 4. The Arm AGI CPU Approach

### 4.1 What Problem Is It Trying to Solve?

Arm AGI CPU is Arm’s own production-silicon data center CPU, designed for **agentic AI infrastructure**.

Here, agentic AI infrastructure does not simply mean model inference. It refers to the many CPU-side tasks in AI data centers:

- managing accelerators;
- handling the control plane;
- scheduling agent tasks;
- processing API and tool invocations;
- running application and service frameworks;
- handling data movement and orchestration.

### 4.2 Key Specifications of Arm AGI

| Item | Information |
|---|---|
| Product type | Arm’s own production data center CPU |
| Cores | Up to 136 Neoverse V3 cores |
| Architecture | Armv9.2 |
| L2 | 2MB/core |
| System-level cache | 128MB |
| Memory | 12x DDR5, up to 8800MT/s |
| Memory throughput/core | About 6GB/s per core on the 136-core SKU |
| I/O | 96 lanes PCIe Gen6 |
| CXL | CXL 3.0 Type 3 |
| Base TDP | 300W |
| 2-socket support | Yes |

### 4.3 Design Priorities of Arm AGI

Arm AGI emphasizes:

- high core density;
- high memory bandwidth;
- standard DDR5;
- PCIe Gen6;
- CXL 3.0;
- rack-scale AI infrastructure;
- providing the CPU orchestration layer for AI accelerators.

It is not meant to replace GPUs. Instead, it is meant to make the CPU-side work around GPUs and accelerators more efficient.

### 4.4 Key Takeaway for the Arm AGI Approach

Arm AGI can be understood as:

> Arm’s attempt to move from “providing IP/CSS” to “providing a finished data center CPU,” targeting the growing demand for CPU orchestration in agentic AI infrastructure.

## 5. The Microsoft Azure Cobalt 200 Approach

### 5.1 What Problem Is It Trying to Solve?

Cobalt 200 is Azure’s second-generation in-house Arm CPU / SoC, designed for Azure’s own data centers and VM products.

It is not a renamed Arm AGI CPU, nor is it a general retail CPU. Its core objective is:

> To optimize from silicon to server to service around Azure’s real cloud workloads.

Typical target workloads include:

- cloud-native workloads;
- Linux-based agentic AI workloads;
- databases;
- Web/API services;
- caching;
- data pipelines;
- encrypted communication;
- multi-tenant VMs.

### 5.2 Key Specifications of Cobalt 200

| Item | Information |
|---|---|
| Product type | Microsoft Azure in-house / customized SoC |
| Generation | Second-generation Azure Cobalt Arm CPU |
| Core architecture | Arm Neoverse CSS V3 |
| Process | TSMC 3nm / N3P |
| Core count | 132 active cores |
| L2 | 3MB/core |
| L3 | 192MB system-level L3 |
| Memory controller | Microsoft custom memory controller |
| Security | Memory encryption by default, with negligible performance impact |
| Confidential computing | Arm Confidential Compute Architecture |
| Accelerators | Compression, cryptography, and data movement accelerators |
| Power control | Per-core Dynamic Voltage and Frequency Scaling |
| Platform capability | Azure Boost networking / remote storage offload integration |
| HSM | Azure Integrated HSM |

### 5.3 Why Cobalt 200 Emphasizes Cache and Accelerators

In cloud workloads, much of the overhead is not pure computation, but:

- compression;
- decompression;
- encryption;
- data movement;
- network and remote storage I/O;
- multi-tenant isolation;
- VM security boundaries.

Cobalt 200 uses larger L2/L3 caches and dedicated accelerators to reduce the amount of CPU core time consumed by such general-purpose operations.

For example:

- databases can offload compression and encryption;
- Web/API services can improve encrypted communication efficiency;
- caches and data pipelines can benefit from data movement and compression acceleration;
- multi-tenant VMs can benefit from memory encryption by default and Arm CCA.

### 5.4 Cobalt 200 VM-Level Capabilities

| Item | Information |
|---|---|
| Performance improvement | Up to 50% CPU performance improvement versus Cobalt 100 |
| Remote storage IOPS | Up to 20% improvement |
| Remote storage throughput | Up to 10% improvement |
| Network bandwidth | Up to 15% improvement |
| vCPU | Up to 128 vCPUs per VM |
| VM series | Dplsv7 / Dpldsv7, Dpsv7 / Dpdsv7, Epsv7 / Epdsv7, Mpsv4 / Mpdsv4, Lpsv5 |

### 5.5 Do Not Confuse Cobalt 200 with Arm AGI

Arm AGI and Cobalt 200 both reflect the data center CPU trend of the Neoverse V3 / CSS V3 era, but they should not be described as the same chip.

At present, the following Arm AGI information should **not** be written as Cobalt 200 specifications:

- DDR5-8800;
- PCIe Gen6;
- CXL 3.0;
- 136 cores;
- 2MB L2/core;
- 128MB system-level cache.

These are officially disclosed Arm AGI CPU specifications, not the primary specifications officially disclosed for Cobalt 200.

### 5.6 Key Takeaway for the Cobalt 200 Approach

Cobalt 200 can be understood as:

> A cloud-platform-customized CPU built by Azure on top of Arm Neoverse CSS V3, using stronger cache, security, accelerators, and Azure Boost integration to optimize real cloud workloads.

It represents the hyperscaler custom silicon model:

> Cloud providers are no longer just buying general-purpose CPUs; they are customizing CPUs around their own data center workloads.

## 6. The Relationship Between Arm AGI and Cobalt 200

### 6.1 Similarities

Both reflect the same broad technology trends:

- Arm Neoverse V3 / CSS V3;
- high core density;
- multi-channel DDR5;
- designed for AI and cloud infrastructure;
- emphasis on memory bandwidth per core;
- emphasis on system-level energy efficiency;
- a tendency toward chiplet / multi-die scaling;
- serving CPU-side workloads in the age of agentic AI.

### 6.2 Differences

| Dimension | Arm AGI CPU | Microsoft Azure Cobalt 200 |
|---|---|---|
| Product type | Arm’s own production data center CPU | Microsoft Azure in-house / customized SoC |
| Target customers | AI data centers, partners, ODM/OEM ecosystem | Azure’s own cloud platform and VM products |
| Core count | Up to 136 cores | 132 active cores |
| L2 | 2MB/core | 3MB/core |
| System cache / L3 | 128MB system-level cache | 192MB system-level L3 |
| Memory | 12x DDR5, up to 8800MT/s | 12-channel DDR5 direction is clear, but public materials do not emphasize DDR5-8800 |
| I/O | 96 lanes PCIe Gen6, CXL 3.0 Type 3 | Public materials do not clearly confirm PCIe Gen6 / CXL 3.0 |
| Security | Focused on AI data center platform security | Memory encryption by default, Arm CCA, Azure Integrated HSM |
| Dedicated accelerators | Less emphasized in official messaging | Compression / cryptography / data movement accelerators |
| Main scenarios | Agentic AI infrastructure, accelerator orchestration | Azure cloud-native workloads, databases, caching, Web/API, agentic AI VMs |

### 6.3 A Simple Way to Remember It

You can remember it like this:

- **Arm AGI**: a data center CPU built by Arm itself for agentic AI infrastructure.
- **Cobalt 200**: a Neoverse CSS V3-based customized CPU built by Azure for its own cloud platform.

They are **different implementations emerging from the same technology wave**, not “twin chips.”

## 7. Comparing the Three Approaches

| Dimension | NVIDIA Vera / Second-Generation SCF | AMD EPYC Chiplet | Arm AGI / Cobalt 200 |
|---|---|---|---|
| Core idea | High-IPC Olympus cores + second-generation SCF + LPDDR5X/SOCAMM + strong NVLink ecosystem | Modular chiplets + x86 ecosystem | Neoverse V3/CSS V3 + cloud/AI customization |
| Interconnect focus | Second-generation SCF, single monolithic compute die, NVLink-C2C | Infinity Fabric, CCD/IOD topology | Arm CMN/CSS mesh, chiplet/multi-die |
| Memory approach | LPDDR5X + SOCAMM, up to 1.2TB/s | Socketed DDR5 | Multi-channel DDR5 |
| Strengths | High single-thread performance, high memory bandwidth, agentic sandboxes, CPU-GPU coordination | Yield, scalability, x86 software ecosystem | Cloud-native optimization, security, agentic AI orchestration |
| Main trade-off | More dependent on NVIDIA AI factory / Vera Rubin / MGX platform ecosystem | NUMA and cross-CCD topology complexity | Depends on specific cloud platform and ecosystem adoption |
| Typical scenarios | Agentic AI, RL sandboxes, tool invocation, data analytics, orchestration, HPC, CPU-GPU coordination | General cloud computing, virtualization, databases | Cloud-native workloads, multi-tenant computing, AI control plane, platform-customized infrastructure |

## 8. How to Read a Server CPU Architecture Diagram

When reading a CPU architecture diagram, follow this order.

### 8.1 First Look at the Physical Form

Ask yourself:

- Is it monolithic or chiplet-based?
- How many compute dies are there?
- Is there a separate I/O die?
- Is it dual-socket or a superchip?

This determines the broad structure of the data path.

### 8.2 Then Look at Cores and Cache

Ask yourself:

- How many cores are there?
- How large is the L2 per core?
- Is the L3 centralized or distributed?
- What is the total L3 / system cache capacity?

A CPU can have many cores, but if cache is too small, memory latency can become the bottleneck.

### 8.3 Then Look at the Memory Path

Ask yourself:

- Where is the memory controller located?
- Is the memory DDR5 DIMM, LPDDR5X on-module, or HBM?
- How many memory channels are there?
- Is memory bandwidth per core sufficient?

Many cloud workloads and AI orchestration workloads are heavily dependent on memory bandwidth.

### 8.4 Then Look at On-Die / Cross-Die Interconnect

Ask yourself:

- What fabric is used for inter-core communication?
- What protocol is used across dies?
- What protocol is used across sockets?
- Is NUMA involved?
- Is the topology symmetric?

Interconnect determines real multi-core scaling efficiency.

### 8.5 Finally Look at Platform Coordination

Ask yourself:

- Is there network or storage offload?
- Are there encryption or compression accelerators?
- Is confidential computing supported?
- Is the CPU coordinated with cloud scheduling, VM, and container ecosystems?

Competition in modern cloud CPUs is no longer just about one chip; it is about the whole platform.

## 9. Common Misconceptions

### 9.1 Misconception: More Cores Always Means Better Performance

Core count is only an upper bound. Real performance also depends on:

- per-core performance;
- cache hierarchy;
- memory bandwidth;
- NUMA topology;
- I/O paths;
- whether the workload parallelizes well;
- whether software scheduling is correct.

### 9.2 Misconception: Chiplets Are Always Slower

Chiplets do introduce cross-die costs, but they also bring strong advantages in yield, cost, and scalability. For many high-concurrency, multi-instance, and cloud virtualization scenarios, chiplets are a very reasonable engineering choice.

### 9.3 Misconception: Monolithic Is Always Best

Monolithic designs provide a more uniform path, but very large dies face significant pressure in yield and cost, and core-count scaling is more limited. Monolithic is not universally optimal; it is simply better suited to some high-bandwidth, low-latency, tightly coupled scenarios.

### 9.4 Misconception: Arm AGI and Cobalt 200 Are the Same Chip

They both follow the Neoverse V3 / CSS V3 technology direction, but their product identities are different:

- Arm AGI is Arm’s own production CPU.
- Cobalt 200 is a Microsoft Azure customized SoC.

### 9.5 Misconception: Cobalt 200 Has Confirmed PCIe Gen6 / CXL 3.0

Those are officially disclosed Arm AGI CPU specifications. Publicly available Cobalt 200 materials do not currently present PCIe Gen6 / CXL 3.0 as clearly confirmed specifications, so the distinction should be preserved in learning notes.

## 10. Memory Aids

### 10.1 Three Approaches

- **NVIDIA Vera**: optimized first for agentic AI execution, single-thread performance, memory bandwidth, and the NVLink ecosystem.
- **AMD EPYC**: optimized first for chiplet scaling and the x86 ecosystem.
- **Arm AGI / Cobalt 200**: optimized first for cloud-native workloads, AI orchestration, security, and platform customization.

### 10.2 Three Keyword Sets

- Vera: **Olympus core + second-generation SCF + LPDDR5X/SOCAMM + NVLink-C2C**
- EPYC: **CCD + I/O Die + Infinity Fabric**
- Cobalt 200: **CSS V3 + 3MB L2/core + Azure Boost**

### 10.3 One-Sentence Analogy

> NVIDIA Vera is like a high-speed execution workshop inside an AI factory; AMD EPYC is like a modular city road network; Cobalt 200 is like an intelligent transportation system customized by Azure for its own cloud business.

## 11. Review Questions

1. Why should server CPUs not be judged only by core count?
2. How is SCF different from a conventional bus?
3. Why does Vera choose LPDDR5X/SOCAMM instead of traditional DDR5 DIMMs?
4. Why does AMD EPYC separate CCDs and the I/O Die?
5. How does NUMA affect database and virtualization workloads?
6. What kinds of CPU-side tasks in AI data centers are primarily served by Arm AGI CPU?
7. Why does Cobalt 200 emphasize compression, cryptography, and data movement accelerators?
8. Why should Arm AGI and Cobalt 200 not be described as “twin chips”?
9. Which Cobalt 200 facts are publicly confirmed, and which should not be inferred from Arm AGI?
10. If you were selecting a CPU for a cloud-native database, which architectural metrics would matter most?

## 12. Sources and Credibility Notes

These notes are organized based on public official sources and first-hand materials, including:

- official NVIDIA Vera CPU materials;
- NVIDIA Vera CPU technical blog posts;
- official NVIDIA Grace CPU / Grace CPU Superchip materials, used as background reference for Vera’s predecessor;
- official AMD EPYC product materials;
- official Arm Neoverse CSS V3 materials;
- official Arm AGI CPU materials and Arm Newsroom posts;
- Microsoft Azure Blog posts on Cobalt 200;
- Microsoft TechCommunity Azure Infrastructure Blog posts.
